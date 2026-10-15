# Chapter 13: Capstone, Part 2: Build, Version and Push Container Images

> **Level:** advanced · **Time:** 4–5 hours · **Prerequisites:** [Chapters 6, 8, 11, 12](12_test_workflow.md); Docker basics
> **Next:** [Chapter 14: Capstone, Part 3: GitOps, Releases and the Rest](14_remaining_workflows_conclusion.md)

## Learning objectives

After this chapter you can:

- Design a versioning scheme that separates production releases from development builds.
- Compute versions from Git tags with `git describe`, per service in a monorepo.
- Build a **plan** job that outputs a matrix of objects (service, image name, version).
- Build multi-architecture images with Buildx, push them to GitHub Container Registry (GHCR) using only `GITHUB_TOKEN`, and cache layers.
- Trigger builds from pushes to `main`, release tags, and manual dispatch, while keeping untrusted text out of shell scripts.

## 1. Problem statement

After tests pass, every changed service needs an immutable, traceable container image. The pipeline must answer: *which services changed? what version is each? is this a release or a development build? where does the image go?* It must do so without pushing from untrusted contexts and without hard-coding anything per service.

## 2. Core explanation

### Lesson 13.1: Versioning scheme

| Build kind | Trigger | Version | Example |
|---|---|---|---|
| **Release** (production) | Push of a Git tag `<service-path>@x.y.z` | `x.y.z` | `1.0.9` |
| **Development** (non-production) | Push to `main`, manual dispatch | `x.y.z-NNNN-<sha>` | `1.0.9-0047-452cc9a` |

`NNNN` is the number of commits since the last release tag of that service, **zero-padded to four digits** so that tools that sort tags alphabetically order them correctly (`0003` before `0011`). `<sha>` is the short commit hash, which links an image to its source.

Because this is a monorepo, each service has its own release tags. A tag looks like `services/go/api-golang@1.0.9`. (Git allows slashes and `@` in tag names.) To find the last release of one service, `git describe` must be limited to that service's tags with `--match`:

```bash
git describe --tags --long --match 'services/go/api-golang@*'
# services/go/api-golang@1.0.9-47-g452cc9a
```

The version script below (`scripts/image-version.sh`) wraps this. It was run against a scratch repository during preparation of this guide with these results: no tag → `0.0.0-0001-<sha>`; at the tag → `1.0.9` (release mode) and `1.0.9-0000-<sha>` (development mode); 47 commits later → `1.0.9-0047-<sha>`; release mode on an untagged commit → fails, as intended.

```bash
#!/usr/bin/env bash
# Usage: scripts/image-version.sh <service-dir> <production|nonproduction>
set -euo pipefail
service="$1"
mode="$2"

if [ "$mode" = "production" ]; then
  # The commit must carry a release tag named <service>@x.y.z
  tag="$(git describe --tags --exact-match --match "${service}@*" HEAD)"
  printf '%s\n' "${tag#"${service}@"}"
  exit 0
fi

# Non-production: x.y.z-NNNN-<short sha> (commits since last release, zero-padded)
if desc="$(git describe --tags --long --match "${service}@*" 2>/dev/null)"; then
  rest="${desc#"${service}@"}"          # 1.0.9-47-g452cc9a
  sha="${rest##*-g}"
  rest="${rest%-g*}"                    # 1.0.9-47
  count="${rest##*-}"
  version="${rest%-*}"                  # 1.0.9
else
  version="0.0.0"
  count="$(git rev-list --count HEAD)"
  sha="$(git rev-parse --short=7 HEAD)"
fi
printf '%s-%04d-%s\n' "$version" "$count" "$sha"
```

Notes: it requires full history and tags, so the workflow checks out with `fetch-depth: 0`. A pre-release suffix in the tag (for example `1.0.0-rc1`) is outside this simple scheme; add handling if you need it.

### Lesson 13.2: Triggers, and one `push` key

All push-related triggers belong under a **single** `push:` key. The original notes defined `push:` twice, which silently discards one of them (or invalidates the file).

```yaml
on:
  workflow_dispatch:
    inputs:
      service:
        description: Service directory to build
        type: choice
        required: true
        options:
          - services/node/api-node
          - services/go/api-golang
          - services/python/load-generator
      push:
        description: Push the image to the registry
        type: boolean
        default: false
  push:
    branches: [main]
    paths: ['services/**']
    tags: ['**@[0-9]+.[0-9]+.[0-9]+']
```

Points to understand:

