# Git & GitHub — Cheatsheet

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

> Dense command and config reference grouped by task, ending with the top 15 interview one-liners.

[⬅ Back to Git & GitHub](./README.md) · [🏠 Home](../README.md)

## Table of contents

- [Setup and config](#setup-and-config)
- [Everyday workflow](#everyday-workflow)
- [Inspecting history](#inspecting-history)
- [Branching and merging](#branching-and-merging)
- [Rebase and history rewriting](#rebase-and-history-rewriting)
- [Undo and recovery](#undo-and-recovery)
- [Remotes](#remotes)
- [Tags and releases](#tags-and-releases)
- [Stash, worktree, bisect](#stash-worktree-bisect)
- [Large repos](#large-repos)
- [Hooks, secrets, signing](#hooks-secrets-signing)
- [GitHub CLI and API](#github-cli-and-api)
- [Config files](#config-files)
- [Top 15 interview one-liners](#top-15-interview-one-liners)

---

## Setup and config

```bash
git config --global user.name  "Your Name"
git config --global user.email "you@example.com"
git config --global init.defaultBranch main
git config --global pull.rebase true          # or: pull.ff only
git config --global fetch.prune true
git config --global rebase.autoSquash true
git config --global merge.conflictStyle zdiff3
git config --global rerere.enabled true
git config --list --show-origin               # where each value comes from
```

| Scope | Flag | File |
|-------|------|------|
| System | `--system` | `/etc/gitconfig` |
| Global (user) | `--global` | `~/.gitconfig` or `~/.config/git/config` |
| Local (repo) | `--local` (default) | `.git/config` |

## Everyday workflow

| Task | Command |
|------|---------|
| Start a repo | `git init` / `git clone <url>` |
| Status, short form | `git status -sb` |
| Stage everything / part of a file | `git add -A` / `git add -p` |
| Commit | `git commit -m "feat: add x"` |
| Amend last commit (unpushed) | `git commit --amend` / `--amend --no-edit` |
| Fixup an earlier commit | `git commit --fixup <sha>` |
| See unstaged / staged changes | `git diff` / `git diff --staged` |
| Delete or rename tracked file | `git rm <f>` / `git mv <old> <new>` |
| Stop tracking, keep file | `git rm --cached <f>` |
| Which rule ignores this? | `git check-ignore -v <path>` |

## Inspecting history

```bash
git log --oneline --graph --decorate --all
git log -p -- path/to/file                 # patches touching a file
git log -S'needle'                         # commits that add/remove the string
git log -G'regex'                          # commits whose diff matches regex
git log --since="2 weeks ago" --author="alice"
git log main..feature                      # in feature, not in main
git log main...feature --left-right        # symmetric difference
git show <sha>:path/file                   # file at a commit
git blame -L 10,30 file                    # who changed lines 10-30
git blame -w -C -C file                    # ignore whitespace, follow moves
git shortlog -sn                           # commits per author
git describe --tags                        # nearest tag + distance
git rev-parse HEAD                         # full SHA
git ls-files -s                            # index entries with mode and hash
```

| Revision syntax | Meaning |
|-----------------|---------|
| `HEAD~2` | Two commits back along first parents |
| `HEAD^2` | Second parent of a merge commit |
| `A..B` | Reachable from B but not from A |
| `A...B` | Reachable from either, not both |
| `@{u}` | Upstream of current branch |
| `HEAD@{3}` | Where HEAD was 3 moves ago (reflog) |
| `:/fix login` | Latest commit whose message matches |

## Branching and merging

```bash
git switch -c feat/x            # create and switch
git switch -                    # previous branch
git branch -vv                  # tracking info
git branch --merged main        # safe to delete
git branch -d feat/x            # delete merged   (-D forces)
git merge --no-ff feat/x        # always create a merge commit
git merge --squash feat/x       # stage the combined changes, commit yourself
git merge --abort
git push origin --delete feat/x
```

| Merge strategy | When |
|----------------|------|
| Merge commit | Preserve full branch history and context |
| Squash merge | One commit per PR, clean trunk, easy revert |
| Rebase merge | Linear history with every commit kept |
| Fast-forward only (`--ff-only`) | Refuse to create merge commits locally |

## Rebase and history rewriting

```bash
git rebase main                        # replay current branch on main
git rebase -i HEAD~5                   # pick reword edit squash fixup drop
git rebase -i --autosquash main
git rebase --onto newbase oldbase branch
git rebase --continue | --skip | --abort
git rebase -x "make test" main         # run a command after each commit
git push --force-with-lease            # safe-ish force push
git push --force-with-lease=branch:<expected-sha>
```

Rule: rewrite only unpublished commits, or branches that only you use. Never rebase `main`.

## Undo and recovery

| Goal | Command |
|------|---------|
| Discard working changes to a file | `git restore <f>` |
| Unstage a file | `git restore --staged <f>` |
| Undo last commit, keep changes staged | `git reset --soft HEAD~1` |
| Undo last commit, keep changes unstaged | `git reset HEAD~1` (mixed) |
| Undo last commit and changes (destructive) | `git reset --hard HEAD~1` |
| Undo a pushed commit | `git revert <sha>` (merge commit: `-m 1`) |
| Revert a range | `git revert A..B` |
| Remove untracked files (dry run first) | `git clean -nd` then `git clean -fd` |
| Take one file from another commit | `git restore --source=<sha> <f>` |
| Abort a stuck operation | `git merge --abort`, `rebase --abort`, `cherry-pick --abort` |

```bash
# Reflog: your safety net (local only, ~90 days reachable / 30 unreachable)
git reflog
git reflog show origin/main
git branch rescue 'HEAD@{2}'              # resurrect a lost state
git reset --hard ORIG_HEAD                # undo a merge/rebase/pull just done
git fsck --lost-found                     # dangling commits after reflog expiry

# Recover a deleted branch
git reflog | grep 'checkout: moving from deleted-branch'
git branch deleted-branch <sha>

# Recover after a bad force-push
git fetch origin <old-sha>                # if the server still has it
git push --force-with-lease=main:<current-remote-sha> origin <old-sha>:main
```

## Remotes

```bash
git remote -v
git remote add origin git@github.com:org/repo.git
git remote set-url origin git@github.com:org/repo.git
git remote add upstream https://github.com/org/repo.git      # fork workflow
git fetch --all --prune
git push -u origin feat/x               # set upstream
git push origin HEAD                    # push current branch by name
git pull --rebase
git branch --set-upstream-to=origin/main main
git remote prune origin
```

| Fork sync | Commands |
|-----------|----------|
| Update fork's main | `git fetch upstream && git switch main && git merge --ff-only upstream/main && git push origin main` |
| GitHub CLI shortcut | `gh repo sync` |

## Tags and releases

```bash
git tag -a v1.4.0 -m "v1.4.0"            # annotated (use for releases)
git tag -s v1.4.0 -m "v1.4.0"            # signed
git tag -v v1.4.0                        # verify signature
git tag --list 'v1.*' --sort=-v:refname
git push origin v1.4.0                   # push one tag
git push origin --tags                   # push all tags
git tag -d v1.4.0 && git push origin :refs/tags/v1.4.0
git checkout -b hotfix/1.4.x v1.4.0
git cat-file -t v1.4.0                   # tag (annotated) or commit (lightweight)
```

| SemVer bump | When |
|-------------|------|
| MAJOR `2.0.0` | Breaking public API change |
| MINOR `1.5.0` | Backward-compatible feature |
| PATCH `1.4.1` | Backward-compatible bug fix |
| Pre-release | `1.5.0-rc.1`, `1.5.0-beta.2` (sorts before `1.5.0`) |
| Build metadata | `1.5.0+build.7` (ignored in precedence) |

Conventional Commits mapping: `fix:` is PATCH, `feat:` is MINOR, `feat!:` or a `BREAKING CHANGE:` footer is MAJOR.

## Stash, worktree, bisect

```bash
git stash push -u -m "wip: login"       # include untracked
git stash list
git stash show -p stash@{0}
git stash pop | apply | drop
git stash branch new-branch stash@{0}

git worktree add ../hotfix hotfix/1.4.x
git worktree list
git worktree remove ../hotfix

git bisect start
git bisect bad                          # current is broken
git bisect good v1.3.0
git bisect run ./test.sh                # exit 0 good, 1-127 bad, 125 skip
git bisect reset

git cherry-pick -x <sha>                # record origin in message
git cherry-pick -m 1 <merge-sha>
git cherry -v main feature              # which commits are not upstream yet
```

## Large repos

```bash
git clone --depth 1 <url>                              # shallow, for CI
git clone --filter=blob:none <url>                     # partial clone, blobs on demand
git clone --filter=blob:none --sparse <url>
git sparse-checkout set services/a infra/
git fetch --unshallow                                  # make a shallow clone full
git maintenance start                                  # background gc, commit-graph
git count-objects -vH
git gc --prune=now
git lfs install && git lfs track "*.psd" && git add .gitattributes
git lfs ls-files
git filter-repo --strip-blobs-bigger-than 10M          # rewrites history, use a fresh clone
```

## Hooks, secrets, signing

```bash
# pre-commit framework
pip install pre-commit
pre-commit install --install-hooks
pre-commit install --hook-type commit-msg
pre-commit run --all-files
pre-commit autoupdate

# gitleaks
gitleaks detect --source . -v               # scan history
gitleaks detect --no-git --source .         # scan working files only
gitleaks protect --staged -v                # staged changes (v8; newer: gitleaks git --staged)

# bypass (and why CI must re-check)
git commit --no-verify
git push --no-verify

# share hooks through the repo
git config core.hooksPath .githooks

# remove a secret from history (rotate FIRST)
git filter-repo --replace-text replacements.txt
git filter-repo --path secrets.env --invert-paths
```

SSH commit signing:

```bash
ssh-keygen -t ed25519 -f ~/.ssh/id_signing -N ""
git config --global gpg.format ssh
git config --global user.signingkey ~/.ssh/id_signing.pub
git config --global commit.gpgsign true
git config --global gpg.ssh.allowedSignersFile ~/.ssh/allowed_signers
gh ssh-key add ~/.ssh/id_signing.pub --type signing
git log --show-signature -1
git log --format='%h %G? %GS'               # G good, B bad, U unknown validity, N none
```

| Hook | Runs | Typical use |
|------|------|-------------|
| `pre-commit` | before commit is created | lint, format, secret scan |
| `commit-msg` | after message is written | enforce Conventional Commits |
| `pre-push` | before push | run fast tests, block protected branch |
| `post-merge` | after merge/pull | reinstall deps |
| Server-side (`pre-receive`) | on the server | real enforcement (on GitHub: rulesets and branch protection) |

## GitHub CLI and API

```bash
gh auth login && gh auth status
gh repo create org/name --private --clone
gh pr create --fill --base main
gh pr list --author "@me"
gh pr checks --watch
gh pr review --approve
gh pr merge --squash --delete-branch
gh pr merge --auto --squash                 # merge when checks pass
gh pr checkout 123
gh issue create --title "Bug" --body "..."
gh run list && gh run watch && gh run view --log-failed
gh release create v1.4.0 --generate-notes --verify-tag
gh workflow run ci.yml --ref main
gh api repos/OWNER/REPO/branches/main/protection
gh api -X PUT repos/OWNER/REPO/branches/main/protection --input policy.json
gh api repos/OWNER/REPO/events --jq '.[] | select(.type=="PushEvent")'
```

## Config files

`.github/CODEOWNERS` (last match wins):

```
*                       @org/engineering
/terraform/             @org/platform
/.github/               @org/platform
*.sql                   @org/data
docs/**                 @org/docs
```

`.github/pull_request_template.md`:

```markdown
## What and why
## How it was tested
## Risk and rollback
- [ ] Safe to revert
- [ ] No secrets added
```

`.pre-commit-config.yaml`:

```yaml
repos:
  - repo: https://github.com/gitleaks/gitleaks
    rev: v8.24.0
    hooks:
      - id: gitleaks
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v5.0.0
    hooks:
      - id: detect-private-key
      - id: check-added-large-files
```

`.gitattributes` and `.gitignore` essentials:

```
* text=auto eol=lf
*.png binary
*.psd filter=lfs diff=lfs merge=lfs -text
```

```
.env
*.pem
node_modules/
__pycache__/
```

Release workflow trigger:

```yaml
on:
  push:
    tags: ["v[0-9]+.[0-9]+.[0-9]+"]
permissions:
  contents: write
```

Branch protection / ruleset checklist for `main`:

| Setting | Why |
|---------|-----|
| Require a pull request, 1+ approvals | No unreviewed code |
| Require review from Code Owners | Sensitive paths need their owners |
| Dismiss stale approvals, require last-push approval | A late push cannot ride an old approval |
| Require status checks, up to date with base | Tests ran on what will actually merge |
| Require conversation resolution | Review comments are not ignored |
| Require signed commits | Authenticity of changes |
| Require linear history | Squash or rebase only |
| Block force-pushes and deletions | Prevents history loss |
| Do not allow bypassing, include admins | Rules apply to everyone |
| Merge queue (`merge_group` trigger) | Serialises merges, tests the combined result |

## Top 15 interview one-liners

1. **Git is a content-addressable object store**: blobs, trees, commits and tags are named by the hash of their content; branches and tags are just refs pointing at commits.
2. **A branch is a 41-byte file** holding a commit hash, which is why branching is instant and cheap.
3. **`git pull` is `git fetch` plus `merge` or `rebase`**; fetch never touches your working tree.
4. **Working tree, index, repository**: `add` moves working to index, `commit` moves index to repository, `restore` and `reset` move data back.
5. **`reset --soft/--mixed/--hard`** move the branch only, branch plus index, or branch plus index plus working tree. Only `--hard` destroys uncommitted work.
6. **Revert on shared history, reset on private history**: `revert` adds a new inverse commit and is safe after pushing; `reset` rewrites and is not.
7. **Merge preserves history, rebase rewrites it**: never rebase commits others have based work on.
8. **Use `--force-with-lease`, never bare `--force`**: it refuses to overwrite commits you have not seen.
9. **The reflog is the undo log for refs**: lost commits after a bad reset, rebase or force-push are usually still recoverable for weeks.
10. **Squash merge plus linear history** gives one revertable commit per PR, which is what we want for trunk-based delivery.
11. **Trunk-based development** means short-lived branches merged to `main` at least daily, with feature flags hiding unfinished work.
12. **CODEOWNERS plus branch protection** is how you enforce that the platform team reviews infrastructure changes; the rule must be enabled, the file alone does nothing.
13. **Client-side hooks are advice, server-side checks are policy**: `--no-verify` bypasses hooks, so the same checks run in CI as required status checks.
14. **If a secret is committed, rotate first, scrub second**: removing it from history does not un-leak it.
15. **Annotated, signed SemVer tags drive releases**: `MAJOR.MINOR.PATCH`, pushed tags trigger the release workflow, and a published tag is never moved, you ship a new patch instead.

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
