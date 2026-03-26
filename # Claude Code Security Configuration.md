# Claude Code Security Configuration

> **Note:** Claude Code reads `settings.json` only.  
> `settings-comments.jsonc` is the annotated reference — keep it in sync when `settings.json` changes.

---

## Design Philosophy

Two independent enforcement layers are applied to every sensitive resource:

| Layer | Mechanism | What it blocks |
|---|---|---|
| **Model-level** | `permissions.deny` / `permissions.ask` | Claude's intent — the model refuses or pauses |
| **OS-level** | `sandbox.filesystem` | The underlying syscall — blocked even if the model is deceived |

This is defense-in-depth. A prompt injection attack that fools the model still hits a hard kernel-level wall.

---

## Protection Categories

### 1. SSH Keys & Remote Access
**Files:** `~/.ssh/**`  
Private keys are sufficient on their own to establish remote access. Read access alone is enough to exfiltrate them — write access could silently add an attacker's public key to `authorized_keys`.

### 2. Cloud Provider Credentials
**Files:** `~/.aws/**`, `~/.azure/**`, `~/.config/gcloud/**`  
Long-lived tokens and service account keys. Exposure can result in full cloud account takeover, data exfiltration, or infrastructure destruction.

### 3. Git Credentials & Config
**Files:** `~/.gitconfig`, `~/.git-credentials`, `~/.config/git/**`, `**/.git/config`  
Git credential stores often hold Personal Access Tokens with repo read/write or org-level access. Config modification could silently redirect push targets to an attacker-controlled remote.

### 4. Container & Orchestration Secrets
**Files:** `~/.docker/config.json`, `~/.kube/**`, `~/.config/helm/**`  
Docker config stores registry auth tokens. Kubeconfig grants cluster control plane access. Exposure can allow silent image publishing or cluster manipulation.

### 5. Package Manager Tokens
**Files:** `~/.npmrc`, `~/.cargo/credentials.toml`, `~/.gem/credentials`, `~/.pypirc`, `~/.composer/auth.json`, `~/.gradle/gradle.properties`, `~/.m2/settings.xml`  
Registry auth tokens often carry publish rights. A leaked npm token could be used to push a malicious package update to a public or private registry.

### 6. Password Managers & Vaults
**Files:** `~/.config/op/**`, `~/.config/bw/**`, `~/.vault-token`, `~/.config/vault/**`  
These directories store local session tokens for 1Password, Bitwarden, and HashiCorp Vault. A valid session token can unlock the full credential store without a master password.

### 7. TLS / PKI Certificates & Private Keys
**Files:** `**/*.pem`, `**/*.key`, `**/*.p12`, `**/*.pfx`, `**/*.crt`, `**/*.cer`, `**/*.der`  
Applied as a filesystem-wide wildcard to catch project-local certificates and keys that developers sometimes store inside repositories. Private key material allows impersonation and traffic decryption.

### 8. Shell History
**Files:** `~/.bash_history`, `~/.zsh_history`, `~/.zsh_history`, `~/.local/share/fish/fish_history`  
Shell history frequently contains plaintext credentials passed as CLI arguments — a common developer habit (e.g., `curl -H "Authorization: Bearer <token>"`).

### 9. Browser Data
**Files:** `~/Library/Application Support/Google/Chrome/**`, `Firefox`, `Edge`, `Safari`  
Browser profiles contain saved passwords, session cookies, and OAuth tokens. Chrome extension storage (covered here) includes MetaMask's encrypted wallet vault.

### 10. CI/CD & Hosting Provider Tokens
**Files:** `~/.config/gh/**`, `~/.config/glab/**`, `~/.fly/**`, `~/.config/heroku/**`, `~/.config/netlify/**`, `~/.config/vercel/**`  
Access tokens for deployment platforms. Exposure could allow manipulation of production deployments or CI/CD pipelines.

### 11. Infrastructure-as-Code Secrets
**Files:** `~/.terraformrc`, `~/.terraform.d/**`  
Terraform stores provider tokens (AWS, Azure, GCP, etc.) locally. Exposure allows silent infrastructure modification outside normal change control.

### 12. Database Credentials
**Files:** `~/.pgpass`, `~/.my.cnf`, `~/.mylogin.cnf`, `~/.mongosh/**`  
PostgreSQL, MySQL, and MongoDB store connection credentials — sometimes in plaintext — for developer convenience.

