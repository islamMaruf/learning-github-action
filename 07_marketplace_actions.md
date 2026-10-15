# Chapter 7: Using Marketplace Actions Safely

> **Level:** intermediate · **Time:** 2 hours · **Prerequisites:** [Chapters 5–6](05_core_features.md)
> **Next:** [Chapter 8: Authoring Actions](08_authoring_actions.md)

## Learning objectives

After this chapter you can:

- Find actions and read their source, `action.yml`, inputs and outputs.
- Evaluate a third-party action with concrete trust signals, and know what those signals do *not* prove.
- Explain the four ways to reference an action version and why only a full commit SHA is immutable.
- Look up the commit SHA for a release tag, and keep pinned SHAs up to date automatically.
- Recognise the main categories of common actions.

## 1. Problem statement

Every `uses:` line runs someone else's code inside your job, with access to the workspace, the `GITHUB_TOKEN`, and any secrets you pass to it. Reusing actions saves hours; blindly reusing them is a supply-chain risk. This chapter gives you a routine for deciding *which* actions to use and *how* to reference them.

## 2. Core explanation

### Where actions come from

- **The Marketplace** (github.com/marketplace, filter by *Actions*) lists published actions. Publishing there is optional: **any public repository with an `action.yml` can be used** with `uses: owner/repo@ref`, whether or not it is listed.
- **Your own repository**: `uses: ./.github/actions/my-action` (Chapter 8).
- **Docker images**: `uses: docker://alpine:3.20` (rarely needed).

Counts of available actions change constantly, so treat any number in a blog post as stale.

### Anatomy of a `uses:` reference

```yaml
- uses: actions/checkout@v5     # owner/repo@ref
  with:                         # inputs documented in the action's README / action.yml
    fetch-depth: 0
```

To learn what an action accepts, open its repository and read:

| File / page | Tells you |
|---|---|
| `README.md` | Usage examples, caveats |
| `action.yml` | The exact `inputs`, `outputs` and how it runs (`runs.using`: `node*`, `docker`, or `composite`) |
| Releases / changelog | Breaking changes between versions |
| Issues / pull requests | Known bugs, maintainer responsiveness |
| `SECURITY.md` / advisories | How vulnerabilities are handled |

### Evaluating an action: signals and limits

| Signal | Why it helps | What it does **not** prove |
|---|---|---|
| **Publisher** is GitHub (`actions/*`), a cloud vendor (`aws-actions/*`, `azure/*`, `google-github-actions/*`) or the tool's own vendor (`docker/*`, `cypress-io/*`) | Maintained by the party with the most at stake | That the code is free of bugs |
| **Marketplace "verified creator" badge** | GitHub has verified the publisher's organization identity (see GitHub's docs for exact criteria) | That the action is safe or well made |
| **Recent releases and commits** | Likely to be maintained and compatible with current runners (for example current Node versions) | That new commits are benign |
| **Active, answered issues** | Real users, responsive maintainers | |
| **Stars and usage** | Popularity means more eyes | Quality or security; popular actions are the most attractive targets |
| **Small, readable source** | You can audit it in minutes | That its dependencies are clean |
| **Clear docs and versioned releases** | Easier to upgrade safely | |

There is no star-count threshold that makes an action safe. Combine signals, read the code of anything that receives secrets or write permissions, and prefer an official action over a community one when both exist. If the action is a tiny wrapper around a command-line tool, consider just calling the tool in a `run:` step.

### Versions: four ways to reference an action

| Reference | Example | Mutable? | Trade-off |
|---|---|---|---|
| Branch | `@main` | Yes, changes with every commit | Never use for anything you care about |
| Major tag | `@v5` | Yes: the maintainer moves it with each `v5.x.y` release | Convenient; you get fixes automatically (and any compromise) |
| Exact tag | `@v5.0.0` | Yes: tags can be deleted and re-created | Reproducible only if the maintainer behaves |
| **Full commit SHA** | `@11bd71901bbe5b1630ceea73d27597364c9af683` | **No** | Verbose; needs tooling to update |

Note that tags are published with a leading `v` (`v4.2.2`, not `4.2.2`). An example that uses a tag that does not exist fails with "unable to resolve action".

**Why SHA pinning matters.** A tag is only a movable pointer. If a maintainer account or token is compromised, an attacker can re-point existing tags to malicious code, and every workflow that used the tag runs it on its next execution. This is not hypothetical: widely used actions have been compromised this way, with the attack stealing secrets from CI logs and environments. A full 40-character commit SHA identifies exact content and cannot be re-pointed.

Caveats: SHA pinning protects the action's own code, **not necessarily what that action downloads at run time** (for example, an unpinned dependency or a "latest" binary). It also means you must update deliberately. GitHub also supports organization and repository policies that can *require* actions to be pinned to a full SHA or restrict the allowed actions; see the repository's **Settings → Actions**.

#### Finding the SHA for a tag

