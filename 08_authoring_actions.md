# Chapter 8: Authoring Reusable Logic: Composite Actions, Reusable Workflows, JavaScript and Container Actions

> **Level:** intermediate → advanced · **Time:** 4 hours · **Prerequisites:** [Chapters 5–7](05_core_features.md)
> **Next:** [Chapter 9: Common Workflows](09_common_workflows.md)

## Learning objectives

After this chapter you can:

- Build a **composite action** and use it from a workflow.
- Build a **reusable workflow** with inputs, secrets and outputs, and call it.
- Describe what JavaScript and container actions are, and when they are worth their cost.
- Choose between the four mechanisms using a decision table.
- Use workflow commands correctly (and avoid the removed ones).
- Share actions privately, within an organization, or publicly.

## 1. Problem statement

By the time you have five repositories, the same 30 lines (checkout, set up Node, restore cache, build, scan) exist in all of them. Copy-paste drifts: a fix in one place is missing in four others. GitHub offers four ways to define logic once and reuse it, each with different power and cost.

## 2. Core explanation

### The four mechanisms at a glance

| Mechanism | Reuses | Defined in | Runs as | Language |
|---|---|---|---|---|
| **Composite action** | A sequence of *steps* | `action.yml` | Steps inside the caller's job (same runner, same workspace) | YAML + shell/`uses` |
| **Reusable workflow** | Whole *jobs* | A workflow file with `on: workflow_call` | New job(s) on their own runner(s) | YAML |
| **JavaScript action** | Arbitrary logic via Node | `action.yml` + JS | A Node process on the runner | JS/TS |
| **Container action** | Arbitrary logic in any language | `action.yml` + Dockerfile | A Docker container (Linux runners only) | Anything |

### Lesson 8.1: Composite actions

A composite action packages steps. Create `.github/actions/setup-app/action.yml`:

```yaml
name: Setup app
description: Install Node and dependencies with caching

inputs:
  node-version:
    description: Node.js version
    required: false
    default: '22'

outputs:
  cache-hit:
    description: Whether the npm cache was restored
    value: ${{ steps.node.outputs.cache-hit }}

runs:
  using: composite
  steps:
    - id: node
      uses: actions/setup-node@v4
      with:
        node-version: ${{ inputs.node-version }}
        cache: npm
    - name: Install dependencies
      shell: bash                  # required for every run step in a composite action
      run: npm ci
```

Use it from a workflow in the same repository (checkout must happen first so the file exists on the runner):

```yaml
steps:
  - uses: actions/checkout@v5
  - uses: ./.github/actions/setup-app
    with:
      node-version: '22'
  - run: npm test
```

Facts to remember:

- Every `run:` step needs an explicit `shell:`.
- Inputs arrive as `${{ inputs.name }}`; define them in `action.yml`. A composite action **cannot read the `secrets` context**; the caller passes secrets in as inputs (`with: token: ${{ secrets.TOKEN }}`).
- Outputs must be declared in `outputs:` with a `value:` that points at a step output.
- Composite actions appear as a *single step* in the caller's log (the inner steps are expandable), run on the caller's runner and see its filesystem and environment variables.
- They can call other actions with `uses:`, and `if:` works inside them.
- They cannot declare `runs-on`, `services` or `container`; those belong to jobs.

Pass inputs to scripts through `env:` to avoid injection (Chapter 5):

```yaml
- shell: bash
  env:
    NAME: ${{ inputs.who-to-greet }}
  run: echo "Hello, $NAME"
```

### Lesson 8.2: Reusable workflows

A reusable workflow is a normal workflow file with a `workflow_call` trigger. `.github/workflows/reusable-test.yml`:

