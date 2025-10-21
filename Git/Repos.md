# Managing GitHub Repositories — Practical Guide (Expanded)

Index
1. Introduction & Purpose
2. Repository structure & files
3. Repo types & strategy
4. Access control & roles
5. Branch & release workflow
6. Pull requests & code review
7. Automation & CI/CD
8. Issues, projects & roadmap
9. Templates & onboarding
10. Permissions & machines
11. Testing & quality gates
12. Archiving & cleanup
13. Backups & mirrors
14. Governance & policies
15. Tools & integrations
16. Quick checklist for new repo
17. Useful commands
18. Examples & templates to add
19. Final notes & next steps

---

## 1. Introduction & Purpose
- How to structure, manage access, and operate repositories so projects remain maintainable, secure and auditable.
- Practical explanations, command examples (PowerShell / bash / gh), templates and tips for common workflows.

---

## 2. Repository structure & files (detailed)

Why structure matters
- Predictable layout eases onboarding, CI, review. Keep code, tests, docs separated and avoid large binaries in source.

Required files (what and why)
- README.md — purpose, quick start, build/run/test.
- LICENSE — legal permissions (add SPDX identifier where applicable).
- CONTRIBUTING.md — branch policy, CI expectations.
- SECURITY.md — vulnerability reporting instructions.
- CODE_OF_CONDUCT.md — community behavior expectations.

Suggested folders
- src/ or lib/ — source code.
- tests/ or spec/ — unit/integration tests.
- docs/ — design docs, ADRs.
- infra/ or deploy/ — deployment configs (no secrets).
- data/ or assets/ — small files only; large datasets in external storage.
- examples/ — minimal usage examples.

Example commands (create initial files)
```powershell
# PowerShell (Windows)
Set-Content -Path README.md -Value "# Project name`nShort description and quickstart"
Invoke-RestMethod -Uri "https://raw.githubusercontent.com/licenses/license-templates/master/templates/mit.txt" -OutFile LICENSE
git add README.md LICENSE
git commit -m "Add README and LICENSE"
```

```bash
# bash
echo "# Project name" > README.md
curl -sS https://raw.githubusercontent.com/licenses/license-templates/master/templates/mit.txt -o LICENSE
git add README.md LICENSE && git commit -m "Add README and LICENSE"
```

.gitignore guidance
- Add platform/language ignores and exclude secrets, env files, and build artifacts.

Example .gitignore snippet:
```
# Node
node_modules/
.env

# Python
__pycache__/
*.py[cod]
.env

# OS/editor
.DS_Store
.vscode/
```

Large files
- Use Git LFS for binaries: `git lfs track "*.bin"` and commit `.gitattributes`.
- Prefer external storage (S3, artifact storage) and references.

---

## 3. Repo types & strategy (expanded)

Single-purpose vs monorepo
- Single-purpose repo: clear ownership, simpler CI. Good for small teams and independent services.
- Monorepo: easier refactors across services, unified CI, requires tooling and governance.

Template repositories
- Standardize new projects with templates (README, CI, issue templates).
```bash
gh repo create my-repo --public --source=. --remote=origin
# or create from a template:
gh repo create org/repo-from-template --template TEMPLATE_OWNER/TEMPLATE_REPO
```

When to split repos
- Different lifecycles, release cadences, or access/security requirements.

---

## 4. Access control & roles (expanded)

Use an Organization
- Host projects under a GitHub Organization (not personal accounts) to centralize ownership, billing, policy and audit controls.
- Benefits: team management, SSO/SCIM, org-wide policies (branch protection defaults, required sign‑offs), and centralized code scanning/config.

Teams & roles — principles
- Principle of least privilege: grant the minimum role required for the task and use teams (not direct collaborator invites) to apply permissions.
- Prefer ephemeral elevated access (time‑boxed) rather than permanent privilege increases.
- Use naming conventions and documentation for teams (e.g., infra-admins, backend-maintain, docs-write).

Common roles and recommended use
- Owner: org‑level administrators (very few). Manage billing, SSO, org settings. Use for emergency recovery only.
- Admin: repository administrators for a set of repos or teams; can manage repo settings, secrets and teams.
- Maintain: day‑to‑day maintainers — can merge, manage issues and PRs, configure some settings.
- Write: contributors who push branches and open PRs.
- Triage: issue/PR triaging, labeling and assigning, without push permissions.
- Read: auditors, reviewers, and external stakeholders (read-only).
- Map teams to roles: e.g. @org/backend -> Maintain, @org/frontend -> Write, @org/support -> Triage.

Team structure & best practices
- Create teams per logical responsibility (service, platform, infra, security) and subteams for scoped access.
- Assign repository permissions to teams, not users. Keep team membership synced with HR/SSO where possible.
- Use team descriptions to document purpose, on-call rotation and escalation contacts.

Branch protection & codeowners
- Enforce branch protection at repo or org level: require PRs, status checks, linear history and required review counts.
- Use CODEOWNERS to require review from the appropriate team(s) (e.g., /src/ @org/backend).
- Ensure branch protection prevents accidental admin bypass (enforce_admins=true where appropriate).

Machine access, deploy keys & automation
- Use GitHub Apps or GitHub Actions with fine‑grained permissions for automation rather than personal tokens.
- Use deploy keys (read‑only) for single‑repo machine access; use machine accounts or Apps for cross‑repo automation.
- Restrict actions to approved workflows and use repository/workflow-level secrets with least privilege.

Onboarding, offboarding & periodic review
- Automate onboarding: provision team membership from identity provider (SSO/SCIM) where possible.
- Offboard promptly: revoke team membership, personal access tokens, and rotate repository-level secrets.
- Schedule periodic audit (quarterly): review org owners, team memberships, deploy keys, and access logs.

Auditing & logging
- Enable audit log retention (Enterprise) and integrate with SIEM for critical org events.
- Monitor actions: new key adds, team membership changes, repo permission changes, and webhooks.

Examples (gh CLI / API)
- Create team and add repo:
```bash
gh api orgs/ORG/teams -f name="backend-maintain" -f privacy=closed
gh api orgs/ORG/teams/backend-maintain/repos/OWNER/REPO -X PUT -f permission=maintain
```
- Invite user to team:
```bash
gh api orgs/ORG/teams/TEAM_SLUG/memberships/USERNAME -X PUT -f role="member"
```
- Add deploy key (read-only):
```bash
gh api repos/OWNER/REPO/keys --field title="deploy-key" --field key="$(cat ~/.ssh/id_rsa.pub)" --raw-field read_only=true
```
- Apply branch protection (example):
```bash
gh api repos/OWNER/REPO/branches/main/protection -X PUT \
  -f required_status_checks='{"strict":true,"contexts":["ci"]}' \
  -f enforce_admins=true \
  -f required_pull_request_reviews='{"required_approving_review_count":1}'