### 13. GPG & macOS Keychain
**Files:** `~/.gnupg/**`, `~/.netrc`, `~/Library/Keychains/**`  
GPG keyring contains private signing keys used for commit signing and encrypted communication. macOS Keychain stores Wi-Fi passwords, app tokens, and system certificates.

### 14. Environment Files & Secret Naming Patterns
**Files:** `**/.env*`, `**/secrets/**`, `**/*credential*`, `**/*secret*`  
Wildcard rules that catch developer secrets regardless of location. `.env` files in particular are a common source of accidental credential exposure.

### 15. Cryptocurrency Wallets & Keys
**Files:** Bitcoin, Ethereum keystore, Solana, Ledger, Trezor, `*.wallet`, `*.seed`, `*mnemonic*`, `*seed_phrase*`  
Unlike most credentials, crypto key material **cannot be revoked**. A drained wallet is an unrecoverable loss. Coverage includes software wallets, CLI keypairs, hardware wallet software data, and common seed phrase file naming patterns.

### 16. macOS System Integrity
**Commands:** `launchctl`, `osascript`, `~/Library/LaunchAgents/**`  
LaunchAgents are a standard macOS persistence mechanism. `osascript` (AppleScript) can interact with the UI and bypass permission prompts. `launchctl` can load and unload system services.

### 17. Claude Self-Modification Boundary
**Files:** `~/.claude/settings.json`, `~/.claude/CLAUDE.md`, `~/.claude/hooks/**`  
A critical containment rule: Claude cannot modify its own permission configuration, system prompt, or hooks. Without this, a sufficiently deceptive prompt could instruct Claude to expand its own access.

### 18. Shell Profile Modification
**Files:** `~/.zshrc`, `~/.zshenv`, `~/.bashrc`, `~/.bash_profile`, `~/.profile`, `~/.config/fish/config.fish`  
Shell profiles execute on every new terminal session. Modification is a classic persistence and environment hijacking vector.

### 19. Destructive Shell Commands
**Commands:** `rm -rf`, `rm -r`, `rmdir`, `dd`, `mkfs`, `shred`  
Hard-denied — these commands cause irreversible data loss and have no place in an AI coding assistant's default capability set.

### 20. Privilege Escalation
**Commands:** `sudo`, `su`, `doas`, `chown`, `chmod 777`  
Prevents Claude from acquiring root or elevated privileges. `chmod 777` is included as it is commonly used to weaken file permissions as a precursor to exploitation.

### 21. Network Exfiltration
**Commands (denied):** `nc`, `ncat`, `scp`, `rsync`, `WebFetch`  
**Commands (ask):** `curl`, `wget`, `ssh`  
Raw socket and silent file transfer tools are denied outright. `curl`, `wget`, and `ssh` have legitimate development uses but require explicit approval before each invocation.

### 22. Code Injection Patterns
**Commands:** `python -c`, `node -e`, `perl -e`, `bash -c`, `sh -c`, `eval`, etc.  
Interpreter one-liner flags are a common prompt injection technique — a malicious instruction embedded in a file Claude reads could use these to execute arbitrary code.

### 23. Dangerous Git Operations
**Commands (denied):** `git push --force`, `git push -f`, `git commit --no-verify`, `git commit -n`  
**Commands (ask):** `git push`  
Force push can permanently destroy remote history. `--no-verify` bypasses pre-commit hooks, which may include security scanners. All pushes require human approval.

---

## What Requires Human Approval (`ask`)

These commands are not denied but will pause and prompt for confirmation:

| Command | Reason |
|---|---|
| `curl` / `wget` | HTTP requests / file downloads could exfiltrate data or fetch payloads |
| `ssh` | Remote shell access |
| `export` | Environment variable mutation — could affect subsequent commands |
| `git push` | All pushes require a human sign-off |
| `npm` / `pip` / `cargo` / `gem install` | Unvetted package installation |
| `docker run` / `docker pull` | Arbitrary container execution / unvetted images |

---

## Keeping This In Sync

When you update `settings.json`, update `settings-comments.jsonc` to match.  
This README reflects the rationale — update it when the threat model changes.

Last reviewed: 2026-03-26