```yaml
name: Reusable test

on:
  workflow_call:
    inputs:
      node-version:
        description: Node.js version
        type: string
        required: false
        default: '22'
    secrets:
      NPM_TOKEN:
        required: false
    outputs:
      result:
        description: Test outcome
        value: ${{ jobs.test.outputs.result }}

permissions:
  contents: read

jobs:
  test:
    runs-on: ubuntu-24.04
    outputs:
      result: ${{ steps.done.outputs.result }}
    steps:
      - uses: actions/checkout@v5
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ inputs.node-version }}
          cache: npm
      - run: npm ci
        env:
          NPM_TOKEN: ${{ secrets.NPM_TOKEN }}
      - run: npm test
      - id: done
        run: echo "result=passed" >> "$GITHUB_OUTPUT"
```

Call it. Note that the **job** has `uses:` and no `steps:` or `runs-on:`:

```yaml
name: CI
on:
  pull_request:

permissions:
  contents: read

jobs:
  unit:
    uses: ./.github/workflows/reusable-test.yml     # same repository, same commit
    with:
      node-version: '22'
    secrets:
      NPM_TOKEN: ${{ secrets.NPM_TOKEN }}           # explicit is better than inherit

  report:
    needs: unit
    runs-on: ubuntu-24.04
    steps:
      - run: echo "unit tests ${{ needs.unit.outputs.result }}"
```

From another repository: `uses: my-org/shared-workflows/.github/workflows/reusable-test.yml@<tag-or-SHA>`. The same pinning advice applies as for actions. The repository must allow access (public, or internal/private with the "Actions → Access" setting enabled for sharing).

Facts to remember:

