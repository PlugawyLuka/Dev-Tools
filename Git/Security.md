# GitHub Security, Safety & IP Protection — Quick Guide

Purpose
- Short practical checklist to protect accounts, repositories, secrets and intellectual property (IP) when using GitHub and related tooling.

## Account & device hygiene

- Enable 2FA (TOTP or hardware security key) on your GitHub account.
- Use a unique, strong password and a password manager.
- Create one SSH key per device and name keys clearly in GitHub (e.g., "pi3b+ — livingroom — 2025-09-25").
- Revoke SSH keys or PATs when a device is retired.

Commands:
```bash
# generate an ed25519 SSH key (per device)
ssh-keygen -t ed25519 -C "device-name@location"

# show the public key to copy to GitHub
cat ~/.ssh/id_ed25519.pub

# add key to ssh-agent (optional if you set a passphrase)
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/id_ed25519
```

## Credentials & secrets

- Never commit secrets (keys, tokens) to repo history.
- Use GitHub Secrets for Actions; use environment secrets or a vault (HashiCorp/KeyVault) for production.
- Use pre-commit hooks or scanning to detect accidental secrets (git-secrets, truffleHog).
- Use repository secret scanning (GitHub Advanced Security) and rotate leaked credentials immediately.

Commands:
```bash
# set a repository secret using GitHub CLI
echo "supersecretvalue" | gh secret set MY_SECRET -R owner/repo

# scan local repo with git-secrets (example)
git secrets --scan

# basic truffleHog scan against a remote repo
trufflehog git https://github.com/owner/repo.git
```

## Least privilege & credentials lifecycle

- Use least-privilege tokens and deploy keys scoped to single repos when possible.
- Prefer deploy keys for CI/machines restricted to one repo (read-only if possible).
- Use machine/service accounts with minimal scopes for automation; keep audit logs.

Commands (deploy key via CLI/API):
```bash
# add a deploy key (read-only) via gh api (example)
gh api repos/OWNER/REPO/keys --field title="deploy-pi" --field key="$(cat ~/.ssh/id_ed25519_pi.pub)" --raw-field read_only=true
```

## Repository visibility & access control

- Choose private repos for proprietary work; public only if you intend to share.
- Organize people in an Organization and use Teams for role-based access.
- Built-in GitHub roles: owner/admin, maintain, write, triage, read. Apply least privilege.
- For sensitive repos, restrict who can create branches, merge, or push directly.

Commands:
```bash
# list collaborators for quick audit
gh api repos/OWNER/REPO/collaborators

# add collaborator (web UI recommended) - example CLI invite:
gh api repos/OWNER/REPO/collaborators/USERNAME -X PUT -f permission="push"
```

## Branch protection & code review

- Protect default/main branches: require pull requests, require status checks.
- Require at least one or two reviewers before merge; use CODEOWNERS for automatic reviewer assignment.
- Require passing CI and security scans before merge.

Commands:
```bash
# view branch protection via API (example)
gh api repos/OWNER/REPO/branches/main/protection

# enable required checks/approvals typically via web UI or templates; use API for automation
```

## Commit & release provenance

- Sign commits and tags with GPG or SSH to strengthen provenance.
- Use signed releases and publish checksums for binary artifacts.

Commands:
```bash
# generate a GPG key (interactive)
gpg --full-generate-key

# list key IDs
gpg --list-secret-keys --keyid-format=long

# configure git to sign commits
git config --global user.signingkey <KEYID>
git config --global commit.gpgSign true

# create a signed tag
git tag -s v1.0 -m "Signed release"
git push --tags
```

## IP & legal protections

- Add a clear LICENSE file in each repo. Choose license that matches your intent.
- Add COPYRIGHT header or NOTICE files where appropriate.
- For commercial projects, use contributor agreements (CLA/DCO) to ensure contributions are assignable/licensable.
- Keep provenance (original source, authorship) in commit history.

Commands:
```bash
# add LICENSE file quickly (example: MIT)
curl -s https://raw.githubusercontent.com/licenses/license-templates/master/templates/mit.txt > LICENSE
git add LICENSE && git commit -m "Add MIT license"
```

## Security tooling & automation

- Enable Dependabot for dependency updates and alerts.
- Enable GitHub Code Scanning (CodeQL) and secret scanning where available.

Commands:
```bash
# enable Dependabot alerts or check configuration via CLI (requires organization permissions)
gh api orgs/ORG/dependabot/alerts

# run local security scans or integrate tools in CI
```

## Backups, auditing & incident response

- Regularly back up important repos (mirrors using `git clone --mirror`).
- Enable audit logs for Organizations (paid plans).
- Prepare an incident response checklist: revoke keys/PATs, rotate secrets, contact affected services, restore from backups.

Commands:
```bash
# mirror-clone a repo for backup
git clone --mirror git@github.com:owner/repo.git

# revoke local auths quickly
gh auth logout
ssh-add -D
```

## Policies & documentation

- Add SECURITY.md describing vulnerability disclosure process and contact.
- Add CONTRIBUTING.md with contributor expectations and licensing terms.
- Add README, LICENSE, and optionally a short IP/ownership section.

Short checklist (minimum)
- [ ] 2FA enabled
- [ ] Keys/PATs named and audited
- [ ] Repos private for proprietary code
- [ ] Branch protection + required PR reviews
- [ ] Secrets stored outside repo
- [ ] LICENSE and SECURITY.md present
- [ ] Dependabot / code scanning enabled

Resources
- GitHub Docs: Licensing, Branch protection, Secrets, Dependabot
- Choose a License: https://choosealicense.com

