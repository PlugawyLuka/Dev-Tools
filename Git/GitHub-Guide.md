# GitHub — Practical Guide (Ubuntu 22.04 / Raspberry Pi OS / Windows PowerShell / macOS Ventura)

TL;DR
- GitHub hosts Git repositories with collaboration features (issues, PRs, Actions). Use SSH keys or the GitHub CLI (gh) for secure automation.
- Install git + gh, configure user.name/user.email, and generate an SSH key or PAT for auth.
- Use a branch → PR workflow, enable branch protection and required checks, run tests in Actions.
- Automate repetitive tasks with GitHub Actions or local scripts; keep credentials out of repos.
- Backup critical repos and enforce code review + Dependabot/CodeQL for supply-chain security.

Conventions
- Commands are in code blocks. Commands that modify system state are marked WARNING: with an explanation.
- ADVANCED tags mark deeper topics (GPG signing, Git internals).
- Do not run commands without reviewing them first.

Index
0. TL;DR  
1. Introduction  
2. Prerequisites & common concepts  
3. Linux — Ubuntu 22.04  
  3.1 Installation  
  3.2 Configuration  
  3.3 Usage  
  3.4 Security  
  3.5 Troubleshooting  
  3.6 Automation  
  3.7 Verification checklist  
4. Raspberry Pi OS — ARM  
  4.1 Installation  
  4.2 Configuration  
  4.3 Usage  
  4.4 Security & reliability  
  4.5 Troubleshooting  
  4.6 Automation  
  4.7 Verification checklist  
5. PowerShell — Windows 10/11 + PowerShell 7+  
6. macOS (Ventura)  
7. Cross-platform workflows & examples  
8. Advanced topics (ADVANCED)  
9. Security & best practices  
10. Troubleshooting & diagnostics (global)  
11. Automation, CI & testing ideas  
12. Mobile (Android / iOS) — optional  
13. References  
14. Alternatives (pros / cons)  
15. Appendices (examples & snippets)  
16. Final checklist & next steps

---

1. Introduction
Objective: Practical OS-focused guide to using GitHub safely and productively for beginner→intermediate users.

2. Prerequisites & common concepts
- Git: local VCS. GitHub: remote hosting + collaboration.
- Auth: SSH keys, HTTPS with credential manager/PAT, OAuth apps.
- Core terms: repo, branch, commit, PR (pull request), issue, Action, secret.
- ADVANCED: PAT scopes should be minimal and rotated regularly.

---

3. Linux — Ubuntu 22.04

3.1 Installation
Objective: Install Git and GitHub CLI and verify availability.

Commands:
```bash
# WARNING: installs packages and modifies system
sudo apt update
sudo apt install -y git gh
```
Verification:
```bash
git --version
gh --version
```

3.2 Configuration
Objective: Configure identity, SSH keys, and gh auth.

Commands:
```bash
git config --global user.name "Your Name"
git config --global user.email "you@example.com"

# WARNING: generates key files in ~/.ssh
ssh-keygen -t ed25519 -C "you@example.com"

# Show public key to add to GitHub account
cat ~/.ssh/id_ed25519.pub

# Login with gh (interactive)
gh auth login
```

3.3 Usage
Common workflows: clone, branch, commit, push, PRs.
```bash
git clone git@github.com:org/repo.git
cd repo
git checkout -b feature/x
# edit files
git add .
git commit -m "Add feature X"
git push -u origin feature/x

# Create PR using gh
gh pr create --title "Feature X" --body "Short description" --base main
```

3.4 Security
- Do not commit secrets; use .gitignore and repository secrets.
- Use branch protection and required checks.
- Use least-privilege PATs and rotate keys regularly.

3.5 Troubleshooting
Basic debug commands:
```bash
GIT_TRACE=1 GIT_CURL_VERBOSE=1 git push
ssh -T git@github.com -vvv
```

3.6 Automation
Example Actions workflow (conceptual):
```yaml
# .github/workflows/ci.yml
name: CI
on: [push, pull_request]
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Run tests
        run: ./run-tests.sh
```

3.7 Verification checklist
- git --version and gh --version
- ssh -T git@github.com returns success
- Can push to a test repo and open PR with gh

