# GitHub Guide for Beginners

This guide covers the most important things to know and do with GitHub for beginners. It includes step-by-step tutorials, useful commands, SSH setup, collaboration workflows, recommended tools, tables for quick reference, and alternatives.

## Table of Contents

- Introduction
- Quick setup (Git + GitHub)
- SSH setup (keys & troubleshooting)
- Common Git commands (cheat sheet)
- Collaboration (branches, pull requests, permissions)
- Working from multiple devices (tips & tools)
- CI/CD basics with GitHub Actions
- Security best practices
- Recommended tools & extensions
- Troubleshooting & resources
- Appendix: Useful commands and examples

---

## Introduction

GitHub is a hosted Git service that lets you store, share, and collaborate on code. This guide focuses on practical workflows and commands so you can start contributing to projects and managing your own repositories.

## Quick setup (Git + GitHub)

1. Install Git
   - Windows: https://git-scm.com/download/win
   - macOS: Use Homebrew `brew install git` or download installer
   - Linux: Use your package manager `sudo apt install git` or equivalent

2. Configure Git

```bash
git config --global user.name "Your Name"
git config --global user.email "you@example.com"
# Use your preferred editor
git config --global core.editor "code --wait"
```

3. Generate SSH key (recommended)

```bash
ssh-keygen -t ed25519 -C "your_email@example.com"
# Or RSA if ed25519 unavailable
ssh-keygen -t rsa -b 4096 -C "your_email@example.com"
```

4. Add SSH key to GitHub

- Show the public key (Linux/macOS):
```bash
cat ~/.ssh/id_ed25519.pub
```

- Show the public key (Windows PowerShell):
```powershell
Get-Content $env:USERPROFILE\.ssh\id_ed25519.pub
```

- Add via GitHub web UI:
  - Open GitHub → Settings → SSH and GPG keys → New SSH key
  - Give a clear title (e.g., "Raspberry Pi 3B+ — LivingRoom — 2025-09-25") and paste the public key.

- Add from the device (command-line) — preferred for headless devices:

  1) Using GitHub CLI (gh) — easiest and interactive:
  ```bash
  # login once (follow prompts)
  gh auth login

  # add the key
  gh ssh-key add ~/.ssh/id_ed25519.pub --title "Pi3B+ — $(hostname)"
  ```

  2) Using GitHub REST API with a Personal Access Token (non-interactive):
  - Export/set a token with the "admin:public_key" or appropriate scope (keep it secret and session-only).
  ```bash
  export GITHUB_TOKEN="ghp_xxx"    # session-only recommended
  curl -H "Authorization: token $GITHUB_TOKEN" \
       -H "Content-Type: application/json" \
       -d "{\"title\":\"Pi3B+\",\"key\":\"$(sed -e 's/\"/\\\"/g' ~/.ssh/id_ed25519.pub)\"}" \
       https://api.github.com/user/keys
  ```

  PowerShell (REST API) variant (see git-manual/PowerShell.md for more):
  ```powershell
  $token = $env:GITHUB_TOKEN
  $key = Get-Content $env:USERPROFILE\.ssh\id_ed25519.pub -Raw
  $body = @{ title = "Pi3B+"; key = $key } | ConvertTo-Json
  Invoke-RestMethod -Method Post -Uri "https://api.github.com/user/keys" `
    -Headers @{ Authorization = "token $token"; "User-Agent" = "git-manual" } `
    -Body $body -ContentType "application/json"
  ```

- Test SSH connection:
```bash
ssh -T git@github.com
# verbose debug if needed:
ssh -vT git@github.com
```

Notes & security
- Prefer `gh` for convenience; prefer web UI if you do not want to expose tokens on the device.
- When using API + PAT, set the token only for the session and remove it after use (do not commit it).
- Name keys clearly so you can revoke a single device later. For Windows/PowerShell equivalents and more snippets see git-manual/PowerShell.md.

Note: PowerShell command examples have been moved to git-manual/PowerShell.md — see that file for Windows/PowerShell equivalents of the commands used in this guide.

## SSH setup (keys & troubleshooting)

- Generate key per device, name them (e.g., `id_ed25519_work`, `id_ed25519_home`).
- Add keys to your GitHub account.
- Use `ssh-agent` to cache passphrases.

Example `ssh-agent` usage (Linux/macOS):

```bash
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/id_ed25519
```

Troubleshooting
- Permission denied (publickey): Ensure the key is added to GitHub and agent.
- Wrong key used: create `~/.ssh/config`:

```
Host github.com
  HostName github.com
  User git
  IdentityFile ~/.ssh/id_ed25519
  IdentitiesOnly yes
```

## Common Git commands (cheat sheet)

| Action | Command |
|---|---|
| Clone repo (SSH) | `git clone git@github.com:owner/repo.git` |
| Clone repo (HTTPS) | `git clone https://github.com/owner/repo.git` |
| Check status | `git status` |
| Stage changes | `git add <file>` |
| Commit | `git commit -m "message"` |
| Push | `git push` |
| Pull | `git pull` |
| Create branch | `git checkout -b feature/name` |
| Switch branch | `git switch <branch>` or `git checkout <branch>` |
| Merge branch | `git merge feature/name` |
| Rebase | `git rebase main` |