```

Emergency & break‑glass procedures
- Maintain a small, audited break‑glass owner team with multi-person approval for emergency changes.
- Document and log any emergency privilege escalations; rotate credentials after use.

Checklist (quick)
- [ ] Organization created and SSO configured
- [ ] Teams mapped to repos (no per-user repo invites)
- [ ] Branch protection and CODEOWNERS applied
- [ ] Deploy keys/applications reviewed and rotated
- [ ] Quarterly access audit scheduled

---

## 5. Branch & release workflow (expanded)

Branching models
- Trunk-based: short-lived branches, frequent merges — recommended for CD.
- GitFlow: suitable for longer release cycles with release branches.

Protect main
- Require PRs, CI success, and code owner reviews.

Example branch creation and pushing:
```bash
git checkout -b feature/add-logger
# work...
git add .
git commit -m "Add logger"
git push -u origin feature/add-logger
```

Releases & tags
- Use semantic versioning and GitHub Releases:
```bash
gh release create v1.0.0 --title "v1.0.0" --notes "Initial release"
```

Release automation
- Trigger releases from CI when release branch merged or tag pushed.

---

## 6. Pull requests & code review (expanded)

Require PRs
- Use PR templates and CODEOWNERS to request reviewers automatically.

Example CODEOWNERS:
```
# file: .github/CODEOWNERS
/docs/ @doc-team
/src/ @backend-team
```

Create PR via CLI:
```bash
gh pr create --base main --head feature/add-logger --title "Add logger" --body "Adds structured logging"
```

Review checklist (recommended)
- Tests added/passing
- Lint/static analysis clean
- Docs/README updated if API changed
- Security review for new deps

Pre-merge checks
- Require reviewer(s), CI pass, and signed commits if policy requires.

---

## 7. Automation & CI/CD (expanded)

CI choices
- GitHub Actions (native), or Jenkins/GitLab/CircleCI depending on infra.

Secrets management
- Use GitHub Actions Secrets or external vaults (Vault, Azure/AWS Key Vault).
```bash
echo "secret-value" | gh secret set ACTIONS_TOKEN -R owner/repo
```

Example minimal GitHub Actions CI (.github/workflows/ci.yml):
```yaml
name: CI
on: [push, pull_request]
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.11'
      - name: Install deps
        run: pip install -r requirements.txt
      - name: Run tests
        run: pytest -q