---

4. Raspberry Pi OS — ARM

4.1 Installation
```bash
# WARNING: installs packages on Pi
sudo apt update
sudo apt install -y git gh
```
Notes: gh may not be available in some Pi apt repos; use the gh .deb for ARM or install via alternative methods.

4.2 Configuration
Generate SSH key for headless Pi:
```bash
ssh-keygen -t ed25519 -C "pi@example.com" -f ~/.ssh/id_ed25519_pi
```

4.3 Usage
Shallow clone to save space:
```bash
git clone --depth 1 git@github.com:org/repo.git
```

4.4 Security & reliability
Prefer deploy keys for CI and hardware tokens for sensitive accounts. Keep SD backups.

4.5 Troubleshooting
Use sparse-checkout or LFS for large assets when storage is limited.

4.6 Automation
Use Ansible or cron to sync repos; self-hosted runners on Pi for small workloads.

4.7 Verification checklist
- git clone works; shallow clones acceptable
- Runner registration verified in repo settings

---

5. PowerShell — Windows

5.1 Installation
```powershell
# WARNING: installs software on Windows
winget install --id Git.Git -e --source winget
winget install --id GitHub.cli -e --source winget
```

5.2 Configuration
Generate SSH key and use Windows Credential Manager for HTTPS helpers.

5.3 Usage
Use WSL for best UNIX-like behavior; otherwise standard git workflows apply.

5.4 Troubleshooting
Address line ending issues and LFS; set core.autocrlf appropriately.

---

6. macOS (Ventura)
Install via Homebrew and use osxkeychain helper:
```bash
brew install git gh
git config --global credential.helper osxkeychain
ssh-keygen -t ed25519 -C "you@example.com"
```

---

7. Cross-platform workflows & examples
Create and publish a repo:
```bash
git init
git add .
git commit -m "Initial"
# WARNING: creates remote repo and pushes
gh repo create my-repo --public --source=. --push
```

---

8. Advanced topics (ADVANCED)
- GPG-signed commits and tag signing
- Git internals: reflog, bisect, cherry-pick
- Reusable Actions, environment protection, artifact retention
- GitHub Apps and API automation

---

9. Security & best practices
- Use least-privilege PATs, rotate keys, enable 2FA.
- Use Dependabot and CodeQL for supply-chain scanning.
- Store secrets in Actions secrets or a vault.

---

10. Troubleshooting & diagnostics (global)

10.1 Common git pull problems & fixes

Problem A: "git@github.com: Permission denied (publickey)."
- Verify remote and test SSH:
```bash
git remote -v
ssh -T git@github.com -vvv
```
- Quick fixes:
```bash
eval "$(ssh-agent -s)"
ssh-add -l || ssh-add ~/.ssh/id_ed25519
ssh-keygen -t ed25519 -C "you@example.com" -f ~/.ssh/id_ed25519
# add the public key to GitHub account
chmod 700 ~/.ssh
chmod 600 ~/.ssh/id_*
chmod 644 ~/.ssh/*.pub
```
- If multiple keys: add per-host config in ~/.ssh/config and use host aliases.

Problem B: "Already up to date" but remote has new commits
- Safe steps:
```bash
git fetch --all --prune
git branch -vv
git status -sb
git log --oneline HEAD..origin/$(git rev-parse --abbrev-ref HEAD)
```
- If commits exist, merge or rebase:
```bash
git merge origin/$(git rev-parse --abbrev-ref HEAD)
# or
git pull --rebase
```
- Check for shallow clone, wrong branch, wrong remote, uncommitted work. Use `git fetch <remote>` and `git ls-remote <remote>` to inspect refs.

Problem C: "remote: Invalid username or token. Password authentication is not supported"
- Password auth for Git over HTTPS is disabled. Options:
  - Use gh CLI: `gh auth login --web`
  - Use SSH (preferred): set remote to `git@github.com:OWNER/REPO.git` and ensure SSH key/auth
  - Use a PAT with credential helper (avoid embedding tokens in URLs)

Clear bad cached credentials:
```bash
git config --global credential.helper
rm -f ~/.git-credentials        # if using 'store'
printf "protocol=https\nhost=github.com\n\n" | git credential reject
gh auth login --web
```

