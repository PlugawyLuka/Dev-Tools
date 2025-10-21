# PowerShell Guide — Git & GitHub command equivalents

This file collects PowerShell equivalents for commands used in the GitHub Guide for Beginners (git-manual/github-guide.md). Use these on Windows PowerShell / PowerShell Core.

## Show public SSH key
```powershell
Get-Content $env:USERPROFILE\.ssh\id_ed25519.pub
```

## GitHub CLI (gh) — add SSH key
```powershell
# login (interactive)
gh auth login

# add SSH key
gh ssh-key add $env:USERPROFILE\.ssh\id_ed25519.pub --title "Pi3B+ — $env:COMPUTERNAME"
```

## REST API add SSH key (PowerShell)
```powershell
# set token in environment for the session
$env:GITHUB_TOKEN = "ghp_xxx"    # replace with token; prefer session-only

# prepare body
$key = Get-Content $env:USERPROFILE\.ssh\id_ed25519.pub -Raw
$body = @{ title = "Pi3B+"; key = $key } | ConvertTo-Json

# post
Invoke-RestMethod -Method Post -Uri "https://api.github.com/user/keys" `
  -Headers @{ Authorization = "token $env:GITHUB_TOKEN"; "User-Agent" = "git-manual" } `
  -Body $body -ContentType "application/json"
```

## Start ssh-agent and add key (PowerShell)
```powershell
# Start OpenSSH agent service (Windows)
Start-Service ssh-agent
# add key to agent
ssh-add $env:USERPROFILE\.ssh\id_ed25519
```

## Test SSH connection
```powershell
ssh -T git@github.com
# verbose
ssh -vT git@github.com
```

## Git basics (PowerShell examples)
```powershell
# clone repo (SSH)
git clone git@github.com:owner/repo.git

# create branch
git checkout -b feature/name

# stage and commit
git add .
git commit -m "message"

# push branch
git push -u origin feature/name
```

## GPG commit signing (PowerShell / Windows)
```powershell
# list secret keys (GPG must be installed)
gpg --list-secret-keys --keyid-format=long

# configure git to sign commits
git config --global user.signingkey <KEYID>
git config --global commit.gpgSign true
```

## GitHub CLI common tasks (PowerShell)
```powershell
# create a repo
gh repo create my-repo --public --source=. --remote=origin

# create a PR
gh pr create --base main --head feature/name --title "My PR" --body "Description"

# set a repo secret
"secret-value" | gh secret set ACTIONS_TOKEN -R owner/repo
```

## Misc admin / API tasks (PowerShell)
```powershell
# list user keys
gh api user/keys

# delete key by id
gh api user/keys/KEY_ID -X DELETE

# add deploy key via API
$deployKey = Get-Content $env:USERPROFILE\.ssh\id_ed25519_deploy.pub -Raw
$body = @{ title = "deploy-pi"; key = $deployKey; read_only = $true } | ConvertTo-Json
gh api repos/OWNER/REPO/keys -f body="$body"
```

## Additional PowerShell tips & commands (semi-advanced)

These snippets complement the SSH and basic git examples — useful on Windows when managing multiple devices, automation, backups and secrets.

### Copy public key to clipboard (convenient for web UI)
```powershell
Get-Content $env:USERPROFILE\.ssh\id_ed25519.pub | Set-Clipboard
```

### Test network / connectivity to GitHub
```powershell
# TCP port test for SSH
Test-NetConnection -ComputerName github.com -Port 22

# HTTP(S) test for API/web
Test-NetConnection -ComputerName github.com -Port 443
```

### Git Credential Manager (store HTTPS credentials securely)
```powershell
# ensure credential helper set
git config --global credential.helper manager-core

# check current helper
git config --global credential.helper
```

### PowerShell Secret Management (store tokens securely)
```powershell
# install modules (one-time)
Install-Module -Name Microsoft.PowerShell.SecretManagement,Microsoft.PowerShell.SecretStore -Scope CurrentUser

# register default local vault (follow prompts)
Register-SecretVault -Name SecretStore -ModuleName Microsoft.PowerShell.SecretStore -DefaultVault

# set a secret (interactive prompt)
Set-Secret -Name GITHUB_TOKEN
# read a secret
$token = Get-Secret -Name GITHUB_TOKEN
```
Use Get-Secret value when calling gh or Invoke-RestMethod instead of plain env vars.