```bash
# Annotated tags need ^{} to dereference to the commit:
git ls-remote https://github.com/actions/checkout 'refs/tags/v4.2.2*'
# or inspect the release page on GitHub
```

For `actions/checkout`, the tag `v4.2.2` resolves to commit `11bd71901bbe5b1630ceea73d27597364c9af683`. (An earlier version of these notes listed a different SHA, `b4ffde65…`, which is `v4.1.1`; always verify a SHA against the tag yourself and never copy one from a document you have not checked.)

```yaml
- uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683  # v4.2.2
```

The trailing comment records which release the SHA corresponds to, so humans and update tools can read it.

#### Keeping pinned actions current

Pinning without updating leaves you running old, possibly vulnerable code. Let a bot propose updates as pull requests: **Dependabot** supports the `github-actions` ecosystem and understands the `# vX.Y.Z` comment convention; Renovate offers similar behavior.

`.github/dependabot.yml`:

```yaml
version: 2
updates:
  - package-ecosystem: github-actions
    directory: /
    schedule:
      interval: weekly
```

Review each update PR: read the release notes, look at the diff for unexpected changes, and let your CI run on the PR (Dependabot PRs get a read-only token and no repository secrets by default).

### Common action categories

| Category | Examples | Notes |
|---|---|---|
| Repository & artifacts | `actions/checkout`, `actions/upload-artifact`, `actions/download-artifact`, `actions/cache` | Use current major versions (Chapter 6) |
| Language setup | `actions/setup-node`, `actions/setup-python`, `actions/setup-go`, `actions/setup-java` | Most have a `cache:` input |
| GitHub API scripting | `actions/github-script` | Inline JavaScript with an authenticated client |
| Containers | `docker/setup-buildx-action`, `docker/setup-qemu-action`, `docker/login-action`, `docker/metadata-action`, `docker/build-push-action` | Chapter 13 |
| Cloud auth | `aws-actions/configure-aws-credentials`, `google-github-actions/auth`, `azure/login` | Prefer OIDC (Chapter 6) |
| Quality & security | `super-linter/super-linter`, CodeQL actions, vendor scanners | Run on pull requests, read-only |
| Change detection | `dorny/paths-filter` | Chapter 12 |
| Release & chores | `actions/stale`, release tooling | Chapter 9 |

Always read the current README for the major version you pick: input names and defaults change between majors.

## 3. Mental model

An action is a **dependency like an npm package, except it runs with your credentials in your build pipeline**. You wouldn't `npm install` something from `main` of an unknown repo into production; the same discipline applies. The tag is a label on a box; the SHA is the box's fingerprint.

## 4. Syntax breakdown

| Syntax | Meaning |
|---|---|
| `uses: owner/repo@ref` | Action in a public (or accessible) repository |
| `uses: owner/repo/path@ref` | Action in a subdirectory of a repository |
| `uses: ./path` | Action in the same repository (requires a prior checkout) |
| `uses: docker://image:tag` | Run a Docker image directly |
| `with:` | Inputs (strings; booleans and numbers are converted to strings) |
| `# v4.2.2` trailing comment | Human- and Dependabot-readable record of the SHA's release |
| `id:` + `${{ steps.id.outputs.x }}` | Read an action's outputs |

## 5. Practical example

A CI workflow with pinned, minimal-permission steps. The checkout SHA is verified above; for the other actions, look up and verify each SHA yourself using the procedure in this chapter. The placeholders are marked so that the file cannot be mistaken for finished.

```yaml
name: CI (pinned)

on:
  pull_request:
  push:
    branches: [main]

permissions:
  contents: read

jobs:
  test:
    runs-on: ubuntu-24.04
    timeout-minutes: 15
    steps:
      - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683  # v4.2.2
        with:
          persist-credentials: false
      # Replace <SHA> with the commit for the release tag, then keep the version comment.
      - uses: actions/setup-node@v4        # TODO pin: actions/setup-node@<SHA>  # v4.x.y
        with:
          node-version: 22
          cache: npm
      - run: npm ci
      - run: npm test
```

`persist-credentials: false` stops checkout from leaving the token in the repository's git config where later steps (or a malicious dependency) could read it, a sensible default when a job doesn't need to push.

## 6. Execution walkthrough

When the runner prepares the job, before any step runs it resolves each `uses:` reference:

1. For `owner/repo@ref`, it fetches that exact ref (a SHA resolves to exactly one commit) and downloads the repository contents.
2. It reads `action.yml` to learn the runtime (Node, Docker, or composite). Docker actions are built or pulled; Node actions run with the Node runtime bundled with the runner.
3. When the step executes, the action receives inputs as `INPUT_*` environment variables, the job's environment, and the workspace.
4. Outputs are collected and made available as `steps.<id>.outputs.*`.

With a mutable tag, step 1 can resolve to different code than yesterday's run. With a SHA it cannot.

