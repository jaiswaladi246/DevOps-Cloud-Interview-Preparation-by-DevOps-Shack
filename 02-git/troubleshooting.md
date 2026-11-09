# Git — Troubleshooting

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

> Realistic Git and GitHub symptoms with exact error text, diagnosis commands and fixes · 200 questions · 🟢 Beginner 🟡 Intermediate 🔴 Advanced

[⬅ Back to Git & GitHub](./README.md) · [🏠 Home](../README.md)

## Table of contents
- [Authentication and Remote Access](#authentication-and-remote-access) — Q1–Q25
- [Push, Pull and Fetch Errors](#push-pull-and-fetch-errors) — Q26–Q50
- [Merge and Rebase Problems](#merge-and-rebase-problems) — Q51–Q75
- [Lost Work, HEAD and History Problems](#lost-work-head-and-history-problems) — Q76–Q100
- [Repository Health, Size and Performance](#repository-health-size-and-performance) — Q101–Q125
- [Submodules, LFS and Hooks](#submodules-lfs-and-hooks) — Q126–Q150
- [GitHub, Actions and Pull Request Issues](#github-actions-and-pull-request-issues) — Q151–Q175
- [Config, Line Endings, Files and Platform Quirks](#config-line-endings-files-and-platform-quirks) — Q176–Q200

---

## Authentication and Remote Access

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q1. `git@github.com: Permission denied (publickey).` followed by `fatal: Could not read from remote repository.` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** No SSH key is loaded, the public key was never added to GitHub, or the wrong key is offered.

**Diagnose →**
```bash
ssh -vT git@github.com        # look for "Offering public key" and the final result
ssh-add -l                    # keys loaded in the agent
```
**Fix →** Generate a key if needed (`ssh-keygen -t ed25519 -C "you@example.com"`), start the agent, `ssh-add ~/.ssh/id_ed25519`, and paste `~/.ssh/id_ed25519.pub` into GitHub → Settings → SSH keys. Test with `ssh -T git@github.com`; success prints "Hi <user>! You've successfully authenticated".

**Prevent →** Keep a documented onboarding step for keys and an `~/.ssh/config` entry with `IdentityFile`.

</details>

### Q2. `remote: Support for password authentication was removed on August 13, 2021.` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** You typed your GitHub account password over HTTPS. GitHub only accepts tokens or SSH keys for Git operations.

**Diagnose →** `git remote -v` shows an `https://` URL; `git config --show-origin credential.helper` shows what stored the old password.

**Fix →** Create a personal access token (fine-grained, with Contents read/write) and use it as the password, or log in with `gh auth login` and run `gh auth setup-git`. Alternatively switch the remote to SSH: `git remote set-url origin git@github.com:ORG/REPO.git`.

**Prevent →** Use a credential manager so the token is stored once, and rotate before expiry.

</details>

### Q3. `fatal: Authentication failed for 'https://github.com/org/repo.git/'` with a token that worked last month. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The token expired or was revoked, a stale token is cached by the credential helper, or the token lost access to the repo.

**Diagnose →**
```bash
git config --get-all credential.helper
GIT_CURL_VERBOSE=1 GIT_TRACE=1 git ls-remote origin 2>&1 | tail -20
```
**Fix →** Remove the cached credential (`printf 'protocol=https\nhost=github.com\n\n' | git credential reject`), create a new token, and retry so the helper stores the new one.

**Prevent →** Prefer SSH or `gh auth` over long-lived tokens, and calendar token expiry dates.

</details>

### Q4. `WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!` when connecting to GitHub over SSH. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The key in `~/.ssh/known_hosts` no longer matches. GitHub rotated its RSA host key in March 2023, or something is intercepting the connection.

**Diagnose →** Read the message for the offending line number in `known_hosts`. Compare the fingerprint ssh shows against the ones published in GitHub's docs (`https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/githubs-ssh-key-fingerprints`).

**Fix →** If the fingerprint matches GitHub's published list: `ssh-keygen -R github.com`, then reconnect and accept the new key. If it does not match, stop and check your network.

**Prevent →** Never blindly delete the entry on an untrusted network; always compare fingerprints.

</details>

### Q5. `remote: Repository not found.` for a repo you can open in the browser. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** GitHub returns "not found" for private repos you cannot access. Usually the wrong account or token is used, or the repo was renamed or transferred.

**Diagnose →**
```bash
git remote -v
ssh -T git@github.com                 # which user are you?
gh auth status                        # which account is gh using?
```
**Fix →** Authenticate as the user who has access, fix the URL (`git remote set-url origin ...`), and for orgs authorise the token or key for SSO.

**Prevent →** Use one credential per host, or per-directory identities with `includeIf`.

</details>

### Q6. `ssh -T git@github.com` says "Hi someone-else!" but you need your work account. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** SSH offers the first key in the agent, and GitHub maps that key to the personal account. A key can belong to only one account.

**Diagnose →** `ssh -vT git@github.com 2>&1 | grep -i "offering"` shows the order of keys offered.

**Fix →** Create a host alias with `IdentitiesOnly yes` and the work key:
```text
Host github-work
  HostName github.com
  User git
  IdentityFile ~/.ssh/id_ed25519_work
  IdentitiesOnly yes
```
Then use `git remote set-url origin git@github-work:ORG/REPO.git`.

**Prevent →** Keep one key per account and always set `IdentitiesOnly yes`.

</details>

### Q7. `ssh: connect to host github.com port 22: Connection timed out` on a corporate or hotel network. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Outbound port 22 is blocked by a firewall.

**Diagnose →**
```bash
nc -vz github.com 22
nc -vz ssh.github.com 443
```
**Fix →** GitHub accepts SSH on port 443 via `ssh.github.com`. In `~/.ssh/config`:
```text
Host github.com
  Hostname ssh.github.com
  Port 443
  User git
```
Verify with `ssh -T git@github.com`.

**Prevent →** Commit this snippet to your team's setup guide for restricted networks.

</details>

### Q8. `fatal: unable to access 'https://github.com/...': SSL certificate problem: unable to get local issuer certificate` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** A corporate proxy re-signs TLS traffic with a private CA that Git does not trust, or the system CA bundle is outdated.

**Diagnose →**
```bash
GIT_CURL_VERBOSE=1 git ls-remote https://github.com/ORG/REPO 2>&1 | grep -i -E "ssl|certificate"
git config --show-origin --get http.sslCAInfo
```
**Fix →** Point Git to a bundle that includes the corporate root: `git config --global http.sslCAInfo /etc/ssl/certs/corp-ca-bundle.pem`. On Windows, `git config --global http.sslBackend schannel` uses the OS store. Do not set `http.sslVerify false` except as a short diagnostic step.

**Prevent →** Distribute the corporate CA to dev machines and CI images.

</details>

### Q9. `fatal: unable to access '...': Could not resolve host: github.com` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** No network, DNS failure, VPN changing DNS, or a proxy that is required but not set.

**Diagnose →**
```bash
ping -c1 8.8.8.8
nslookup github.com
curl -sI https://github.com | head -1
```
**Fix →** Fix connectivity or DNS (reconnect VPN, change resolver). If a proxy is mandatory, set `git config --global http.proxy http://proxy.corp:8080`.

**Prevent →** In CI, use a runner image with correct DNS and proxy variables.

</details>

### Q10. Git asks for your username and token on every push. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** No credential helper is configured, so nothing remembers the token.

**Diagnose →** `git config --show-origin --get-all credential.helper` returns nothing.

**Fix →**
```bash
gh auth setup-git                                 # if using GitHub CLI
git config --global credential.helper osxkeychain # macOS
git config --global credential.helper manager     # Windows / Git Credential Manager
git config --global credential.helper 'cache --timeout=3600'  # Linux, memory only
```
Or switch to SSH keys.

**Prevent →** Include helper setup in onboarding; avoid `credential.helper store` since it saves plain text.

</details>

### Q11. After rotating your PAT, pushes keep failing with the old token on macOS. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The macOS Keychain still holds the old token and Git keeps sending it.

**Diagnose →** Open Keychain Access, search for `github.com` and look at internet-password entries.

**Fix →**
```bash
printf 'protocol=https\nhost=github.com\n\n' | git credential-osxkeychain erase
git push        # prompts and stores the new token
```
Or delete the entry in Keychain Access.

**Prevent →** Use `gh auth login`, so refresh and storage are handled consistently.

</details>

### Q12. `remote: Resource protected by organization SAML enforcement. You must grant your OAuth token or SSH key access to this organization.` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The org requires SAML SSO, and your token or SSH key has not been authorised for it.

**Diagnose →** In GitHub → Settings → SSH keys (or Tokens), look for a "Configure SSO" button next to the credential.

**Fix →** Click **Configure SSO → Authorize** for the organisation, sign in through your identity provider, then retry the Git command. For `gh`, run `gh auth refresh -s repo`.

**Prevent →** Document the SSO authorisation step for new joiners and use short-lived tokens.

</details>

### Q13. `Received disconnect from 140.82.x.x port 22: Too many authentication failures` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Your SSH agent holds many keys, and the server drops the connection after the allowed number of attempts before the right key is tried.

**Diagnose →** `ssh-add -l | wc -l` shows how many keys are offered; `ssh -vT git@github.com` shows the attempt order.

**Fix →** Offer only the intended key: set `IdentityFile` and `IdentitiesOnly yes` in the `Host` block, or run `ssh -o IdentitiesOnly=yes -i ~/.ssh/id_ed25519 -T git@github.com`.

**Prevent →** Per-host SSH config rather than loading every key in a global agent.

</details>

### Q14. `WARNING: UNPROTECTED PRIVATE KEY FILE! Permissions 0644 for '~/.ssh/id_ed25519' are too open.` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The private key is readable by other users, so SSH refuses to use it.

**Diagnose →** `ls -l ~/.ssh`.

**Fix →**
```bash
chmod 700 ~/.ssh
chmod 600 ~/.ssh/id_ed25519
chmod 644 ~/.ssh/id_ed25519.pub
```
On WSL, files under `/mnt/c` cannot hold Unix modes; copy the key into the Linux home directory.

**Prevent →** Create keys with `ssh-keygen` in the proper directory and never share private keys.

</details>

### Q15. `fatal: not a git repository (or any of the parent directories): .git` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** You are in the wrong directory, the repo was never initialised, or the `.git` folder was deleted.

**Diagnose →**
```bash
pwd
git rev-parse --show-toplevel
ls -la | grep .git
```
**Fix →** `cd` into the project, or run `git init` for a new one. If `.git` was removed, re-clone and copy your working files over.

**Prevent →** Show the branch in your shell prompt so you notice when you are outside a repo.

</details>

### Q16. `error: remote origin already exists.` when adding a remote. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** A remote named `origin` is already configured, often pointing at a template or old URL.

**Diagnose →** `git remote -v`.

**Fix →** Change the URL instead of adding: `git remote set-url origin git@github.com:ORG/REPO.git`. Or rename the old one: `git remote rename origin old-origin`, then add the new `origin`.

**Prevent →** Check `git remote -v` before cloning from templates and adding remotes.

</details>

### Q17. `fatal: 'origin' does not appear to be a git repository` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** No remote called `origin` exists (a repo made with `git init`), the name is misspelt, or the URL is malformed.

**Diagnose →** `git remote -v` and `git config --get remote.origin.url`.

**Fix →** `git remote add origin git@github.com:ORG/REPO.git`, then `git push -u origin main`.

**Prevent →** Prefer `git clone` for existing projects, which configures `origin` for you.

</details>

### Q18. `remote: Permission to org/repo.git denied to user.` then `The requested URL returned error: 403` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Your account has read-only access, the token has no write scope, or an org policy restricts classic tokens.

**Diagnose →**
```bash
gh api repos/ORG/REPO --jq .permissions
gh auth status
```
**Fix →** Ask for write access, or create a fine-grained token with "Contents: Read and write" for that repo, and authorise SSO if required. If you only have read access, push to a fork and open a PR.

**Prevent →** Grant access through teams, not individuals.

</details>

### Q19. `fatal: unable to access '...': Failed to connect to proxy` or `Proxy CONNECT aborted` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Wrong proxy address in Git config or environment, proxy needs authentication, or the proxy does not allow the host.

**Diagnose →**
```bash
git config --get-regexp 'http.*proxy'
env | grep -i proxy
curl -v -x "$HTTPS_PROXY" https://github.com 2>&1 | tail
```
**Fix →** Set the correct value (`git config --global http.proxy http://user:pass@proxy:8080`) or remove it (`git config --global --unset http.proxy`) when off the corporate network. Add `NO_PROXY` entries for internal hosts.

**Prevent →** Avoid hard-coding proxies globally on laptops that move between networks.

</details>

### Q20. `error: RPC failed; HTTP 400 curl 22` / `send-pack: unexpected disconnect while reading sideband packet` on a large push over HTTPS. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** A very large pack exceeds the HTTP buffer or a proxy or load balancer limit, or the connection drops mid-upload.

**Diagnose →**
```bash
GIT_TRACE_PACKET=1 GIT_CURL_VERBOSE=1 git push 2>&1 | tail -30
git count-objects -vH
```
**Fix →** Switch to SSH for the push. Or raise the buffer: `git config http.postBuffer 524288000`. Push in slices: `git push origin <older-sha>:refs/heads/main` then the rest. Find and remove accidental big blobs before pushing.

**Prevent →** Keep large files out of Git (use LFS) and set a repo-size policy.

</details>

### Q21. `fatal: early EOF` and `fatal: fetch-pack: invalid index-pack output` during clone. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The connection was dropped mid-transfer: flaky network, proxy timeouts, or memory limits during pack generation.

**Diagnose →** Retry with `GIT_TRACE=1 GIT_CURL_VERBOSE=1 git clone ...`; check whether the failure is always at the same point.

**Fix →**
```bash
git clone --depth 1 URL repo && cd repo
git fetch --unshallow          # or --deepen=1000 in steps
```
Switch to SSH, or raise `git config --global pack.windowMemory 256m`. Use `--filter=blob:none` for huge repos.

**Prevent →** Use shallow or partial clones in CI and on weak links.

</details>

### Q22. In WSL, `git push` keeps prompting for credentials even though Windows already has them stored. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** WSL Git uses its own config and does not know about Git Credential Manager on Windows.

**Diagnose →** `git config --show-origin --get-all credential.helper` inside WSL.

**Fix →** Point WSL at the Windows helper:
```bash
git config --global credential.helper \
  "/mnt/c/Program\ Files/Git/mingw64/bin/git-credential-manager.exe"
```
Or use SSH keys stored in the WSL home directory.

**Prevent →** Pick one place (WSL or Windows) to run Git per project and stay consistent.

</details>

### Q23. `sign_and_send_pubkey: signing failed for ED25519 ... from agent: agent refused operation` 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The agent (ssh-agent, 1Password, Secretive, a hardware key) refused to sign: it is locked, confirmation timed out or was denied, or the key requires a touch.

**Diagnose →**
```bash
echo $SSH_AUTH_SOCK
ssh-add -l
ssh -vvv -T git@github.com 2>&1 | grep -i -E "agent|sign"
```
**Fix →** Unlock the agent or approve the prompt; replug the hardware key and touch it when it blinks. If `SSH_AUTH_SOCK` points to a dead socket, restart the agent (`eval "$(ssh-agent -s)"` and `ssh-add`).

**Prevent →** Use one agent consistently and avoid stale sockets in long-lived tmux sessions.

</details>

### Q24. Adding a deploy key to a second repository fails with "Key is already in use". 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** GitHub allows a given public key to be a deploy key on only one repository (and it cannot also be a user key).

**Diagnose →** Compare fingerprints: `ssh-keygen -lf key.pub` and check the other repo's deploy keys.

**Fix →** Generate a separate key per repo (`ssh-keygen -t ed25519 -f ~/.ssh/deploy_repo2`) and use host aliases in `~/.ssh/config` to select them. For many repos, use a GitHub App or a machine account instead.

**Prevent →** Naming keys by repo in secrets management avoids reuse attempts.

</details>

### Q25. `git clone` over HTTPS works, but the SSH URL fails with `Permission denied` or a timeout. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Port 22 is blocked, no SSH key is registered, or the key belongs to an account without access. HTTPS uses a different credential path.

**Diagnose →**
```bash
ssh -vT git@github.com
nc -vz github.com 22
```
**Fix →** Use port 443 (`ssh.github.com`, see Q7), add the right key, or stay on HTTPS with a credential helper.

**Prevent →** Standardise on one protocol per team and document it.

</details>

## Push, Pull and Fetch Errors

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q26. `! [rejected] main -> main (fetch first)` / `(non-fast-forward)` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The remote branch has commits you do not have locally, so your push would discard them.

**Diagnose →**
```bash
git fetch origin
git log --oneline --graph HEAD origin/main -10
```
**Fix →** Integrate first: `git pull --rebase` (or `git merge origin/main`), fix conflicts, test, then `git push`. Do not use `--force` on a shared branch.

**Prevent →** Pull before you start work and keep branches short-lived.

</details>

### Q27. `remote: error: GH006: Protected branch update failed for refs/heads/main.` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Branch protection requires a pull request, reviews or status checks and you pushed directly.

**Diagnose →** Read the lines after GH006: they say which rule failed (for example "Changes must be made through a pull request" or "Required status check ... is expected").

**Fix →**
```bash
git switch -c fix/my-change
git push -u origin fix/my-change
gh pr create --fill
```
Merge once checks and reviews pass.

**Prevent →** Make feature branches the habit, and keep local hooks to warn on commits to `main`.

</details>

### Q28. `! [rejected] feature -> feature (stale info)` with `--force-with-lease`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The remote branch has a commit that your remote-tracking ref does not know about (a teammate pushed), so the lease check failed. That is the safety net working.

**Diagnose →**
```bash
git fetch origin
git log --oneline feature..origin/feature   # what you would overwrite
```
**Fix →** Inspect the new commits. If wanted, `git rebase origin/feature` to include them and push again. If unwanted and agreed, re-run the push after fetching.

**Prevent →** Avoid IDE background fetch before push, which hides the divergence by refreshing the ref, and tell the team before rewriting a shared branch.

</details>

### Q29. `You are not currently on a branch.` when running `git pull` or `git push`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** HEAD is detached, usually after checking out a tag or commit, or in the middle of a rebase.

**Diagnose →** `git status` ("HEAD detached at ...") and `git branch --show-current` (empty).

**Fix →** To keep commits made here: `git switch -c my-branch`. To leave: `git switch main`. If a rebase is in progress, finish with `git rebase --continue` or abort.

**Prevent →** Use `git switch -c` before experimenting, and show the branch in your prompt.

</details>

### Q30. `There is no tracking information for the current branch.` when running `git pull`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The local branch has no configured upstream, typically because it was created locally and never pushed with `-u`.

**Diagnose →** `git branch -vv` shows no `[origin/...]` next to the branch.

**Fix →**
```bash
git branch --set-upstream-to=origin/feature feature
# or for first push
git push -u origin feature
```
**Prevent →** Set `git config --global push.autoSetupRemote true` (Git 2.37+).

</details>

### Q31. `fatal: The current branch feature has no upstream branch.` on `git push`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** First push of a new branch: Git does not know where to send it.

**Diagnose →** `git remote -v` and `git branch -vv`.

**Fix →** `git push -u origin HEAD` (Git even prints the exact command). Next pushes need no arguments.

**Prevent →** `git config --global push.autoSetupRemote true`.

</details>

### Q32. `error: Your local changes to the following files would be overwritten by merge:` during `git pull`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** You have uncommitted edits in files that the incoming commits also change.

**Diagnose →** `git status` and `git diff --stat`.

**Fix →**
```bash
git stash push -m "wip before pull"
git pull --rebase
git stash pop
```
Or commit your work first. Discard only if the changes are unwanted: `git restore <file>`.

**Prevent →** Commit or stash before pulling, or use `git pull --rebase --autostash`.

</details>

### Q33. `fatal: Need to specify how to reconcile divergent branches.` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Local and remote both have new commits, and Git (2.27+) refuses to guess between merge, rebase and fast-forward-only.

**Diagnose →** `git fetch && git status` shows "have diverged".

**Fix →** Choose per command: `git pull --rebase`, `git pull --no-rebase`, or `git pull --ff-only`. Set the default: `git config --global pull.rebase true`.

**Prevent →** Put the setting in your team's recommended config.

</details>

### Q34. `error: cannot pull with rebase: You have unstaged changes.` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `pull --rebase` needs a clean working tree and index.

**Diagnose →** `git status`.

**Fix →**
```bash
git pull --rebase --autostash
# or
git stash && git pull --rebase && git stash pop
```
**Prevent →** `git config --global rebase.autoStash true`.

</details>

### Q35. `remote: error: File data/dump.sql is 138.42 MB; this exceeds GitHub's file size limit of 100.00 MB` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** A commit in the push contains a file over 100 MB. Deleting it in a later commit does not help, since the blob is still in the pushed history.

**Diagnose →**
```bash
git rev-list --objects origin/main..HEAD | \
  git cat-file --batch-check='%(objectname) %(objectsize) %(rest)' | sort -k2 -n -r | head
git log --oneline -- data/dump.sql
```
**Fix →** Remove it from the unpushed commits (`git rebase -i`, or `git filter-repo --path data/dump.sql --invert-paths` for older history). Re-add via Git LFS if it must be versioned: `git lfs track` then `git lfs migrate import --include="data/dump.sql"`.

**Prevent →** Add the pattern to `.gitignore`, and use a pre-commit `check-added-large-files` hook.

</details>

### Q36. `error: src refspec main does not match any` / `failed to push some refs` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** There is no local branch named `main`: no commits yet, or your branch is called `master`.

**Diagnose →** `git branch` (may show nothing or `master`) and `git log -1`.

**Fix →** Make an initial commit, or rename: `git branch -M main`, then `git push -u origin main`.

**Prevent →** `git config --global init.defaultBranch main`.

</details>

### Q37. `! [rejected] v1.2.0 -> v1.2.0 (already exists)` when pushing a tag. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** A tag with that name exists on the remote pointing to a different commit.

**Diagnose →**
```bash
git ls-remote --tags origin v1.2.0
git rev-parse v1.2.0^{}
```
**Fix →** If the remote tag is correct, fetch it (`git fetch origin tag v1.2.0`) and stop. If you really need to move it and nobody consumed it, `git push origin :refs/tags/v1.2.0` then re-push. Otherwise use a new version.

**Prevent →** Make tags immutable with a tag ruleset, and let CI create release tags.

</details>

### Q38. `fatal: couldn't find remote ref feature/x` on fetch or pull. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The branch was deleted or renamed on the remote, or the name is misspelt.

**Diagnose →**
```bash
git ls-remote --heads origin | grep feature
git branch -vv
```
**Fix →** Fetch with prune (`git fetch --prune`), repoint your branch (`git branch -u origin/<new-name>`) or push it again if it should still exist.

**Prevent →** Auto-prune with `git config --global fetch.prune true`.

</details>

### Q39. `error: cannot lock ref 'refs/remotes/origin/feature/x': 'refs/remotes/origin/feature' exists; cannot create 'refs/remotes/origin/feature/x'` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Refs are stored as files, so `feature` and `feature/x` cannot coexist. A branch `feature` was deleted remotely and `feature/x` created, but your stale remote-tracking ref `origin/feature` still exists.

**Diagnose →** `git branch -r | grep feature` and `git ls-remote --heads origin 'feature*'`.

**Fix →** `git remote prune origin` (or `git fetch --prune`) and fetch again. Also delete a conflicting local branch if one exists.

**Prevent →** Agree on branch naming that avoids a name being both a branch and a prefix, and enable `fetch.prune`.

</details>

### Q40. `remote: This repository moved. Please use the new location:` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The repo was renamed or transferred; GitHub redirects, but your remote still has the old URL.

**Diagnose →** `git remote -v`.

**Fix →** `git remote set-url origin <new URL printed in the message>`.

**Prevent →** Update remotes, CI configs, submodule URLs and docs after renames or transfers.

</details>

### Q41. `git push` prints `Everything up-to-date` but your changes are not on GitHub. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Nothing is committed (changes only in the working tree), you committed on a different branch, or you are pushing a branch other than the one you changed.

**Diagnose →**
```bash
git status
git branch -vv
git log --oneline -3 --all --decorate
```
**Fix →** Commit the changes, switch to the right branch, or push explicitly: `git push origin <branch>`.

**Prevent →** Look at `git status -sb` before pushing.

</details>

### Q42. `error: object file .git/objects/ab/cdef... is empty` / `fatal: bad object HEAD` / `error: bad object refs/heads/main` 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Repository corruption: a crash, disk full, or a killed process left zero-byte or truncated objects.

**Diagnose →**
```bash
git fsck --full
find .git/objects -type f -empty
```
**Fix →** Delete the empty objects (`find .git/objects -type f -empty -delete`), then `git fetch origin` to re-download missing ones. If a ref is bad, repoint it: `git update-ref refs/heads/main origin/main`. If it is hopeless, clone again and copy over unpushed work (patch with `git format-patch` from the backup or reflog).

**Prevent →** Keep disks from filling, avoid syncing `.git` with cloud tools (Dropbox, OneDrive), and push often.

</details>

### Q43. `error: Your local changes to the following files would be overwritten by checkout:` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The target branch has a different version of a file that you have modified locally.

**Diagnose →** `git status` and `git diff <file>`.

**Fix →** Commit or stash first (`git stash push -m "wip"`), switch, then `git stash pop`. Use `git switch -m <branch>` for a three-way merge of your changes into the new branch.

**Prevent →** Keep the working tree clean before switching.

</details>

### Q44. `fatal: refusing to fetch into current branch refs/heads/main of non-bare repository` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** A refspec such as `git fetch origin main:main` tries to update the branch you currently have checked out.

**Diagnose →** `git branch --show-current` and the refspec you typed.

**Fix →** Use `git pull`, or `git fetch origin` and then merge. To update another branch without leaving yours, `git fetch origin main:main` works only when `main` is not checked out. Otherwise `git switch main && git pull --ff-only`.

**Prevent →** Know that `fetch` never touches your working branch unless you tell it to.

</details>

### Q45. `fatal: cannot do a partial commit during a merge.` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** You ran `git commit <file>` or `git commit -o` while a merge is unfinished.

**Diagnose →** `git status` shows "All conflicts fixed but you are still merging".

**Fix →** Stage everything that belongs to the merge and use a plain `git commit` (or `git merge --continue`). Do not commit files individually during a merge.

**Prevent →** Complete or abort merges before making unrelated commits.

</details>

### Q46. `remote: fatal: pack exceeds maximum allowed size` 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The push contains too much data in one pack, for example a first push of a huge repo or an accidentally committed dataset.

**Diagnose →**
```bash
git count-objects -vH
git rev-list --objects --all | git cat-file --batch-check='%(objectsize) %(rest)' | sort -n -r | head
```
**Fix →** Find and remove big blobs (`git filter-repo --strip-blobs-bigger-than 50M` on a clone), or push in increments by branch or history slices. Move large assets to Git LFS.

**Prevent →** Push early and often, and enforce size limits with rulesets.

</details>

### Q47. `fatal: the remote end hung up unexpectedly` with `send-pack: unexpected disconnect while reading sideband packet` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Network drop, a server-side hook crash, a proxy timeout or oversized pack.

**Diagnose →**
```bash
GIT_TRACE_PACKET=1 GIT_TRACE=1 git push 2>&1 | tail -40
```
Check whether a `remote:` message appears just before the disconnect.

**Fix →** Retry on a stable connection or over SSH. If a size issue, see Q20 or Q46. If a server hook fails, the `remote:` lines usually tell you what to fix.

**Prevent →** Use SSH for large pushes, and keep a stable network for big transfers.

</details>

### Q48. `fatal: The upstream branch of your current branch does not match the name of your current branch.` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** With `push.default=simple`, Git refuses to push when your local branch tracks a remote branch with a different name (`fix-bug` tracking `origin/main`).

**Diagnose →** `git branch -vv` shows the mismatch.

**Fix →** Push to the same name: `git push origin HEAD`, or to the specific target: `git push origin fix-bug:fix-bug`. Fix the tracking: `git branch --set-upstream-to=origin/fix-bug`.

**Prevent →** Create branches with `git switch -c name` off `origin/main` using `--no-track`, so they are not accidentally tracking `main`.

</details>

### Q49. `git pull` says `Already up to date.`, yet teammates' new commits are visible on GitHub. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Your branch tracks the wrong upstream, the fetch refspec is narrowed to certain branches, you are on a different branch, or the remote URL points to a fork or mirror.

**Diagnose →**
```bash
git remote -v
git config --get-all remote.origin.fetch
git branch -vv
git fetch origin --verbose
```
**Fix →** Reset the refspec to `+refs/heads/*:refs/remotes/origin/*` (`git remote set-branches origin '*'`), correct the URL, and fix tracking with `git branch -u origin/<branch>`.

**Prevent →** Avoid hand-editing `.git/config`, and use `--single-branch` clones only when you mean it.

</details>

### Q50. `warning: refname 'release' is ambiguous.` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** A branch and a tag (or two other refs) share the same name, so Git cannot tell which you mean.

**Diagnose →**
```bash
git show-ref release
git for-each-ref | grep release
```
**Fix →** Be explicit: `refs/heads/release` or `refs/tags/release`. Rename one: `git tag -d release` and recreate with a clear name, or `git branch -m release release-2.0`.

**Prevent →** Use prefixes (`release/2.0`, tags `v2.0.0`) so names never collide.

</details>

## Merge and Rebase Problems

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q51. Production failed to start, and the file contains `<<<<<<< HEAD` lines that were committed. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Conflict markers were left in the file, staged with `git add .`, and committed without anyone reading the diff.

**Diagnose →**
```bash
git grep -nE '^(<<<<<<<|=======|>>>>>>>)( |$)'
git log -S'<<<<<<< HEAD' --oneline -- path/to/file
```
**Fix →** Resolve the file by hand (use `git show <merge-sha>^1:path` and `^2:path` to see both sides), commit the fix, and deploy. If it is on shared `main`, fix forward quickly.

**Prevent →** Use the `check-merge-conflict` pre-commit hook and run linters and tests in CI before merge.

</details>

### Q52. `error: you need to resolve your current index first` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Some files are still marked unmerged (`U`) from an earlier conflicted merge, rebase, stash pop or cherry-pick.

**Diagnose →** `git status` lists "Unmerged paths"; `git diff --name-only --diff-filter=U`.

**Fix →** Resolve each file and `git add` it. If you want to start over: `git merge --abort`, `git rebase --abort`, or `git reset --merge`.

**Prevent →** Finish or abort an operation before starting another.

</details>

### Q53. `fatal: You have not concluded your merge (MERGE_HEAD exists).` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** A merge stopped on conflicts and you tried to `pull`, `commit` selectively or switch branches.

**Diagnose →** `git status` shows "You have unmerged paths"; `ls .git/MERGE_HEAD` exists.

**Fix →** Resolve conflicts, `git add`, then `git merge --continue` (or `git commit`). To abandon: `git merge --abort`.

**Prevent →** Keep the working tree clean before merging and finish what you start.

</details>

### Q54. Rebase stops: `error: could not apply 1a2b3c4... Add retry` and later `No changes - did you forget to use 'git add'?` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The first message is a normal conflict. The second appears when you run `git rebase --continue` but the resolved files were not staged, or the resolution made the commit empty.

**Diagnose →**
```bash
git status
git diff --name-only --diff-filter=U
git diff --cached --stat
```
**Fix →** `git add <resolved files>` then `git rebase --continue`. If the commit became empty because upstream already has the change, use `git rebase --skip`.

**Prevent →** Resolve, stage, and only then continue; check `git status` between each step.

</details>

### Q55. `fatal: It seems that there is already a rebase-merge directory, and I wonder if you are in the middle of another rebase.` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** A previous rebase was interrupted (terminal closed, crash, conflict never resolved) and left state in `.git/rebase-merge/`.

**Diagnose →** `git status` ("interactive rebase in progress") and `ls .git/rebase-merge`.

**Fix →** Continue with `git rebase --continue`, abort with `git rebase --abort`, or keep the current HEAD and drop the state with `git rebase --quit`. If Git cannot read the state, remove `.git/rebase-merge` after saving anything important and check the branch with `git reflog`.

**Prevent →** Do not run two Git processes in the same repo (IDE plus terminal).

</details>

### Q56. `CONFLICT (modify/delete): config/old.yml deleted in HEAD and modified in feature.` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** One side deleted (or moved) the file while the other edited it.

**Diagnose →** `git log --diff-filter=D --oneline -- config/old.yml` finds who deleted it and why; `git diff main...feature -- config/old.yml` shows the edits.

**Fix →** Decide: keep the deletion (`git rm config/old.yml`), or keep the file (`git add config/old.yml`). If the file moved, apply the edit to the new location by hand.

**Prevent →** Announce file moves and deletions, and do renames in their own commits.

</details>

### Q57. `CONFLICT (file/directory): There is a directory with name docs in HEAD. Adding docs as docs~feature instead.` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** One branch has a file called `docs`, the other has a directory `docs/`.

**Diagnose →** `git ls-tree HEAD docs` and `git ls-tree feature docs` show which side is the file.

**Fix →** Choose one layout. Keep the directory and move the stray file: `git mv docs~feature docs/README.md` (or delete it), then `git add` and continue.

**Prevent →** Coordinate repo layout changes and avoid names that turn from file into folder.

</details>

### Q58. `CONFLICT (add/add): Merge conflict in README.md` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Both branches created the same path independently (typical for the initial README or `.gitignore`).

**Diagnose →** `git diff --diff-filter=U` and `git show :2:README.md`, `git show :3:README.md` (stage 2 is ours, stage 3 is theirs).

**Fix →** Merge the two versions by hand, or choose one with `git checkout --ours README.md` / `--theirs`, then `git add README.md` and continue.

**Prevent →** For new repos, create the remote repo empty, or clone it first.

</details>

### Q59. A rebase of a branch with merge commits keeps replaying the same conflicts and resurrects old changes. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Default rebase flattens merge commits and re-applies every commit from both sides, so earlier hand-resolved conflicts reappear, and changes that were reverted can come back.

**Diagnose →**
```bash
git log --oneline --graph --merges origin/main..HEAD
git rev-list --merges --count origin/main..HEAD
```
**Fix →** Abort. Either keep the topology with `git rebase --rebase-merges origin/main`, or avoid rebasing: `git merge origin/main`. Another option is squash first (`git reset --soft $(git merge-base HEAD origin/main)`, recommit) and rebase one commit. Enable `rerere` to remember resolutions.

**Prevent →** Do not merge `main` into a branch you plan to rebase; rebase it from the start.

</details>

### Q60. `error: Committing is not possible because you have unmerged files.` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** You ran `git commit` while files are still conflicted.

**Diagnose →** `git status` ("Unmerged paths"), `git diff --name-only --diff-filter=U`.

**Fix →** Edit each file to remove markers, `git add` it, then commit. For a deleted-vs-modified conflict, use `git rm` or `git add`.

**Prevent →** Run `git status` after resolving; nothing should be listed under "Unmerged paths".

</details>

### Q61. After a merge, a colleague's change is silently missing and nobody saw a conflict. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** A bad hand-resolution kept only one side; `-X ours` or `-s ours` was used; an earlier reverted merge masks commits (Git thinks they are already in); or the commit was dropped during a rebase or squash.

**Diagnose →**
```bash
git log --full-history --oneline -S'missing_function' -- path/to/file
git show --cc <merge-sha>
git merge-base --is-ancestor <lost-commit> HEAD && echo "ancestor, so content was lost in a resolution"
git log --merges --grep='Revert' --oneline
```
**Fix →** Re-apply the lost change with `git cherry-pick <lost-commit>` (or revert the earlier revert-of-merge). Add a regression test.

**Prevent →** Review merge commits with `--cc`, avoid blanket `-X` strategies, and require passing CI on the merged result.

</details>

### Q62. `warning: Cannot merge binary files: assets/logo.png (HEAD vs. feature)` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Both branches changed the same binary file, and Git cannot do a textual three-way merge.

**Diagnose →** `git diff --name-only --diff-filter=U` and `git log --oneline main..feature -- assets/logo.png`.

**Fix →** Pick a version: `git checkout --ours assets/logo.png` or `--theirs`, or open both versions and produce a combined one. Then `git add` and continue.

**Prevent →** Use Git LFS locking (`git lfs lock`) for assets that cannot be merged.

</details>

### Q63. You merged `main` locally, but the PR on GitHub still says "This branch has conflicts that must be resolved." 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** You merged a stale local `main` (not `origin/main`), you did not push, `main` moved again, or the PR's base branch is not `main`.

**Diagnose →**
```bash
git fetch origin
git log --oneline HEAD..origin/main     # still missing commits?
git status -sb                          # ahead/behind your upstream
```
Check the PR's base branch at the top of the PR page.

**Fix →** `git merge origin/main` (or rebase), resolve, push. For the web editor, use "Resolve conflicts" for simple text conflicts.

**Prevent →** Always merge `origin/main` after `git fetch`, not a local `main` that may be old.

</details>

### Q64. `git merge feature` prints `Already up to date.` although the feature branch clearly has new work. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The branch is already an ancestor of your current branch, you are merging a stale local copy instead of `origin/feature`, or you are on the wrong branch.

**Diagnose →**
```bash
git branch --show-current
git merge-base --is-ancestor feature HEAD && echo "already merged"
git log --oneline HEAD..origin/feature
```
**Fix →** `git fetch`, merge `origin/feature`, and double-check the branch you are on. If the work was squash-merged earlier, compare content with `git diff HEAD feature`.

**Prevent →** Fetch first, and verify the branch name in your prompt.

</details>

### Q65. `fatal: Not possible to fast-forward, aborting.` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `pull.ff=only` (or `--ff-only`) is set, and your local branch has commits that the remote does not.

**Diagnose →** `git fetch && git status -sb` shows ahead and behind counts.

**Fix →** Pick how to integrate: `git pull --rebase` or `git merge origin/main`. If your local commits are junk, `git reset --hard origin/main`.

**Prevent →** `ff-only` is a good setting, because it makes you choose consciously.

</details>

### Q66. Git hangs with `hint: Waiting for your editor to close the file...` or fails with `error: There was a problem with the editor 'vi'`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Git opened an editor (for a commit message, rebase todo or merge message) that is running in another window, not in the terminal, or no editor is installed. GUI editors need a "wait" flag.

**Diagnose →** `git config --get core.editor`, `echo $EDITOR $GIT_EDITOR`.

**Fix →** Save and close the editor. To set one: `git config --global core.editor "code --wait"` or `"nano"`. In Vim, `:wq` saves and `:cq` aborts. For scripts, `git commit -m "msg"` or `GIT_EDITOR=true git rebase --continue`.

**Prevent →** Set the editor once globally.

</details>

### Q67. `git rebase -i` fails with `error: invalid line 3: ...` or the todo list says `Nothing to do`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** You edited a line into an invalid command (typo, missing SHA, extra text), or the range contains no commits (you passed the wrong base, or HEAD equals the base).

**Diagnose →**
```bash
git log --oneline origin/main..HEAD     # what the range contains
git rebase --edit-todo                  # reopen the todo list
```
**Fix →** Use valid commands (`pick`, `reword`, `edit`, `squash`, `fixup`, `drop`, `exec`). If the list is empty, pass the correct base, e.g. `git rebase -i HEAD~5` or `--root`. Deleting every line aborts the rebase.

**Prevent →** Do not delete lines to drop commits; use `drop`.

</details>

### Q68. A recorded `rerere` resolution is wrong and gets re-applied automatically on every rebase. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** rerere stored an earlier, incorrect resolution for that conflict and replays it silently.

**Diagnose →**
```bash
git rerere status
git rerere diff
ls .git/rr-cache
```
**Fix →** Forget the bad resolution and redo the conflict: `git rerere forget path/to/file`, then `git checkout --merge path/to/file` to bring the markers back and resolve again. As a last resort, delete the specific folder under `.git/rr-cache`.

**Prevent →** Review the files rerere resolved (`git diff`) before staging them, and clear the cache after major refactors.

</details>

### Q69. `error: could not read '.git/rebase-merge/head-name': No such file or directory` 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Rebase state is inconsistent: the rebase directory was partly deleted, a second Git process interfered, or a crash corrupted it.

**Diagnose →**
```bash
ls -la .git/rebase-merge .git/rebase-apply 2>/dev/null
git status
git reflog -15
```
**Fix →** Record the pre-rebase tip from the reflog (`rebase (start)` entry, minus one). Remove the broken state (`rm -rf .git/rebase-merge`), then `git switch <branch>` and `git reset --hard <pre-rebase-sha>`. Redo the rebase carefully.

**Prevent →** Do not run IDE Git actions and terminal Git commands in parallel during a rebase.

</details>

### Q70. After a merge, the same function appears twice in the file. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Both branches added the same function at different places (no textual conflict), or a resolution kept both sides.

**Diagnose →**
```bash
git diff <merge-sha>^1 <merge-sha> -- file
git log --oneline -S'def calculate_tax' -- file
```
**Fix →** Remove the duplicate, run tests and linters, and commit. Check the other branch's version to see which has the newer logic.

**Prevent →** Run the build and tests on the merge result in CI. Duplicate definitions are often caught by a compiler or linter.

</details>

### Q71. Generated files (`CHANGELOG.md`, lockfiles) conflict in almost every merge. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Several branches append to the same region of a file, or tooling rewrites the whole file.

**Diagnose →** `git log --oneline -20 -- CHANGELOG.md` shows how often it changes; `git diff --name-only --diff-filter=U` during merges.

**Fix →** For append-only files: `.gitattributes` with `CHANGELOG.md merge=union` keeps both sides. For generated files, resolve source files then regenerate. Or stop editing the file by hand: generate the changelog at release time.

**Prevent →** Generate changelogs from commit history (release-please, changesets) rather than editing in every PR.

</details>

### Q72. `error: commit 8f3a2c1 is a merge but no -m option was given.` on `git cherry-pick`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The commit has two parents and Git does not know which diff to apply.

**Diagnose →** `git show --no-patch --format='%h parents: %p' 8f3a2c1`.

**Fix →** `git cherry-pick -m 1 8f3a2c1` applies the changes relative to the first parent (the branch that was merged into). Or cherry-pick the individual commits of the merged branch.

**Prevent →** Use squash merges so every PR is a single normal commit.

</details>

### Q73. `git mergetool` says `The merge tool vimdiff is not available` or leaves `*.orig` files everywhere. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The configured tool is not installed or on PATH, and mergetool keeps backups by default.

**Diagnose →**
```bash
git config --get merge.tool
git mergetool --tool-help
```
**Fix →** Install and set a tool, for example `git config --global merge.tool vscode` with `mergetool.vscode.cmd 'code --wait --merge $REMOTE $LOCAL $BASE $MERGED'`. Stop backup files: `git config --global mergetool.keepBackup false`. Remove existing ones: `git clean -n '*.orig'` then `-f`.

**Prevent →** Add `*.orig` to the global ignore file.

</details>

### Q74. After `git pull` on a branch someone rebased, your history contains every commit twice. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Remote history was rewritten, so the old commits (yours) and the new ones (rewritten) have different SHAs. A merge pull combined both lines.

**Diagnose →**
```bash
git log --oneline --graph -20
git log --cherry-pick --left-right --oneline HEAD...origin/feature
```
**Fix →** If you have no local work: `git reset --hard origin/feature`. With local commits: `git rebase --onto origin/feature <old-remote-tip> feature` (find the old tip in `git reflog show origin/feature`). Undo the merge first with `git reset --hard ORIG_HEAD`.

**Prevent →** Use `git pull --rebase` on branches that might be rewritten, and tell the team when you force-push.

</details>

### Q75. `fatal: invalid upstream 'origin/main'` when you run `git rebase origin/main`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The remote-tracking ref does not exist locally: you never fetched, the remote is named differently, or the default branch is `master`.

**Diagnose →**
```bash
git branch -r
git remote -v
git ls-remote --heads origin
```
**Fix →** `git fetch origin`, or use the right name (`origin/master`). Repair the HEAD pointer if needed: `git remote set-head origin -a`.

**Prevent →** Fetch before rebasing.

</details>

## Lost Work, HEAD and History Problems

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q76. `fatal: Unable to create '/repo/.git/index.lock': File exists.` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Another Git process is running (IDE, GUI client, background fetch), or a previous command crashed and left the lock file.

**Diagnose →**
```bash
ps aux | grep -i '[g]it'
ls -l .git/index.lock
```
**Fix →** If no Git process is running, remove the stale lock: `rm .git/index.lock`. If one is running, wait or close that tool.

**Prevent →** Avoid running parallel Git operations from several tools, and do not kill Git mid-operation.

</details>

### Q77. You added a file to `.gitignore`, but `git status` still shows it as modified. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The file is already tracked. Ignore rules apply only to untracked files.

**Diagnose →**
```bash
git ls-files --error-unmatch path/to/file
git check-ignore -v path/to/file
```
**Fix →** `git rm --cached path/to/file`, commit, and the ignore rule takes over. For a whole folder, `git rm -r --cached dir/`.

**Prevent →** Write `.gitignore` before the first commit, using a template for your language.

</details>

### Q78. `fatal: your current branch 'main' does not have any commits yet` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** A fresh repo with no commit; commands like `git log` or `git branch -m` need a commit first.

**Diagnose →** `git status` shows "No commits yet on main".

**Fix →** Stage and commit: `git add . && git commit -m "Initial commit"`. Then push with `-u`.

**Prevent →** Create an initial commit (README, `.gitignore`) as part of repo setup.

</details>

### Q79. `Author identity unknown *** Please tell me who you are.` and `fatal: unable to auto-detect email address` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `user.name` and `user.email` are not set (new machine, container, CI runner).

**Diagnose →** `git config --list --show-origin | grep user.`

**Fix →**
```bash
git config --global user.name "Your Name"
git config --global user.email "you@example.com"
```
For one repo, omit `--global`. In CI, set them before committing.

**Prevent →** Put identity into your dotfiles or CI bootstrap script.

</details>

### Q80. `fatal: ambiguous argument 'HEAD~1': unknown revision or path not in the working tree.` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The repo has only one commit, so there is no parent; or you are in a shallow clone, or the revision was misspelt.

**Diagnose →**
```bash
git rev-list --count HEAD
git rev-parse --is-shallow-repository
```
**Fix →** In a one-commit repo, use `git update-ref -d HEAD` to undo the first commit (keeps files staged), or `git commit --amend`. For shallow clones: `git fetch --deepen=10`.

**Prevent →** Use `fetch-depth: 0` in CI jobs that need history.

</details>

### Q81. Your commits do not show on your GitHub profile contribution graph. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The commit email is not linked to your GitHub account, commits are on a fork's non-default branch, the repo is private (and private contributions are hidden), or it was not merged to the default branch.

**Diagnose →**
```bash
git log --format='%h %ae %an' -5
git config user.email
```
**Fix →** Add the email under GitHub Settings → Emails, or change the config (`git config user.email`) and keep `user.useConfigOnly true` to avoid guessing. For privacy use the `ID+username@users.noreply.github.com` address.

**Prevent →** Per-directory identities with `includeIf`, and verify the email after setting up a new laptop.

</details>

### Q82. `fatal: 'main' is already checked out at '/home/dev/app-hotfix'` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Git forbids the same branch being checked out in two worktrees at once.

**Diagnose →** `git worktree list`.

**Fix →** Use a different branch in the second worktree (`git worktree add -b hotfix ../app-hotfix origin/main`), or remove the other worktree (`git worktree remove ../app-hotfix`). Stale entries: `git worktree prune`. As a last resort, `--force`, but understand the risk.

**Prevent →** Create a new branch per worktree, and remove worktrees when done.

</details>

### Q83. `git reset --hard HEAD@{5}` fails with `fatal: bad revision 'HEAD@{5}'` or `git reflog` is empty. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Fewer than six reflog entries exist, the reflog expired or was cleared (`git reflog expire`, `gc`), it is a fresh clone (reflogs are local), or you are in a different repository or worktree.

**Diagnose →**
```bash
git reflog | head
git config --get core.logAllRefUpdates
git reflog show origin/main
```
**Fix →** Look at other reflogs (`git reflog show <branch>`), search dangling commits with `git fsck --lost-found`, or fetch the lost work from a teammate or backup. Use dates: `HEAD@{yesterday}`.

**Prevent →** Push work-in-progress to a private branch, and avoid running `git reflog expire --expire=now --all`.

</details>

### Q84. `git stash` says `No local changes to save` even though files changed. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Changes are only untracked or ignored files (stash skips them by default), or you are in a different directory or repo from where you edited.

**Diagnose →** `git status --short --ignored` and `git rev-parse --show-toplevel`.

**Fix →** `git stash push -u` includes untracked files; `-a` also includes ignored ones. Confirm with `git stash list`.

**Prevent →** Use `-u` when you have new files.

</details>

### Q85. After renaming a file, `git log file` shows only the rename commit and the old history appears lost. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Git has no rename tracking; `git log <path>` follows only the current path name unless told otherwise. Large edits plus the rename in one commit also defeat similarity detection.

**Diagnose →**
```bash
git log --follow --oneline -- new/path.py
git log --stat -M --diff-filter=R -3
```
**Fix →** Use `git log --follow`, and `git blame -C -C` to trace copied or moved code. On GitHub, the History button follows renames in the UI.

**Prevent →** Rename in a separate commit without editing contents.

</details>

### Q86. `git bisect` points to a commit that is obviously unrelated, or the result flips between runs. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Good and bad were swapped, the test is flaky, the build fails on some commits (not the bug), or merge commits confuse the search.

**Diagnose →**
```bash
git bisect log
git bisect visualize --oneline
```
**Fix →** `git bisect reset`, then restart with a deterministic script and use exit code 125 for untestable commits: `git bisect run ./check.sh`. For "fix" hunts use `git bisect start --term-old=broken --term-new=fixed`. Try `--first-parent` on merge-heavy history.

**Prevent →** Keep every commit buildable and write the test before bisecting.

</details>

### Q87. `error: pathspec 'feature/login' did not match any file(s) known to git` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The branch does not exist locally (not fetched yet or a typo), or you ran `checkout` with a path that does not exist.

**Diagnose →**
```bash
git fetch --all --prune
git branch -a | grep login
```
**Fix →** Fetch, then `git switch feature/login` (it creates a tracking branch if exactly one remote has it), or create a new one with `git switch -c feature/login`.

**Prevent →** Use tab completion, and fetch before switching.

</details>

### Q88. `warning: unable to rmdir 'build': Directory not empty` after switching branches. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The directory still contains untracked or ignored files (build output, caches), so Git removed only the tracked ones.

**Diagnose →** `git status --ignored` and `ls -la build`.

**Fix →** The warning is harmless. Remove the leftovers if unwanted: `git clean -fdX build/` (ignored only; preview with `-n`).

**Prevent →** Keep build output outside the source tree or in clearly ignored paths.

</details>

### Q89. `fatal: --unshallow on a complete repository does not make sense`, or history stops with `(grafted)`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The first error means the repo is already complete (not shallow). A grafted commit appears at the bottom of a shallow clone, where history was cut off.

**Diagnose →**
```bash
git rev-parse --is-shallow-repository
cat .git/shallow 2>/dev/null | head
```
**Fix →** If shallow: `git fetch --unshallow` (or `--deepen=500`). If the repo is complete and tools still say shallow, check you are in the right repo.

**Prevent →** Use `fetch-depth: 0` for jobs doing `git describe`, changelogs or blame.

</details>

### Q90. `error: index file smaller than expected` or `fatal: index file corrupt` 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `.git/index` was truncated by a crash, full disk, or cloud sync tool. The index is only a cache of the staging area, so objects are normally safe.

**Diagnose →**
```bash
ls -l .git/index
git fsck --full
df -h .
```
**Fix →** Rebuild the index from HEAD: `rm .git/index && git reset`. Working tree files stay as they are, and you re-stage what you need (`git status` shows the differences).

**Prevent →** Do not keep repositories inside Dropbox or OneDrive, and monitor disk space.

</details>

### Q91. `warning: refs/heads/main does not point to a valid object!` or `fatal: your current branch appears to be broken` 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The branch ref points to a missing or corrupt commit: interrupted write, disk problem, or manual editing of `.git`.

**Diagnose →**
```bash
cat .git/refs/heads/main
git cat-file -t "$(cat .git/refs/heads/main)"
git fsck --full 2>&1 | head
git reflog show main
```
**Fix →** Point the ref to a good commit: `git update-ref refs/heads/main origin/main`, or the last good SHA from the reflog. If objects are missing, `git fetch origin` restores them from the remote.

**Prevent →** Push regularly and keep an off-machine copy of unpublished work.

</details>

### Q92. `error: unable to read sha1 file of src/app.js (3b18e5...)` or `error: inflate: data stream error` 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** A loose object or pack is corrupt (disk error, bad memory, interrupted write, sync tool).

**Diagnose →**
```bash
git fsck --full --strict
git count-objects -v
```
**Fix →** Move the corrupt object away (`mv .git/objects/3b/18e5... /tmp/`), run `git fetch origin` (pulls missing objects from remote), then `git fsck` again. For a bad pack, run `git unpack-objects` on salvageable parts or re-clone and copy over unpushed branches via `git format-patch` or `git bundle`. Also test the disk and RAM.

**Prevent →** Use reliable storage, and do not interrupt `git gc` or repack.

</details>

### Q93. After rebasing and force-pushing, the release tag `v2.1.0` is no longer on any branch. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** A tag points at a specific commit SHA. Rewriting history created new commits, so the tag still points at the old ones.

**Diagnose →**
```bash
git branch -a --contains v2.1.0       # empty output means orphaned
git log --oneline -1 v2.1.0
git cherry -v main v2.1.0^{}
```
**Fix →** Decide which is the truth. If the shipped artifact was built from the old commit, keep the tag and merge or cherry-pick the old line back, or document the divergence. If the tag was meant to follow the new history, re-tag only if no one has consumed it. Otherwise cut `v2.1.1`.

**Prevent →** Never rewrite history that carries tags, and protect `main` and release branches from force pushes.

</details>

### Q94. `git filter-repo` aborts: `Aborting: Refusing to destructively overwrite repo history since this does not look like a fresh clone.` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** filter-repo's safety check found signs of an existing working repo (remote-tracking refs, stash, reflog, multiple packs).

**Diagnose →**
```bash
git count-objects -v
git stash list
git for-each-ref | head
```
**Fix →** Work on a fresh clone made for the rewrite: `git clone --no-local /path/to/repo rewrite && cd rewrite`. Use `--force` only if you have a verified backup.

**Prevent →** Always run history rewrites on a throwaway clone and keep the original until verified.

</details>

### Q95. `error: the following file has staged content different from both the file and the HEAD:` on `git rm`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The file has changes staged and different edits in the working tree. Git refuses to delete without losing one of them.

**Diagnose →** `git diff --staged file` and `git diff file`.

**Fix →** Remove from the index but keep the file: `git rm --cached file`. To delete everything: `git rm -f file`. To stop tracking while keeping local edits, use `--cached`.

**Prevent →** Commit or restore before removing files.

</details>

### Q96. `fatal: clean.requireForce defaults to true and neither -i, -n, nor -f given; refusing to clean` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `git clean` is destructive, so Git requires an explicit flag.

**Diagnose →** `git clean -nd` previews what would be deleted.

**Fix →** After checking the list: `git clean -fd` (also directories), `-x` to include ignored files, or `-i` for interactive mode.

**Prevent →** Always run the dry run first. Deleted untracked files cannot be recovered.

</details>

### Q97. `git log` shows commits out of order and with dates in the future. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Wrong system clock or timezone on a dev machine or CI runner, imported history with odd dates, or rebases that preserve author dates while committer dates differ.

**Diagnose →**
```bash
git log --format='%h %ad | %cd | %s' --date=iso -10
date -u
```
**Fix →** Fix the clock (NTP). To correct commit dates before pushing: `git rebase --committer-date-is-author-date` or `git rebase -i --exec 'git commit --amend --no-edit --date=now'`. For tooling, sort by topology (`--topo-order`).

**Prevent →** Keep NTP enabled on laptops and CI runners.

</details>

### Q98. `fatal: a branch named 'feature/login' already exists` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The branch exists locally, maybe from an earlier attempt.

**Diagnose →** `git branch --list 'feature/*'` and `git log --oneline -3 feature/login`.

**Fix →** Switch to it (`git switch feature/login`), pick a different name, or reset it to a fresh start: `git switch -C feature/login origin/main` (this discards its old pointer; check the reflog if needed).

**Prevent →** Use ticket IDs in branch names and delete merged branches.

</details>

### Q99. `error: Cannot delete branch 'feature/x' checked out at '/home/dev/app-wt'` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The branch is the current branch in another worktree.

**Diagnose →** `git worktree list`.

**Fix →** Remove or switch that worktree first (`git worktree remove ../app-wt`), then `git branch -d feature/x`. If the directory was deleted manually, run `git worktree prune`.

**Prevent →** Remove worktrees with `git worktree remove`, not `rm -rf`.

</details>

### Q100. `fatal: Cannot do hard reset with paths.` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** You tried `git reset --hard <file>`. Hard reset works on the whole tree, not on single paths.

**Diagnose →** Re-read your command: path arguments are only allowed with mixed or soft modes.

**Fix →** To restore one file to HEAD: `git restore --source=HEAD --staged --worktree path/to/file`. To restore from another commit: `git restore --source=<sha> path`. To unstage only: `git reset path` or `git restore --staged path`.

**Prevent →** Use `git restore` for files and `git reset` for branches.

</details>

## Repository Health, Size and Performance

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q101. `git status` takes 8–15 seconds in a large monorepo. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Git is `lstat()`-ing hundreds of thousands of tracked files and walking huge untracked directories (`node_modules`, build output) on every call. Antivirus scanning on Windows makes it worse.

**Diagnose →**
```bash
GIT_TRACE2_PERF=1 git status 2>&1 | grep -E 'read_directory|refresh_index|preload'
git ls-files | wc -l
git status --untracked-files=no     # if this is fast, untracked scanning is the cost
```
**Fix →** Turn on the built-in file system monitor and caches:
```bash
git config core.fsmonitor true          # builtin daemon on macOS/Windows (Git 2.37+)
git config core.untrackedCache true
git config feature.manyFiles true       # index v4 + untracked cache
git update-index --index-version 4
```
On Linux, use Watchman with the `fsmonitor-watchman` hook, or reduce the tree with sparse checkout. `scalar register` applies all of this in one go.

**Prevent →** Make ignored build dirs actually ignored, and ship a repo bootstrap script that sets these configs.

</details>

### Q102. `error: The last gc run reported the following. Please correct the root cause and remove .git/gc.log` followed by `warning: There are too many unreachable loose objects; run 'git prune' to remove them.` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Background auto-gc ran, but thousands of unreachable loose objects are newer than `gc.pruneExpire` (2 weeks), so they cannot be pruned yet. Git writes the warning to `.git/gc.log` and skips auto-gc until you delete that file. Common after big rebases, `filter-repo` attempts or tools that create many temporary objects.

**Diagnose →**
```bash
cat .git/gc.log
git count-objects -vH       # "count" = loose objects, "size" = their size
```
**Fix →** If you do not need those unreachable objects (check `git fsck --lost-found` first if you lost work):
```bash
git prune --expire=now
git gc
rm -f .git/gc.log
```
**Prevent →** Use `git maintenance start` so housekeeping runs regularly, and avoid scripts that hash large amounts of throwaway content into the repo.

</details>

### Q103. Every few commands Git prints `Auto packing the repository in background for optimum performance.` and the machine stutters. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** More than `gc.auto` (6700) loose objects or more than `gc.autoPackLimit` (50) packs exist, so Git keeps triggering auto-gc. If gc never finishes (killed laptop lid, CI container exits), it starts again next time.

**Diagnose →** `git count-objects -v` (look at `count` and `packs`).

**Fix →** Run one full gc in the foreground and let it finish: `git gc`. On CI runners where gc is pointless, disable it: `git config --global gc.auto 0`. For long-lived dev clones, prefer scheduled maintenance: `git maintenance start`.

**Prevent →** Do not kill Git processes mid-gc; let scheduled maintenance do the work instead of auto-gc.

</details>

### Q104. A fresh clone downloads 4 GB, but the checked-out files are only 200 MB. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Large binaries (dumps, builds, videos, `.zip` artifacts) were committed at some point. Even if deleted later, every version lives in history forever.

**Diagnose →** List the biggest blobs across all history:
```bash
git rev-list --objects --all |
  git cat-file --batch-check='%(objecttype) %(objectname) %(objectsize) %(rest)' |
  awk '$1=="blob"' | sort -k3 -n -r | head -20
git count-objects -vH
```
**Fix →** Short-term, give developers `git clone --filter=blob:none` (partial clone). Long-term, rewrite history with `git filter-repo --strip-blobs-bigger-than 10M` or `--path big/dir --invert-paths`, move real binaries to Git LFS, force-push, and have everyone re-clone.

**Prevent →** Block large files with a pre-receive check or a ruleset (max file size), and a `.gitignore` for build outputs.

</details>

### Q105. `git gc` or `git clone` dies with `fatal: Out of memory, malloc failed (tried to allocate 1073741824 bytes)` or `error: pack-objects died of signal 9`. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Delta compression of huge blobs inside a memory-limited container or small VM. `pack-objects` keeps a window of objects in memory per thread, so 8 threads × big blobs = OOM killer (signal 9).

**Diagnose →**
```bash
dmesg | grep -i -E 'killed process|out of memory'
git count-objects -vH
nproc; free -h
```
**Fix →** Limit packing memory and threads, then retry:
```bash
git config pack.windowMemory 256m
git config pack.threads 2
git config pack.deltaCacheSize 128m
git config core.bigFileThreshold 50m   # store bigger files without delta attempts
git gc
```
For clones on tiny runners, use `--depth 1` or `--filter=blob:none`.

**Prevent →** Keep huge binaries out of Git (LFS), and size Git servers and runners for the repo you actually have.

</details>

### Q106. `error: unable to create temporary file: No space left on device` followed by `fatal: failed to write object`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The disk holding `.git` (or `$TMPDIR`) is full. On CI runners this is usually old workspaces, Docker images or a runaway `tmp_pack_*` file from an interrupted fetch.

**Diagnose →**
```bash
df -h . /tmp
du -sh .git
ls -lh .git/objects/pack/ | grep tmp_
```
**Fix →** Free space (clean old workspaces, `docker system prune`), delete stale `tmp_pack_*` files when no Git process is running, then run `git fsck` to confirm nothing half-written stayed behind and retry the operation.

**Prevent →** Monitor runner disk usage and clean workspaces after jobs.

</details>

### Q107. `git fsck` prints dozens of `dangling commit ...` and `dangling blob ...` lines. Is the repo broken? 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** No. Dangling objects are normal leftovers from amends, rebases, resets, dropped stashes and `git add` of content you never committed. They are unreachable but harmless and get pruned by gc after the expiry window.

**Diagnose →**
```bash
git fsck --no-dangling          # shows only real problems
git fsck --lost-found           # copies dangling commits/blobs to .git/lost-found/ for inspection
```
**Fix →** Nothing to fix. If you are looking for lost work, inspect them: `git show <sha>`. Otherwise let `git gc` clean up, or `git prune --expire=now` if you are sure.

**Prevent →** Learn the difference: "dangling" is fine, "missing", "broken link" and "corrupt" are not.

</details>

### Q108. `git fsck` reports `broken link from commit 1a2b3c4... to tree 4d5e6f7...` and `missing tree 4d5e6f7...`. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Objects are genuinely missing: disk or filesystem corruption, a repo copied while Git was writing, an interrupted `git gc` on a network filesystem, or an `objects/info/alternates` file pointing at a repo that has since been deleted or pruned.

**Diagnose →**
```bash
git fsck --full
cat .git/objects/info/alternates 2>/dev/null
git cat-file -t 4d5e6f7
```
**Fix →** Missing objects can only be restored from another copy. If the remote has them, re-fetch into a fresh clone and move over local branches, or copy objects back:
```bash
git clone --mirror git@github.com:org/repo.git /tmp/good.git
git --git-dir=/tmp/good.git cat-file -p 4d5e6f7 >/dev/null && echo "remote has it"
cp -rn /tmp/good.git/objects/pack/* .git/objects/pack/
git fsck --full
```
If an alternate was removed, run `git repack -a -d` from a working copy *before* deleting the alternate next time.

**Prevent →** Never copy or sync a live `.git` with rsync/Dropbox, avoid alternates in shared setups, and keep backups (mirror clones).

</details>

### Q109. A long-lived CI workspace `.git` is 9.8 GiB, while a fresh clone of the same repo is 900 MiB. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The cached workspace has accumulated every fetched PR ref (`refs/remotes/pull/*`), deleted branches that were never pruned, old tags, and loose objects from thousands of fetches. Auto-gc is often disabled on runners, so nothing is ever cleaned.

**Diagnose →**
```bash
git count-objects -vH
git for-each-ref | wc -l
git for-each-ref --format='%(refname)' | cut -d/ -f1-3 | sort | uniq -c | sort -rn | head
```
**Fix →**
```bash
git fetch --prune --prune-tags origin
git for-each-ref --format='%(refname)' refs/remotes/pull | xargs -r -n1 git update-ref -d
git reflog expire --expire=now --all
git gc --prune=now
```
Or simply delete the workspace and use a fresh shallow clone.

**Prevent →** Fetch only what the job needs (specific refspec, `--depth`), set `fetch.prune=true`, and schedule a weekly cleanup job on persistent runners.

</details>

### Q110. `warning: inexact rename detection was skipped due to too many files.` and `warning: you may want to set your merge.renameLimit variable to at least 5402 and retry the command.` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** A merge, cherry-pick or diff involves more added/deleted files than the rename-detection limit, so Git stopped looking for renames. A big directory move then shows as delete + add and produces bogus modify/delete conflicts.

**Diagnose →** Re-read the warning for the suggested number, and check `git diff --stat -M <base> <branch> | tail -1`.

**Fix →** Abort and retry with a higher limit:
```bash
git merge --abort
git -c merge.renameLimit=6000 merge feature
# for diffs: git -c diff.renameLimit=6000 diff -M main...feature
```
**Prevent →** Do large moves/renames in a dedicated commit with no content changes, so rename detection is exact and cheap.

</details>

### Q111. You deleted a 600 MB file with `git rm` and committed, but the repo is still huge. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `git rm` only removes the file from the next snapshot. Every previous commit still references the blob, so it stays in history, in every clone and on the server.

**Diagnose →** `git log --all --oneline -- path/to/bigfile` shows the commits that still contain it.

**Fix →** Rewrite history to remove it from all commits, then force-push:
```bash
git filter-repo --path path/to/bigfile --invert-paths   # run in a fresh clone
git push --force --all && git push --force --tags
```
All collaborators must re-clone (or hard-reset to the new history), and open PRs need rebasing.

**Prevent →** Add a pre-commit size check (`check-added-large-files` in pre-commit) and ignore dumps and build artifacts.

</details>

### Q112. After purging a leaked file with `git filter-repo` and force-pushing, the old commit is still viewable on GitHub by its SHA. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** GitHub keeps unreachable objects for a while, and other refs still point at the old history: pull request refs (`refs/pull/*/head`, which you cannot delete), forks, and cached views. Anyone who cloned before the rewrite also still has it.

**Diagnose →**
```bash
git ls-remote origin | grep -v -E 'refs/heads|refs/tags'   # refs/pull/* still exist
gh api repos/org/repo/commits/<old-sha> --jq .sha            # still resolvable?
```
**Fix →** Treat the secret as compromised and **rotate it first**. Then contact GitHub Support with the list of affected SHAs to remove cached views and PR refs and run gc on their side; ask fork owners to delete their forks.

**Prevent →** Enable secret scanning push protection so the secret never lands, and keep secrets in a vault, not in files that can be committed.

</details>

### Q113. In CI, `git describe --tags` fails with `fatal: No names found, cannot describe anything.` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The checkout is shallow (`actions/checkout` defaults to `fetch-depth: 1`) and has no tags, so there is nothing to describe from. It can also happen if the tags are lightweight and you omitted `--tags`.

**Diagnose →**
```bash
git rev-parse --is-shallow-repository
git tag --list | head
```
**Fix →** Fetch full history and tags:
```yaml
- uses: actions/checkout@v4
  with:
    fetch-depth: 0       # full history + tags
```
Or on an existing clone: `git fetch --unshallow --tags`.

**Prevent →** Any job that computes versions from tags (semantic-release, `git describe`, changelogs) must use `fetch-depth: 0`.

</details>

### Q114. In a blobless clone, `git log -p` and `git blame` are painfully slow, and offline they fail with `fatal: could not fetch 3f2a9c... from promisor remote`. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The repo was cloned with `--filter=blob:none`. Historical blobs are fetched lazily from the promisor remote the first time a command needs them, often one round-trip at a time. Without network, or with the remote gone, those commands fail.

**Diagnose →**
```bash
git config --get remote.origin.promisor
git config --get remote.origin.partialclonefilter
```
**Fix →** For history-heavy work, backfill the blobs once:
```bash
git fetch --refetch origin                 # Git 2.36+: refetch everything, dropping the filter
git config --unset remote.origin.partialclonefilter
```
Or use a treeless/blobless clone only on CI and a full clone for developers who blame and bisect a lot.

**Prevent →** Pick the clone type per use case: `blob:none` for build agents and quick checkouts, full clones for archaeology.

</details>

### Q115. `git add` prints `The following paths and/or pathspecs matched paths that exist outside of your sparse-checkout definition, so will not be updated in the index:` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** You are in a sparse checkout (cone mode) and the file you edited or created is outside the configured directories. Git protects sparse entries from accidental updates.

**Diagnose →**
```bash
git sparse-checkout list
git config core.sparseCheckout
```
**Fix →** Add the directory to the sparse definition, then add the file:
```bash
git sparse-checkout add services/payments
git add services/payments/handler.go
```
For a one-off you can force it with `git add --sparse <path>`.

**Prevent →** Document which directories each team needs and provide a `git sparse-checkout set ...` command per team in the onboarding guide.

</details>

### Q116. `git blame` attributes half the file to a single "Reformat with Prettier" commit, hiding who really wrote the code. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** A bulk formatting commit touched every line, so it becomes the last change for all of them.

**Diagnose →** `git log --oneline --format='%h %s' -- src/app.ts | grep -i format`

**Fix →** Tell blame to skip it:
```bash
echo "a1b2c3d4e5f6...  # Prettier reformat" >> .git-blame-ignore-revs
git config blame.ignoreRevsFile .git-blame-ignore-revs
git blame src/app.ts
```
GitHub's blame view honours `.git-blame-ignore-revs` at the repo root automatically.

**Prevent →** Do formatting changes in isolated commits and record their SHAs in `.git-blame-ignore-revs` immediately after merging.

</details>

### Q117. `git fetch` fails with `error: cannot lock ref 'refs/remotes/origin/main': is at 3f2a9c1 but expected 9c1d7e2`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Two fetches raced (IDE auto-fetch plus your terminal, or two CI steps in the same workspace), or two remote branches differ only by case (`Feature/x` vs `feature/x`) on a case-insensitive filesystem and keep overwriting the same file.

**Diagnose →**
```bash
git ls-remote --heads origin | awk '{print tolower($2)}' | sort | uniq -d   # case collisions
ps aux | grep -E 'git (fetch|remote-https)'
```
**Fix →** Wait for the other fetch and retry. For case collisions, delete one of the colliding branches on the server, then `git remote prune origin` and `git pack-refs --all`.

**Prevent →** Disable aggressive IDE auto-fetch on shared workspaces, enforce lowercase branch names with a ruleset or naming check, and serialize Git steps in CI.

</details>

### Q118. `git branch -r` lists hundreds of remote branches that were deleted on GitHub months ago. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Fetch does not remove remote-tracking refs for deleted branches unless pruning is on.

**Diagnose →** `git remote prune origin --dry-run`

**Fix →**
```bash
git fetch --prune
git config --global fetch.prune true
# local branches whose upstream is gone:
git branch -vv | awk '/: gone]/{print $1}' | xargs -r git branch -d
```
**Prevent →** Enable "Automatically delete head branches" in the GitHub repo settings and keep `fetch.prune=true`.

</details>

### Q119. `git count-objects -v` shows `garbage: 3` and `warning: garbage found: .git/objects/pack/tmp_pack_aB3dEf`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** A fetch, push receive or repack was interrupted (network drop, Ctrl+C, OOM kill, full disk) and left a half-written temporary pack. These files can be very large.

**Diagnose →**
```bash
git count-objects -vH
ls -lh .git/objects/pack/tmp_* .git/objects/pack/*.pack
pgrep -a git        # make sure nothing is still writing
```
**Fix →** With no Git process running, delete the temp files and verify:
```bash
rm -f .git/objects/pack/tmp_pack_* .git/objects/pack/tmp_idx_*
git fsck --connectivity-only
```
`git gc` also removes stale temp files older than the prune expiry.

**Prevent →** Fix the root cause (disk, memory, flaky network) and let scheduled maintenance run.

</details>

### Q120. `git log --graph`, `git merge-base` and `git branch --contains` take tens of seconds on a repo with 500k commits. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** No commit-graph file, so Git parses every commit object to walk history and compute generation numbers. Many loose refs and many packs add more overhead.

**Diagnose →**
```bash
ls .git/objects/info/commit-graph* .git/objects/info/commit-graphs 2>/dev/null
git count-objects -v | grep packs
GIT_TRACE2_PERF=1 git merge-base main feature 2>&1 | tail -5
```
**Fix →**
```bash
git commit-graph write --reachable --changed-paths   # changed-paths also speeds up "git log -- <path>"
git config fetch.writeCommitGraph true
git pack-refs --all
git multi-pack-index write
```
**Prevent →** `git maintenance start` keeps the commit-graph, multi-pack-index and prefetches up to date on a schedule.

</details>

### Q121. The CI "Checkout" step takes 5–6 minutes on every run. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The job clones full history and all branches/tags (`fetch-depth: 0`), pulls LFS objects or submodules it does not need, or the repo is simply large.

**Diagnose →** Check the checkout step options and the clone size: `git count-objects -vH` in a debug step.

**Fix →** Fetch only what the job needs:
```yaml
- uses: actions/checkout@v4
  with:
    fetch-depth: 1          # default; use 0 only for versioning jobs
    lfs: false
    submodules: false
    sparse-checkout: |
      services/api
      shared
```
For huge repos use `filter: blob:none`, or a persistent runner with a reference mirror.

**Prevent →** Review checkout settings per job; only release/versioning jobs need full history.

</details>

### Q122. Your large repo was fast after `git maintenance start`, but months later it is slow again and no maintenance seems to run. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The scheduler entry is gone (new laptop, OS reinstall, cron/launchd/systemd timer removed), the repo moved to a new path so `maintenance.repo` points at the old one, or the repo was never registered on this machine.

**Diagnose →**
```bash
git config --global --get-all maintenance.repo
crontab -l | grep 'git for-each-repo'                     # Linux cron
systemctl --user list-timers | grep git-maintenance       # systemd scheduler
```
**Fix →**
```bash
git config --global --unset-all maintenance.repo /old/path/app
cd /new/path/app && git maintenance register
git maintenance start
git maintenance run --task=gc       # catch up once now
```
**Prevent →** Include `git maintenance start` in the machine bootstrap script, and re-register after moving repos.

</details>

### Q123. A PR job runs `git diff --name-only origin/main...HEAD` and fails with `fatal: origin/main...HEAD: no merge base`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The CI clone is shallow, so the common ancestor of `main` and the PR branch is not in the local history. Three-dot syntax needs that merge base.

**Diagnose →**
```bash
git rev-parse --is-shallow-repository
git log --oneline -3 origin/main; git log --oneline -3 HEAD
```
**Fix →** Fetch enough history for both sides, or everything:
```bash
git fetch --no-tags --depth=200 origin main
git fetch --deepen=200            # deepen the PR side too
# or simply: actions/checkout with fetch-depth: 0
git diff --name-only origin/main...HEAD
```
**Prevent →** For "changed files" logic use `fetch-depth: 0`, or a dedicated action/API (`gh pr diff --name-only`) instead of guessing a depth.

</details>

### Q124. `git gc` fails with `fatal: gc is already running on machine 'ci-runner-3' pid 4242 (use --force if not)`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** A `.git/gc.pid` lock file exists. Either another gc really is running, or a previous gc was killed (container stopped, host rebooted) and left the file behind.

**Diagnose →**
```bash
cat .git/gc.pid
ps -p 4242 -o pid,cmd       # only meaningful on the same host
```
**Fix →** If no gc process is running, remove the stale file or force it:
```bash
rm -f .git/gc.pid
git gc            # or: git gc --force
```
**Prevent →** Avoid running gc in parallel jobs that share a workspace; disable auto-gc on runners (`gc.auto=0`) and run maintenance in one scheduled job.

</details>

### Q125. Pushing to a shared bare repo on a server fails with `error: insufficient permission for adding an object to repository database ./objects`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Several Unix users push over SSH to the same bare repo, and someone created object directories with a private umask (or ran Git as root), so others cannot write there.

**Diagnose →**
```bash
ls -ld /srv/git/app.git/objects/*/ | awk '{print $1, $3, $4}' | sort | uniq -c
git -C /srv/git/app.git config core.sharedRepository
```
**Fix →**
```bash
cd /srv/git/app.git
git config core.sharedRepository group
sudo chgrp -R developers .
sudo chmod -R g+rwX .
sudo find . -type d -exec chmod g+s {} +
```
**Prevent →** Create shared repos with `git init --bare --shared=group`, never operate on them as root, and prefer a Git server (GitHub, GitLab, Gitea) over raw shared accounts.

</details>

## Submodules, LFS and Hooks

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q126. `fatal: No url found for submodule path 'libs/common' in .gitmodules` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The index contains a gitlink (mode `160000`) for `libs/common`, but `.gitmodules` has no matching entry. Usually someone ran `git add` on a directory that was itself a Git repo (a nested clone), or deleted the `.gitmodules` section without removing the gitlink.

**Diagnose →**
```bash
git ls-files -s libs/common          # mode 160000 = gitlink
cat .gitmodules
```
**Fix →** Either register it properly or remove the gitlink:
```bash
# register as a real submodule
git rm --cached libs/common
git submodule add https://github.com/org/common.git libs/common
# or remove it and vendor the files instead
git rm --cached libs/common && rm -rf libs/common/.git && git add libs/common
```
**Prevent →** Never `git add` a directory containing its own `.git`; Git warns with `warning: adding embedded git repository` — read it.

</details>

### Q127. After cloning, the submodule directories exist but are empty. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** A plain `git clone` does not fetch submodule contents; it only creates empty directories for the gitlinks.

**Diagnose →** `git submodule status` — a leading `-` means not initialized.

**Fix →**
```bash
git submodule update --init --recursive
# next time:
git clone --recurse-submodules https://github.com/org/app.git
```
**Prevent →** Set `git config --global submodule.recurse true` so checkout, pull and switch keep submodules in sync, and document the clone command in the README.

</details>

### Q128. `git submodule update` fails: `fatal: remote error: upload-pack: not our ref 3f2a9c1...` and `Fetched in submodule path 'libs/common', but it did not contain 3f2a9c1. Direct fetching of that commit failed.` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The superproject points at a submodule commit that does not exist on the submodule's remote. Someone committed the new gitlink but never pushed the submodule commit, or the submodule branch was force-pushed and the commit is gone.

**Diagnose →**
```bash
git ls-tree HEAD libs/common                 # commit the parent expects
git -C libs/common branch -r --contains 3f2a9c1
git log -1 --format='%an %ae' -- libs/common # who bumped it
```
**Fix →** Ask the author to push the submodule commit (`git -C libs/common push origin HEAD:main`). If it is lost, point the gitlink at a commit that exists and commit that: `git -C libs/common checkout <good-sha> && git add libs/common && git commit`.

**Prevent →** Set `git config --global push.recurseSubmodules check` (or `on-demand`) so pushing the parent fails if submodule commits are not on a remote.

</details>

### Q129. `git status` keeps showing `modified: libs/common (new commits)` even though you never touched it. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The submodule's checked-out commit differs from what the superproject records. Typically you pulled the parent (which bumped the gitlink) without updating submodules, or someone ran `git pull` inside the submodule.

**Diagnose →**
```bash
git diff --submodule=log libs/common
git submodule status
```
**Fix →** To match the parent: `git submodule update --init libs/common`. If you really want the new submodule commit, commit the bump: `git add libs/common && git commit -m "Bump common to v1.4.0"`.

**Prevent →** `submodule.recurse=true` and `status.submoduleSummary=true` make this visible and automatic.

</details>

### Q130. You committed inside a submodule, ran `git submodule update`, and your commits vanished. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Submodules are checked out in detached HEAD at the commit the parent records. Your commits were made on that detached HEAD, and `submodule update` moved HEAD back to the recorded commit, leaving yours unreferenced.

**Diagnose →** `git -C libs/common reflog -10` shows your commits.

**Fix →**
```bash
cd libs/common
git branch rescue <sha-from-reflog>
git switch main && git merge rescue && git push origin main
cd .. && git add libs/common && git commit -m "Bump common"
```
**Prevent →** Always `git switch <branch>` inside a submodule before committing, or use `git submodule update --remote --merge` to work on a branch.

</details>

### Q131. `git submodule add` fails with `fatal: 'libs/common' already exists in the index`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The path is already tracked, either as normal files (vendored copy) or as an old gitlink.

**Diagnose →** `git ls-files -s libs/common | head` (mode `100644` = files, `160000` = gitlink).

**Fix →**
```bash
git rm -r --cached libs/common
rm -rf libs/common
git commit -m "Remove vendored common"
git submodule add https://github.com/org/common.git libs/common
```
**Prevent →** When converting vendored code to a submodule, remove and commit the old files in a separate step first.

</details>

### Q132. Re-adding a removed submodule fails: `A git directory for 'libs/common' is found locally with remote(s): origin https://github.com/org/common.git`. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Removing a submodule leaves its repository in `.git/modules/libs/common`. Git refuses to silently reuse it, in case it belongs to a different repo.

**Diagnose →**
```bash
ls .git/modules/libs/
git config -f .git/modules/libs/common/config remote.origin.url
```
**Fix →** If it is the same repo, reuse it: `git submodule add --force https://github.com/org/common.git libs/common`. If it is a different repo, delete the stale directory first: `rm -rf .git/modules/libs/common`, or pick another name with `--name`.

**Prevent →** Remove submodules completely:
```bash
git submodule deinit -f libs/common
git rm -f libs/common
rm -rf .git/modules/libs/common
```

</details>

### Q133. In GitHub Actions, checking out submodules fails with `fatal: could not read Username for 'https://github.com': terminal prompts disabled`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The submodule is a private repo. The default `GITHUB_TOKEN` is scoped to the current repository only, so it cannot clone other private repos. SSH URLs in `.gitmodules` fail similarly with `Permission denied (publickey)`.

**Diagnose →** Check `.gitmodules` URLs and the checkout step's `token` / `ssh-key` inputs.

**Fix →** Give checkout a credential that can read the submodule repo:
```yaml
- uses: actions/checkout@v4
  with:
    submodules: recursive
    token: ${{ secrets.SUBMODULE_READ_TOKEN }}   # fine-grained PAT or GitHub App token
    # or: ssh-key: ${{ secrets.SUBMODULE_DEPLOY_KEY }}
```
Prefer relative URLs in `.gitmodules` (`url = ../common.git`) so they follow the parent's protocol.

**Prevent →** Use a GitHub App installation token with read-only access to the needed repos instead of a personal PAT.

</details>

### Q134. `batch response: This repository is over its data quota. Account responsible for LFS bandwidth should purchase more data packs to restore access.` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The account or organization exhausted its Git LFS storage or monthly bandwidth. CI jobs that pull all LFS files on every run are the usual culprit.

**Diagnose →** Check billing (Settings → Billing → Git LFS) and which workflows use `lfs: true`. Locally: `git lfs ls-files -s | sort -k3 -h | tail`.

**Fix →** Buy data packs or raise the budget for immediate relief. Then cut usage: skip LFS in jobs that do not need it (`GIT_LFS_SKIP_SMUDGE=1`), fetch only required paths (`git lfs pull --include="assets/ui/**"`), and cache LFS objects in CI.

**Prevent →** Cache `.git/lfs` between CI runs and alert on LFS usage before hitting limits.

</details>

### Q135. After cloning, `model.bin` contains three lines starting with `version https://git-lfs.github.com/spec/v1` instead of the real file. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** You got the LFS pointer file. Git LFS is not installed, `git lfs install` was never run (no smudge filter configured), or the clone used `GIT_LFS_SKIP_SMUDGE=1`.

**Diagnose →**
```bash
git lfs version
git config --get filter.lfs.smudge
git lfs ls-files        # "-" after the oid means pointer only, "*" means downloaded
```
**Fix →**
```bash
git lfs install
git lfs pull
```
In CI, enable LFS in the checkout (`actions/checkout` with `lfs: true`).

**Prevent →** Add `git lfs install` to the onboarding script and a CI check that fails if pointer files reach the build.

</details>

### Q136. `git push` fails with `This repository is configured for Git LFS but 'git-lfs' was not found on your path.` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The repo has LFS hooks in `.git/hooks/pre-push`, but the `git-lfs` binary is not installed or not on the `PATH` of the program running Git (often a GUI client or IDE with a minimal PATH).

**Diagnose →**
```bash
which git-lfs; git lfs version
cat .git/hooks/pre-push
```
**Fix →** Install Git LFS (`brew install git-lfs`, `apt install git-lfs`, or bundled with Git for Windows), then `git lfs install`. For GUI clients, make sure they use the system Git or have the right PATH.

**Prevent →** Standardize developer tooling (devcontainer or bootstrap script) that installs `git-lfs`.

</details>

### Q137. `git status` or a clone prints `Encountered 3 file(s) that should have been pointers, but weren't:` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Files matching an LFS pattern in `.gitattributes` were committed as regular blobs, because the committer lacked LFS or the pattern was added after the files.

**Diagnose →**
```bash
git lfs fsck --pointers
git check-attr filter -- assets/logo.psd
```
**Fix →** Convert them in a new commit:
```bash
git add --renormalize .
git commit -m "Store tracked files in LFS"
```
To also clean history: `git lfs migrate import --include="*.psd" --everything` and force-push.

**Prevent →** Add `.gitattributes` patterns *before* adding files, and run `git lfs fsck --pointers` in CI.

</details>

### Q138. Checkout fails: `Smudge error: Error downloading assets/model.bin (a1b2c3...): ... Object does not exist on the server` then `fatal: assets/model.bin: smudge filter lfs failed`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** A commit references an LFS object that was never uploaded. The author pushed with `--no-verify` (skipping the LFS pre-push hook), pushed from a machine without LFS, or the repo was migrated without copying LFS objects.

**Diagnose →**
```bash
git log -1 --format='%h %an' -- assets/model.bin
git lfs ls-files --long | grep model.bin
```
**Fix →** Ask the author to upload the missing objects from their clone: `git lfs push --all origin`. To get unblocked meanwhile: `GIT_LFS_SKIP_SMUDGE=1 git checkout <branch>`, then `git lfs pull --exclude="assets/model.bin"`.

**Prevent →** Don't use `--no-verify` on LFS repos; after migrations run `git lfs fetch --all old && git lfs push --all new`.

</details>

### Q139. You ran `git lfs track "*.psd"` and committed, but clones are still huge. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `git lfs track` only affects future commits. All the old PSD versions are still regular blobs in history.

**Diagnose →**
```bash
git lfs migrate info --everything --include="*.psd"
git count-objects -vH
```
**Fix →** Rewrite history so past versions become LFS pointers (in a fresh clone, coordinated with the team):
```bash
git lfs migrate import --include="*.psd" --everything
git push --force --all && git push --force --tags
```
Everyone must re-clone; open PRs need rebasing onto the new history.

**Prevent →** Decide on LFS patterns at repo creation and block large binaries with a size limit rule.

</details>

### Q140. `git push` fails with `Unable to push locked files:` and `ERROR: Cannot update locked files.` 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The repo uses Git LFS file locking (common for binary assets like PSDs or Unity scenes). You modified a file that someone else locked, and the server rejects the push.

**Diagnose →**
```bash
git lfs locks
git lfs locks --path=art/hero.psd
```
**Fix →** Coordinate with the lock owner; once they unlock (`git lfs unlock art/hero.psd`), lock it yourself, rebase and push. An admin can break a stale lock: `git lfs unlock art/hero.psd --force`. Your own edits may need to be redone, since binary files cannot be merged.

**Prevent →** Mark lockable types in `.gitattributes` (`*.psd filter=lfs diff=lfs merge=lfs -text lockable`) so they are read-only until you `git lfs lock` them.

</details>

### Q141. Your pre-commit hook never runs, and Git prints `hint: The '.git/hooks/pre-commit' hook was ignored because it's not set as executable.` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The hook file lacks the execute bit (created in an editor, copied from Windows, or extracted from an archive).

**Diagnose →** `ls -l .git/hooks/pre-commit`

**Fix →** `chmod +x .git/hooks/pre-commit`. Also check the shebang line (`#!/usr/bin/env bash`) and that the file has no `.sample` suffix.

**Prevent →** Keep hooks in the repo (e.g. `.githooks/`) committed with mode `100755` (`git update-index --chmod=+x .githooks/pre-commit`), and install them with a script.

</details>

### Q142. Hooks committed in `.githooks/` work for you but not for teammates. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Git never runs hooks from the working tree by default and never installs them on clone (for security). You set `core.hooksPath` locally; your teammates did not.

**Diagnose →** `git config --show-origin core.hooksPath`

**Fix →** Each developer runs once:
```bash
git config core.hooksPath .githooks
```
Automate it via `npm prepare` (Husky), `make setup`, or the `pre-commit install` step.

**Prevent →** Treat client hooks as convenience only; enforce the same checks in CI and branch protection.

</details>

### Q143. `git commit` fails with `[ERROR] Your pre-commit configuration is unstaged. git add .pre-commit-config.yaml to fix this.` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The pre-commit framework requires its config to be staged, because it stashes unstaged changes while running hooks and needs a consistent config.

**Diagnose →** `git status -- .pre-commit-config.yaml`

**Fix →** `git add .pre-commit-config.yaml` and commit again. If you intentionally do not want to commit it yet, stash your config change or temporarily skip with `SKIP=hook-id git commit`.

**Prevent →** Commit pre-commit config changes in their own commit, and run `pre-commit run --all-files` after changing hook versions (`pre-commit autoupdate`).

</details>

### Q144. Commits work from the terminal, but from VS Code or a GUI client they fail with `husky - pre-commit script failed (code 127)` / `npx: command not found`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** GUI apps don't load your shell profile, so tools installed via nvm, asdf, volta or Homebrew are not on the PATH the hook sees. Exit code 127 means "command not found".

**Diagnose →** Add `echo $PATH > /tmp/hook-path.txt` at the top of the hook and commit from the GUI.

**Fix →** For Husky v9, put PATH setup in `~/.config/husky/init.sh` (sourced before each hook):
```bash
# ~/.config/husky/init.sh
export NVM_DIR="$HOME/.nvm"
. "$NVM_DIR/nvm.sh"
```
Or launch the editor from a terminal (`code .`) so it inherits your PATH.

**Prevent →** Keep hooks dependency-light and document the GUI PATH fix in the contributing guide.

</details>

### Q145. Lint and secret-scan hooks exist, yet broken code and a leaked key landed on `main`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Client-side hooks are optional: anyone can skip them with `git commit --no-verify`, never install them, or edit via the GitHub web UI.

**Diagnose →** Find the commits and check whether they went through a PR with checks: `gh pr list --state merged --search <sha>`.

**Fix →** Move enforcement server-side: required status checks (lint, tests, secret scan) in branch protection or rulesets, "Require a pull request before merging", and GitHub secret scanning push protection. Rotate the leaked key immediately.

**Prevent →** Treat local hooks as a fast feedback loop and CI plus rulesets as the gate.

</details>

### Q146. A hook fails with `/usr/bin/env: 'bash\r': No such file or directory`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The hook script has Windows CRLF line endings, so the shebang interpreter becomes `bash\r`. Usually caused by `core.autocrlf=true` on Windows checking out scripts with CRLF and then a Linux/WSL/container shell running them.

**Diagnose →**
```bash
file .githooks/pre-commit          # "with CRLF line terminators"
git ls-files --eol .githooks/
```
**Fix →** Force LF for scripts and renormalize:
```bash
printf '*.sh text eol=lf\n.githooks/* text eol=lf\n' >> .gitattributes
git add --renormalize .
git commit -m "Force LF for scripts"
```
Locally fix the file with `dos2unix .githooks/pre-commit`.

**Prevent →** Commit a `.gitattributes` with `eol=lf` for all shell scripts from day one.

</details>

### Q147. Pushing to a self-hosted Git server fails with `! [remote rejected] main -> main (pre-receive hook declined)`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** A server-side hook rejected the push: commit message policy, file size limit, secret detection, forbidden paths, unsigned commits, or a direct push to a protected branch. The reason is printed in the `remote:` lines just above.

**Diagnose →** Scroll up for `remote:` output; then inspect the offending commits:
```bash
git log --format='%h %s' origin/main..HEAD
git diff --stat origin/main..HEAD
```
**Fix →** Fix the commits to satisfy the policy (`git commit --amend`, `git rebase -i` to reword or drop), then push again. If the hook is buggy, the server admin can check its logs (GitLab: `gitlab-shell` logs; Gitea/Bitbucket: hook output in admin UI).

**Prevent →** Mirror server-side rules as local hooks and CI checks so developers hit them earlier.

</details>

### Q148. The `commit-msg` hook rejected your long, carefully written commit message, and you don't want to retype it. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The message failed a convention check (Conventional Commits, ticket ID). The message you typed is saved in `.git/COMMIT_EDITMSG`.

**Diagnose →** `cat .git/COMMIT_EDITMSG`

**Fix →** Edit it and reuse it:
```bash
vim .git/COMMIT_EDITMSG          # fix the subject, e.g. "feat(api): add retry"
git commit -F .git/COMMIT_EDITMSG
# or open it in the editor: git commit -e -F .git/COMMIT_EDITMSG
```
**Prevent →** Use a commit template (`git config commit.template .gitmessage`) that reminds you of the format.

</details>

### Q149. The pre-commit hook takes 90 seconds on every commit, so people start using `--no-verify`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The hook lints, type-checks or tests the entire repo instead of the staged files, or downloads tools on every run.

**Diagnose →** `time .git/hooks/pre-commit`, then read what it runs.

**Fix →** Run checks only on staged files and move heavy work to CI:
```bash
# get staged files for a custom hook
git diff --cached --name-only --diff-filter=ACMR -- '*.py' | xargs -r ruff check
```
With the pre-commit framework, hooks already receive only staged files; with JS repos use `lint-staged`. Keep full test suites in `pre-push` or CI.

**Prevent →** Set a budget (a few seconds) for pre-commit hooks and measure it when adding new ones.

</details>

### Q150. `git submodule update --remote` fails with `fatal: Unable to find current origin/master revision in submodule path 'libs/common'`. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `--remote` tracks the branch in `.gitmodules` (`submodule.<name>.branch`), and without one older behavior assumes `master`. The submodule repo renamed its default branch to `main`, so `origin/master` no longer exists.

**Diagnose →**
```bash
git config -f .gitmodules --get-regexp 'submodule\..*\.branch'
git -C libs/common ls-remote --symref origin HEAD
```
**Fix →** Point it at the right branch (or `.` to follow the parent's branch name):
```bash
git submodule set-branch --branch main libs/common
git submodule sync --recursive
git submodule update --remote libs/common
git add .gitmodules libs/common && git commit -m "Track main in common"
```
**Prevent →** Always set `branch = main` explicitly in `.gitmodules` for submodules that use `--remote`.

</details>

## GitHub, Actions and Pull Request Issues

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q151. A workflow step that pushes a version bump fails with `remote: Permission to org/repo.git denied to github-actions[bot].` and `The requested URL returned error: 403`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The `GITHUB_TOKEN` is read-only. Since 2023 new repos and orgs default workflow permissions to "Read repository contents", and the workflow does not request `contents: write`. A branch ruleset may also block the bot from pushing to `main`.

**Diagnose →** Check Settings → Actions → General → Workflow permissions, the `permissions:` block in the workflow, and the branch's rulesets.

**Fix →**
```yaml
permissions:
  contents: write
jobs:
  release:
    steps:
      - uses: actions/checkout@v4
      - run: |
          git config user.name "github-actions[bot]"
          git config user.email "41898282+github-actions[bot]@users.noreply.github.com"
          git commit -am "chore: bump version" && git push
```
If `main` is protected, add the GitHub App used for releases to the ruleset bypass list rather than opening protection for everyone.

**Prevent →** Declare least-privilege `permissions:` per job explicitly.

</details>

### Q152. A workflow pushes a commit or tag, but the workflow that should run on that push or tag never starts. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** By design, events created with the repository `GITHUB_TOKEN` do not trigger new workflow runs (except `workflow_dispatch` and `repository_dispatch`). This prevents recursive loops.

**Diagnose →** Check which token the push used, and the Actions tab: no run exists for that push event.

**Fix →** Push with a GitHub App installation token or a fine-grained PAT:
```yaml
- uses: actions/create-github-app-token@v1
  id: app
  with:
    app-id: ${{ vars.RELEASE_APP_ID }}
    private-key: ${{ secrets.RELEASE_APP_KEY }}
- uses: actions/checkout@v4
  with:
    token: ${{ steps.app.outputs.token }}
```
Or chain explicitly with `workflow_run` or `gh workflow run`.

**Prevent →** Prefer a GitHub App token for release automation; avoid long-lived personal PATs.

</details>

### Q153. A container job fails on any Git command with `fatal: detected dubious ownership in repository at '/__w/app/app'`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Since Git 2.35.2 (CVE-2022-24765), Git refuses to work in a repo owned by a different user. The workspace was checked out by the runner user, but the container runs as root (or vice versa). Same issue on shared servers and Docker volume mounts.

**Diagnose →**
```bash
id -u; stat -c '%u %n' /__w/app/app
git --version
```
**Fix →** Mark that path as safe, inside the same user context that runs Git:
```bash
git config --global --add safe.directory /__w/app/app
# or, in a throwaway container only:
git config --global --add safe.directory '*'
```
Recent `actions/checkout` sets this for you; the problem appears when later steps run as another user or in another container.

**Prevent →** Run containers with the same UID as the workspace owner, and avoid the `'*'` wildcard on shared hosts.

</details>

### Q154. Your PR suddenly shows 340 changed files and dozens of commits you did not write. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The PR's base is wrong (e.g. targeting `main` while the branch was cut from `release/2.x`), you merged another branch into yours, or the base branch was rewritten/force-pushed after you branched.

**Diagnose →**
```bash
git fetch origin
git log --oneline origin/main..HEAD | wc -l
git merge-base --fork-point origin/main HEAD
gh pr view --json baseRefName,headRefName
```
**Fix →** Change the base branch in the PR UI if it is wrong. Otherwise replay only your commits onto the correct base:
```bash
git rebase --onto origin/main <last-commit-before-yours> feature
git push --force-with-lease
```
**Prevent →** Branch from an up-to-date `origin/main`, and do not merge unrelated branches into feature branches.

</details>

### Q155. The PR is stuck on `Expected — Waiting for status to be reported` for a required check called `build`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The required check name in branch protection no longer matches any job (job renamed, matrix changed `build` to `build (20.x)`), or the workflow did not run at all because of `paths:` / `branches:` filters, so the check is never reported.

**Diagnose →** Compare the required check names (Settings → Branches/Rulesets) with the check names in the PR's Checks tab. Look at the workflow's `on:` filters.

**Fix →** Update the required check to the exact job name, or make the workflow always run and skip internally:
```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: dorny/paths-filter@v3
        id: f
        with: { filters: "src: ['src/**']" }
      - if: steps.f.outputs.src == 'true'
        run: make build
```
A common pattern is one always-running "ci-ok" job that `needs:` all others, and only that job is required.

**Prevent →** Treat job names as an API; change required checks in the same PR that renames jobs.

</details>

### Q156. The PR was approved, but merging shows "Review required" again after you pushed a small fix. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Branch protection has "Dismiss stale pull request approvals when new commits are pushed" or "Require approval of the most recent reviewable push" enabled. Your new commit invalidated the approval.

**Diagnose →** `gh pr view --json reviewDecision,latestReviews` and the protection settings.

**Fix →** Ask for a re-review (`gh pr edit --add-reviewer alice`). Batch your fixes before requesting review to avoid repeated cycles.

**Prevent →** This setting is deliberate for security; keep it on for `main`, and agree on review etiquette so re-approvals are quick.

</details>

### Q157. CODEOWNERS is committed, but GitHub does not request reviews from the listed team. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The file is not in `.github/`, the root, or `docs/` on the **base** branch; the team or user lacks write access to the repo; the syntax is invalid; or a later, more specific line overrides the pattern (last match wins).

**Diagnose →** Open the CODEOWNERS file on GitHub; invalid lines are highlighted with errors such as "Unknown owner". Check the team's repo permission.
```bash
gh api repos/org/repo/codeowners/errors
```
**Fix →** Move the file to `.github/CODEOWNERS` on the base branch, give the team write access, fix pattern order:
```text
*                @org/platform
/services/pay/   @org/payments     # later lines override earlier ones
```
**Prevent →** Validate CODEOWNERS in CI using the `codeowners/errors` API and enable "Require review from Code Owners".

</details>

### Q158. `git push` fails with `remote: error: GH013: Repository rule violations found for refs/heads/main.` and `- Changes must be made through a pull request.` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** A repository or organization ruleset blocks direct pushes to `main` (or requires signed commits, linear history, specific commit message patterns). Other bullet lines in the output show which rule failed.

**Diagnose →** Read every `remote:` bullet, then check the active rules:
```bash
gh api repos/org/repo/rules/branches/main --jq '.[].type'
```
**Fix →** Push to a branch and open a PR:
```bash
git switch -c fix/config
git push -u origin fix/config
gh pr create --fill
```
Then reset local `main` to `origin/main` so it doesn't drift.

**Prevent →** Never commit on local `main`; start every change with `git switch -c`.

</details>

### Q159. Push rejected: `remote: - GITHUB PUSH PROTECTION` and `Push cannot contain secrets` pointing at a commit from three commits ago. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Secret scanning push protection found a credential pattern (AWS key, GitHub token, Slack webhook) in one of the commits you are pushing — not necessarily the latest one. Removing it in a new commit does not help; it is still in history.

**Diagnose →** The output lists the secret type, commit SHA and path. Confirm with `git show <sha> -- <path>`.

**Fix →** Remove it from the commit itself, then push:
```bash
git rebase -i <sha>^          # mark the commit as "edit"
# remove the secret from the file, use an env var instead
git add config/app.yml && git commit --amend --no-edit
git rebase --continue
git push
```
If the secret was real, rotate it. Only use the "allow secret" link for verified false positives or test credentials.

**Prevent →** Run `gitleaks` or `trufflehog` in pre-commit, and load secrets from a vault or environment.

</details>

### Q160. A ruleset requires signed commits; your commits show "Unverified" on GitHub and the push is rejected with `Commits must have verified signatures.` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The commits are unsigned, signed with a key not uploaded to GitHub as a **signing** key, or signed with an email that is not verified on your GitHub account.

**Diagnose →**
```bash
git log --show-signature -3
git config --get user.email
git config --get gpg.format; git config --get user.signingkey
```
**Fix →** Configure signing (SSH signing is simplest), upload the public key under Settings → SSH and GPG keys as "Signing key", then re-sign your commits:
```bash
git config --global gpg.format ssh
git config --global user.signingkey ~/.ssh/id_ed25519.pub
git config --global commit.gpgsign true
git rebase --exec 'git commit --amend --no-edit -S' origin/main
git push --force-with-lease
```
**Prevent →** Enable `commit.gpgsign` globally and make sure `user.email` matches a verified address.

</details>

### Q161. `git commit` fails with `error: gpg failed to sign the data` and `fatal: failed to write commit object`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `commit.gpgsign=true` but GPG cannot prompt for the passphrase (no TTY, pinentry missing in a GUI or SSH session), the key expired, or `user.signingkey` points at a key that does not exist on this machine.

**Diagnose →**
```bash
echo test | gpg --clearsign            # shows the real gpg error
gpg --list-secret-keys --keyid-format=long
git config --get user.signingkey
```
**Fix →** Set the TTY and restart the agent:
```bash
export GPG_TTY=$(tty)        # add to ~/.bashrc or ~/.zshrc
gpgconf --kill gpg-agent
```
On macOS install `pinentry-mac` and set it in `~/.gnupg/gpg-agent.conf`. For an expired key, extend it (`gpg --edit-key <id>` → `expire`) and re-upload the public key.

**Prevent →** Consider SSH signing (`gpg.format=ssh`), which reuses your SSH agent and avoids pinentry issues.

</details>

### Q162. `git log --show-signature` prints `error: gpg.ssh.allowedSignersFile needs to be configured and exist for ssh signature verification`. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** You use SSH commit signing. Signing works, but local **verification** needs an allowed-signers file mapping emails to public keys; GitHub has its own list, your laptop does not.

**Diagnose →** `git config --get gpg.ssh.allowedSignersFile`

**Fix →**
```bash
mkdir -p ~/.config/git
echo "you@example.com $(cat ~/.ssh/id_ed25519.pub)" >> ~/.config/git/allowed_signers
git config --global gpg.ssh.allowedSignersFile ~/.config/git/allowed_signers
git log --show-signature -1     # Good "git" signature for you@example.com
```
For team-wide verification, commit an `allowed_signers` file of maintainer keys and point the config at it.

**Prevent →** Include the allowed-signers setup in the same onboarding step that enables SSH signing.

</details>

### Q163. In CI, `gh pr comment` fails with `HTTP 401: Bad credentials`, or locally `gh` says `You are not logged into any GitHub hosts. To log in, run: gh auth login`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `gh` found no token: in Actions it needs `GH_TOKEN` (or `GITHUB_TOKEN`) in the step's environment. A 401 means the token is set but expired, revoked or malformed.

**Diagnose →**
```bash
gh auth status
env | grep -E '^(GH|GITHUB)_TOKEN=' | cut -c1-12
```
**Fix →**
```yaml
- run: gh pr comment "$PR" --body "Deployed to staging"
  env:
    GH_TOKEN: ${{ github.token }}
    PR: ${{ github.event.pull_request.number }}
permissions:
  pull-requests: write
```
Locally: `gh auth login` (or `gh auth refresh -s repo,workflow` to add scopes).

**Prevent →** Always pass `GH_TOKEN` explicitly in workflow steps that call `gh`.

</details>

### Q164. PRs added to the merge queue are removed with "failed to merge" because required checks never ran. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The merge queue creates temporary `gh-readonly-queue/main/pr-...` branches and expects required checks to report on them via the `merge_group` event. Workflows triggered only by `pull_request` never run there, so the queue times out.

**Diagnose →** Check the workflow's `on:` block and the queue entry's checks; there will be no run for the `merge_group` event.

**Fix →**
```yaml
on:
  pull_request:
  merge_group:
    types: [checks_requested]
```
Make sure the same job names are produced for both events, since required checks match by name.

**Prevent →** Add `merge_group` to every required workflow before enabling the merge queue on a branch.

</details>

### Q165. After your PR was squash-merged, you continued on the same branch; the next PR shows conflicts with your own already-merged changes. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Squash merge creates a brand-new commit on `main`; your original commits are not ancestors of `main`. Git sees them as different changes touching the same lines.

**Diagnose →**
```bash
git log --oneline origin/main..feature     # still lists the commits that were squashed
git cherry -v origin/main feature          # "+" = not found upstream by patch-id
```
**Fix →** Rebase only the new commits onto `main`:
```bash
git fetch origin
git rebase --onto origin/main <last-squashed-commit> feature
git push --force-with-lease
```
**Prevent →** Start a fresh branch from `origin/main` after each squash merge, and enable auto-delete of merged head branches.

</details>

### Q166. A PR from a fork fails with `Error: Input required and not supplied: token` or empty secrets, while PRs from branches pass. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** For `pull_request` events from forks, GitHub does not pass repository secrets and gives a read-only `GITHUB_TOKEN`. This protects secrets from untrusted code in the fork.

**Diagnose →** `github.event.pull_request.head.repo.fork == true` and the secret expands to empty.

**Fix →** Make fork PR jobs work without secrets (skip deploy/publish steps with `if: github.event.pull_request.head.repo.full_name == github.repository`). If you need privileged actions like labeling or commenting, use `pull_request_target` or a `workflow_run` follow-up **without** checking out and executing the fork's code.

**Prevent →** Never combine `pull_request_target` with `actions/checkout` of `github.event.pull_request.head.sha` and running its build — that's the classic "pwn request".

</details>

### Q167. An action fails with `HttpError: Resource not accessible by integration` (403) when commenting on a PR or adding a label. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The `GITHUB_TOKEN` lacks the needed permission (`pull-requests: write`, `issues: write`), the workflow sets a restrictive `permissions:` block, or the run comes from a fork or Dependabot with a read-only token.

**Diagnose →** In the run logs, expand "Set up job" → "GITHUB_TOKEN Permissions" to see what was granted.

**Fix →**
```yaml
permissions:
  contents: read
  pull-requests: write
  issues: write
```
For fork PRs, move the commenting to a `workflow_run` workflow triggered after the CI run.

**Prevent →** Declare permissions per job and check the "GITHUB_TOKEN Permissions" log section when debugging 403s.

</details>

### Q168. Dependabot PRs fail CI because secrets like `NPM_TOKEN` are empty, while human PRs pass. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Workflows triggered by Dependabot run with Dependabot secrets, not Actions secrets, and get a read-only `GITHUB_TOKEN`.

**Diagnose →** `github.actor == 'dependabot[bot]'` in the failing runs.

**Fix →** Add the needed values under Settings → Secrets and variables → **Dependabot** (same names), and for private registries configure them in `.github/dependabot.yml` under `registries:`. Grant extra token permissions explicitly in the workflow if needed.

**Prevent →** Keep a list of secrets required by CI and mirror them in the Dependabot secret store when you add new ones.

</details>

### Q169. In a `pull_request` workflow, `git log -1` shows `Merge 4f1c2d3 into 9a8b7c6` and `git rev-parse HEAD` differs from the PR's head SHA. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** For `pull_request` events, `actions/checkout` checks out `refs/pull/<n>/merge`, a temporary merge of the PR head into the base. That is intended (it tests the merge result), but it surprises tools that tag images or report status by `HEAD`.

**Diagnose →**
```bash
echo "$GITHUB_SHA"                                     # merge commit SHA
echo "${{ github.event.pull_request.head.sha }}"       # real PR head
```
**Fix →** If you need the PR head itself:
```yaml
- uses: actions/checkout@v4
  with:
    ref: ${{ github.event.pull_request.head.sha }}
```
Use `github.event.pull_request.head.sha` for image tags and commit statuses.

**Prevent →** Decide deliberately per workflow: merge ref for testing, head SHA for reporting.

</details>

### Q170. A "changed files" step runs `git diff --name-only ${{ github.event.before }} HEAD` and fails with `fatal: bad object 0000000000000000000000000000000000000000`. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** On the first push of a new branch, `github.event.before` is all zeros. After a force-push it is a SHA that no longer exists in the remote, and with a shallow clone the old commit is never present locally either (`fatal: bad object <sha>`).

**Diagnose →** `echo "${{ github.event.before }}"` and `git rev-parse --is-shallow-repository`.

**Fix →** Handle these cases explicitly:
```bash
BEFORE="${{ github.event.before }}"
if [ "$BEFORE" = "0000000000000000000000000000000000000000" ] || ! git cat-file -e "$BEFORE^{commit}" 2>/dev/null; then
  git fetch --no-tags origin main
  BEFORE=$(git merge-base origin/main HEAD)
fi
git diff --name-only "$BEFORE" HEAD
```
Use `fetch-depth: 0` (or enough depth) so the base commit exists.

**Prevent →** Prefer comparing against the merge base with the default branch over `event.before`.

</details>

### Q171. Every PR needs "Update branch" before merging; with 20 open PRs, the team spends the day rebasing and re-running CI. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Branch protection has "Require branches to be up to date before merging". Each merge to `main` makes every other PR out of date.

**Diagnose →** The PR shows "This branch is out-of-date with the base branch".

**Fix →** Enable a merge queue: it tests each PR on top of the latest `main` plus queued PRs and merges in order, without manual updates. Alternatively, drop the up-to-date requirement for low-risk repos and rely on post-merge CI.

**Prevent →** For busy repos, plan for a merge queue early instead of the strict up-to-date rule.

</details>

### Q172. Opening a PR shows "There isn't anything to compare. main and import are entirely different commit histories." and locally `git merge` says `fatal: refusing to merge unrelated histories`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The two branches share no common ancestor: someone ran `git init` and pushed a new history to the same repo, created the GitHub repo with a README and then pushed an existing local repo, or imported code with a tool that started fresh history.

**Diagnose →** `git merge-base main import || echo "no common ancestor"`

**Fix →** If you really want to combine them:
```bash
git switch import
git merge origin/main --allow-unrelated-histories
# resolve conflicts (often README/.gitignore), commit, push, open the PR
```
If the new history should simply replace the old one, rebase the import commits onto `main` instead.

**Prevent →** Create empty GitHub repos (no README/licence) when pushing existing code, or clone first and then add files.

</details>

### Q173. `gh release create v1.2.0` fails with `tag v1.2.0 exists locally but has not been pushed to org/repo, please push it before continuing or specify the --target flag to create a new tag`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** You tagged locally but did not push the tag, so GitHub cannot attach a release to it.

**Diagnose →**
```bash
git ls-remote --tags origin v1.2.0
git show --no-patch v1.2.0
```
**Fix →**
```bash
git push origin v1.2.0
gh release create v1.2.0 --generate-notes
```
**Prevent →** Release from CI on tag push (`on: push: tags: ['v*']`), or use `git push --follow-tags` for annotated tags.

</details>

### Q174. After your PR was merged and the branch auto-deleted, `git pull` says `Your configuration specifies to merge with the ref 'refs/heads/feature/login' from the remote, but no such ref was fetched.` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Your local branch still tracks `origin/feature/login`, which was deleted on GitHub after the merge.

**Diagnose →** `git branch -vv` shows `[origin/feature/login: gone]`.

**Fix →**
```bash
git switch main
git pull
git branch -d feature/login       # -D if it was squash-merged
git fetch --prune
```
**Prevent →** Switch back to `main` after merging and enable `fetch.prune=true`.

</details>

### Q175. Two concurrent workflow runs both try to push a changelog commit; one fails with `! [rejected] main -> main (fetch first)`. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Runs triggered close together each checked out an older `main`, committed, and raced to push. The loser's commit is based on a stale parent.

**Diagnose →** Compare run start times in the Actions tab and `git log origin/main -3` after the failure.

**Fix →** Serialize the jobs and retry with rebase:
```yaml
concurrency:
  group: release-main
  cancel-in-progress: false
steps:
  - run: |
      for i in 1 2 3; do
        git pull --rebase origin main && git push origin HEAD:main && break
        sleep $((i * 5))
      done
```
**Prevent →** Avoid bots committing to `main` for every merge; generate changelogs at release time, or have a single release workflow guarded by a `concurrency` group.

</details>

## Config, Line Endings, Files and Platform Quirks

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q176. `warning: in the working copy of 'app.sh', LF will be replaced by CRLF the next time Git touches it` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** You are on Windows with `core.autocrlf=true` (the Git for Windows default), and the file has LF endings. Git warns it will convert it to CRLF on checkout. Harmless for most source files, harmful for shell scripts and files consumed on Linux.

**Diagnose →**
```bash
git config --show-origin core.autocrlf
git ls-files --eol app.sh        # i/lf w/lf attr/...
```
**Fix →** Make line endings explicit per file type in the repo, so every OS behaves the same:
```gitattributes
* text=auto
*.sh text eol=lf
*.bat text eol=crlf
*.png binary
```
Then `git add --renormalize . && git commit -m "Normalize line endings"`.

**Prevent →** Commit `.gitattributes` early; do not rely on each developer's `core.autocrlf`.

</details>

### Q177. `git diff` shows `old mode 100755` / `new mode 100644` for hundreds of files with no content change. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The working tree sits on a filesystem that does not store Unix permissions correctly (Windows drive under WSL `/mnt/c`, FAT/exFAT USB drive, some network shares or container mounts), or someone ran `chmod -R`.

**Diagnose →** `git diff --summary | grep 'mode change' | head` and `git config core.fileMode`.

**Fix →** Tell Git to ignore the filesystem's mode bits for this repo: `git config core.fileMode false`. If the mode change was a real mistake (e.g. `chmod -R`), undo it with `git restore .` when no content changes are pending.

**Prevent →** Keep repos on the native Linux filesystem in WSL (`~/src`, not `/mnt/c`), and set the execute bit explicitly in Git with `git update-index --chmod=+x`.

</details>

### Q178. Cloning on Windows fails with `error: unable to create file src/very/deep/.../component.test.ts: Filename too long`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Windows' legacy 260-character `MAX_PATH` limit. Deep `node_modules`-style paths plus a long clone directory exceed it.

**Diagnose →** `git config --show-origin core.longpaths`

**Fix →**
```powershell
git config --global core.longpaths true
# also allow long paths in Windows (admin PowerShell):
New-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Control\FileSystem" -Name LongPathsEnabled -Value 1 -PropertyType DWORD -Force
```
Then re-run the checkout: `git restore --source=HEAD :/` or clone again. Cloning into a short path (`C:\src\app`) also helps.

**Prevent →** Keep repository paths reasonably shallow and include `core.longpaths` in the Windows dev setup.

</details>

### Q179. You renamed `header.tsx` to `Header.tsx` on macOS. Locally everything builds, but Linux CI fails with `Module not found: Can't resolve './components/Header'`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** macOS and Windows filesystems are case-insensitive, and Git sets `core.ignorecase=true` there. Renaming only the case in Finder or the IDE is invisible to Git, so the repo still contains `header.tsx`.

**Diagnose →**
```bash
git ls-files | grep -i 'components/header'
git config core.ignorecase
```
**Fix →** Rename through Git so the index changes:
```bash
git mv -f src/components/header.tsx src/components/Header.tsx
git commit -m "Rename header.tsx to Header.tsx"
```
**Prevent →** Use `git mv` for case-only renames, enable case-sensitive import checks in linters/TS (`forceConsistentCasingInFileNames`), and run CI on Linux.

</details>

### Q180. Checkout on Windows fails with `error: invalid path 'logs/2024-01-01T10:00:00.log'` (or `'docs/aux.md'`) and `fatal: unable to checkout working tree`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The repo contains paths that are illegal on NTFS: characters such as `:` `*` `?` `"` `<` `>` `|`, trailing dots or spaces, or reserved device names (`CON`, `PRN`, `AUX`, `NUL`, `COM1`). Git for Windows refuses them (`core.protectNTFS`) for safety.

**Diagnose →** `git ls-tree -r --name-only HEAD | grep -E '[:*?"<>|]|(^|/)(con|prn|aux|nul|com[0-9]|lpt[0-9])(\.|/|$)'`

**Fix →** Rename the files from Linux, macOS or WSL and push:
```bash
git mv 'logs/2024-01-01T10:00:00.log' 'logs/2024-01-01T10-00-00.log'
git commit -m "Use Windows-safe file names" && git push
```
To work in the meantime, exclude the path with sparse checkout. Don't disable `core.protectNTFS` — it guards against real path-traversal CVEs.

**Prevent →** Add a CI check that rejects Windows-invalid paths if any contributor uses Windows.

</details>

### Q181. Clone prints `warning: the following paths have collided (e.g. case-sensitive paths on a case-insensitive filesystem) and only one from the same colliding group is in the working tree:` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The repo contains two paths that differ only by case (e.g. `src/Utils.js` and `src/utils.js`), created on Linux. On macOS/Windows only one can exist, so `git status` constantly shows one of them as modified.

**Diagnose →** `git ls-files | sort -f | uniq -di`

**Fix →** Decide which one is correct and remove the other in the repo (from any machine, using the index only):
```bash
git rm --cached src/Utils.js
git commit -m "Remove duplicate Utils.js (case collision)"
git push
```
Merge the content first if both files differ.

**Prevent →** Add a CI check for case-insensitive duplicates (the `check-case-conflict` hook in pre-commit does this).

</details>

### Q182. In your `~/work` repos commits still use your personal email, even though you configured `includeIf` for work. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The `gitdir:` pattern does not match: missing trailing slash (`gitdir:~/work` matches only that exact path), wrong case on Windows/macOS (use `gitdir/i:`), symlinked directories, the included file path is wrong, or the `includeIf` block comes *before* the personal `[user]` block, so the later value wins.

**Diagnose →**
```bash
cd ~/work/app
git config --show-origin --show-scope user.email
git config --list --show-origin | grep -i includeif
```
**Fix →** In `~/.gitconfig`, put defaults first and the conditional include last:
```ini
[user]
    name = Aditi Dev
    email = me@personal.dev
[includeIf "gitdir:~/work/"]
    path = ~/.gitconfig-work
```
`~/.gitconfig-work` contains `[user] email = aditi@company.com`. Fix already-made commits with `git rebase -r --exec 'git commit --amend --no-edit --reset-author' origin/main`.

**Prevent →** Verify with `git config --show-origin user.email` in a new work repo before the first commit.

</details>

### Q183. `.gitignore` has `logs/` and `!logs/keep.txt`, but `logs/keep.txt` is still ignored. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Git cannot re-include a file if its parent directory is excluded; once `logs/` is ignored, Git never looks inside it, so the negation has no effect.

**Diagnose →** `git check-ignore -v logs/keep.txt` shows which rule wins.

**Fix →** Ignore the contents, not the directory, then negate:
```gitignore
logs/*
!logs/keep.txt
```
For nested paths, each parent level must be un-ignored (`build/*`, `!build/config/`, `build/config/*`, `!build/config/app.yml`).

**Prevent →** Test ignore rules with `git check-ignore -v` and `git status --ignored`.

</details>

### Q184. `git config --global ...` fails with `error: could not lock config file /home/dev/.gitconfig: Permission denied` (or `: File exists`). 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** "Permission denied": the file (or `$HOME`) is owned by root, often after `sudo git config --global`. "File exists": a stale `~/.gitconfig.lock` from a crashed Git or tool.

**Diagnose →**
```bash
ls -l ~/.gitconfig ~/.gitconfig.lock 2>/dev/null
echo "$HOME"
```
**Fix →**
```bash
sudo chown "$USER":"$(id -gn)" ~/.gitconfig
rm -f ~/.gitconfig.lock          # only if no git process is running
```
**Prevent →** Never run Git with `sudo` in your own repos; fix file permissions instead.

</details>

### Q185. On Windows, a symlink in the repo (`config/current -> envs/prod.yml`) is checked out as a small text file containing `envs/prod.yml`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `core.symlinks=false`, the default on Windows when the user lacks permission to create symlinks. Git writes the link target as plain text instead.

**Diagnose →**
```bash
git config --show-origin core.symlinks
git ls-files -s config/current     # mode 120000 = symlink in the repo
```
**Fix →** Enable Windows Developer Mode (or run with the symlink privilege), then:
```bash
git config core.symlinks true
git rm --cached -r config/current && git restore config/current   # recreate the link
```
Or clone again with `git clone -c core.symlinks=true ...`.

**Prevent →** Avoid symlinks in cross-platform repos; use config files or build-time copies instead.

</details>

### Q186. `git apply fix.patch` fails with `error: patch failed: src/app.js:12` and `error: src/app.js: patch does not apply`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The file changed since the patch was created (context lines no longer match), whitespace or line endings differ, or the path prefix level is wrong.

**Diagnose →**
```bash
git apply --check -v fix.patch
head -20 fix.patch       # check a/ b/ prefixes and the base index line
```
**Fix →** Try, in order:
```bash
git apply --3way fix.patch              # uses blob IDs in the patch to do a real merge
git apply --ignore-whitespace fix.patch
git apply -p0 fix.patch                 # when paths have no a/ b/ prefix
git apply --reject fix.patch            # applies what fits, writes *.rej for the rest
```
**Prevent →** Share changes as branches or `git format-patch` output (with full index lines) rather than plain `diff` output.

</details>

### Q187. `git am` stops with `Patch failed at 0001 Fix retry logic` and `hint: Use 'git am --show-current-patch=diff' to see the failed patch`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The mailbox patch does not apply cleanly to your current branch, usually because the base moved on.

**Diagnose →**
```bash
git am --show-current-patch=diff
git status
```
**Fix →** Retry with a three-way merge, resolve, and continue:
```bash
git am --abort
git am -3 patches/*.patch
# on conflict: edit files, then
git add src/retry.go
git am --continue        # or: git am --skip / git am --abort
```
**Prevent →** Ask contributors to rebase on the current base before running `git format-patch`.

</details>

### Q188. A deploy script run with `sudo -u deploy git pull` prints `warning: unable to access '/root/.config/git/attributes': Permission denied`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `sudo` changed the user but kept `HOME=/root`, so Git, now running as `deploy`, tries to read root's config files and fails. It may also ignore the deploy user's real config (credentials, `safe.directory`).

**Diagnose →** `sudo -u deploy env | grep ^HOME`

**Fix →** Reset HOME to the target user:
```bash
sudo -H -u deploy git -C /srv/app pull --ff-only
# or: sudo -iu deploy git -C /srv/app pull --ff-only
```
**Prevent →** Run deploy commands as the service user via a systemd unit or CI agent rather than ad-hoc `sudo`.

</details>

### Q189. Your alias fails with `expansion of alias 'cleanup' failed; 'branch' is not a git command`, or runs only half of what you wrote. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** You wrote a shell pipeline or `$(...)` in a normal alias. Without a leading `!`, aliases are passed to Git as arguments, not to a shell, and quoting is easily broken in `.gitconfig`.

**Diagnose →** `git config --get alias.cleanup` and `GIT_TRACE=1 git cleanup`.

**Fix →** Prefix shell aliases with `!` and quote them correctly:
```bash
git config --global alias.cleanup '!git branch --merged main | grep -v -E "^\*|main$" | xargs -r git branch -d'
```
Shell aliases run from the repo's top-level directory; use `$GIT_PREFIX` if you need the original subdirectory.

**Prevent →** Put anything non-trivial in a script named `git-cleanup` on your PATH; Git runs it as `git cleanup`.

</details>

### Q190. Commits on a build agent show author `Jenkins <jenkins@build-07>` even though `git config user.email` returns the bot address. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Environment variables override config: `GIT_AUTHOR_NAME`, `GIT_AUTHOR_EMAIL`, `GIT_COMMITTER_NAME`, `GIT_COMMITTER_EMAIL` set by the CI tool, a plugin, or a `direnv` file.

**Diagnose →**
```bash
env | grep -E '^GIT_(AUTHOR|COMMITTER)_'
git var GIT_AUTHOR_IDENT
git var GIT_COMMITTER_IDENT
```
**Fix →** Unset or set them correctly in the job:
```bash
export GIT_AUTHOR_NAME="release-bot" GIT_AUTHOR_EMAIL="release-bot@company.com"
export GIT_COMMITTER_NAME="$GIT_AUTHOR_NAME" GIT_COMMITTER_EMAIL="$GIT_AUTHOR_EMAIL"
```
**Prevent →** Define the bot identity in one place (pipeline-level env) and check `git var GIT_AUTHOR_IDENT` in a debug step.

</details>

### Q191. `pre-commit install` fails with `[ERROR] Cowardly refusing to install hooks with core.hooksPath set.` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `core.hooksPath` is set somewhere (often globally by a corporate security tool, or by Husky in the repo). Git would ignore `.git/hooks`, so pre-commit refuses to install there.

**Diagnose →** `git config --show-origin --show-scope --get-all core.hooksPath`

**Fix →** If the setting is a leftover, unset it (`git config --unset-all core.hooksPath` at the right scope) and re-run `pre-commit install`. If a global hook directory is mandatory, have its hooks call `pre-commit` as well, or set a repo-local `core.hooksPath` that points to a directory with both.

**Prevent →** Pick one hook manager per repo and document it; avoid stacking Husky, pre-commit and global hooks.

</details>

### Q192. `git diff` says `Binary files a/strings.rc and b/strings.rc differ` for a text file, and merges of it always conflict. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The file is UTF-16 (common for Windows resource files, PowerShell output, some SQL Server scripts). The NUL bytes make Git treat it as binary, so no line diffs or text merges.

**Diagnose →**
```bash
file strings.rc                 # "Little-endian UTF-16 Unicode text"
git check-attr -a strings.rc
```
**Fix →** Store it as UTF-8 in the repo and convert on checkout:
```gitattributes
*.rc text working-tree-encoding=UTF-16LE-BOM eol=crlf
```
```bash
git add --renormalize strings.rc && git commit -m "Store .rc as UTF-8 in repo"
```
For read-only diffs without changing storage, use a textconv driver: `diff=utf16` with `git config diff.utf16.textconv 'iconv -f utf-16 -t utf-8'`.

**Prevent →** Prefer UTF-8 everywhere; add `working-tree-encoding` rules for tools that insist on UTF-16.

</details>

### Q193. A hook (or a script it calls) runs `git -C /srv/other-repo pull`, which fails with `fatal: not a git repository: '.git'` or operates on the wrong repo. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Git exports `GIT_DIR` (and in some hooks `GIT_INDEX_FILE`, `GIT_WORK_TREE`) to hook processes. Child Git commands inherit them and ignore `-C`/the current directory, so they target the original repo or a relative `.git` that does not exist there.

**Diagnose →** Log the environment in the hook: `env | grep ^GIT_ > /tmp/hook-env.txt`.

**Fix →** Clear the variables before touching another repo:
```bash
#!/usr/bin/env bash
unset GIT_DIR GIT_WORK_TREE GIT_INDEX_FILE
git -C /srv/other-repo pull --ff-only
```
In a bare-repo `post-receive` deploy hook, set them deliberately instead: `git --work-tree=/srv/app --git-dir="$PWD" checkout -f main`.

**Prevent →** Treat hooks as running inside Git's environment; always reset `GIT_*` variables before invoking Git elsewhere.

</details>

### Q194. On macOS, a file named `café.md` committed from Linux shows as both deleted and untracked, or appears as `"cafe\314\201.md"`. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Unicode normalization: macOS file APIs return decomposed names (NFD, `e` + combining accent) while Linux stores precomposed (NFC, `é`). Git sees two different byte sequences. The octal escapes come from `core.quotePath=true`.

**Diagnose →**
```bash
git config core.precomposeUnicode
git ls-files | grep -n caf
git -c core.quotePath=false status
```
**Fix →**
```bash
git config --global core.precomposeUnicode true   # convert NFD to NFC when Git reads names
git config --global core.quotePath false           # show real characters
git rm --cached 'café.md' && git add 'café.md'     # if an NFD path was already committed
```
**Prevent →** Keep file names ASCII in shared repos, or enforce NFC names in a CI check.

</details>

### Q195. `.gitattributes` has `config/prod.yml merge=ours`, but merges still overwrite or conflict on that file. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `merge=ours` in `.gitattributes` only names a driver; the driver itself must be defined in each clone's config, and config is never committed. Also, a custom merge driver runs only when both sides changed the file — if only the other branch changed it, Git simply takes their version.

**Diagnose →**
```bash
git check-attr merge -- config/prod.yml
git config --get merge.ours.driver          # empty = undefined driver
```
**Fix →**
```bash
git config merge.ours.driver true       # "true" exits 0 and keeps our version
```
For files that differ per environment, the better fix is not to merge them at all: keep env-specific config outside the branch flow (templating, separate config repo, Helm values per env).

**Prevent →** Set custom drivers in a bootstrap script, and remember the built-in `merge=union` needs no config (useful for append-only files).

</details>

### Q196. After cloning, files under `secrets/` are binary gibberish, and after unlocking they show as modified forever. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The repo uses a clean/smudge filter (git-crypt or similar) declared in `.gitattributes`. Without the key and filter config, you see the encrypted blobs. The "always modified" state happens when files were committed before the `.gitattributes` filter rule existed, so the stored blob does not match what the clean filter produces.

**Diagnose →**
```bash
git check-attr filter diff -- secrets/db.env
git config --get-regexp '^filter\.'
git-crypt status -e
```
**Fix →** Unlock with the shared key or your GPG identity (`git-crypt unlock /path/to/keyfile`). For files that were committed in plaintext, re-add them through the filter and commit:
```bash
git add --renormalize secrets/
git commit -m "Encrypt secrets via git-crypt"
```
Note that the plaintext remains in history — rotate those secrets.

**Prevent →** Add the filter rule before adding secret files, run `git-crypt status` in CI, and prefer an external secrets manager or SOPS for anything production.

</details>

### Q197. Right after a fresh clone, `git status` shows files as modified, and `git restore .` or `git reset --hard` cannot make them clean. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The files were committed with CRLF line endings, but `.gitattributes` (`* text=auto` or `eol=lf`) says they should be normalized to LF in the repo. Every checkout produces a working file that, once normalized, differs from the CRLF blob in the index, so Git reports it as modified.

**Diagnose →**
```bash
git ls-files --eol | grep 'i/crlf'
git diff --ignore-cr-at-eol --stat    # empty = only line endings differ
```
**Fix →** Normalize the repo once and commit:
```bash
git add --renormalize .
git commit -m "Renormalize line endings"
```
**Prevent →** Commit `.gitattributes` at repo creation and add a CI check that `git ls-files --eol` contains no `i/crlf` for text files.

</details>

### Q198. Inside a dev container with a bind-mounted repo, `git status` lists every file as modified, while Git on the Windows/macOS host says the tree is clean. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Two different Git installations share one working tree with different settings: the host checked files out with CRLF (`core.autocrlf=true`) but container Git has no autocrlf; or the mount reports all files as `0755`, so container Git sees mode changes. Stat data (inode, uid, ctime) also differs between host and container, forcing rehashing and making `status` slow.

**Diagnose →**
```bash
git diff --stat | tail -1
git diff --summary | grep -c 'mode change'
git config --show-origin core.autocrlf; git config core.fileMode
```
**Fix →** Align the configs in the container for this repo:
```bash
git config core.fileMode false
git config core.autocrlf true      # match the host if it checked out CRLF
git config core.trustctime false
git update-index --refresh
```
Better: commit a `.gitattributes` with explicit `eol` rules so autocrlf no longer matters.

**Prevent →** Clone the repo inside the container volume (VS Code "Clone Repository in Container Volume") instead of sharing a host checkout between two Gits.

</details>

### Q199. You just edited a file, but `git status` says `nothing to commit, working tree clean`. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** A file system monitor (`core.fsmonitor`, builtin daemon or Watchman) missed the change and returned a stale answer, common after sleep/resume, editing through a network mount or Docker, or when the daemon crashed. Less often the file is marked `assume-unchanged` or `skip-worktree`.

**Diagnose →**
```bash
git config --show-origin core.fsmonitor
git -c core.fsmonitor=false status            # if the file appears now, fsmonitor is stale
git ls-files -v | grep -E '^[a-zS] '          # lowercase = assume-unchanged, S = skip-worktree
```
**Fix →** Restart the monitor and refresh:
```bash
git fsmonitor--daemon stop          # builtin; it restarts on the next command
# Watchman: watchman watch-del "$(pwd)" && watchman watch-project "$(pwd)"
git update-index --really-refresh
```
For flagged files: `git update-index --no-assume-unchanged <file>` or `--no-skip-worktree <file>`.

**Prevent →** Do not enable fsmonitor on network or container mounts, and avoid `assume-unchanged` as a way to "ignore" tracked config files.

</details>

### Q200. `git switch release` fails with `error: The following untracked working tree files would be overwritten by checkout:` followed by `Please move or remove them before you switch branches.` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Files that are untracked on your current branch are tracked on the target branch (for example, a generated `config.json` or a file you created that someone else committed there). Git will not destroy untracked data silently.

**Diagnose →**
```bash
git status --short --untracked-files=all | grep '^??'
git diff --stat HEAD release -- config.json
```
**Fix →** Keep your copy if it matters, then switch:
```bash
mv config.json /tmp/config.json.mine      # or: git stash push --include-untracked
git switch release
diff /tmp/config.json.mine config.json
```
To discard instead: `git clean -n` to preview, then `git clean -f config.json`, or `git switch --discard-changes release`, which overwrites untracked files in the way (they are lost).

**Prevent →** Ignore generated files consistently on all branches, and don't commit files that the build generates.

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
