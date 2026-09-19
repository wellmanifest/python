# Troubleshooting Python Environments in Autonomous Architectures

This guide provides diagnostics and quick fixes for issues encountered when developing, running, and publishing Python packages across the host ecosystem.

---

## 1. Subcommand or Argument Not Recognized in CLI

### Symptoms
```text
koru: error: unrecognized arguments: github token
koru: 'github' is not a known subcommand. Did you mean 'koru git'?
```

### Diagnostic Steps
1. Verify which binary the current shell is executing:
   ```bash
   which koru
   type -a koru
   ```
2. Check if the shell has cached a stale binary location:
   ```bash
   hash -t koru
   ```
3. Check active `$PATH` and `$PYTHONPATH`:
   ```bash
   echo "PATH=$PATH"
   echo "PYTHONPATH=$PYTHONPATH"
   ```

### Fix
* Clear command hash in active shell:
  ```bash
  hash -r
  ```
* Re-source profile configuration:
  ```bash
  source ~/.bash_profile
  ```
* Ensure `~/.local/bin` wrapper points to the repo's `src/` directory.

---

## 2. Changes in Repo Disappeared (`git stash` hazard)

### Symptoms
Files modified in `~/github/*/*` suddenly revert to clean git HEAD while background agents are running.

### Cause
A background daemon, CI runner, or PR validator (such as `publish-local-onedev-validator.sh`) executed on the main repository working directory and stashed unstaged changes before running git operations.

### Diagnostic Steps
List all git stashes in the repository:
```bash
git stash list
```

### Fix
1. Inspect the top stash:
   ```bash
   git stash show -p stash@{0}
   ```
2. Restore the changes:
   ```bash
   git stash pop stash@{0}
   ```
3. Commit the changes immediately on a dedicated ticket branch or worktree:
   ```bash
   git commit -m "feat(...): commit changes"
   ```

---

## 3. GitHub API Rate Limit (403 RateLimitExceededException)

### Symptoms
```text
github.GithubException.RateLimitExceededException: 403 {"message": "API rate limit exceeded for user ID ..."}
```

### Cause
The hourly REST API quota (5,000 requests/hour for personal accounts) is shared by all tokens under the same GitHub user account. Alternatively, secondary burst limits were triggered.

### Fix
1. Check quota status and reset time:
   ```bash
   gh api rate_limit
   ```
2. Switch to an auxiliary bot account:
   ```bash
   gh auth switch --user <bot_username>
   koru git switch-profile <bot_username>
   ```
3. Wait until the reset timestamp displayed in `gh api rate_limit`.

---

## 4. Virtualenv Ignores User Packages (`ENABLE_USER_SITE=False`)

### Symptoms
Installing packages via `pip install --user <pkg>` has no effect when running tools from `/opt/...` virtualenv.

### Cause
Python virtual environments intentionally disable user site-packages to ensure isolation.

### Fix
Do not rely on `user-site` for virtualenv overrides. Instead:
1. Use `~/.local/bin/<cli>` wrapper that explicitly injects `PYTHONPATH="/path/to/repo/src:$PYTHONPATH"`.
2. Or publish new package versions to PyPI via `goal -a` and update the virtualenv with `goal -a -u` or `/opt/llmhub/provider-tools-venv/bin/pip install --upgrade <pkg>`.
