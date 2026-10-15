# Chapter 11: Best Practices: Performance, Maintainability and Security

> **Level:** intermediate → advanced · **Time:** 3 hours · **Prerequisites:** [Chapters 5–10](05_core_features.md)
> **Next:** [Chapter 12: Capstone, Part 1: The Test Workflow](12_test_workflow.md)

## Learning objectives

After this chapter you can:

- Measure a pipeline and decide what to optimize (wait less, do less, use resources better).
- Decide what to cache and what not to.
- Keep many repositories or services consistent with a standard task interface, composite actions and reusable workflows.
- Weigh monorepo against multirepo for CI.
- Apply a security baseline: least privilege, no long-lived credentials, SHA pinning, allow-lists, protected environments, safe handling of pull requests from forks and of untrusted input.
- Audit a workflow with a checklist.

## 1. Problem statement

A workflow that "works" can still be slow (developers stop waiting for it), brittle (nobody dares to change it), or dangerous (a pull request can steal your deploy keys). This chapter turns experience into rules you can check.

## 2. Core explanation

### Part A: Performance

#### Lesson 11.1: Measure first

Optimize the slowest critical path, not what you feel is slow. Open recent runs and look at the **critical path**: the longest chain of dependent jobs, since parallel jobs don't add up. Then ask, for each long step: *can I wait less, do less, or use resources better?* Keep a baseline (median duration of the last 20 runs; see the `gh run list` snippet in Chapter 10) so you can tell whether a change helped.

#### Lesson 11.2: Wait less

Time is lost in queues and in sequential steps.

| Technique | Notes |
|---|---|
| **Parallelize independent jobs** | Lint, unit tests, and type checks usually don't `need` each other |
| **Split slow test suites** across matrix legs | Use your test runner's sharding |
| **Cancel superseded runs** | `concurrency` with `cancel-in-progress` for CI |
| **Faster runners** | Larger or vendor-provided runners cost more per minute but can finish sooner; compare cost per run, not per minute |
| **Run only what's affected** | Path filters (below) |

#### Lesson 11.3: Do less work

- **Path and change filters.** Workflow-level `paths:` skips an entire workflow when only unrelated files changed. Be careful with required checks: a workflow skipped by `paths:` never reports a status, so a *required* check that is skipped by path filters can leave pull requests "pending" forever. A common fix is a lightweight always-running job that decides what to run (Chapter 12's pattern) and a final "all-done" gate job that is the only required check.
- **Job-level conditions.** An `if:` on a job (for example `if: ${{ !contains(github.event.head_commit.message, '[skip ci]') }}`) is **not** a path filter; it still starts a workflow run and only skips jobs. (GitHub also natively honors `[skip ci]`-style phrases in commit messages for `push` and `pull_request` events; see the docs.)
- **Cache** dependencies and tool downloads (Lesson 11.5).
- **Don't rebuild**: build once, pass the artifact or image to later jobs (Chapter 2's "build once, promote").
- **Shallow clones.** `actions/checkout` fetches one commit by default (`fetch-depth: 1`), which is fastest. Use `fetch-depth: 0` only when you need history (changelogs, `git describe`).

#### Lesson 11.4: Use resources better

- Match the runner to the workload: a CPU-bound compile benefits from more cores; a lint job doesn't.
- Use **Arm64 runners** (`ubuntu-24.04-arm`) to build Arm images natively instead of emulating them with QEMU, which is much slower (Chapter 13).
- Remember GitHub-hosted jobs start on fresh machines: tools that are pre-installed on the runner image don't need installing; check the [runner image software lists](https://github.com/actions/runner-images).
- Self-hosted runners can keep warm caches, at the price of operations and security work.
- Set `timeout-minutes` on every job so a hang doesn't burn a six-hour default.

#### Lesson 11.5: What to cache

| Layer | Cache? | How |
|---|---|---|
| Git checkout | Usually not: shallow clone is fast; large repos may benefit from sparse checkout | `fetch-depth`, `sparse-checkout` |
| Language toolchain | Often pre-installed; if downloaded, cache via setup action | `actions/setup-*` with `cache:` |
| Dependency downloads | **Yes**, keyed on the lock file | `actions/cache@v4`, or setup-action `cache:` |
| Build/test intermediates | Sometimes (incremental compilers, Gradle/Bazel caches) | Key on source + lock hash; verify correctness |
| Container layers | **Yes** with Buildx: `cache-from`/`cache-to` (`type=gha` or a registry cache) | Chapter 13 |

