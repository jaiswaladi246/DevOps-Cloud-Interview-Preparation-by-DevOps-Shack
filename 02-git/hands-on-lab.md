# Git & GitHub — Hands-on Lab: A Team Repo Workflow, End to End

<!-- DEVOPS-SHACK:HEADER -->
<p align="center">
  <a href="https://www.youtube.com/@devopsshack"><img alt="YouTube DevOps Shack" src="https://img.shields.io/badge/YouTube-DevOps%20Shack-FF0000?style=flat-square&logo=youtube&logoColor=white"/></a>
  <a href="https://www.youtube.com/@shackverse"><img alt="YouTube DevOps Shack Hindi" src="https://img.shields.io/badge/Hindi-@shackverse-FF0000?style=flat-square&logo=youtube&logoColor=white"/></a>
  <a href="https://devopsshack.com"><img alt="devopsshack.com" src="https://img.shields.io/badge/Courses-devopsshack.com-C6FF4D?style=flat-square&labelColor=0B0F1A"/></a>
  <a href="https://www.instagram.com/devopsshack"><img alt="Instagram" src="https://img.shields.io/badge/Instagram-@devopsshack-E4405F?style=flat-square&logo=instagram&logoColor=white"/></a>
  <a href="https://github.com/jaiswaladi246/DevOps-Cloud-Interview-Preparation-by-DevOps-Shack"><img alt="Original repo" src="https://img.shields.io/badge/Original-DevOps%20Shack%20repo-4DE8FF?style=flat-square&logo=github&labelColor=0B0F1A"/></a>
</p>
<p align="center"><sub>Part of <b><a href="https://github.com/jaiswaladi246/DevOps-Cloud-Interview-Preparation-by-DevOps-Shack">DevOps &amp; Cloud Interview Preparation</a></b> — created by <b><a href="https://www.youtube.com/@devopsshack">DevOps Shack</a></b>. Free to learn from; please credit DevOps Shack when sharing.</sub></p>
<!-- /DEVOPS-SHACK:HEADER -->

> Build the repo setup we use in production: trunk-based branching, protected `main`, CODEOWNERS, PR template, pre-commit with gitleaks, signed commits, SemVer release tags, and a rehearsed recovery from a bad force-push.

[⬅ Back to Git & GitHub](./README.md) · [🏠 Home](../README.md)

## Goal

By the end you will have a GitHub repository where nobody (including admins) can push straight to `main`, every change arrives through a reviewed and tested pull request, secrets are blocked before they are committed, commits are cryptographically signed, releases are cut by pushing a SemVer tag, and you have personally recovered a branch after a destructive force-push.

## Architecture

```mermaid
flowchart LR
    DEV["Developer laptop"] -->|"pre-commit: gitleaks, lint"| LOCAL["Local commit (SSH signed)"]
    LOCAL -->|"git push feature branch"| GH["GitHub repo"]
    GH --> PR["Pull request + template"]
    PR --> CI["CI: checks job"]
    PR --> CO["CODEOWNERS review"]
    CI --> RULE{"Branch protection: all green?"}
    CO --> RULE
    RULE -->|yes| MAIN["Squash merge to main"]
    MAIN -->|"git tag v1.0.0 + push"| REL["Release workflow"]
    REL --> GHREL["GitHub Release + notes"]
```

## Prerequisites

- A GitHub account and a **second identity to act as reviewer** (a teammate, a second account, or a GitHub team). Code-owner approval cannot come from the PR author.
- Git 2.34+ (SSH signing needs it), `gh` CLI 2.40+, Python 3.9+ with `pip`, an SSH key pair.
- Authenticated CLI: `gh auth login`, then `gh auth refresh -h github.com -s admin:ssh_signing_key` so the CLI can register a signing key.
- A throwaway repo. Never run this in a repo you care about.

```bash
git --version
gh --version
gh auth status
export OWNER=$(gh api user -q .login)     # change if using an org
export REPO=team-workflow-lab
export REVIEWER=your-second-account       # GitHub username that will be code owner
```