- `paths:` is **not evaluated for tag pushes**, so the tag trigger fires regardless.
- The tag pattern needs `**` because the tag contains slashes and `*` does not match `/`. The original pattern `*@…` would not match `services/go/api-golang@1.0.9`. `+` means "one or more of the preceding character or class" in GitHub's glob syntax, and `[0-9]` is a character class. Always re-validate the tag in a script; globs are coarse.
- Manual runs default to **not pushing**, so you can rehearse safely.

### Lesson 13.3: A plan job that outputs a matrix of objects

In Chapter 12, matrix legs only needed a path. Now each leg needs a path, an image name and a version, and Chapter 14 needs the same data after the builds. Matrix legs cannot give separate outputs under one name, so compute everything **up front** in a `plan` job and emit an array of objects:

```json
[{"service":"services/go/api-golang","name":"api-golang","version":"1.0.9-0047-452cc9a"}]
```

Both the build matrix and (in Chapter 14) the deploy matrix consume it with `include:`.

The `plan` job:

1. gets the candidate services from the event (tag, manual input, or paths-filter result),
2. validates each (shape of path, directory exists, has a `Dockerfile`),
3. computes versions with the script,
4. emits `matrix` and `environment`.

### Lesson 13.4: Build, cache and push

| Piece | Purpose |
|---|---|
| `docker/setup-qemu-action` | Emulation so an x86 runner can build Arm images (slow). Not needed if you build on native Arm runners. |
| `docker/setup-buildx-action` | Installs the Buildx builder (multi-platform, advanced caching) |
| `docker/login-action` | Logs in to a registry. For GHCR: username `github.actor`, password `GITHUB_TOKEN`, job needs `packages: write` |
| `docker/build-push-action` | Builds and optionally pushes. Takes `context`, `tags`, `platforms`, `push`, `cache-from`, `cache-to` |

`cache-to: type=gha,mode=max` stores all layers (including intermediate stages of multi-stage builds) in the GitHub Actions cache service; `cache-from: type=gha` restores them. This counts against the repository's cache quota (Chapter 6).

**Registry choice.** The course notes pushed to Docker Hub, which needs a Docker Hub account, a personal access token as a secret and a `DOCKERHUB_USERNAME` variable. This guide uses **GHCR** because it needs no extra account or long-lived secret. To use Docker Hub instead, change `registry`, the `login-action` credentials and the image name, and store the token as a secret scoped to the job's environment.

**Image names.** GHCR requires lowercase names, and `github.repository` may contain capitals, so lowercase it in a script: `ghcr.io/<owner>/<repo>/<service-name>`.

### Lesson 13.5: Scaffold additions

Add a `Dockerfile` to each scaffold service from Chapter 12:

`services/node/api-node/Dockerfile`:

```dockerfile
FROM node:22-alpine
WORKDIR /app
COPY package*.json ./
RUN npm ci --omit=dev
COPY . .
CMD ["node", "-e", "console.log('api-node up')"]
```

`services/go/api-golang/Dockerfile`:

```dockerfile
FROM golang:1.23-alpine AS build
WORKDIR /src
COPY . .
RUN go build -o /out/app .

FROM alpine:3.20
COPY --from=build /out/app /app
CMD ["/app"]
```

`services/python/load-generator/Dockerfile`:

```dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY . .
CMD ["python", "-c", "print('load-generator up')"]
```