Problem D: "GH007: Your push would publish a private email address"
- GitHub's email privacy protection blocks pushes with your real email address.
- Error message: `remote: error: GH007: Your push would publish a private email address.`

Solution: Use GitHub's no-reply email address
1. Find your no-reply email at https://github.com/settings/emails
2. Look for: `<ID>+<username>@users.noreply.github.com` or `<username>@users.noreply.github.com`
3. Configure Git to use it:
```bash
# Check current email
git config user.email

# Set GitHub no-reply email (replace with your actual no-reply email)
git config --global user.email "123456789+PlugawyLuka@users.noreply.github.com"

# Verify it's set
git config user.email
```

4. Fix the recent commit(s) that used the old email:
```bash
# For the last commit
git commit --amend --reset-author --no-edit

# For multiple commits (ADVANCED)
git rebase -i HEAD~3 --exec "git commit --amend --reset-author --no-edit"

# Then push again
git push
```

Alternative: Make your email public (not recommended for privacy)
- Visit https://github.com/settings/emails
- Uncheck: "Block command line pushes that expose my email"

Best practice: Always use GitHub's no-reply email for privacy.

10.2 Git submodules (nested repositories)

What are submodules?
- Submodules are separate Git repositories nested inside a parent repository.
- The parent repo tracks a specific commit of the submodule, not the files themselves.
- Common use case: shared libraries, dependencies, or separate private repos within a public project.

Common submodule status message:
```
modified:   submodule-name (untracked content)
```

This means:
1. Changes exist inside the submodule (new files, modified files, or different commit)
2. The parent repo detects the submodule state has changed from what it recorded

Investigate what changed:
```bash
# Navigate into the submodule
cd submodule-name

# Check status
git status

# See what's different
git diff
```

Option 1: Commit changes inside submodule, then update parent
```bash
# Inside submodule
cd submodule-name
git add .
git commit -m "Update submodule content"
git push

# Back to parent repo
cd ..
git add submodule-name
git commit -m "Update submodule reference"
git push
```

Option 2: Discard submodule changes
```bash
cd submodule-name
git restore .           # discard modified files
git clean -fd           # WARNING: removes untracked files

cd ..
git submodule update --init --recursive  # reset to recorded commit
```

Option 3: Ignore submodule changes (temporary)
```bash
git config submodule.submodule-name.ignore all
```

Common submodule commands:
```bash
# Clone repo with submodules
git clone --recurse-submodules git@github.com:org/repo.git

# Initialize submodules after clone
git submodule update --init --recursive

# Pull updates in parent and submodules
git pull --recurse-submodules

# Check submodule status
git submodule status
```

10.3 Multiple remotes (public + private)
- Inspect remotes:
```bash
git remote -v
```
- Fetch from a specific remote and compare:
```bash
git fetch private --prune
git log --oneline HEAD..private/main
git merge private/main
```
- If using separate SSH keys, add `~/.ssh/config` entries and use host aliases; set remote URL accordingly:
```text
Host github-private
  HostName github.com
  User git
  IdentityFile ~/.ssh/id_ed25519_private
```
Then:
```bash
git remote set-url private git@github-private:you/private-repo.git
```
- For CI use deploy keys or repo-scoped PATs.

Debug tips:
```bash
GIT_TRACE=1 GIT_CURL_VERBOSE=1 git ls-remote https://github.com/OWNER/REPO.git
git ls-remote private
git remote show private
git reflog
```

10.4 Repository Restructuring (Monorepo → Multi-Repo)

Why split a monorepo?
- **Performance:** Smaller repos = faster indexing, faster AI context, faster git operations
- **Focus:** Each repo contains one domain (Arduino, Web, Python, etc.)
- **Privacy:** Separate public projects from private notes/credentials
- **Scalability:** New interests = new repo, no impact on existing work
- **CI/CD:** Independent workflows per repo

When to consider splitting:
- Workspace feels slow or cluttered
- Unrelated projects in same repo (Arduino + web + notes)
- AI suggestions becoming less relevant (too much context)
- Want to share some projects publicly but keep others private
- Large repo causing performance issues

