# Git — Production & Best Practices

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

> Real-world Git and GitHub workflow design, protection, security, CI/CD, scale and incident recovery, plus rapid-fire commands · 200 questions · 🟢 Beginner 🟡 Intermediate 🔴 Advanced

[⬅ Back to Git](./README.md) · [🏠 Home](../README.md)

## Table of contents
- [Branching and workflow design](#branching-and-workflow-design) — Q1–Q24
- [Branch protection, rulesets, CODEOWNERS and review at scale](#branch-protection-rulesets-codeowners-and-review-at-scale) — Q25–Q46
- [Security: secrets, signing, access and supply chain](#security-secrets-signing-access-and-supply-chain) — Q47–Q73
- [GitHub Actions, environments and releases in production](#github-actions-environments-and-releases-in-production) — Q74–Q94
- [Monorepos, large files and repository performance](#monorepos-large-files-and-repository-performance) — Q95–Q122
- [Recovery and incident handling](#recovery-and-incident-handling) — Q123–Q142
- [Governance, migration, automation and cost](#governance-migration-automation-and-cost) — Q143–Q164
- [Rapid-fire commands](#rapid-fire-commands) — Q165–Q200

---

## Branching and workflow design

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q1. How would you choose a branching strategy for a team of 40 engineers shipping a SaaS product several times a day? 🟡

<details>
<summary><b>Answer</b></summary>


Trunk-based development with short-lived branches (under a day or two), required CI, a merge queue and feature flags. In production we deploy from main, so there is no develop or release branch to keep in sync. The cost is investment in fast tests, flag hygiene and progressive delivery (canary, automated rollback). If CI takes 40 minutes, fix that before anything else, because slow CI is what forces teams back to long-lived branches.

</details>


### Q2. When is GitFlow still the right choice in 2026? 🟡

<details>
<summary><b>Answer</b></summary>


When you ship versioned artifacts to customers who upgrade on their own schedule (mobile apps, on-prem software, libraries) and must support several release lines with security back-ports. Then `release/*` and `hotfix/*` branches map to real obligations. For a web service deployed continuously it adds merge overhead and drift without benefit, so we use trunk or GitHub Flow there.

</details>


### Q3. How do you support multiple released versions (1.x, 2.x, 3.x) without chaos? 🔴

<details>
<summary><b>Answer</b></summary>


Keep main as the single place for new work, cut `release/N.M` branches when a version ships, and require every fix to land on main first and then be cherry-picked with `-x` to each supported branch. Automate the back-port with labels (`backport release/2.x`) using a bot or an Action so it is not a manual chore, and publish a support policy so nobody back-ports to dead versions. Tag each release from its branch, never from main.

```bash
git switch release/2.x
git cherry-pick -x <fix-sha>
git tag -s v2.4.1 -m "2.4.1" && git push origin release/2.x v2.4.1
```

</details>


### Q4. How do feature flags change your merge and release process? 🟡

<details>
<summary><b>Answer</b></summary>


They separate merge from release. Incomplete code merges daily behind a default-off flag, QA enables it per environment or per cohort, and the release is a config change that can be reverted without a deploy. The trade-offs are flag debt and combinatorial testing, so each flag gets an owner, an expiry ticket and a removal PR. Without that, trunk-based development degrades into dead code.

</details>


### Q5. What branch naming and commit conventions do you enforce and how? 🟢

<details>
<summary><b>Answer</b></summary>


`type/TICKET-123-short-description` for branches, Conventional Commits for the PR title (since we squash), and a PR template. Enforcement is automated: a rule on branch names via rulesets, a PR-title lint action, and commitlint in a `commit-msg` hook for local feedback. Conventions that are not checked by a bot are gone in three months.

</details>


### Q6. Squash, merge commit or rebase: what do you set as the allowed method on production repos and why? 🟡

<details>
<summary><b>Answer</b></summary>


For services we allow squash only. One PR becomes one commit on main, which makes `git revert` of a whole feature trivial, keeps `git log --first-parent` meaningful, and lets the PR title carry the Conventional Commit message. For release branches we allow merge commits so forward-merging fixes does not produce duplicate commits. We disable the methods we do not want in repo settings so reviewers do not argue about it.

</details>


### Q7. Design a branching and release process for a monorepo with 15 services deployed independently. 🔴

<details>
<summary><b>Answer</b></summary>


Single trunk, path-filtered CI, and per-service versions. Use CODEOWNERS per directory, `paths:` filters or a build graph tool (Bazel, Nx, Turborepo) to build only what changed, and tag releases per service (`payments/v1.8.2`). Deploys are triggered by changes under the service path. Shared libraries get their own versioning rules and a dependent-test run. Avoid release branches unless a service really needs LTS.

```yaml
on:
  push:
    branches: [main]
    paths: ['services/payments/**', 'libs/common/**']
```

</details>


### Q8. How do you keep long-running feature work from diverging without long-lived branches? 🟡

<details>
<summary><b>Answer</b></summary>


Slice the work: land the refactor first, add the new code path behind a flag, migrate callers, then remove the old path. Use branch by abstraction for internals and expand-and-contract for schemas. If a branch must live a week or more, merge main in daily and keep a draft PR open so CI proves compatibility every day.

</details>


### Q9. What does a healthy PR size look like and how do you keep it there? 🟢

<details>
<summary><b>Answer</b></summary>


Most of our PRs are under 300 changed lines and touch one concern. We keep them small by separating refactors from behaviour changes, by stacking dependent changes, and by letting a bot warn above a size threshold. Large PRs get shallow reviews, and shallow reviews are how incidents ship.

</details>


### Q10. How do you handle hotfixes in a trunk-based setup? 🟡

<details>
<summary><b>Answer</b></summary>


A hotfix is just a small PR to main that goes through the same pipeline but is flagged urgent, with a reduced reviewer requirement defined in advance (one approval from on-call owner). If main has unreleased risky changes, we deploy from a tag or a short-lived `release/*` cut from the last good deploy SHA and cherry-pick the fix, then make sure the fix is on main too. We never hand-edit production branches.

</details>


### Q11. How do you design the release process so that any production artifact is traceable to a commit? 🔴

<details>
<summary><b>Answer</b></summary>


Build once per commit, tag the image with the Git SHA, embed `git describe` output and SHA in the binary and as OCI labels, and promote the same digest through environments. Release notes and the deploy record link tag, PR numbers and digest. During an incident the question "what is running?" becomes a one-command answer.

```bash
docker build -t app:${GITHUB_SHA} \
  --label org.opencontainers.image.revision=${GITHUB_SHA} \
  --label org.opencontainers.image.source=https://github.com/org/app .
```

</details>


### Q12. What problems do you see with a `develop` branch in a continuous-deployment team? 🟡

<details>
<summary><b>Answer</b></summary>


Two sources of truth, merge-back work after every hotfix, and code that "works on develop" but has never run in production configuration. Releases become batch events with big diffs, which raises risk and slows recovery. If the team really needs an integration branch, use a short-lived environment branch for preview deploys instead of a permanent one.

</details>


### Q13. How should a new engineer get set up with the repo on day one? 🟢

<details>
<summary><b>Answer</b></summary>


A documented `make bootstrap` or devcontainer, a README with clone and branch conventions, hooks installed automatically (`core.hooksPath` or lefthook), commit signing configured, and a "good first issue" to push a real PR on day one. The first merged PR on day one tells you the pipeline, access and review are all working.

</details>


### Q14. How do you manage infrastructure-as-code repositories differently from application repos? 🟡

<details>
<summary><b>Answer</b></summary>


Terraform and Kubernetes repos are reviewed by plan output, not just by diff. We run `terraform plan` on every PR and post it as a comment, require code owner approval from SRE, apply only from main through a pipeline with OIDC credentials, and protect state. Environments live in directories (not long-lived branches) so promotion is a visible diff between folders.

</details>


### Q15. Branch-per-environment versus directory-per-environment for GitOps: how do you decide? 🔴

<details>
<summary><b>Answer</b></summary>


We use directories (or overlays) on a single branch. Branch-per-environment (`dev`, `staging`, `prod`) makes promotion a merge that carries unrelated changes, hides drift between environments and complicates hotfixes. Directories give a reviewable diff for what differs between environments and one history. Argo CD or Flux tracks a path per environment.

```text
apps/payments/base
apps/payments/overlays/dev
apps/payments/overlays/prod
```

</details>


### Q16. How do you promote a change across environments with Git as the source of truth? 🟡

<details>
<summary><b>Answer</b></summary>


CI builds an image and a bot opens a PR bumping the image tag in the dev overlay, merge deploys to dev automatically, and subsequent promotion PRs bump staging and prod after checks pass or approval. Every deploy is a commit, so rollback is `git revert` and audit is `git log`.

</details>


### Q17. What is the value of a PR template and CODEOWNERS together? 🟢

<details>
<summary><b>Answer</b></summary>


The template makes authors state the why, the test evidence and the rollback plan; CODEOWNERS pulls in the right reviewers automatically. Together they reduce the "who should look at this" overhead and raise review quality without more meetings.

</details>


### Q18. How do you deal with a team that keeps force-pushing to shared branches? 🟡

<details>
<summary><b>Answer</b></summary>


Fix the system, not the person: protect shared branches so force-push is blocked, give each developer private feature branches (rebase freely there) and teach `--force-with-lease`. Add a lightweight rule that a branch with more than one contributor is never rebased without agreement.

</details>


### Q19. How do you design Git workflows for regulated environments (SOX, PCI, ISO 27001)? 🔴

<details>
<summary><b>Answer</b></summary>


Enforce separation of duties: no direct pushes, at least one independent approver who is not the author, required status checks, signed commits where policy demands it, immutable audit logs of merges and deployments, and approvals recorded in the PR. Use environments with required reviewers for production deploys and stream the org audit log to a SIEM. The evidence is the PR, the check run and the deployment record, so make each of them complete and undeletable.

</details>


### Q20. How do you decide when to split a repo or merge repos together? 🟡

<details>
<summary><b>Answer</b></summary>


Split when teams have independent release cadences, ownership and tooling, and the shared surface is a stable API. Merge into a monorepo when changes routinely span repos, versioning shared libraries causes pain, or you want atomic refactors. Count how often a typical change touches more than one repo. If it is above roughly one in five, the repo boundary is costing you.

</details>


### Q21. What is your rule for keeping `main` green? 🟢

<details>
<summary><b>Answer</b></summary>


Nothing merges without required checks, a broken main is the top priority and gets reverted first and debugged second, and merge queue prevents interaction failures. The revert should take minutes, which is why small squash commits matter.

</details>


### Q22. How do you version internal libraries consumed from Git? 🟡

<details>
<summary><b>Answer</b></summary>


Tag releases with semantic versions and consume tags (or published packages), never a branch. Pin by tag or SHA in dependency files, use Renovate or Dependabot to propose upgrades, and avoid `git+https://...#main` dependencies in production builds because they make builds non-reproducible.

</details>


### Q23. You inherit a repo with 400 stale branches and no conventions. How do you clean it up safely? 🔴

<details>
<summary><b>Answer</b></summary>


List merged and stale branches, publish the deletion list with a deadline, then delete in batches while keeping tags. Turn on auto-delete head branches, add a branch-name ruleset and a scheduled workflow that reports branches with no commits for 90 days.

```bash
git for-each-ref --sort=committerdate refs/remotes/origin \
  --format='%(committerdate:short) %(refname:short)' | head -50
git branch -r --merged origin/main | grep -v 'main$'
```

</details>


### Q24. How do you set expectations for review turnaround and what do you do when reviews stall? 🟡

<details>
<summary><b>Answer</b></summary>


A team agreement such as first review within one working day, a rotation or auto-assignment via CODEOWNERS teams, a daily review slot, and dashboards for PR age. When one PR stalls, escalate in the team channel and pair on the review; do not bypass protections.

## Branch protection, rulesets, CODEOWNERS and review at scale

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

</details>


### Q25. What branch protection settings do you enable on `main` by default? 🟢

<details>
<summary><b>Answer</b></summary>


Require a PR with at least one approval, dismiss stale approvals on new commits, require status checks to pass (and branches up to date or use merge queue), require conversation resolution, require linear history if we squash, block force pushes and deletions, and apply the rules to admins too. Code owner review is added on sensitive paths.

</details>


### Q26. Why do rulesets win over classic branch protection for an organisation? 🟡

<details>
<summary><b>Answer</b></summary>


Rulesets can be defined once at org level and target repositories by name or property, cover tags as well as branches, stack with other rulesets (most restrictive wins), include per-actor bypass lists with audit and have an evaluate mode to test before enforcing. Classic protection must be configured per repo and drifts.

</details>


### Q27. How do you roll out a new org-wide ruleset without breaking 300 repos? 🔴

<details>
<summary><b>Answer</b></summary>


Create it in Evaluate mode, review the "would have blocked" insights for a week, communicate exceptions, then enforce for a pilot set of repos and expand. Pair it with a script or Terraform (`github_organization_ruleset`) so it is versioned.

```hcl
resource "github_organization_ruleset" "main" {
  name        = "protect-default-branch"
  target      = "branch"
  enforcement = "evaluate"
}
```

</details>


### Q28. How do you protect release tags from being moved or deleted? 🟡

<details>
<summary><b>Answer</b></summary>


A tag ruleset that targets `v*` blocks deletion and non-fast-forward updates (and restricts who can create them). Release assets are immutable if you enable immutable releases, and we also consume artifacts by digest so a moved tag cannot change what is deployed.

</details>


### Q29. How do you structure CODEOWNERS for a large org? 🟡

<details>
<summary><b>Answer</b></summary>


Own by team, never by individual, with a catch-all owner at the top and specific overrides below (last match wins). Keep owners teams small enough that review is fast, require approval only on high-risk paths such as `/infra`, `/auth` and CI config, and validate the file in CI because a typo silently disables ownership.

```text
*                      @org/eng
/.github/              @org/platform
/services/payments/    @org/payments @org/security
```

</details>


### Q30. CODEOWNERS says three teams own a path but PRs stall waiting for any one of them. How do you fix it? 🔴

<details>
<summary><b>Answer</b></summary>


Code owner review needs one approval from any listed owner for each matching pattern, so the problem is usually response time or a team with no active members. Use a smaller on-call or rota team, enable auto-assignment with load balancing in the team's review settings, and audit with a scheduled job that flags teams with no recent reviews. For urgent paths define a documented break-glass bypass in the ruleset rather than admin overrides.

</details>


### Q31. Why require status checks by job name and what is a common pitfall? 🟢

<details>
<summary><b>Answer</b></summary>


The required check is matched by name. If a workflow is skipped through path filters, the required check stays pending forever and blocks the PR. Fix by having an always-running aggregate job (`ci-ok`) as the single required check, with `needs:` on the real jobs and `if: always()` logic.

```yaml
ci-ok:
  needs: [build, test, lint]
  if: always()
  runs-on: ubuntu-latest
  steps:
    - run: test "${{ contains(needs.*.result, 'failure') || contains(needs.*.result, 'cancelled') }}" = "false"
```

</details>


### Q32. What do "dismiss stale approvals" and "require approval of the most recent push" prevent? 🟡

<details>
<summary><b>Answer</b></summary>


Without them an author can get approval and then push different code. Dismissal removes approvals when new commits arrive, and the most-recent-push rule requires someone other than the pusher to approve the final state. This closes the last-minute-change hole that audit teams ask about.

</details>


### Q33. How do you let a bot (Renovate, release-please) merge without weakening protection? 🔴

<details>
<summary><b>Answer</b></summary>


Use a GitHub App with only the permissions it needs and add that app to the ruleset bypass list narrowly, or better, keep protection intact and enable auto-merge on PRs that pass required checks and an approval from a bot-approver workflow for low-risk updates (patch bumps). Never give a long-lived PAT with admin scope to a bot.

</details>


### Q34. How do you handle emergency changes that need to skip review? 🟡

<details>
<summary><b>Answer</b></summary>


Pre-define a break-glass path: a ruleset bypass limited to the incident-commander team, usage logged in the audit log, and a mandatory retroactive PR review within 24 hours. The existence of a sanctioned fast path keeps people from disabling protections ad hoc.

</details>


### Q35. What are the pros and cons of requiring a linear history? 🟡

<details>
<summary><b>Answer</b></summary>


Pros: simple `git log`, easy bisect and straightforward reverts. Cons: it forbids merge commits, so release branches that merge forward must use rebase or squash and cherry-pick, and rebased histories lose the true merge context. It pairs naturally with squash-only repos.

</details>


### Q36. What should a PR checklist enforce for production-bound changes? 🟢

<details>
<summary><b>Answer</b></summary>


Tests added or updated, no secrets or debug code, docs and runbooks updated, migrations backward compatible, feature flag named, rollout and rollback described, and dashboards or alerts adjusted. Keep it to six lines or people stop reading it.

</details>


### Q37. How do you make code review effective at scale (hundreds of PRs a day)? 🔴

<details>
<summary><b>Answer</b></summary>


Automate everything mechanical (format, lint, types, SAST, dependency review), keep PRs small, auto-route via CODEOWNERS, set review SLOs and track PR cycle time with the GitHub API, use AI-assisted review for first-pass comments but keep a human approver, and measure review depth (comments per PR, post-merge defects), not just speed.

</details>


### Q38. What is dependency review and why make it a required check? 🟡

<details>
<summary><b>Answer</b></summary>


`actions/dependency-review-action` compares dependencies added by a PR against the advisory database and licence policy, failing on known vulnerabilities or disallowed licences. Making it required stops vulnerable packages entering at the PR rather than being found by a scanner later.

```yaml
- uses: actions/dependency-review-action@v4
  with:
    fail-on-severity: high
```

</details>


### Q39. How do you audit who can push or merge across an organisation? 🟡

<details>
<summary><b>Answer</b></summary>


Review org roles and team permissions through the API, list rulesets and bypass actors, and stream the audit log to a SIEM. Run periodic checks for repos with no protection, admins with direct push, and outside collaborators.

```bash
gh api orgs/ORG/rulesets
gh api repos/ORG/REPO/collaborators --jq '.[] | {login, role_name}'
```

</details>


### Q40. Design a least-privilege access model for GitHub across 50 teams. 🔴

<details>
<summary><b>Answer</b></summary>


Org owners are a tiny group; everything else is by team with roles Read, Triage, Write, Maintain and Admin granted through IdP-synced teams (SAML SSO + SCIM). Default repo permission is None or Read, outside collaborators are rare and time-boxed, admin rights belong to a platform team, and custom repository roles cover odd cases. Quarterly access reviews are generated from the API.

</details>


### Q41. What is the difference between a GitHub App, an OAuth app and a PAT for automation? 🟡

<details>
<summary><b>Answer</b></summary>


A GitHub App has fine-grained permissions, acts as its own identity, installs per repo and produces short-lived tokens (1 hour) and higher rate limits. OAuth apps act as the user with broad scopes. PATs are tied to a person and are the riskiest to leave in CI. We choose Apps for bots, `GITHUB_TOKEN` for workflows, and fine-grained PATs only when nothing else fits.

</details>


### Q42. Why make "Require conversation resolution" mandatory? 🟢

<details>
<summary><b>Answer</b></summary>


It stops merges while a reviewer's open concern is unaddressed. It is cheap, creates a record of how feedback was handled and prevents "I'll fix it later" from silently becoming never.

</details>


### Q43. How do you stop people merging their own PRs with admin rights? 🟡

<details>
<summary><b>Answer</b></summary>


Enable "do not allow bypassing the above settings" and keep the bypass list empty or limited to a break-glass group. Require approval from someone other than the last pusher. Admin ability to bypass is a policy decision; making it explicit and audited is what satisfies auditors.

</details>


### Q44. PRs pass CI but main breaks after merging. What process fix do you apply? 🔴

<details>
<summary><b>Answer</b></summary>


This is a semantic merge conflict from two PRs that were each green against an older base. Turn on merge queue (or require branches to be up to date for critical repos), have the queue run the full required checks on the combined candidate, and make the `merge_group` trigger part of every required workflow. Track queue wait time so engineers do not circumvent it.

</details>


### Q45. How would you roll back a bad deploy that came from a merged PR? 🟡

<details>
<summary><b>Answer</b></summary>


Revert on main with a PR (`git revert -m 1` for merge commits, plain revert for squash), run the pipeline, and deploy; in parallel, if the deploy tool supports it, roll back the artifact to the previous digest to restore service faster than a pipeline. The revert PR then becomes the permanent fix record.

</details>


### Q46. What labels and automation do you put on PRs? 🟢

<details>
<summary><b>Answer</b></summary>


Size labels, area labels from changed paths (`actions/labeler`), `needs-docs`, `breaking-change`, and auto-assign reviewers. Labels feed release notes categories and backport automation, so they pay for themselves.

## Security: secrets, signing, access and supply chain

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

</details>


### Q47. What are the first things you do when a secret is committed to a repo? 🟢

<details>
<summary><b>Answer</b></summary>


Revoke and rotate the credential immediately, because assume it is already copied. Then check logs for use of the key, remove it from history only as a secondary step, and add a preventive control. Cleaning history without rotating is theatre.

</details>


### Q48. How do you prevent secrets from entering the repo in the first place? 🟡

<details>
<summary><b>Answer</b></summary>


Layered controls: a local pre-commit scanner (gitleaks or detect-secrets), GitHub push protection that blocks at push time, secret scanning with custom patterns, a CI scan as a backstop, and an engineering habit of using a secrets manager or OIDC. No single layer is enough because hooks are skippable.

```yaml
repos:
  - repo: https://github.com/gitleaks/gitleaks
    rev: v8.21.2
    hooks:
      - id: gitleaks
```

</details>


### Q49. Walk through purging a leaked secret from a 5 GB repo with 200 contributors. 🔴

<details>
<summary><b>Answer</b></summary>


Rotate first. Announce a freeze, take a mirror backup, run `git filter-repo --replace-text` on a fresh mirror clone, push all branches and tags with force (temporarily lifting protection through the ruleset bypass), then contact GitHub support to remove cached PR refs (`refs/pull/*`) and dangling views. Everyone must re-clone because old clones can reintroduce the secret on the next push. Forks and local clones outside your control still hold it, which is why rotation matters.

```bash
git clone --mirror git@github.com:org/app.git && cd app.git
git filter-repo --replace-text ../replacements.txt
git push --force --mirror
```

</details>


### Q50. Why use OIDC federation for cloud credentials from GitHub Actions instead of stored keys? 🟡

<details>
<summary><b>Answer</b></summary>


The workflow gets a short-lived OIDC token per run, and the cloud (AWS, Azure, GCP) trusts it only for specific repos, branches or environments through the `sub` claim. No long-lived key sits in secrets to leak or rotate. Scope the trust policy tightly, for example to `repo:org/app:environment:production`.

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

</details>


### Q51. What is your policy on signed commits and how do you enforce it? 🟡

<details>
<summary><b>Answer</b></summary>


Require signed commits on protected branches through the ruleset, standardise on SSH signing (simple to distribute and works with hardware keys) or gitsign for keyless, and document how to configure it. With squash merges via the web UI GitHub signs the merge commit itself, so the rule stays workable. Signing proves the commit came from a key bound to the account; it does not prove the code is safe.

</details>


### Q52. What do signed commits not protect against, and what supply chain controls complement them? 🔴

<details>
<summary><b>Answer</b></summary>


A compromised developer account or laptop signs malicious code happily, and signatures say nothing about dependencies or build integrity. Complement with required review, build provenance (SLSA attestations from `actions/attest-build-provenance`), signed artifacts (cosign), pinned dependencies and actions by SHA, and a hardened, ephemeral build environment.

</details>


### Q53. Why pin third-party GitHub Actions to a commit SHA? 🟡

<details>
<summary><b>Answer</b></summary>


Tags are mutable, so a compromised maintainer can repoint `v4` to malicious code (as in the tj-actions/changed-files incident). A full 40-character SHA is immutable. Use Dependabot or Renovate to bump pinned SHAs, and add a comment with the human version.

```yaml
- uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4.2.2
```

</details>


### Q54. How do you harden GitHub Actions workflows against injection? 🔴

<details>
<summary><b>Answer</b></summary>


Never interpolate untrusted values (`github.event.pull_request.title`, branch names, issue bodies) directly in `run:`. Pass them through `env:` and quote them in the shell, set minimal `permissions`, avoid `pull_request_target` with checkout of PR code, restrict allowed actions at org level, and run `zizmor` or `actionlint` in CI.

```yaml
- env:
    TITLE: ${{ github.event.pull_request.title }}
  run: echo "$TITLE"
```

</details>


### Q55. How do you manage SSH keys and deploy keys safely? 🟡

<details>
<summary><b>Answer</b></summary>


Personal SSH keys are per user with passphrases or hardware-backed (FIDO2 `ed25519-sk`) keys and are revoked when people leave, via SSO. Deploy keys are repo-scoped and read-only unless truly needed. Where possible use GitHub Apps rather than deploy keys because they are auditable and expire tokens.

</details>


### Q56. How do you offboard a developer's Git access? 🟢

<details>
<summary><b>Answer</b></summary>


Remove them from the IdP so SSO and SCIM deprovision the GitHub membership, which cuts their SSH keys and tokens for org resources. Reassign their open PRs and CODEOWNERS entries, revoke any PATs and deploy keys they created, and review recent audit-log activity.

</details>


### Q57. What is a safe approach to storing environment-specific config and secrets in Git? 🟡

<details>
<summary><b>Answer</b></summary>


Keep config in Git, keep secrets out. For secrets that must be versioned (GitOps), encrypt them with SOPS (age or KMS keys) or use Sealed Secrets, or reference an external secrets manager through External Secrets Operator. Reviewers see changes to the key names but not the values.

```bash
sops --encrypt --age age1... secrets.yaml > secrets.enc.yaml
```

</details>


### Q58. How would you detect a history rewrite or tampering on a critical branch? 🔴

<details>
<summary><b>Answer</b></summary>


Block force-push and deletion through rulesets, stream the audit log for `protected_branch.force_push` style events, and keep an independent mirror whose refs are compared on a schedule. Signed tags with a pinned verifier key give an additional integrity anchor for releases.

</details>


### Q59. How do you scan a repo's full history for secrets? 🟡

<details>
<summary><b>Answer</b></summary>


Run a history-aware scanner, since scanning only HEAD misses removed secrets. Triage by verified-live credentials first.

```bash
gitleaks detect --source . --log-opts="--all" --report-format sarif --report-path gl.sarif
trufflehog git file://. --only-verified
```

</details>


### Q60. What is Dependabot versus Renovate and how do you tune them to avoid noise? 🟡

<details>
<summary><b>Answer</b></summary>


Both open dependency update PRs. Dependabot is built in and simple; Renovate is more configurable (grouping, regex managers, automerge, schedules, monorepo support). Group related updates, set weekly schedules, automerge safe patches with passing checks and require a human for majors.

</details>


### Q61. How do you secure the CI path for pull requests from forks of a public repo? 🔴

<details>
<summary><b>Answer</b></summary>


Use `pull_request` (no secrets) for untrusted code, require approval for first-time and all outside contributors' workflow runs, and do privileged follow-up work (labeling, commenting) in a separate `workflow_run` workflow that never checks out or executes PR code, only consumes sanitised artifacts. Set tokens to read-only by default.

</details>


### Q62. What sensitive files should never be in Git, and what do you use instead? 🟢

<details>
<summary><b>Answer</b></summary>


Private keys, `.env` files, cloud credentials, database dumps, Terraform state and kubeconfigs. Use a secrets manager, OIDC, encrypted state backends and `.gitignore` plus pre-commit scanning as guard rails.

</details>


### Q63. How do you enforce two-factor authentication and SSO across an organisation? 🟡

<details>
<summary><b>Answer</b></summary>


Require 2FA in org settings (members without it are removed) and enforce SAML SSO so tokens and SSH keys must be authorised for the org. Prefer passkeys or hardware keys for admins. With Enterprise Managed Users the IdP fully owns the accounts.

</details>


### Q64. What does a software supply chain attack on a Git-based pipeline look like and how do you reduce blast radius? 🔴

<details>
<summary><b>Answer</b></summary>


Typical paths: malicious dependency, compromised GitHub Action, stolen PAT, poisoned cache and injected workflow input. Reduce blast radius with pinned versions and lockfiles, minimal job permissions, environment-scoped secrets with reviewers, OIDC instead of keys, ephemeral runners, egress restrictions (Harden-Runner), no cache sharing across trust levels, and artifact attestations that deployments verify.

</details>


### Q65. How do you verify the integrity of what you deploy back to the source commit? 🟡

<details>
<summary><b>Answer</b></summary>


Attest at build time with provenance linking the digest to the repo, workflow and SHA, then verify before deploy with an admission policy (Kyverno or Sigstore policy-controller) or `gh attestation verify`.

```bash
gh attestation verify oci://ghcr.io/org/app@sha256:... --owner org
```

</details>


### Q66. What are the risks of using a self-hosted runner for a public repo? 🟡

<details>
<summary><b>Answer</b></summary>


Untrusted PR code runs on your infrastructure, can persist malware between jobs, pivot into your network and read other jobs' leftovers. Use self-hosted runners only for private repos, make them ephemeral (one job per VM or pod via Actions Runner Controller), isolate their network, and keep them off production credentials.

</details>


### Q67. How do you restrict who can create or push release tags and trigger production deploys? 🔴

<details>
<summary><b>Answer</b></summary>


Tag ruleset restricting creation of `v*` to the release team and the release bot app, a production environment with required reviewers and a deployment branch/tag policy, and an OIDC trust policy that only accepts `environment:production` from that repo. Each layer fails safe on its own.

</details>


### Q68. What does GitHub's audit log give you and what do you export? 🟢

<details>
<summary><b>Answer</b></summary>


Org-level events: permission changes, repo visibility changes, rule bypasses, token usage, secret-scanning alert dismissals. Enterprise customers stream it to Splunk, Datadog or S3 for retention beyond the built-in window and alerting.

</details>


### Q69. How do you prevent accidental exposure by making a private repo public? 🟡

<details>
<summary><b>Answer</b></summary>


Restrict repository visibility changes to org owners, disable "members can change visibility", require a pre-publish checklist run (secret scan of full history, licence check, remove internal URLs), and consider publishing a squashed fresh history for the first public release.

</details>


### Q70. What is a safe pattern for third-party contributions to an internal repo? 🟡

<details>
<summary><b>Answer</b></summary>


Have outside collaborators work from forks with PRs, give them no write access, require maintainers to approve workflow runs, and enforce CLA or DCO (`Signed-off-by`) checks as required status checks.

</details>


### Q71. How do you design secrets management for a Git-driven deployment (GitOps) on Kubernetes? 🔴

<details>
<summary><b>Answer</b></summary>


Git holds references, not values. External Secrets Operator syncs from AWS Secrets Manager or Vault into cluster Secrets using workload identity, so no secret material crosses Git or CI. If encryption in Git is preferred, use SOPS with KMS and Flux or Argo CD decryption support, and rotate keys on a schedule. Secret access is audited at the secrets-manager layer.

</details>


### Q72. What is `git-crypt` and when would you not use it? 🟡

<details>
<summary><b>Answer</b></summary>


It transparently encrypts selected files in a repo with a symmetric key shared to users via GPG. It does not hide file names or history metadata, makes diffs useless and key rotation or removing a user requires re-encrypting history. For modern teams SOPS plus a KMS is easier to operate.

</details>


### Q73. How do you give CI read access to a private submodule or private dependency repo? 🟡

<details>
<summary><b>Answer</b></summary>


Use a GitHub App installation token minted per run (`actions/create-github-app-token`) with read access to the specific repos, instead of a PAT. For checkout, pass the token and enable submodules.

```yaml
- uses: actions/create-github-app-token@v1
  id: app
  with:
    app-id: ${{ vars.APP_ID }}
    private-key: ${{ secrets.APP_KEY }}
- uses: actions/checkout@v4
  with: { token: "${{ steps.app.outputs.token }}", submodules: recursive }
```

## GitHub Actions, environments and releases in production

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

</details>


### Q74. How do you structure workflows for a typical service (build, test, deploy)? 🟢

<details>
<summary><b>Answer</b></summary>


One CI workflow on PRs (lint, unit tests, build, security scans) and one CD workflow on main or tags (build image, push, deploy dev, then promote). Reuse logic via reusable workflows or composite actions rather than copying YAML between repos.

</details>


### Q75. How do reusable workflows differ from composite actions and when do you use each? 🟡

<details>
<summary><b>Answer</b></summary>


A reusable workflow (`workflow_call`) is a whole job graph with its own runners, secrets and permissions, called by a job. A composite action bundles steps and runs inside the caller's job. Use reusable workflows to standardise pipelines org-wide, composite actions for small repeated step groups.

```yaml
jobs:
  ci:
    uses: org/.github/.github/workflows/node-ci.yml@v3
    secrets: inherit
```

</details>


### Q76. How do you speed up slow GitHub Actions pipelines? 🟡

<details>
<summary><b>Answer</b></summary>


Cache dependencies and Docker layers (`actions/cache`, `setup-*` caching, buildx `cache-to type=gha`), run independent jobs in parallel, shard tests with a matrix, use path filters, cancel superseded runs with `concurrency`, and move to larger or faster runners for the long pole. Measure with job timing before tuning.

```yaml
concurrency:
  group: ci-${{ github.ref }}
  cancel-in-progress: true
```

</details>


### Q77. Design a progressive delivery pipeline from merge to production using GitHub Actions and environments. 🔴

<details>
<summary><b>Answer</b></summary>


Merge to main builds one immutable image tagged with the SHA. A `dev` environment deploys automatically, integration tests run, `staging` deploys with a smoke suite, and `production` needs a required reviewer and a wait timer. Production rolls out via canary (Argo Rollouts or similar) with automated analysis and rollback on error-rate SLO breach. Every stage uses the same digest, and each deploy is recorded as a GitHub Deployment so the history is visible.

</details>


### Q78. How do you manage secrets and variables across org, repo and environment levels? 🟡

<details>
<summary><b>Answer</b></summary>


Org secrets for shared items scoped to selected repos, repo secrets for repo-specific values, and environment secrets for stage-specific values that only unlock after protection rules pass. Non-secret config goes in variables (`vars.*`). Prefer OIDC over any of them for cloud access.

</details>


### Q79. What does `concurrency` do in a deploy workflow and why does it matter? 🟢

<details>
<summary><b>Answer</b></summary>


It serialises runs in a group so only one deploy per environment runs at a time. For deploys set `cancel-in-progress: false` so a running deployment is never killed halfway.

```yaml
concurrency:
  group: deploy-production
  cancel-in-progress: false
```

</details>


### Q80. How do you automate versioning and release notes? 🟡

<details>
<summary><b>Answer</b></summary>


With Conventional Commits and release-please or semantic-release: the tool opens a release PR that bumps the version and updates `CHANGELOG.md`, and merging it creates the tag and GitHub Release. This turns a release into a reviewed merge. Alternatively use auto-generated notes configured through `.github/release.yml` categories.

</details>


### Q81. How do you make a release reproducible and rollback-safe? 🔴

<details>
<summary><b>Answer</b></summary>


Build from a tag, pin base images by digest and dependencies by lockfile, record the SBOM and provenance, store artifacts immutably in a registry, and keep the last N versions deployable. Rollback is redeploying a previous digest, not rebuilding old source, since rebuilds can differ.

</details>


### Q82. How do you deal with flaky tests in CI that block merges? 🟡

<details>
<summary><b>Answer</b></summary>


Quarantine them with a tracked ticket and tag so they run but do not block, retry at test level not job level, record flake rates, and enforce an SLA to fix or delete. Blanket "re-run until green" trains people to ignore real failures.

</details>


### Q83. How do you keep Actions costs under control? 🟡

<details>
<summary><b>Answer</b></summary>


Use Linux runners (cheaper than macOS or Windows multipliers), cancel redundant runs, set `timeout-minutes`, restrict triggers and path filters, clean old artifacts and caches with retention settings, and self-host or use larger-runner groups only where they pay off. Review usage reports monthly.

```yaml
jobs:
  test:
    timeout-minutes: 15
```

</details>


### Q84. What is a matrix build and when do you use it? 🟢

<details>
<summary><b>Answer</b></summary>


A job strategy that expands into multiple runs across variable combinations such as OS, language version or test shard. Use `fail-fast: false` when you need complete results.

```yaml
strategy:
  fail-fast: false
  matrix:
    node: [20, 22]
```

</details>


### Q85. How do you build a safe automated rollback in a Git-driven deployment? 🔴

<details>
<summary><b>Answer</b></summary>


Gate every rollout on health signals (SLO burn, error rate, latency) via progressive delivery. On failure the controller reverts traffic, and a bot opens a revert PR for the offending commit so Git matches reality. The key rule is that Git and the cluster must never silently diverge for long.

</details>


### Q86. How do you handle database migrations in a Git-based release process? 🟡

<details>
<summary><b>Answer</b></summary>


Ship migrations in the same PR but deploy them separately using expand and contract: backward-compatible schema change first, code that uses it next, cleanup migration later. Run migrations as a pipeline stage with approvals in production, never automatically at app startup on every replica.

</details>


### Q87. How do you share workflow templates and policy across hundreds of repos? 🟡

<details>
<summary><b>Answer</b></summary>


An org `.github` repo with starter workflows, reusable workflows pinned to tags, required workflows through rulesets, and a template repository for new services. Updates propagate via Renovate PRs that bump the pinned reusable-workflow version.

</details>


### Q88. What are the production risks of `workflow_run` and `repository_dispatch` triggers? 🔴

<details>
<summary><b>Answer</b></summary>


They run in the default branch context with write tokens and secrets, so any untrusted data they consume (artifact contents, client_payload) is an injection vector. Validate and sanitise inputs, never execute artifact content, restrict who can dispatch, and keep the privileged workflow minimal.

</details>


### Q89. How do you use GitHub Environments for approvals? 🟢

<details>
<summary><b>Answer</b></summary>


Create `staging` and `production` environments, add required reviewers, a wait timer and deployment branch rules, and reference them from deploy jobs. The deploy pauses for approval and the approver and time are logged in the Deployment history.

</details>


### Q90. How do you cache Docker builds in Actions for faster image builds? 🟡

<details>
<summary><b>Answer</b></summary>


Use Buildx with the GitHub cache backend (or a registry cache) and keep layer order stable: dependency manifests first, source last.

```yaml
- uses: docker/build-push-action@v6
  with:
    push: true
    tags: ghcr.io/org/app:${{ github.sha }}
    cache-from: type=gha
    cache-to: type=gha,mode=max
```

</details>


### Q91. How do you run Actions workflows locally or test them before merging? 🟡

<details>
<summary><b>Answer</b></summary>


`act` runs many workflows in Docker locally for quick feedback, `actionlint` catches syntax and expression errors, and a sandbox repo plus `workflow_dispatch` is the most faithful test. Treat workflow changes as production code with review via CODEOWNERS on `.github/`.

</details>


### Q92. How do you design CI for a repo where a full pipeline takes 50 minutes? 🔴

<details>
<summary><b>Answer</b></summary>


Break it into affected-only builds using a dependency graph, shard tests, remote build cache (Bazel, Nx), push expensive end-to-end tests to post-merge or nightly with fast pre-merge subsets, and set a target such as p90 under 15 minutes. Track the numbers as a team metric, because CI time is the biggest driver of branch lifetime.

</details>


### Q93. What do you put in `permissions:` for a build, a release and a PR comment workflow? 🟡

<details>
<summary><b>Answer</b></summary>


Build: `contents: read`. Release: `contents: write` (and `packages: write` or `id-token: write` only if needed). PR comment bot: `pull-requests: write`. Declaring at the workflow level and then granting more at job level keeps blast radius small.

</details>


### Q94. How do you version and publish container images from Git tags? 🟡

<details>
<summary><b>Answer</b></summary>


On `v*` tags build and push `ghcr.io/org/app:1.4.0`, `:1.4` and `:latest`, but deploy by digest. Use `docker/metadata-action` to derive tags from refs consistently and attach provenance.

## Monorepos, large files and repository performance

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

</details>


### Q95. What are the main trade-offs of a monorepo? 🟡

<details>
<summary><b>Answer</b></summary>


Pros: atomic cross-project changes, one dependency version per library, easy refactors and shared tooling. Cons: slow clones and status without tuning, CI must be affected-aware, ownership needs CODEOWNERS discipline, and a single repo outage or bad merge hits everyone. It works when you invest in build caching and sparse checkout early.

</details>


### Q96. How do you make CI efficient in a monorepo? 🔴

<details>
<summary><b>Answer</b></summary>


Compute the affected set from the dependency graph (Bazel query, Nx affected, Turborepo `--filter=...[origin/main]`), cache build outputs remotely, run only affected tests on PRs and everything on main or nightly. Use `fetch-depth` large enough (or `git merge-base`) to diff against the base.

```bash
npx turbo run test --filter="...[origin/main]"
git diff --name-only $(git merge-base origin/main HEAD)
```

</details>


### Q97. How do you onboard developers to a very large repo without a 30-minute clone? 🟡

<details>
<summary><b>Answer</b></summary>


Provide a partial clone plus cone-mode sparse checkout script for each team, preload caches in a devcontainer or a prebuilt volume, and run background `git maintenance`. New developers clone only their service and shared libs.

```bash
git clone --filter=blob:none --sparse git@github.com:org/mono.git
git -C mono sparse-checkout set services/billing libs/auth
```

</details>


### Q98. What does Scalar do and when do you recommend it? 🔴

<details>
<summary><b>Answer</b></summary>


Scalar (shipped with Git) is a wrapper that clones with partial clone and sparse checkout defaults and enables a tuned set of options: fsmonitor, commit-graph, background maintenance and `feature.experimental`. We recommend it for repos above a few GB or hundreds of thousands of files.

```bash
scalar clone https://github.com/org/mono.git
scalar register
```

</details>


### Q99. `git status` takes 20 seconds in a big working tree. How do you fix it? 🔴

<details>
<summary><b>Answer</b></summary>


Enable the filesystem monitor and untracked cache so Git stops walking every file, enable split index and commit-graph, check for huge untracked directories that need ignoring, and confirm antivirus is excluding `.git`.

```bash
git config core.fsmonitor true
git config core.untrackedCache true
git config feature.manyFiles true
GIT_TRACE_PERFORMANCE=1 git status
```

</details>


### Q100. When should a team adopt Git LFS and when should it avoid it? 🟡

<details>
<summary><b>Answer</b></summary>


Adopt for large binaries that change (design files, game assets, ML model weights up to a few GB) so clones stay small. Avoid for files that never change, for build artifacts (use a registry) and for huge datasets (use object storage, DVC or lakeFS). LFS adds server cost, bandwidth quotas and a migration pain if started late.

</details>


### Q101. How do you control LFS cost and bandwidth in CI? 🟡

<details>
<summary><b>Answer</b></summary>


Skip LFS download unless needed using `GIT_LFS_SKIP_SMUDGE=1` or `lfs: false` in checkout, fetch only needed paths with `git lfs pull --include`, cache the LFS directory in CI, and clean old LFS objects.

```bash
GIT_LFS_SKIP_SMUDGE=1 git clone git@github.com:org/assets.git
git lfs pull --include="textures/hero/*"
```

</details>


### Q102. You discover 3 GB of binaries in history and the repo clones slowly. What are the options? 🔴

<details>
<summary><b>Answer</b></summary>


Rewrite history to drop or LFS-migrate the blobs (`git filter-repo --strip-blobs-bigger-than`, `git lfs migrate import --everything`), coordinate a freeze and re-clone, or start a fresh squashed repo and archive the old one read-only. Rewriting forces every open PR to rebase, so the cheaper path for some orgs is to leave history alone and use partial clones so developers do not download old blobs.

</details>


### Q103. How do you find the largest objects in a repo? 🟡

<details>
<summary><b>Answer</b></summary>


List all objects with sizes and sort. `git-sizer` gives a health report on commits, trees, blobs and refs.

```bash
git rev-list --objects --all \
 | git cat-file --batch-check='%(objecttype) %(objectname) %(objectsize) %(rest)' \
 | sort -k3 -n -r | head -20
git sizer --verbose
```

</details>


### Q104. What causes slow fetches and how do you diagnose them on the server side? 🔴

<details>
<summary><b>Answer</b></summary>


Typical causes: too many refs (thousands of stale PR/branch refs), no reachability bitmaps, many small packs, and lack of a commit-graph. Delete stale refs, enable bitmaps and multi-pack-index with scheduled repack, and use protocol v2. Check by tracing packet exchange.

```bash
GIT_TRACE_PACKET=1 GIT_TRACE_PERFORMANCE=1 git fetch origin 2>&1 | head -50
git for-each-ref | wc -l
```

</details>


### Q105. How do you reduce CI checkout time? 🟡

<details>
<summary><b>Answer</b></summary>


Use `fetch-depth: 1` when history is not needed, `filter: blob:none` plus sparse checkout for monorepos, cache the repo (or a mirror) on self-hosted runners and reuse workspace via a reference clone.

```yaml
- uses: actions/checkout@v4
  with:
    fetch-depth: 1
    sparse-checkout: |
      services/billing
      libs
```

</details>


### Q106. What are the risks of shallow clones in release automation? 🔴

<details>
<summary><b>Answer</b></summary>


Version tools that count commits or read tags (`git describe`, semantic-release, changelog generators) fail or give wrong output because history and tags are missing. Release jobs use `fetch-depth: 0` and `fetch-tags: true`, while PR validation jobs stay shallow.

</details>


### Q107. How do you manage vendored or third-party code: submodule, subtree or package manager? 🟡

<details>
<summary><b>Answer</b></summary>


A package manager with a lockfile is the default. Submodules suit pinning a source dependency you also develop; subtree suits importing code you want to patch locally, giving contributors zero extra steps. Avoid copying code by hand because provenance is lost.

</details>


### Q108. How do you operate submodules safely in CI/CD? 🔴

<details>
<summary><b>Answer</b></summary>


Pin them to commits and bump through reviewed PRs, check out with `submodules: recursive` and a token that can read them, verify submodule commits are reachable on the remote before merging (`git push --recurse-submodules=check`), and run a CI check that the recorded SHA exists on its upstream branch.

```bash
git submodule status --recursive
git submodule update --init --depth 1 --recursive
```

</details>


### Q109. What is the process to move a directory into its own repo while keeping history? 🟡

<details>
<summary><b>Answer</b></summary>


Use `git filter-repo --subdirectory-filter` on a fresh clone, push to the new repo, then replace the directory in the original with a package dependency or submodule.

```bash
git clone git@github.com:org/mono.git svc && cd svc
git filter-repo --subdirectory-filter services/billing
git remote add origin git@github.com:org/billing.git && git push -u origin main
```

</details>


### Q110. How do you merge multiple repos into a monorepo with history preserved? 🔴

<details>
<summary><b>Answer</b></summary>


Rewrite each repo into a subdirectory with `git filter-repo --to-subdirectory-filter`, then fetch each into the target and merge with `--allow-unrelated-histories`. Tags must be renamed with a prefix to avoid collisions. Open PRs need to be re-created, so schedule a cut-over window.

```bash
git filter-repo --to-subdirectory-filter services/api
git remote add api ../api && git fetch api --tags
git merge api/main --allow-unrelated-histories
```

</details>


### Q111. How do you keep a sparse checkout list maintainable for a team? 🟡

<details>
<summary><b>Answer</b></summary>


Store named profiles in the repo (`tools/sparse/payments.txt`) and a script that applies them, so the list is versioned and reviewed like code. Cone mode keeps matching fast.

```bash
git sparse-checkout set --stdin < tools/sparse/payments.txt
```

</details>


### Q112. How do you scale a self-hosted Git server (GitHub Enterprise Server, GitLab, Gitea) for heavy CI traffic? 🔴

<details>
<summary><b>Answer</b></summary>


Offload fetches to read replicas or caching proxies (Git mirrors, GitHub Enterprise cache replicas), use shallow and filtered clones in CI, enable bitmaps, schedule repacks off-peak, and watch ref count and pack count. Large fan-out from CI is the usual load, not developers.

</details>


### Q113. Why shouldn't build output and dependencies be committed? 🟢

<details>
<summary><b>Answer</b></summary>


They bloat the repo, produce noisy diffs and merge conflicts, and can differ between machines. Commit lockfiles and build from source, publish artifacts to a registry.

</details>


### Q114. How do you prevent large files from being committed accidentally? 🟡

<details>
<summary><b>Answer</b></summary>


A pre-receive or ruleset "file size limit" restriction (GitHub rulesets can block files over a size), a pre-commit hook such as `check-added-large-files`, and CI check on diff size. GitHub itself rejects files over 100 MB.

```yaml
- id: check-added-large-files
  args: ['--maxkb=1024']
```

</details>


### Q115. How does `git gc`/repack tuning differ for a repo that is mostly large binaries? 🔴

<details>
<summary><b>Answer</b></summary>


Delta compression is nearly useless on compressed binaries and burns CPU and memory. Exclude them from delta via `.gitattributes` (`-delta`) or set `core.bigFileThreshold`, reduce pack window and memory limits, and prefer LFS or an external store for them.

```text
*.zip -delta
```
```bash
git config core.bigFileThreshold 50m
```

</details>


### Q116. What should a repo health dashboard track? 🟡

<details>
<summary><b>Answer</b></summary>


Clone time, `git status` time on a reference machine, repo and pack size, ref count, largest blobs, PR cycle time, CI time, and default branch breakage rate. Alerts on trend, not absolute thresholds.

</details>


### Q117. How do you keep `blame` and history useful after a big mechanical reformat or move? 🔴

<details>
<summary><b>Answer</b></summary>


Do the reformat in a dedicated commit, add its SHA to `.git-blame-ignore-revs`, configure `blame.ignoreRevsFile` in onboarding, and keep renames separate from edits so `--follow` works. GitHub honours the ignore file automatically.

</details>


### Q118. How would you mirror a repo to a second provider for disaster recovery? 🟡

<details>
<summary><b>Answer</b></summary>


Use a scheduled workflow or server job doing `git clone --mirror` then `git push --mirror` to the secondary, with a read-only deploy key, and verify with `git ls-remote` ref counts. Mirror issues and PR metadata separately via API exports.

```bash
git clone --mirror git@github.com:org/app.git
git -C app.git push --mirror git@gitlab.com:org/app.git
```

</details>


### Q119. How do you back up GitHub beyond Git data? 🔴

<details>
<summary><b>Answer</b></summary>


Git data is recoverable from any clone, but issues, PRs, wikis, releases, Actions secrets/config and settings are not. Use the migration/export API or a backup tool, export org config as Terraform, and test a restore. Backups you never restored are assumptions.

</details>


### Q120. When is a shallow, single-branch clone enough in production tooling? 🟢

<details>
<summary><b>Answer</b></summary>


For deploy and build jobs that only need the exact commit: `git clone --depth 1 --single-branch --branch <tag>`. This minimises time and data and removes the temptation to rely on history.

</details>


### Q121. What is a reference clone and when does it help? 🟡

<details>
<summary><b>Answer</b></summary>


`git clone --reference /cache/repo.git` borrows objects from a local mirror, so many CI workspaces on one host fetch only what is missing. It speeds up self-hosted runners but couples workspaces to the cache's life, so use `--dissociate` if the workspace must stand alone.

</details>


### Q122. How do you keep a long-lived fork (vendor patch set) maintainable? 🔴

<details>
<summary><b>Answer</b></summary>


Keep patches as a small, rebased branch on top of upstream tags, with each patch isolated and documented, `rerere` enabled, and a scheduled job that attempts the rebase on each upstream release and opens an issue on conflict. Push upstream whatever you can to shrink the diff.

## Recovery and incident handling

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

</details>


### Q123. Someone force-pushed over main and 12 commits vanished. How do you recover? 🟡

<details>
<summary><b>Answer</b></summary>


Find the old tip: from a teammate's clone or CI runner (`git reflog`, `origin/main@{1}`), GitHub's push events API or the Activity view on the branch, or the PR refs. Then restore with a force push, which needs a temporary ruleset bypass.

```bash
gh api repos/ORG/REPO/activity -f ref=refs/heads/main -f activity_type=force_push
git push --force-with-lease=main:<bad-sha> origin <good-sha>:main
```

</details>


### Q124. After recovery, what do you change so a force-push to main cannot happen again? 🔴

<details>
<summary><b>Answer</b></summary>


Protect main with a ruleset that blocks non-fast-forward updates and deletions with an empty bypass list, review who had admin rights, alert on audit-log events for ref force-pushes, and write the postmortem including how detection worked.

</details>


### Q125. A bad migration commit was deployed. Walk through your response in the first 15 minutes. 🟡

<details>
<summary><b>Answer</b></summary>


Stop the bleeding first: roll back the artifact to the previous digest or toggle the flag. Declare an incident and assign roles. Then revert the commit in Git through a fast-track PR so main matches production, and only after service is stable investigate and fix forward. Do not debug while the site is down.

</details>


### Q126. How do you recover a deleted repository or branch on GitHub? 🟡

<details>
<summary><b>Answer</b></summary>


A deleted branch tied to a PR can be restored from the PR page. A deleted repo can be restored by an org owner within 90 days from the settings (for repos that were not forks with downstream forks), or by contacting support. Always keep a mirror backup for anything critical.

</details>


### Q127. How do you find who or what pushed a specific commit to a protected branch? 🔴

<details>
<summary><b>Answer</b></summary>


Combine `git log --format='%H %an %cn'` with the audit log and the PR that merged it (`gh pr list --search <sha> --state merged`). Merge commits show the committer as GitHub for web merges. The audit log shows pushes, bypasses and force-pushes with actor and time.

```bash
gh pr list --search "<sha>" --state merged --json number,author,mergedBy
```

</details>


### Q128. CI is red on main after a merge. What is your policy? 🟡

<details>
<summary><b>Answer</b></summary>


Revert first. The author or release manager reverts within minutes if the fix is not trivially obvious, since an unprotected, broken main blocks everybody. Fix forward later in a PR that includes a regression test.

</details>


### Q129. A teammate rebased a shared branch and now everyone has duplicate commits. How do you repair it? 🔴

<details>
<summary><b>Answer</b></summary>


Pick the canonical history (the rebased one), have each person fetch and move their unpushed local work onto it using `git rebase --onto`, and discard their stale copies. The key command is to replay only their unique commits.

```bash
git fetch origin
git rebase --onto origin/feature <old-upstream-sha> my-local-feature
```

</details>


### Q130. How do you recover work when a laptop dies mid-feature? 🟡

<details>
<summary><b>Answer</b></summary>


The answer is made before the failure: push WIP branches regularly (`git push origin HEAD:wip/me/feature`), enable auto-save through frequent commits, and keep work in a devcontainer or cloud dev environment. Afterwards, `git fetch` and continue; unpushed work is gone unless the disk can be read.

</details>


### Q131. You must remove a customer's personal data accidentally committed to a public repo. What are the steps? 🔴

<details>
<summary><b>Answer</b></summary>


Treat it as a data breach: notify the privacy team, take the repo private if possible, rewrite history with `git filter-repo --replace-text` or `--path ... --invert-paths`, force-push all refs, ask GitHub support to purge cached refs and PR diffs, and make sure forks are handled via a DMCA or support request. Document timeline for regulators if required.

</details>


### Q132. How do you investigate "it worked yesterday" regressions using Git? 🟡

<details>
<summary><b>Answer</b></summary>


Narrow by time and path, then bisect. List what changed in the deploy range, compare the exact SHAs running in each environment, and run `git bisect run` with a reproducer test.

```bash
git log --since="yesterday" --oneline -- services/payments
git bisect start HEAD <last-good-sha> && git bisect run ./repro.sh
```

</details>


### Q133. What goes into a Git-related postmortem? 🟡

<details>
<summary><b>Answer</b></summary>


Timeline with commit SHAs and PR links, what protection should have caught it and why it did not, detection time, recovery steps, and concrete actions (ruleset change, new test, alert). Blameless in tone, specific in actions.

</details>


### Q134. A merge queue is stuck and blocking all merges. What do you do? 🔴

<details>
<summary><b>Answer</b></summary>


Identify the PR at the head whose checks hang (the queue builds on top of it), remove it from the queue, rerun or cancel the stuck workflow, and check for a required check name that no longer reports (renamed job or path filter). Longer term add timeouts to required jobs and an alert on queue wait time.

</details>


### Q135. How do you roll back a release made from a tag? 🟡

<details>
<summary><b>Answer</b></summary>


Redeploy the previous tag's artifact digest. If the tag itself is bad, publish a new patch tag (`v1.4.1`) rather than deleting and recreating `v1.4.0`, because consumers and caches already saw the old one. Mark the broken GitHub Release as pre-release or edit notes with a warning.

</details>


### Q136. What is the first thing to check when a developer says "my push is rejected"? 🟢

<details>
<summary><b>Answer</b></summary>


Read the exact message. `fetch first` means pull or rebase; `protected branch hook declined` or `GH006` means the branch rules require a PR; `GH013` or secret detected means push protection blocked a secret; `remote: error: File ... exceeds GitHub's file size limit` means a file over 100 MB.

</details>


### Q137. How do you recover from a bad `git filter-repo` run on the shared repo? 🟡

<details>
<summary><b>Answer</b></summary>


Always run it on a fresh mirror clone and keep the untouched original, so recovery is just discarding the rewritten copy. If it was already pushed, the original history exists in the backup mirror, and `git push --force` of the original refs restores it.

</details>


### Q138. After a history rewrite, how do you get all contributors back in sync safely? 🔴

<details>
<summary><b>Answer</b></summary>


Announce the cut-over and freeze merges, have everyone re-clone or run a documented reset, and close or retarget open PRs. Old clones that push can reintroduce removed history, so add a pre-receive or ruleset check for the old root commit if necessary.

```bash
git fetch origin
git reset --hard origin/main
git reflog expire --expire=now --all && git gc --prune=now
```

</details>


### Q139. How do you detect a stale or orphaned deployment branch left in the pipeline? 🟡

<details>
<summary><b>Answer</b></summary>


Report on branches with no commits over N days and no open PR, scheduled weekly, and require a ticket or owner for any long-lived branch. Auto-delete merged PR branches.

</details>


### Q140. Why keep a runbook for Git emergencies? 🟢

<details>
<summary><b>Answer</b></summary>


At 3 a.m. nobody remembers the exact flags for restoring a force-pushed branch or the break-glass bypass. A one-page runbook covering restore, revert, history purge and escalation contacts converts stress into steps.

</details>


### Q141. You find a malicious commit pushed by a compromised account. What is your response? 🔴

<details>
<summary><b>Answer</b></summary>


Contain: suspend the account, revoke tokens and SSH keys, rotate secrets that could have been exposed. Investigate with the audit log and commit scope, revert or remove the commit, rebuild and redeploy affected artifacts from known-good SHAs, check dependent repos and releases for the same code, and notify per policy. Then add signed-commit requirements or stronger review on the affected paths.

</details>


### Q142. How do you restore a file's deleted history after a repo migration lost it? 🟡

<details>
<summary><b>Answer</b></summary>


If the old repo exists, fetch from it as a second remote and graft or use `git replace --graft` to link histories, or add the old repo as an archive reference in README. If it is gone, rebuild from backups or mirrors.

## Governance, migration, automation and cost

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

</details>


### Q143. How do you migrate 200 repos from Bitbucket or GitLab to GitHub with minimal disruption? 🟡

<details>
<summary><b>Answer</b></summary>


Inventory repos and owners, pick a pilot team, then migrate in waves using GitHub Enterprise Importer (`gh gei`) or `git clone --mirror` plus API for metadata. Recreate CI (Jenkinsfile or `.gitlab-ci.yml` to Actions via `gh actions-importer`), rotate and re-add secrets, update webhooks and deploy keys, set rulesets, then make the old repos read-only with a pointer. Run old and new CI in parallel for one release cycle.

```bash
gh gei migrate-repo --github-source-org SRC --source-repo app \
  --github-target-org DEST --target-repo app
```

</details>


### Q144. What does a migration from SVN (or a legacy monolith repo) to Git need beyond the conversion? 🔴

<details>
<summary><b>Answer</b></summary>


History conversion (`git svn clone` or `svn2git`) with author mapping, but also cleanup: remove binaries and generated files during conversion, define branching rules, retrain the team and freeze the old system on a date. Converting 15 years of history verbatim often drags in GBs of junk, so decide how much history is worth carrying over.

</details>


### Q145. What standard files should every repo have? 🟢

<details>
<summary><b>Answer</b></summary>


`README.md`, `LICENSE`, `CODEOWNERS`, `.gitignore`, `.gitattributes`, `SECURITY.md`, `CONTRIBUTING.md`, PR and issue templates, and a CI workflow. Put them in a template repository and the org-level `.github` repo for defaults.

</details>


### Q146. How do you manage GitHub organisation settings as code? 🟡

<details>
<summary><b>Answer</b></summary>


With the Terraform GitHub provider or Probot settings: repos, teams, rulesets, branch protection and environments are declared in a repo, reviewed by PR and applied by a pipeline. Drift is detected with scheduled `terraform plan`. Manual changes become visible instead of silently accumulating.

```hcl
resource "github_repository" "app" {
  name       = "app"
  visibility = "private"
  delete_branch_on_merge = true
}
```

</details>


### Q147. What is InnerSource and how does Git workflow support it? 🟡

<details>
<summary><b>Answer</b></summary>


Applying open-source practices inside a company: readable repos, CONTRIBUTING docs, forks or branches from other teams, PRs reviewed by the owning team and CODEOWNERS routing. It spreads fixes across teams without ticket queues, but needs clear review SLAs from owners.

</details>


### Q148. When do you use DCO versus CLA? 🟢

<details>
<summary><b>Answer</b></summary>


DCO is a lightweight `Signed-off-by` trailer attesting you may submit the code (`git commit -s`), enforced by a check. A CLA is a legal agreement, signed once, used by projects that need broader rights such as relicensing. Choose DCO for simplicity unless legal requires a CLA.

</details>


### Q149. What is your repo lifecycle policy? 🟢

<details>
<summary><b>Answer</b></summary>


New repos come from a template with owners recorded, inactive repos with no commits for 12 months are flagged, then archived (read-only, still searchable), and deleted only after a retention period. Archiving beats deleting because links and history remain.

</details>


### Q150. How do you measure delivery performance using Git and GitHub data? 🟡

<details>
<summary><b>Answer</b></summary>


DORA metrics: deployment frequency and lead time for changes (first commit or PR open to production deploy) come from Deployments and merge timestamps, change failure rate from reverts and incidents, and time to restore from incident data. Use them to find bottlenecks (PR wait time, CI duration), not to rank individuals.

</details>


### Q151. How do you control the cost of a GitHub Enterprise bill? 🟡

<details>
<summary><b>Answer</b></summary>


Audit seats against active users and remove dormant ones through SCIM, set Actions budgets and spending limits, reduce minutes with caching and cancellation, trim LFS and packages storage, and review Copilot seat usage. Cost reviews belong in the monthly platform review.

</details>


### Q152. How do you govern ownership in a monorepo with hundreds of teams? 🔴

<details>
<summary><b>Answer</b></summary>


CODEOWNERS per directory, enforced by a validator in CI that every path has an owner, ownership recorded in a catalogue (Backstage) and checked against the HR/team source of truth, and rotation rules for orphaned directories. Path-based rulesets add stricter review on sensitive areas. Owners are teams, so reorgs update one file.

</details>


### Q153. How do you move a large org from `master` to `main` safely? 🟡

<details>
<summary><b>Answer</b></summary>


Rename through the GitHub UI or API, which redirects and retargets open PRs, then update CI triggers, deployment configs, badges and protection. Search the org for hardcoded branch names before the change and announce a cut-over date with a one-liner for local clones.

```bash
gh api -X POST repos/ORG/REPO/branches/master/rename -f new_name=main
git branch -m master main && git fetch && git branch -u origin/main main
```

</details>


### Q154. How do you assess repo security posture at scale? 🟡

<details>
<summary><b>Answer</b></summary>


OpenSSF Scorecard runs checks (branch protection, pinned dependencies, token permissions, signed releases) per repo and reports a score. Run it on a schedule across the org, track the trend, and open issues for failures.

```bash
scorecard --repo=github.com/org/app --format=json
```

</details>


### Q155. What do you do when GitHub itself has an outage during a release? 🟡

<details>
<summary><b>Answer</b></summary>


Have a documented fallback: mirrored repo on a second host for source, artifacts and images already in a registry so deploys do not depend on GitHub, and a break-glass deploy path for critical fixes. Check githubstatus.com early and avoid risky releases during degraded service.

</details>


### Q156. What commit message policy supports automated changelogs? 🟢

<details>
<summary><b>Answer</b></summary>


Conventional Commits on the squashed PR title, validated by a PR-title check. The release tool maps `feat`, `fix` and breaking markers to sections of the changelog and the version bump.

</details>


### Q157. How do you give every PR a preview environment? 🟡

<details>
<summary><b>Answer</b></summary>


A workflow on `pull_request` builds an image, deploys it to a namespaced environment (for example `pr-482.preview.example.com`) via Helm or Argo CD ApplicationSet with the PR generator, comments the URL, and tears down on `closed`. Use quotas and a TTL job to avoid orphaned previews.

</details>


### Q158. What is auto-merge and when do you use it? 🟢

<details>
<summary><b>Answer</b></summary>


GitHub auto-merge merges a PR as soon as required reviews and checks pass. It is good for dependency bumps and for authors who do not want to babysit the PR, while protections still apply.

```bash
gh pr merge 482 --auto --squash
```

</details>


### Q159. How do you manage docs-as-code in the same Git workflow? 🟡

<details>
<summary><b>Answer</b></summary>


Docs live next to code, changes to behaviour require changes to docs in the same PR (checked by a label or CODEOWNERS for `/docs`), preview builds render on PRs, and link checkers and linters (Vale, markdownlint) run in CI. Review is the same process as for code.

</details>


### Q160. How do you version data and ML artifacts alongside Git? 🟡

<details>
<summary><b>Answer</b></summary>


Keep code and small configs in Git, large data in object storage tracked by pointers: DVC, LFS or lakeFS commits. The Git commit then pins code plus data version, giving reproducible experiments. Avoid committing notebooks with outputs, using `nbstripout`.

</details>


### Q161. How do you produce and keep an SBOM tied to a commit? 🔴

<details>
<summary><b>Answer</b></summary>


Generate it in the build (Syft, `cyclonedx` plugins), attach it as a build attestation or release asset and store it with the image digest. GitHub's dependency graph also exports an SPDX SBOM through the API for a repo snapshot.

```bash
syft ghcr.io/org/app@sha256:... -o spdx-json > sbom.spdx.json
gh api repos/ORG/REPO/dependency-graph/sbom
```

</details>


### Q162. What changes when data residency or regulation requires GitHub Enterprise Cloud with data residency or a self-hosted instance? 🔴

<details>
<summary><b>Answer</b></summary>


Your org lives on a regional tenant (GHE.com) or on GitHub Enterprise Server in your own environment. Account model is Enterprise Managed Users, URLs and API endpoints differ, some features lag, and you own upgrades, backups, HA and capacity on Server. Plan for runner placement and network paths too.

</details>


### Q163. Why is a platform team's Git guidance important for developers? 🟢

<details>
<summary><b>Answer</b></summary>


A short, opinionated guide (one branching model, one merge method, one commit style, one way to sign) removes daily friction, speeds onboarding and makes automation possible. The more options developers have, the more tooling breaks.

</details>


### Q164. How do you decide what to automate versus leave to humans in the PR process? 🟡

<details>
<summary><b>Answer</b></summary>


Automate anything deterministic: formatting, linting, labelling, assignment, dependency updates, release PRs and stale branch cleanup. Humans own design, risk and trade-off judgment. If a review comment is repeated three times, turn it into a lint rule.

## Rapid-fire commands

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

</details>


### Q165. Print the repository root directory. 🟢

<details>
<summary><b>Answer</b></summary>


```bash
git rev-parse --show-toplevel
```

</details>


### Q166. Print the current branch name (script-friendly). 🟢

<details>
<summary><b>Answer</b></summary>


```bash
git branch --show-current
```

</details>


### Q167. List files changed in the last commit. 🟢

<details>
<summary><b>Answer</b></summary>


```bash
git show --name-only --pretty=format: HEAD
```

</details>


### Q168. Show the commits I made in the last week. 🟢

<details>
<summary><b>Answer</b></summary>


```bash
git log --author="$(git config user.email)" --since="1 week ago" --oneline
```

</details>


### Q169. Show commits on my branch that are not pushed yet, and commits waiting to be pulled. 🟢

<details>
<summary><b>Answer</b></summary>


```bash
git log @{u}..HEAD --oneline   # to push
git log HEAD..@{u} --oneline   # to pull
```

</details>


### Q170. List remote branches sorted by most recent commit. 🟡

<details>
<summary><b>Answer</b></summary>


```bash
git for-each-ref --sort=-committerdate refs/remotes/origin \
  --format='%(committerdate:relative)  %(refname:short)  %(authorname)'
```

</details>


### Q171. Search the working tree for a string with line numbers. 🟢

<details>
<summary><b>Answer</b></summary>


```bash
git grep -n "TODO" -- '*.py'
```

</details>


### Q172. List modified and untracked files separately. 🟢

<details>
<summary><b>Answer</b></summary>


```bash
git ls-files -m
git ls-files -o --exclude-standard
```

</details>


### Q173. Which remote branches contain a given commit? 🟢

<details>
<summary><b>Answer</b></summary>


```bash
git branch -r --contains <sha>
git tag --contains <sha>
```

</details>


### Q174. Find the root commit of the repo. 🟢

<details>
<summary><b>Answer</b></summary>


```bash
git rev-list --max-parents=0 HEAD
```

</details>


### Q175. Show a per-file change summary between main and my branch. 🟢

<details>
<summary><b>Answer</b></summary>


```bash
git diff --name-status main...HEAD
```

</details>


### Q176. Compare one file between two branches. 🟢

<details>
<summary><b>Answer</b></summary>


```bash
git diff main feature -- src/app.py
```

</details>


### Q177. Print a file as it exists on another branch without switching. 🟢

<details>
<summary><b>Answer</b></summary>


```bash
git show main:docs/runbook.md
```

</details>


### Q178. Bring one file from another branch into the working tree. 🟢

<details>
<summary><b>Answer</b></summary>


```bash
git restore --source=release/1.4 --worktree -- config/app.yaml
```

</details>


### Q179. List tags sorted by version number, newest first. 🟢

<details>
<summary><b>Answer</b></summary>


```bash
git tag --sort=-v:refname | head
```

</details>


### Q180. Delete a tag locally and on the remote. 🟡

<details>
<summary><b>Answer</b></summary>


```bash
git tag -d v1.4.0
git push origin --delete tag v1.4.0
```

</details>


### Q181. Show the URL of a remote and a single config value. 🟢

<details>
<summary><b>Answer</b></summary>


```bash
git remote get-url origin
git config --get user.email
```

</details>


### Q182. Reset the author of the last commit to the current config identity. 🟢

<details>
<summary><b>Answer</b></summary>


```bash
git commit --amend --reset-author --no-edit
```

</details>


### Q183. Rewrite the author on your own unpublished commits. 🟡

<details>
<summary><b>Answer</b></summary>


```bash
git rebase -r origin/main --exec 'git commit --amend --reset-author --no-edit'
```

</details>


### Q184. Ignore whitespace-only changes in a diff, and detect whitespace errors before committing. 🟢

<details>
<summary><b>Answer</b></summary>


```bash
git diff -w
git diff --check
```

</details>


### Q185. Create a patch file from your changes and apply it elsewhere, testing first. 🟢

<details>
<summary><b>Answer</b></summary>


```bash
git diff > fix.patch
git apply --check fix.patch && git apply fix.patch
```

</details>


### Q186. Find commits by message text. 🟢

<details>
<summary><b>Answer</b></summary>


```bash
git log --grep="payment" -i --oneline
```

</details>


### Q187. Find the commit that was HEAD of main on a given date. 🟡

<details>
<summary><b>Answer</b></summary>


```bash
git rev-list -1 --before="2026-10-01 00:00" origin/main
```

</details>


### Q188. Jump back to the previously checked-out branch. 🟢

<details>
<summary><b>Answer</b></summary>


```bash
git switch -
```

</details>


### Q189. Push tags that belong to your commits only, and push several refs atomically. 🟢

<details>
<summary><b>Answer</b></summary>


```bash
git push --follow-tags
git push --atomic origin main v1.4.0
```

</details>


### Q190. Automatically stash and unstash around a rebase. 🟢

<details>
<summary><b>Answer</b></summary>


```bash
git rebase --autostash origin/main
git config --global rebase.autoStash true
```

</details>


### Q191. Remove the upstream tracking relationship of a branch. 🟡

<details>
<summary><b>Answer</b></summary>


```bash
git branch --unset-upstream
```

</details>


### Q192. Create a branch with no history (for gh-pages or a clean start). 🟡

<details>
<summary><b>Answer</b></summary>


```bash
git switch --orphan gh-pages
```

</details>


### Q193. List staged file names, handy inside a pre-commit hook. 🟢

<details>
<summary><b>Answer</b></summary>


```bash
git diff --cached --name-only --diff-filter=ACM
```

</details>


### Q194. List every tracked file in a directory at a revision. 🟢

<details>
<summary><b>Answer</b></summary>


```bash
git ls-tree -r --name-only v1.4.0 -- src/
```

</details>


### Q195. Delete a ref directly with plumbing, such as a leftover PR ref in a local mirror. 🔴

<details>
<summary><b>Answer</b></summary>


```bash
git update-ref -d refs/pull/482/head
```

</details>


### Q196. Show a short SHA of HEAD for a build tag. 🟢

<details>
<summary><b>Answer</b></summary>


```bash
git rev-parse --short=8 HEAD
```

</details>


### Q197. Remove a file from Git and disk, versus Git only. 🟢

<details>
<summary><b>Answer</b></summary>


```bash
git rm old.txt
git rm --cached old.txt
```

</details>


### Q198. Make `git pull` refuse to create surprise merge commits. 🟢

<details>
<summary><b>Answer</b></summary>


```bash
git config --global pull.ff only
```

</details>


### Q199. Show the diff you are about to commit while writing the message. 🟢

<details>
<summary><b>Answer</b></summary>


```bash
git commit -v
```

</details>


### Q200. Show remote branches that are already merged into main, for cleanup. 🟡

<details>
<summary><b>Answer</b></summary>


```bash
git branch -r --merged origin/main | grep -v -E 'main|release'
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
