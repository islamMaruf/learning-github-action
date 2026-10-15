# Chapter 1: What GitHub Actions Is and How a Workflow Runs

> **Level:** beginner · **Time:** 45–60 minutes · **Prerequisites:** basic Git (clone, branch, commit, pull request), a terminal, a GitHub account
> **Next:** [Chapter 2: Why CI/CD Exists](02_history_and_motivation.md)

## Learning objectives

After this chapter you can:

- Explain what GitHub Actions is, and the difference between CI and CD.
- Name the five building blocks (event, workflow, job, step, runner) and say how they relate.
- Describe what GitHub does between "I pushed a commit" and "a green check appears".
- Explain the difference between GitHub's orchestration service and the runner that executes your commands.
- Say when a workflow uses a VM, when it uses a container, and when it uses neither.
- Run your first workflow.

## 1. Problem statement

A team that changes code dozens of times a day cannot rely on people remembering to run tests, check formatting, build artifacts and deploy. Manual steps are slow, inconsistent between people, and easy to skip under pressure.

**Automation** means writing those steps down once as code, and letting a machine run them every time something happens. **GitHub Actions** is the automation platform built into GitHub: your automation lives in the repository next to your code, is versioned with it, and reacts to what happens in the repository.

## 2. Core explanation

### CI and CD

| Term | Stands for | What it does | Typical trigger |
|---|---|---|---|
| **CI** | Continuous Integration | Every change is built and checked automatically (tests, lint, type checks, security scans) so problems surface within minutes of being introduced. | Push, pull request |
| **CD (delivery)** | Continuous Delivery | Every change that passes CI produces a releasable artifact (for example a container image), ready to deploy at the push of a button. | Merge to the main branch |
| **CD (deployment)** | Continuous Deployment | Every change that passes CI is deployed automatically, with no manual approval. | Merge, tag |

GitHub Actions can do all three, and it can also automate repository chores (labeling issues, closing stale issues, publishing releases). It is not only a CI tool.

### The five building blocks

| Concept | What it is | Where it lives |
|---|---|---|
| **Event** | Something that happens: a push, a pull request, a schedule tick, a button click. | Declared with `on:` |
| **Workflow** | One YAML file describing what to do when an event occurs. | `.github/workflows/*.yml` |
| **Job** | A group of steps that runs on one runner. Jobs run in parallel unless you say otherwise. | `jobs:` in the workflow |
| **Step** | One task inside a job: a shell command (`run:`) or a reusable action (`uses:`). Steps run in order. | `steps:` in the job |
| **Runner** | The machine that executes a job. | `runs-on:` selects it |

```
Event ──triggers──▶ Workflow ──contains──▶ Job(s) ──contain──▶ Step(s)
                                              │
                                              └──run on──▶ Runner
```

A fifth word you will meet immediately is **action**: a packaged, reusable unit of automation that a step can call with `uses:`. For example, `actions/checkout` downloads your repository onto the runner.

### How it works: orchestration versus execution

Two separate things cooperate, and mixing them up causes most beginner confusion:

1. **GitHub's orchestration service** (a hosted web service). It receives events, reads your workflow files, decides which workflows and jobs should run, queues jobs, tracks their status, stores logs, and reports results on the commit and pull request. It never runs your commands.
2. **The runner** (a small agent program on a machine). It asks GitHub whether there is a job waiting for it (the runner pulls work over HTTPS; GitHub does not connect into your machine), downloads the job definition, executes each step as an operating-system process, streams the output back, and reports each step's exit code.

A **GitHub-hosted runner** is a fresh virtual machine that GitHub provisions for the job and discards afterwards. A **self-hosted runner** is a machine you operate that runs the same agent.

### Is every workflow run inside Docker?

No. This is a common misconception.

