# Chapter 12: Capstone, Part 1: A Monorepo Test Workflow

> **Level:** advanced · **Time:** 4–5 hours · **Prerequisites:** [Chapters 5–11](05_core_features.md)
> **Next:** [Chapter 13: Capstone, Part 2: Build and Push Images](13_build_push_images.md)

The capstone builds one realistic CI/CD system over Chapters 12–14. This chapter builds the first and most important piece: a workflow that tests only the services that changed.

## Learning objectives

After this chapter you can:

- Create a small monorepo scaffold with a **standard Task interface** per service.
- Detect changed services with `dorny/paths-filter` and turn the result into a dynamic matrix.
- Install the right toolchain per service from the version files the service itself owns.
- Guard against an empty matrix, and add a single gate job for branch protection.
- Explain which claims about the original design needed correcting (output names versus paths, empty matrix, token permissions).

## 1. Problem statement

A repository with several services (Node, Go, Python) must not run every service's tests on every commit, and it must not skip a service's tests when that service did change. The workflow must:

1. figure out **which services changed**,
2. run each one's tests **in parallel** with the correct toolchain,
3. produce **one** status that branch protection can require, even when no service changed.

## 2. Core explanation

### The capstone repository (scaffold)

The course's original application is not in this repository, so you will create a tiny stand-in in your **own practice repository**. The point is the CI pattern, not the application code.

```
services/
├── node/api-node/          package.json, package-lock.json, Taskfile.yml, index.test.js
├── go/api-golang/          go.mod, main.go, main_test.go, Taskfile.yml
└── python/load-generator/  .python-version, test_smoke.py, Taskfile.yml
.github/
├── utils/file-filters.yml
└── workflows/test.yml
```

The deliberate layout (`services/<language>/<service>`) lets the workflow infer the toolchain from the path. This is a **convention you control**, not a GitHub feature.

Every service exposes the same task names (Chapter 11): `installci` and `test`.

`services/node/api-node/package.json`:

```json
{
  "name": "api-node",
  "version": "1.0.0",
  "private": true,
  "engines": { "node": ">=20" },
  "scripts": { "test": "node --test" }
}
```

(Run `npm install --package-lock-only` inside the folder once to create `package-lock.json`, which `npm ci` requires.)

`services/node/api-node/index.test.js`:

```javascript
const test = require('node:test');
const assert = require('node:assert');
test('adds', () => assert.strictEqual(1 + 1, 2));
```

`services/node/api-node/Taskfile.yml`:

```yaml
version: '3'
tasks:
  installci:
    cmds: [npm ci]
  test:
    cmds: [npm test]
```

`services/go/api-golang/go.mod`:

```
module example.com/api-golang

go 1.23
```

`services/go/api-golang/main_test.go`:

```go
package main

import "testing"

func TestAdd(t *testing.T) {
	if 1+1 != 2 {
		t.Fatal("math is broken")
	}
}
```

`services/go/api-golang/main.go`:

```go
package main

func main() {}
```

`services/go/api-golang/Taskfile.yml`:

```yaml
version: '3'
tasks:
  installci:
    cmds: [go mod download]
  test:
    cmds: [go test ./...]
```

`services/python/load-generator/.python-version`: `3.12`

`services/python/load-generator/test_smoke.py`:

```python
import unittest

class Smoke(unittest.TestCase):
    def test_add(self):
        self.assertEqual(1 + 1, 2)

if __name__ == "__main__":
    unittest.main()
```

`services/python/load-generator/Taskfile.yml`:

```yaml
version: '3'
tasks:
  installci:
    cmds: ['python -m pip --version']      # no dependencies in this scaffold
  test:
    cmds: ['python -m unittest -v']
```

### Step 1: Change detection

`dorny/paths-filter` compares the files changed in the event against named glob groups and reports which groups matched. Its output `changes` is a **JSON array of the filter names that matched**.

This is the subtle point that the original notes got wrong. They named filters `api-node`, `api-golang`, and so on, and then used those names as `working-directory` values in the matrix, where a path like `services/node/api-node` is required. The two never matched. The fix is to **make each filter name the service's path**, so the output *is* the list of directories:

`.github/utils/file-filters.yml`:

```yaml
# Keys are service directories. Values are the globs that should trigger tests for that service.
services/node/api-node:
  - services/node/api-node/**
  - .github/workflows/test.yml
  - .github/utils/file-filters.yml
services/go/api-golang:
  - services/go/api-golang/**
  - .github/workflows/test.yml
  - .github/utils/file-filters.yml
services/python/load-generator:
  - services/python/load-generator/**
  - .github/workflows/test.yml
  - .github/utils/file-filters.yml
```

