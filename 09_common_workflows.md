# Chapter 9: Common Workflow Patterns: Lint, Test, Build, Deploy, Maintain

> **Level:** intermediate · **Time:** 3 hours · **Prerequisites:** [Chapters 5–8](05_core_features.md)
> **Next:** [Chapter 10: Developer Experience](10_developer_experience.md)

## Learning objectives

After this chapter you can:

- Design a workflow by starting from the local command, then wrapping it.
- Write lint, test, static-analysis, build and release workflows with sensible triggers and permissions.
- Explain the difference between push-based and GitOps deployment workflows and write the safe version of each.
- Automate repository chores: releases, stale issues, dependency updates.
- Use GitHub's starter workflow templates without copying them blindly.

## 1. Problem statement

Chapters 5–8 taught features. Real work needs patterns: *what does a good lint workflow look like? how should deploys be triggered? what automation is worth having?* This chapter is a catalogue of the patterns you will write most, each with the mistakes that usually come with them.

## 2. Core explanation

### Lesson 9.1: How to design a workflow

1. **Make it work on your machine first.** Write the exact commands (`npm run lint`, `go test ./...`). A workflow that wraps commands you already trust is easy to debug.
2. **Wrap, don't invent.** Add `runs-on`, checkout, language setup, then the commands.
3. **Choose the trigger by the question you're answering:** "Is this change OK?" → `pull_request`. "What's now on main?" → `push` to `main`. "Ship it" → tag, release or manual dispatch with approvals.
4. **Add the safety defaults** from Chapters 5–6: `permissions: contents: read`, `timeout-minutes`, `concurrency`.
5. **Then optimize**: caching, path filters, matrix.

### Lesson 9.2: Linting

```yaml
name: Lint
on:
  pull_request:
  push:
    branches: [main]
permissions:
  contents: read
concurrency:
  group: lint-${{ github.ref }}
  cancel-in-progress: true
jobs:
  lint:
    runs-on: ubuntu-24.04
    timeout-minutes: 10
    steps:
      - uses: actions/checkout@v5
      - uses: actions/setup-node@v4
        with:
          node-version: 22
          cache: npm
      - run: npm ci
      - run: npm run lint
```

The "optimized" version versus a naive one is exactly: caching (via `cache: npm`), cancelling superseded runs, and narrow permissions. For multi-language repositories, an aggregator such as `super-linter/super-linter` (a maintained community project) runs many linters in one action; read its README for the current version and configuration. Keep linters in the repository's own config so developers run the same checks locally.

### Lesson 9.3: Testing

Testing workflows add three things beyond lint:

- **A matrix** over supported runtimes/OSes (Chapter 6).
- **Services** such as a database: 

  ```yaml
  services:
    postgres:
      image: postgres:16
      env:
        POSTGRES_PASSWORD: postgres     # throwaway CI-only value
      ports: ['5432:5432']
      options: >-
        --health-cmd "pg_isready -U postgres"
        --health-interval 5s --health-retries 10
  ```
  The `options` health check makes the job wait until the database is ready.
- **Test reports and coverage** uploaded with `if: always()` so you can inspect failures.

```yaml
- run: npm test -- --coverage
- uses: actions/upload-artifact@v4
  if: always()
  with:
    name: coverage-${{ matrix.node }}
    path: coverage/
```

If a test takes long, run it in a separate job so the quick checks report first.

### Lesson 9.4: Static analysis and security scanning