| Situation | Where your commands actually run |
|---|---|
| Default (`runs-on: ubuntu-latest`) | Directly on the runner's operating system (a VM). No container involved. |
| `container:` on a job | The job's steps run inside the container image you name, on top of the runner. |
| `services:` on a job | Extra containers (for example a database) start next to your job. Your steps still run on the runner unless you also set `container:`. |
| A step that runs `docker build` or `docker run` | Docker is used because *you* invoked it. |
| A Docker container action (`runs.using: docker`) | That one action runs in a container. |

## 3. Mental model

Think of GitHub as a **restaurant's order system** and the runner as a **cook**. The order system (orchestration) takes the order (event), turns it into tickets (jobs), and tracks progress. The cook (runner) takes a ticket, does the work in a clean kitchen (a fresh VM), and reports back "done" or "failed". The order system never cooks, and the cook never decides which orders exist.

## 4. Syntax breakdown

The smallest useful workflow:

```yaml
name: Hello

on: workflow_dispatch

jobs:
  say-hello:
    runs-on: ubuntu-24.04
    steps:
      - run: echo "Hello from $RUNNER_OS"
```

| Line | Meaning |
|---|---|
| `name: Hello` | Display name in the **Actions** tab. Optional. |
| `on: workflow_dispatch` | The event. This one means "only when someone starts it manually". |
| `jobs:` | The map of jobs. |
| `say-hello:` | The **job ID**: a key you choose, used to reference the job elsewhere. |
| `runs-on: ubuntu-24.04` | The runner image: a GitHub-hosted Ubuntu 24.04 VM. |
| `steps:` | The ordered list of steps. |
| `- run: echo ...` | A step that runs a shell command. On Ubuntu the default shell is bash. |

## 5. Practical example: a test workflow

This is what a typical CI workflow for a Node.js project looks like. You do not need to understand every line yet; the following chapters explain each piece. Chapter 12 builds a larger version.

```yaml
name: CI

on:
  pull_request:
  push:
    branches: [main]

permissions:
  contents: read

jobs:
  test:
    runs-on: ubuntu-24.04
    steps:
      - uses: actions/checkout@v5
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: npm
      - run: npm ci
      - run: npm test
```

## 6. Execution walkthrough

Here is what happens when you push a commit to a branch and a workflow like the one above is configured for `push`:

1. You run `git push`. GitHub stores the commits and generates a `push` event.
2. The orchestration service looks at `.github/workflows/` **at the pushed commit** and evaluates each workflow's `on:` conditions (branch filters, path filters, event types).
3. Each matching workflow becomes a **workflow run**. Its jobs are queued; a job whose `needs:` are not yet satisfied waits.
4. GitHub finds an eligible runner (one that matches `runs-on`). For a GitHub-hosted runner, it provisions a fresh VM.
5. The runner agent receives the job and prepares the environment: it downloads the actions the job uses, sets environment variables, and creates an empty workspace directory.
6. `actions/checkout` runs and copies your repository code at that commit into the workspace. Without this step, the workspace stays empty.
7. Dependency installation (`npm ci`) and test commands run as ordinary processes on the runner.
8. Each step ends with an **exit code**. Zero means success; any non-zero code fails the step, the remaining steps in that job are skipped (unless they use `if: always()` or `if: failure()`), and the job fails.
9. GitHub records the logs, per-step status and any uploaded artifacts, and publishes a **check** on the commit.
10. If the repository's branch protection requires that check, a failed run blocks the pull request from being merged. If it is not required, the red mark is only advisory.
11. The runner VM is destroyed. Nothing on it survives unless you uploaded it as an artifact or saved it to a cache.

## 7. Common mistakes

| Mistake | Why it fails | Fix |
|---|---|---|
| Putting the workflow file outside `.github/workflows/` | GitHub only discovers workflows in that exact directory. | Move it; the extension must be `.yml` or `.yaml`. |
| Forgetting `actions/checkout` | The runner starts with an empty workspace, so `npm test` finds no `package.json`. | Add `uses: actions/checkout@v5` as the first step. |
| Expecting files from one job in another | Each job gets a fresh runner. | Pass data with job outputs (Chapter 5), files with artifacts (Chapter 6). |
| Assuming a green check means "deployed" | Checks only report that the workflow's commands exited with code 0. | Verify the thing you care about (health check, version endpoint). |
| Wrong indentation in YAML | YAML structure is defined by whitespace. | Use an editor with YAML validation (Chapter 4). |

