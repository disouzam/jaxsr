# Releasing JAXSR

How to cut a new release that ships to PyPI and mints a Zenodo DOI
automatically. Follow this top to bottom — each section is a checklist.

## One-time setup (do once, never again)

These are repo- and account-level prerequisites. They should already be
in place; verify before your first release or if you move the repo.

- [ ] **PyPI trusted publisher** registered at
      <https://pypi.org/manage/project/jaxsr/settings/publishing/>
      with the following exact values. Any mismatch causes
      `invalid-publisher` errors (see Troubleshooting).
      - PyPI Project Name: `jaxsr`
      - Owner: `jkitchin`
      - Repository name: `jaxsr`
      - Workflow name: **`workflow.yml`** (note: not `publish.yml`)
      - Environment name: `pypi`
- [ ] **Zenodo GitHub integration** enabled at
      <https://zenodo.org/account/settings/github/>. The
      `jkitchin/jaxsr` toggle must be ON. Without this, no DOI is minted.
- [ ] **GitHub environment `pypi`** exists at `Settings → Environments`.
      Auto-created on first workflow run, but you can pre-create it to
      add required reviewers as an approval gate.

## Pre-release checklist

Before starting any release work, verify the repo is ready to ship:

- [ ] `git status` on `main` is clean — no stray files, no uncommitted work.
- [ ] `main` is up to date with `origin/main` (`git pull`).
- [ ] `pytest tests/ -v --tb=short --timeout=60` passes locally.
- [ ] `black --check src/ tests/` passes.
- [ ] `ruff check src/ tests/` passes.
- [ ] Docs build without errors if you touched anything under `docs/`.
- [ ] No unresolved issues in the milestone for this release (if you
      use milestones).
- [ ] Breaking changes (if any) are documented in the release notes
      draft and follow semver — a breaking change means a minor or
      major bump, not a patch bump.

### Manuscript & notebook verification

Releases are infrequent enough that this is the right moment to catch
drift between the code and the things that document it. Do not skip.

- [ ] **Manuscript audit** — only if `manuscript/` or `src/jaxsr/`
      changed since the last tag:
      ```bash
      git diff v<LAST>..HEAD -- manuscript/ src/jaxsr/
      ```
      If non-empty, re-check every API call in `manuscript/jaxsr-paper.org`
      against the current signatures. The "Red-Team Review (API Correctness)"
      table in `CLAUDE.md` lists the common mistakes. A runtime-broken
      code example in the paper is a release blocker.

- [ ] **Notebook static validation** — always:
      ```bash
      python scripts/validate_notebooks.py
      ```

- [ ] **Notebook execution** — always, locally, for every release:
      ```bash
      find docs/examples -name "*.ipynb" -print0 | \
        xargs -0 -n1 jupyter execute --timeout=300
      ```
      Every notebook must run to completion without errors. A failing
      notebook blocks the release until it is either fixed, or quarantined
      (rename to `*.ipynb.skip` with a tracking issue in the commit
      message).

### Changelog