## 7. Common mistakes

| Mistake | Effect | Fix |
|---|---|---|
| `@main` / `@master` | Surprise breakage, instant exposure to a compromised branch | Pin to a tag at least, a SHA ideally |
| Copying a SHA from a blog or old notes | Might be a different release, or not exist in the repo | Verify with `git ls-remote` / the release page |
| Tags without `v` (`@4.2.2`) | "Unable to resolve action" | Use the exact tag name |
| Pinning and never updating | Old action versions with known flaws or deprecated runtimes | Dependabot / Renovate |
| Using abandoned actions (archived repos) | No fixes, deprecated Node runtime warnings | Switch to a maintained alternative or call the CLI directly |
| Judging safety by stars | Popular ≠ safe | Read the source of anything privileged |
| Passing all secrets to an action that needs one | Larger blast radius | Pass only what is needed, via `with:` or `env:` on that step |

## 8. Security considerations

- Pin to full SHAs for every action in workflows that touch secrets, deploy, or publish. Official `actions/*` pinned by major tag is a common accepted trade-off for low-risk workflows, but treat that as a conscious decision.
- Use `permissions:` (Chapter 6) so a compromised action only receives what the job needs.
- Don't give a third-party action your cloud credentials if an official one exists; use OIDC so there's nothing long-lived to steal.
- Restrict allowed actions at the organization or repository level where policy matters.
- After a publicly reported action compromise, search your repositories for the action, rotate any secrets that run exposed, and check the run logs.
- Watch for typosquatting: `actons/checkout` is not `actions/checkout`.

## 9. Hands-on exercise

**Goal:** audit and pin the actions in your practice repository.

1. List every `uses:` line in your workflows:
   ```bash
   grep -rhn "uses:" .github/workflows | sort | uniq -c
   ```
2. For each one, open its repository and answer: who owns it, when was the last release, does it have a maintained major version, and what does `action.yml` say it runs?
3. Pick `actions/checkout`. Find the SHA of `v4.2.2`:
   ```bash
   git ls-remote https://github.com/actions/checkout 'refs/tags/v4.2.2*'
   ```
   Compare it with the SHA in this chapter.
4. Replace the tag with the SHA and a `# v4.2.2` comment; run `actionlint`, push on a branch and confirm the run is green.
5. Add the `dependabot.yml` above on the branch. After merging, check **Insights → Dependency graph → Dependabot** for the update schedule.

**Success looks like:** every action in your workflow has an owner you can name, checkout is SHA-pinned, and Dependabot is configured.

**If it fails:** `unable to resolve action` → wrong tag or SHA (is it in the action's repository? did you drop the `v`?).

## 10. Verification checklist

- [ ] No `@main`/`@master` references remain.
- [ ] Each pinned SHA was looked up against the tag, not copied blindly.
- [ ] A `# vX.Y.Z` comment follows every SHA.
- [ ] `dependabot.yml` includes the `github-actions` ecosystem.
- [ ] You can explain what pinning does **not** protect against.

## 11. Knowledge check

1. Why can a tag change but a commit SHA not?
2. What does the "verified creator" badge tell you, and what doesn't it?
3. Where do you find an action's input names?
4. A workflow uses `actions/checkout@4.2.2` and fails to resolve. Why?
5. Name two ways to keep SHA-pinned actions up to date.
6. Why might `persist-credentials: false` be a good default?

<details>
<summary>Answers</summary>

1. A tag is a mutable pointer; a SHA is a hash of the commit's content.
2. It says GitHub verified the publisher's identity; it does not certify safety or quality.
3. The action's `action.yml` and README.
4. The tag is `v4.2.2`; the `v` is part of the name.
5. Dependabot and Renovate.
6. It keeps the token out of git config so later steps can't read it.
</details>

## 12. Key takeaways

- An action is third-party code with your credentials; vet it like a dependency.
- Signals help, but none proves safety; read the code of privileged actions.
- Only a full commit SHA is immutable; tags and branches can be moved.
- Pin, comment the version, and automate updates.
- Pair pinning with minimal `permissions` and OIDC.

## Sources and further reading

- Original notes: `git show master:07_marketplace_actions.md`. Corrections: the mismatched SHA (it belonged to v4.1.1, not v4.2.2), outdated `@v3` cache and artifact actions, nonexistent version pins, dubious "verification criteria" and star-count thresholds, and unverifiable marketplace counts.
- Official docs: [Workflow syntax: `jobs.<job_id>.steps[*].uses`](https://docs.github.com/en/actions/reference/workflows-and-actions/workflow-syntax) · [Dependency caching](https://docs.github.com/en/actions/reference/workflows-and-actions/dependency-caching) · [Marketplace](https://github.com/marketplace?type=actions)
- Verification note: SHA `11bd7190…` for `actions/checkout` v4.2.2 was checked against the repository's tags with `git ls-remote`. Other SHAs are intentionally not listed.