## Collaboration (branches, pull requests, permissions)

- Use feature branches for any new work.
- Open a Pull Request (PR) when ready to merge to main.
- Add reviewers and use GitHub's review features.
- For private collaborations: Settings → Collaborators & teams → Add people.

Permissions model
- Collaborator: Full repo access per role
- Teams (org): Granular access controls
- Deploy keys: Read-only or read-write for CI

## Working from multiple devices (tips & tools)

- Add an SSH key per device and name them.
- Use dotfiles repo to sync configs (Zsh, VSCode settings, gitconfig).
- Use cloud dev environments: GitHub Codespaces, Gitpod, Replit.
- Keep a list of your SSH keys in a secure password manager.
- Use `git config --global include.path` to include shared configs.

### Adding a Raspberry Pi 3B+ as a GitHub device (step-by-step)

This procedure assumes Raspberry Pi OS (Debian) and an existing GitHub account. Preferred method: SSH keys. Keep keys named and revocable.

1) Prepare the Pi
- Update packages and install essentials:

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y git curl ca-certificates
```

- (Optional) Install GitHub CLI for convenience:

```bash
curl -fsSL https://cli.github.com/packages/githubcli-archive-keyring.gpg | \
  sudo dd of=/usr/share/keyrings/githubcli-archive-keyring.gpg
sudo chmod go+r /usr/share/keyrings/githubcli-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/githubcli-archive-keyring.gpg] https://cli.github.com/packages stable main" | \
  sudo tee /etc/apt/sources.list.d/github-cli.list > /dev/null
sudo apt update
sudo apt install -y gh
```

2) Create an SSH key for the Pi
- Recommended: ed25519

```bash
ssh-keygen -t ed25519 -C "pi3b+@<location-or-hostname>"
# Use a descriptive filename if you keep multiple keys:
# ~/.ssh/id_ed25519_pi3
```

- Secure permissions:

```bash
chmod 700 ~/.ssh
chmod 600 ~/.ssh/id_ed25519*
```

3) Add the public key to GitHub
- Copy the key:

```bash
cat ~/.ssh/id_ed25519.pub
```

- Add via GitHub web UI: Settings → SSH and GPG keys → New SSH key. Use a clear title (e.g., "Raspberry Pi 3B+ — LivingRoom — 2025-09-25").
- Or use GitHub CLI:

```bash
gh auth login   # one-time
gh ssh-key add ~/.ssh/id_ed25519.pub --title "Raspberry Pi 3B+"
```

4) Configure SSH agent (optional, for passphrases)

```bash
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/id_ed25519
```

To persist across reboots, add `ssh-add` to your shell startup or use key without passphrase (understand risks).

5) Test SSH connection and Git operations

```bash
ssh -T git@github.com
# Expected: a welcome message mentioning your GitHub username

# Test clone/push
git clone git@github.com:youruser/yourrepo.git
cd yourrepo
touch pi-test.txt
git add pi-test.txt
git commit -m "test commit from Pi"
git push origin main
```

6) Alternatives and automation
- Deploy key: repo-scoped key added in repo settings — use for single-repo devices (can be read-only).
- PAT (Personal Access Token): use for non-interactive HTTPS automation; store in environment variables or use `gh auth login --with-token`.
- Machine user: separate GitHub account for CI/devices if you need isolated credentials.

7) Security & housekeeping
- Name keys clearly and record where each device is.
- Revoke keys when retiring devices: GitHub → Settings → SSH and GPG keys → Delete.
- Don’t commit private keys. Back them up securely (offline/encrypted) if needed.

- IMPORTANT: If a key or secret is ever exposed, follow the full removal & incident steps in git-manual/github-security.md — see the section "Removing leaked keys/secrets from repository history (step-by-step)" for immediate actions, revocation commands, and safe history rewrite instructions.

8) Troubleshooting
- Permission denied (publickey): run `ssh -vT git@github.com` to inspect offered keys.
- Wrong key used: add `IdentitiesOnly yes` and explicit `IdentityFile` to `~/.ssh/config`:

```
Host github.com
  HostName github.com
  User git
  IdentityFile ~/.ssh/id_ed25519
  IdentitiesOnly yes
```

- Check `~/.ssh` permissions and ensure uploaded public key matches local key.
- If multiple keys interfere, use `ssh-add` to control which is offered.

9) Quick checklist before deploying the Pi
- [ ] OS updated and git installed
- [ ] SSH key generated and public key added to GitHub
- [ ] SSH test successful (`ssh -T git@github.com`)
- [ ] Git user configured (`git config --global user.name/email`)
- [ ] Backup private key (securely) or document revocation plan
- [ ] (Optional) Deploy key or PAT created for automation with minimal scope

## CI/CD basics with GitHub Actions

- Create `.github/workflows/ci.yml`.
- Example Node.js CI workflow (basic):

```yaml
name: CI
on: [push, pull_request]
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Set up Node
        uses: actions/setup-node@v3
        with:
          node-version: '18'
      - run: npm ci
      - run: npm test