Caching trade-offs: the cache has size limits and eviction (Chapter 6); restoring a very large cache can take longer than rebuilding; a bad key serves stale data; and shared caches can be poisoned. Measure restore time against rebuild time.

### Part B: Maintainability

#### Lesson 11.6: A standard CI/CD interface

If every service builds differently, every workflow differs. Define a **standard set of task names** (`install`, `lint`, `test`, `build`, `dev`) implemented per service in its `Taskfile.yml` (or `Makefile`, or `package.json` scripts), and have workflows call only those names. Developers get the same commands locally; workflow files become near-identical and thus reusable.

```yaml
steps:
  - uses: actions/checkout@v5
  - uses: ./.github/actions/setup          # composite action: toolchain + task
  - run: task test
```

#### Lesson 11.7: Share via composite actions and reusable workflows

Use Chapter 8's rules: composite action for a few steps; reusable workflow for whole stages. Version the shared repository with tags and have callers pin to them. Avoid piping install scripts straight into a shell (`curl … | sh`) in shared actions; use a pinned setup action (for example `go-task/setup-task@v1`) so the supply chain is explicit.

#### Lesson 11.8: Monorepo or multirepo

| | Monorepo | Multirepo |
|---|---|---|
| Workflow sprawl | One set of workflows covers many services; needs change detection | Each repo has its own, duplicated workflows (reduce with reusable workflows) |
| Atomic cross-service change | Easy: one PR | Needs coordinated PRs and versioning |
| CI cost | Risk of running everything for every change → needs path filtering and caching | Naturally scoped |
| Permissions | Coarse: everyone with write access can change every service's pipeline | Fine-grained per repo |
| Required checks | Tricky with path filters | Simple |
| Tooling | Often needs build-graph tooling at scale | Dependency versioning between repos |

Neither is right in general. If you choose a monorepo, invest in change detection and a single "gate" job. If you choose multirepo, invest in reusable workflows and Dependabot-based version propagation.

### Part C: Security

#### Lesson 11.9: Least privilege

Set `permissions: contents: read` at workflow level and elevate per job (Chapter 6). Also set the organization/repository default **Workflow permissions** to read-only (**Settings → Actions → General**), so that a workflow without a `permissions:` block starts with minimal rights. Review the defaults of older repositories: older repositories and organizations may default to read-write.

#### Lesson 11.10: No long-lived credentials

Use OIDC for AWS, Google Cloud, Azure, Vault and others (Chapter 6). Where a static secret is unavoidable, scope it to one environment, make it narrow (read-only, one resource), and rotate it. Prefer GitHub App tokens to personal access tokens for automation that must push or trigger workflows.

#### Lesson 11.11: Control which actions can run, and pin them

- In **Settings → Actions → General** you can allow only actions from your organization, GitHub, verified creators, or an explicit allow-list, and (where available) require actions to be pinned to a full commit SHA.
- Pin third-party actions to SHAs and update with Dependabot (Chapter 7).
- Review changes to `.github/workflows/**` like code that runs with production access: use **CODEOWNERS** so the security or platform team must approve.

#### Lesson 11.12: Pull requests and untrusted code

| Trigger | Runs with | Risk |
|---|---|---|
| `pull_request` | Workflow from the merge commit; for fork PRs a **read-only token and no secrets** | Low: safe for tests |
| `pull_request_target` | Workflow from the **base** branch with a normal token and secrets | **High** if it checks out and runs the PR's code. Use only for metadata tasks (labeling), never to build or run PR code |
| `workflow_run` | Triggered after another workflow; has secrets | Same pitfall: do not execute artifacts or checkouts from untrusted PRs |
| `issue_comment`, `issues` | Base-branch context with secrets | Treat comment text as untrusted input |

Configure **Settings → Actions → General → Fork pull request workflows** to require approval for first-time or all outside contributors before their workflows run. Never "fix" a failing fork-PR workflow by switching to `pull_request_target` and checking out the PR head.

#### Lesson 11.13: Script injection and untrusted input

Treat `github.event.*` text fields (PR titles, issue bodies, branch names, commit messages), `github.head_ref`, workflow inputs, and artifacts from other runs as attacker-controlled. Never place `${{ … }}` of such values inside `run:` scripts or `github-script` code; pass them through `env:` (Chapter 5). Run `actionlint`, which flags many of these patterns.

#### Lesson 11.14: Protect environments and self-hosted runners