## 8. Security considerations

- A workflow executes code on a machine that may hold secrets. Treat workflow files like production code: review changes to `.github/workflows/` carefully.
- Pull requests from forks can propose workflow changes. GitHub limits what those runs can access by default; Chapter 11 explains the risks.
- Prefer GitHub-hosted runners while learning; self-hosted runners put *your* machine at the mercy of the code that runs on it.

## 9. Hands-on exercise: your first workflow

**Goal:** create a practice repository, add the workflow from section 4, run it manually and read the logs.

1. Create a repository you do not mind experimenting in (do not use a real project):
   ```bash
   gh repo create actions-lab --public --clone
   cd actions-lab
   mkdir -p .github/workflows
   ```
2. Create `.github/workflows/hello.yml` with the workflow from section 4.
3. Commit and push it to the default branch. (A `workflow_dispatch` workflow only shows its **Run workflow** button once the file exists on the default branch.)
   ```bash
   git add .github/workflows/hello.yml
   git commit -m "Add hello workflow"
   git push
   ```
4. Run it: open the repository → **Actions** → **Hello** → **Run workflow**, or from the terminal:
   ```bash
   gh workflow run hello.yml
   gh run watch
   ```
5. Open the run, click the `say-hello` job, and expand the step.

**Why:** you see the orchestration (queued → running → success) and the execution (the echo output) as separate things.

**Success looks like:** a green check, and the log line `Hello from Linux`.

**If it fails:** if the workflow does not appear in the **Actions** tab, check the path and that the file is on the default branch. If you see a YAML error banner, check indentation.

**Stretch:** change the step to `run: exit 1`, push, run again, and observe what a failure looks like. Then add a second step after it and confirm it never runs.

## 10. Verification checklist

- [ ] The workflow file is at `.github/workflows/hello.yml` on the default branch.
- [ ] The run appears under **Actions** and ends green.
- [ ] You can point to the log line produced by your `echo`.
- [ ] You saw that a failing step skips the steps after it.

## 11. Knowledge check

1. What is the difference between a workflow, a job and a step?
2. Which component decides *which* jobs should run, and which component actually runs your shell commands?
3. A job has `runs-on: ubuntu-latest` and no `container:`. Where do its commands run?
4. Why does a second job not see files created by the first?
5. A pull request shows a failing check but can still be merged. Why might that be?

<details>
<summary>Answers</summary>

1. A workflow is one YAML file; a job is a group of steps on one runner; a step is a single command or action call.
2. GitHub's orchestration service decides; the runner executes.
3. Directly on the runner VM, not in a container.
4. Each job runs on its own fresh runner.
5. The check is not marked as required in the branch protection rules.
</details>

## 12. Key takeaways

- GitHub Actions automates tasks in response to repository events, using YAML files in `.github/workflows/`.
- Orchestration (GitHub's service) and execution (the runner) are separate.
- Containers are optional; the default is a plain VM.
- Exit codes decide success or failure; a failing step stops its job.
- A passing run is evidence about *your commands*, not proof that production works.

## Sources and further reading

- Original notes for this chapter: `git show master:01_introduction.md` (course introduction notes, now condensed; the logistics sections were removed).
- Official docs: [GitHub Actions documentation](https://docs.github.com/en/actions) · [Workflow syntax](https://docs.github.com/en/actions/reference/workflows-and-actions/workflow-syntax) · [Contexts](https://docs.github.com/en/actions/reference/workflows-and-actions/contexts)
- The learning material in this repository is based on notes from the DevOps Directive course *GitHub Actions: Beginner to Pro*; see the [README](README.md) for attribution.