```

Protecting secrets
- Use environment-level secrets or OIDC for cloud access.

CI tips
- Cache dependencies, fail fast on lint, limit matrix size to control costs.

---

## 8. Issues, Projects & Roadmap (expanded)

Use issues effectively
- Templates for bug/feature/chore. Use labels consistently.

Create issue example:
```bash
gh issue create --title "Bug: login fails on edge" --body "Steps to reproduce:\n1. ...\nExpected..."
```

Projects & milestones
- Use GitHub Projects for boards; Milestones for releases. Keep ROADMAP.md in docs.

Automation
- Use Actions to auto-create issues or move project cards on PR events.

---

## 9. Templates & onboarding (expanded)

Templates
- Add ISSUE_TEMPLATE, PULL_REQUEST_TEMPLATE in .github/.

Example PR template snippet:
```
### Summary
- What changed and why

### Checklist
- [ ] Tests added
- [ ] Documentation updated
```

Onboarding checklist (ONBOARDING.md)
- Repo purpose, local run instructions, test commands, contact points.

---

## 10. Permissions & machines (expanded)

Deploy keys vs machine users vs GitHub Apps
- Deploy key: read-only per-repo. GitHub App: recommended for automation.
- Rotate keys regularly and audit access.

Add deploy key example (API)
```bash
gh api repos/OWNER/REPO/keys --field title="deploy-key" --field key="$(cat ~/.ssh/id_rsa.pub)" --raw-field read_only=true
```

---

## 11. Testing & quality gates (expanded)

Recommended checks
- Unit/integration tests, linting, security scans. Block merges on failing checks.

Local test examples
```bash
# Python
pytest -q

# Node
npm ci && npm test
```

Pre-commit hooks
- Use pre-commit to run linters locally: https://pre-commit.com

Security scanning
- Enable Dependabot and CodeQL.

---

## 12. Archiving & cleanup (expanded)

When to archive
- Project inactive or replaced.

Archive via API:
```bash
gh api repos/OWNER/REPO -X PATCH -f archived=true
```

Cleanup before delete
- Tag final snapshot, export/close issues, notify stakeholders.

---

## 13. Backups & mirrors (expanded)

Create mirror backups
```bash
git clone --mirror git@github.com:owner/repo.git
# push mirror to offsite periodically
```

Store build artifacts externally (S3/GCS) and encrypt backups.

---

## 14. Governance & policies (expanded)

Repository lifecycle policy
- Document create → maintain → deprecate → archive.

Security policy
- SECURITY.md with contact steps and PGP/GPG keys for secure reporting.

Audit cadence
- Quarterly audit of collaborators, deploy keys and secrets.

---

## 15. Tools & integrations (expanded)

Useful tools
- gh (GitHub CLI), VS Code + GitLens, Dependabot, CodeQL, Snyk.

Integration tips
- Connect CI to issue tracker; store artifacts as workflow artifacts or in object storage.

---

## 16. Quick checklist for new repo (expanded)
- [ ] README with quickstart and diagrams
- [ ] LICENSE (SPDX)
- [ ] CONTRIBUTING.md + .github templates
- [ ] Branch protection for main
- [ ] CI configured and passing on PR
- [ ] Dependabot/code scanning enabled
- [ ] Teams & permissions configured
- [ ] Secrets and deploy keys audited

---

## 17. Useful commands (repo management & workflows) — expanded

Basic Git
```bash
git clone git@github.com:owner/repo.git
git checkout -b feature/name
git add .
git commit -m "message"
git push -u origin feature/name
git switch main
```

GitHub CLI
```bash
gh repo create my-repo --public --source=. --remote=origin
gh pr create --base main --head feature/name --title "My PR" --body "Description"
gh pr checkout 123
gh pr merge 123 --merge
```

Branch protection & CI
```bash
gh workflow list -R owner/repo
gh workflow run ci.yml -R owner/repo
```

Teams, access & deploy keys
```bash
gh api repos/OWNER/REPO/collaborators
gh api repos/OWNER/REPO/keys --field title="deploy-key-pi" --field key="$(cat ~/.ssh/id_ed25519_pi.pub)" --raw-field read_only=true
```

CI secrets & automation
```bash
echo "secret-value" | gh secret set ACTIONS_TOKEN -R owner/repo
gh secret list -R owner/repo
```

Archive, mirror & cleanup
```bash
gh api repos/OWNER/REPO -X PATCH -f archived=true
git clone --mirror git@github.com:owner/repo.git
```

---

## 18. Examples & templates to add to repo

Add these to `.github/`:
- ISSUE_TEMPLATE/bug.md
- PULL_REQUEST_TEMPLATE.md
- CODEOWNERS
- .github/workflows/ci.yml

Example CODEOWNERS
```
/docs/ @doc-team
/src/ @backend-team
```

---

## 19. Final notes & next steps
- Decide branch model (trunk vs gitflow) and document in CONTRIBUTING.md.
- Create repo template including README, CI, templates, and .gitignore.
- Configure org-level policies (teams, SSO, branch protection).
- Offer help generating CI workflows or CODEOWNERS for specific stacks (Node, Python, Go).