- Inputs have types (`string`, `number`, `boolean`); secrets are declared separately.
- `secrets: inherit` passes all of the caller's accessible secrets to the called workflow. It is convenient but broad; use it only within one organization where you control the called workflow.
- The called workflow's permissions can be **no broader** than the caller's token permission; the caller's `permissions:` caps it.
- Reusable workflows can be nested, but only to a limited depth, and a caller job cannot combine `uses:` with its own `steps:`. Check the [reusable workflows docs](https://docs.github.com/en/actions/reference/workflows-and-actions/reusable-workflows) for current limits.
- **Environment secrets:** a job reads environment secrets only when *that job* declares `environment: <name>`. If the called workflow's job needs staging/production secrets, declare `environment:` inside the called job (for example from an input). Do not assume that a caller's environment makes its secrets available to the called workflow; test this on a throwaway environment.
- Matrix and `needs` work on the calling job, so a caller can run a reusable workflow once per service.

### Lesson 8.3: Composite action vs. reusable workflow

| Question | Composite action | Reusable workflow |
|---|---|---|
| Unit of reuse | Steps | Jobs (including runner choice, matrix, services, environments) |
| Runs on | Caller's runner and workspace | Its own runner(s) |
| Appears in the log as | One step | One or more jobs |
| Can use `secrets` directly? | No (pass as input) | Yes (declared `secrets:` or `inherit`) |
| Can define `runs-on`/`services`/`environment`/`permissions`? | No | Yes |
| Share files with caller? | Yes (same workspace) | No (use artifacts) |
| Nest depth | Actions can call actions | Limited |
| Typical use | "Setup toolchain", "build and lint" | "Standard CI for every service", "deploy to env X" |

Rule of thumb: if the shared thing is *a few steps inside a job*, make a composite action. If it is *a whole pipeline stage*, make a reusable workflow.

### Lesson 8.4: JavaScript actions

A JavaScript action runs a Node.js program directly on the runner. It starts faster than a container action and works on every runner OS. `action.yml`:

```yaml
name: Greeter
description: Prints a greeting
inputs:
  who-to-greet:
    description: Who to greet
    required: true
outputs:
  time:
    description: Time of greeting
runs:
  using: node24          # check the docs for the Node versions the runner currently supports
  main: dist/index.js
```

`src/index.js` using the toolkit:

```javascript
const core = require('@actions/core');

try {
  const who = core.getInput('who-to-greet', { required: true });
  core.notice(`Hello, ${who}!`);
  core.setOutput('time', new Date().toISOString());
} catch (error) {
  core.setFailed(error.message);
}
```

Key points:

- The runner does **not** run `npm install` for you. Either commit the dependencies, or (more usual) **bundle** the code and dependencies into one file with a tool such as `@vercel/ncc` or esbuild and commit `dist/`.
- `@actions/core` gives `getInput`, `setOutput`, `setFailed`, `setSecret`, logging helpers; `@actions/github` gives an authenticated Octokit client; `@actions/exec`, `@actions/cache` and others exist.
- The supported Node runtime for actions changes over time (older ones such as `node16` and `node20` are retired or being retired). Check the metadata syntax docs for the current value when you author or update an action.
- TypeScript is just a compile step before bundling; the committed `dist/` is what runs. Add a CI check that `dist/` is up to date.

Use a JavaScript action when you need real programming logic, GitHub API calls, performance, or cross-platform support.

### Lesson 8.5: Container actions

A container action ships its own filesystem and any language. `action.yml`:

```yaml
name: Python greeter
description: Greets in Python
inputs:
  who-to-greet:
    description: Who to greet
    required: true
runs:
  using: docker
  image: Dockerfile            # built on each run; or  docker://ghcr.io/org/image:tag  (pre-built, faster)
```

`Dockerfile`:

```dockerfile
FROM python:3.13-slim
COPY entrypoint.py /entrypoint.py
ENTRYPOINT ["python", "/entrypoint.py"]
```

`entrypoint.py`:

```python
import os

who = os.environ.get("INPUT_WHO-TO-GREET", "World")
print(f"::notice::Hello, {who}!")
with open(os.environ["GITHUB_OUTPUT"], "a") as out:
    out.write("time=now\n")
```

Input environment variable names are `INPUT_` + the input name **uppercased** (spaces become underscores; **hyphens are kept**). So `who-to-greet` becomes `INPUT_WHO-TO-GREET`, which shell code cannot read as `$INPUT_WHO-TO-GREET`; read it with `os.environ[...]` (Python), `env` (shell) or pass inputs as `args:` in `action.yml`.

Trade-offs: Linux runners only; a `Dockerfile` image is built every run (slow), a pre-built `docker://` image is much faster; larger attack surface; full language freedom. Use it when you need a specific toolchain or language that isn't available on the runner.

### Lesson 8.6: Workflow commands (what actions use to talk to the runner)

| Purpose | Mechanism |
|---|---|
| Notice / warning / error annotation | `echo "::notice file=app.js,line=1::message"` (`::warning`, `::error`) |
| Group log lines | `::group::Title` … `::endgroup::` |
| Mask a value in logs | `echo "::add-mask::value"` |
| Debug message (shown with debug logging) | `echo "::debug::message"` |
| Set a step output | append `name=value` to `$GITHUB_OUTPUT` |
| Set an env var for later steps | append to `$GITHUB_ENV` |
| Prepend to `PATH` | append the directory to `$GITHUB_PATH` |
| Markdown on the run summary | append to `$GITHUB_STEP_SUMMARY` |

The old `::set-output`, `::save-state` (in its original form), `::set-env` and `::add-path` commands are deprecated and the last two were disabled for security reasons; use the environment files above.

### Lesson 8.7: Sharing and publishing

| Where | How | Notes |
|---|---|---|
| Same repository | `uses: ./.github/actions/name` | Requires checkout first |
| Public repository | `uses: owner/repo@v1` | Anyone can use it; tag releases, move a major tag (`v1`) deliberately, and document inputs |
| Private repository, same org | Enable sharing in the repository's **Settings → Actions → General → Access** | Callers need access |
| Marketplace | Add `name`, `description`, branding to `action.yml`, then publish a release with the "Publish to Marketplace" option | Optional; listing is not required to use an action |
| Reusable workflows in a central repo | `uses: org/repo/.github/workflows/x.yml@tag` | Version them with tags |

Version your action like a library. Publish `v1.2.3` tags, move the `v1` tag only for backward-compatible changes, and write release notes.

## 3. Mental model

- A **composite action** is a *macro*: it pastes steps into your job.
- A **reusable workflow** is a *function that creates jobs*: it has parameters, its own machines, and a return value.
- A **JavaScript action** is a *small program* the runner executes.
- A **container action** is a *program shipped with its own operating system*.

## 4. Decision table

| Need | Choose |
|---|---|
| Share 3–10 steps; same runner and files | Composite action |
| Share an entire CI or deploy pipeline; own runners/environments/matrix | Reusable workflow |
| Complex logic or GitHub API work, runs on all OSes, fast start | JavaScript action |
| Non-JS language or special system dependencies | Container action |
| Logic you'd also like to run locally | A script or task runner in the repo (Chapter 10), called from any of the above |

Often the best answer is none of them: write `scripts/thing.sh`, test it locally, and call it with `run:`. Create an action when you need reuse across repositories or a clean inputs/outputs contract.

## 5. Practical example

A reusable workflow that calls a composite action, called once per service via a matrix. Layout:

```
.github/
  actions/setup-app/action.yml        # Lesson 8.1
  workflows/
    reusable-test.yml                 # Lesson 8.2, adding a `service` input
    ci.yml                            # caller
```

Caller:

```yaml
name: CI
on:
  pull_request:
permissions:
  contents: read
jobs:
  test:
    strategy:
      matrix:
        service: [api, web]
    uses: ./.github/workflows/reusable-test.yml
    with:
      node-version: '22'
      service: ${{ matrix.service }}
```

In `reusable-test.yml`, add the input `service` (type `string`) and use it in a step with `working-directory: services/${{ inputs.service }}`. Pass any `inputs.*` through `env:` if used inside a `run:` script.

## 6. Execution walkthrough

1. A pull request fires `ci.yml`. The matrix creates two *calling* jobs (`test (api)`, `test (web)`).
2. Each calling job starts the reusable workflow as its own set of jobs, each on a fresh runner, with the matrix value passed as `inputs.service`.
3. In the called job, `actions/checkout` runs, then the local composite action is read from the workspace and its inner steps run inline on the same runner.
4. The called workflow's job output flows up to the caller's `needs.test.outputs.*`. If any called job fails, the calling job fails.

## 7. Common mistakes

| Mistake | Effect | Fix |
|---|---|---|
| Composite `run:` without `shell:` | Error: required | Add `shell: bash` |
| Using `secrets.X` inside a composite action | Not available | Pass as an input |
| Calling a local composite action before `checkout` | Action not found | Checkout first |
| Giving the calling job both `uses:` and `steps:` | Invalid | One or the other |
| Expecting files from a reusable workflow to appear in the caller | Separate runners | Upload/download artifacts |
| `secrets: inherit` across trust boundaries | Over-sharing | Pass secrets explicitly |
| Assuming a caller's environment secrets reach the called workflow | Empty values | Declare `environment:` in the called job and test |
| JS action without committed `dist/` | "Cannot find module" | Bundle and commit it |
| Reading `$INPUT_WHO-TO-GREET` in bash | Invalid variable name | Use `env`/`printenv` or `args:` |
| Using `::set-output`, `::set-env`, `::add-path` | Deprecated or disabled | Use the environment files |
| Tagging a breaking change as `v1` | Breaks every consumer | Bump the major version |

## 8. Security considerations

- Reusable workflows and shared actions are a **centralized attack surface**: whoever can push to the shared repository can change every caller's pipeline. Protect its branches and pin callers to tags or SHAs.
- Callers should limit `permissions:` and pass only needed secrets.
- Treat inputs as untrusted text; never interpolate `${{ inputs.* }}` directly into a script (Chapter 5).
- Container actions run as root by default and with the workspace mounted; keep images minimal and pin base images by digest where practical.
- A published JavaScript action's `dist/` file is what runs; verify it matches the source (CI check).

## 9. Hands-on exercise

**Goal:** build and call a composite action and a reusable workflow.

1. On a branch in your practice repository create `.github/actions/hello/action.yml`, a composite action with input `name` (default `world`) and output `greeting`. Its steps: print a greeting using `env:` for the input, and write `greeting=Hello, <name>` to `$GITHUB_OUTPUT`.
2. Create `.github/workflows/reusable-hello.yml` (`workflow_call`) with an input `name` that checks out the repository, calls your composite action, and exposes the greeting as a workflow output.
3. Create `.github/workflows/hello-caller.yml` with `workflow_dispatch` and two calling jobs: one with `name: Ada`, one with a matrix over `[Ada, Linus]`. Add a third job that `needs:` the first and prints its output.
4. Run `actionlint`, push, merge to the default branch, trigger via `gh workflow run hello-caller.yml`.
5. Experiment: remove `shell: bash` and observe the error; add `secrets.X` inside the composite action and observe the context error; change `uses:` to include both `uses` and `steps` and read the validation message.

**Success looks like:** three called workflows visible in the run graph, with the output printed by the dependent job.

**If it fails:** "can't find action.yml" → missing checkout or wrong path; "workflow is not reusable" → no `workflow_call` trigger; "invalid input" → `with:` name differs from the declared input.

## 10. Verification checklist

- [ ] `actionlint` is clean.
- [ ] The composite action has `shell:` on all run steps and no `secrets` reference.
- [ ] The reusable workflow declares typed inputs and outputs.
- [ ] The caller sets `permissions:` and passes secrets explicitly (if any).
- [ ] You can state why you chose a composite action vs. a reusable workflow for each piece.

## 11. Knowledge check

1. What can a reusable workflow do that a composite action cannot?
2. Which runs on the caller's runner and workspace?
3. How does a composite action receive a secret?
4. Why must a JavaScript action's dependencies be bundled?
5. What is `INPUT_WHO-TO-GREET`, and why can't bash read it directly as a variable?
6. Why prefer `echo "x=y" >> "$GITHUB_OUTPUT"` over `::set-output`?
7. When is a script in the repository a better choice than any of the four?

<details>
<summary>Answers</summary>

1. Define jobs: its own runner, matrix, services, environments, permissions, and directly use secrets.
2. A composite action.
3. As an input passed by the caller (`with: token: ${{ secrets.TOKEN }}`).
4. The runner does not run `npm install` for actions.
5. The environment variable for input `who-to-greet`; hyphens make it an invalid shell identifier, so use `env`/`printenv`, a language runtime, or `args:`.
6. The old command is deprecated; the file-based approach is supported.
7. When the logic is used in one repository and should be runnable locally.
</details>

## 12. Key takeaways

- Composite = reuse steps; reusable workflow = reuse jobs; JavaScript = programmable and fast; container = any language.
- Don't create an action until a script isn't enough.
- Secrets, permissions and environments are handled differently in each; test them.
- Central shared logic needs central protection and versioning.

## Sources and further reading

- Original notes: `git show master:08_authoring_actions.md`. Corrections: environment-secret inheritance claim reworded and flagged for verification; removed `::add-path::`/`::set-env::`; clarified the `INPUT_` naming (hyphens retained, as the later chapters also assume); updated the JS runtime guidance; trimmed the duplicate decision flowchart that Chapter 11 also repeated.
- Official docs: [Reusable workflows](https://docs.github.com/en/actions/reference/workflows-and-actions/reusable-workflows) · [Workflow commands](https://docs.github.com/en/actions/reference/workflows-and-actions/workflow-commands) · [Workflow syntax](https://docs.github.com/en/actions/reference/workflows-and-actions/workflow-syntax) · [Contexts](https://docs.github.com/en/actions/reference/workflows-and-actions/contexts)
- Not verified here: exact limits on reusable-workflow nesting depth and the Node runtimes currently accepted in `runs.using`; check the docs when you build.
