# Environment Variables & Private Repository Management — Complete Guide

Purpose
- Complete tutorial for managing sensitive data using environment variables and private repositories, based on practical session experience with Luka workspace setup.

## Table of Contents
1. [Understanding the Problem](#understanding-the-problem)
2. [Environment Variables Fundamentals](#environment-variables-fundamentals)
3. [Private Repository Setup](#private-repository-setup)
4. [File Migration Strategy](#file-migration-strategy)
5. [Advanced Security Practices](#advanced-security-practices)
6. [Troubleshooting Common Issues](#troubleshooting-common-issues)

## Understanding the Problem

**Default GitHub behavior:**
- Public repositories are readable by anyone worldwide
- All files, commit history, and repository structure are visible
- Search engines can index your repository content
- Accidental secret commits are permanent in Git history

**What happens when secrets leak:**
- API keys can be scraped by bots within minutes
- Database credentials lead to data breaches
- SSH keys allow unauthorized access to your systems
- Router passwords expose your entire network

**Real-world impact:**
- GitHub automatically scans for common secret patterns and alerts you
- Cloud providers monitor for leaked API keys and may suspend accounts
- Malicious actors use automated tools to find and exploit leaked credentials

## Environment Variables Fundamentals

### What are environment variables?
Environment variables are key-value pairs stored in your operating system that applications can read at runtime. They exist outside your code and are not committed to version control.

### Basic PowerShell commands:
```powershell
# Set permanent environment variable (survives reboot)
setx VARIABLE_NAME "value"

# Set temporary environment variable (current session only)
$env:VARIABLE_NAME = "value"

# Read environment variable
echo $env:VARIABLE_NAME
# or
$env:VARIABLE_NAME

# List all environment variables
Get-ChildItem Env:

# List variables matching pattern
Get-ChildItem Env: | Where-Object Name -like "*API*"

# Remove environment variable (current session)
Remove-Item Env:VARIABLE_NAME

# Remove permanent environment variable
[Environment]::SetEnvironmentVariable("VARIABLE_NAME", $null, "User")
```

### Advanced PowerShell techniques:
```powershell
# Set multiple variables from a file
Get-Content .env | ForEach-Object {
    if ($_ -match '^([^=]+)=(.*)$') {
        [Environment]::SetEnvironmentVariable($matches[1], $matches[2], "User")
    }
}

# Export current environment to file (for backup)
Get-ChildItem Env: | Where-Object Name -like "LAB_*" | 
    ForEach-Object { "$($_.Name)=$($_.Value)" } | 
    Out-File -FilePath "env-backup.txt"

# Check if variable exists before using
if ($env:API_KEY) {
    Write-Host "API key is configured"
} else {
    Write-Host "Warning: API_KEY not set"
}
```

### Using environment variables in different languages:

**Python:**
```python
import os

# Get environment variable with default fallback
api_key = os.getenv('OPENAI_API_KEY', 'default-key')

# Require environment variable (fail if missing)
api_key = os.environ['OPENAI_API_KEY']  # Raises KeyError if missing

# Check if variable exists
if 'DATABASE_URL' in os.environ:
    db_url = os.environ['DATABASE_URL']

# Load from .env file (requires python-dotenv)
from dotenv import load_dotenv
load_dotenv()  # Loads .env file into environment
```

**Node.js:**
```javascript
// Access environment variables
const apiKey = process.env.OPENAI_API_KEY;
const dbUrl = process.env.DATABASE_URL || 'localhost:5432';

// Require environment variable
if (!process.env.REQUIRED_VAR) {
    throw new Error('REQUIRED_VAR environment variable is not set');
}
```

**PowerShell scripts:**
```powershell
# Read environment variables in PowerShell scripts
$apiKey = $env:OPENAI_API_KEY
if (-not $apiKey) {
    Write-Error "OPENAI_API_KEY environment variable not set"
    exit 1
}

# Use in REST API calls
$headers = @{
    'Authorization' = "Bearer $env:OPENAI_API_KEY"
    'Content-Type' = 'application/json'
}
```

## Private Repository Setup

### Step-by-step private repository creation:

**1. Create repository on GitHub:**
```powershell
# Using GitHub CLI (if installed)
gh repo create Luka-Private --private --description "Private configurations and secrets"

# Or create manually on GitHub.com:
# 1. Go to github.com/new
# 2. Enter repository name: Luka-Private
# 3. Select "Private"
# 4. Click "Create repository"
```

**2. Clone and set up local repository:**
```powershell
# Navigate to workspace
cd C:\Users\Luka\Code

# Clone private repository
git clone https://github.com/YOUR_USERNAME/Luka-Private.git
cd Luka-Private

# Verify repository is private
git remote -v
# Should show: origin  https://github.com/YOUR_USERNAME/Luka-Private.git

# Create folder structure
mkdir Credentials, NetworkConfigs, PersonalNotes, LabSecrets, Backups

# Create initial documentation
@"
# Luka Private Repository

**⚠️ WARNING: This repository contains sensitive information**

## Folder Structure
- `Credentials/` - API keys, tokens, certificates
- `NetworkConfigs/` - Router configurations, VPN settings
- `PersonalNotes/` - Private planning and thoughts
- `LabSecrets/` - SSH keys, device passwords
- `Backups/` - Configuration backups

## Security Rules
- Never make this repository public
- Regularly rotate stored credentials
- Use strong passwords for all accounts
- Keep local backups encrypted

## Access Control
- Repository access limited to owner only
- No collaborators unless absolutely necessary
- Regular audit of stored secrets
"@ | Out-File -FilePath README.md -Encoding UTF8
```

**3. Set up security policies:**
```powershell
# Create .gitattributes for security
@"
# Security attributes
*.key binary
*.pem binary
*.p12 binary
*.pfx binary

# Force LFS for large files
*.backup filter=lfs diff=lfs merge=lfs -text
*.dump filter=lfs diff=lfs merge=lfs -text
"@ | Out-File -FilePath .gitattributes -Encoding UTF8

# Create comprehensive .gitignore
@"
# Temporary files
*.tmp
*.temp
*.log
*.cache

# OS files
.DS_Store
Thumbs.db
desktop.ini

# Editor files
.vscode/settings.json
*.swp
*.swo
*~

# Never accidentally commit these patterns anywhere
*password*
*secret*
*private-key*
"@ | Out-File -FilePath .gitignore -Encoding UTF8
```

## File Migration Strategy

### Audit existing repositories for sensitive content:
```powershell
# Navigate to each public repository and audit
cd C:\Users\Luka\Code\LukaSolutions

# Search for potentially sensitive files
Get-ChildItem -Recurse -File | Where-Object { 
    $_.Name -match "(key|password|secret|credential|config|backup)" 
} | Select-Object FullName

# Search file contents for sensitive patterns
Get-ChildItem -Recurse -File -Include "*.md","*.txt","*.py","*.ps1","*.json" | 
    Select-String -Pattern "(password|api[_-]?key|secret|token)" -CaseSensitive:$false |
    Select-Object Filename, LineNumber, Line

# Check git history for sensitive files
git log --name-only --pretty=format: | Sort-Object | Get-Unique | 
    Where-Object { $_ -match "(key|password|secret|credential)" }
```

### Safe file migration process:
```powershell
# Create migration script
@"
# File Migration Script - Run section by section

# 1. Backup current state
cd C:\Users\Luka\Code
git clone --mirror LukaSolutions LukaSolutions-backup
git clone --mirror Luka Luka-backup

# 2. Identify sensitive files (replace with actual files found)
`$sensitiveFiles = @(
    "config\router-backup.cfg",
    "credentials\api-keys.txt",
    "ssh\lab-key.pem"
)

# 3. Move files to private repository
cd Luka-Private
foreach (`$file in `$sensitiveFiles) {
    if (Test-Path "..\LukaSolutions\`$file") {
        `$destDir = Split-Path `$file -Parent
        if (!(Test-Path `$destDir)) { mkdir `$destDir -Force }
        Move-Item "..\LukaSolutions\`$file" "`$destDir\"
        Write-Host "Moved: `$file"
    }
}
"@ | Out-File -FilePath migrate-files.ps1 -Encoding UTF8

# Execute migration (review script first!)
# .\migrate-files.ps1
```

### Remove sensitive files from Git history:
```powershell
# Method 1: Using git filter-branch (for specific files)
git filter-branch --force --index-filter `
    "git rm --cached --ignore-unmatch path/to/sensitive-file.key" `
    --prune-empty --tag-name-filter cat -- --all

# Method 2: Using BFG Repo-Cleaner (recommended for multiple files)
# Download BFG from: https://rtyley.github.io/bfg-repo-cleaner/
# java -jar bfg.jar --delete-files "*.key" .
# java -jar bfg.jar --replace-text passwords.txt .  # Replace patterns

# Method 3: Interactive rebase (for recent commits)
git rebase -i HEAD~10  # Edit last 10 commits

# After cleaning, force push (DANGEROUS - only for personal repos)
# git push --force-with-lease origin main
```

## Advanced Security Practices

### 1. Environment variable management:
```powershell
# Create environment variable management script
@"
# Environment Variable Manager
param(
    [Parameter(Mandatory=`$true)]
    [ValidateSet('set','get','list','backup','restore')]
    [string]`$Action,
    
    [string]`$Name,
    [string]`$Value,
    [string]`$BackupFile = "env-backup.json"
)

switch (`$Action) {
    'set' {
        if (!`$Name -or !`$Value) { 
            Write-Error "Name and Value required for set action"
            return 
        }
        [Environment]::SetEnvironmentVariable(`$Name, `$Value, "User")
        `$env:"`$Name" = `$Value
        Write-Host "Set `$Name"
    }
    'get' {
        if (!`$Name) { 
            Write-Error "Name required for get action"
            return 
        }
        [Environment]::GetEnvironmentVariable(`$Name, "User")
    }
    'list' {
        Get-ChildItem Env: | Where-Object Name -like "*LAB*" -or Name -like "*API*" | 
            Sort-Object Name | Format-Table Name, Value
    }
    'backup' {
        `$vars = Get-ChildItem Env: | Where-Object Name -match "^(LAB_|API_|ROUTER_|DB_)" |
            ForEach-Object { @{Name=`$_.Name; Value=`$_.Value} }
        `$vars | ConvertTo-Json | Out-File `$BackupFile
        Write-Host "Backed up $(`$vars.Count) variables to `$BackupFile"
    }
    'restore' {
        if (!(Test-Path `$BackupFile)) {
            Write-Error "Backup file `$BackupFile not found"
            return
        }
        `$vars = Get-Content `$BackupFile | ConvertFrom-Json
        foreach (`$var in `$vars) {
            [Environment]::SetEnvironmentVariable(`$var.Name, `$var.Value, "User")
            Write-Host "Restored `$(`$var.Name)"
        }
    }
}
"@ | Out-File -FilePath Manage-EnvVars.ps1 -Encoding UTF8

# Usage examples:
# .\Manage-EnvVars.ps1 -Action set -Name "LAB_ROUTER_PASSWORD" -Value "secret123"
# .\Manage-EnvVars.ps1 -Action list
# .\Manage-EnvVars.ps1 -Action backup
```

### 2. Automated secret scanning:
```powershell
# Create secret scanning script
@"
# Secret Scanner for Git Repositories
param([string]`$Path = ".")

`$patterns = @{
    'API Keys' = @(
        'sk-[a-zA-Z0-9]{32,}',  # OpenAI keys
        'AIza[0-9A-Za-z\\-_]{35}',  # Google API keys
        'AKIA[0-9A-Z]{16}'      # AWS keys
    )
    'Passwords' = @(
        'password\s*=\s*["\'][^"\']+["\']',
        'passwd\s*=\s*["\'][^"\']+["\']'
    )
    'Private Keys' = @(
        '-----BEGIN.*PRIVATE KEY-----',
        '-----BEGIN RSA PRIVATE KEY-----'
    )
    'Database URLs' = @(
        'mysql://[^\\s]+',
        'postgresql://[^\\s]+',
        'mongodb://[^\\s]+'
    )
}

foreach (`$category in `$patterns.Keys) {
    Write-Host "`n=== Scanning for `$category ===" -ForegroundColor Yellow
    foreach (`$pattern in `$patterns[`$category]) {
        Get-ChildItem -Path `$Path -Recurse -File | 
            Select-String -Pattern `$pattern -AllMatches |
            ForEach-Object {
                Write-Host "FOUND: `$(`$_.Filename)::`$(`$_.LineNumber)" -ForegroundColor Red
                Write-Host "  `$(`$_.Line.Trim())" -ForegroundColor Gray
            }
    }
}
"@ | Out-File -FilePath Scan-Secrets.ps1 -Encoding UTF8
```

### 3. Secure development workflow:
```powershell
# Pre-commit hook to prevent secret commits
@"
#!/bin/sh
# Save as .git/hooks/pre-commit and make executable

# Run secret scanner
powershell.exe -ExecutionPolicy Bypass -File "./Scan-Secrets.ps1" -Path "."

if [ `$? -ne 0 ]; then
    echo "❌ Secrets detected! Commit blocked."
    echo "Review the output above and remove secrets before committing."
    exit 1
fi

echo "✅ No secrets detected."
exit 0
"@ | Out-File -FilePath pre-commit-hook.sh -Encoding UTF8
```

## Troubleshooting Common Issues

### Environment variable problems:
```powershell
# Problem: Environment variable not visible after setx
# Solution: Refresh environment in current session
[System.Environment]::GetEnvironmentVariable("VARIABLE_NAME","User")

# Problem: Variable works in PowerShell but not in applications
# Solution: Restart applications or entire system

# Problem: Variable disappears after reboot
# Solution: Use setx instead of $env:, check User vs System scope
setx VARIABLE_NAME "value"  # User scope
setx VARIABLE_NAME "value" /M  # System scope (requires admin)

# Problem: Special characters in environment variable values
# Solution: Use proper escaping
setx API_KEY "sk-abc`"def`'ghi"  # Escape quotes with backticks
```

### Git repository problems:
```powershell
# Problem: Accidentally pushed secrets to public repo
# Solution: Immediate response checklist
# 1. Rotate the compromised secrets immediately
# 2. Remove from Git history
# 3. Force push (if personal repo)
# 4. Monitor for unauthorized usage

# Problem: Private repo becomes public accidentally
# Solution: Check repository settings, audit access logs
gh repo view --json visibility
gh repo edit --visibility private

# Problem: Large files in private repo
# Solution: Use Git LFS
git lfs track "*.backup"
git lfs track "*.dump"
git add .gitattributes
```

### Access control issues:
```powershell
# Problem: Lost access to private repository
# Solution: Check GitHub account settings, SSH keys
gh auth status
ssh -T git@github.com

# Problem: Need to share private repo with team member
# Solution: Add collaborator with minimal permissions
gh repo edit --add-collaborator USERNAME --permission read
```

## Integration with Existing Workflow

### Update existing scripts to use environment variables:
```powershell
# Before: Hard-coded credentials
$password = "admin123"
$apiKey = "sk-1234567890"

# After: Environment variables with validation
function Get-SecureConfig {
    param([string]$VarName, [string]$Description)
    
    $value = [Environment]::GetEnvironmentVariable($VarName, "User")
    if (!$value) {
        Write-Error "$Description not configured. Run: setx $VarName 'your-value'"
        throw "Missing configuration: $VarName"
    }
    return $value
}

$password = Get-SecureConfig "ROUTER_ADMIN_PASSWORD" "Router admin password"
$apiKey = Get-SecureConfig "OPENAI_API_KEY" "OpenAI API key"
```

### Create configuration templates:
```powershell
# Template for new projects
@"
# Configuration Template
# Copy this file to config.ps1 and set your environment variables

# Required environment variables:
# setx PROJECT_API_KEY "your-api-key"
# setx PROJECT_DB_URL "your-database-url"  
# setx PROJECT_SECRET_KEY "your-secret-key"

# Validate configuration
function Test-Configuration {
    `$required = @('PROJECT_API_KEY', 'PROJECT_DB_URL', 'PROJECT_SECRET_KEY')
    `$missing = @()
    
    foreach (`$var in `$required) {
        if (!([Environment]::GetEnvironmentVariable(`$var, "User"))) {
            `$missing += `$var
        }
    }
    
    if (`$missing) {
        Write-Error "Missing environment variables: `$(`$missing -join ', ')"
        Write-Host "Set them using: setx VARIABLE_NAME 'value'"
        return `$false
    }
    
    Write-Host "✅ All configuration variables are set"
    return `$true
}

# Test configuration on import
if (!(Test-Configuration)) {
    throw "Configuration incomplete"
}
"@ | Out-File -FilePath config-template.ps1 -Encoding UTF8
```

## Best Practices Summary

### Do's:
- ✅ Always use environment variables for secrets
- ✅ Keep private repositories truly private
- ✅ Regularly audit and rotate credentials
- ✅ Use descriptive environment variable names (PROJECT_API_KEY vs KEY)
- ✅ Document required environment variables in README files
- ✅ Test configuration validation in scripts
- ✅ Keep backups of environment variable configurations
- ✅ Use version control for configuration templates (without values)

### Don'ts:
- ❌ Never commit secrets to any repository (public or private)
- ❌ Don't share environment variable values in chat/email
- ❌ Don't use simple names like PASSWORD or KEY
- ❌ Don't skip validation of required environment variables
- ❌ Don't rely solely on .gitignore for security
- ❌ Don't use the same password/key across multiple services
- ❌ Don't forget to rotate credentials after team members leave

## Extended Examples

### Complete router automation with secrets management:
```python
# router_manager.py
import os
import requests
import logging
from typing import Optional

class RouterManager:
    def __init__(self):
        self.base_url = os.getenv('ROUTER_BASE_URL', 'http://192.168.1.1')
        self.username = os.getenv('ROUTER_USERNAME', 'admin')
        self.password = self._get_required_env('ROUTER_ADMIN_PASSWORD')
        self.session = requests.Session()
        
    def _get_required_env(self, var_name: str) -> str:
        """Get required environment variable or raise error."""
        value = os.getenv(var_name)
        if not value:
            raise ValueError(f"Environment variable {var_name} is required")
        return value
    
    def login(self) -> bool:
        """Login to router."""
        try:
            response = self.session.post(f"{self.base_url}/login", {
                'username': self.username,
                'password': self.password
            })
            return response.status_code == 200
        except Exception as e:
            logging.error(f"Login failed: {e}")
            return False
    
    def backup_config(self, backup_path: str) -> bool:
        """Backup router configuration."""
        if not self.login():
            return False
            
        try:
            response = self.session.get(f"{self.base_url}/backup")
            with open(backup_path, 'wb') as f:
                f.write(response.content)
            return True
        except Exception as e:
            logging.error(f"Backup failed: {e}")
            return False

# Usage
if __name__ == "__main__":
    # Environment variables required:
    # ROUTER_ADMIN_PASSWORD=your-password
    # ROUTER_BASE_URL=http://192.168.1.1 (optional)
    # ROUTER_USERNAME=admin (optional)
    
    router = RouterManager()
    success = router.backup_config("C:/Users/Luka/Code/Luka-Private/NetworkConfigs/router-backup.cfg")
    print(f"Backup {'successful' if success else 'failed'}")
```

This comprehensive guide covers everything needed to securely manage sensitive data in development workflows, based on the practical experience from our session and extended with additional security best practices.