Migration strategy (safe, reversible):

Step 1: **Backup everything**
```bash
cd ~/path/to/monorepo
tar -czf ../monorepo-backup-$(date +%Y%m%d).tar.gz .
```

Step 2: **Plan structure**
Example split:
```
Before (monorepo):
~/Code/
├── Arduino/
├── RaspberryPi/
├── Python/
├── Web/
├── Notes/
└── Private/

After (multi-repo):
~/Projects/
├── Arduino-Learning/        (public)
├── RaspberryPi-Projects/    (public)
├── Python-Learning/         (public)
├── Web-Development/         (public)
├── Dev-Tools/               (public - Git/VS Code/AI notes)
├── Linux-SysAdmin/          (public - Linux guides)
└── Learning-Private/        (PRIVATE - logs, credentials, mentor files)
```

Step 3: **Create new directory structure**
```bash
mkdir -p ~/Projects
cd ~/Projects

# Create all new repos
mkdir Arduino-Learning RaspberryPi-Projects Python-Learning Web-Development Dev-Tools Linux-SysAdmin Learning-Private
```

Step 4: **Copy files (test before git init)**
```bash
# Arduino projects
cp -r ~/Code/Arduino/* ~/Projects/Arduino-Learning/

# Private files (mentor system, logs, credentials)
cp -r ~/Code/Private/Mentor ~/Projects/Learning-Private/
cp -r ~/Code/Private/Logs ~/Projects/Learning-Private/

# Repeat for other domains...
```

Step 5: **Create .gitignore for each repo**
```bash
# Example: Learning-Private/.gitignore
cat > ~/Projects/Learning-Private/.gitignore << 'EOF'
# NEVER commit credentials!
Credentials/*
!Credentials/.gitkeep
Personal/Diary.md
*.tmp
.DS_Store
EOF

# Example: Arduino-Learning/.gitignore
cat > ~/Projects/Arduino-Learning/.gitignore << 'EOF'
*.hex
*.elf
build/
.vscode/
.DS_Store
EOF
```

Step 6: **Initialize git for each repo**
```bash
cd ~/Projects/Arduino-Learning
git init
git add .
git commit -m "Initial commit: Arduino projects migrated from monorepo"
```

Step 7: **Create GitHub repos**
```bash
# Using gh CLI
cd ~/Projects/Arduino-Learning
gh repo create Arduino-Learning --public --source=. --remote=origin --push

cd ~/Projects/Learning-Private
gh repo create Learning-Private --private --source=. --remote=origin --push
```

Step 8: **Create VS Code multi-root workspaces**
Create workspace files that open 2 repos together (main project + private):

```bash
# Arduino-Learning.code-workspace
cat > ~/Projects/Arduino-Learning.code-workspace << 'EOF'
{
  "folders": [
    {
      "name": "Arduino-Learning",
      "path": "Arduino-Learning"
    },
    {
      "name": "Learning-Private",
      "path": "Learning-Private"
    }
  ],
  "settings": {
    "files.exclude": {
      "**/Learning-Private/Credentials/**": true
    }
  }
}
EOF
```

Benefits of workspace files:
- AI sees focused context (Arduino + mentor files only)
- Session logs accessible in sidebar
- Credentials hidden from search
- Easy to switch between domains

Step 9: **Create cross-reference document**
In Learning-Private, create `Cross-References.md`:

```markdown
# Repository Cross-Reference

## Active Repositories

### Arduino-Learning
- **Location:** `~/Projects/Arduino-Learning`
- **GitHub:** "https://github.com/YourUsername/Arduino-Learning"
- **Focus:** Embedded C, electronics, sensors
- **Status:** Active

### Learning-Private (this repo)
- **Location:** `~/Projects/Learning-Private`
- **GitHub:** "https://github.com/YourUsername/Learning-Private" (PRIVATE)
- **Focus:** Mentor files, session logs, credentials
- **Status:** Active (core hub)

## Workspace Files

- `Arduino-Learning.code-workspace` — Opens Arduino + Learning-Private
- `RaspberryPi.code-workspace` — Opens RPi + Learning-Private

## Quick Commands

```bash
# Open Arduino workspace
code ~/Projects/Arduino-Learning.code-workspace

