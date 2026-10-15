# GitHub Actions: A Structured Learning Guide

A 14-chapter guide that takes you from "what is a workflow?" to a multi-service CI/CD pipeline (test → build → version → GitOps → release), with exercises, checks and security notes in every chapter.

## Attribution and scope

This guide is a rewrite of study notes taken while following the DevOps Directive course *GitHub Actions: Beginner to Pro* by Sid Palas ([course site](https://courses.devopsdirective.com/github-actions-beginner-to-pro)). The notes were restructured as learning material, corrected against GitHub's documentation and action repositories, and extended with exercises. The structure and ideas owe much to that course; any errors in this version are not the course author's. Where the original notes disagreed with GitHub's current behavior, the chapter's **Sources** section says what changed.

This repository contains **documentation only**: it has no workflow files of its own. Every workflow in the chapters is an example for you to create in your own practice repository.

## How the chapters are organized

Each chapter follows the same pattern: learning objectives → problem → core explanation → mental model → syntax → worked example → execution walkthrough → common mistakes → security → hands-on exercise → verification checklist → knowledge check (with answers) → key takeaways → sources.

| # | Chapter | You will be able to… | Level |
|---|---|---|---|
| 1 | [What GitHub Actions is and how a workflow runs](01_introduction.md) | Explain events, jobs, steps, runners and the orchestrator/runner split; run a first workflow | Beginner |
| 2 | [Why CI/CD exists](02_history_and_motivation.md) | Place CI tools in history; contrast push-based and GitOps delivery | Beginner |
| 3 | [Why (and when not) GitHub Actions](03_why_github_actions.md) | Compare tools honestly; reduce lock-in | Beginner |
| 4 | [Your development environment](04_development_environment.md) | Lint with `actionlint`, rehearse with `act`, use `gh` | Beginner |
| 5 | [Core features](05_core_features.md) | Triggers, filters, jobs and `needs`, env scopes, outputs, secrets, contexts, safe inputs | Beginner → Intermediate |
| 6 | [Advanced features](06_advanced_features.md) | Runners, artifacts, caching, permissions, OIDC, matrix, concurrency | Intermediate |
| 7 | [Using marketplace actions safely](07_marketplace_actions.md) | Evaluate actions, pin to SHAs, automate updates | Intermediate |
| 8 | [Authoring reusable logic](08_authoring_actions.md) | Composite actions, reusable workflows, JS and container actions | Intermediate → Advanced |
| 9 | [Common workflow patterns](09_common_workflows.md) | Lint/test/build/deploy/release/chores | Intermediate |
| 10 | [Developer experience](10_developer_experience.md) | Taskfile-driven pipelines, debug logging, measuring | Intermediate |
| 11 | [Best practices](11_best_practices.md) | Performance, maintainability, security baseline, audit checklist | Intermediate → Advanced |
| 12 | [Capstone 1: monorepo test workflow](12_test_workflow.md) | Change detection, dynamic matrix, gate job | Advanced |
| 13 | [Capstone 2: build, version, push images](13_build_push_images.md) | `git describe` versions, Buildx, GHCR | Advanced |
| 14 | [Capstone 3: GitOps, releases, chores, timing](14_remaining_workflows_conclusion.md) | Reusable GitOps update, retries, approvals, wrap-up | Advanced |

**Suggested reading order:** straight through, 1 → 14. If you already know the basics, read 5, 6, 11 first, then do the capstone (12 → 14). Do the exercises in a **practice repository you own**, never in a real project, and use branches and pull requests rather than pushing experiments to a production branch.

## Prerequisites

Git (clone, branch, commit, pull request), a terminal, a GitHub account, and a code editor. Docker is needed from Chapter 13. On Windows, use WSL 2.

## How to Read, Verify, and Revisit This Guide

**Reading.** Read a chapter, then do its exercise before moving on. The knowledge-check answers are collapsed under each chapter; try first.

**What was verified, and how.**

- Facts about GitHub behavior were checked against GitHub's documentation where the page could be retrieved, and action tags and commit SHAs were checked with `git ls-remote` against the action repositories. Chapters list items that were **not** verified under "Sources and further reading".
- Scripts were executed in scratch Git repositories: `scripts/image-version.sh` and the *Plan builds* step of the Chapter 13 workflow (push, tag, empty and invalid-tag cases), and `scripts/update-image-tag.sh` (Chapter 14). The three shell scripts also passed `bash -n`.
- All 74 YAML code blocks parse as YAML (PyYAML). The 23 complete workflows also passed a basic structural check (jobs have `runs-on`/`steps` or `uses`, `needs` refer to existing jobs, no inline untrusted values in `run:`).
- **`actionlint` and `shellcheck` could not be installed in the environment used for this rewrite, so neither was run.** Run `actionlint` yourself on every workflow you create.
- **No workflow in this guide was run on GitHub**, and no Docker image was built, during this rewrite. Treat the workflows as carefully reviewed examples: run `actionlint` on them and test them on a branch in a practice repository.

**Things that go stale.** Action major versions (the examples use versions that were current when the chapters were written; newer majors of several actions, such as the `actions/setup-*`, `actions/cache` and `docker/*` actions, already exist), runner image labels, pricing and quotas, Node runtimes, UI wording, and repository-setting names. When a chapter tells you to check the docs, do.

**Revisiting.** Re-read Chapters 7 and 11 when you add a new third-party action or change a deploy path. After a security advisory about an action, search your workflows for it. Update pinned SHAs through Dependabot, not by hand.

### Source reference table

The original files are the 14 chapter notes from the previous version of this repository (retrievable with `git show master:<file>`). This repository has no workflow files, so the "workflow file" column points to examples inside the guide.

| Learning topic | Guide section | Original source file | Relevant workflow file | Verification notes |
|---|---|---|---|---|
| Orchestrator vs runner; VM vs container | Ch. 1 | `01_introduction.md` | `hello.yml` example in Ch. 1 | Conceptual; no live run |
| CI/CD history; push vs GitOps | Ch. 2 | `02_history_and_motivation.md` | none | Dates approximate; unsourced statistics removed |
| Tool comparison | Ch. 3 | `03_why_github_actions.md` | thin-workflow example in Ch. 3 | Vendor claims removed; check current pricing yourself |
| Local tooling | Ch. 4 | `04_development_environment.md` | `check.yml` in Ch. 4 | Tool commands not executed here |
| Events, jobs, outputs, secrets, contexts | Ch. 5 | `05_core_features.md` | `core-lab.yml` in Ch. 5 | `node` is not a documented shell; `secrets` unavailable in `if` (docs) |
| Runners, artifacts, cache, permissions, OIDC, matrix, concurrency | Ch. 6 | `06_advanced_features.md` | `advanced-lab.yml` in Ch. 6 | Artifact v3 shutdown and valid runner labels checked; pricing/limits deferred to docs |
| Marketplace actions, pinning | Ch. 7 | `07_marketplace_actions.md` | `ci.yml` (pinned) in Ch. 7 | `actions/checkout` v4.2.2 SHA checked via `git ls-remote` |
| Composite actions, reusable workflows, JS/container actions | Ch. 8 | `08_authoring_actions.md` | examples in Ch. 8 | Environment-secret behavior and nesting limits flagged for verification |
| Lint/test/build/deploy/release/stale/dependency patterns | Ch. 9 | `09_common_workflows.md` | `ci.yml`, `deploy.yml`, `release.yml` in Ch. 9 | Archived status of `actions/create-release` from knowledge, not re-checked |
| Taskfile, `act`, debugging, measuring | Ch. 10 | `10_developer_experience.md` | `check`/diagnostic examples in Ch. 10 | `breakpoint-action` has `@v0`, no `@v1` (checked) |
| Performance, maintainability, security baseline | Ch. 11 | `11_best_practices.md` | gate-job example in Ch. 11 | Settings UI wording flagged for verification |
| Change detection, dynamic matrix, gate | Ch. 12 | `12_test_workflow.md` | `test.yml` in Ch. 12 | `dorny/paths-filter` v3.0.2 SHA checked; not run on GitHub |
| Versioning, Buildx, GHCR | Ch. 13 | `13_build_push_images.md` | `build-push.yml` in Ch. 13 | Version script run in scratch repo; Docker builds not run |
| GitOps update, retry, releases, timing export | Ch. 14 | `14_remaining_workflows_conclusion.md` | `gitops-update.yml`, `release.yml` in Ch. 14 | Update script run in scratch repo; Honeycomb action's README read, action not run |

## Safety rules used throughout

- Practice in your own repository; use branches and pull requests, or manual triggers, to test.
- Never put real credentials in a workflow or document. Use secrets, and prefer OIDC to stored cloud keys.
- Pass untrusted text (PR titles, branch names, inputs) through `env:`, never inline into `run:` scripts.
- Declare `permissions:` in every workflow; start from `contents: read`.
- Pin third-party actions to commit SHAs and update them with automation.

## Further reading

- [GitHub Actions documentation](https://docs.github.com/en/actions)
- [Workflow syntax reference](https://docs.github.com/en/actions/reference/workflows-and-actions/workflow-syntax)
- [Runner images](https://github.com/actions/runner-images)
- [actionlint](https://github.com/rhysd/actionlint) and [act](https://github.com/nektos/act)
- The original course: [DevOps Directive, GitHub Actions: Beginner to Pro](https://courses.devopsdirective.com/github-actions-beginner-to-pro)
