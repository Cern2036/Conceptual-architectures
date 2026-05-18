# Changelog

## [Unreleased]
### Changed
- Repository restructured: flat .txt files organized into docs/en/ and docs/ru/
- Converted all documents to Markdown (.md) with YAML frontmatter
- Added structured README.md with documentation index
- Added .gitignore

## [4.0.0] - 2026-01
### Added
- Full technical documentation suite for Project ZERO-1
- English and Russian versions of all documents
- Expert evaluation and industrial readiness analysis
- Deployment scripts, disaster recovery plans, troubleshooting guides

## [4.1.0] - 2026-05-19
### Added
- ZERO-1 MVP: working prototype with FastAPI, web UI, and 12 DISA STIG rules
- MVP components: engine.py (Cisco parser + STIG checker + DeepSeek AI), zero1_web.py (web interface)
- Dockerfile for containerized deployment
- requirements.txt with pinned dependencies
- MVP verified: syntax OK, imports OK, all 12 STIG rules functional

### Changed
- Repository scoped to Project ZERO-1

### Removed
- Empty/placeholder repositories cleaned up
