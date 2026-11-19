# workstation-platform

Reproducible Ansible workstation configuration with mise-managed development tools, prek hooks, Conventional Commit validation, and SemVer releases.

The release subsystem is project-independent. The same `.cz.toml`, `.version`, `.mise/tasks/release/` tasks, and release hooks can be reused in the chezmoi repository.

## Prerequisites

Install these **before** running `scripts/bootstrap` or using project mise tasks:

| Prerequisite | Required | Purpose |
|---|---:|---|
| Git | Yes | Clone, commit, tag, and push |
| Bash | Yes | Bootstrap and release tasks |
| mise | Yes | Install and activate every project tool |
| Network access to package registries | Yes | Initial tool and hook installation |
| GitHub access using SSH or another authenticated Git transport | For publishing | Push commits and tags |
| GitHub CLI (`gh`) | Optional | Publish releases from the command line |
| Docker-compatible container runtime | Optional | Run Molecule Docker scenarios |

Important:

- Install mise through your workstation's existing owner, such as Homebrew. Do not install a second copy for this project.
- Activate mise in your shell. For zsh, add `eval "$(mise activate zsh)"` to the managed zsh configuration and open a new shell.
- Python, uv, prek, Commitizen, Ansible, and the linters are **not** prerequisites. The project installs them through mise.
- Project settings, environment variables, tools, and tasks become available only after you enter the repository, trust its `mise.toml`, and run `mise install`.
- Bootstrap does not use `sudo` and does not change the login shell or system configuration.

## Repository layout

```text
.
├── .cz.toml                    # Commit rules, current version, tag and changelog policy
├── .mise/
│   └── tasks/
│       └── release/
│           ├── bump            # Create a local release commit and annotated tag
│           ├── check           # Validate staged release artifacts
│           └── publish         # Push and publish the GitHub release
├── .pre-commit-config.yaml     # prek hook definitions
├── .version                    # Plain-text SemVer version; managed by Commitizen
├── CHANGELOG.md                # Generated during release bumps
├── mise.toml                   # Project tools and settings
└── scripts/
    └── bootstrap               # One-time setup after cloning
```

## Initial maintainer setup

1. Make the scripts executable:

   ```sh
   chmod +x scripts/bootstrap .mise-tasks/release/*
   ```

2. Install the mise tools:

   ```sh
   mise install
   ```

3. Install hooks and run the baseline checks:

   ```sh
   prek install --hook-type pre-commit --hook-type commit-msg --hook-type pre-push
   prek run --all-files
   ```

## Setup after cloning

Run:

```sh
git clone <repository-url>
cd workstation-platform
scripts/bootstrap
```

Bootstrap performs these operations:

1. Checks that Git and mise already exist.
2. Trusts this checkout's `mise.toml`.
3. Runs `mise install`.
4. Installs pre-commit, commit-msg, and pre-push hooks with prek.
5. Runs `prek run --all-files` to establish a clean baseline.

If bootstrap fails, stop and fix the reported prerequisite or check failure. Do not bypass hooks to make bootstrap pass.

## Development commands

| Command | Purpose |
|---|---|
| `mise tasks` | List project tasks |
| `mise install` | Reinstall exactly the pinned tools |
| `prek run` | Check files staged for the current commit |
| `prek run --all-files` | Check the complete repository |
| `prek run ansible-lint` | Run one hook on staged files |
| `prek run ansible-lint --all-files` | Run one hook on the complete repository |
| `cz check --rev-range origin/main..HEAD` | Check branch commit messages manually |
| `cz bump --dry-run --yes --changelog` | Preview automatic version detection |
| `mise run release:check` | Run the release-artifact check manually |

Hooks run automatically after bootstrap:

- **pre-commit:** YAML, Ansible, shell, whitespace, and release-artifact checks.
- **commit-msg:** the new commit message must follow Conventional Commits.
- **pre-push:** all commits in `origin/main..HEAD` are checked. This catches commits created through another UI or with commit-msg hooks bypassed.

CI should also run `prek run --all-files` and the branch commit check. Local hooks are not a security boundary because contributors can bypass them.

## Commit messages

Use this form:

```text
<type>(optional scope): <description>

[optional body]

[optional footer]
```

Examples:

```text
feat: add local container registry authentication
fix(mise): preserve the existing tool lockfile
docs: explain fork synchronization
feat!: remove the legacy shell bootstrap
```

Version effects:

| Commit content | Automatic bump |
|---|---|
| `fix:` | PATCH |
| `feat:` | MINOR |
| `BREAKING CHANGE:` footer or `!` after the type | MAJOR |
| `docs:`, `chore:`, `ci:`, `test:`, and similar | None |

