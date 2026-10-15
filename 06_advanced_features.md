# Chapter 6: Advanced Features: Runners, Artifacts, Caching, Permissions, OIDC, Matrix and Concurrency

> **Level:** intermediate · **Time:** 3–4 hours · **Prerequisites:** [Chapter 5](05_core_features.md)
> **Next:** [Chapter 7: Marketplace Actions](07_marketplace_actions.md)

Chapter 5 gave you the language. This chapter gives you the features that make real pipelines fast, safe and flexible. Each lesson is independent; the lab at the end combines them.

## Learning objectives

After this chapter you can:

- Choose a runner (GitHub-hosted, larger/third-party hosted, self-hosted) and a container for a job.
- Pass files between jobs and keep build outputs with **artifacts**.
- Speed up builds with **caching**, and say how a cache differs from an artifact.
- Limit the `GITHUB_TOKEN` with `permissions:`.
- Authenticate to a cloud provider without stored keys, using **OIDC**.
- Fan a job out across configurations with a **matrix**.
- Use **concurrency** groups to cancel or serialize runs.

## 1. Problem statement

Simple workflows run one job on one machine with default settings. Real projects need to: test on several OS/language versions, avoid reinstalling dependencies on every run, hand a built file to a later job, deploy to a cloud without storing a long-lived password, stop two deployments from colliding, and give each workflow only the power it needs. These are the features that do that.

## 2. Core explanation

### Lesson 6.1: Runners

| Kind | Who runs it | Use when |
|---|---|---|
| **GitHub-hosted (standard)** | GitHub; fresh VM per job | Default. Zero maintenance. |
| **Larger / third-party hosted** | GitHub (larger runners, paid) or a vendor selected by label | You need more CPU/RAM, GPUs, faster disks, or cheaper minutes |
| **Self-hosted** | You | Special hardware, private network access, large caches, or compliance needs |

#### GitHub-hosted labels

Pick an explicit image rather than relying on `-latest` when reproducibility matters. As of this writing, labels documented for standard hosted runners include:

| OS | Labels |
|---|---|
| Linux (x64) | `ubuntu-24.04`, `ubuntu-22.04`, `ubuntu-latest` (an alias that GitHub repoints over time) |
| Linux (Arm64) | `ubuntu-24.04-arm` |
| Windows | `windows-2025`, `windows-2022`, `windows-latest` |
| Windows (Arm64) | `windows-11-arm` |
| macOS | `macos-15`, `macos-latest` (and others listed in the docs) |

