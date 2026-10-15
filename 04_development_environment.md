# Chapter 4: Your Workflow Development Environment

> **Level:** beginner · **Time:** 60–90 minutes · **Prerequisites:** [Chapter 1](01_introduction.md); Git and a GitHub account
> **Next:** [Chapter 5: Core Features](05_core_features.md)

## Learning objectives

After this chapter you can:

- Set up a practice repository where you can experiment safely.
- Configure an editor so workflow YAML is validated as you type.
- Lint workflows locally with `actionlint` and run simple ones with `act`.
- Explain what `act` can and cannot reproduce about real GitHub runs.
- Use the GitHub CLI (`gh`) to trigger and watch workflow runs.

## 1. Problem statement

The default feedback loop for a workflow is: edit YAML → commit → push → wait for a runner → read logs → repeat. Each loop takes minutes and fills your history with "fix workflow" commits. A small amount of local tooling removes most of those loops: catch syntax and type errors before pushing, and run simple workflows on your own machine.

## 2. Core explanation

### The toolbox

| Tool | Purpose | Required? |
|---|---|---|
| **Git** | Version control; workflows are files in the repository. | Yes |
| **GitHub CLI (`gh`)** | Create repos, trigger workflows, watch runs, read logs, get an auth token. | Strongly recommended |
| **Editor with YAML support** (VS Code + *YAML* and *GitHub Actions* extensions) | Syntax highlighting, schema validation, autocompletion. | Recommended |
| **`actionlint`** | Static checker for workflow files: catches wrong keys, bad expressions, type errors, and risky shell patterns. | Strongly recommended |
| **Docker** | Container runtime. Needed by `act`, and for building images in later chapters. | Needed from Chapter 6 |
| **`act`** | Runs workflows locally in containers that imitate GitHub runners. | Optional |
| **Task** (`go-task`) | A task runner so pipeline logic can live in a `Taskfile.yml` you can run locally. | Optional until Chapter 10 |
| **Devbox / mise / asdf** | Pin tool versions per project. | Optional |

On Windows, use WSL 2 and work inside the Linux environment: runners are Linux by default, and most scripts and tools assume it.

### Why a separate practice repository?

Workflows run in a repository that **you own or can write to**. A local clone of someone else's repository does nothing on GitHub: to run workflows you need a repository of your own on GitHub (a fork, or a fresh repository). Forks also have scheduled and event-triggered workflows disabled until you enable Actions in the fork's **Actions** tab. A fresh practice repository avoids that surprise and risks nothing important.

### The editor gotcha: file associations

If both the *YAML* (Red Hat) and *GitHub Actions* extensions are installed, VS Code can end up treating workflow files with the wrong language mode, and you lose YAML validation. Force the association in your settings:

```json
{
  "files.associations": {
    ".github/workflows/*.yml": "yaml",
    ".github/workflows/*.yaml": "yaml"
  }
}
```

If the *GitHub Actions* extension shows its own validation, leave it enabled alongside; the point is that the file is recognised as YAML first.

### `act`: what it is and what it is not

`act` reads your workflow file and runs jobs in Docker containers on your machine.

**Good for:** checking step ordering, shell logic, environment variables, and simple build/test jobs.

**Poor at:**

- Anything depending on GitHub's services: the `github` context fields from real events, caches and artifacts (partial support), OIDC tokens, deployments, environments and approvals.
- Exact runner parity. Its default images are much smaller than GitHub's runner images; larger images are available but are multiple gigabytes.
- Steps that change repository state (pushing commits, creating releases).

Treat `act` as a fast pre-check. **The real GitHub run is the authority.**

## 3. Mental model

Think of local tooling as **spell-check, then a rehearsal, then opening night**:

1. `actionlint` and the editor check the *grammar* (structure and expressions).
2. `act` or your task runner rehearses the *commands*.
3. A run on GitHub is the performance, in the real environment, with real permissions.

## 4. Syntax and commands

### Install the essentials

