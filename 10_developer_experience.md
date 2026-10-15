# Chapter 10: Developer Experience: Iterating, Debugging and Measuring Workflows

> **Level:** intermediate · **Time:** 3 hours · **Prerequisites:** [Chapters 4–9](04_development_environment.md)
> **Next:** [Chapter 11: Best Practices](11_best_practices.md)

## Learning objectives

After this chapter you can:

- Explain why the "edit, push, wait" loop is slow and which techniques shorten it.
- Move pipeline logic out of YAML into a task runner (Task) that works locally and in CI.
- Run workflows locally with `act` and know its limits.
- Test an action's logic locally and with unit tests.
- Debug failing runs with debug logging, re-run with debug, and (carefully) a remote shell.
- Measure where time goes in a pipeline.

## 1. Problem statement

A workflow executes on a remote machine you cannot touch. Each experiment costs a commit, a queue wait and a log read: minutes per attempt, plus a history full of "try again" commits. The goal of this chapter is to move as much of the loop as possible onto your own machine and to have good tools for the part that has to run on GitHub.

## 2. Core explanation

### Lesson 10.1: The three levels of iteration

| Level | What you iterate on | Fastest feedback |
|---|---|---|
| **Workflow YAML** | Triggers, jobs, `needs`, expressions | `actionlint`, editor schema, `act` |
| **Commands inside steps** | Build, test, deploy scripts | Run the same script locally |
| **Actions you write** | Action code | Run/unit-test the code locally |

Reducing the *YAML* to a thin shell around *commands you can run anywhere* is the single highest-leverage habit in this chapter.

### Lesson 10.2: Extract logic from YAML into a task runner

Anti-pattern: 40 lines of Bash in a `run: |` block. You can't run it locally, can't lint it as shell, and can't reuse it.