## Useful commands (security & secrets)

### SSH keys & testing
```bash
# generate an ed25519 key
ssh-keygen -t ed25519 -C "device-name@location"

# display public key to copy to GitHub
cat ~/.ssh/id_ed25519.pub

# test GitHub SSH auth
ssh -T git@github.com
```

### GitHub CLI (manage keys, auth, secrets)
```bash
# authenticate (interactive)
gh auth login

# add SSH key to account
gh ssh-key add ~/.ssh/id_ed25519.pub --title "Pi3B+ — LivingRoom"

# logout / revoke local auth
gh auth logout

# list repository secrets (requires repo context or -R owner/repo)
gh secret list -R owner/repo

# set a repo secret (reads value from stdin)
echo "supersecret" | gh secret set MY_SECRET -R owner/repo
```

### Secrets scanning / detection (local checks)
```bash
# scan staged/committed content for common secrets (git-secrets or truffleHog)
# git-secrets: https://github.com/awslabs/git-secrets
git secrets --scan

# truffleHog (python tool)
trufflehog git https://github.com/owner/repo.git
```

### GPG commit signing (provenance)
```bash
# generate a GPG key (interactive)
gpg --full-generate-key

# list key IDs
gpg --list-secret-keys --keyid-format=long

# set Git to sign commits by default
git config --global user.signingkey <KEYID>
git config --global commit.gpgSign true

# create signed tag
git tag -s v1.0 -m "Signed release"
```

### Backups & mirrors
```bash
# mirror-clone a repo for offsite backup
git clone --mirror git@github.com:owner/repo.git
# push mirror to backup remote
cd repo.git
git remote add backup ssh://backup-host/path/repo.git
git push --mirror backup
```

### Incident & credential revocation (quick)
```bash
# unauthenticate local gh and ssh-agent
gh auth logout
ssh-add -D          # remove all keys from agent
# Revoke PATs or SSH keys via web UI or Org audit (recommended)
```

## Removing leaked keys/secrets from repository history (step-by-step)

Important: If a private key or secret is committed, first assume it's compromised. Revoke credentials immediately (GitHub keys/PATs/secrets) before changing repository history. Rewriting history is disruptive — coordinate with collaborators.

1) Emergency revoke (do this first)
```bash
# List your SSH keys on GitHub (gets id and title)
gh api user/keys

# Delete a compromised key by ID
gh api user/keys/KEY_ID -X DELETE

# Revoke a PAT via web UI or use gh/web to delete tokens
gh auth logout   # unauth local session
```

2) Backup the repo (mirror) before any rewrite
```bash
git clone --mirror git@github.com:owner/repo.git repo.git
cd repo.git
# keep this mirror copy safe; you can restore if needed
```

3) Remove leaked files or paths (recommended: git-filter-repo)
- Install git-filter-repo (preferred over filter-branch)
- Examples:

Remove a specific file (e.g., id_ed25519) from all history:
```bash
git filter-repo --invert-paths --paths id_ed25519 --force
```

Replace sensitive strings with a placeholder using a replacements file:
Create replacements.txt:
```text
# format: exact_literal==>replacement
-----BEGIN OPENSSH PRIVATE KEY----- ==> [REDACTED_PRIVATE_KEY]
my-very-secret-token ==> [REDACTED_TOKEN]
```
Run:
```bash
git filter-repo --replace-text ../replacements.txt --force
```

4) Alternative: Use BFG Repo-Cleaner (simpler for many cases)
```bash
# work on mirror clone
# download bfg.jar (https://rtyley.github.io/bfg-repo-cleaner/)
java -jar bfg.jar --delete-files id_ed25519 repo.git
# or replace text
java -jar bfg.jar --replace-text ../replacements.txt repo.git
cd repo.git
git reflog expire --expire=now --all
git gc --prune=now --aggressive
```

5) Verify the leak is gone
```bash
# search for the leaked string or filename
git grep -n "BEGIN OPENSSH PRIVATE KEY" --all
# or use local secret scanners
trufflehog filesystem repo.git
git secrets --scan
```

6) Force-push cleaned history (coordinate with team)
```bash
# push cleaned mirror back to origin (this rewrites all history)
git push --force --mirror origin
```
Safety note: After a forced rewrite, anyone who cloned the repo must reclone (not pull) or follow careful rebase instructions. Announce the change and provide migration steps to collaborators.

7) Post-cleanup actions (must do)
- Rotate all revoked credentials (create new SSH keys / PATs / deploy keys).
- Update any CI/CD secrets and service tokens.
- Invalidate cached artifacts that included secrets (containers, build artifacts).
- Notify users and document the incident in internal diary/SECURITY.md.
- Run secret scanning / CodeQL across the repo to confirm.

8) If you accidentally committed a key and pushed a public repo
- Revoke immediately (see step 1).
- Consider legal/contract steps if secrets exposed production systems.
- Preserve forensic copies (the mirror) for post-incident review before wiping.

9) Helpful commands to finalize local cleanup
```bash
# after rewriting, expire reflogs and run garbage collection
git reflog expire --expire=now --all
git gc --prune=now --aggressive

# confirm no dangling objects reference the secret
git fsck --full
```

Notes and cautions
- Never attempt history rewrite without a verified backup.
- Use git-filter-repo or BFG rather than git filter-branch for speed and reliability.
- Coordinate with all contributors — a force-push invalidates existing local clones and PR branches.
- Rewriting history does not remove copies that others already cloned. Treat the secret as leaked and rotate credentials regardless of clean-up success.