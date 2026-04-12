# Changelog

All notable changes to JAXSR are documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

Releases prior to 0.2.2 predate this changelog; see the git history
for details.

## [Unreleased]

### Added
- `RELEASING.md` — release checklist and troubleshooting guide covering
  version bumps, CHANGELOG.md promotion, manuscript currency audit,
  notebook execution, tagging, GitHub release, PyPI trusted publishing,
  and Zenodo archival.
- `CHANGELOG.md` — this file, following Keep a Changelog format.

## [0.2.2] - 2026-04-12

### Added
- Automated release publishing to PyPI via GitHub release trigger,
  using OIDC trusted publishing (no long-lived API tokens in the repo).
- Zenodo archival integration — each GitHub release is now automatically
  archived on Zenodo and issued a citable DOI.
- `CITATION.cff` with linked DOI, so GitHub renders a "Cite this
  repository" button in the sidebar.
- Zenodo DOI badge in `README.md`.
- GitHub release badge in `README.md`.
- Manuscript source now tracked in the repo at
  `manuscript/jaxsr-paper.org` and `manuscript/references.bib`.
- `.zenodo.json` with metadata used by Zenodo on each release.

### Fixed
- `manuscript/jaxsr-paper.org`: the `ResponseSurface` code example now
  passes the required `bounds` argument (was a runtime crash
  as-written).
- `manuscript/jaxsr-paper.org`: architecture section updated to reflect
  20 modules and ~20,000 lines of Python.
- `manuscript/jaxsr-paper.org`: `MultiOutputSymbolicRegressor` moved
  from "future work" to shipped contributions, reflecting that it is
  already exported from `jaxsr.__init__`.
- `manuscript/jaxsr-paper.org`: Physical Constraints section now
  documents `constraint_selection_weight` for constraint-aware
  selection, not only post-selection refit.

[Unreleased]: https://github.com/jkitchin/jaxsr/compare/v0.2.2...HEAD
[0.2.2]: https://github.com/jkitchin/jaxsr/releases/tag/v0.2.2