Better: put commands in a **Taskfile** (using [Task](https://taskfile.dev)) and call it from the workflow.

`Taskfile.yml`:

```yaml
version: '3'

tasks:
  install:
    cmds:
      - npm ci
  lint:
    deps: [install]
    cmds:
      - npm run lint
  test:
    deps: [install]
    cmds:
      - npm test
  ci:
    cmds:
      - task: lint
      - task: test
```

Workflow:

```yaml
name: CI
on:
  pull_request:
permissions:
  contents: read
jobs:
  ci:
    runs-on: ubuntu-24.04
    timeout-minutes: 15
    steps:
      - uses: actions/checkout@v5
      - uses: actions/setup-node@v4
        with:
          node-version: 22
          cache: npm
      - uses: go-task/setup-task@v1
      - run: task ci
```

Locally: `task ci`. In CI: the same. Benefits: one source of truth, easy to reproduce a CI failure on your laptop, and if you ever change CI providers the Taskfile moves with you. (Make, just, npm scripts or plain shell scripts serve the same purpose; pick whatever your team already uses.)

### Lesson 10.3: Run workflows locally with `act`

Chapter 4 introduced `act`. Beyond the basics:

```bash
act -l                                   # list jobs
act pull_request -j test                 # simulate a PR event for one job
act -W .github/workflows/ci.yml          # a specific workflow file
act --secret-file .secrets               # secrets from a local file (never commit it)
act -P ubuntu-24.04=catthehacker/ubuntu:act-24.04   # choose a runner image (check act docs for current names)
act -n                                   # dry run
```

Event payloads for `github.event` can be supplied with `-e event.json`.

What `act` does well: shell logic, env vars, ordering, matrix expansion. What it does not: real GitHub contexts and services, OIDC, many caches/artifacts, environments and approvals, exact runner images. If a workflow passes under `act` and fails on GitHub, trust GitHub. Combined with a Taskfile, `act` becomes less necessary: you run `task ci` directly.

### Lesson 10.4: Test actions locally

A JavaScript action reads its inputs from environment variables named `INPUT_<NAME>` (name uppercased, spaces to underscores, hyphens kept). To run the compiled action locally:

```bash
export "INPUT_WHO-TO-GREET=Ada"
export GITHUB_OUTPUT="$(mktemp)"
node dist/index.js
cat "$GITHUB_OUTPUT"
```

(Use `export "…"` with quotes because the name contains a hyphen; a plain `export INPUT_WHO-TO-GREET=…` is invalid in bash. Alternatively use `env 'INPUT_WHO-TO-GREET=Ada' node dist/index.js`.)

Better: structure the action so the logic lives in a function that takes plain values, and **unit-test** that function with your language's test framework (Jest for TypeScript). Keep the thin entry file (reading inputs, calling `core.setOutput`) untested or lightly tested. Add an *integration test workflow* in the action's repository that runs the action against fixtures using `uses: ./`.

### Lesson 10.5: Debugging runs on GitHub

#### Re-run with debug logging

On a finished run, **Re-run jobs → Re-run jobs with debug logging** (or `gh run rerun <id> --debug`) turns on extra logging for that run only. This is the first thing to try.

#### Debug logging via settings

Two switches (as repository **secrets or variables**; the names are exactly these):

| Name | Effect |
|---|---|
| `ACTIONS_STEP_DEBUG` = `true` | Shows `::debug::` messages from steps and actions (for example `core.debug()`) |
| `ACTIONS_RUNNER_DEBUG` = `true` | Adds runner diagnostic logs (downloadable with the log archive) |

Set them only while investigating, then remove them; verbose logs can expose more data and clutter every run. Remember that secret masking still applies, but debug output can contain values that are *not* secrets yet are sensitive.

#### Make failures visible

- Add `set -x` temporarily in a `run:` block to echo commands, **but not around commands that handle secrets**.
- Print the environment you depend on (`env | sort`), but **not** if secrets are present in it: secrets passed to a step via `env:` appear there masked, yet other tools may transform them.
- Use `::group::` / `::endgroup::` to fold noisy output.
- Write a short report to `$GITHUB_STEP_SUMMARY`.
- Upload logs or reports with `if: always()`.

```yaml
- name: Dump context (learning only; remove afterwards)
  run: |
    echo "ref=$GITHUB_REF sha=$GITHUB_SHA actor=$GITHUB_ACTOR"
    echo "runner: $RUNNER_OS $RUNNER_ARCH"
```

#### Get a shell on the runner

GitHub has no built-in "SSH into the runner". Community actions provide a time-limited remote shell when something fails, for example `namespacelabs/breakpoint-action` (the published tag is `@v0`; there is no `@v1`) or `mxschmitt/action-tmate`. They are powerful and risky:

```yaml
- name: Breakpoint on failure
  if: failure()
  uses: namespacelabs/breakpoint-action@v0     # pin to a SHA in real use
  with:
    duration: 10m
    authorized-users: your-github-username
```

Rules: use them only in a **private or practice repository**, restrict to named users, set a short duration, remove the step when done. Anyone who reaches the session can read environment variables and secrets available to the job. Never leave a breakpoint step in a workflow on a public repository's default branch.

### Lesson 10.6: Measure performance

GitHub's run page shows durations per job and step, but little history or aggregation. Options:

| Option | What you get |
|---|---|
| Run page and `gh run view --json` / API | Per-job and per-step timings you can script |
| GitHub's Actions usage and performance views (availability depends on plan) | Aggregated usage and trends |
| Third-party runner vendors | Their dashboards of runner CPU/RAM/time |
| Export timings to an observability platform (for example Honeycomb with a build-events action) | Traces of each run, queries across many runs |

Chapter 14 sets up an export of this kind in the capstone. A simple start without any service:

```bash
gh run list --workflow CI --limit 20 --json databaseId,conclusion,startedAt,updatedAt \
  --jq '.[] | [.databaseId, .conclusion, .startedAt, .updatedAt] | @tsv'
```

Measure before optimizing (Chapter 11).

## 3. Mental model

Think of the pipeline as a **product with a development loop**, like application code. You'd never develop an app by deploying to production every time you change a line. The Taskfile is your local dev server; `actionlint` is the compiler; `act` is the emulator; debug logs are the debugger; timing data is the profiler.

## 4. Syntax breakdown

| Item | Meaning |
|---|---|
| `Taskfile.yml` → `tasks.<name>.cmds`, `deps` | Commands and prerequisites |
| `uses: go-task/setup-task@v1` | Installs Task on the runner |
| `act <event> -j <job>` | Local run of one job |
| `INPUT_<NAME>` | How an action receives an input |
| `ACTIONS_STEP_DEBUG`, `ACTIONS_RUNNER_DEBUG` | Debug switches |
| `gh run rerun <id> --debug` | Re-run with debug logging |
| `if: failure()` | Run a step only after an earlier failure |

## 5. Practical example

A workflow that preserves diagnostics on failure and gives a readable summary:

```yaml
name: Diagnosable CI
on:
  workflow_dispatch:
permissions:
  contents: read
jobs:
  ci:
    runs-on: ubuntu-24.04
    timeout-minutes: 10
    steps:
      - uses: actions/checkout@v5
      - uses: go-task/setup-task@v1
      - name: Run CI task
        id: ci
        shell: bash                # explicit shell enables -eo pipefail, so `tee` cannot hide a failure
        run: task ci 2>&1 | tee ci.log
      - name: Summarize
        if: always()
        run: |
          {
            echo "### CI result"
            echo "Outcome: \`${{ steps.ci.outcome }}\`"
            echo '```'
            tail -n 20 ci.log || true
            echo '```'
          } >> "$GITHUB_STEP_SUMMARY"
      - uses: actions/upload-artifact@v4
        if: always()
        with:
          name: ci-log
          path: ci.log
```

(Piping to `tee` would hide the exit code of `task` under the implicit default shell, which runs `bash -e` without `pipefail`. Setting `shell: bash` explicitly runs `bash --noprofile --norc -eo pipefail`, so the failure is detected. Exercise step 6 lets you see the difference.)

## 6. Execution walkthrough

1. Checkout and Task setup run. `task ci` executes `lint` then `test` (the Taskfile's order) and `tee` writes the output to `ci.log` while streaming it to the log.
2. With `shell: bash` (explicit, which adds `pipefail`), a failing `task` fails the step; without it the pipeline's exit code is that of `tee` (0) and the step falsely succeeds.
3. The summary step runs regardless (`always()`), reading `steps.ci.outcome` and appending to the run summary page.
4. The artifact step uploads the log so you can download it after the run.

## 7. Common mistakes

| Mistake | Effect | Fix |
|---|---|---|
| 50 lines of Bash in YAML | Unreproducible, unlintable | Move to scripts/Taskfile |
| Trusting `act` as proof | False confidence | Confirm on GitHub |
| Leaving `ACTIONS_STEP_DEBUG` on | Noisy logs, more exposure | Remove after investigating |
| `namespacelabs/breakpoint-action@v1` | Tag doesn't exist | `@v0` (pin to a SHA) |
| SSH/tmate debug step on a public repo | Outsiders may reach secrets | Private repo, restricted users |
| `command \| tee` without `pipefail` | Failures hidden | `shell: bash` |
| `export INPUT_WHO-TO-GREET=x` in bash | Syntax error | `export "INPUT_WHO-TO-GREET=x"` or `env '…'` |
| Optimizing without measuring | Wasted effort | Measure first |

## 8. Security considerations

- Debug logging and `set -x` can reveal more than you expect; secret masking only covers exact known values.
- Remote-shell actions are deliberately a back door. Use them only where you accept that.
- `act --secret-file`: keep the file out of Git, and use throwaway or fine-grained credentials.
- Observability exports send run metadata (repository, branch, commit messages) to a third party. Check what you share.
- Don't print the `github` context with `toJSON` on pull-request events from forks and then leave that step in place: it can surface untrusted text in logs.

## 9. Hands-on exercise

**Goal:** make a pipeline reproducible locally and debug a deliberately broken run.

1. Create `Taskfile.yml` in your practice repository with tasks `lint`, `test`, `ci` (use simple `echo` commands) and run `task ci` locally.
2. Change the workflow from Lesson 10.2 to call `task ci`; run `actionlint`; push on a branch and run it.
3. Break it: make `test` run `exit 1`. Push, run, and observe failure.
4. Run **Re-run jobs with debug logging**; compare the extra lines.
5. Add the "diagnosable CI" pieces: summary, artifact upload. Check the summary page.
6. Reproduce the `tee` trap: remove `shell: bash` from a step that pipes to `tee` and see the step pass despite the failure; add the shell back.
7. (Optional, private practice repo only) Add the breakpoint step with only your username authorized, trigger a failure and connect. Remove the step afterwards.

**Success looks like:** `task ci` fails the same way locally and in CI; you can open the summary and the log artifact.

**If it fails:** `task: command not found` locally → install Task (Chapter 4); the breakpoint action can't resolve → wrong tag.

## 10. Verification checklist

- [ ] Local and CI both run `task ci`.
- [ ] A broken test fails the job in both places.
- [ ] A debug re-run shows additional output.
- [ ] No debug secret/variable is left enabled.
- [ ] No remote-shell step remains in committed workflows.

## 11. Knowledge check

1. Why move logic from YAML into a Taskfile or script?
2. Which two names enable extra logging, and what does each show?
3. How can you get extra debug output without changing repository settings?
4. Why is a remote shell action dangerous on a public repository?
5. Why can `cmd | tee file` hide a failure?
6. What are the limits of `act`?

<details>
<summary>Answers</summary>

1. Reproducible locally, lintable, reusable, provider-independent.
2. `ACTIONS_STEP_DEBUG` (step/action debug messages), `ACTIONS_RUNNER_DEBUG` (runner diagnostics).
3. Re-run jobs with debug logging.
4. Others may reach a session that exposes the job's environment and secrets.
5. The pipeline's exit status is `tee`'s unless `pipefail` is set.
6. Different images, contexts and services; no OIDC/environments; partial cache/artifact support.
</details>

## 12. Key takeaways

- Shrink the feedback loop: thin YAML, commands you can run locally.
- `actionlint` → local run → remote run, and trust the remote run last.
- Debug with logging switches and debug re-runs before reaching for a remote shell.
- Measure before you optimize.

## Sources and further reading

- Original notes: `git show master:10_developer_experience.md`. Corrections: `namespacelabs/breakpoint-action@v1` (does not exist; the tag is `v0`), `INPUT_` environment-variable handling, debug-logging setup wording, and the duplicated Honeycomb walkthrough (kept once, in Chapter 14). The Rainbow Indent extension tip and the long sample SSH session transcript were dropped.
- Tools: [Task](https://taskfile.dev/) · [act](https://github.com/nektos/act) · [actionlint](https://github.com/rhysd/actionlint)
- Official docs: [Workflow commands](https://docs.github.com/en/actions/reference/workflows-and-actions/workflow-commands) · [Workflow syntax](https://docs.github.com/en/actions/reference/workflows-and-actions/workflow-syntax)
- Not verified here: the exact UI labels for debug re-runs and the availability of GitHub's performance views by plan.
