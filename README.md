# Wellmanifest Python Standard (`wellmanifest/python`)

> Canonical standard and reference guide for Python runtime architecture, CLI distribution lifecycles (`goal -a`), zero-downtime development workflows, and multi-credential resilience in autonomous agent environments.

---

## 1. Problem Overview: The Dual-Runtime Hazard

In automated development ecosystems (such as Subactor and Koru), developers and autonomous agents frequently face runtime discrepancies caused by the collision of two execution contexts:

1. **The System / Host Runtime:** Installed as pre-built wheels in a host virtualenv (e.g. `/opt/llmhub/provider-tools-venv` or `/usr/local/bin`), owned by `root`, static and frozen.
2. **The Active Source Working Tree:** Cloned under `~/github/*/*`, actively modified by humans or LLM agents.

### Why standard Python mechanics fail in this setup:
* **`ENABLE_USER_SITE: False` in virtual environments:** Standard Python virtual environments explicitly disable user site-packages (`~/.local/lib/python3.X/site-packages`). Running `pip install --user` or dropping files in user-site does not affect commands executing through the virtualenv's Python binary.
* **Shell Path and `hash -r` caching:** Interactive shells (`bash`, `zsh`) cache binary paths in an internal hash table. After changing files in `~/.local/bin` or modifying `$PATH`, existing shells continue executing the cached binary path until `hash -r` is called.
* **Login Shell vs Non-Login Shell Discrepancies:** Login shells (`bash -l`, SSH, display managers) read `~/.bash_profile` or `~/.profile`, whereas non-login subshells read `~/.bashrc`. If `$PATH` and `$PYTHONPATH` are not harmonized across both, background tasks and agent subprocesses run against different Python packages than interactive terminals.

---

## 2. Zero-Downtime Development Workflow

To ensure that code changes made in local repositories (`~/github/*/*`) take effect immediately in the command line without requiring rebuilding wheels or reinstalling:

### A. Private Developer Bin Wrappers (`~/.local/bin/<cmd>`)
Place a prioritized wrapper script in `~/.local/bin/` that points `PYTHONPATH` to the repository's `src/` directory before delegating to the target interpreter:

```bash
#!/bin/bash
export PYTHONPATH="/home/tom/github/semcod/koru/src:${PYTHONPATH}"
exec /opt/llmhub/provider-tools-venv/bin/python -m koru.cli "$@"
```

### B. Path Precedence Configuration
Ensure `~/.profile` and `~/.bash_profile` always place `~/.local/bin` at the beginning of `$PATH`:

```bash
export PATH="$HOME/.local/bin:$PATH"

if [ -n "$BASH_VERSION" ] && [ -f "$HOME/.bashrc" ]; then
    . "$HOME/.bashrc"
fi
```

### C. Live Shell Invalidation
When modifying CLI arguments or subcommands during pair-programming, invalidate the shell cache in active windows:
```bash
source ~/.bash_profile && hash -r
```

---

## 3. Distribution Lifecycle: Publishing with `goal -a`

While the zero-downtime workflow handles local development, official distribution across systems and venvs is managed via `goal -a`.

### What `goal -a` executes:
`goal -a` (or `goal all`) runs the full governed release workflow:
1. **Quality Gates & Pre-flight Testing:** Runs local test suites (`pytest`, linters).
2. **Version Bump:** Updates versions across `pyproject.toml`, `VERSION`, and internal metadata (e.g. patch increment `0.1.461` → `0.1.462`).
3. **Git Packaging:** Adds, commits with structured summaries, creates tags, and pushes to remote.
4. **Registry Publication:** Builds distributions (wheel & sdist) and uploads them to PyPI using `$PYPI_TOKEN` or `~/.pypirc`.
5. **Dependency Propagation (`-u` / `--upgrade-deps`):** Updates project and consumer virtual environments to install the newly published package version.

### Key `goal -a` Commands:
```bash
# Full workflow (tests, bump, commit, push, publish)
goal -a

# Force publication when changes are already committed
goal -a --force-publish

# Full workflow with automated dependency upgrades in downstream venvs
goal -a -u
```