## Step 1. Create the repo and an initial trunk

```bash
gh repo create "$OWNER/$REPO" --public --clone --add-readme
cd "$REPO"
git branch --show-current            # main
git config pull.rebase true
git config fetch.prune true
```

Add a tiny app and test so CI has something real to run.

```bash
mkdir -p src tests
cat > src/app.py <<'PY'
def add(a: int, b: int) -> int:
    return a + b
PY
cat > tests/test_app.py <<'PY'
from src.app import add


def test_add():
    assert add(2, 3) == 5
PY
touch src/__init__.py
printf "pytest\n" > requirements-dev.txt
printf "__pycache__/\n.pytest_cache/\n.env\n*.pem\n" > .gitignore
git add -A && git commit -m "chore: bootstrap app and tests" && git push origin main
```

**Verify:** `git log --oneline` shows two commits and `gh repo view --web` opens a repo with `src/` and `tests/`.

## Step 2. Configure SSH commit signing

SSH signing avoids GPG key management and works with GitHub's "Verified" badge.

```bash
ssh-keygen -t ed25519 -C "signing key for $OWNER" -f ~/.ssh/id_ed25519_signing -N ""
git config --global gpg.format ssh
git config --global user.signingkey ~/.ssh/id_ed25519_signing.pub
git config --global commit.gpgsign true
git config --global tag.gpgsign true
# local verification of signatures
echo "$(git config user.email) $(cat ~/.ssh/id_ed25519_signing.pub)" > ~/.ssh/allowed_signers
git config --global gpg.ssh.allowedSignersFile ~/.ssh/allowed_signers
# register the public key on GitHub as a *signing* key
gh ssh-key add ~/.ssh/id_ed25519_signing.pub --type signing --title "lab signing key"
```

Make sure `git config user.email` matches a verified email on your GitHub account, otherwise the badge shows "Unverified".

**Verify:**

```bash
git commit --allow-empty -m "chore: test signing"
git log --show-signature -1
```

You should see `Good "git" signature for you@example.com with ED25519 key SHA256:...`. After a push the commit shows **Verified** on GitHub. Drop this test commit with `git reset --hard HEAD~1` before pushing.

## Step 3. Add pre-commit with gitleaks

```bash
pip install pre-commit
cat > .pre-commit-config.yaml <<'YAML'
repos:
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v5.0.0
    hooks:
      - id: trailing-whitespace
      - id: end-of-file-fixer
      - id: check-yaml
      - id: check-added-large-files
        args: ["--maxkb=500"]
      - id: detect-private-key
  - repo: https://github.com/gitleaks/gitleaks
    rev: v8.24.0
    hooks:
      - id: gitleaks
  - repo: https://github.com/compilerla/conventional-pre-commit
    rev: v4.0.0
    hooks:
      - id: conventional-pre-commit
        stages: [commit-msg]
        args: [feat, fix, docs, chore, refactor, test, ci, build]
YAML
pre-commit install --install-hooks
pre-commit install --hook-type commit-msg
pre-commit autoupdate        # bumps the pinned revs to current releases
```

**Verify with a deliberately bad commit.** Use the well-known AWS documentation example key, not a real one.

```bash
git switch -c chore/precommit
echo 'AWS_ACCESS_KEY_ID=AKIAIOSFODNN7EXAMPLE' > leak.env
echo 'aws_secret_access_key = "wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY"' >> leak.env
git add leak.env
git commit -m "chore: add config"
```

Expected: gitleaks fails the commit with a finding such as `RuleID: aws-access-token` (or `generic-api-key`) and prints the file and line. Nothing is committed. If the documented example key is allow-listed by your gitleaks version, use a random 32-character token assigned to `api_key =` instead.

```bash
git restore --staged leak.env && rm leak.env
git add .pre-commit-config.yaml
git commit -m "ci: add pre-commit with gitleaks and conventional commits"
```

Also try `git commit -m "added stuff"`: the commit-msg hook rejects it. Remember `git commit --no-verify` bypasses all of this, which is why CI re-runs the same hooks in Step 5.