### Persist environment variables (user-level) — use with caution
```powershell
[Environment]::SetEnvironmentVariable('GITHUB_TOKEN','ghp_xxx','User')
# remove
[Environment]::SetEnvironmentVariable('GITHUB_TOKEN',$null,'User')
```

### Git LFS (large files)
```powershell
# install (if Git LFS available)
git lfs install

# track file types
git lfs track "*.bin"
git add .gitattributes
git commit -m "Track large binary files with LFS"
```

### Working with WSL and shared SSH keys
```powershell
# copy Windows key into WSL (example)
wsl -- bash -lc "mkdir -p ~/.ssh && cat > ~/.ssh/id_ed25519" < $env:USERPROFILE\.ssh\id_ed25519
wsl -- bash -lc "chmod 600 ~/.ssh/id_ed25519"
```
Alternatively use the same key from Windows by mounting the Windows .ssh folder.

### Scheduled backups (example script + scheduled task)
```powershell
# simple mirror backup script (save as C:\scripts\backup-repos.ps1)
$backupRoot = "C:\backups\github"
$timestamp = (Get-Date).ToString("yyyyMMdd-HHmm")
$repos = @("git@github.com:owner/repo.git")
foreach ($r in $repos) {
  $name = ($r.Split('/')[-1] -replace '\.git$','')
  $dest = Join-Path $backupRoot "$name-$timestamp.git"
  git clone --mirror $r $dest
  Compress-Archive -Path $dest -DestinationPath "$dest.zip"
  Remove-Item -Recurse -Force $dest
}

# register scheduled task to run daily at 03:00
$action = New-ScheduledTaskAction -Execute "pwsh.exe" -Argument "-File C:\scripts\backup-repos.ps1"
$trigger = New-ScheduledTaskTrigger -Daily -At 3am
Register-ScheduledTask -TaskName "GitHubRepoBackup" -Action $action -Trigger $trigger -RunLevel Highest
```

### Archive & compress repo snapshots
```powershell
Compress-Archive -Path "C:\backups\repo.git" -DestinationPath "C:\backups\repo.zip"
```

### Automating deploy-key / secret uploads using gh + secret store
```powershell
# read token from SecretStore and export for gh
$token = Get-Secret -Name GITHUB_TOKEN
$env:GITHUB_TOKEN = $token
# add deploy key via gh api (example)
$pub = Get-Content $env:USERPROFILE\.ssh\id_ed25519_deploy.pub -Raw
gh api repos/OWNER/REPO/keys --field title="deploy-pi" --field key="$pub" --raw-field read_only=true
# clear env var after use
Remove-Item Env:GITHUB_TOKEN
```

### Install convenient dev tooling (chocolatey / winget)
```powershell
# Chocolatey (admin)
Set-ExecutionPolicy Bypass -Scope Process -Force
irm https://community.chocolatey.org/install.ps1 | iex

# install gh, git, posh-git, oh-my-posh
choco install git gh poshgit oh-my-posh -y
```
Or use winget on modern Windows: winget install --id GitHub.cli

### Improve shell experience (posh-git, oh-my-posh)
```powershell
# install modules for prompt and tab completion
Install-Module posh-git -Scope CurrentUser
Install-Module oh-my-posh -Scope CurrentUser

# enable in profile (~\Documents\PowerShell\Microsoft.PowerShell_profile.ps1)
Import-Module posh-git
Import-Module oh-my-posh
Set-PoshPrompt -Theme Paradox
```

### GPG on Windows (commit signing)
```powershell
# list GPG keys (Gpg4win or gpg installed)
gpg --list-secret-keys --keyid-format=long
# configure git signing key
git config --global user.signingkey <KEYID>
git config --global commit.gpgSign true
```

### File permissions and secure storage
```powershell
# restrict folder permissions (example, only current user)
$path = "$env:USERPROFILE\.ssh"
icacls $path /inheritance:r
icacls $path /grant:r "$env:USERNAME:(F)"
```

### Troubleshooting hints
```powershell
# verbose SSH debug
ssh -vT git@github.com

# view gh auth status
gh auth status

# view git config
git config --list --show-origin
```

Notes
- Prefer SecretManagement for automation instead of plain env vars or files.
- Avoid storing long-lived PATs on devices; use deploy keys or least-privilege tokens.
- Test scheduled tasks and backup scripts manually before enabling automation.