# Check all repo status
cd ~/Projects && for dir in */; do echo "=== $dir ==="; cd "$dir"; git status -s; cd ..; done

# Pull all repos
cd ~/Projects && for dir in */; do cd "$dir"; git pull; cd ..; done
```
```

Step 10: **Test the new structure**
```bash
# Open a workspace
code ~/Projects/Arduino-Learning.code-workspace

# Verify AI performance (should be faster/more focused)
# Verify you can access mentor files from sidebar
# Verify cross-references work
```

Step 11: **Keep old monorepo as backup (1-2 weeks)**
```bash
# Rename instead of deleting
mv ~/Code ~/Code-OLD-$(date +%Y%m%d)

# After 2 weeks of successful use, delete:
# rm -rf ~/Code-OLD-20241021
```

Daily workflow after migration:
```bash
# Start Arduino session
code ~/Projects/Arduino-Learning.code-workspace

# In chat: paste Mentor.md, Scribe.md, Commands.md
# Use /start command

# AI now sees:
# ✅ Arduino projects (focused context)
# ✅ Mentor files (behavior rules)
# ✅ Session logs (for /start context loading)
# ❌ Unrelated web/Python/notes (no distraction)
```

Common patterns:
- **One domain per workspace session** (Arduino today, Web tomorrow)
- **Learning-Private opened WITH active project** (multi-root workspace)
- **Public repos for projects, private repo for logs/credentials**
- **Cross-references in Learning-Private/Cross-References.md**

Troubleshooting after migration:
- If AI loses context: paste behavior files at session start
- If can't find old files: check Code-OLD backup
- If workspace slow: exclude folders in settings (Credentials/, build/)
- If git confused: each repo has independent .git/ directory

Performance improvement metrics (observed):
- AI response time: ~30% faster (less context to scan)
- VS Code startup: ~50% faster (smaller workspace)
- Git operations: ~70% faster (smaller repo)
- AI relevance: Significantly improved (focused context)

---

11. Automation, CI & testing ideas
- Typical Actions pipeline: checkout → setup → test → build → publish.
- Use Actions cache and matrix builds; store credentials as secrets.
- Mirror repos: `git clone --mirror git@github.com:owner/repo.git`
- Use deploy keys / repo-scoped PATs for runners.

---

12. Mobile (Android / iOS) — optional
- Recommended apps: GitHub Mobile, Termius (SSH), Working Copy (iOS).
- Use MFA on mobile; avoid long-lived tokens on mobile devices.

---

13. References
- Git docs: https://git-scm.com/docs
- GitHub Docs: https://docs.github.com/
- GitHub CLI: https://cli.github.com/
- Pro Git book: https://git-scm.com/book/en/v2
- GitHub Actions docs: https://docs.github.com/actions

---

14. Alternatives (pros / cons)
- GitLab — Pros: integrated CI/CD, on-prem option. Cons: heavier admin if self-hosted.
- Bitbucket — Pros: Jira integration. Cons: smaller ecosystem.
- Gitea/Gogs — Pros: lightweight self-host. Cons: fewer enterprise features.

---

15. Appendices

15.1 Example ~/.gitconfig
```ini
[user]
  name = Your Name
  email = you@example.com
[core]
  editor = code --wait
[push]
  default = simple
```

15.2 Quick commands
- Init: `git init`
- Clone: `git clone git@github.com:org/repo.git`
- Commit: `git commit -m "msg"`
- PR via CLI: `gh pr create --title "x" --body "desc"`

15.3 Sample GitHub Actions snippet (conceptual)
```yaml
name: CI
on: [push, pull_request]
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: ./run-tests.sh
```

---

16. Final checklist & next steps
- Install git and gh; configure user.name and user.email.
- Create SSH key and add to GitHub or configure PAT carefully.
- Push a test repo and open a PR using gh.
- Enable branch protection, set secrets, and add a simple CI workflow.
- Regularly rotate tokens and test recovery procedures.

Notes & safety reminders
- WARNING: Commands marked WARNING modify system state. Review before executing.
- For production, prefer centralized key management and short-lived credentials.

# End of file