- Put deployment secrets in **environments** with required reviewers, deployment branch restrictions and (optionally) wait timers. A job gets the environment's secrets only after approval.
- Self-hosted runners: do not attach to public repositories; use ephemeral runners; isolate the network; don't reuse a runner between trust levels.

#### Lesson 11.15: Audit checklist

See Section 10. Use it in code review of any workflow change.

## 3. Mental model

Think of three dials: **speed** (wait less, do less, use resources better), **clarity** (a standard interface and shared building blocks), and **blast radius** (what a mistake or a compromised dependency can reach). Good pipelines are fast because they're simple, simple because they are standardized, and safe because every credential and permission is narrowly scoped.

## 4. Syntax breakdown

A workflow that applies many of the rules:

| Rule | Syntax |
|---|---|
| Least privilege | `permissions: { contents: read }` |
| Cancel superseded | `concurrency: { group: …, cancel-in-progress: true }` |
| Bound runtime | `timeout-minutes: 15` |
| Pin action | `uses: owner/repo@<40-char SHA>  # vX.Y.Z` |
| Untrusted input | `env: TITLE: ${{ github.event.pull_request.title }}` then `"$TITLE"` |
| Gate job | `needs: [...]`, `if: always()`, check results |
| Production gate | `environment: production` |

## 5. Practical example: a gate job and safe input handling

```yaml
name: CI

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
  test:
    runs-on: ubuntu-24.04
    timeout-minutes: 15
    steps:
      - uses: actions/checkout@v5
        with:
          persist-credentials: false
      - uses: go-task/setup-task@v1
      - run: task test

  label-title:
    runs-on: ubuntu-24.04
    timeout-minutes: 5
    if: github.event_name == 'pull_request'
    steps:
      - name: Show PR title safely
        env:
          TITLE: ${{ github.event.pull_request.title }}    # untrusted: via env, never inline
        run: |
          printf 'Title: %s\n' "$TITLE"

  # The ONE check to mark as required in branch protection
  ci-ok:
    if: always()
    needs: [test, label-title]
    runs-on: ubuntu-24.04
    timeout-minutes: 5
    steps:
      - name: Fail if any needed job failed or was cancelled
        env:
          RESULTS: ${{ toJSON(needs.*.result) }}
        run: |
          echo "results: $RESULTS"
          if echo "$RESULTS" | grep -Eq '"(failure|cancelled)"'; then exit 1; fi
```

## 6. Execution walkthrough

1. A pull request starts the workflow. `test` and `label-title` run in parallel.
2. `label-title` receives the title as data in an environment variable; a malicious title such as `"; rm -rf ~; "` is printed literally, not executed.
3. `ci-ok` waits for both, runs even if one failed (`always()`), collects `needs.*.result` and exits non-zero if any job failed or was cancelled. Jobs that were *skipped* are accepted, which keeps the gate green when parts of the pipeline are legitimately skipped.
4. Branch protection requires only `ci-ok`, so adding or renaming jobs doesn't require editing the protection rule, and skipped jobs can't block merges.

## 7. Common mistakes

| Mistake | Effect | Fix |
|---|---|---|
| Making path-filtered jobs "required checks" | PRs stuck "waiting" | A single always-running gate job |
| Calling a job-level `if` on the commit message a "path filter" | Misunderstanding of cost: the workflow still starts | Use `paths:` for file-based filtering |
| Caching without measuring | Restores slower than rebuilds | Time both |
| `pull_request_target` + checkout of PR head | Secrets exposed to attacker code | Use `pull_request`, or split into untrusted build and trusted comment workflows with care |
| Required reviewers missing on production | Anyone with write access can deploy | Environments with reviewers |
| Interpolating `${{ github.head_ref }}` in `run:` | Injection | `env:` |
| Long-lived cloud keys in repository secrets | Large blast radius | OIDC |
| Org default token permission = read-write | Forgotten `permissions:` is dangerous | Set default to read-only |
| `curl … \| sh` in shared actions | Unpinned supply chain | Pinned setup action |

## 8. Security considerations (summary of this chapter's rules)

1. Default `GITHUB_TOKEN` to read-only; grant scopes per job.
2. OIDC over static keys; environment-scoped secrets.
3. Pin third-party actions to SHAs; restrict the allowed set; automate updates.
4. Treat PRs from forks, comments, issue text, branch names, inputs and artifacts as hostile.
5. Never run untrusted code in a privileged trigger.
6. Protect workflow files with CODEOWNERS and branch protection.
7. Use environments with reviewers for production.
8. Keep self-hosted runners private, ephemeral and isolated.
9. Set `timeout-minutes`, `concurrency` and logging hygiene.