Create `.github/utils/build-filters.yml` (same shape as Chapter 12's file, but referencing the build workflow):

```yaml
services/node/api-node:
  - services/node/api-node/**
  - .github/workflows/build-push.yml
services/go/api-golang:
  - services/go/api-golang/**
  - .github/workflows/build-push.yml
services/python/load-generator:
  - services/python/load-generator/**
  - .github/workflows/build-push.yml
```

## 3. Mental model

A **factory with a planning office and an assembly line**. The planning office (`plan`) reads the order (a tag, a button press, a diff), decides which products to make, and stamps each with its serial number (version). The assembly line (matrix) builds each product in parallel, using the serial number it was given. Nothing on the line decides anything; this is why the same plan can later drive deployment.

## 4. Syntax breakdown

| Syntax | Meaning |
|---|---|
| `matrix: include: ${{ fromJSON(...) }}` | Each array element becomes a leg with its own properties (`matrix.service`, `matrix.name`, `matrix.version`) |
| `github.ref_type == 'tag'` | True for tag pushes (use through `env:`, see below) |
| `${REF_NAME%@*}` | Shell: drop `@…` to get the service path from a tag |
| `${GITHUB_REPOSITORY,,}` | Bash: lowercase |
| `fetch-depth: 0` | Full history and tags |
| `push: ${{ github.event_name != 'workflow_dispatch' \|\| inputs.push }}` | Push on automatic triggers; on manual runs only if the box is checked |
| `permissions: packages: write` | Lets `GITHUB_TOKEN` push to GHCR |

## 5. Practical example: `.github/workflows/build-push.yml`

```yaml
name: Build and Push Images

on:
  workflow_dispatch:
    inputs:
      service:
        description: Service directory to build
        type: choice
        required: true
        options:
          - services/node/api-node
          - services/go/api-golang
          - services/python/load-generator
      push:
        description: Push the image to the registry
        type: boolean
        default: false
  push:
    branches: [main]
    paths: ['services/**']
    tags: ['**@[0-9]+.[0-9]+.[0-9]+']

permissions:
  contents: read

concurrency:
  group: build-${{ github.ref }}
  cancel-in-progress: false

jobs:
  plan:
    runs-on: ubuntu-24.04
    timeout-minutes: 10
    outputs:
      matrix: ${{ steps.plan.outputs.matrix }}
      environment: ${{ steps.plan.outputs.environment }}
    steps:
      - uses: actions/checkout@v5
        with:
          fetch-depth: 0
          persist-credentials: false

      - uses: dorny/paths-filter@de90cc6fb38fc0963ad72b210f1f284cd68cea36  # v3.0.2
        id: changed
        if: github.event_name == 'push' && github.ref_type == 'branch'
        with:
          filters: .github/utils/build-filters.yml

      - name: Plan builds
        id: plan
        env:
          EVENT: ${{ github.event_name }}
          REF_TYPE: ${{ github.ref_type }}
          REF_NAME: ${{ github.ref_name }}      # tag and branch names are user-chosen: via env only
          INPUT_SERVICE: ${{ inputs.service }}
          CHANGED: ${{ steps.changed.outputs.changes }}
        run: |
          set -euo pipefail

          if [ "$REF_TYPE" = "tag" ]; then
            services="$(jq -cn --arg s "${REF_NAME%@*}" '[$s]')"
            mode="production"
          elif [ "$EVENT" = "workflow_dispatch" ]; then
            services="$(jq -cn --arg s "$INPUT_SERVICE" '[$s]')"
            mode="nonproduction"
          else
            services="${CHANGED:-[]}"
            mode="nonproduction"
          fi

          matrix='[]'
          for svc in $(jq -r '.[]' <<<"$services"); do
            # Validate: expected shape, existing directory, Dockerfile present
            if ! [[ "$svc" =~ ^services/[a-z0-9_-]+/[a-z0-9_-]+$ ]]; then
              echo "::error::Unexpected service path: $svc"; exit 1
            fi
            if [ ! -f "$svc/Dockerfile" ]; then
              echo "::error::No Dockerfile in $svc"; exit 1
            fi
            version="$(scripts/image-version.sh "$svc" "$mode")"
            matrix="$(jq -c --arg s "$svc" --arg n "$(basename "$svc")" --arg v "$version" \
              '. + [{service:$s, name:$n, version:$v}]' <<<"$matrix")"
          done

          echo "matrix=$matrix" >> "$GITHUB_OUTPUT"
          echo "environment=$mode" >> "$GITHUB_OUTPUT"
          echo "Plan: $matrix ($mode)" >> "$GITHUB_STEP_SUMMARY"

  build:
    needs: plan
    if: ${{ needs.plan.outputs.matrix != '[]' }}
    runs-on: ubuntu-24.04
    timeout-minutes: 45
    permissions:
      contents: read
      packages: write
    strategy:
      fail-fast: false
      matrix:
        include: ${{ fromJSON(needs.plan.outputs.matrix) }}
    steps:
      - uses: actions/checkout@v5
        with:
          persist-credentials: false

      - uses: docker/setup-qemu-action@v3
      - uses: docker/setup-buildx-action@v3

      - uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Compute image name
        id: image
        env:
          NAME: ${{ matrix.name }}
          VERSION: ${{ matrix.version }}
        run: |
          repo="${GITHUB_REPOSITORY,,}"
          echo "tag=ghcr.io/${repo}/${NAME}:${VERSION}" >> "$GITHUB_OUTPUT"

      - uses: docker/build-push-action@v6
        with:
          context: ${{ matrix.service }}
          platforms: linux/amd64,linux/arm64
          push: ${{ github.event_name != 'workflow_dispatch' || inputs.push }}
          tags: ${{ steps.image.outputs.tag }}
          cache-from: type=gha,scope=${{ matrix.name }}
          cache-to: type=gha,mode=max,scope=${{ matrix.name }}

      - name: Summary
        env:
          TAG: ${{ steps.image.outputs.tag }}
        run: echo "Built \`$TAG\`" >> "$GITHUB_STEP_SUMMARY"
```

`scripts/image-version.sh` is the script from Lesson 13.1 (commit it with the executable bit: `git update-index --chmod=+x scripts/image-version.sh`).

Local rehearsal without Docker: run `scripts/image-version.sh services/go/api-golang nonproduction` in a clone, and test the planning logic by copying the `Plan builds` script into a file and exporting `EVENT`, `REF_TYPE`, `REF_NAME`, `INPUT_SERVICE` and `CHANGED` by hand.

## 6. Execution walkthrough

**A. A merge to `main` that changes only the Go service.**

1. The `push` event matches `branches: [main]` and `paths: services/**`.
2. `plan` checks out full history, `paths-filter` returns `["services/go/api-golang"]`, and the script validates the path, finds its `Dockerfile`, runs `image-version.sh … nonproduction`, and emits `[{"service":"services/go/api-golang","name":"api-golang","version":"1.0.9-0047-452cc9a"}]`.
3. `build` has one leg: QEMU and Buildx are set up, GHCR login succeeds with the temporary token (`packages: write`), the image tag becomes `ghcr.io/<owner>/<repo>/api-golang:1.0.9-0047-452cc9a`, and `build-push-action` builds for two architectures using the GHA cache and pushes.

**B. You push the tag `services/go/api-golang@1.0.10`.**

1. The tag pattern matches (`**` crosses the slashes). `paths-filter` is skipped (`ref_type == 'tag'`).
2. `plan` derives `services/go/api-golang` from `REF_NAME%@*`, runs the script in `production` mode: the tag exactly matches `HEAD`, so version is `1.0.10`. The image is `…/api-golang:1.0.10`.

**C. Manual run, `push` unchecked.** The build runs and loads nothing into the registry (`push: false`); this is the safe rehearsal.

## 7. Common mistakes

| Mistake | Effect | Fix |
|---|---|---|
| Two `push:` keys | One discarded or invalid | One key with `branches`, `paths`, `tags` |
| `tags: ['*@…']` | Won't match tags containing `/` | `**@…` |
| `git describe` without `--match` | Wrong "last tag" in a multi-service repo | `--match "<service>@*"` |
| Shallow checkout | `git describe` fails | `fetch-depth: 0` |
| `echo "x=${{ github.ref_name }}"` in a script | Injection through tag/branch names | Pass via `env:` |
| Undefined helper tasks (`generate-version-tag-extended`) | Step fails | Keep logic in one tested script |
| Missing `packages: write` | `denied: permission_denied` on push | Add to the build job |
| Uppercase in image name | Registry rejects the name | Lowercase it |
| Building matrix from unvalidated strings | Wrong directory, path traversal | Validate shape and existence |
| Relying on matrix legs for outputs | Last leg wins | Compute in `plan` |
| Pushing from untrusted triggers | Poisoned images | Push only from `main`, tags, or maintainers' manual runs |

## 8. Security considerations

- **Injection.** Tag names, branch names and `inputs.*` are attacker-influenced in some setups. The workflow passes them only through `env:`, validates the path with a regex and checks that a `Dockerfile` exists before using it.
- **Least privilege.** Only the `build` job gets `packages: write`. The `plan` job is read-only.
- **No long-lived registry secret** when using GHCR with `GITHUB_TOKEN`. If you use Docker Hub, use a narrowly scoped access token held as a secret, preferably in an environment.
- **Provenance and supply chain.** Pin third-party actions to SHAs in a real project (Chapter 7). Consider image signing and SBOM/provenance attestations; they are out of scope here.
- **Package visibility.** New GHCR packages from a workflow are private by default; check **Packages** settings before making an image public.
- **Cache poisoning.** GHA build caches are shared within the repository's scope rules; do not build or push from untrusted pull requests (this workflow has no `pull_request` trigger on purpose).

## 9. Hands-on exercise

**Goal:** build the pipeline in your practice repository and watch the three scenarios.

1. Add the Dockerfiles, `build-filters.yml`, `scripts/image-version.sh` (executable), and `build-push.yml`. Run `actionlint` and `shellcheck scripts/image-version.sh` if installed.
2. Tag a first release locally and verify the script:
   ```bash
   git tag services/go/api-golang@1.0.0
   scripts/image-version.sh services/go/api-golang production
   scripts/image-version.sh services/go/api-golang nonproduction
   ```
3. Merge the files to `main`. Then run the workflow manually for `services/node/api-node` with **push unchecked**. Read the plan summary.
4. Run again with **push checked**. Find the package under the repository's **Packages** and its tag.
5. Change a file in `services/go/api-golang`, merge to `main`, and confirm only that service builds with a `1.0.0-000N-<sha>` version.
6. Push the tag: `git push origin services/go/api-golang@1.0.1`. Confirm the image `1.0.1` appears.
7. Try to break it on a throwaway branch: push a tag such as `services/go/does-not-exist@1.0.0` and read the error message from the validation step.
8. Delete the practice packages and tags when finished.

**Success looks like:** a development image with `-NNNN-sha` on `main`, a clean `x.y.z` image for tags, and a clear failure for a bad tag.

**If it fails:** `denied: permission_denied` → job lacks `packages: write`, or organization policy blocks package creation; "no tag exactly matches" → the production-mode run is not on a tagged commit; plan produces `[]` on a push → the commit didn't touch a path in `build-filters.yml`.

## 10. Verification checklist

- [ ] One `push:` key; tag pattern uses `**`.
- [ ] `plan` validates path and `Dockerfile` before use.
- [ ] Release images use plain `x.y.z`; development images use `x.y.z-NNNN-sha`.
- [ ] Only `build` has `packages: write`.
- [ ] No `${{ }}` of user-controlled text inside `run:` scripts.
- [ ] Manual runs default to not pushing.
- [ ] You can state the image tag produced for each of the three scenarios.

## 11. Knowledge check

1. Why zero-pad the commit count?
2. Why does `git describe` need `--match` in a monorepo with per-service tags?
3. Why compute versions in a `plan` job instead of inside each matrix leg?
4. Why `**@…` instead of `*@…` for the tag pattern?
5. What do `cache-from`/`cache-to` with `mode=max` do?
6. What permission does the build job need to push to GHCR?
7. Why is `${{ github.ref_name }}` passed through `env:`?

<details>
<summary>Answers</summary>

1. So alphabetical sorting matches numerical order.
2. Otherwise it returns the nearest tag of any service.
3. Matrix legs can't expose separate outputs; a plan job makes one authoritative list that later jobs reuse.
4. `*` does not match `/`; service paths contain slashes.
5. They restore and save build layers, including intermediate stages, in the GHA cache.
6. `packages: write`.
7. Tag and branch names are user-chosen text; inline interpolation could execute them.
</details>

## 12. Key takeaways

- Separate release versions (`x.y.z`) from development versions (`x.y.z-NNNN-sha`) and derive both from Git.
- Per-service tags in a monorepo need `git describe --match`.
- Plan once, build many: emit a matrix of objects.
- One `push:` key; `**` for tags with slashes.
- GHCR plus `GITHUB_TOKEN` avoids a stored registry secret.
- Validate and pass untrusted text through `env:`.

## Sources and further reading

- Original notes: `git show master:13_build_push_images.md`. Corrections: duplicate `push:` keys; tag glob that would not match slash-containing tags; undefined `generate-version-tag-extended` task; `git describe` without `--match`; unsafe interpolation of `ref_name`/inputs into scripts; missing `permissions`; nonexistent bare-number action tags; unavailable `battila7/setup-binary`; the Docker Hub-only design (GHCR used here). The `ACT` environment checks were replaced by the manual `push` input.
- Official docs: [Workflow syntax](https://docs.github.com/en/actions/reference/workflows-and-actions/workflow-syntax) (filter patterns, `push`) · [Dependency caching](https://docs.github.com/en/actions/reference/workflows-and-actions/dependency-caching) · [Authenticate with GITHUB_TOKEN](https://docs.github.com/en/actions/tutorials/authenticate-with-github_token)
- Tools: [docker/build-push-action](https://github.com/docker/build-push-action) · [docker/setup-buildx-action](https://github.com/docker/setup-buildx-action) · [docker/login-action](https://github.com/docker/login-action) · [dorny/paths-filter](https://github.com/dorny/paths-filter)
- Verification note: `scripts/image-version.sh` was run in a scratch Git repository (results in Lesson 13.1). The workflow was not run on GitHub and Docker builds were not executed in preparing this guide.