macOS with Homebrew (use your OS package manager or the project's install docs on other systems):

```bash
brew install gh actionlint act go-task
gh auth login          # sign in once
```

Verify:

```bash
git --version
gh --version
actionlint -version
docker --version       # needed for act and image builds
act --version
```

### The commands you will use daily

| Command | What it does |
|---|---|
| `actionlint` | Lints every file in `.github/workflows/`. Exit code 0 means no problems found. |
| `gh workflow list` | Lists workflows in the repository. |
| `gh workflow run hello.yml` | Starts a `workflow_dispatch` workflow. |
| `gh run list` / `gh run watch` | Shows recent runs / follows one live. |
| `gh run view --log-failed` | Prints logs of only the failed steps. |
| `act -l` | Lists the jobs `act` would run. |
| `act push -j test` | Runs the job `test` as if a push event happened. |

### Optional: pin tool versions with Devbox

[Devbox](https://www.jetify.com/devbox) (formerly from Jetpack.io, now Jetify) gives each project an isolated, reproducible set of tools. A minimal `devbox.json`:

```json
{
  "packages": ["gh@latest", "actionlint@latest", "act@latest", "go-task@latest", "jq@latest"]
}
```

`devbox shell` opens a shell with exactly those tools. Pin concrete versions instead of `latest` for real reproducibility, and check the project's current installation instructions, as the installer URL has changed over time.

## 5. Practical example: a lint-and-run loop

Create `.github/workflows/check.yml` in your practice repository:

```yaml
name: Check

on:
  workflow_dispatch:

permissions:
  contents: read

jobs:
  tools:
    runs-on: ubuntu-24.04
    steps:
      - uses: actions/checkout@v5
      - name: Show tool versions
        run: |
          git --version
          bash --version | head -n 1
          docker --version
          node --version
```

Now iterate locally before pushing:

```bash
actionlint                       # structure and expression check
act workflow_dispatch -j tools   # run the job in a local container
```

## 6. Execution walkthrough

1. `actionlint` parses the YAML, validates keys against the workflow schema and checks expressions such as `${{ ... }}`. It reports file, line and column for each problem.
2. `act` reads the same file, pulls a container image that stands in for `ubuntu-24.04`, mounts your working directory and runs each step in order.
3. After both pass locally, you push and let the real runner execute. If behavior differs, the difference is usually the environment (tool versions, permissions, secrets), which is exactly what you investigate in Chapter 10.

Note that on the first `act` run it asks which default image size to use, and downloads it; later runs reuse it.

## 7. Common mistakes

| Mistake | Why it hurts | Fix |
|---|---|---|
| Treating `act` output as proof the workflow is correct | It differs from GitHub in images, contexts, caches and permissions. | Always confirm on GitHub. |
| Cloning someone else's repository and expecting workflows to run | They run only in repositories on GitHub that you can write to. | Create your own repository or fork. |
| Skipping `actionlint` because "the editor validates" | Editors check structure; `actionlint` also catches expression and type errors. | Run it before every push (consider a pre-commit hook). |
| Passing real tokens to `act` casually | Secrets can end up in terminal history and logs. | Use a short-lived token, never commit a secrets file, and add it to `.gitignore`. |
| Windows without WSL | Path, shell and line-ending problems. | Use WSL 2. |

## 8. Security considerations

- `act -s GITHUB_TOKEN="$(gh auth token)"` hands your personal token to a local container. That token usually has broad rights. Prefer a fine-grained token with the minimum scopes, or avoid passing one.
- Keep any `.secrets` or `.env` file out of Git: add it to `.gitignore` before creating it.
- Docker socket access lets a container control your machine. Only run workflows you have read.

## 9. Hands-on exercise

**Goal:** set up the toolbox and complete one full lint → local run → remote run loop.

1. Create the practice repository (or reuse `actions-lab` from Chapter 1).
2. Install the tools from section 4 and run the verification commands.
3. Add `.github/workflows/check.yml` from section 5 on a branch:
   ```bash
   git checkout -b add-check-workflow
   ```
4. Run `actionlint`. Introduce a deliberate mistake (for example rename `runs-on` to `run-on`), run it again and read the error, then fix it.
5. Run `act workflow_dispatch -j tools`.
6. Push the branch, open a pull request, merge it, then trigger the workflow on GitHub:
   ```bash
   gh workflow run check.yml
   gh run watch
   ```
7. Compare tool versions printed locally by `act` and remotely by GitHub. Note any differences.

**Why:** you see the three layers (lint, rehearsal, real run) and the differences between them.

**Success looks like:** `actionlint` exits cleanly, `act` prints the versions, and the GitHub run is green.

**If it fails:** if `act` cannot reach Docker, start Docker first. If it picks an image that lacks `node`, use a larger image (`-P ubuntu-24.04=catthehacker/ubuntu:act-24.04`; check `act`'s documentation for current image names).

## 10. Verification checklist

- [ ] `gh auth status` shows you are signed in.
- [ ] `actionlint` catches your deliberate mistake and passes after the fix.
- [ ] `act -l` lists the `tools` job.
- [ ] The remote run is green and `gh run view --log-failed` prints nothing for it.

## 11. Knowledge check

1. Why might a workflow that passes under `act` still fail on GitHub?
2. What does `actionlint` catch that a YAML validator would not?
3. Why can you not run a cloned third-party repository's workflows on GitHub?
4. Name two risks of passing a personal access token into `act`.

<details>
<summary>Answers</summary>

1. Different runner image, missing GitHub contexts/services, different permissions or secrets.
2. Workflow-specific problems: invalid keys for the context, wrong expression types, shell issues in `run:`.
3. Workflows run where the repository lives on GitHub; a local clone is not on GitHub under your account.
4. Over-broad token rights and leaking it through logs/shell history.
</details>

## 12. Key takeaways

- Shorten the feedback loop with local checks: editor → `actionlint` → `act` → GitHub.
- Use a practice repository you own; never experiment on production repositories.
- `act` is a rehearsal, not the authority.
- Handle tokens used locally with the same care as production secrets.

## Sources and further reading

- Original notes: `git show master:04_development_environment.md`. The course-specific parts (forking the course repository, git submodules for its capstone) were replaced by the practice-repository approach; Devbox installer URLs in the notes were out of date.
- Tools: [GitHub CLI](https://cli.github.com/) · [actionlint](https://github.com/rhysd/actionlint) · [act](https://github.com/nektos/act) · [Task](https://taskfile.dev/)
- Official docs: [Workflow syntax](https://docs.github.com/en/actions/reference/workflows-and-actions/workflow-syntax)
