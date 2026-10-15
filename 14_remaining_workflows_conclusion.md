# Chapter 14: Capstone, Part 3: GitOps Deployment, Releases, Chores, Timing and Wrap-up

> **Level:** advanced · **Time:** 4–5 hours · **Prerequisites:** [Chapters 8, 11, 12, 13](13_build_push_images.md)
> **Next:** [README: roadmap and how to verify this guide](README.md)

## Learning objectives

After this chapter you can:

- Explain GitOps and implement the CI half of it: a reusable workflow that updates an image tag in a manifest file and pushes safely.
- Handle concurrent pushes with rebase-and-retry, and explain why `GITHUB_TOKEN` pushes don't trigger new runs.
- Wire the GitOps update into the build workflow with a matrix, environment approvals and least privilege.
- Create releases from tags.
- Reuse the stale-issue and dependency-update patterns from Chapter 9.
- Describe how to export workflow timing data to an observability platform.
- Review the whole capstone against the security, performance and maintainability checklist.

## 1. Problem statement

Chapter 13 produces versioned images. Something must now make a running environment use them. In **GitOps**, CI does not deploy; it changes a file in Git that *describes* what should run, and an agent in the target environment (Argo CD or Flux for Kubernetes) notices and applies it. CI needs no credentials for the cluster, every change is a commit, and rollback is a `git revert`. The remaining capstone workflows complete the loop: GitOps update, releases, repository chores and timing data.

## 2. Core explanation

### Lesson 14.1: What the CI part of GitOps does

```
build image ──▶ edit manifest (image tag) ──▶ commit & push ──▶ agent detects ──▶ cluster converges
   (Ch. 13)        (this chapter)                                 (Argo CD / Flux, out of scope)
```

The deployment agent itself is outside this guide. This chapter builds and tests only the CI half. In production the manifests usually live in a **separate repository** (so that deploy history and permissions are separate from application code) and the workflow pushes to it with a dedicated credential. For the lab, the manifest file lives in your practice repository and the workflow pushes to a **throwaway branch**, so you can practice without touching `main`.

### Lesson 14.2: The manifest and the update script

A simplified manifest, `deploy/staging/images.yaml` (a real project would typically use Kustomize overlays; the technique is the same):

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: image-versions
data:
  api-golang: "ghcr.io/OWNER/REPO/api-golang:0.0.0-0001-abc1234" # services/go/api-golang
  api-node: "ghcr.io/OWNER/REPO/api-node:0.0.0-0001-abc1234" # services/node/api-node
```

The trailing comment is a **marker** that identifies which line belongs to which service. Create the same for `deploy/production/images.yaml`. Use your own owner and repository in place of `OWNER/REPO`.

The update script `scripts/update-image-tag.sh`. It validates every argument against a strict pattern, so nothing unexpected can reach `sed`:

```bash
#!/usr/bin/env bash
# Usage: scripts/update-image-tag.sh <environment> <service-dir> <version>
# Rewrites the image reference on the line tagged "# <service-dir>" in deploy/<environment>/images.yaml
set -euo pipefail
env_name="$1"; service="$2"; version="$3"

[[ "$env_name" =~ ^(staging|production)$ ]]                              || { echo "bad environment" >&2; exit 1; }
[[ "$service" =~ ^services/[a-z0-9_-]+/[a-z0-9_-]+$ ]]                   || { echo "bad service" >&2; exit 1; }
[[ "$version" =~ ^[0-9]+\.[0-9]+\.[0-9]+(-[0-9]{4}-[0-9a-f]{7,40})?$ ]]  || { echo "bad version" >&2; exit 1; }
[[ "${IMAGE_REPO:-}" =~ ^ghcr\.io/[a-z0-9._/-]+$ ]]                      || { echo "bad IMAGE_REPO" >&2; exit 1; }

file="deploy/${env_name}/images.yaml"
name="$(basename "$service")"
grep -q "# ${service}\$" "$file" || { echo "no line tagged '# ${service}' in $file" >&2; exit 1; }