Including the workflow and filter files means that editing the pipeline retests everything, which is usually what you want. A service that depends on another (for example a Go API that depends on a shared migrations directory) simply lists that directory in its globs.

How the comparison base is chosen (see the action's README for details): on `pull_request` it compares to the PR's base; on `push` it compares to the previous commit (or `base` input if set); for `workflow_dispatch` there is no diff, so you must decide what to do (see below). The action needs `pull-requests: read` on pull requests.

### Step 2: The dynamic matrix, with a guard

```yaml
test:
  needs: filter
  if: ${{ needs.filter.outputs.services != '[]' }}
  strategy:
    fail-fast: false
    matrix:
      service: ${{ fromJSON(needs.filter.outputs.services) }}
```

If nothing changed, `changes` is `[]` and a matrix with no values would error. The `if:` skips the job instead.

### Step 3: Toolchains from the service's own version files

| Language | Step | Version source |
|---|---|---|
| Node | `actions/setup-node@v4` with `node-version-file: <service>/package.json` | `engines.node` |
| Go | `actions/setup-go@v5` with `go-version-file: <service>/go.mod` | `go` directive |
| Python | `actions/setup-python@v5` with `python-version-file: <service>/.python-version` | the file |

Keeping the authoritative version in the service means developers, local builds and CI agree. Each setup step gets an `if:` based on the path prefix, again using the layout convention.

For Node, `cache: npm` plus `cache-dependency-path` caches dependencies. The Go and Python steps here omit caching because the scaffold has no dependencies (and therefore no `go.sum` or requirements file to key on). In a real service, add `cache-dependency-path: <service>/go.sum` for Go and `cache: pip` with a requirements file for Python.

### Step 4: Install Task and call the standard interface

`go-task/setup-task@v1` installs Task. (An earlier version of these notes used `battila7/setup-binary`, which could not be found publicly, so this guide uses the maintained official setup action.)

### Step 5: A single gate job

If no service changed, the `test` job is skipped; if a service failed, it fails. Branch protection needs one stable check name. Add a final job that aggregates:

```yaml
gate:
  if: always()
  needs: [filter, test]
  runs-on: ubuntu-24.04
  timeout-minutes: 5
  steps:
    - env:
        RESULTS: ${{ toJSON(needs.*.result) }}
      run: |
        echo "results: $RESULTS"
        if echo "$RESULTS" | grep -Eq '"(failure|cancelled)"'; then exit 1; fi
```

A skipped `test` job (nothing changed) counts as acceptable; a failed `filter` or `test` fails the gate.

## 3. Mental model

The workflow is a **dispatcher and workers**. `filter` is the dispatcher: it reads the diff and writes a work list (`["services/go/api-golang"]`). The matrix is a worker pool that takes the list and runs one worker per item. The gate is the supervisor who reports "all work finished OK" (or "something failed") with one signal.

## 4. Syntax breakdown

| Syntax | Meaning |
|---|---|
| `needs.filter.outputs.services` | Job output string containing a JSON array |
| `fromJSON(...)` | Parses that string into an array for the matrix |
| `matrix.service` | The current service directory (also `working-directory`) |
| `if: startsWith(matrix.service, 'services/node')` | Convention-based toolchain selection |
| `fail-fast: false` | Let all legs finish so you see every failure |
| `permissions: pull-requests: read` | Needed by paths-filter on PRs |
| `if: always()` on `gate` | Gate runs even if upstream jobs fail |

## 5. Practical example: the complete workflow

`.github/workflows/test.yml`:

```yaml
name: Test

on:
  pull_request:
  push:
    branches: [main]
  workflow_dispatch:

permissions:
  contents: read

concurrency:
  group: test-${{ github.ref }}
  cancel-in-progress: true

jobs:
  filter:
    runs-on: ubuntu-24.04
    timeout-minutes: 5
    permissions:
      contents: read
      pull-requests: read       # paths-filter lists PR files through the API
    outputs:
      services: ${{ steps.pick.outputs.services }}
    steps:
      - uses: actions/checkout@v5
        with:
          persist-credentials: false
      - uses: dorny/paths-filter@de90cc6fb38fc0963ad72b210f1f284cd68cea36  # v3.0.2
        id: changed
        if: github.event_name != 'workflow_dispatch'
        with:
          filters: .github/utils/file-filters.yml
      - name: Decide which services to test
        id: pick
        env:
          EVENT: ${{ github.event_name }}
          CHANGED: ${{ steps.changed.outputs.changes }}
        run: |
          if [ "$EVENT" = "workflow_dispatch" ]; then
            # Manual runs test everything: there is no diff to compare
            echo 'services=["services/node/api-node","services/go/api-golang","services/python/load-generator"]' >> "$GITHUB_OUTPUT"
          else
            echo "services=${CHANGED:-[]}" >> "$GITHUB_OUTPUT"
          fi

  test:
    needs: filter
    if: ${{ needs.filter.outputs.services != '[]' }}
    runs-on: ubuntu-24.04
    timeout-minutes: 20
    strategy:
      fail-fast: false
      matrix:
        service: ${{ fromJSON(needs.filter.outputs.services) }}
    steps:
      - uses: actions/checkout@v5
        with:
          persist-credentials: false

      - name: Set up Node.js
        if: startsWith(matrix.service, 'services/node/')
        uses: actions/setup-node@v4
        with:
          node-version-file: ${{ matrix.service }}/package.json
          cache: npm
          cache-dependency-path: ${{ matrix.service }}/package-lock.json

      - name: Set up Go
        if: startsWith(matrix.service, 'services/go/')
        uses: actions/setup-go@v5
        with:
          go-version-file: ${{ matrix.service }}/go.mod
          cache: false              # scaffold has no go.sum; enable caching in a real service

      - name: Set up Python
        if: startsWith(matrix.service, 'services/python/')
        uses: actions/setup-python@v5
        with:
          python-version-file: ${{ matrix.service }}/.python-version

      - uses: go-task/setup-task@v1

      - name: Install dependencies
        working-directory: ${{ matrix.service }}
        run: task installci

      - name: Run tests
        working-directory: ${{ matrix.service }}
        run: task test

  gate:
    if: always()
    needs: [filter, test]
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

Why the `pick` step exists: `workflow_dispatch` has no diff, and `paths-filter` isn't useful there, so manual runs test everything. Passing `CHANGED` through `env:` also keeps the data separate from the script text. If `changes` is empty the fallback is `[]`.

Local rehearsal (optional): `actionlint`, then `task` commands directly in each service folder (`cd services/go/api-golang && task installci && task test`). `act` can run the workflow, but `dorny/paths-filter` depends on git history and GitHub context, so results under `act` are only indicative (Chapter 4). If you use `act`, pass a fine-grained token only for the run, and keep any `event.json` and secret files out of Git.

## 6. Execution walkthrough

Scenario: a pull request changes only `services/go/api-golang/main.go`.

1. The `pull_request` event starts the workflow. `filter` checks out the repository and `paths-filter` compares the PR's changed files against `file-filters.yml`.
2. Only the filter named `services/go/api-golang` matches, so `changes` is `["services/go/api-golang"]`. The `pick` step writes that to the job output `services`.
3. `test` evaluates `if: services != '[]'` (true) and builds a matrix of **one** leg.
4. In that leg only the Go setup step runs (the Node and Python steps are skipped by their `if:`); Task is installed; `task installci` and `task test` run in `services/go/api-golang`.
5. `gate` runs last (`always()`), sees `success` for both jobs and passes.

Scenario: a PR changes only `README.md`: `changes` is `[]`, `test` is **skipped**, `gate` still passes. A PR that breaks a Node test: that matrix leg fails, other legs still finish (`fail-fast: false`), `gate` fails.

## 7. Common mistakes

| Mistake | Effect | Fix |
|---|---|---|
| Filter names like `api-node` used as `working-directory` | Wrong paths; job fails | Name filters after the directory paths (this chapter) |
| No guard when `changes` is `[]` | "Matrix vector does not contain any values" | `if: needs.filter.outputs.services != '[]'` |
| Requiring the `test` job in branch protection | PRs with no relevant changes wait forever | Require only `gate` |
| `paths:` at workflow level plus required checks | Same hang | Filter inside the workflow instead |
| Version tags without `v` (`@5.0.0`) | Action not found | `@v5` etc.; tags carry a `v` |
| Missing `pull-requests: read` | 403 listing PR files | Add it to the `filter` job |
| Running tests with `workflow_dispatch` through the filter | No diff, so nothing runs | Explicit "test all" branch |
| Forgetting the workflow files in the globs | Pipeline changes aren't tested | Add them to each group |
| Interpolating `CHANGED` into the script | Risky if the content were attacker-influenced | Use `env:` (as shown) |

## 8. Security considerations

- Pull requests from forks run with a read-only token and no secrets, which is fine for tests; keep it that way (Chapter 11).
- The matrix values come from your own filter keys, not from user text, so using them in `working-directory` is safe. Do not build matrices from PR titles, branch names or other free text.
- `persist-credentials: false` keeps the checkout token out of the repository config in jobs that don't push.
- `dorny/paths-filter` is pinned to a verified SHA (the tag `v3.0.2` resolves to `de90cc6f…`). Pin and review the other third-party actions in a real project (Chapter 7).

## 9. Hands-on exercise

**Goal:** build the scaffold and watch the matrix change with the diff.

1. In your practice repository create branch `capstone-tests` and add the scaffold files, `file-filters.yml`, and `test.yml`.
2. Run `actionlint`. Run each service's tasks locally to prove they pass: `cd services/node/api-node && npm install --package-lock-only && task installci && task test` (and similarly for Go and Python).
3. Push, open a PR into `main`. Confirm all three services ran (everything is new).
4. Merge. Create branch `touch-go`, change only `services/go/api-golang/main.go` (add a comment), open a PR and confirm only one matrix leg runs.
5. Create branch `touch-docs`, change only `README.md`, open a PR and confirm `test` is skipped and `gate` is green.
6. Create branch `break-node`, make the Node test fail (`1 + 1` equals `3`), open a PR and confirm that leg fails and `gate` is red.
7. In **Settings → Branches/Rulesets**, require only the `gate` check on `main`.

**Success looks like:** matrix legs that match the changed folders each time, a skipped `test` job for documentation-only changes, and a red `gate` when a leg fails.

**If it fails:** `npm ci` complains about a missing lock file → generate it; paths-filter returns everything as changed on the first PR (expected, since all files are new); required check shows "Expected": the check name must be `gate` (the job's `name` or ID).

## 10. Verification checklist

- [ ] `actionlint` is clean on `test.yml`.
- [ ] Each service passes `task installci && task test` locally.
- [ ] Changing one service runs one leg; changing only docs runs none.
- [ ] A failing leg turns `gate` red; skipped `test` keeps it green.
- [ ] Only `gate` is a required check.
- [ ] No action is referenced by a nonexistent tag.

## 11. Knowledge check

1. Why must the filter names equal the service paths in this design?
2. What does `changes` contain, and what type is it when it reaches `fromJSON`?
3. Why does the matrix need an `if:` guard?
4. Why require `gate` instead of `test`?
5. What happens for `workflow_dispatch`, and why?
6. Why include the workflow file in each filter group?

<details>
<summary>Answers</summary>

1. The output is the list of matching names and the matrix uses them as directories.
2. A JSON array of matching filter names, passed as a string; `fromJSON` parses it.
3. An empty array is an error for a matrix.
4. `test` may be skipped (no relevant changes), which would leave a required check pending; `gate` always reports.
5. It runs all services because there is no diff to compare.
6. So that editing the pipeline triggers tests that prove the change works.
</details>

## 12. Key takeaways

- Detect, then dispatch: a small filter job produces a work list; a matrix does the work.
- Make machine-readable names equal the values you need downstream.
- Guard empty matrices; aggregate with a gate job.
- A standard task interface and per-service version files keep the workflow short.
- Use `env:` for data flowing into scripts.

## Sources and further reading

- Original notes: `git show master:12_test_workflow.md`. Corrections: filter output names used as paths (the original workflow could not work as written), missing empty-matrix guard, nonexistent tags without the `v` prefix, unavailable `battila7/setup-binary`, missing `pull-requests: read`, and the unsupported performance numbers ("70% reduction", "~10 minutes") which were removed. The course's real application is not part of this repository; this chapter substitutes a small scaffold.
- Tools: [dorny/paths-filter](https://github.com/dorny/paths-filter) · [go-task/setup-task](https://github.com/go-task/setup-task) · [Task](https://taskfile.dev/)
- Official docs: [Workflow syntax](https://docs.github.com/en/actions/reference/workflows-and-actions/workflow-syntax) · [Contexts](https://docs.github.com/en/actions/reference/workflows-and-actions/contexts)
- Verification note: the paths-filter SHA was checked with `git ls-remote`. The complete workflow and scaffold were written for this guide; see the final report for what was and was not executed.
