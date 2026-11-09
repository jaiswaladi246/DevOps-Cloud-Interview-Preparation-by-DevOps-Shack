# Git — Core Concepts

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

> Git internals, branching, merging, undo, remotes, hooks, tooling and GitHub fundamentals explained for interviews · 200 questions · 🟢 Beginner 🟡 Intermediate 🔴 Advanced

[⬅ Back to Git](./README.md) · [🏠 Home](../README.md)

## Table of contents
- [Git internals: objects, refs and the index](#git-internals-objects-refs-and-the-index) — Q1–Q25
- [Commits, branches and tags](#commits-branches-and-tags) — Q26–Q57
- [Merge, rebase and cherry-pick](#merge-rebase-and-cherry-pick) — Q58–Q81
- [Undoing changes: reset, revert, restore and reflog](#undoing-changes-reset-revert-restore-and-reflog) — Q82–Q103
- [Conflicts, remotes, fetch, pull and push](#conflicts-remotes-fetch-pull-and-push) — Q104–Q127
- [Branching strategies, pull requests and code review](#branching-strategies-pull-requests-and-code-review) — Q128–Q148
- [Hooks, submodules, LFS and signing](#hooks-submodules-lfs-and-signing) — Q149–Q174
- [GitHub features, history rewriting and scale](#github-features-history-rewriting-and-scale) — Q175–Q200

---

## Git internals: objects, refs and the index

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q1. What are the four object types in Git's object database? 🟢

<details>
<summary><b>Answer</b></summary>


Blob, tree, commit and annotated tag. A blob stores file contents only (no name, no mode). A tree maps names and modes to blobs and sub-trees, so it represents a directory. A commit points to one tree, zero or more parent commits, and carries author, committer and message. An annotated tag points to another object and carries a tagger and message.

```bash
git cat-file -t HEAD        # commit
git cat-file -p HEAD^{tree} # list the root tree
```

</details>


### Q2. Is Git a snapshot system or a diff system? 🟢

<details>
<summary><b>Answer</b></summary>


Conceptually Git stores snapshots: every commit points to a complete tree describing the whole project at that moment. Diffs are computed on demand by comparing two trees. Packfiles do use delta compression for storage efficiency, but that is an on-disk optimisation and invisible to the data model.

</details>


### Q3. What is the SHA-1 (or SHA-256) hash of an object and how is it computed? 🟢

<details>
<summary><b>Answer</b></summary>


The object ID is the hash of a header plus the content: `<type> <size>\0<content>`. Identical content therefore always yields the same ID, which gives free deduplication and integrity checking. You can reproduce a blob ID by hand.

```bash
echo 'hello' | git hash-object --stdin
printf 'blob 6\0hello\n' | sha1sum   # same value
```

</details>


### Q4. Where does Git store objects on disk, and what are loose objects versus packfiles? 🟡

<details>
<summary><b>Answer</b></summary>


Under `.git/objects`. A loose object is one zlib-compressed file at `objects/ab/cdef...` named by its ID. When objects pile up, `git gc` (or `git repack`) bundles them into `objects/pack/*.pack` with an `.idx` index, using delta compression between similar objects. Fetch and push also transfer packfiles.

```bash
git count-objects -vH
ls .git/objects/pack
```

</details>


### Q5. What exactly is a branch, and what does creating one cost? 🟡

<details>
<summary><b>Answer</b></summary>


A branch is a file containing a 40-hex commit ID, stored at `.git/refs/heads/<name>` (or in `packed-refs`). Creating one writes 41 bytes, so it is effectively free. Committing on a branch just moves that pointer forward to the new commit.

```bash
cat .git/refs/heads/main
git rev-parse main
```

</details>


### Q6. What is HEAD? 🟢

<details>
<summary><b>Answer</b></summary>


HEAD is a reference to your current position, normally a symbolic ref pointing at a branch such as `ref: refs/heads/main`. When you commit, Git creates the commit and advances the branch HEAD points to. `git switch` rewrites HEAD to point at another branch.

```bash
cat .git/HEAD
git symbolic-ref HEAD
```

</details>


### Q7. What is a detached HEAD and how do you recover work done in it? 🟡

<details>
<summary><b>Answer</b></summary>


HEAD contains a raw commit ID instead of a branch name, which happens when you check out a tag, a commit or a remote-tracking branch. Commits made there are not reachable from any branch and can be garbage collected eventually. To keep them, create a branch before leaving.

```bash
git switch -c rescue            # while still detached
git reflog                      # if you already moved away
git branch rescue <sha-from-reflog>
```

</details>


### Q8. What does the staging area (index) do? 🟢

<details>
<summary><b>Answer</b></summary>


The index is the proposed next commit. `git add` copies the current content of files into object storage and records their blob IDs in `.git/index`; `git commit` builds a tree from the index. It lets you commit part of your working changes with `git add -p` and review exactly what will be committed with `git diff --staged`.

</details>


### Q9. What information does `.git/index` hold for each file? 🟡

<details>
<summary><b>Answer</b></summary>


Per path it stores the mode, blob ID, stage number (0 normally, 1-3 during a conflict), flags, and cached filesystem metadata (mtime, ctime, size, inode). The cached stat data lets `git status` skip hashing files whose metadata has not changed. Inspect it with plumbing.

```bash
git ls-files --stage
git ls-files -u   # unmerged entries only
```

</details>


### Q10. What are the three trees Git compares in `status`, `diff` and `reset`? 🟡

<details>
<summary><b>Answer</b></summary>


HEAD (last commit), the index (staging area) and the working tree. `git diff` compares working tree to index, `git diff --staged` compares index to HEAD, and `git status` shows both. `reset` and `restore` choose which of the three to overwrite, which is the cleanest mental model for undo commands.

</details>


### Q11. Walk through what `git commit` does at the object level. 🟡

<details>
<summary><b>Answer</b></summary>


Git writes the index as tree objects (one per directory, bottom up), creates a commit object referencing the root tree, the current HEAD commit as parent, plus author/committer and message, then updates the branch ref that HEAD points to. Finally the reflog entries for the branch and HEAD are appended.

```bash
git cat-file -p HEAD
# tree <id>
# parent <id>
# author ... committer ...
```

</details>


### Q12. What is the difference between plumbing and porcelain commands? 🟡

<details>
<summary><b>Answer</b></summary>


Porcelain commands (`commit`, `merge`, `rebase`, `status`) are the user-facing interface with human-oriented output that can change between versions. Plumbing commands (`hash-object`, `cat-file`, `update-ref`, `write-tree`, `rev-parse`) are low level, stable and designed for scripts. In automation prefer plumbing or `--porcelain` output flags.

```bash
git status --porcelain=v2 --branch
```

</details>


### Q13. How does Git guarantee history integrity? 🟡

<details>
<summary><b>Answer</b></summary>


Each commit ID is a hash over its tree ID and parent IDs, so changing any file, message or ancestor changes every descendant ID. This forms a Merkle DAG. Rewriting history is therefore always creating new commits, never mutating old ones, and `git fsck` can verify the whole graph.

```bash
git fsck --full
```

</details>


### Q14. What is a packed ref and why does `packed-refs` exist? 🟡

<details>
<summary><b>Answer</b></summary>


Thousands of tiny ref files are slow on some filesystems, so `git pack-refs` (run by `gc`) moves them into the single `.git/packed-refs` file. A loose ref file, when present, overrides the packed entry. Newer Git also supports the `reftable` backend, which stores refs in a compact binary format for large repos.

```bash
git pack-refs --all
git init --ref-format=reftable demo
```

</details>


### Q15. Explain how a tree object is encoded and why renames are not stored. 🔴

<details>
<summary><b>Answer</b></summary>


A tree is a list of entries `<mode> <name>\0<20-byte-binary-id>` sorted by name. Modes include 100644 (file), 100755 (executable), 120000 (symlink), 040000 (tree) and 160000 (gitlink, a submodule). Git records no rename metadata; `git log --follow`, `diff -M` and `blame -C` detect renames heuristically by comparing blob similarity (default threshold 50%).

```bash
git ls-tree -r HEAD
git diff -M --stat HEAD~1
```

</details>


### Q16. How do packfile delta chains work and what controls them? 🔴

<details>
<summary><b>Answer</b></summary>


Inside a pack, an object is either stored whole or as a delta against another object (OFS_DELTA by offset or REF_DELTA by ID). Git sorts candidates by type, name and size to find good bases, and chains can be up to `pack.depth` deep (default 50) within a `pack.window` of candidates (default 10). Deeper or wider settings shrink the pack at the cost of CPU on repack and slower reads.

```bash
git repack -adf --depth=250 --window=250
git verify-pack -v .git/objects/pack/*.idx | head
```

</details>


### Q17. What does `git gc` do, and when does it run automatically? 🟡

<details>
<summary><b>Answer</b></summary>


It packs loose objects, packs refs, prunes unreachable objects older than `gc.pruneExpire` (2 weeks), and expires old reflog entries. `git gc --auto` runs after commands like commit and merge once there are more than about 6700 loose objects or 50 packs. Large teams usually switch to `git maintenance` for scheduled background tasks.

```bash
git maintenance start
git gc --prune=now --aggressive   # rarely needed
```

</details>


### Q18. What is the reflog and how is it different from `git log`? 🟡

<details>
<summary><b>Answer</b></summary>


The reflog is a local, per-ref journal of where a ref pointed over time (checkouts, commits, resets, rebases). `git log` walks the commit graph from a ref; reflog records movement of the ref itself, so it still lists commits that are no longer reachable. Entries expire after 90 days (30 for unreachable) by default and are never pushed.

```bash
git reflog show HEAD
git log -g --oneline
```

</details>


### Q19. What does a revision like `HEAD~2`, `HEAD^2`, `main@{yesterday}` or `:/fix` mean? 🟡

<details>
<summary><b>Answer</b></summary>


`~N` follows first parents N times. `^N` selects the Nth parent of a merge commit (`HEAD^2` is the merged-in side). `@{time}` and `@{N}` use the reflog, so they are local. `:/text` finds the youngest reachable commit whose message matches. `A..B` is commits in B not in A, `A...B` is the symmetric difference.

```bash
git log main..feature
git show :/"fix login"
```

</details>


### Q20. What are refs/heads, refs/tags and refs/remotes? 🟢

<details>
<summary><b>Answer</b></summary>


They are namespaces under `.git/refs`. `refs/heads/*` are your local branches, `refs/tags/*` are tags, and `refs/remotes/<remote>/*` are remote-tracking branches that record where the remote's branches were at last fetch. You never commit onto a remote-tracking ref directly; only fetch updates it.

</details>


### Q21. What is a refspec and how do you read `+refs/heads/*:refs/remotes/origin/*`? 🟡

<details>
<summary><b>Answer</b></summary>


A refspec is `[+]<src>:<dst>`. This default fetch refspec says: take every branch on the remote and store it under `refs/remotes/origin/`, and the leading `+` allows non-fast-forward updates of the tracking ref. Push refspecs work the same way in reverse and can delete refs with an empty source.

```bash
git config --get-all remote.origin.fetch
git push origin :old-branch        # delete remote branch
git push origin HEAD:refs/heads/hotfix
```

</details>


### Q22. What is the commit-graph file and what does it speed up? 🔴

<details>
<summary><b>Answer</b></summary>


`.git/objects/info/commit-graph` caches commit parents, root tree IDs and generation numbers so that `log --graph`, `merge-base`, `branch --contains` and reachability checks do not have to parse commit objects. Generation numbers let traversals stop early. Enable writing it on fetch and via maintenance.

```bash
git commit-graph write --reachable --changed-paths
git config fetch.writeCommitGraph true
```

</details>


### Q23. What are `.git/info/exclude` and `.git/config` for? 🟡

<details>
<summary><b>Answer</b></summary>


`info/exclude` is a per-clone ignore file that is never committed, handy for editor files or local scratch paths. `.git/config` holds repo-level settings (remotes, branch tracking, user identity), overriding the global `~/.gitconfig` and the system file. Precedence is system, then global, then local, then worktree.

```bash
git config --show-origin --list
```

</details>


### Q24. What is the difference between `git worktree` and cloning the repo twice? 🔴

<details>
<summary><b>Answer</b></summary>


A worktree is an additional working directory attached to the same `.git` object store, each with its own HEAD and index. It shares objects and refs, so no extra download or disk for history, and a branch can be checked out in only one worktree at a time. It is ideal for reviewing a PR or running a hotfix without stashing.

```bash
git worktree add ../hotfix -b hotfix/1.4 origin/release-1.4
git worktree list
git worktree remove ../hotfix
```

</details>


### Q25. What is the difference between SHA-1 and SHA-256 repositories? 🟡

<details>
<summary><b>Answer</b></summary>


SHA-1 is the default and still used by GitHub, because SHA-1 collisions (SHAttered) are detected by hardened `sha1dc` in Git. SHA-256 repos (`git init --object-format=sha256`) use 64-hex IDs and are incompatible with SHA-1 repos at the protocol level, with interoperability still maturing. Most teams stay on SHA-1 today.

## Commits, branches and tags

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

</details>


### Q26. How do you write a good commit message? 🟢

<details>
<summary><b>Answer</b></summary>


A short imperative subject of about 50 characters, a blank line, then a body explaining why the change was needed rather than restating the diff. Reference the ticket at the end. Many teams use Conventional Commits (`feat:`, `fix:`, `chore:`) so changelogs and semantic versions can be generated.

```text
fix(auth): reject expired refresh tokens

Tokens past exp were accepted when clock skew exceeded 5 min.
Closes #482
```

</details>


### Q27. What is the difference between `git add .`, `git add -A` and `git add -u`? 🟢

<details>
<summary><b>Answer</b></summary>


`git add -A` stages new, modified and deleted files across the whole repo. `git add .` stages everything under the current directory (since Git 2.0 it includes deletions). `git add -u` stages modifications and deletions of already tracked files only, ignoring untracked ones.

</details>


### Q28. How do you commit only part of the changes in a file? 🟢

<details>
<summary><b>Answer</b></summary>


Use interactive patch staging. Git splits the diff into hunks and asks per hunk: `y` stage, `n` skip, `s` split, `e` edit manually.

```bash
git add -p path/to/file
git diff --staged
```

</details>


### Q29. How do you change the last commit message or add a forgotten file? 🟢

<details>
<summary><b>Answer</b></summary>


Use `git commit --amend`. It replaces the last commit with a new one, so the ID changes. Only do this if the commit has not been shared, or be ready to force-push with lease.

```bash
git add forgotten.txt
git commit --amend --no-edit
git commit --amend -m "better message"
```

</details>


### Q30. What is the difference between `git switch`, `git restore` and `git checkout`? 🟢

<details>
<summary><b>Answer</b></summary>


`checkout` is the older overloaded command that both changes branches and restores files. Git 2.23 split it: `git switch` only changes branches, `git restore` only restores file content from the index or a commit. Prefer the new ones for clarity.

```bash
git switch -c feature/x
git restore --staged app.py
git restore --source=HEAD~2 app.py
```

</details>


### Q31. What is the difference between lightweight and annotated tags? 🟢

<details>
<summary><b>Answer</b></summary>


A lightweight tag is just a ref to a commit. An annotated tag is a full object with tagger, date, message and optional GPG/SSH signature, and it is what `git describe` and releases should use. Tags are not pushed by default.

```bash
git tag -a v1.4.0 -m "Release 1.4.0"
git push origin v1.4.0
git push origin --tags
```

</details>


### Q32. How does `git describe` work and where is it useful? 🟡

<details>
<summary><b>Answer</b></summary>


It finds the nearest reachable annotated tag and prints `<tag>-<commits-since>-g<short-sha>`, such as `v1.4.0-12-g3fa9c21`. CI uses it to stamp build versions that are traceable to an exact commit. Add `--tags` to include lightweight tags and `--dirty` to flag uncommitted changes.

```bash
git describe --tags --always --dirty
```

</details>


### Q33. How do you delete a branch locally and remotely, and what does `-d` versus `-D` mean? 🟡

<details>
<summary><b>Answer</b></summary>


`-d` refuses to delete a branch that has unmerged commits, `-D` forces it. Remote deletion is a push of an empty source. After that, other clones must prune stale tracking refs.

```bash
git branch -d feature/x
git push origin --delete feature/x
git fetch --prune     # or: git config fetch.prune true
```

</details>


### Q34. How do upstream (tracking) branches work? 🟡

<details>
<summary><b>Answer</b></summary>


A local branch can record `branch.<name>.remote` and `branch.<name>.merge`, which makes bare `git pull`, `git push` and the ahead/behind counts in `git status` work. Set it with `-u` on first push or `--set-upstream-to`. With `push.autoSetupRemote true` the first `git push` does it automatically.

```bash
git push -u origin feature/x
git branch -vv
```

</details>


### Q35. How do you rename a branch, including the default branch? 🟢

<details>
<summary><b>Answer</b></summary>


Rename locally with `git branch -m`, push the new name, and delete the old one remotely. For the default branch also change it in the hosting platform settings and tell everyone to update their local tracking.

```bash
git branch -m master main
git push -u origin main
git push origin --delete master
git remote set-head origin -a
```

</details>


### Q36. How do you find which commit introduced a bug? 🟡

<details>
<summary><b>Answer</b></summary>


Use `git bisect`, which binary-searches history. Mark one bad and one good commit, then test each checkout or let a script decide with its exit code (0 good, 1-124 bad, 125 skip).

```bash
git bisect start
git bisect bad HEAD
git bisect good v1.3.0
git bisect run ./test.sh
git bisect reset
```

</details>


### Q37. What do `git blame` and `git log -S` / `-G` answer? 🟢

<details>
<summary><b>Answer</b></summary>


`git blame` shows who last changed each line. `git log -S'text'` (pickaxe) finds commits where the number of occurrences of a string changed, and `-G'regex'` finds commits whose diff matches a regex. `git log -L` follows a function or line range through history.

```bash
git blame -w -C -L 40,60 app.py
git log -S'MAX_RETRIES' --oneline
git log -L :parse_config:config.py
```

</details>


### Q38. How do you ignore a bulk-reformat commit in blame? 🟡

<details>
<summary><b>Answer</b></summary>


Put the commit SHA in a file and point Git at it. GitHub's blame UI automatically honours a `.git-blame-ignore-revs` file at the repo root.

```bash
echo "<sha>  # prettier reformat" >> .git-blame-ignore-revs
git config blame.ignoreRevsFile .git-blame-ignore-revs
```

</details>


### Q39. What is the difference between author and committer? 🟡

<details>
<summary><b>Answer</b></summary>


The author wrote the change, the committer applied it. They differ after `cherry-pick`, `rebase`, `am` or amend, where the committer and committer date update while author data stays. `git log --pretty=fuller` shows both. GitHub displays co-authors from `Co-authored-by:` trailers.

```bash
git log -1 --pretty=fuller
git commit -m "pair work" -m "Co-authored-by: Asha <asha@example.com>"
```

</details>


### Q40. How do you see what changed between two branches, tags or commits? 🟢

<details>
<summary><b>Answer</b></summary>


Use `git diff` with two refs and choose the range operator deliberately. Two dots compares the tips directly; three dots compares the second tip against the merge base, which matches what a PR shows.

```bash
git diff v1.3.0 v1.4.0 --stat
git diff main...feature
git log --oneline main..feature
```

</details>


### Q41. How do you set committer identity per repository or per directory? 🟡

<details>
<summary><b>Answer</b></summary>


Set `user.name` and `user.email` locally, or use conditional includes in the global config so work and personal repos pick different identities automatically.

```ini
# ~/.gitconfig
[includeIf "gitdir:~/work/"]
    path = ~/.gitconfig-work
```

</details>


### Q42. What is `git clean` and how do you use it safely? 🟢

<details>
<summary><b>Answer</b></summary>


It deletes untracked files. It is destructive and bypasses the reflog, so always dry-run first. `-d` includes directories, `-x` also removes ignored files, `-n` previews.

```bash
git clean -ndx
git clean -fd
```

</details>


### Q43. How do you move or rename files so Git tracks it cleanly? 🟡

<details>
<summary><b>Answer</b></summary>


Use `git mv old new`, which is just `mv` plus staging both sides. Git infers the rename at diff time, so renaming a file and heavily editing it in one commit can break rename detection and hurt blame/history. Do the pure rename in its own commit.

```bash
git mv src/util.py src/helpers.py
git commit -m "refactor: rename util to helpers"
```

</details>


### Q44. What is the difference between `git log --oneline --graph` and `--first-parent`? 🟡

<details>
<summary><b>Answer</b></summary>


`--graph` draws every commit including those from merged branches. `--first-parent` follows only the first parent of each merge, so on a main branch that uses merge commits you see one entry per merged PR. It is the best view of "what landed when" and works well with `bisect --first-parent`.

```bash
git log --oneline --first-parent main
```

</details>


### Q45. What does `git status` output mean: "Your branch is ahead of 'origin/main' by 2 commits"? 🟢

<details>
<summary><b>Answer</b></summary>


It compares your local branch with its remote-tracking ref as of the last fetch. Ahead means you have 2 local commits not yet pushed. Diverged means both sides have unique commits. Run `git fetch` first because the comparison is stale otherwise.

</details>


### Q46. How do you create a branch from a past commit or tag, and from a remote branch? 🟡

<details>
<summary><b>Answer</b></summary>


Pass a start point. For a remote branch, `git switch <name>` auto-creates a tracking branch if exactly one remote has it.

```bash
git switch -c hotfix/1.3.1 v1.3.0
git switch -c review/pr-482 origin/feature/x
git switch feature/x    # DWIM tracking
```

</details>


### Q47. How does `git log` decide commit order, and what do `--topo-order` and `--date-order` change? 🔴

<details>
<summary><b>Answer</b></summary>


By default commits appear in reverse chronological order by commit date as walked from the tips, which can interleave branches oddly when clocks differ. `--topo-order` guarantees children before parents with branches grouped, and `--date-order` shows no parent before its children but otherwise orders by timestamp. `--graph` implies topo order.

</details>


### Q48. What are Git trailers and how are they used? 🟡

<details>
<summary><b>Answer</b></summary>


Trailers are `Key: value` lines at the end of a commit message, such as `Signed-off-by`, `Co-authored-by`, `Reviewed-by` and `Fixes`. `git interpret-trailers` can add and parse them, and `git log --format='%(trailers:key=Fixes)'` extracts them. `Signed-off-by` implements the Developer Certificate of Origin (`commit -s`).

```bash
git commit -s -m "feat: add retries"
```

</details>


### Q49. What is a sparse checkout and how does it differ from a partial clone? 🔴

<details>
<summary><b>Answer</b></summary>


A partial clone (`--filter=blob:none`) limits which objects are downloaded; the rest are fetched lazily from the remote. Sparse checkout limits which paths are materialised in the working tree. They are used together for monorepos, and cone mode is the fast, directory-based pattern set.

```bash
git clone --filter=blob:none --sparse https://github.com/org/mono.git
cd mono && git sparse-checkout set services/payments libs/common
```

</details>


### Q50. What does `git stash` store and where? 🟢

<details>
<summary><b>Answer</b></summary>


It saves your modified tracked files and index state as commits under `refs/stash`, then resets the working tree to HEAD. By default untracked files are skipped; use `-u` to include them. It is a stack, so `stash@{0}` is the newest.

```bash
git stash push -u -m "wip: login form"
git stash list
git stash pop
```

</details>


### Q51. What is the difference between `git stash pop` and `git stash apply`? 🟡

<details>
<summary><b>Answer</b></summary>


`pop` applies the stash and drops it if it applies cleanly. `apply` leaves the stash in the list so you can apply it to more than one branch. If `pop` hits conflicts the stash is kept, which prevents loss.

```bash
git stash apply stash@{2}
git stash drop stash@{2}
git stash branch fix-login stash@{0}
```

</details>


### Q52. What is `.gitignore` and why doesn't it ignore files that are already tracked? 🟢

<details>
<summary><b>Answer</b></summary>


`.gitignore` only affects untracked files. If a file is already in the index, Git keeps tracking it. Remove it from the index while keeping the local copy, then commit.

```bash
git rm --cached .env
echo ".env" >> .gitignore
git commit -m "chore: stop tracking .env"
```

</details>


### Q53. How do you debug why a path is ignored? 🟡

<details>
<summary><b>Answer</b></summary>


Use `git check-ignore -v` which prints the exact file, line and pattern responsible. Remember precedence: later patterns override earlier ones, `!` negates but cannot re-include a file whose parent directory is excluded, and a nested `.gitignore` applies to its subtree.

```bash
git check-ignore -v build/app.js
```

</details>


### Q54. What does `git show` display? 🟢

<details>
<summary><b>Answer</b></summary>


For a commit it prints metadata and the patch; for a tag the tag object then the target; for a tree or blob the listing or content. `git show REV:path` prints a file as it existed at that revision, without checking it out.

```bash
git show HEAD~3:config/app.yaml
git show --stat v1.4.0
```

</details>


### Q55. How do you list all commits that touched a path, including deleted files? 🟡

<details>
<summary><b>Answer</b></summary>


Use `git log -- <path>`; the double dash disambiguates paths from refs and works for deleted paths. Add `--follow` for a single file across renames, and `--diff-filter=D` to find deletion commits.

```bash
git log --oneline --diff-filter=D -- old/module.py
git log --follow -p -- src/helpers.py
```

</details>


### Q56. What is a `.gitattributes` file used for? 🟢

<details>
<summary><b>Answer</b></summary>


It sets per-path behaviour: line-ending normalisation (`text=auto eol=lf`), binary markers, custom diff and merge drivers, LFS filters, and `export-ignore` for `git archive`. Committing it makes behaviour consistent for the whole team.

```text
* text=auto eol=lf
*.png binary
*.psd filter=lfs diff=lfs merge=lfs -text
```

</details>


### Q57. What is `git archive` and why use it instead of zipping the folder? 🟡

<details>
<summary><b>Answer</b></summary>


It exports a tree at any revision as tar or zip without the `.git` directory, honouring `export-ignore` and `export-subst` attributes. It is reproducible and uses committed content only, so no stray local files leak into a release tarball.

```bash
git archive --format=tar.gz --prefix=app-1.4.0/ -o app-1.4.0.tar.gz v1.4.0
```

## Merge, rebase and cherry-pick

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

</details>


### Q58. What is the difference between a fast-forward merge and a merge commit? 🟢

<details>
<summary><b>Answer</b></summary>


A fast-forward happens when the target branch has no new commits since the feature branched, so Git just moves the branch pointer and no new commit is created. Otherwise Git creates a merge commit with two parents. `--no-ff` forces a merge commit even when a fast-forward is possible, preserving the feature's grouping in history.

```bash
git merge --ff-only feature
git merge --no-ff feature
```

</details>


### Q59. How does a three-way merge work? 🟡

<details>
<summary><b>Answer</b></summary>


Git finds the merge base (the best common ancestor), then compares each side against it. A hunk changed on only one side is taken automatically. A hunk changed on both sides differently is a conflict. This is why merging the same branch twice is safe: the new merge base is the previous merge commit.

```bash
git merge-base main feature
```

</details>


### Q60. What does `git rebase` actually do? 🟡

<details>
<summary><b>Answer</b></summary>


It finds commits in your branch that are not in the new base, saves them as patches, resets the branch to the new base, and replays each patch as a brand new commit. The new commits have different IDs even if the content is the same, so rebasing rewrites history. The old commits remain reachable via the reflog (`ORIG_HEAD`).

```bash
git rebase origin/main
```

</details>


### Q61. Merge versus rebase: how do you decide? 🟡

<details>
<summary><b>Answer</b></summary>


Rebase to keep a linear, readable history for your own unpublished work and to update a feature branch with latest main. Merge when you want to preserve the true integration history or when a branch is shared by several people. The golden rule: never rebase commits others have based work on. Many teams combine them: rebase locally, squash-merge on the platform.

</details>


### Q62. What are squash merge, merge commit and rebase-and-merge on GitHub? 🟡

<details>
<summary><b>Answer</b></summary>


Merge commit keeps every commit plus a merge commit. Squash merge collapses the PR into one commit on the base branch, giving one commit per PR but losing granular history and making the branch look unmerged to Git. Rebase and merge replays each commit onto the base without a merge commit, creating new SHAs. Admins can restrict which methods are allowed per repo.

</details>


### Q63. Why does a squash-merged branch show as unmerged to `git branch --merged`? 🟡

<details>
<summary><b>Answer</b></summary>


Squash creates a new commit with no parent link to the feature commits, so the feature tip is not an ancestor of main. Git cannot see it as merged. Delete the branch with `-D` after confirming the PR merged, or use `git cherry` or `git log --cherry` to compare patch IDs, and enable auto-delete of head branches on GitHub.

```bash
git cherry -v main feature
```

</details>


### Q64. What does `git cherry-pick` do? 🟢

<details>
<summary><b>Answer</b></summary>


It applies the change introduced by one or more existing commits onto the current branch as new commits with new IDs. It is the tool for back-porting a fix to a release branch. For merge commits you must choose the mainline parent with `-m`.

```bash
git cherry-pick 3fa9c21
git cherry-pick -x A^..B     # range, records origin in message
git cherry-pick -m 1 <merge-sha>
```

</details>


### Q65. What does `cherry-pick -x` add and why is it useful? 🟡

<details>
<summary><b>Answer</b></summary>


It appends `(cherry picked from commit <sha>)` to the message. On a maintenance branch this gives auditors and future engineers a pointer back to the original fix and helps tools detect duplicates.

</details>


### Q66. What is interactive rebase and what can you do with it? 🟡

<details>
<summary><b>Answer</b></summary>


`git rebase -i <base>` opens a todo list where each line is a command: `pick`, `reword`, `edit`, `squash`, `fixup`, `drop`, and you can reorder lines. It is used to clean a branch before opening a PR.

```bash
git rebase -i HEAD~5
git commit --fixup=<sha>
git rebase -i --autosquash origin/main
```

</details>


### Q67. What is the difference between `squash` and `fixup` in a rebase todo? 🟡

<details>
<summary><b>Answer</b></summary>


Both fold a commit into the previous one. `squash` combines the messages and opens the editor so you can edit the result; `fixup` discards the folded commit's message. `fixup -C` instead keeps only the later message.

</details>


### Q68. What does `git rebase --onto` do and when do you need it? 🔴

<details>
<summary><b>Answer</b></summary>


It transplants a range of commits to a new base: `git rebase --onto <newbase> <oldbase> <branch>`. Typical use: you branched `topic-b` off `topic-a`, then `topic-a` was squash-merged, so you must move only `topic-b`'s own commits onto main.

```bash
git rebase --onto main topic-a topic-b
```

</details>


### Q69. What does `git rebase --rebase-merges` change? 🔴

<details>
<summary><b>Answer</b></summary>


By default rebase flattens merge commits away. `--rebase-merges` (successor of `--preserve-merges`) recreates the merge topology by replaying merges with `label`, `reset` and `merge` todo commands. Use it when you must rebase a branch that contains meaningful internal merges.

</details>


### Q70. What is `rerere` and when is it worth enabling? 🔴

<details>
<summary><b>Answer</b></summary>


Reuse recorded resolution remembers how you resolved a conflict and automatically applies the same resolution when the same conflict recurs, for example when repeatedly rebasing a long-lived branch or testing merges.

```bash
git config --global rerere.enabled true
git rerere status
```

</details>


### Q71. What are merge strategies and options such as `ort`, `ours` and `-X theirs`? 🟡

<details>
<summary><b>Answer</b></summary>


`ort` is the default strategy (since 2.34, replacing `recursive`) and handles renames well and quickly. `-X ours` and `-X theirs` are options that resolve only conflicting hunks in favour of one side. The `ours` strategy (`-s ours`) is totally different: it records a merge commit but keeps your tree unchanged, discarding the other branch's content.

```bash
git merge -X theirs feature
git merge -s ours obsolete-branch
```

</details>


### Q72. How does `ort` differ from `recursive`? 🔴

<details>
<summary><b>Answer</b></summary>


`ort` (Ostensibly Recursive's Twin) is a rewrite that does not touch the working tree or index until the merge is done, is much faster on big repos, handles directory-rename detection and has fewer edge-case bugs. It still builds a virtual merge base when several merge bases exist (criss-cross merges).

</details>


### Q73. What is a criss-cross merge? 🟡

<details>
<summary><b>Answer</b></summary>


Two branches merge each other in both directions, so there are multiple equally good merge bases. Git merges those bases into a virtual ancestor and uses it. Results can be surprising, and the conflicts shown may include markers from the nested merge.

</details>


### Q74. What does `git pull --rebase` do differently from `git pull`? 🟢

<details>
<summary><b>Answer</b></summary>


`git pull` is `fetch` plus `merge`, producing a merge commit if histories diverged. `git pull --rebase` is `fetch` plus `rebase`, replaying your local commits on top of upstream and keeping history linear. You can make it the default.

```bash
git config --global pull.rebase true
git config --global pull.ff only   # alternative: refuse non-ff pulls
```

</details>


### Q75. How do you abort or continue a merge, rebase or cherry-pick that has conflicts? 🟡

<details>
<summary><b>Answer</b></summary>


Each has `--abort`, `--continue` and (rebase, cherry-pick) `--skip`. The state lives in `.git/rebase-merge`, `.git/MERGE_HEAD` or `.git/CHERRY_PICK_HEAD`. `git status` always tells you which operation is in progress.

```bash
git rebase --abort
git merge --continue
git cherry-pick --skip
```

</details>


### Q76. What is `ORIG_HEAD` and how can it save you? 🔴

<details>
<summary><b>Answer</b></summary>


Git saves the previous tip of HEAD in `ORIG_HEAD` before dangerous operations (merge, rebase, reset, pull). After a bad rebase or merge you can return to the pre-operation state.

```bash
git reset --hard ORIG_HEAD
```

</details>


### Q77. What is `git range-diff` and when would you use it? 🔴

<details>
<summary><b>Answer</b></summary>


It compares two versions of a patch series commit-by-commit, matching them even when rebased or amended. It is excellent for reviewing what changed between iteration 1 and iteration 2 of a force-pushed PR.

```bash
git range-diff origin/main old-feature new-feature
git range-diff main...feature@{1} main...feature
```

</details>


### Q78. Why does a rebase cause the same conflict to appear on many commits? 🟡

<details>
<summary><b>Answer</b></summary>


Rebase replays commits one at a time, so a change that conflicts with upstream conflicts for every later commit that touches the same lines. Squash first (or use `rerere`), or merge instead to resolve once. If conflicts are huge, `git merge` is usually less painful.

</details>


### Q79. How do you undo a merge that is already pushed? 🔴

<details>
<summary><b>Answer</b></summary>


Do not reset shared history. Revert the merge commit by naming which parent is the mainline, and note that re-merging the same branch later will not bring its changes back unless you first revert the revert.

```bash
git revert -m 1 <merge-sha>
# later, to re-merge the branch:
git revert <revert-sha>
```

</details>


### Q80. What is `git merge --squash` and how is it different from a squash on GitHub? 🟡

<details>
<summary><b>Answer</b></summary>


It stages the combined changes from the branch but does not create a commit or record a merge parent. You commit it yourself. GitHub's squash merge does the same server-side and uses the PR title and description as the message.

```bash
git merge --squash feature && git commit
```

</details>


### Q81. What is a patch ID and how does Git use it to skip duplicates during rebase? 🔴

<details>
<summary><b>Answer</b></summary>


`git patch-id` hashes the diff content ignoring line numbers and whitespace. Rebase and `git cherry` compare patch IDs so a commit already applied upstream (for example cherry-picked) is dropped from the replay instead of conflicting. This does not work for squash merges, which change the combined patch.

## Undoing changes: reset, revert, restore and reflog

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

</details>


### Q82. What are the three modes of `git reset`? 🟢

<details>
<summary><b>Answer</b></summary>


`--soft` moves only the branch pointer, leaving index and working tree. `--mixed` (default) moves the branch and resets the index, keeping working files. `--hard` resets all three, discarding uncommitted changes.

```bash
git reset --soft HEAD~1    # undo commit, keep staged
git reset HEAD~1           # undo commit, keep unstaged
git reset --hard HEAD~1    # undo commit and changes
```

</details>


### Q83. What is the difference between `git reset` and `git revert`? 🟢

<details>
<summary><b>Answer</b></summary>


`reset` moves the branch pointer backwards and rewrites history, which is fine locally but disruptive on shared branches. `revert` adds a new commit that applies the inverse of an earlier commit, so history stays intact and it is safe on pushed branches.

```bash
git revert abc1234
git revert HEAD~3..HEAD
```

</details>


### Q84. How do you discard changes in one file? 🟢

<details>
<summary><b>Answer</b></summary>


Restore it from the index or a commit. This permanently discards the uncommitted edits since they were never stored.

```bash
git restore app.py
git restore --staged app.py              # unstage only
git restore --source=origin/main app.py  # take main's version
```

</details>


### Q85. How do you recover a commit after `git reset --hard`? 🟡

<details>
<summary><b>Answer</b></summary>


The commit still exists until garbage collected. Find it in the reflog and point a branch to it.

```bash
git reflog
git reset --hard HEAD@{1}
# or
git branch recovered 3fa9c21
```

</details>


### Q86. Can you recover uncommitted changes lost by `git reset --hard` or `git checkout .`? 🟡

<details>
<summary><b>Answer</b></summary>


Not from the reflog, since they were never committed. Anything that was ever staged (`git add`) exists as a dangling blob and can be found with `git fsck --lost-found`. Unstaged edits are gone unless an IDE local history has them.

```bash
git fsck --lost-found
ls .git/lost-found/other
```

</details>


### Q87. What does `git commit --amend` do to the reflog and to the old commit? 🟡

<details>
<summary><b>Answer</b></summary>


It creates a new commit replacing the tip, and the old one becomes unreachable but stays in the reflog until expiry. If you already pushed, the remote will reject a normal push and you must use `--force-with-lease`.

</details>


### Q88. How do you undo `git add` for a file? 🟢

<details>
<summary><b>Answer</b></summary>


Unstage it with `git restore --staged <file>` (older equivalent: `git reset HEAD <file>`). The working tree edits remain.

</details>


### Q89. What is the difference between `--force` and `--force-with-lease`? 🟡

<details>
<summary><b>Answer</b></summary>


`--force` overwrites the remote ref regardless. `--force-with-lease` only overwrites if the remote ref still equals what you last fetched (or an explicit expected SHA), so you do not clobber a colleague's push you have not seen. Add `--force-if-includes` to guard against a background fetch that updated your tracking ref.

```bash
git push --force-with-lease --force-if-includes origin feature/x
```

</details>


### Q90. How do you stash only some files, or stash untracked and ignored files? 🟡

<details>
<summary><b>Answer</b></summary>


Use pathspecs, `-u` for untracked, `-a` for all including ignored, and `-p` to pick hunks. `--keep-index` stashes only unstaged changes.

```bash
git stash push -m "wip" -- src/api.py
git stash push -u
git stash push -p
```

</details>


### Q91. How do you view the contents of a stash? 🟡

<details>
<summary><b>Answer</b></summary>


`git stash show` gives a diffstat, `-p` the patch. A stash is a commit with up to three parents (HEAD, index, untracked) so you can also `git show stash@{1}`.

```bash
git stash show -p stash@{1}
git diff stash@{0}^1 stash@{0}
```

</details>


### Q92. How can you recover a dropped stash? 🔴

<details>
<summary><b>Answer</b></summary>


A dropped stash commit becomes dangling but is still in the object store until pruned. List dangling commits and inspect them.

```bash
git fsck --no-reflogs --unreachable | grep commit
git show <sha>
git stash apply <sha>
```

</details>


### Q93. What is `reflog expire` and how do `gc.reflogExpire` and `gc.pruneExpire` interact? 🔴

<details>
<summary><b>Answer</b></summary>


Reachable reflog entries expire after 90 days and unreachable ones after 30 (`gc.reflogExpire`, `gc.reflogExpireUnreachable`). Objects only become prunable when not reachable from refs or reflog and older than `gc.pruneExpire` (2 weeks). So a lost commit is typically recoverable for at least a month, but never rely on it for shared data.

</details>


### Q94. What is the safest way to undo a pushed commit that exposed a bug? 🟡

<details>
<summary><b>Answer</b></summary>


`git revert <sha>` on the shared branch, open a PR, merge. History stays append-only so no one needs to re-sync. If the bad commit is a merge, add `-m 1`. Rewriting main with `reset --hard` plus a force-push should be reserved for secret leaks and done with team coordination.

</details>


### Q95. What does `git checkout -- .` or `git restore .` do? 🟢

<details>
<summary><b>Answer</b></summary>


It overwrites all tracked modified files in the working tree with the index version, losing unstaged edits irreversibly. It does not touch untracked files and it does not touch staged content unless `--staged` is given.

</details>


### Q96. What does `git reset <commit> -- <path>` do, and how does it differ from `git restore --source`? 🟡

<details>
<summary><b>Answer</b></summary>


`git reset <commit> -- path` copies the file's version at `<commit>` into the index only, leaving the working tree alone. `git restore --source=<commit> path` writes into the working tree (and with `--staged` also the index). Neither moves the branch pointer.

</details>


### Q97. How do you split a single commit into two? 🔴

<details>
<summary><b>Answer</b></summary>


Reset the commit softly and stage it in pieces, or use rebase `edit`.

```bash
git rebase -i HEAD~3      # mark commit as edit
git reset HEAD^           # mixed: keep changes unstaged
git add -p && git commit -m "part 1"
git add -p && git commit -m "part 2"
git rebase --continue
```

</details>


### Q98. How do you remove a file from the last commit but keep it on disk? 🟡

<details>
<summary><b>Answer</b></summary>


`git rm --cached` stages the removal, then amend.

```bash
git rm --cached secrets.json
git commit --amend --no-edit
```

</details>


### Q99. How do you undo a rebase that went wrong after it finished? 🟡

<details>
<summary><b>Answer</b></summary>


Use the reflog: the branch moved at each replayed commit, so find the entry just before `rebase (start)` and reset to it. `ORIG_HEAD` also works if nothing else has moved it.

```bash
git reflog
git reset --hard HEAD@{N}   # entry before "rebase (start)"
```

</details>


### Q100. What does `git restore -p` and `git checkout -p` offer? 🔴

<details>
<summary><b>Answer</b></summary>


They let you discard selected hunks interactively rather than whole files, which is useful to remove debug prints while keeping real changes. `git restore -p --staged` unstages hunks.

</details>


### Q101. What is `git commit --allow-empty` used for? 🟢

<details>
<summary><b>Answer</b></summary>


It creates a commit with no changes. Common uses are retriggering CI without touching code, marking a point in history, or creating an initial root commit for a branch.

```bash
git commit --allow-empty -m "ci: retrigger"
```

</details>


### Q102. How does `git revert` handle a commit that has already been partially undone or conflicts? 🔴

<details>
<summary><b>Answer</b></summary>


Revert is a three-way merge using the commit as the base and its parent as the "theirs" side. If later commits changed the same lines you get a conflict to resolve normally, then `git revert --continue`. `-n` applies the inverse without committing so several reverts can be combined into one commit.

```bash
git revert -n abc123 def456
git commit -m "revert: roll back checkout redesign"
```

</details>


### Q103. Git has no undo command, so what safety net do you set up for a team? 🟡

<details>
<summary><b>Answer</b></summary>


Git has no undo command; safety comes from habits: commit often, push backup branches, enable `rerere`, and know the reflog. Use `--force-with-lease` and protected branches so history rewriting is limited to private branches, and set `gc.reflogExpire` generously on developer machines.

## Conflicts, remotes, fetch, pull and push

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

</details>


### Q104. What do conflict markers look like and how do you resolve them? 🟢

<details>
<summary><b>Answer</b></summary>


Git writes `<<<<<<< HEAD`, `=======` and `>>>>>>> branch` around the competing versions. Edit the file to the final intended content, remove the markers, stage it and continue the operation.

```bash
git status                # "both modified"
# edit the file
git add app.py
git merge --continue      # or rebase --continue
```

</details>


### Q105. What is `merge.conflictStyle=zdiff3` and why use it? 🟡

<details>
<summary><b>Answer</b></summary>


By default markers show only your side and theirs. `diff3` also shows the base version between `|||||||` and `=======`, and `zdiff3` (Git 2.35+) does the same while trimming common lines. Seeing the base makes it much clearer who changed what.

```bash
git config --global merge.conflictStyle zdiff3
```

</details>


### Q106. How do you quickly take one side's version for a conflicted file? 🟡

<details>
<summary><b>Answer</b></summary>


During a merge, `--ours` is your current branch and `--theirs` the branch being merged. During a rebase the meanings flip, because your commits are being replayed onto the upstream: `--ours` is the upstream and `--theirs` is your commit.

```bash
git checkout --theirs path/file && git add path/file
git restore --theirs path/file
```

</details>


### Q107. What are stages 1, 2 and 3 in the index during a conflict? 🔴

<details>
<summary><b>Answer</b></summary>


Git keeps three entries for an unmerged path: stage 1 is the common ancestor, stage 2 is "ours", stage 3 is "theirs". You can read them directly and use `git mergetool` or `git checkout-index`.

```bash
git show :1:app.py   # base
git show :2:app.py   # ours
git show :3:app.py   # theirs
git diff --base app.py
```

</details>


### Q108. How do you see what a merge conflict is about before resolving? 🟡

<details>
<summary><b>Answer</b></summary>


Use `git log --merge -p -- <path>` to show only commits from both sides that touch the conflicted file, and `git diff` to see the combined diff of the conflicted state.

```bash
git log --merge --oneline -- app.py
```

</details>


### Q109. What is the difference between `git fetch` and `git pull`? 🟢

<details>
<summary><b>Answer</b></summary>


`fetch` downloads new objects and updates remote-tracking refs like `origin/main` without touching your branches or working tree. `pull` runs fetch and then integrates (merge or rebase) into the current branch. Fetch is always safe and lets you inspect first.

```bash
git fetch origin
git log HEAD..origin/main --oneline
```

</details>


### Q110. What is a remote and what does `origin` mean? 🟢

<details>
<summary><b>Answer</b></summary>


A remote is a named URL stored in config. `origin` is only the default name given by `git clone`, with no special meaning. You can add several, such as `upstream` for the original repo when working from a fork.

```bash
git remote -v
git remote add upstream https://github.com/org/repo.git
git remote set-url origin git@github.com:me/repo.git
```

</details>


### Q111. Why was my push rejected with "non-fast-forward" or "fetch first"? 🟡

<details>
<summary><b>Answer</b></summary>


The remote branch has commits you do not have, so updating it would discard them. Fetch, then integrate with merge or rebase and push again. Force-pushing is only right for your own feature branch, and then only with lease.

```text
! [rejected] main -> main (fetch first)
hint: Updates were rejected because the remote contains work that you do not have locally.
```

</details>


### Q112. What do `push.default` options (`simple`, `current`, `upstream`) mean? 🟡

<details>
<summary><b>Answer</b></summary>


`simple` (default) pushes the current branch to a same-named remote branch, and when pulling-and-pushing to different names it refuses. `current` pushes to a same-named branch regardless of upstream. `upstream` pushes to the configured tracking branch. `matching` (legacy) pushed all same-named branches and is dangerous.

</details>


### Q113. How does `git clone` differ for HTTPS, SSH and `--mirror`/`--bare`? 🟡

<details>
<summary><b>Answer</b></summary>


HTTPS uses tokens or a credential helper, SSH uses keys. `--bare` creates a repo with no working tree (used for servers). `--mirror` is bare plus all refs mapped one-to-one, used for backups and migrations, and `git push --mirror` replicates every ref.

```bash
git clone --mirror git@old:org/repo.git
cd repo.git && git push --mirror git@new:org/repo.git
```

</details>


### Q114. What are shallow clones and when are they dangerous? 🟡

<details>
<summary><b>Answer</b></summary>


`--depth N` downloads only the last N commits. It saves time in CI, but `git describe`, `merge-base`, changelog generation and blame can break because history is truncated. Deepen with `git fetch --unshallow` or `--deepen`, and for CI use `fetch-depth: 0` where full history matters.

```bash
git clone --depth 1 --branch v1.4.0 https://github.com/org/repo.git
git fetch --unshallow
```

</details>


### Q115. How do partial clones (`--filter`) work and which filter would you pick? 🔴

<details>
<summary><b>Answer</b></summary>


The server omits some objects and the client fetches them on demand via the promisor remote. `blob:none` skips file contents until checkout (best for developers who need full history), `tree:0` skips trees too (good for CI doing log-only work), and `blob:limit=1m` skips large blobs.

```bash
git clone --filter=blob:none https://github.com/org/big.git
```

</details>


### Q116. How do you keep a fork in sync with the original repo? 🟢

<details>
<summary><b>Answer</b></summary>


Add the original as `upstream`, fetch it, then merge or rebase and push to your fork. GitHub also has a Sync fork button and `gh repo sync`.

```bash
git fetch upstream
git switch main && git merge --ff-only upstream/main
git push origin main
```

</details>


### Q117. What does `git fetch --prune` do and why enable `fetch.prune`? 🟡

<details>
<summary><b>Answer</b></summary>


It deletes remote-tracking refs for branches that no longer exist on the remote, so `git branch -r` stops showing deleted PR branches. Setting `fetch.prune true` does it on every fetch/pull.

</details>


### Q118. What is the difference between `origin/main`, `main` and `FETCH_HEAD`? 🟡

<details>
<summary><b>Answer</b></summary>


`main` is your local branch, `origin/main` is a read-only snapshot of the remote's main taken at the last fetch, and `FETCH_HEAD` is a temporary ref holding what the most recent fetch retrieved (useful with `git fetch origin pull/12/head`).

```bash
git fetch origin pull/12/head && git switch -c pr-12 FETCH_HEAD
```

</details>


### Q119. Describe what happens on the wire during `git push` over smart HTTP. 🔴

<details>
<summary><b>Answer</b></summary>


The client asks `info/refs?service=git-receive-pack`, receives the server's ref advertisement with capabilities, computes the objects the server lacks, and sends commands (old SHA, new SHA, ref) followed by a packfile. The server's receive-pack runs hooks (pre-receive, update), writes objects and updates refs, then reports per-ref status.

</details>


### Q120. What are `git bundle` and `git format-patch`/`git am` for? 🔴

<details>
<summary><b>Answer</b></summary>


A bundle packages objects and refs into one file for transfer without network access (air-gapped environments). `format-patch` emits emailable patch files and `am` applies them with authorship preserved, the workflow used by the Linux kernel.

```bash
git bundle create repo.bundle --all
git clone repo.bundle restored
git format-patch -3 -o patches/
git am patches/*.patch
```

</details>


### Q121. How do you set up SSH authentication to GitHub? 🟢

<details>
<summary><b>Answer</b></summary>


Generate a key, add the public key in GitHub settings, and test. Use Ed25519 and a passphrase with `ssh-agent`.

```bash
ssh-keygen -t ed25519 -C "me@example.com"
ssh-add ~/.ssh/id_ed25519
ssh -T git@github.com
```

</details>


### Q122. How should you authenticate over HTTPS on a developer machine and in CI? 🟡

<details>
<summary><b>Answer</b></summary>


Passwords are no longer accepted by GitHub. Developers use Git Credential Manager or `gh auth login`, which stores an OAuth token in the OS keychain. In CI use the short-lived `GITHUB_TOKEN`, a GitHub App installation token, or a fine-grained PAT, never a personal classic PAT.

```bash
gh auth login
gh auth setup-git
```

</details>


### Q123. How do multiple remotes and push URLs allow mirroring on push? 🔴

<details>
<summary><b>Answer</b></summary>


A remote can have several `pushurl` entries, so one `git push origin` goes to all of them. Fetch uses only the first URL, so this is a lightweight way to mirror to a second host.

```bash
git remote set-url --add --push origin git@github.com:org/repo.git
git remote set-url --add --push origin git@gitlab.com:org/repo.git
```

</details>


### Q124. What does "Your local changes would be overwritten by merge" mean? 🟡

<details>
<summary><b>Answer</b></summary>


The merge or checkout would change files that you have uncommitted edits in. Commit, stash, or discard them first. Git refuses rather than risk losing your work.

```bash
git stash push -m wip && git pull && git stash pop
```

</details>


### Q125. How does Git handle line endings and what is `core.autocrlf`? 🔴

<details>
<summary><b>Answer</b></summary>


`autocrlf=true` converts CRLF to LF on commit and LF to CRLF on checkout (Windows), `input` converts only on commit. A committed `.gitattributes` with `* text=auto eol=lf` is more reliable because it applies to everyone regardless of local config. Mixed endings produce whole-file diffs and phantom conflicts.

```bash
git add --renormalize .
```

</details>


### Q126. What is `git remote prune origin` versus `git gc`? 🟡

<details>
<summary><b>Answer</b></summary>


`remote prune` removes stale remote-tracking refs only (same effect as `fetch --prune`). `gc` cleans up objects locally. They solve different problems.

</details>


### Q127. What are `git ls-remote` and `git remote show origin` good for? 🟡

<details>
<summary><b>Answer</b></summary>


`ls-remote` lists refs on a remote without cloning, useful to check a tag or branch SHA in scripts. `remote show` prints tracking relationships, stale branches and what push/pull would do.

```bash
git ls-remote --tags origin 'v1.*'
git ls-remote origin refs/heads/main
```

## Branching strategies, pull requests and code review

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

</details>


### Q128. What is a feature-branch workflow? 🟢

<details>
<summary><b>Answer</b></summary>


Every change is developed on a short-lived branch off main, pushed, reviewed in a pull request and merged back. Main stays releasable. It is the default for most teams on GitHub.

</details>


### Q129. What is GitFlow and what are its branches? 🟡

<details>
<summary><b>Answer</b></summary>


GitFlow uses long-lived `main` and `develop`, plus `feature/*`, `release/*` and `hotfix/*` branches. Features merge into develop, a release branch stabilises and is merged to both main (tagged) and develop. It suits versioned products with scheduled releases and several supported versions, but is heavy for continuously deployed services.

</details>


### Q130. What is trunk-based development and what does it require? 🟡

<details>
<summary><b>Answer</b></summary>


Developers integrate into a single trunk at least daily using very short-lived branches or direct commits. Incomplete work is hidden behind feature flags. It requires fast reliable CI, good automated tests, small PRs and a culture of fixing a broken trunk immediately. DORA research links it to high delivery performance.

</details>


### Q131. What is GitHub Flow and how does it differ from GitFlow? 🟢

<details>
<summary><b>Answer</b></summary>


GitHub Flow has one long-lived branch (`main`) that is always deployable. You branch, open a PR, get review and passing checks, merge, deploy. There are no develop or release branches, so it is simpler but expects continuous delivery.

</details>


### Q132. What is release-branch strategy with cherry-picked fixes? 🟡

<details>
<summary><b>Answer</b></summary>


Cut `release/1.4` from main when you freeze, keep developing on main, and cherry-pick (or merge forward) only approved fixes into the release branch. Always fix on main first and back-port, to avoid regressions in the next release.

</details>


### Q133. What makes a good pull request? 🟢

<details>
<summary><b>Answer</b></summary>


Small and focused (aim for under about 400 changed lines), a clear title, a description with the why, how it was tested and rollout risk, links to the issue, screenshots when UI changes, passing CI, and self-review before requesting others.

</details>


### Q134. What is a draft pull request? 🟢

<details>
<summary><b>Answer</b></summary>


A PR marked not ready for review. It cannot be merged, CODEOWNERS are not automatically requested, and it signals work in progress while still running CI and allowing early feedback.

```bash
gh pr create --draft --fill
gh pr ready
```

</details>


### Q135. What are the typical required status checks on a PR? 🟡

<details>
<summary><b>Answer</b></summary>


Build, unit tests, lint, security scans (SAST, dependency, secret scan) and sometimes a preview deploy. In branch protection you mark them required so the PR cannot merge until they pass, and you can require branches to be up to date with the base.

</details>


### Q136. What are stacked PRs and how do you manage them in Git? 🟡

<details>
<summary><b>Answer</b></summary>


A chain of dependent PRs, each branching from the previous, so a large change is reviewed in small pieces. In Git you rebase the stack when a lower PR changes (`git rebase --onto`, `--update-refs` in Git 2.38+), and tools like Graphite, ghstack or `git-branchless` automate it.

```bash
git rebase --update-refs origin/main
```

</details>


### Q137. How do you review a teammate's PR locally? 🟢

<details>
<summary><b>Answer</b></summary>


Fetch the PR head ref or use the GitHub CLI.

```bash
gh pr checkout 482
git fetch origin pull/482/head:pr-482
git diff origin/main...pr-482
```

</details>


### Q138. What do `Closes #123` and `Fixes #123` do in a PR or commit? 🟡

<details>
<summary><b>Answer</b></summary>


On merge into the default branch, GitHub auto-closes the referenced issue. Keywords include close, closes, fix, fixes, resolve, resolves. Cross-repo form is `Fixes org/repo#123`. It only works when the PR targets the default branch.

</details>


### Q139. How do you decide between merge, squash and rebase for PRs on a team? 🟡

<details>
<summary><b>Answer</b></summary>


Squash gives a clean one-commit-per-PR main and easy reverts, but loses granular commits. Merge commits keep full history and suit release branches where you want to merge forward without conflicts. Rebase-and-merge gives linear history with each commit retained but needs atomic, well-written commits. Most teams choose squash for apps and merge commits for GitFlow release branches.

</details>


### Q140. Why can long-lived branches be harmful and how do you mitigate? 🔴

<details>
<summary><b>Answer</b></summary>


Divergence grows, so merges become large, conflicts costly and integration bugs surface late. Mitigate by merging main into the branch often (or rebasing), shipping behind feature flags, branching by abstraction, and breaking work into independently mergeable slices.

</details>


### Q141. What is a feature flag and how does it relate to branching? 🟡

<details>
<summary><b>Answer</b></summary>


A runtime toggle that decouples deploying code from releasing it. With flags, unfinished code can merge to trunk without being user-visible, which removes the need for long feature branches. Flags need ownership and cleanup to avoid permanent debt.

</details>


### Q142. How do you name branches and why does it matter? 🟢

<details>
<summary><b>Answer</b></summary>


Use a consistent pattern such as `feature/ABC-123-short-desc`, `fix/...`, `hotfix/...`, `release/1.4`. Prefixes allow branch protection rulesets and CI rules to target patterns, and ticket IDs enable linking in Jira or GitHub.

</details>


### Q143. What are Conventional Commits and semantic versioning, and how do they work together? 🟡

<details>
<summary><b>Answer</b></summary>


Commit types map to version bumps: `fix:` is a patch, `feat:` is minor, and `feat!:` or a `BREAKING CHANGE:` footer is major. Tools like release-please or semantic-release read the commit history to compute the next version, update the changelog and tag.

```text
feat(api)!: drop v1 endpoints

BREAKING CHANGE: /v1 removed
```

</details>


### Q144. Explain how `git merge-base` and `A...B` relate to what a GitHub PR diff shows. 🔴

<details>
<summary><b>Answer</b></summary>


The PR "Files changed" tab shows the diff between the merge base of base and head, and the head tip, equal to `git diff base...head`. If the base advanced, the PR diff does not include those commits, but the merge result may still differ from what you reviewed, which is why up-to-date and merge queue checks matter.

</details>


### Q145. What does a merge queue do? 🟡

<details>
<summary><b>Answer</b></summary>


GitHub merge queue builds a temporary branch containing the base plus queued PRs in order and runs required checks on that combination before merging. It prevents semantic conflicts and broken main when many PRs pass CI individually but fail together. Workflows must listen to the `merge_group` event.

```yaml
on:
  pull_request:
  merge_group:
```

</details>


### Q146. What is code review for, beyond finding bugs? 🟢

<details>
<summary><b>Answer</b></summary>


Knowledge sharing, consistency, design feedback, security and operational review (is it observable, is it safe to roll back). Good reviewers comment on behaviour and risk, leave style to linters, and approve when remaining nits are non-blocking.

</details>


### Q147. What are CODEOWNERS and how does GitHub resolve them? 🟡

<details>
<summary><b>Answer</b></summary>


A `CODEOWNERS` file in `.github/`, root or `docs/` maps path patterns to users or teams. The last matching pattern wins. When "Require review from Code Owners" is enabled, a PR touching a path needs approval from an owner of that path.

```text
*                @org/platform
/infra/          @org/sre
*.tf             @org/sre @org/security
```

</details>


### Q148. How do you cleanly update a PR branch after reviews without destroying review context? 🔴

<details>
<summary><b>Answer</b></summary>


Prefer adding new commits (or fixup commits) while the review is active, then squash at merge. If you must rebase, use `--force-with-lease`, mention it in the PR, and use `git range-diff` so reviewers can see only what changed. GitHub's "Update branch" button merges or rebases the base into the PR branch.

## Hooks, submodules, LFS and signing

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

</details>


### Q149. What are Git hooks and where do they live? 🟢

<details>
<summary><b>Answer</b></summary>


Scripts that Git runs at defined points, stored in `.git/hooks` (not versioned). Client hooks include `pre-commit`, `commit-msg`, `pre-push`; server hooks include `pre-receive` and `update`. A hook that exits non-zero aborts the operation.

```bash
cat > .git/hooks/pre-commit <<'H'
#!/bin/sh
npm run lint --silent || exit 1
H
chmod +x .git/hooks/pre-commit
```

</details>


### Q150. How do you share hooks with a team, given `.git/hooks` is not committed? 🟡

<details>
<summary><b>Answer</b></summary>


Commit them in a directory and point Git at it with `core.hooksPath`, or use a framework such as pre-commit, Husky or lefthook that installs them on setup. Hooks are bypassable with `--no-verify`, so always enforce the same checks in CI.

```bash
git config core.hooksPath .githooks
```

</details>


### Q151. What does the `commit-msg` hook receive and what is it good for? 🟡

<details>
<summary><b>Answer</b></summary>


Git passes the path to a file holding the message as `$1`. The hook can validate Conventional Commit format, require a ticket ID, or append a trailer, then exit non-zero to reject.

```bash
grep -qE '^(feat|fix|chore|docs|refactor|test)(\(.+\))?!?: .+' "$1" || { echo "bad commit message"; exit 1; }
```

</details>


### Q152. What do `pre-receive` and `update` hooks do and how do they differ? 🔴

<details>
<summary><b>Answer</b></summary>


Both run on the server during push. `pre-receive` runs once for the whole push and gets all ref updates on stdin (`old new ref`); rejecting stops everything. `update` runs once per ref and receives the ref and SHAs as arguments, so it can reject branches individually. Hosted GitHub does not allow custom hooks except on GitHub Enterprise Server, which is why rulesets are used.

</details>


### Q153. What is a Git submodule? 🟢

<details>
<summary><b>Answer</b></summary>


A pointer from a parent repo to a specific commit in another repo, recorded as a gitlink entry (mode 160000) plus `.gitmodules`. The parent pins an exact commit, so builds are reproducible.

```bash
git submodule add https://github.com/org/lib.git libs/lib
git clone --recurse-submodules https://github.com/org/app.git
git submodule update --init --recursive
```

</details>


### Q154. Why do submodules confuse people and what are the common pitfalls? 🟡

<details>
<summary><b>Answer</b></summary>


The parent tracks a commit, not a branch, so after `git pull` the submodule stays at the old commit until `git submodule update`. Commits made inside the submodule must be pushed first, otherwise the parent references a SHA nobody else can fetch. Detached HEAD in submodules is the default.

```bash
git config submodule.recurse true
git push --recurse-submodules=check
```

</details>


### Q155. Submodules versus subtrees versus a monorepo: how do you choose? 🔴

<details>
<summary><b>Answer</b></summary>


Submodules give strict version pinning and separate histories but need extra commands and discipline. Subtree (`git subtree`) copies the other repo's history into a subdirectory, so cloners need nothing special but sync is manual. Monorepo gives atomic cross-project changes and simple tooling but needs build caching, ownership rules and sparse checkout at scale. Choose by how tightly the code changes together.

```bash
git subtree add --prefix=vendor/lib https://github.com/org/lib.git main --squash
```

</details>


### Q156. What problem does Git LFS solve? 🟢

<details>
<summary><b>Answer</b></summary>


Git stores every version of every file forever, so big binaries (videos, datasets, PSDs) bloat the repo. LFS replaces them in the repo with small text pointers and stores the actual content on an LFS server, downloaded on checkout.

```bash
git lfs install
git lfs track "*.psd"
git add .gitattributes model.psd
```

</details>


### Q157. What does an LFS pointer file look like and what are the GitHub LFS limits? 🟡

<details>
<summary><b>Answer</b></summary>


A pointer is a three-line text file: spec version, `oid sha256:<hash>` and `size`. GitHub caps individual files at 100 MB for regular Git (warning at 50 MB) and charges LFS storage and bandwidth beyond the free quota; per-file LFS limits depend on plan (2 GB on Free).

```text
version https://git-lfs.github.com/spec/v1
oid sha256:4d7a2146...
size 132714
```

</details>


### Q158. How do you migrate existing large files into LFS? 🔴

<details>
<summary><b>Answer</b></summary>


Use `git lfs migrate import`, which rewrites history so matching files become pointers. It rewrites commits, so everyone must re-clone, and protected branches need temporary adjustment.

```bash
git lfs migrate import --include="*.zip,*.psd" --everything
git push --force-with-lease --all
```

</details>


### Q159. Why sign commits and what are the options? 🟢

<details>
<summary><b>Answer</b></summary>


Author name and email are just text and can be forged, so signatures prove who made a commit. Options are GPG, SSH keys (Git 2.34+) and Sigstore gitsign (keyless). GitHub shows a Verified badge when the signature matches a key on the account.

```bash
git config --global gpg.format ssh
git config --global user.signingkey ~/.ssh/id_ed25519.pub
git config --global commit.gpgsign true
```

</details>


### Q160. How does GitHub verify an SSH or GPG signature, and what is vigilant mode? 🟡

<details>
<summary><b>Answer</b></summary>


GitHub checks the signature against signing keys uploaded to the committer's account and the email on the commit. Vigilant mode marks all unsigned commits attributed to you as Unverified, flagging impersonation. Locally verify with `git log --show-signature` and, for SSH, an `allowed_signers` file.

```bash
git config gpg.ssh.allowedSignersFile ~/.config/git/allowed_signers
git log --show-signature -1
```

</details>


### Q161. How do you sign tags and verify them? 🟡

<details>
<summary><b>Answer</b></summary>


Use `-s` for signed annotated tags and `git tag -v` to verify.

```bash
git tag -s v1.4.0 -m "Release 1.4.0"
git tag -v v1.4.0
git verify-commit HEAD
```

</details>


### Q162. What are `git credential helpers`? 🟡

<details>
<summary><b>Answer</b></summary>


Programs that store or supply credentials for HTTPS remotes: `osxkeychain`, `manager` (Git Credential Manager), `libsecret`, or `cache`. The `store` helper writes plaintext to disk and should be avoided.

```bash
git config --global credential.helper 'cache --timeout=3600'
```

</details>


### Q163. What belongs in a `.gitignore` and what does not? 🟢

<details>
<summary><b>Answer</b></summary>


Ignore generated and local files: build output, dependencies (`node_modules/`), virtualenvs, IDE settings, logs and `.env`. Do not rely on it for secrets already committed. Commit lock files, and keep personal editor patterns in your global ignore.

```bash
git config --global core.excludesFile ~/.gitignore_global
```

</details>


### Q164. What are the `.gitignore` pattern rules for `*`, `**`, `/` and `!`? 🟡

<details>
<summary><b>Answer</b></summary>


`*` matches within a path segment, `**` matches across directories, a leading `/` anchors to the repo root, a trailing `/` matches directories only, `!` re-includes a path (not possible if a parent directory is ignored). Last match wins.

```text
/build/
*.log
!important.log
**/temp/
```

</details>


### Q165. What is `git update-index --assume-unchanged` versus `--skip-worktree`? 🔴

<details>
<summary><b>Answer</b></summary>


Both make Git ignore local changes to a tracked file. `assume-unchanged` is a performance hint that Git may undo during pulls; `skip-worktree` is the right flag for "I keep a local version of this tracked config". They are local only and can cause confusing merge failures, so prefer a template file plus an ignored local copy.

```bash
git update-index --skip-worktree config/local.yml
git ls-files -v | grep '^S'
```

</details>


### Q166. What is `git config --global alias` and which aliases are worth having? 🟡

<details>
<summary><b>Answer</b></summary>


Aliases map short names to commands or shell snippets.

```bash
git config --global alias.lg "log --oneline --graph --decorate --all"
git config --global alias.st "status -sb"
git config --global alias.undo "reset --soft HEAD~1"
```

</details>


### Q167. What does `git filter-branch` versus `git filter-repo` do, and which should you use? 🔴

<details>
<summary><b>Answer</b></summary>


Both rewrite history, for example to remove a file or change author data. `filter-branch` is slow, error-prone and officially discouraged. `git filter-repo` (separate Python tool) is fast, safe by default (works only on fresh clones) and is the recommended replacement. BFG Repo-Cleaner is a simpler alternative for removing files or secrets.

```bash
git filter-repo --invert-paths --path secrets.json
```

</details>


### Q168. What is `git notes`? 🔴

<details>
<summary><b>Answer</b></summary>


Notes attach extra data to commits without changing their IDs, stored under `refs/notes/*`. They are not pushed or fetched by default and GitHub does not render them, so they are used mostly by tooling such as code review systems.

```bash
git notes add -m "deployed to prod 2026-11-02" HEAD
git push origin refs/notes/commits
```

</details>


### Q169. What does `git config --list --show-origin` tell you? 🟢

<details>
<summary><b>Answer</b></summary>


Every effective setting along with the file it came from, which is the quickest way to debug why a setting (such as user email or a hook path) is not what you expect.

</details>


### Q170. What is `git maintenance` and which tasks does it schedule? 🔴

<details>
<summary><b>Answer</b></summary>


It runs repository upkeep in the background through the system scheduler: `prefetch`, `commit-graph`, `loose-objects`, `incremental-repack`, `gc`. `git maintenance start` registers hourly and daily jobs so foreground commands stay fast on big repos.

```bash
git maintenance start
git maintenance run --task=commit-graph
```

</details>


### Q171. What is `git diff --word-diff` and `git diff --color-moved` for? 🟡

<details>
<summary><b>Answer</b></summary>


`--word-diff` highlights changed words inside lines, good for prose and long lines. `--color-moved` colours blocks that were moved rather than changed, which makes refactors far easier to review.

```bash
git diff --color-moved=dimmed-zebra
```

</details>


### Q172. How does `git replace` work and when might it be used? 🔴

<details>
<summary><b>Answer</b></summary>


It creates a ref under `refs/replace/` so that Git transparently substitutes one object for another on read, without rewriting history. It has been used to graft old history onto a trimmed repo after a migration. Replacement refs are local unless pushed.

```bash
git replace --graft <new-root> <old-tip>
```

</details>


### Q173. What is `git rev-list --count` and `git shortlog` used for? 🟡

<details>
<summary><b>Answer</b></summary>


`rev-list --count` returns the number of commits in a range (useful for build numbers or ahead/behind), `shortlog -sn` ranks authors by commit count.

```bash
git rev-list --count origin/main..HEAD
git shortlog -sn --since="3 months ago"
```

</details>


### Q174. How do `git switch --detach` and `git checkout <tag>` differ from checking out a branch for releases? 🟡

<details>
<summary><b>Answer</b></summary>


A tag or SHA checkout gives a detached HEAD at an immutable point, which is exactly what you want for building a release reproducibly. Branch checkouts move over time. Release pipelines should build from a tag or SHA.

## GitHub features, history rewriting and scale

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

</details>


### Q175. What does a GitHub Actions workflow look like at a basic level? 🟢

<details>
<summary><b>Answer</b></summary>


YAML in `.github/workflows/` with triggers (`on`), jobs, and steps that either run commands or call actions. Jobs run in parallel on runners by default and can depend on each other via `needs`.

```yaml
name: ci
on: [push, pull_request]
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm ci && npm test
```

</details>


### Q176. What are GitHub environments and what protection do they offer? 🟡

<details>
<summary><b>Answer</b></summary>


An environment (such as `staging`, `production`) groups deployment secrets and variables and can require reviewers, a wait timer and branch/tag restrictions before a job targeting it runs. A job opts in with `environment: production`, and the secrets only become available after approval.

```yaml
jobs:
  deploy:
    environment: production
    runs-on: ubuntu-latest
```

</details>


### Q177. What is a GitHub Release and how does it relate to a Git tag? 🟢

<details>
<summary><b>Answer</b></summary>


A release is a GitHub object built on a tag, with notes and optional binary assets. Creating a release can create the tag. Pre-release flags and auto-generated notes (from merged PRs) are supported.

```bash
gh release create v1.4.0 --generate-notes ./dist/app.tar.gz
```

</details>


### Q178. What are the permissions of `GITHUB_TOKEN` and how do you restrict them? 🟡

<details>
<summary><b>Answer</b></summary>


It is a per-run token with repo-scoped permissions. Set the default to read-only in org settings and grant only what each workflow needs with a `permissions:` block.

```yaml
permissions:
  contents: read
  pull-requests: write
```

</details>


### Q179. Why do fork PRs get no secrets, and what is `pull_request_target`? 🔴

<details>
<summary><b>Answer</b></summary>


For `pull_request` events from forks, secrets are withheld and the token is read-only, so untrusted code cannot steal them. `pull_request_target` runs in the base repo context with secrets, so checking out and running fork code in it is a well-known remote-code-execution risk. Never execute untrusted PR code in that trigger.

</details>


### Q180. What is the difference between a GitHub fork and a clone? 🟢

<details>
<summary><b>Answer</b></summary>


A fork is a server-side copy of a repo under your account that stays linked to the original, enabling PRs back to it. A clone is a local copy of any repo. You usually fork on GitHub and then clone your fork.

</details>


### Q181. What is a branch protection rule versus a ruleset? 🟡

<details>
<summary><b>Answer</b></summary>


Branch protection is the classic per-branch-pattern setting (required reviews, status checks, no force-push). Rulesets are the newer layer: multiple rulesets can stack, apply to branches and tags, target by pattern across many repos at org level, support bypass lists and can run in evaluate (dry-run) mode first.

</details>


### Q182. What does GitHub Dependabot do? 🟡

<details>
<summary><b>Answer</b></summary>


It opens PRs to update dependencies (version updates) and raises alerts and security-update PRs for vulnerable ones, configured in `.github/dependabot.yml`. Group updates and schedule them to reduce noise.

```yaml
version: 2
updates:
  - package-ecosystem: npm
    directory: /
    schedule: { interval: weekly }
```

</details>


### Q183. You committed a password and pushed. Is `git rm` plus a new commit enough? 🔴

<details>
<summary><b>Answer</b></summary>


No. The secret remains in history and in every clone and fork. First rotate or revoke the credential, since that is the only real fix. Then optionally remove it from history with `git filter-repo`, force-push, and ask GitHub support to purge cached views and PR refs.

```bash
git filter-repo --replace-text <(echo 'AKIA...==>REMOVED')
git push --force --all && git push --force --tags
```

</details>


### Q184. How do you remove a large file from the entire Git history? 🔴

<details>
<summary><b>Answer</b></summary>


Rewrite with `git filter-repo`, then expire and gc so the objects are actually dropped, and force-push. Collaborators must re-clone or hard reset.

```bash
git filter-repo --strip-blobs-bigger-than 50M
git filter-repo --invert-paths --path assets/video.mp4
```

</details>


### Q185. What is GitHub secret scanning and push protection? 🟡

<details>
<summary><b>Answer</b></summary>


Secret scanning looks for known token patterns in repos (and history) and alerts, notifying providers for partner patterns. Push protection blocks a push containing a detected secret before it lands, with a bypass option that can be audited or disabled.

</details>


### Q186. What is `git log --stat`, `--name-only` and `--pretty=format:`? 🟢

<details>
<summary><b>Answer</b></summary>


`--stat` shows files changed with insert/delete counts, `--name-only` lists just the file names, and `--pretty=format:` lets you design the output for scripts.

```bash
git log --since=2.weeks --pretty=format:'%h %an %s' --name-only
```

</details>


### Q187. Why is a repo with hundreds of thousands of files or a huge history slow, and which Git features help? 🔴

<details>
<summary><b>Answer</b></summary>


`git status` stats every file, `log` and `blame` walk huge graphs, and clones transfer the full pack. Remedies: fsmonitor and untracked cache for status, commit-graph and multi-pack-index for history, partial clone and sparse checkout for transfer, and `git maintenance`.

```bash
git config core.fsmonitor true
git config core.untrackedCache true
git sparse-checkout init --cone
```

</details>


### Q188. What is the multi-pack-index and what is a cruft pack? 🔴

<details>
<summary><b>Answer</b></summary>


A multi-pack-index (MIDX) is one lookup index across several packs so Git does not search each `.idx`, and it enables incremental repacking and reachability bitmaps. A cruft pack stores unreachable objects with their mtimes in a separate pack, so `gc` can expire them without exploding them into thousands of loose files.

```bash
git multi-pack-index write --bitmap
git repack --cruft -d
```

</details>


### Q189. What are reachability bitmaps? 🔴

<details>
<summary><b>Answer</b></summary>


A `.bitmap` file next to a pack precomputes which objects are reachable from selected commits, so the server can answer clone and fetch requests without walking the commit graph. Hosting providers rely on it to serve large repos quickly.

```bash
git repack -adb
```

</details>


### Q190. What is `git bisect skip` and `git bisect visualize`? 🟡

<details>
<summary><b>Answer</b></summary>


`skip` tells bisect to ignore an untestable commit (for example, one that does not build); `visualize` opens the remaining suspects in `gitk` or `log`. `git bisect log` saves the session for replay.

</details>


### Q191. How does GitHub display the "Verified" badge and what causes "Unverified"? 🟡

<details>
<summary><b>Answer</b></summary>


Causes include a key not uploaded to the account, an email on the commit that is not verified in the account, a key that expired or was revoked, or a commit rewritten after signing (rebase, squash by the web UI without a platform signature). Web-UI merges are signed by GitHub's own key.

</details>


### Q192. What is the difference between `git gc --aggressive` and `git repack -adf`? 🔴

<details>
<summary><b>Answer</b></summary>


`gc --aggressive` uses expensive repack options (`--depth=50 --window=250` and recomputes deltas) and rarely pays off on a regular basis. `repack -a -d -f` repacks everything into one pack and discards old pack deltas; add depth and window to tune. Both are heavy; for routine upkeep use `git maintenance`.

</details>


### Q193. How do you see the history of a deleted branch or find a commit that is no longer on any branch? 🟢

<details>
<summary><b>Answer</b></summary>


Use `git reflog` for local activity, or `git fsck --lost-found` for dangling commits. On GitHub, a deleted PR branch can be restored from the closed PR page, and `refs/pull/N/head` keeps the commits reachable.

</details>


### Q194. What do `git shortlog`, `git log --since` and `git whatchanged` give you for release notes? 🟡

<details>
<summary><b>Answer</b></summary>


`git log --pretty=format:'- %s (%h)' v1.3.0..v1.4.0` lists subjects between tags, and `--no-merges` removes noise. With Conventional Commits it can be grouped by type; GitHub's generated release notes use merged PR titles and labels.

</details>


### Q195. Why can `git pull` on a large monorepo be slow and how do you mitigate it? 🔴

<details>
<summary><b>Answer</b></summary>


Fetch computes and transfers many objects, then merge or checkout touches many files. Use `--filter=blob:none`, shallow-for-CI, `fetch.negotiationAlgorithm=skipping`, background prefetch via maintenance, sparse checkout and fsmonitor, and avoid full `--all` fetches.

```bash
git config fetch.negotiationAlgorithm skipping
git config feature.manyFiles true
```

</details>


### Q196. What is `git rebase --exec` and `git rebase -x` useful for? 🟡

<details>
<summary><b>Answer</b></summary>


It runs a command after each replayed commit and stops if it fails, which proves every commit in a series builds or passes tests, a nice pre-PR check.

```bash
git rebase -x "npm test" origin/main
```

</details>


### Q197. How does Git wire protocol v2 improve fetch performance? 🔴

<details>
<summary><b>Answer</b></summary>


Protocol v2 (default since Git 2.26) replaces the huge ref advertisement with a `ls-refs` command that accepts prefixes, so the client only lists refs it cares about. On repos with hundreds of thousands of refs this cuts the initial handshake dramatically. It is also the base for `fetch` filters and the `object-format` capability.

```bash
GIT_TRACE_PACKET=1 git ls-remote origin refs/heads/main 2>&1 | head
git -c protocol.version=2 fetch origin
```

</details>


### Q198. How can you build a commit by hand with plumbing commands? 🔴

<details>
<summary><b>Answer</b></summary>


Write blobs, build a tree from the index, create the commit with `commit-tree` and move the ref with `update-ref`. This is what `git commit` does internally and is handy for scripted release commits and bots.

```bash
blob=$(echo 'v2' | git hash-object -w --stdin)
git update-index --add --cacheinfo 100644,$blob,VERSION
tree=$(git write-tree)
commit=$(git commit-tree $tree -p HEAD -m "bump version")
git update-ref refs/heads/main $commit
```

</details>


### Q199. How do custom merge drivers and `merge=union` work? 🔴

<details>
<summary><b>Answer</b></summary>


In `.gitattributes` you assign a merge driver per path. Built-ins include `union` (keep lines from both sides, useful for changelogs) and `binary`. A custom driver is defined in config and receives the base, ours and theirs files as `%O %A %B`, writing the result into `%A`.

```text
CHANGELOG.md merge=union
package-lock.json merge=lockfile
```
```bash
git config merge.lockfile.driver "npm install --package-lock-only --no-audit"
```

</details>


### Q200. How does `git blame -C -C -C` detect code moved between files? 🔴

<details>
<summary><b>Answer</b></summary>


`-M` detects lines moved within a file and `-C` detects lines copied from other files modified in the same commit. Repeating `-C` widens the search: twice looks in the file-creating commit, three times looks in any commit. It is how blame attributes extracted functions to the original author rather than to the refactor commit.

```bash
git blame -w -M -C -C -C src/service.py
```


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