- [ ] Promote the `## [Unreleased]` section in `CHANGELOG.md` to
      `## [X.Y.Z] - YYYY-MM-DD` (today's date, ISO format). Start a
      fresh empty `## [Unreleased]` section above it for the next
      release cycle.
- [ ] Review the promoted section one more time — it will become the
      GitHub release notes verbatim, so each bullet should be written
      for users, not for you. Strip internal-only items
      (`chore: bump version`, gitignore tweaks, etc.).

## Release steps

### 1. Bump the version in exactly three places

Keep these in sync or `pip install jaxsr` will ship with one version
while `jaxsr.__version__` reports another.

| File | Line | Format |
|---|---|---|
| `pyproject.toml` | `version = "X.Y.Z"` | TOML string |
| `src/jaxsr/__init__.py` | `__version__ = "X.Y.Z"` | Python string |
| `CITATION.cff` | `version: X.Y.Z` | YAML bare scalar (no quotes) |

Quick grep to verify:

```bash
grep -rn '0\.[0-9]\+\.[0-9]\+' pyproject.toml CITATION.cff src/jaxsr/__init__.py
```

All three should show the new version.

### 2. Commit the version bump

```bash
git add pyproject.toml src/jaxsr/__init__.py CITATION.cff
git commit -m "chore: bump version to X.Y.Z"
```

**Do not tag yet.** The tag needs to point at this commit.

### 3. Push `main` first, then tag, then push tag

Order matters. Tagging before pushing main means the tag exists on a
commit the remote doesn't yet have — harmless but noisy. Push in this
order:

```bash
git push origin main
git tag -a vX.Y.Z -m "JAXSR vX.Y.Z"
git push origin vX.Y.Z
```

Use **annotated tags** (`-a`), not lightweight tags. Zenodo and GitHub
releases both expect an annotated tag object with a message.

### 4. Extract release notes from CHANGELOG.md and create the release

The `[X.Y.Z]` section you promoted during the pre-release checklist
becomes the release notes. Extract it with awk:

```bash
VERSION="X.Y.Z"
awk -v ver="$VERSION" '
  $0 ~ "^## \\[" ver "\\]" { in_section = 1; next }
  in_section && (/^## \[/ || /^\[[^]]+\]:/) { exit }
  in_section { print }
' CHANGELOG.md > /tmp/jaxsr-v${VERSION}-notes.md
```

Optionally prepend an install hint and a citation reminder (these are
boilerplate you probably don't want duplicated in CHANGELOG.md itself):

```bash
cat > /tmp/jaxsr-v${VERSION}-header.md <<'EOF'
Install with:

    pip install jaxsr

EOF
cat /tmp/jaxsr-v${VERSION}-header.md /tmp/jaxsr-v${VERSION}-notes.md > \
    /tmp/jaxsr-v${VERSION}-full.md
cat >> /tmp/jaxsr-v${VERSION}-full.md <<'EOF'

### Citing

If you use JAXSR in published work, please cite via the "Cite this
repository" button (uses `CITATION.cff`). A Zenodo DOI is minted
automatically for each release.
EOF
```

Then create the release — **this is the "go" button** that triggers
both Zenodo archival and the PyPI publish workflow:

```bash
gh release create "v${VERSION}" \
  --title "v${VERSION}" \
  --notes-file /tmp/jaxsr-v${VERSION}-full.md
```

### 5. Watch the publish workflow

```bash
gh run list --workflow=workflow.yml --limit 1
gh run watch <run-id> --exit-status --interval 5
```

Expected: `Build distribution` passes (~10s), then `Publish to PyPI`
passes (~15–25s). Total ~30s.

## Post-release verification

Within a minute or two of a successful workflow run, check all three:

- [ ] **PyPI**: <https://pypi.org/project/jaxsr/> shows `X.Y.Z` as the
      latest version. Verify with:
      ```bash
      curl -sf https://pypi.org/pypi/jaxsr/json | python3 -c \
        "import sys,json; print(json.load(sys.stdin)['info']['version'])"
      ```
- [ ] **GitHub release**: <https://github.com/jkitchin/jaxsr/releases>
      shows `vX.Y.Z` marked *Latest*.
- [ ] **Zenodo DOI**: <https://zenodo.org/account/settings/github/>
      shows a new entry under `jkitchin/jaxsr` with a DOI. Copy the
      DOI (format: `10.5281/zenodo.NNNNNNN`).

### Follow-up commit (only needed on the *first* release after a DOI changes)

The concept DOI (the one that always points at the latest version) is
minted once and never changes, but if you want to link a specific
version DOI in `CITATION.cff`, or if this is the very first time a DOI
exists for the project, do this small follow-up:

1. Add the DOI to `CITATION.cff`:
   ```yaml
   doi: 10.5281/zenodo.NNNNNNN
   ```
2. Add the Zenodo badge to `README.md` if it isn't already there:
   ```markdown
   [![DOI](https://zenodo.org/badge/REPO_ID.svg)](https://doi.org/10.5281/zenodo.NNNNNNN)
   ```
   Get `REPO_ID` from the badge URL Zenodo shows on your deposit page.
3. Commit with a message like `docs: add Zenodo DOI badge and link
   citation metadata`. No new release needed.

## Troubleshooting

### `invalid-publisher: Publisher with matching claims was not found`

The OIDC token the workflow sent to PyPI doesn't match any registered
trusted publisher. The error page prints the claims the workflow is
sending; compare each one against
<https://pypi.org/manage/project/jaxsr/settings/publishing/>.

Fields that must match exactly:
- `repository_owner` (case-sensitive) ↔ Owner
- `repository` (without the owner prefix) ↔ Repository name
- `workflow_ref` filename ↔ Workflow name (including the `.yml` suffix)
- `environment` claim ↔ Environment name

Common mismatches:
- Environment is blank on PyPI but set to `pypi` in the workflow
  (or vice versa). Either side's blank plus the other side's value
  will reject.
- Workflow filename typo: `workflow` vs `workflow.yml` vs `publish.yml`.
- Registered on <https://test.pypi.org> by mistake instead of
  <https://pypi.org>.

**Important:** if the mismatch is in a committed file (e.g. the
workflow filename), you **cannot** fix it by rerunning the failed job —
the OIDC claim is frozen at the commit the workflow was triggered from.
You must either (a) fix the registration on PyPI to match the committed
workflow, or (b) rename the workflow in a new commit and cut a new
release. Option (a) is always preferable.

### `gh run rerun --failed` fixed it without a new release

This only works when the failure is on PyPI's side and the committed
files are already correct. If you rename a workflow file locally, a
rerun will NOT pick up the rename — reruns use the original commit.

### PyPI has a version gap (e.g. 0.2.0 → 0.2.2, missing 0.2.1)

This happens when a release fails to publish and you cut a new one at
a higher version instead of reclaiming the number. Once any release
has been *attempted* under a version number (even if nothing reached
PyPI), it is cleanest to skip that number rather than try to reuse it.
PyPI version numbers are cheap; history clarity is not. Accept the gap.

### Zenodo didn't mint a DOI

The most common causes:
- `jkitchin/jaxsr` toggle is OFF at
  <https://zenodo.org/account/settings/github/>.
- You created the release *before* the toggle was flipped on. Zenodo
  only listens forward in time — earlier releases won't be archived
  retroactively.
- You used a lightweight tag instead of an annotated tag. Annotate
  with `git tag -a`.

Zenodo is independent of the PyPI workflow — a PyPI failure does not
affect Zenodo, and vice versa.

### The tag exists but there's no GitHub release

A tag is not a release on GitHub. A release is a separate object that
references a tag. Creating `git tag && git push --tags` alone does
**not** trigger the publish workflow — the trigger is `on: release:
[published]`. You must run `gh release create` (or click "Publish
release" in the UI) to actually cut a release.

## Quick reference: full release in one paste

For when you've done this enough times to trust yourself. Not a
substitute for the checklists above — still do the manuscript audit
and the notebook execution pass manually before running any of this.

```bash
VERSION="X.Y.Z"

# 1. Promote CHANGELOG.md [Unreleased] to [X.Y.Z] - today, add new
#    empty [Unreleased]. Bump version in pyproject.toml,
#    src/jaxsr/__init__.py, CITATION.cff. Do these by hand.

# 2. Commit, push, tag, push tag.
git add CHANGELOG.md pyproject.toml src/jaxsr/__init__.py CITATION.cff
git commit -m "chore: bump version to ${VERSION}"
git push origin main
git tag -a "v${VERSION}" -m "JAXSR v${VERSION}"
git push origin "v${VERSION}"

# 3. Extract notes from CHANGELOG.md and cut the release.
awk -v ver="$VERSION" '
  $0 ~ "^## \\[" ver "\\]" { in_section = 1; next }
  in_section && (/^## \[/ || /^\[[^]]+\]:/) { exit }
  in_section { print }
' CHANGELOG.md > "/tmp/jaxsr-v${VERSION}-notes.md"

gh release create "v${VERSION}" \
  --title "v${VERSION}" \
  --notes-file "/tmp/jaxsr-v${VERSION}-notes.md"

# 4. Watch the workflow.
gh run list --workflow=workflow.yml --limit 1
gh run watch <run-id> --exit-status --interval 5

# 5. Verify the three endpoints (PyPI, GitHub, Zenodo).
```