```

## Security best practices

- Enable 2FA on GitHub
- Use minimal permissions for tokens
- Use limited-scope Personal Access Tokens (PATs)
- Rotate keys and tokens regularly
- Use Dependabot for dependency updates

## Recommended tools & extensions

- VS Code + GitLens
- GitHub CLI (`gh`)
- GitKraken, Sourcetree (GUI clients)
- Docker for reproducible environments
- Terraform/Ansible for infra as code

## Troubleshooting & resources

- `git help <command>`
- GitHub Docs: https://docs.github.com
- Pro Git book: https://git-scm.com/book/en/v2

## Appendix: Useful commands and examples

- Create a repo and push local project

```bash
git init
git add .
git commit -m "initial commit"
# create repo on GitHub via UI or CLI
gh repo create my-repo --public --source=. --remote=origin
git push -u origin main
```

- Forking and contributing
  - Fork on GitHub
  - Clone your fork
  - Add upstream remote: `git remote add upstream git@github.com:original/repo.git`
  - Keep fork updated: `git fetch upstream; git merge upstream/main`

---

End of guide.

## Installing software on Debian-based Linux (apt, dpkg, aptitude, Synaptic, GNOME Software)

When using Raspberry Pi OS, Ubuntu or similar Debian-based systems you have several ways to install programs. Each tool fits different needs — GUI, scripting, low-level .deb install, or interactive resolution of dependency problems.

Why multiple tools?
- apt / apt-get: high-level package manager for most day-to-day installs and upgrades. Handles dependencies and repositories.
- dpkg: low-level tool that installs a single .deb file but does not resolve missing dependencies.
- aptitude: curses-based UI and alternative resolver; useful if apt cannot resolve dependency conflicts.
- Synaptic / GNOME Software: graphical front-ends for browsing and installing packages.
- snap / flatpak: sandboxed packaging formats with cross-distro apps.

Practical examples & common commands (use on Pi/Ubuntu)

apt (recommended for most tasks)
```bash
# update package lists, upgrade packages
sudo apt update
sudo apt upgrade -y

# install, remove, purge
sudo apt install -y git build-essential python3-pip
sudo apt remove --purge -y package-name

# search, show package info, list upgradable
apt search <term>
apt show <package>
apt list --upgradable

# simulate before running
sudo apt -s install package-name   # -s = simulate
```

dpkg (use when you downloaded a .deb)
```bash
# install local .deb (may require fixing deps after)
sudo dpkg -i package-file.deb
# fix missing dependencies
sudo apt -f install

# list installed packages or files provided by a package
dpkg -l | grep package
dpkg -L package-name
```

aptitude (interactive resolver / alternative)
```bash
# install (may offer different dependency options)
sudo apt install -y aptitude
sudo aptitude install package-name

# run interactive text UI
sudo aptitude
```

Synaptic (GUI package manager)
```bash
# install synaptic and run it (GUI)
sudo apt install -y synaptic
sudo synaptic    # or find "Synaptic Package Manager" in the menu
```
Use Synaptic to search, inspect package versions and pin/hold packages visually.

GNOME Software / Software Center (graphical app store)
```bash
# run GNOME Software (if desktop installed)
gnome-software
# or on Debian/Ubuntu variants:
software-center    # (varies by distro)
```
Useful for snap/flatpak apps and user-friendly installs.

snap & flatpak (sandboxed cross-distro apps)
```bash
# snap (install core first on some distros)
sudo apt install snapd
sudo snap install hello-world

# flatpak (install and add Flathub)
sudo apt install flatpak
flatpak remote-add --if-not-exists flathub https://flathub.org/repo/flathub.flatpakrepo
flatpak install flathub org.gimp.GIMP
```

Helpful tips & safety
- Always run `sudo apt update` before installing to get latest package lists.
- Prefer official distro repos. For third-party repos add their signing key and apt source (inspect key before trusting).
- Use `apt -s` to simulate changes before applying.
- To hold a package version: `sudo apt-mark hold package-name` (release with `apt-mark unhold`).
- If apt is locked or stuck: `sudo killall apt apt-get` then `sudo dpkg --configure -a` and `sudo apt -f install`.
- To check which package provides a command: `apt-file search /usr/bin/command` (install apt-file and run `apt-file update` first).
- Avoid running random scripts from the internet with sudo; inspect them first.
- For reproducible installs on servers/PI, script `apt update && apt install -y ...` and use `--no-install-recommends` if you want minimal installs.

Recommended apps to try on Pi
- git, gh (GitHub CLI), build-essential, python3-pip, docker.io (if supported), code (VS Code / code-oss), vlc, gimp, libreoffice.

When to use which tool
- Use apt for normal installs and upgrades.
- Use dpkg only for local .deb files; follow with `sudo apt -f install`.
- Use aptitude if apt cannot resolve dependency conflicts or you want an interactive resolver.
- Use Synaptic/GNOME Software for GUI-driven management or when you prefer visuals.
- Use snap/flatpak for sandboxed releases not available in distro repos.
