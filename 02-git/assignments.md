# Git & GitHub — Assignments

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

> 25 practical assignments · 🟢 10 Beginner · 🟡 10 Intermediate · 🔴 5 Advanced. Do them in a scratch directory or a throwaway GitHub repo.

[⬅ Back to Git & GitHub](./README.md) · [🏠 Home](../README.md)

## Table of contents

- [🟢 Beginner](#beginner) — Assignments 1–10
- [🟡 Intermediate](#intermediate) — Assignments 11–20
- [🔴 Advanced](#advanced) — Assignments 21–25

---

## Beginner

### Assignment 1. Identity and configuration 🟢

**Objective:** Configure Git properly and understand the three config scopes.

**Tasks**
1. Set `user.name` and `user.email` globally.
2. Set `init.defaultBranch main`, `pull.rebase true` and a default editor.
3. Override `user.email` for one repo only.
4. List every setting with its origin file.

**Expected outcome:** `git config --list --show-origin` shows the global values and the repo-local email override. A commit made in that repo carries the local email.

<details>
<summary>Hints</summary>

`--system`, `--global` and `--local` are the three scopes; local wins over global over system. `git log -1 --format='%an <%ae>'` shows what a commit recorded.
</details>

**Bonus:** Use `includeIf "gitdir:~/work/"` so everything under `~/work/` uses a work email automatically.

### Assignment 2. First repository and commits 🟢

**Objective:** Walk through the init, add, commit cycle and read `git status` correctly.

**Tasks**
1. `git init` a new folder and add three files.
2. Stage two of them, run `git status`, then commit.
3. Modify a committed file, and compare `git diff` with `git diff --staged`.
4. Make five commits with meaningful messages.

**Expected outcome:** `git log --oneline` shows five commits. You can say which area (working tree, index, repository) each line of `git status` describes.

<details>
<summary>Hints</summary>

`git diff` compares working tree to index; `git diff --staged` compares index to HEAD. Use `git add -p` to stage part of a file.
</details>

**Bonus:** Write messages in Conventional Commits style (`feat:`, `fix:`, `docs:`).

### Assignment 3. .gitignore and untracking 🟢

**Objective:** Keep junk and secrets out of the repo, and fix a file that is already tracked.

**Tasks**
1. Create `.env`, `node_modules/` and `build/` and add the right ignore rules.
2. Commit `.env` by mistake, then stop tracking it without deleting the local file.
3. Use `git check-ignore -v` to prove which rule matches a path.
4. Add a global ignore file for editor junk.

**Expected outcome:** `.env` stays on disk but no longer appears in `git ls-files`, and new edits to it do not show in `git status`.

<details>
<summary>Hints</summary>

`git rm --cached .env` removes it from the index only. Ignore rules never affect already tracked files. Global file: `git config --global core.excludesFile ~/.gitignore_global`.
</details>

**Bonus:** Explain why removing `.env` from the latest commit does not remove the secret from history.

### Assignment 4. Branching and merging 🟢

**Objective:** Create, switch, and merge branches, and tell fast-forward from true merges.

**Tasks**
1. Create `feature/login` and add two commits.
2. Merge into `main` with a fast-forward.
3. Create another branch, commit on both it and `main`, then merge to get a merge commit.
4. Delete merged branches and view the graph.

**Expected outcome:** `git log --oneline --graph --all` shows one linear section and one diamond. `git branch --merged` lists only branches you can safely delete.

<details>
<summary>Hints</summary>

`git switch -c name` creates and switches. `git merge --no-ff` forces a merge commit even when a fast-forward is possible.
</details>

**Bonus:** Draw the same history from memory, labelling HEAD and each branch pointer.

### Assignment 5. Resolve a merge conflict 🟢

**Objective:** Produce and resolve a conflict deliberately.

**Tasks**
1. On two branches, change the same line of one file differently.
2. Merge and read the conflict markers.
3. Resolve by hand, stage, and finish the merge.
4. Repeat, but abort the merge with `git merge --abort`.

**Expected outcome:** A merge commit with the resolved content and no `<<<<<<<` markers left (check with `git grep -n '<<<<<<<'`).

<details>
<summary>Hints</summary>

`git status` lists "both modified" files. `git checkout --ours file` / `--theirs file` takes a whole side. `git config merge.conflictStyle zdiff3` shows the common ancestor too.
</details>

**Bonus:** Enable `rerere` and show Git replaying your resolution the second time.

### Assignment 6. Undo things safely 🟢

**Objective:** Pick the right undo command for each situation.

**Tasks**
1. Discard an uncommitted edit to one file.
2. Unstage a file you added by mistake.
3. Amend the last commit to fix a typo in its message and add a forgotten file.
4. Undo a commit that is already pushed without rewriting history.

**Expected outcome:** Four scenarios solved with `restore`, `restore --staged`, `commit --amend` and `revert` respectively. History for the pushed case contains a new revert commit.

<details>
<summary>Hints</summary>

`git restore file`, `git restore --staged file`, `git revert <sha>`. Never amend a commit that others have pulled.
</details>

**Bonus:** Write a table of when to use `reset --soft`, `--mixed`, `--hard` and `revert`.

### Assignment 7. Remotes, SSH, and push/pull 🟢

**Objective:** Connect a local repo to GitHub over SSH.

**Tasks**
1. Generate an ed25519 key and add it to GitHub.
2. Test with `ssh -T git@github.com`.
3. Create a repo, add `origin`, and push with upstream tracking.
4. Clone it a second time to another folder, push from one, and `git pull` in the other.

**Expected outcome:** Both clones converge. `git branch -vv` shows `[origin/main]` tracking.

<details>
<summary>Hints</summary>

`git remote add origin git@github.com:USER/REPO.git`, `git push -u origin main`. Success message from ssh: "Hi USER! You've successfully authenticated".
</details>

**Bonus:** Explain what `git fetch` changed in the second clone before you pulled (`git log HEAD..origin/main`).

### Assignment 8. Stash and context switching 🟢

**Objective:** Park unfinished work cleanly.

**Tasks**
1. Edit two files, one staged and one not.
2. Stash with a message, including untracked files.
3. Switch branch, make a hotfix commit, switch back.
4. Pop the stash and handle any conflict.

**Expected outcome:** Working tree restored exactly, and `git stash list` is empty after the pop.

<details>
<summary>Hints</summary>

`git stash push -u -m "wip: login form"`, `git stash list`, `git stash pop` (or `apply` to keep it).
</details>

**Bonus:** Create a branch directly from a stash with `git stash branch`.

### Assignment 9. Tags and releases 🟢

**Objective:** Understand lightweight versus annotated tags.

**Tasks**
1. Create a lightweight tag and an annotated tag on different commits.
2. Show the difference with `git cat-file -t`.
3. Push one tag, then all tags, then delete a tag locally and remotely.
4. Create a GitHub release from a tag with `gh release create`.

**Expected outcome:** `git cat-file -t` prints `commit` for the lightweight tag and `tag` for the annotated one. The release appears under the repo's Releases page.

<details>
<summary>Hints</summary>

`git tag -a v0.1.0 -m "msg"`, `git push origin v0.1.0`, `git push origin :refs/tags/v0.1.0` (or `--delete`).
</details>

**Bonus:** Use `git describe --tags` to name an untagged commit and explain the suffix `-3-gabc1234`.

### Assignment 10. Your first pull request 🟢

**Objective:** Run the basic GitHub collaboration loop.

**Tasks**
1. Fork a practice repo (or use your own with a second branch).
2. Commit on a branch, push, and open a PR with `gh pr create`.
3. Leave yourself a review comment and resolve it.
4. Merge with squash and delete the branch.

**Expected outcome:** One squashed commit on `main`, closed PR, branch gone on both remote and local (`git fetch --prune`).

<details>
<summary>Hints</summary>

`gh pr create --fill`, `gh pr view --web`, `gh pr merge --squash --delete-branch`.
</details>

**Bonus:** Link an issue with `Closes #N` in the PR body and confirm it auto-closes.

---

## Intermediate

### Assignment 11. Interactive rebase cleanup 🟡

**Objective:** Turn messy work-in-progress commits into a reviewable series.

**Tasks**
1. Make eight commits including `wip`, `fix typo` and `oops`.
2. Run `git rebase -i HEAD~8` and squash, fixup, reword and reorder to leave three clean commits.
3. Use `git commit --fixup <sha>` and `git rebase -i --autosquash` for one of the fixes.
4. Push the cleaned branch with `--force-with-lease`.

**Expected outcome:** Three commits with clear messages, tests passing at each commit (`git rebase -x "pytest -q" main`).

<details>
<summary>Hints</summary>

In the todo list: `pick`, `reword`, `squash`, `fixup`, `drop`. If it goes wrong, `git rebase --abort` or recover from `git reflog`.
</details>

**Bonus:** Set `rebase.autoSquash true` and `rebase.updateRefs true` globally.

### Assignment 12. Rebase vs merge on a long branch 🟡

**Objective:** Experience both integration styles and compare the history.

**Tasks**
1. Create `feature/x` with 4 commits while `main` gets 3 unrelated commits.
2. Copy the branch. Integrate one copy with `git merge main`, the other with `git rebase main`.
3. Compare graphs and the number of conflicts you had to resolve.
4. Write five lines on which you would use for a shared branch and why.

**Expected outcome:** Two graphs with the same final tree (`git diff copyA copyB` is empty) but different histories.

<details>
<summary>Hints</summary>

`git log --oneline --graph --all`. Rebase rewrites each commit and can ask you to resolve the same conflict repeatedly; merge resolves once.
</details>

**Bonus:** Try `git pull --rebase` and `git pull --ff-only` and describe when each fails.

### Assignment 13. Cherry-pick a hotfix across release branches 🟡

**Objective:** Backport a fix.

**Tasks**
1. Create `release/1.0` from a tag, and fix a bug on `main`.
2. Cherry-pick the fix to `release/1.0` using `-x`.
3. Cherry-pick a commit that conflicts and resolve it with `--continue`.
4. Tag `v1.0.1`.

**Expected outcome:** The backported commit message contains `(cherry picked from commit <sha>)`, and `v1.0.1` points at the release branch tip.

<details>
<summary>Hints</summary>

`git cherry-pick -x <sha>`. For a range: `git cherry-pick A^..B`. For a merge commit you need `-m 1`.
</details>

**Bonus:** Use `git branch --contains <sha>` and `git cherry -v` to prove which branches have the fix.

### Assignment 14. Find a regression with bisect 🟡

**Objective:** Locate a breaking commit by binary search.

**Tasks**
1. Generate a 40-commit history where a test starts failing at commit 27 (script it).
2. Run `git bisect` manually, marking good and bad.
3. Repeat automatically with `git bisect run`.
4. Reset and report the culprit commit.

**Expected outcome:** Bisect identifies commit 27 in at most 6 steps.

<details>
<summary>Hints</summary>

`git bisect start`, `git bisect bad HEAD`, `git bisect good <sha>`, `git bisect run pytest -q`. Exit code 125 means "skip this commit". Finish with `git bisect reset`.
</details>

**Bonus:** Explain why `log2(40)` is about 5.3 and what happens when a commit does not build.

### Assignment 15. CODEOWNERS and required reviews 🟡

**Objective:** Enforce ownership with GitHub.

**Tasks**
1. Create `/terraform/`, `/app/` and `/.github/` directories.
2. Write `.github/CODEOWNERS` with different owners per path, including one team or second account.
3. Turn on "Require review from Code Owners" in branch protection.
4. Open PRs touching each path and observe who is auto-requested.

**Expected outcome:** A PR editing `/terraform/` cannot merge until the owner approves, while a docs-only PR is not blocked by it.

<details>
<summary>Hints</summary>

Last matching pattern wins. Owners need write access. Validate with the "Code owners" errors banner on the file view, or `gh api repos/OWNER/REPO/codeowners/errors`.
</details>

**Bonus:** Add a catch-all `*` owner and explain why it can accidentally make every PR slow.

### Assignment 16. Branch protection as code 🟡

**Objective:** Configure protection through the API instead of clicking.

**Tasks**
1. Write a JSON policy: required PR review, one required status check, linear history, no force-push, no deletions, enforce for admins.
2. Apply it with `gh api -X PUT repos/OWNER/REPO/branches/main/protection --input policy.json`.
3. Read it back with `gh api` and `--jq`.
4. Try to violate each rule and record the error text.

**Expected outcome:** A table of rule, attempt, and the exact `GH006` or merge-box message.

<details>
<summary>Hints</summary>

`restrictions` must be present (null is allowed on personal repos). Contexts are check run names, not workflow names. Requires admin rights on the repo.
</details>

**Bonus:** Recreate the same policy as a repository ruleset and list two things rulesets can do that classic protection cannot.

### Assignment 17. Pre-commit with gitleaks 🟡

**Objective:** Block secrets before they reach history.

**Tasks**
1. Install `pre-commit` and add a config with `gitleaks`, `detect-private-key` and `check-added-large-files`.
2. Try to commit a fake AWS-style key and a PEM header.
3. Add a `.gitleaksignore` entry for a documented false positive, with a comment.
4. Add the same hooks to a GitHub Actions job so `--no-verify` cannot bypass them.

**Expected outcome:** Commit fails locally with a gitleaks finding; the PR check fails in CI when the hook is skipped.

<details>
<summary>Hints</summary>

`pre-commit install`, `pre-commit run --all-files`, `gitleaks detect --source . -v` scans history, `gitleaks protect --staged -v` scans staged changes (newer versions: `gitleaks git --staged`). CI needs `fetch-depth: 0`.
</details>

**Bonus:** Write a custom rule in `.gitleaks.toml` for an internal token format like `acme_[a-z0-9]{32}`.

### Assignment 18. Signed commits and tags 🟡

**Objective:** Produce Verified commits with SSH signing.

**Tasks**
1. Create an ed25519 key and register it on GitHub as a signing key.
2. Configure `gpg.format ssh`, `user.signingkey`, `commit.gpgsign`.
3. Sign a commit and a tag, verify both locally and on GitHub.
4. Enable "Require signed commits" on a branch and show an unsigned push being rejected.

**Expected outcome:** Verified badge on GitHub, `git log --show-signature` showing `Good "git" signature`, and a rejection message for the unsigned push.

<details>
<summary>Hints</summary>

Local verification needs `gpg.ssh.allowedSignersFile`. The commit email must be a verified email on the account. `git log --format='%h %G? %GS'`.
</details>

**Bonus:** Repeat with GPG and list two operational differences (key expiry/revocation, tooling).

### Assignment 19. SemVer release automation 🟡

**Objective:** Release by pushing a tag.

**Tasks**
1. Adopt Conventional Commits for ten commits (feat, fix, one with `!` breaking).
2. Decide the next version by hand from those commits.
3. Write a workflow triggered by `push: tags: ['v*.*.*']` that creates a GitHub release with generated notes.
4. Tag, push, and verify the release.

**Expected outcome:** A release page whose notes list the merged PRs, and a version number you can justify from the commit types.

<details>
<summary>Hints</summary>

`gh release create "$GITHUB_REF_NAME" --generate-notes` needs `permissions: contents: write`. Breaking change means major bump (or minor while below 1.0.0 by convention).
</details>

**Bonus:** Replace the manual decision with `release-please` or `semantic-release` in a PR-driven flow.

### Assignment 20. Worktrees and sparse monorepo checkouts 🟡

**Objective:** Work on two branches simultaneously and check out only part of a big repo.

**Tasks**
1. Add a second working tree for a hotfix branch with `git worktree add`.
2. Build in both without stashing.
3. Create a sample monorepo with `services/a`, `services/b`, `infra/`, and clone it with `--filter=blob:none --sparse`.
4. Restrict the checkout to `services/a` with `git sparse-checkout set`.

**Expected outcome:** Two directories with different branches sharing one `.git` object store; the sparse clone shows only `services/a` and top-level files.

<details>
<summary>Hints</summary>

`git worktree list`, `git worktree remove ../hotfix`. A branch cannot be checked out in two worktrees at once. `git sparse-checkout list`.
</details>

**Bonus:** Compare clone size and time for full, `--depth 1`, and `--filter=blob:none` clones.

---

## Advanced

### Assignment 21. Recover from a force-push disaster 🔴

**Objective:** Restore a shared branch after history was overwritten.

**Tasks**
1. Push five commits to an unprotected branch from clone A.
2. From clone B, `git reset --hard HEAD~5` and `git push --force`.
3. Recover from clone A's reflog and restore the remote with `--force-with-lease` pinned to the current remote value.
4. Prove recovery via the GitHub events API as an alternative path when no clone has the commits.
5. Add protection so the same mistake is rejected next time.

**Expected outcome:** Remote branch tip equals the original SHA; `GH006` shows up when you retry on the protected branch.

<details>
<summary>Hints</summary>

`git reflog`, `git reflog show origin/<branch>`, `gh api repos/OWNER/REPO/events`. Reflog entries expire (90 days reachable, 30 unreachable by default).
</details>

**Bonus:** Write a short runbook: first five minutes after discovering a force-push.

### Assignment 22. Purge a leaked secret from history 🔴

**Objective:** Remediate properly, in the right order.

**Tasks**
1. Commit a fake credential three commits deep in a repo with a few branches and a tag.
2. Rotate (simulate: document the steps for a real AWS key) before touching history.
3. Remove the file from all history with `git filter-repo --path secrets.txt --invert-paths`, or replace the string with `--replace-text`.
4. Force-push all branches and tags, and describe what every collaborator must do.
5. Explain what remains exposed (forks, PR refs, caches, clones).

**Expected outcome:** `git log --all -S'FAKESECRET'` returns nothing in the cleaned repo, and a written checklist that puts rotation first.

<details>
<summary>Hints</summary>

Run `filter-repo` on a fresh clone; it removes `origin` on purpose. Use `git push --force --all` and `git push --force --tags`. GitHub keeps `refs/pull/*` and cached views, so contact Support for sensitive data removal.
</details>

**Bonus:** Turn on secret scanning with push protection and show a push being blocked.

### Assignment 23. Design a branching and release strategy for 30 engineers 🔴

**Objective:** Make and defend an architecture decision.

**Tasks**
1. Write a one-page design for a monorepo with 6 services, daily deploys, and one service that must support the previous major version.
2. Choose trunk-based or release branches per service and justify it.
3. Specify branch protection or rulesets, required checks, merge method, CODEOWNERS layout, tag format for multiple services, and hotfix flow.
4. Include a Mermaid diagram of the flow and a table of risks with mitigations.

**Expected outcome:** A design a reviewer can poke at: it states trade-offs (review latency, flag debt, backport cost) and has an explicit rollback path.

<details>
<summary>Hints</summary>

Per-service tags like `payments/v2.4.1` work well in monorepos. Use path filters in workflows so a docs change does not run every service's pipeline. Feature flags are what make trunk-based safe.
</details>

**Bonus:** Add merge queue (`merge_group` trigger) and explain the problem it solves.

### Assignment 24. Repair a corrupted or oversized repo 🔴

**Objective:** Diagnose and fix repository health issues.

**Tasks**
1. Create a repo and commit a 100 MB random file, then delete it in the next commit.
2. Show that `.git` is still large with `git count-objects -vH`.
3. Find the biggest blobs in history.
4. Remove them with `git filter-repo --strip-blobs-bigger-than 10M`, then `git gc --prune=now --aggressive`.
5. Move large binaries to Git LFS going forward and verify with `git lfs ls-files`.

**Expected outcome:** `.git` shrinks by about the file size; `git fsck --full` reports no problems.

<details>
<summary>Hints</summary>

Biggest objects: `git rev-list --objects --all | git cat-file --batch-check='%(objecttype) %(objectname) %(objectsize) %(rest)' | sort -k3 -n -r | head`. Run on a fresh clone. Also try `git maintenance start` for background upkeep.
</details>

**Bonus:** Simulate a missing object by deleting a loose object file and recover it from another clone with `git fetch`.

### Assignment 25. Git hooks and policy enforcement across an org 🔴

**Objective:** Build layered guardrails that a developer cannot casually bypass.

**Tasks**
1. Write a local `commit-msg` hook enforcing Conventional Commits and a `pre-push` hook that blocks pushes to `main`.
2. Show both being bypassed with `--no-verify`.
3. Re-implement the policy server-side: a required workflow check for commit messages and a ruleset (or protection rule) for the branch.
4. Distribute the hooks with `pre-commit` and a shared `core.hooksPath`, and document how a new engineer is onboarded in under five minutes.
5. Add a required signed-commit rule and a tag ruleset that restricts who can create `v*` tags.

**Expected outcome:** Local hooks give fast feedback, server rules are the actual enforcement, and a table maps each policy to the layer that enforces it.

<details>
<summary>Hints</summary>

`git config core.hooksPath .githooks` shares hooks via the repo, but it is still client-side and optional. Rulesets support tag targets and bypass lists. Remember `GITHUB_TOKEN` cannot trigger further workflows from events it creates.
</details>

**Bonus:** Add a `merge_group`-aware required check and demonstrate the workflow running in a merge queue.

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