A bump command exits without creating a release when no eligible commits exist. A forced increment or exact version can still create an empty release; see below.

## Version and release policy

`.cz.toml` is the Commitizen configuration. `.version` contains the same current version in plain text. Commitizen updates both during a bump. Do not edit either version value by hand.

Release tags use bare SemVer without a `v` prefix: `1.0.0`, `1.1.0`, `1.1.1`. Tags are annotated. `major_version_zero = false` means breaking changes increment MAJOR even before `1.0.0`.

Releases have two deliberate phases:

1. `release:bump` creates the release commit, updates `CHANGELOG.md`, and creates the annotated tag locally.
2. `release:publish` pushes `main` and the tag, then creates the GitHub release.

Nothing is published by the bump phase. Review the local release before publishing it.

### Preview a release

```sh
mise run release:bump -- --dry-run
```

### Automatic release from commit inspection

```sh
mise run release:bump
```

Commitizen examines commits since the previous release tag and selects MAJOR, MINOR, or PATCH.

### Force a segment on demand

```sh
mise run release:bump -- --increment MAJOR
mise run release:bump -- --increment MINOR
mise run release:bump -- --increment PATCH
```

Forced increments work even when there are no eligible commits.

### Break-glass exact version

```sh
mise run release:bump -- --version 2.0.0
```

Use this only to repair version metadata or handle an exceptional release. The task accepts stable `major.minor.patch` versions only. It still generates the changelog, release commit, and annotated tag. Record the reason in the issue or release discussion; break-glass releases are intentionally visible in Git history.

### Release-artifact check

The `release-state` pre-commit hook exits immediately for ordinary commits. When `.version` or `CHANGELOG.md` is staged, it requires both files and checks:

- `.version` is stable SemVer.
- `.version` and `.cz.toml` agree.
- The latest changelog release heading matches the new version.

`CHANGELOG.md` is therefore generated as part of each release, not regenerated after every commit.

### Publish from the command line

Prerequisites: install `gh`, run `gh auth login`, and ensure the Git remote is authenticated.

```sh
mise run release:publish
```

The task verifies that the tag is annotated and points to `HEAD`, generates notes for that release from Commitizen, pushes without force, and creates the GitHub release. It does not overwrite an existing GitHub release.

### Publish through the GitHub UI

Use this when `gh` publication is unavailable but Git push is available:

1. Run and review `mise run release:bump` locally.
2. Push the release commit and annotated tag:

   ```sh
   git push origin main
   git push origin "$(tr -d '[:space:]' < .version)"
   ```

3. In GitHub, open **Releases → Draft a new release**.
4. Select the **existing** SemVer tag. Do not create a second tag.
5. Use the version as the title.
6. Copy that version's section from `CHANGELOG.md` into the notes. Do not substitute GitHub's automatically generated notes.
7. Publish the release.

The GitHub UI can also create a new tag, but that is a degraded path. The normal process requires the annotated tag produced by Commitizen.

### If a release goes wrong

Before publication, inspect and correct local state without rewriting any public history. After publication, do not delete or move the public tag to conceal the error. Create a corrective PATCH release or use a documented break-glass version bump.

## Fork and upstream workflow

Colleagues should fork the repository and treat their fork as `origin` and this repository as `upstream`:

```sh
git clone git@github.com:<colleague>/workstation-platform.git
cd workstation-platform
git remote add upstream git@github.com:<maintainer>/workstation-platform.git
git fetch upstream
git switch -c feat/my-change upstream/main
```

Keep fork `main` available for synchronization. Develop on feature branches. Before a pull request:

```sh
git fetch upstream
git rebase upstream/main
prek run --all-files
cz check --rev-range upstream/main..HEAD
```

Only the canonical maintainer cuts upstream releases on upstream `main`. Forks may customize behavior, but they should not create same-named release tags on divergent histories if they intend to keep merging upstream releases.

## Updating development tools

Tool versions belong in `mise.toml` and `mise.lock`, not in local hook definitions. Update them deliberately:

```sh
# Edit the desired pins in mise.toml first.
mise lock
mise install
prek run --all-files
git add mise.toml mise.lock
git commit -m "build: update development tools"
```

Review lockfile changes like code. Do not run broad upgrades immediately before a release.

## Reuse in the chezmoi repository

Copy `.cz.toml`, `.version`, `CHANGELOG.md`, `.mise/tasks/release/`, `scripts/bootstrap`, the Commitizen hook entries, and the `release-state` hook. Add the same `pypi:commitizen` pin. Remove Ansible-only hooks where they do not apply; the version, tag, changelog, and publication workflow needs no Ansible-specific changes.