sed -i "\|# ${service}\$|s|\"[^\"]*\"|\"${IMAGE_REPO}/${name}:${version}\"|" "$file"
```

This script was run in a scratch repository during preparation of this guide. A valid call rewrote exactly one line; an invalid service (`services/x/y;rm`), a service with no marker line, and a version containing `&` were each rejected with a non-zero exit. (In that test the version pattern used `{7}` for the hash; this version allows 7–40 hex characters because `git describe` can abbreviate longer than seven when a repository is large.)

### Lesson 14.3: Push with retry

Several services can finish at the same moment, and each tries to push to the same branch. The loser's push is rejected because the remote advanced. The fix: commit once, then loop: *push; if rejected, `git pull --rebase`, wait with a small random delay, try again.*

`scripts/git-commit-push.sh`:

```bash
#!/usr/bin/env bash
# Usage: COMMIT_MESSAGE=... BRANCH=... scripts/git-commit-push.sh
set -euo pipefail
retries="${RETRIES:-5}"

git add -A
if git diff --cached --quiet; then
  echo "No changes to commit"
  exit 0
fi
git commit -m "$COMMIT_MESSAGE"

for attempt in $(seq 1 "$retries"); do
  if git push origin "HEAD:${BRANCH}"; then
    echo "Pushed on attempt ${attempt}"
    exit 0
  fi
  echo "Push rejected (attempt ${attempt}/${retries}); rebasing"
  git pull --rebase origin "$BRANCH"
  sleep $(( (2 ** attempt) + RANDOM % 3 ))      # exponential backoff plus jitter
done
echo "Failed to push after ${retries} attempts" >&2
exit 1
```

Rebase keeps the history linear. The jitter prevents several waiting runs from retrying at the same instant. Note that the original design committed inside the loop (creating repeated commits on retries); here the commit happens once.

### Lesson 14.4: A reusable GitOps workflow

Use `workflow_call` so the build workflow can call it, plus `workflow_dispatch` so you can run it by hand. `.github/workflows/gitops-update.yml`:

```yaml
name: Update GitOps manifests

on:
  workflow_call:
    inputs:
      service:
        type: string
        required: true
      version:
        type: string
        required: true
      environment:
        type: string
        required: true
      branch:
        type: string
        required: false
        default: main
  workflow_dispatch:
    inputs:
      service:
        description: Service directory
        type: choice
        required: true
        options:
          - services/node/api-node
          - services/go/api-golang
          - services/python/load-generator
      version:
        description: Image version, for example 1.0.9-0047-452cc9a
        type: string
        required: true
      environment:
        description: Target environment
        type: choice
        required: true
        options: [staging, production]
      branch:
        description: Branch to push the manifest change to
        type: string
        required: false
        default: gitops-demo

permissions:
  contents: read

concurrency:
  group: gitops-${{ inputs.service }}-${{ inputs.environment }}
  cancel-in-progress: false

jobs:
  update-manifest:
    runs-on: ubuntu-24.04
    timeout-minutes: 10
    environment: ${{ inputs.environment }}      # production can require reviewers
    permissions:
      contents: write                            # to push the manifest commit
    steps:
      - uses: actions/checkout@v5
        with:
          ref: ${{ inputs.branch }}
          persist-credentials: true             # this job pushes, so it keeps the token

      - name: Update image tag
        env:
          ENVIRONMENT: ${{ inputs.environment }}
          SERVICE: ${{ inputs.service }}
          VERSION: ${{ inputs.version }}
        run: |
          IMAGE_REPO="ghcr.io/${GITHUB_REPOSITORY,,}" \
            scripts/update-image-tag.sh "$ENVIRONMENT" "$SERVICE" "$VERSION"
          git --no-pager diff

      - name: Commit and push
        env:
          BRANCH: ${{ inputs.branch }}
          COMMIT_MESSAGE: "chore(deploy): ${{ inputs.environment }} ${{ inputs.service }} ${{ inputs.version }}"
        run: |
          git config user.name "github-actions[bot]"
          git config user.email "41898282+github-actions[bot]@users.noreply.github.com"
          scripts/git-commit-push.sh