## Step 4. PR template and CODEOWNERS

```bash
mkdir -p .github
cat > .github/pull_request_template.md <<'MD'
## What and why
<!-- One or two sentences. Link the issue: Closes #123 -->

## How it was tested
<!-- Commands run, screenshots, test names -->

## Risk and rollback
- [ ] Safe to revert with `git revert` of the squash commit
- [ ] No secrets, tokens or personal data added
- [ ] Docs or runbook updated if behaviour changed

## Release impact
- [ ] Patch (fix)
- [ ] Minor (feature)
- [ ] Major (breaking)
MD

cat > .github/CODEOWNERS <<OWNERS
# Last matching pattern wins. Owners must have write access.
*                         @$OWNER
/.github/                 @$REVIEWER
/.pre-commit-config.yaml  @$REVIEWER
/src/                     @$OWNER @$REVIEWER
OWNERS
git add .github
git commit -m "docs: add PR template and CODEOWNERS"
```

For an organisation, use teams: `/terraform/ @my-org/platform-team`. The `.github/` entry matters: without it, anyone could weaken the CI checks through a normal PR.

## Step 5. CI workflow that mirrors the local hooks

```bash
mkdir -p .github/workflows
cat > .github/workflows/ci.yml <<'YAML'
name: ci
on:
  pull_request:
  push:
    branches: [main]
permissions:
  contents: read
concurrency:
  group: ci-${{ github.ref }}
  cancel-in-progress: true
jobs:
  checks:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0          # gitleaks scans history, needs full clone
      - uses: actions/setup-python@v5
        with:
          python-version: "3.12"
      - name: Install tools
        run: pip install pre-commit -r requirements-dev.txt
      - name: Run pre-commit on all files
        run: pre-commit run --all-files --show-diff-on-failure
      - name: Unit tests
        run: python -m pytest -q
YAML
git add .github/workflows/ci.yml
git commit -m "ci: run pre-commit and tests on every PR"
```

The job is named `checks`. That exact name is what branch protection will require.

## Step 6. Ship the setup through the first PR (while main is still open)

```bash
git push -u origin chore/precommit
gh pr create --fill --base main
gh pr checks --watch
```

**Verify:** `gh pr checks` shows `checks` passing. The `conventional-pre-commit` hook only runs at the commit-msg stage, so it does not appear in CI. Merge this one to bootstrap, using squash:

```bash
gh pr merge --squash --delete-branch
git switch main && git pull
```

## Step 7. Enable branch protection on `main`

Now lock it down. Write the policy as code so it is reviewable and repeatable.

```bash
cat > protection.json <<'JSON'
{
  "required_status_checks": { "strict": true, "contexts": ["checks"] },
  "enforce_admins": true,
  "required_pull_request_reviews": {
    "required_approving_review_count": 1,
    "require_code_owner_reviews": true,
    "dismiss_stale_reviews": true,
    "require_last_push_approval": true
  },
  "restrictions": null,
  "required_linear_history": true,
  "allow_force_pushes": false,
  "allow_deletions": false,
  "required_conversation_resolution": true
}
JSON
gh api -X PUT "repos/$OWNER/$REPO/branches/main/protection" --input protection.json
# require signed commits (separate endpoint)
gh api -X POST "repos/$OWNER/$REPO/branches/main/protection/required_signatures" \
  -H "Accept: application/vnd.github+json"
# merge button hygiene: squash only, auto-delete merged branches
gh api -X PATCH "repos/$OWNER/$REPO" \
  -F allow_squash_merge=true -F allow_merge_commit=false -F allow_rebase_merge=false \
  -F delete_branch_on_merge=true >/dev/null
# invite the reviewer so CODEOWNERS can work
gh api -X PUT "repos/$OWNER/$REPO/collaborators/$REVIEWER" -f permission=push
```

Do not commit `protection.json` into this repo unless you want it as documentation; delete it with `rm protection.json`.

**Verify the wall holds:**

```bash
git switch main
git commit --allow-empty -m "chore: direct push test"
git push origin main
```

