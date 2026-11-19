## 0.2.0 (2026-11-19)

### Feat

- add yamllint configuration aligned with ansible-lint settings
- move script to list package versions from PyPI
- add pre-commit configuration for code quality checks and commit message validation
- add publish script for pushing releases and creating GitHub releases
- add release bump script for creating SemVer release commits and changelogs
- add release check script for version and changelog validation
- add bootstrap script for project setup
- **cz.toml**: add commitizen configuration
- :sparkles: Update tools and PYPI packages in mise.toml configuration
- :hammer: Add pypi-versions script to list package versions from PyPI

### Fix

- add retry option to release bump script for handling pre-commit hook rejections
- standardize formatting and improve error handling in scripts
- correct path for making release scripts executable in README
- update file permissions to be executable for release scripts and bootstrap