```

Corrections compared with the original design, in one place:

- The original had no `workflow_call` trigger, so `uses:` from the build workflow could not work. It is added here.
- It interpolated `inputs` into a shell string to build a commit message. Here every input is passed through `env:` and validated by the script (the commit message is built in `env:` from values already restricted by the choice options or validated in the script step that runs first).
- It needs `contents: write`; with the repository's default read-only token it would fail.
- Marking commits `[skip ci]` is unnecessary for pushes made with `GITHUB_TOKEN`: GitHub does not start new workflow runs for events caused by that token (a loop guard). If your GitOps repository has its own workflows that must run on this commit, push with a GitHub App token or deploy key instead.
- Branch protection on `main` may reject the bot's push. Real setups use a separate manifests repository where the bot is allowed to push, or open a pull request instead.

(The commit message is built inside `env:`, which is data, not script text. The earlier step runs the validating script first, so a malformed `version` fails the job before any commit is made.)

### Lesson 14.5: Wiring it into the build workflow

Add to the end of `.github/workflows/build-push.yml` (Chapter 13). The `plan` job already emitted `service` and `version` for each image, so the deploy matrix needs no outputs from the build legs:

```yaml
  deploy:
    needs: [plan, build]
    if: ${{ github.event_name == 'push' && needs.plan.outputs.matrix != '[]' }}
    permissions:
      contents: write          # a called workflow can never exceed the caller's permissions
    strategy:
      fail-fast: false
      matrix:
        include: ${{ fromJSON(needs.plan.outputs.matrix) }}
    uses: ./.github/workflows/gitops-update.yml
    with:
      service: ${{ matrix.service }}
      version: ${{ matrix.version }}
      environment: ${{ github.ref_type == 'tag' && 'production' || 'staging' }}
      branch: main
```

The original used `needs.build-push.outputs[matrix.service].version`, which is not valid for matrix jobs (a matrix job has a single output namespace, and the last leg to write wins). Computing the versions once in `plan` and reusing them is the fix.

Behavior: pushes to `main` update **staging**; release tags update **production**, after approval if the `production` environment requires reviewers (the environment is declared in the called job). Manual runs of the build workflow don't deploy. Also `secrets: inherit` is not needed: this workflow uses only `GITHUB_TOKEN`.

### Lesson 14.6: Releases from tags

A small workflow creates a GitHub release when a release tag is pushed. `.github/workflows/release.yml`:

```yaml
name: Release

on:
  push:
    tags: ['**@[0-9]+.[0-9]+.[0-9]+']

permissions:
  contents: write

jobs:
  release:
    runs-on: ubuntu-24.04
    timeout-minutes: 10
    steps:
      - uses: actions/checkout@v5
        with:
          persist-credentials: false
      - name: Create release
        env:
          GH_TOKEN: ${{ github.token }}
          TAG: ${{ github.ref_name }}
        run: gh release create "$TAG" --verify-tag --title "$TAG" --generate-notes