## 9. Hands-on exercise

**Goal:** harden and speed up a workflow, then prove the gate works.

1. Take your `ci.yml` from Chapter 9 on a new branch.
2. Add `timeout-minutes`, `concurrency`, `persist-credentials: false`, and the `ci-ok` gate job above.
3. Open **Settings → Actions → General**: set **Workflow permissions** to *Read repository contents* (read-only), and review **Fork pull request workflows**. In **Settings → Branches/Rulesets**, require only `ci-ok` on `main`.
4. Prove the injection defense: create a PR titled `x"; echo INJECTED; "` and confirm the log prints it literally. Then temporarily change the workflow to inline the title in the script (`echo "${{ github.event.pull_request.title }}"`), push, observe `INJECTED` executing, and revert. (Do this only in your practice repository.)
5. Time the pipeline before and after enabling caching or concurrency changes. Write down the numbers.
6. Add a `CODEOWNERS` entry for `.github/workflows/` (for example `/.github/workflows/ @your-username`).

**Success looks like:** PRs show one required check; the injection demo works only in the unsafe version; baseline and new durations are recorded.

**If it fails:** the required check is "Expected, waiting" → its name doesn't match the job name; branch protection options differ between classic rules and rulesets.

## 10. Verification checklist (use in reviews)

**Security**
- [ ] `permissions` declared; minimal scopes
- [ ] No secrets in `if`, artifacts, caches or logs
- [ ] Third-party actions pinned; allowed-actions policy set
- [ ] No `${{ … }}` of untrusted values inside scripts
- [ ] No `pull_request_target`/`workflow_run` running untrusted code
- [ ] OIDC instead of static cloud keys; production uses an environment with reviewers

**Performance**
- [ ] Baseline measured; critical path identified
- [ ] Independent jobs run in parallel; superseded runs cancelled
- [ ] Dependencies and image layers cached where measured to help
- [ ] `timeout-minutes` on every job

**Maintainability**
- [ ] Standard task names; logic in scripts/Taskfile
- [ ] Shared steps/jobs in composite actions or reusable workflows with versioned references
- [ ] A single required gate job

## 11. Knowledge check

1. Why can a path-filtered required check block merges forever, and what is the fix?
2. What is the difference between `pull_request` and `pull_request_target` for a fork PR?
3. Why might caching make a build slower?
4. Why is OIDC preferable to a stored access key?
5. Give two reasons to put a standard task interface in front of every service.
6. What does `fetch-depth: 0` cost and when is it needed?

<details>
<summary>Answers</summary>

1. If the workflow is skipped, the required check never reports; use an always-running gate job.
2. `pull_request`: fork code, read-only token, no secrets. `pull_request_target`: base-branch workflow with secrets; dangerous if it runs PR code.
3. Restoring a large cache can cost more than rebuilding; a bad key serves stale data.
4. Short-lived, claim-scoped credentials; nothing long-lived to leak.
5. Identical commands locally and in CI; reusable, near-identical workflows.
6. Downloads the full history (slower); needed for changelogs, `git describe`, or version computation.
</details>

## 12. Key takeaways

- Measure; then wait less, do less, use resources better.
- Standardize the interface; share via composite actions and reusable workflows.
- One gate job simplifies required checks.
- Security baseline: least privilege, OIDC, pinned actions, hostile-input mindset, protected environments.
- Treat workflow files as production code: review them.

## Sources and further reading

- Original notes: `git show master:11_best_practices.md`. Corrections: invalid `ubuntu-latest-arm` label, a job-level `if` on commit messages mislabeled as a path filter, wording of the fork-PR setting, `curl | sh` in the sample composite action, and a duplicated decision flowchart (see Chapter 8). The reference to "Chapters 12–19" was removed because this guide has Chapters 12–14.
- Official docs: [Workflow syntax](https://docs.github.com/en/actions/reference/workflows-and-actions/workflow-syntax) · [Dependency caching](https://docs.github.com/en/actions/reference/workflows-and-actions/dependency-caching) · [Authenticate with GITHUB_TOKEN](https://docs.github.com/en/actions/tutorials/authenticate-with-github_token) · [Contexts](https://docs.github.com/en/actions/reference/workflows-and-actions/contexts) · [Runner images](https://github.com/actions/runner-images)
- Not verified here: exact current wording and location of repository settings (fork PR approval, SHA-pinning enforcement), and the `[skip ci]` handling details; confirm in the current docs and your settings UI.