---

## 4. Multi-Credential & Rate-Limit Resilience

Autonomous agents communicating with external services (GitHub API, PyPI, LLM providers) frequently encounter authentication and rate-limiting barriers.

### A. Distinguishing Rate Limits from Permission Errors
API backends often return HTTP 403 for both missing permissions and rate limit saturation. Standard implementations must separate them:
* **Primary Rate Limit (Hourly Quota):** E.g. GitHub's 5000 requests/hour tied to the **Account / User ID**, regardless of how many Personal Access Tokens (PATs) are generated for that account.
* **Secondary Rate Limit (Burst / Abuse Detection):** Triggered when creating or editing resources too rapidly, returning HTTP 403 with `Retry-After`.
* **Remediation:** Do not report rate limits as invalid permissions. Report the cooldown timestamp and offer CLI profile switching or vault lease rotation.

### B. Multi-Account Profile Switching
To bypass account-level hourly limits, maintain dedicated machine/bot accounts:
* **GitHub CLI:**
  ```bash
  gh auth login                # Add auxiliary bot account
  gh auth switch --user <bot>  # Switch active identity
  koru git switch-profile <bot> # Switch and sync .env
  ```
* **PyPI Configuration (`~/.pypirc`):**
  Maintain named repository sections rather than a single monolithic token:
  ```ini
  [distutils]
  index-servers = pypi-release pypi-staging

  [pypi-release]
  username = __token__
  password = pypi-...
  ```

### C. Secure Token Leases (`@subactor/credential-vault`)
For automated environments, credentials should not be stored in plaintext. Use `@subactor/credential-vault`:
* Short-lived leases via `VaultLeaseClient`.
* Origin-bound reads and typed operation scopes.
* Masked output in logs and telemetry.

---

## 5. Background Agent Protection & Worktree Isolation

When background daemons (such as PR validators, continuous auto-loops, or sync runners) operate in the same checkout:
1. **Never mutate active developer checkouts:** Background scripts must never run destructive checkouts, resets, or `git stash` operations on the primary branch.
2. **Adopt Wellmanifest Worktrees v5:** Every autonomous task or test runner must execute in an isolated worktree (`.worktrees/ticket-NNN--description`).
3. **Locking & Conformance:** Verify worktree leases (`.subactor/leases/*.json`) before modifying files.

---

## 6. Diagnostic Cheatsheet

| Symptom | Root Cause | Solution |
| :--- | :--- | :--- |
| `unrecognized arguments: subcommand` | Shell is running stale `/usr/local/bin` binary | Run `source ~/.bash_profile && hash -r` |
| `ENABLE_USER_SITE: False` | Virtualenv ignoring `~/.local/lib` | Use `~/.local/bin` wrapper with `PYTHONPATH` |
| `API rate limit exceeded for user ID` | Account hourly limit exhausted | Switch account: `gh auth switch` or wait for reset |
| `403 Forbidden` on ticket sync | Concurrency burst or token missing scope | Verify `gh auth status` and inspect `Retry-After` |
| Uncommitted changes vanished | Background script ran `git stash` | Check `git stash list` and restore via `git stash pop` |
| `build_content_differs` / drift warning | `dist/` wheel differs from active source tree | Rebuild wheel with `python -m build --wheel` (PYTHON-DIST-001) |

---

## 7. Normative Rules Index

* **`PYTHON-DIST-001` (Commit-Artifact Parity):** Every wheel/sdist in `dist/` must match the source tree byte-for-byte; version number equivalence alone does not constitute build parity.
* **`PYTHON-DIST-002` (Three-Fact Introspection):** CLI `--version` commands must report source version/commit, installed environment metadata, and newest built artifact in `dist/`.
* **`PYTHON-DIST-003` (Byte-for-Byte Build Verification):** Application runtime must verify module content parity against the built wheel at startup or test gates and fail closed on drift.