Labels are retired and added regularly. Names such as `ubuntu-20.04` and `macos-12` have been removed, and `ubuntu-latest-arm` is **not** a valid label (use `ubuntu-24.04-arm`). **Always check the [runner images list](https://github.com/actions/runner-images) before using a label.** Hardware specifications, included minutes, per-minute prices and retention periods differ by plan, repository visibility and date, so look them up in GitHub's current billing documentation rather than copying numbers from notes.

#### Self-hosted runners

You install the runner agent on your own machine and register it with a label, then target it with `runs-on: [self-hosted, linux, x64]` (or your own labels). Benefits: access to private networks, custom hardware, persistent caches. Costs: you patch it, secure it, and clean it. **Do not use self-hosted runners on public repositories** unless you fully understand the risk: a pull request from a fork can run arbitrary code on your machine. Prefer ephemeral (one-job) runners and isolated networks.

#### Running a job inside a container

```yaml
jobs:
  test:
    runs-on: ubuntu-24.04
    container:
      image: node:22-bookworm
    steps:
      - uses: actions/checkout@v5
      - run: node --version && npm ci && npm test
```

The steps run inside the container on the runner. `services:` starts sidecar containers (for example PostgreSQL) reachable by their service name when the job itself runs in a container, or on `localhost` with mapped ports when it does not.

**Rule:** choose `runs-on` deliberately; use containers when you need an exact toolchain; keep self-hosted runners away from untrusted code.

### Lesson 6.2: Artifacts: files that outlive a job

An **artifact** is a file or folder you upload from a run so that (a) later jobs in the same run can download it, and (b) people can download it from the run page.

```yaml
jobs:
  build:
    runs-on: ubuntu-24.04
    steps:
      - uses: actions/checkout@v5
      - run: mkdir -p dist && date > dist/build-info.txt
      - uses: actions/upload-artifact@v4
        with:
          name: site
          path: dist/
          retention-days: 7        # optional; shorter is cheaper and safer
          if-no-files-found: error

  use:
    needs: build
    runs-on: ubuntu-24.04
    steps:
      - uses: actions/download-artifact@v4
        with:
          name: site
          path: dist/
      - run: cat dist/build-info.txt
```

Important facts:

- Use **v4** (or newer) of both actions. Version 3 stopped working in January 2025 after GitHub deprecated it ([deprecation notice](https://github.blog/changelog/2024-04-16-deprecation-notice-v3-of-the-artifact-actions/)); workflows still pinned to `@v3` fail.
- In v4, an artifact **name must be unique within a run**. In a matrix, include the matrix value in the name (`name: build-${{ matrix.os }}`), or upload from the matrix jobs and merge afterwards.
- Artifacts are zipped; permissions of files may be lost. Do not upload secrets or `.env` files. Anyone who can read the run can download them; for public repositories that means everyone.
- `if: always()` on the upload step keeps logs and test reports from failed runs.

### Lesson 6.3: Caching: reuse work between runs

A **cache** stores files (typically dependencies) under a **key** and restores them on a later run if the key matches. Unlike artifacts, a cache is a speed optimization: a build must still work when the cache is empty.

| | Artifact | Cache |
|---|---|---|
| Purpose | Deliver outputs (binaries, reports) | Avoid recomputing inputs (dependencies) |
| Scope | One run (and downloadable by humans) | Across runs, per repository and branch rules |
| If missing | The workflow is broken | The workflow is just slower |
| Typical content | `dist/`, test reports | `~/.npm`, `~/.cache/pip`, Go module cache |

The easy path is to let the setup action manage it:

```yaml
- uses: actions/setup-node@v4
  with:
    node-version: 22
    cache: npm            # keyed on package-lock.json automatically
```

The explicit form uses `actions/cache@v4`:

```yaml
- uses: actions/cache@v4
  with:
    path: ~/.npm
    key: npm-${{ runner.os }}-${{ hashFiles('**/package-lock.json') }}
    restore-keys: |
      npm-${{ runner.os }}-
```

**Keys.** The key should change exactly when the cached content should be rebuilt: include the OS and a hash of the lock file. An exact `key` hit restores the cache; otherwise the first matching `restore-keys` prefix restores a *near* cache, and a new cache is saved at the end of a successful job under the exact key. Caches are immutable: a given key is written once.

**Limits and behavior** (check the [dependency caching docs](https://docs.github.com/en/actions/reference/workflows-and-actions/dependency-caching) for current values): there is a per-repository storage limit (10 GB by default, which owners can raise with billing above that), and entries not accessed for 7 days are removed. When the limit is exceeded, the oldest entries are evicted. Caches are also scoped by branch: a feature branch can read the default branch's caches, but not vice versa.

**Security.** Never put secrets in a cache. Because caches can be restored in later runs, a poisoned cache can affect later builds; treat caches from untrusted pull requests with care (see Chapter 11).

### Lesson 6.4: Permissions: least privilege for `GITHUB_TOKEN`

Every run gets a temporary token, `GITHUB_TOKEN`, that actions and `gh` use to call the GitHub API. Its power is set by the `permissions:` key. If you omit the key, the token receives the **repository's or organization's default** (permissive or read-only, depending on a setting that varies by account and age of the repository). Do not depend on the default: declare what you need.

```yaml
permissions:            # workflow level: applies to all jobs
  contents: read

jobs:
  comment:
    runs-on: ubuntu-24.04
    permissions:        # job level: overrides the workflow level for this job
      contents: read
      pull-requests: write
    steps:
      - env:
          GH_TOKEN: ${{ github.token }}
          PR: ${{ github.event.pull_request.number }}
        run: gh pr comment "$PR" --repo "$GITHUB_REPOSITORY" --body "Thanks for the PR!"
```

| Scope (selected) | `read` allows | `write` allows |
|---|---|---|
| `contents` | Checkout, read repository | Push commits/tags, create releases |
| `pull-requests` | Read PRs | Comment, label, review |
| `issues` | Read issues | Comment, label, close |
| `packages` | Pull from GitHub Packages / GHCR | Push images and packages |
| `id-token` | n/a | Request an OIDC token (next lesson) |
| `actions`, `checks`, `deployments`, `statuses`, `security-events`, ... | See the [token docs](https://docs.github.com/en/actions/tutorials/authenticate-with-github_token) | |

Values are `read`, `write` or `none`. Setting any scope in `permissions:` sets every **unlisted** scope to `none`. `permissions: {}` removes all permissions.

Also note: events from forked pull requests get a read-only token regardless of what you request, and commits or tags pushed *using* `GITHUB_TOKEN` do not start new workflow runs (a deliberate guard against loops). If you need the push to trigger a workflow, use a GitHub App token or a deploy key, deliberately.

**Rule:** start every workflow with `permissions: contents: read`, then grant extra scopes per job.

### Lesson 6.5: Authenticating to cloud systems: secrets versus OIDC

Two ways for a workflow to prove itself to AWS, Google Cloud, Azure, Vault and others:

| | Stored credentials | OIDC (OpenID Connect) |
|---|---|---|
| How | You create an access key and store it as a secret | GitHub issues a short-lived signed token per job; the cloud exchanges it for temporary credentials |
| Lifetime | Until you rotate it | Minutes |
| Leak impact | Anyone with the key can use it anywhere | The token is useless outside the run, and the cloud trust policy limits who can mint it |
| Setup | Easy | One-time trust configuration on the cloud side |

With OIDC, the cloud provider is configured to trust tokens issued by GitHub **for specific claims**: which repository, branch, environment, or workflow. The job must have `id-token: write`.

```yaml
permissions:
  id-token: write      # allow requesting the OIDC token
  contents: read

jobs:
  deploy:
    runs-on: ubuntu-24.04
    environment: production
    steps:
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789012:role/github-deploy   # example ARN
          aws-region: us-east-1
      - run: aws sts get-caller-identity
```

The AWS-side trust policy must restrict the token's `sub` claim (for example to `repo:YOUR-ORG/YOUR-REPO:environment:production`). **A trust policy that allows any repository is a serious vulnerability.** The same pattern exists for GCP (`google-github-actions/auth`) and Azure (`azure/login`). Follow your provider's current OIDC guide for setup, and verify the exact `sub` format in GitHub's [OIDC reference](https://docs.github.com/en/actions/concepts/security/openid-connect).

### Lesson 6.6: Matrix strategies

A **matrix** creates one job per combination of values.

```yaml
jobs:
  test:
    runs-on: ${{ matrix.os }}
    strategy:
      fail-fast: false            # let other legs finish if one fails
      max-parallel: 4             # optional throttle
      matrix:
        os: [ubuntu-24.04, windows-2022]
        node: [20, 22]
        exclude:
          - os: windows-2022
            node: 20
        include:
          - os: ubuntu-24.04
            node: 24
            experimental: true
    continue-on-error: ${{ matrix.experimental == true }}
    steps:
      - uses: actions/checkout@v5
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node }}
      - run: npm ci && npm test
```

Combinations here: 2×2 = 4, minus the excluded one = 3, plus the included extra = 4 jobs. `exclude` removes combinations from the generated set; `include` adds new legs (or extra properties to existing ones). Defaults: `fail-fast: true` cancels the remaining legs as soon as one fails, which is often not what you want for compatibility testing. A matrix is limited to 256 jobs per workflow run.

**Dynamic matrix.** A matrix can be built from JSON produced by an earlier job, which is the key to the monorepo capstone:

```yaml
jobs:
  plan:
    runs-on: ubuntu-24.04
    outputs:
      services: ${{ steps.p.outputs.services }}
    steps:
      - id: p
        run: echo 'services=["api","web"]' >> "$GITHUB_OUTPUT"
  build:
    needs: plan
    if: ${{ needs.plan.outputs.services != '[]' }}     # an empty matrix is an error
    runs-on: ubuntu-24.04
    strategy:
      matrix:
        service: ${{ fromJSON(needs.plan.outputs.services) }}
    steps:
      - run: echo "building ${{ matrix.service }}"
```

An empty array makes the workflow fail ("Matrix vector does not contain any values"), so guard it with `if:`.

### Lesson 6.7: Conditionals recap

`if:` works on steps and jobs (Lesson 5.8). Remember the contexts that exist in each place, and that `needs`-dependent jobs are skipped when a dependency fails unless you add `if: ${{ !cancelled() }}` or `always()`. Typical uses:

```yaml
deploy:
  needs: [test]
  if: github.ref == 'refs/heads/main' && github.event_name == 'push'
```

### Lesson 6.8: Concurrency

`concurrency` makes runs that share a **group** string take turns.

```yaml
concurrency:
  group: ci-${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true         # newer run cancels the older one
```

| Pattern | Config | Why |
|---|---|---|
| Cancel superseded CI | `cancel-in-progress: true`, group per branch/PR | Don't waste minutes on outdated commits |
| Serialize deployments | `group: deploy-production`, `cancel-in-progress: false` | Never overlap two deploys; later runs wait (only one run waits; extra pending runs are replaced) |
| Per-environment | `group: deploy-${{ inputs.environment }}` | Staging and production don't block each other |

Do **not** cancel a deployment mid-flight unless you know it is safe. Concurrency can be set at workflow or job level.

## 3. Mental model

- **Runner** = which kitchen. **Container** = which toolbox on the bench.
- **Artifact** = a sealed parcel mailed to the next cook (or the customer). **Cache** = a pantry shelf of ingredients: nice to have, rebuildable.
- **Permissions** = the key-ring you hand the cook; **OIDC** = a one-day visitor badge instead of a master key.
- **Matrix** = photocopying the recipe for each oven type. **Concurrency** = a single-lane bridge sign.

## 4. Syntax breakdown

| Key | Where | Meaning |
|---|---|---|
| `runs-on` | job | Runner label(s) |
| `container` / `services` | job | Job container / sidecar containers |
| `permissions` | workflow or job | Token scopes |
| `strategy.matrix`, `fail-fast`, `max-parallel` | job | Fan-out controls |
| `concurrency.group`, `cancel-in-progress` | workflow or job | Serialization |
| `uses: actions/upload-artifact@v4` / `download-artifact@v4` | step | Files between jobs |
| `uses: actions/cache@v4` | step | Cache restore/save |
| `id-token: write` | `permissions` | Allows OIDC token request |

## 5. Practical example

`.github/workflows/advanced-lab.yml`:

```yaml
name: Advanced Lab

on:
  pull_request:
  workflow_dispatch:

permissions:
  contents: read

concurrency:
  group: advanced-lab-${{ github.ref }}
  cancel-in-progress: true

jobs:
  build:
    runs-on: ubuntu-24.04
    timeout-minutes: 10
    steps:
      - uses: actions/checkout@v5
      - uses: actions/cache@v4
        with:
          path: .cache-demo
          key: demo-${{ runner.os }}-${{ hashFiles('.github/workflows/advanced-lab.yml') }}
      - name: Build (slow only on a cache miss)
        run: |
          if [ ! -f .cache-demo/ready ]; then
            echo "cache miss: doing slow work"
            sleep 15
            mkdir -p .cache-demo && touch .cache-demo/ready
          else
            echo "cache hit: skipping slow work"
          fi
          mkdir -p dist && echo "built at $(date -u)" > dist/info.txt
      - uses: actions/upload-artifact@v4
        with:
          name: dist
          path: dist/

  test:
    needs: build
    runs-on: ${{ matrix.os }}
    timeout-minutes: 10
    strategy:
      fail-fast: false
      matrix:
        os: [ubuntu-24.04, ubuntu-22.04]
    steps:
      - uses: actions/download-artifact@v4
        with:
          name: dist
          path: dist/
      - run: cat dist/info.txt && echo "tested on ${{ matrix.os }}"
```

## 6. Execution walkthrough

First run on a branch:

1. Starting the run cancels any earlier in-progress run of the same group (`concurrency`).
2. `build`: the cache step finds no entry for the key (miss). The slow branch executes (about 15 s), creates `.cache-demo/ready`, writes `dist/info.txt`, and uploads artifact `dist`. At the end of the job a new cache entry is saved under the exact key.
3. `test` starts only after `build` succeeds. The matrix creates two jobs (`ubuntu-24.04`, `ubuntu-22.04`) on separate runners; each downloads `dist` and prints the file.

Second run with unchanged workflow file: the cache **hits**, the slow branch is skipped, and the post-job cache save is skipped because an entry already exists. Change the workflow file (the hash changes) and you get a miss again.

## 7. Common mistakes

| Mistake | Effect | Fix |
|---|---|---|
| `upload-artifact@v3` / `download-artifact@v3` | Fails since the January 2025 shutdown | Use `@v4` or newer |
| Same artifact name from several matrix legs | v4 rejects duplicates | Add `${{ matrix.* }}` to the name |
| Key without a lock-file hash | Stale dependencies forever | Hash the lock file |
| Treating the cache as guaranteed | Flaky builds when evicted | Make the build correct without it |
| Omitting `permissions:` | Token power depends on repo defaults | Declare it |
| Listing one scope and expecting others to stay | Unlisted scopes become `none` | List every scope the job needs |
| `id-token: write` with a wide cloud trust policy | Any repo can assume your role | Restrict `sub` to repo + branch/environment |
| `runs-on: ubuntu-latest-arm` | No such label | `ubuntu-24.04-arm` |
| `cancel-in-progress: true` on a production deploy | A half-applied deploy | Serialize instead |
| Matrix from an empty JSON list | Workflow error | Guard with `if: needs.x.outputs.y != '[]'` |
| Using `secrets.X != ''` in an `if` | Invalid | Use an `env` flag (Chapter 5) |

## 8. Security considerations

- Self-hosted runners and public repositories do not mix.
- Artifacts and caches are readable data. Never store credentials in them.
- Prefer OIDC to stored cloud keys; scope trust to repository + environment/branch; grant `id-token: write` only to the job that needs it.
- Default to `contents: read`; use job-level elevation. Review any third-party action that receives a write-capable token.
- Treat matrix values and `inputs` like any other interpolated text (Chapter 5).

## 9. Hands-on exercise

**Goal:** see cache hit/miss, artifact hand-off, matrix and concurrency.

1. On a new branch, save the lab workflow above and run `actionlint`.
2. Push and open a pull request. Note build duration and the "cache miss" line.
3. Re-run the workflow (**Re-run all jobs**). Confirm "cache hit" and a shorter build.
4. Edit the workflow (add a comment) and push twice quickly. Confirm the first run is cancelled by concurrency.
5. Open the run summary and download the `dist` artifact.
6. Remove `permissions:` from the workflow temporarily and, in the repository **Settings → Actions → General → Workflow permissions**, see which default is selected; restore the explicit block.

**What success looks like:** two matrix jobs, an artifact named `dist`, a miss then a hit, one cancelled run.

**If it fails:** "Artifact name conflict" → name collision; "cache not restored" → key changed; matrix jobs skipped → the `build` job failed.

## 10. Verification checklist

- [ ] No `@v3` artifact actions.
- [ ] `permissions` is declared at the top.
- [ ] You can show a cache miss and a hit in the logs.
- [ ] The artifact downloads and contains `info.txt`.
- [ ] Concurrency cancelled the older run.
- [ ] No secret is stored in cache or artifact.

## 11. Knowledge check

1. When would you use an artifact instead of a cache?
2. Why include `hashFiles('**/package-lock.json')` in a cache key?
3. A job sets `permissions: pull-requests: write` and nothing else. Can it check out a private repository?
4. Why is OIDC safer than storing an access key?
5. A matrix has `os: [a, b]`, `node: [20, 22]`, and `exclude` of one combination. How many jobs?
6. When should `cancel-in-progress` be `false`?
7. Why does a push made with `GITHUB_TOKEN` not trigger another workflow?

<details>
<summary>Answers</summary>

1. When the file is an *output* that must be delivered or inspected; caches are only optimizations.
2. So the cache is invalidated whenever dependencies change.
3. No. Unlisted scopes (including `contents`) are `none`, so checkout of a private repo fails.
4. The credentials are short-lived and tied to claims about the run, so a leak is much less useful.
5. 4 − 1 = 3.
6. For deployments and other operations that must not be interrupted.
7. GitHub blocks it to prevent recursive workflow loops.
</details>

## 12. Key takeaways

- Choose runners explicitly; verify labels; keep self-hosted runners away from untrusted code.
- Artifacts deliver files; caches speed things up; never confuse the two.
- Declare `permissions` everywhere; start at `contents: read`.
- OIDC beats stored keys, but only with a tight trust policy.
- Matrix fans out work; guard dynamic matrices against empty lists.
- Concurrency prevents waste and collisions; do not cancel deploys.

## Sources and further reading

- Original notes: `git show master:06_advanced_features.md`. Corrections: outdated `@v3` artifact/cache actions, outdated runner labels (`ubuntu-20.04`, `macos-12/13`), invalid `ubuntu-latest-arm`, `secrets` in `if`, and the claim about default token permissions (now stated as setting-dependent). Unverified pricing, hardware and retention figures were removed.
- Official docs: [Workflow syntax](https://docs.github.com/en/actions/reference/workflows-and-actions/workflow-syntax) · [Dependency caching](https://docs.github.com/en/actions/reference/workflows-and-actions/dependency-caching) · [Authenticate with GITHUB_TOKEN](https://docs.github.com/en/actions/tutorials/authenticate-with-github_token) · [OIDC](https://docs.github.com/en/actions/concepts/security/openid-connect) · [Runner images](https://github.com/actions/runner-images) · [Artifact v3 deprecation](https://github.blog/changelog/2024-04-16-deprecation-notice-v3-of-the-artifact-actions/)