Typical building blocks: **CodeQL** (GitHub's code scanning; needs `security-events: write`), **dependency review** on pull requests, container image scanners (for example Trivy), and secret scanning (a repository setting rather than a workflow). Pattern:

```yaml
permissions:
  contents: read
  security-events: write       # to upload SARIF results to the Security tab
```

Run scans on pull requests (to block) and on a schedule (to catch newly disclosed vulnerabilities in unchanged code). Pin and review third-party scanners like any other action (Chapter 7). Check each tool's README for its current action name and version; do not copy `uses:` lines from older notes.

### Lesson 9.5: Building

Two common build outputs:

**Executables / bundles** → upload with `upload-artifact` or attach to a release. For multiple OS/architectures use a matrix and name each artifact with the matrix value.

**Container images** → covered end-to-end in Chapter 13 (Buildx, QEMU, metadata, push to a registry). The essentials:

```yaml
permissions:
  contents: read
  packages: write
steps:
  - uses: actions/checkout@v5
  - uses: docker/login-action@v3
    with:
      registry: ghcr.io
      username: ${{ github.actor }}
      password: ${{ secrets.GITHUB_TOKEN }}
  - uses: docker/build-push-action@v6
    with:
      context: .
      push: ${{ github.event_name != 'pull_request' }}   # build on PRs, push only on main/tags
      tags: ghcr.io/${{ github.repository }}:${{ github.sha }}
```

Tag images with something immutable (commit SHA or version), not just `latest`.

### Lesson 9.6: Deploying

| | Push-based | GitOps |
|---|---|---|
| Workflow does | Authenticate to the target and apply the change | Commit a new image tag to a manifest repository; an in-cluster agent syncs |
| Credentials in GitHub | Cloud/cluster credentials (prefer OIDC) | Write access to the manifests repo only |
| Rollback | Re-run an earlier deploy | `git revert` |

**Push-based, safely.** Key rules: use an `environment:` with required reviewers for production; serialize with `concurrency`; get credentials by OIDC; **never interpolate secrets or user text into the shell script**.

```yaml
name: Deploy
on:
  workflow_dispatch:
    inputs:
      environment:
        type: choice
        options: [staging, production]
        required: true

permissions:
  contents: read
  id-token: write

concurrency:
  group: deploy-${{ inputs.environment }}
  cancel-in-progress: false

jobs:
  deploy:
    runs-on: ubuntu-24.04
    timeout-minutes: 20
    environment: ${{ inputs.environment }}
    steps:
      - uses: actions/checkout@v5
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ vars.DEPLOY_ROLE_ARN }}     # non-secret config in vars
          aws-region: ${{ vars.AWS_REGION }}
      - name: Deploy
        env:
          TARGET: ${{ inputs.environment }}
        run: ./scripts/deploy.sh "$TARGET"
```

A frequent anti-pattern is writing a kubeconfig or key into a shell script via `${{ secrets.KUBECONFIG }}` inside `run:`. The secret is expanded into the script text and can leak through `set -x` output, and it makes injection mistakes easier. Pass it via `env:` or write it to a file from an `env:` variable, with `umask 077`, and delete it afterwards.

**GitOps, safely.** The workflow updates a version field in a manifest repository and pushes (Chapter 14 builds this fully). A bot should use a dedicated, narrowly scoped credential (a GitHub App token or deploy key), not a personal access token.

### Lesson 9.7: Repository automation

**Releases.** Options in increasing sophistication:

1. Manual: `gh release create v1.2.3 --generate-notes dist/*` in a job triggered by a tag (`on: push: tags: ['v*']`, `permissions: contents: write`).
2. `softprops/action-gh-release` for richer asset handling.
3. **Release PR tools** such as `googleapis/release-please-action`: it reads conventional commit messages, maintains a "release PR" with the changelog and next version, and creates the release when you merge it. Check its README for the current major version and for the token it needs.

The old `actions/create-release` and `actions/upload-release-asset` actions are archived and should not be used in new workflows; use the GitHub CLI or an actively maintained action.

```yaml
name: Release
on:
  push:
    tags: ['v*']
permissions:
  contents: write
jobs:
  release:
    runs-on: ubuntu-24.04
    steps:
      - uses: actions/checkout@v5
      - name: Create release
        env:
          GH_TOKEN: ${{ github.token }}
          TAG: ${{ github.ref_name }}
        run: gh release create "$TAG" --generate-notes
```

**Stale issues.** `actions/stale` on a schedule:

```yaml
name: Stale
on:
  schedule:
    - cron: '17 3 * * *'     # daily, off the top of the hour
permissions:
  issues: write
  pull-requests: write
jobs:
  stale:
    runs-on: ubuntu-24.04
    steps:
      - uses: actions/stale@v9
        with:
          days-before-stale: 60
          days-before-close: 14
          stale-issue-message: This issue has had no activity for 60 days. It will close in 14 days unless there is new activity.
          exempt-issue-labels: pinned,security
```

Be cautious: auto-closing real bug reports annoys users. Exempt important labels and pick generous timeouts.

**Dependency updates.** Use **Dependabot** (`.github/dependabot.yml`) or Renovate to open PRs for npm/pip/Go/Docker/GitHub Actions updates. Your normal CI runs on these PRs; the bot does not itself close PRs when tests fail, and it does not merge them unless you set up auto-merge deliberately. Dependabot PRs run with a read-only token and without regular repository secrets, so tests must not require them.

Keep auto-merge limited to low-risk updates (for example patch updates with passing required checks) and require status checks via branch protection.

### Lesson 9.8: GitHub's starter workflows

In the **Actions** tab, **New workflow** shows templates (for Node, Docker, deployments to clouds, CodeQL, and more). They are good starting points because they encode the vendor's recommended steps. Before committing one: update the action versions, add `permissions`, narrow triggers, remove sample steps you don't need, and read every line. They are examples, not audited production pipelines.

### Lesson 9.9: Design principles

| Principle | Meaning |
|---|---|
| Fast feedback | Run cheap checks first; cancel superseded runs; cache |
| Fail clearly | Name steps; keep logs readable; upload reports |
| Least privilege | `permissions`, scoped secrets, OIDC |
| Idempotent and re-runnable | A re-run should be safe |
| Same commands everywhere | Developers can run the exact command locally |
| Thin YAML | Logic in scripts (Chapter 10) |
| Right trigger | Don't deploy on `pull_request` from forks |

## 3. Mental model

A good pipeline is a **funnel**: cheap, fast checks at the top catch most problems; expensive, risky steps (deploys) are at the bottom and are protected by approvals. Every item below the neck assumes the top checks passed (`needs:`).

## 4. Syntax breakdown

| Pattern | Key syntax |
|---|---|
| Don't push images from PRs | `push: ${{ github.event_name != 'pull_request' }}` |
| Run on tags | `on: push: tags: ['v*']` |
| Daily cron | `schedule: - cron: 'M H * * *'` (UTC) |
| Gate production | `environment: production` (required reviewers in Settings) |
| One deploy at a time | `concurrency: group: deploy-…, cancel-in-progress: false` |
| Keep reports on failure | `if: always()` on upload steps |
| Push tags/releases | `permissions: contents: write` |

## 5. Practical example

A combined `ci.yml` using the funnel: lint and test in parallel, then build only if both pass, and only push the image from `main`.

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
  lint:
    runs-on: ubuntu-24.04
    timeout-minutes: 10
    steps:
      - uses: actions/checkout@v5
      - uses: actions/setup-node@v4
        with: { node-version: 22, cache: npm }
      - run: npm ci && npm run lint
  test:
    runs-on: ubuntu-24.04
    timeout-minutes: 15
    steps:
      - uses: actions/checkout@v5
      - uses: actions/setup-node@v4
        with: { node-version: 22, cache: npm }
      - run: npm ci && npm test
  build:
    needs: [lint, test]
    runs-on: ubuntu-24.04
    timeout-minutes: 15
    permissions:
      contents: read
      packages: write
    steps:
      - uses: actions/checkout@v5
      - uses: docker/login-action@v3
        if: github.event_name != 'pull_request'
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      - uses: docker/build-push-action@v6
        with:
          context: .
          push: ${{ github.event_name != 'pull_request' }}
          tags: ghcr.io/${{ github.repository }}:${{ github.sha }}
```

## 6. Execution walkthrough

On a pull request: `lint` and `test` start in parallel on separate runners. If both succeed, `build` starts, builds the image, skips login and does **not** push. On a merge to `main`: the same three jobs run; `build` logs in to GHCR with the temporary token and pushes `ghcr.io/<owner>/<repo>:<sha>`. A second push to the same branch while the first run is going cancels the first run.

## 7. Common mistakes

| Mistake | Fix |
|---|---|
| Triggering lint on both `push` and `pull_request` for all branches (double runs) | Limit `push` to `main` |
| `${{ secrets.KUBECONFIG }}` pasted into a `run:` script | Use `env:` and a protected temp file |
| Using archived `actions/create-release` | `gh release create` or a maintained action |
| Believing Dependabot closes failing PRs | It does not; failing checks block merge only if branch protection requires them |
| `actions/stale` with short timeouts and no exemptions | Generous timers, exempt labels |
| Deploying from `pull_request` | Deploy from protected branches, tags or manual dispatch |
| Pushing `latest` only | Immutable tags (SHA or version) |
| Copying starter workflows unchanged | Update versions, add `permissions` |

## 8. Security considerations

- Deploy workflows are the highest-value target: restrict who can trigger them, use environments with reviewers, OIDC, minimal roles, and `concurrency` serialization.
- Scheduled and maintenance workflows with `write` permissions are risky if the action is compromised; pin them.
- Do not run untrusted PR code with secrets (Chapter 11).
- A release workflow with `contents: write` can modify tags and releases; limit to tag pushes from trusted maintainers.

## 9. Hands-on exercise

**Goal:** build the CI funnel and one maintenance workflow in your practice repository.

1. Use a small Node project (`npm init -y`, add `"lint"` and `"test"` scripts that just run `echo`). Add a trivial `Dockerfile` (`FROM alpine:3.20`).
2. Add the combined `ci.yml` above on a branch; `actionlint`; open a PR. Confirm the image is built but not pushed.
3. Merge. Confirm a package appears under the repository's **Packages**.
4. Add `.github/dependabot.yml` for `github-actions` and `npm`.
5. Add the release workflow; create a tag in the practice repo (`git tag v0.0.1 && git push origin v0.0.1`) and confirm a release is created.
6. Add the stale workflow with `workflow_dispatch` temporarily and trigger it. Remove the manual trigger after verifying behavior with a throwaway issue.

**Success looks like:** green CI, an image in GHCR with the SHA tag, a release for `v0.0.1`.

**If it fails:** `denied: permission_denied` pushing to GHCR → missing `packages: write`; release job 403 → missing `contents: write`.

## 10. Verification checklist

- [ ] Each workflow has `permissions`, `timeout-minutes` and sensible `concurrency`.
- [ ] PRs build but don't push.
- [ ] Release uses `gh`/a maintained action, not `create-release`.
- [ ] No secret is interpolated into a script body.
- [ ] Dependabot PRs appear and run CI.

## 11. Knowledge check

1. Why design a workflow starting from the local command?
2. How do you build an image on PRs without pushing it?
3. Why is `${{ secrets.KUBECONFIG }}` inside `run:` a bad idea?
4. What happens to a Dependabot PR whose tests fail?
5. Which permission does a job need to push an image to GHCR? To create a release?
6. When is GitOps preferable to push-based deployment?

<details>
<summary>Answers</summary>

1. So it's debuggable locally and the workflow is just a wrapper.
2. `push: ${{ github.event_name != 'pull_request' }}`.
3. The secret is expanded into script text (leak risk, injection pattern); pass it via `env:`.
4. Nothing automatic: the check shows failed; it blocks merge only if required.
5. `packages: write`; `contents: write`.
6. Many services/clusters, desire to keep production credentials inside the cluster, auditability via Git.
</details>

## 12. Key takeaways

- Wrap local commands; layer on caching, matrix and concurrency.
- Build on PRs, publish on `main`/tags.
- Deploy through protected paths: environments, OIDC, serialized runs.
- Automate chores, but conservatively.
- Starter workflows are starting points.

## Sources and further reading

- Original notes: `git show master:09_common_workflows.md`. Corrections: injection-prone kubeconfig interpolation, the false claim that Dependabot closes failing PRs, archived `actions/create-release`, outdated `release-please` reference and `actions/stale@v8`, outdated artifact/cache versions. Starter-template walkthroughs (Docker, ECS) were condensed into Lesson 9.8 because they duplicate GitHub's own templates, which change often.
- Official docs: [Workflow syntax](https://docs.github.com/en/actions/reference/workflows-and-actions/workflow-syntax) · [Authenticate with GITHUB_TOKEN](https://docs.github.com/en/actions/tutorials/authenticate-with-github_token) · [Reusable workflows](https://docs.github.com/en/actions/reference/workflows-and-actions/reusable-workflows)
- Not verified here: archived status of `actions/create-release` and current major version of `googleapis/release-please-action` (from knowledge; confirm on the repositories).