Expected rejection:

```
remote: error: GH006: Protected branch update failed for refs/heads/main.
remote: - Changes must be made through a pull request.
remote: - Required status check "checks" is expected.
 ! [remote rejected] main -> main (protected branch hook declined)
```

Undo the local commit: `git reset --hard origin/main`.

> On a repo with a single owner, `require_code_owner_reviews` plus `enforce_admins` means you need the second identity to approve. That is the point of the lab. Do not weaken the rule to make it convenient.

## Step 8. The daily loop: a trunk-based change

```bash
git switch -c feat/multiply
cat >> src/app.py <<'PY'


def multiply(a: int, b: int) -> int:
    return a * b
PY
cat >> tests/test_app.py <<'PY'


def test_multiply():
    assert multiply(3, 4) == 12
PY
sed -i 's/import add$/import add, multiply/' tests/test_app.py
python -m pytest -q
git add -A
git commit -m "feat: add multiply"
git push -u origin feat/multiply
gh pr create --fill
gh pr checks --watch
```

The PR body is pre-filled from the template. As the reviewer (second account, or in the web UI), approve:

```bash
gh pr review --approve        # run as $REVIEWER
```

Merge as the author once checks are green and the review is in:

```bash
gh pr merge --squash --delete-branch
git switch main && git pull
git log --oneline --show-signature -3
```

**Verify:** `git log --oneline` on main is a straight line with one commit per PR. The squash commit is created and signed by GitHub, so it shows Verified.

Test the CODEOWNERS rule: open a PR that edits `.github/CODEOWNERS` as the owner and note the merge box says "Review required from code owners".

## Step 9. Cut a release with a SemVer tag

Add the release workflow first, through a PR like any other change.

```bash
git switch -c ci/release-workflow
cat > .github/workflows/release.yml <<'YAML'
name: release
on:
  push:
    tags: ["v[0-9]+.[0-9]+.[0-9]+"]
permissions:
  contents: write
jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Create GitHub release
        env:
          GH_TOKEN: ${{ github.token }}
        run: gh release create "$GITHUB_REF_NAME" --generate-notes --verify-tag
YAML
git add -A && git commit -m "ci: create GitHub release on SemVer tag"
git push -u origin ci/release-workflow
gh pr create --fill && gh pr checks --watch
# approve as reviewer, then:
gh pr merge --squash --delete-branch && git switch main && git pull
```

Tag the first release. Annotated, signed (because `tag.gpgsign` is true), on the exact commit that main points to:

```bash
git tag -a v1.0.0 -m "v1.0.0: add, multiply"
git tag -v v1.0.0                   # verify signature locally
git push origin v1.0.0
gh run watch
gh release view v1.0.0
```

Practise version bumps. A bug fix is `v1.0.1`, a new backward-compatible function is `v1.1.0`, a removed or renamed public function is `v2.0.0`. Find what changed since the last tag when writing notes:

```bash
git describe --tags --abbrev=0
git log "$(git describe --tags --abbrev=0)"..HEAD --oneline
```

**Verify:** `gh release list` shows `v1.0.0` with generated notes listing the merged PRs.

If a tag was pushed by mistake, delete it and cut a new version rather than reusing the number once anyone could have consumed it:

```bash
git push origin :refs/tags/v1.0.1 && git tag -d v1.0.1
```

## Step 10. Simulate and recover from a bad force-push

`main` is protected, so rehearse on an unprotected scratch branch that behaves like a shared feature branch.

```bash
git switch -c scratch/shared-work
for i in 1 2 3; do echo "line $i" >> notes.txt; git add notes.txt; git commit -m "chore: note $i"; done
git push -u origin scratch/shared-work
git rev-parse HEAD        # write this down, call it GOOD
```

Now the accident. A colleague "cleans up" and force-pushes an older state:

```bash
git reset --hard HEAD~3
git push --force origin scratch/shared-work
```

The remote branch has lost three commits, and so has your local branch. Recover in the order below, from most to least convenient.

**Option A: your own reflog (you had the commits locally).**