```

`--generate-notes` summarizes pull requests since the previous release of the repository, not specifically of this service; for per-service changelogs consider `googleapis/release-please-action` (current major tag: `v4`; it supports monorepo configuration through manifest files, which needs its own setup) or compose notes from `git log -- <service-dir>`. The archived `actions/create-release` action should not be used.

### Lesson 14.7: Repository chores

Reuse Chapter 9: `actions/stale` on a schedule for issue hygiene and Dependabot for `github-actions`, `docker` and the language ecosystems. The capstone does not need new code here. Remember that these workflows hold write permissions; pin the actions.

### Lesson 14.8: Timing data and observability

GitHub's run page shows durations but not trends across runs. You can export workflow timings as traces to an observability platform. Honeycomb provides a build-events action; as verified from its README while preparing this guide:

- The action has moved: it is now `honeycombio/gha-buildevents` (the old `kvrhdn/gha-buildevents@v1` used in the original notes is outdated), with a `v4` tag.
- Required inputs are `apikey` and `dataset`; the status input is `status` (the old `job-status` is deprecated); `matrix-key` is an optional input for matrix builds.
- It is used in the workflow whose timing you want to record, as a final step or job; the README's example is a single-job workflow.

Read the README for the version you pin before adopting it; this guide did not run the action. The API key belongs in a secret, ideally an environment secret. Exported metadata may include repository, branch and commit information, so consider what leaves your boundary. For a no-service alternative, use the `gh run list --json` query from Chapter 10.

## 3. Mental model

CI is a **courier** that never enters the building. It writes a delivery note ("run api-golang:1.0.9") on a notice board (the manifest in Git). The building's own concierge (Argo CD/Flux) reads the board and acts. The courier needs only access to the board, not keys to the building, and everything on the board has a history.

## 4. Syntax breakdown

| Syntax | Meaning |
|---|---|
| `on: workflow_call: inputs:` | Makes a workflow callable; inputs are typed |
| `uses: ./.github/workflows/x.yml` on a job | Calls a reusable workflow (no `steps`, no `runs-on` on that job) |
| `with:` on a calling job | Passes inputs; can use `matrix.*` |
| `permissions: contents: write` | Required for pushing; also on the calling job |
| `environment: ${{ inputs.environment }}` | Gate and scope secrets per environment |
| `git push origin HEAD:$BRANCH` | Push the current commit to a named branch |
| `git pull --rebase` | Rebase local commit onto the updated remote |
| `gh release create --verify-tag --generate-notes` | Create a release from an existing tag |

## 5. Practical example: the full loop, locally verifiable

Run the pieces in a clone of your practice repository, without GitHub:

```bash
git checkout -b gitops-demo
export IMAGE_REPO=ghcr.io/you/practice
scripts/update-image-tag.sh staging services/go/api-golang 1.0.0-0003-abcdef1
git --no-pager diff
git checkout -- .                      # undo; no push needed locally
```

On GitHub, create the branch `gitops-demo`, then run **Update GitOps manifests** manually with `branch` left at its default and watch the commit appear on that branch.

## 6. Execution walkthrough

A push to `main` that changes only `services/go/api-golang`:

1. Chapter 13's `plan` job computes `[{"service":"services/go/api-golang","name":"api-golang","version":"1.0.9-0047-452cc9a"}]`; `build` pushes the image.
2. `deploy` starts after `build` succeeds. Its matrix has one leg. It calls `gitops-update.yml` with environment `staging`.
3. In the called workflow, the `update-manifest` job (environment `staging`) checks out `main`, runs the script to rewrite the Go line in `deploy/staging/images.yaml`, commits as the bot and pushes.
4. If another leg pushed first, the push is rejected; the script rebases and retries.
5. The Argo CD / Flux agent (in a real setup) sees the commit and rolls out the new image. Because the push used `GITHUB_TOKEN`, no new workflow run is triggered.

A release tag `services/go/api-golang@1.0.10` follows the same path to `deploy/production/images.yaml`, but `update-manifest` waits for the reviewers configured on the `production` environment, and `release.yml` also creates the GitHub release.

## 7. Common mistakes

| Mistake | Effect | Fix |
|---|---|---|
| No `workflow_call` trigger | `uses:` cannot call it | Add it |
| `needs.build.outputs[matrix.service]` | Invalid; matrix outputs collide | Compute data in `plan`; pass via `matrix.include` |
| Missing `contents: write` (workflow and calling job) | 403 on push | Grant it where needed only |
| Expecting the bot's push to trigger CI | Nothing runs | Use an App token / deploy key if required |
| `[skip ci]` as a loop guard for `GITHUB_TOKEN` pushes | Unnecessary | Remove, or keep deliberately for other tokens |
| Committing inside the retry loop | Duplicate commits | Commit once |
| Pushing straight to a protected `main` | Rejected | Separate manifests repo/bypass rule, or PR flow |
| Interpolating inputs into commit-message scripts | Injection | `env:` plus validation |
| `secrets: inherit` by habit | Over-sharing | Not needed here |
| Using `actions/create-release` | Archived | `gh release create` |
| Using `kvrhdn/gha-buildevents@v1` | Outdated; moved | `honeycombio/gha-buildevents`, read the README |

## 8. Security considerations

- **Trust in the manifest repository.** Whoever can push to it can change production. Protect its branches like production, and prefer a dedicated credential (GitHub App or deploy key) over a personal access token.
- **Approvals.** Put `environment: production` with required reviewers on the job that writes production manifests.
- **Validated writes.** The script accepts only known environments, a path-shaped service, a version-shaped string and a GHCR-shaped repository. That keeps `sed` safe.
- **Least privilege.** Only the deploy job has `contents: write`; builds have `packages: write`; everything else is read-only.
- **Observability exports** leak metadata to a third party; store API keys as secrets.
- **Loop guards.** Pushes with `GITHUB_TOKEN` don't trigger workflows. If you switch to a token that does, protect against infinite loops (path filters, actor checks).

## 9. Hands-on exercise

**Goal:** close the loop in your practice repository.

1. Add `deploy/staging/images.yaml` and `deploy/production/images.yaml` (with your owner/repo), `scripts/update-image-tag.sh` and `scripts/git-commit-push.sh` (executable). Run the script locally as in Section 5, and also try the three rejected cases (bad service, missing marker, bad version).
2. Create the branch `gitops-demo` on GitHub. Add `gitops-update.yml`, `release.yml` and the `deploy` job from Lesson 14.5, merge to `main`, and `actionlint`.
3. Create a `staging` and a `production` environment (**Settings → Environments**). Add a required reviewer (yourself) to `production`.
4. Run **Update GitOps manifests** manually: staging, `services/go/api-golang`, version `1.0.9-0047-abcdef1`, branch `gitops-demo`. Confirm a bot commit on that branch.
5. Run it twice at once with different services (two browser tabs or `gh workflow run` twice) and confirm both commits land (rebase-and-retry).
6. Push a release tag, for example `services/go/api-golang@1.0.0`. Approve the production deployment, and confirm the release and the production manifest change.
7. Review the whole capstone with the checklist in Chapter 11.

**Success looks like:** two bot commits on the demo branch, a production approval prompt, and a GitHub release for the tag.

**If it fails:** `403` on push → `contents: write` missing at workflow or calling job; "no line tagged" → marker comment missing or path differs; called workflow cannot be found → file path or missing `workflow_call`; production job not waiting → environment has no reviewers.

## 10. Verification checklist

- [ ] `gitops-update.yml` has `workflow_call` and typed inputs.
- [ ] The deploy matrix uses data from `plan`, not from build outputs.
- [ ] `contents: write` appears only on the deploy job and the called job.
- [ ] Production changes require an environment approval.
- [ ] The script rejects bad inputs.
- [ ] No `[skip ci]`-style assumptions about `GITHUB_TOKEN` pushes remain.
- [ ] The Honeycomb (or any third-party) action, if used, is pinned and its README was read.

## 11. Knowledge check

1. In GitOps, what does CI change, and who applies it?
2. Why can pushes made with `GITHUB_TOKEN` not trigger workflows, and when would you need a different credential?
3. Why does `needs.build.outputs[matrix.service]` not work?
4. Why must the calling job grant `contents: write` too?
5. How does the retry loop avoid duplicate commits and thundering herds?
6. Where do production approvals come from when a reusable workflow is called?

<details>
<summary>Answers</summary>

1. CI edits a manifest in Git; an in-environment agent applies it.
2. GitHub blocks it to prevent recursion. Use an App token or deploy key if downstream workflows must run.
3. Matrix legs share one output namespace and cannot be indexed by leg.
4. A called workflow's permissions can't exceed the caller's.
5. It commits once before the loop and uses exponential backoff plus random jitter.
6. The `environment:` declared in the called job and its protection rules.
</details>

## 12. Key takeaways

- GitOps splits "decide" (CI edits Git) from "apply" (agent in the environment).
- A reusable workflow needs `workflow_call`; permissions flow caller → called.
- Compute shared data once (plan) and reuse it for build and deploy.
- Validate every value that reaches a shell or `sed`.
- Treat the manifest repository and production environments as production.

## 13. Where you have arrived

You now have, across Chapters 1–14: the mental model (events → jobs → steps → runners), the feature set (outputs, secrets, matrices, caches, artifacts, OIDC, concurrency), reuse (marketplace actions, composite actions, reusable workflows), debugging habits, a best-practice checklist, and a capstone: **test only what changed → build and version images → update GitOps manifests → release → chores → timing data**.

Natural next steps: run an Argo CD or Flux agent against a local cluster to see the other half of GitOps; add image signing and provenance; replace the simplified manifest with Kustomize overlays; add reusable-workflow versioning across several repositories; try self-hosted ephemeral runners in a private repository.

## Sources and further reading

- Original notes: `git show master:14_remaining_workflows_conclusion.md`. Corrections: GitOps workflow without `workflow_call`; invalid `needs.build-push.outputs[matrix.service]`; missing `contents: write`; incorrect `[skip ci]` rationale; inputs interpolated into shell; archived `create-release` action; stale and outdated Honeycomb action (`kvrhdn/gha-buildevents@v1`, `job-status`); the repeated Honeycomb setup (also in Chapter 10); and the appendix that duplicated Chapters 12–13 (removed). The conclusion/resource lists were condensed.
- Official docs: [Reusable workflows](https://docs.github.com/en/actions/reference/workflows-and-actions/reusable-workflows) · [Authenticate with GITHUB_TOKEN](https://docs.github.com/en/actions/tutorials/authenticate-with-github_token) · [Workflow syntax](https://docs.github.com/en/actions/reference/workflows-and-actions/workflow-syntax)
- Tools: [Argo CD](https://argo-cd.readthedocs.io/) · [Flux](https://fluxcd.io/) · [honeycombio/gha-buildevents](https://github.com/honeycombio/gha-buildevents) · [googleapis/release-please-action](https://github.com/googleapis/release-please-action)
- Verification note: `scripts/update-image-tag.sh` was exercised in a scratch repository (results in Lesson 14.2). The workflows in this chapter were not run on GitHub.