```bash
git reflog -10
# abc1234 HEAD@{1}: commit: chore: note 3
git branch rescue abc1234          # or: git reset --hard 'HEAD@{1}'
git log --oneline rescue
```

**Option B: the remote-tracking reflog**, which records where `origin/...` pointed before your last fetch or push.

```bash
git reflog show origin/scratch/shared-work
```

**Option C: a teammate's clone or GitHub itself, if your clone never had the commits.** GitHub keeps push history in the events API:

```bash
gh api "repos/$OWNER/$REPO/events" \
  --jq '.[] | select(.type=="PushEvent") | {ref: .payload.ref, before: .payload.before, head: .payload.head}' | head
```

The `before` of the force-push event is the lost tip. Try fetching it by hash (GitHub often still serves unreachable-but-present commits):

```bash
git fetch origin <before-sha>
git branch rescue FETCH_HEAD
```

**Restore the remote safely.** Use `--force-with-lease` pinned to the value you believe the remote has, so you do not clobber someone else's newer push:

```bash
git switch rescue
git push --force-with-lease=scratch/shared-work:"$(git rev-parse origin/scratch/shared-work)" \
  origin rescue:scratch/shared-work
```

**Verify:** `git log --oneline origin/scratch/shared-work` lists all three note commits again. Tell the team to run `git fetch && git switch scratch/shared-work && git reset --hard origin/scratch/shared-work` (or `git pull --rebase` if they have local work to keep).

Then confirm that the protection you built stops the same mistake on `main`:

```bash
git switch main
git push --force origin HEAD~1:main
# remote: error: GH006: Protected branch update failed ... Cannot force-push to this branch
```

## Step 11. Prove that secrets are caught server-side too

Pre-commit can be skipped, so check the second layer.

```bash
git switch -c test/bypass
printf 'api_key = "9f8b7c6d5e4f3a2b1c0d9e8f7a6b5c4d"\n' > creds.py
git add creds.py
git commit --no-verify -m "chore: sneaky"
git push -u origin test/bypass
gh pr create --fill
gh pr checks --watch
```

The `checks` job fails because `pre-commit run --all-files` runs gitleaks again, and the required status check blocks the merge. Clean up with `gh pr close --delete-branch`.

Real-world note: if a real credential ever lands in history, rotate it first, then remove it from history with `git filter-repo --replace-text`, then force-push and ask everyone to re-clone. Rotation matters more than scrubbing, because the secret has already been exposed.

## Final verification checklist

```bash
gh api "repos/$OWNER/$REPO/branches/main/protection" \
  --jq '{linear: .required_linear_history.enabled, force: .allow_force_pushes.enabled, admins: .enforce_admins.enabled, codeowners: .required_pull_request_reviews.require_code_owner_reviews, checks: .required_status_checks.contexts}'
gh api "repos/$OWNER/$REPO/branches/main/protection/required_signatures" --jq .enabled
git log --format='%h %G? %s' -5       # G = good signature
gh release list
```

Expected: linear true, force false, admins true, codeowners true, checks `["checks"]`, signatures true, recent commits marked `G`, at least one release.

## Cleanup

```bash
cd ..
gh repo delete "$OWNER/$REPO" --yes        # needs: gh auth refresh -s delete_repo
rm -rf "$REPO"
gh ssh-key list                            # find the signing key id
gh ssh-key delete <id>
rm ~/.ssh/id_ed25519_signing ~/.ssh/id_ed25519_signing.pub
git config --global --unset gpg.format
git config --global --unset user.signingkey
git config --global --unset commit.gpgsign
git config --global --unset tag.gpgsign
git config --global --unset gpg.ssh.allowedSignersFile
```

## What to say in an interview about this project

"I set up a repo workflow the way we run it in production. It is trunk-based: short-lived branches, squash merges, linear history, so every PR is one revertable commit. Main is protected with required reviews, a required status check, CODEOWNERS for sensitive paths like CI and infrastructure, stale-approval dismissal and no admin bypass. Shift-left security is two layers: pre-commit with gitleaks locally, then the same hooks in CI as a required check, because hooks can be skipped with `--no-verify`. Commits are SSH-signed and enforced on the branch. Releases are annotated SemVer tags, and pushing the tag triggers a workflow that publishes the GitHub release with generated notes. I also rehearsed a force-push disaster: recovered the old tip from reflog and from the GitHub events API, restored with `--force-with-lease`, and then showed that protection blocks it on main. The trade-offs I'd mention: code-owner reviews slow small teams, signed-commit enforcement needs onboarding, and squash merging loses individual commit granularity."

## Common errors and fixes

| Symptom | Cause | Fix |
|---------|-------|-----|
| `error: gpg failed to sign the data` / `Load key ... invalid format` | `user.signingkey` points at the private key, or `gpg.format` unset | Point to the `.pub` file and set `gpg.format ssh` |
| Commit shows "Unverified" on GitHub | Key added as authentication, not signing, or email not verified | `gh ssh-key add --type signing`; use a verified email in `user.email` |
| `gpg.ssh.allowedSignersFile needs to be configured and exist for ssh signature verification` | `git log --show-signature` without a signers file | Create the file and set `gpg.ssh.allowedSignersFile` |
| `remote: error: GH006: Protected branch update failed` | Pushing to protected `main` | Push a branch and open a PR |
| `Required status check "checks" is expected` forever | Job name differs from required context, or workflow does not run on `pull_request` | Match the job name exactly; check the Actions tab |
| Merge box says "Review required from code owners" for the author | Author cannot approve their own PR | Approve from the second account or a team member |
| `Resource not accessible by integration` in the release job | Missing `permissions: contents: write` | Add it to the workflow or job |
| Release workflow never starts | Tag pattern mismatch, or tag not pushed | Push the tag; `v[0-9]+.[0-9]+.[0-9]+` matches `v1.0.0` but not `1.0.0` |
| gitleaks finds nothing in CI though a secret exists | Shallow clone | Use `fetch-depth: 0` |
| `pre-commit` cannot install hook environments | No route to GitHub to clone hook repos | Set `HTTPS_PROXY`, or pre-populate `~/.cache/pre-commit` |
| `! [rejected] ... (stale info)` on `--force-with-lease` | Remote moved since your last fetch | `git fetch`, inspect, then retry with the new expected value |
| 403 or "Upgrade to GitHub Pro" from the protection API | Private repo on a free plan, or token lacks admin | Use a public repo, a paid plan, or a token with repo admin rights |
| `fatal: remote error: upload-pack: not our ref <sha>` when fetching a lost commit | Server refuses unadvertised SHAs, or the commit was garbage-collected | Use a teammate's clone or CI cache; contact GitHub Support for unreachable commits |

[⬅ Back to Git & GitHub](./README.md) · [🏠 Home](../README.md)

<!-- DEVOPS-SHACK:FOOTER -->
---

<div align="center">

### 🚀 Made with ❤️ by [DevOps Shack](https://www.youtube.com/@devopsshack)

**Learn DevOps, DevSecOps, Cloud and Kubernetes the practical way.**

[▶️ YouTube — DevOps Shack](https://www.youtube.com/@devopsshack) · [▶️ Hindi — @shackverse](https://www.youtube.com/@shackverse) · [🎓 devopsshack.com](https://devopsshack.com) · [📸 Instagram @devopsshack](https://www.instagram.com/devopsshack)

[🏠 Back to Home](../README.md) · [⭐ Star the original repo](https://github.com/jaiswaladi246/DevOps-Cloud-Interview-Preparation-by-DevOps-Shack)

<sub>© DevOps Shack · Licensed CC BY-NC-SA 4.0 · If you found this copy anywhere other than <a href="https://github.com/jaiswaladi246/DevOps-Cloud-Interview-Preparation-by-DevOps-Shack">github.com/jaiswaladi246</a>, the original and latest version is maintained by DevOps Shack.</sub>

</div>
<!-- /DEVOPS-SHACK:FOOTER -->
