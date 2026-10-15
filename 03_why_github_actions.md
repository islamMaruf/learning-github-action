# Chapter 3: Why GitHub Actions? An Honest Comparison

> **Level:** beginner · **Time:** 30–40 minutes · **Prerequisites:** [Chapters 1–2](01_introduction.md)
> **Next:** [Chapter 4: Development Environment](04_development_environment.md)

## Learning objectives

After this chapter you can:

- State the real strengths of GitHub Actions and the trade-offs you accept.
- Compare it with GitLab CI/CD, CircleCI and Jenkins on criteria that matter.
- Pick a tool for a given situation and defend the choice.
- Name the three weakest spots (reuse, debugging, observability) and the standard mitigations.

## 1. Problem statement

You will rarely choose a CI tool in a vacuum: your code is already on GitHub, GitLab or Bitbucket, your team has habits, and some company policies are fixed. The useful question is not "which tool is best?" but "which tool fits *these* constraints, and what will it cost me?"

## 2. Core explanation

### What GitHub Actions is good at

1. **Almost no setup if your code is on GitHub.** Add a YAML file to `.github/workflows/` and it runs. No separate account, no webhook plumbing, no server.
2. **Deep integration.** Workflows can react to nearly any repository event (pushes, pull requests, issues, releases, comments, schedules) and use the same permissions and identity model as the rest of GitHub.
3. **A very large public marketplace.** Setting up a language runtime, logging in to a cloud provider or building a container is often a single `uses:` line.
4. **Community mass.** More questions are already answered, more examples exist, and many third-party tools (security scanners, faster runners) integrate with it.

### What you give up

| Weakness | What it means in practice | Standard mitigation |
|---|---|---|
| **Locked to GitHub** | Moving hosting means rewriting pipelines. | Keep real logic in scripts or a task runner (Chapter 10); keep YAML thin. |
| **Reuse takes effort** | Many near-identical workflows across repos is common. | Composite actions and reusable workflows (Chapter 8). They help a great deal, but are less flexible than a full programming language. |
| **Slow feedback loop** | Edit → push → wait → read logs. Debugging runs on a remote machine. | Lint workflows locally, run them with `act`, move logic into scripts you can run on your laptop, use debug logging (Chapter 10). |
| **No built-in interactive shell** | You cannot SSH into a failing runner by default (CircleCI offers "rerun with SSH"). | Third-party breakpoint actions; self-hosted runners; reproducing locally. |
| **Basic observability** | Per-run logs and durations, but limited trends or resource graphs. | Third-party runner dashboards or exporting timing data (Chapters 10 and 14). GitHub has been adding analytics over time, so check current features. |

### The alternatives in one table

| | GitHub Actions | GitLab CI/CD | CircleCI | Jenkins |
|---|---|---|---|---|
| **Hosting** | Hosted by GitHub; self-hosted runners possible | Hosted (GitLab.com) or self-managed | Hosted (cloud) or self-hosted runners | You operate the server |
| **Tied to a code host?** | GitHub only | GitLab only | GitHub, GitLab, Bitbucket | Any Git host |
| **Config format** | YAML | YAML (with `extends`, includes) | YAML (with orbs) | Groovy `Jenkinsfile` (or UI) |
| **Strengths** | Zero setup, huge marketplace | One platform for code, issues, CI, deployment; Auto DevOps templates | Mature caching and test-splitting, multi-host | Maximum flexibility, thousands of plugins, long enterprise track record |
| **Costs** | Free minutes depending on plan; free for public repos on standard runners | Plan-dependent | Usage-based; can grow at scale | Free software, but you pay in operations time |
| **Watch out for** | Lock-in, debug loop | Lock-in | Separate account and billing | Plugin sprawl and security patching |

(Prices, free tiers and limits change often. Check each vendor's current pricing page rather than relying on figures in notes.)

### A decision guide

| Your situation | Sensible default |
|---|---|
| Code on GitHub, no strong constraints | GitHub Actions |
| Code on GitLab | GitLab CI/CD |
| Code split across hosts, or you want to leave the host later | A host-independent tool (CircleCI, Jenkins, or portable scripts called from any CI) |
| Large existing Jenkins estate that works | Stay until migration clearly pays off |
| Strict data-residency or air-gapped requirements | Self-managed GitLab, Jenkins, or GitHub Actions with self-hosted runners |

## 3. Mental model

Choosing a CI tool is like choosing a **phone plan bundled with your phone**: the bundled one (Actions for GitHub) is easiest to start with and usually good enough; you leave it only for a specific reason, and the more you build on its unique features, the harder leaving becomes.

## 4. Practical example: reducing lock-in

If your pipeline logic is all inline YAML, switching tools means rewriting it. If the YAML only calls scripts, the CI file stays small and portable:

```yaml
# Thin workflow: all logic lives in a script you can run locally
name: CI
on:
  pull_request:
permissions:
  contents: read
jobs:
  check:
    runs-on: ubuntu-24.04
    steps:
      - uses: actions/checkout@v5
      - run: ./scripts/ci.sh
```

The same `./scripts/ci.sh` runs on a laptop and could run under another CI system. Chapter 10 builds on this idea with a task runner.

## 5. Execution walkthrough

When this workflow runs, the runner checks out the repository and then executes `./scripts/ci.sh` as one process. If the script exits non-zero, the step fails and so does the job. The workflow itself contains no test logic, so all debugging can happen on your machine first.

## 6. Common mistakes

| Mistake | Better |
|---|---|
| Picking a tool because of a ranking in a survey | Match the tool to your host, team and constraints. |
| Writing hundreds of lines of Bash inside YAML | Move logic to versioned scripts or a task runner. |
| Copy-pasting workflows across ten repositories | Use reusable workflows or composite actions, and version them. |
| Ignoring runner cost until the bill arrives | Understand minutes, runner sizes and caching early. |

## 7. Security considerations

- Marketplace convenience is also supply-chain exposure: every third-party action runs with access to your repository and possibly secrets. Chapter 7 covers evaluating and pinning actions.
- Self-hosted runners and self-managed Jenkins make you responsible for patching and isolation.

## 8. Hands-on exercise

**Goal:** make and justify a tool decision.

1. Describe a real or imagined project in four lines: where the code is hosted, team size, deployment target, any compliance constraint.
2. Using the decision guide, choose a tool.
3. Write down the **two biggest costs** of your choice and one mitigation for each.

**Success looks like:** a half-page decision note. Keep it; you can reuse the format for architecture decision records.

## 9. Verification checklist

- [ ] Your note names the hosting constraint that drove the decision.
- [ ] You listed concrete costs, not only benefits.
- [ ] You can explain what would make you reconsider.

## 10. Knowledge check

1. Name two ways to reduce lock-in to a CI provider.
2. Why is a long Bash script inside YAML hard to debug?
3. What does CircleCI offer that GitHub Actions does not offer natively, and how do GitHub users work around it?
4. When is staying on Jenkins a reasonable decision?

<details>
<summary>Answers</summary>

1. Keep logic in scripts or a task runner; keep the workflow file as a thin caller.
2. It only runs on the remote runner, so each attempt costs a push and a wait.
3. Interactive "rerun with SSH". Workarounds: third-party breakpoint actions, self-hosted runners, reproducing locally.
4. When it works, the team knows it, and the migration cost outweighs the benefit.
</details>

## 11. Key takeaways

- GitHub Actions wins on adoption cost and ecosystem; it loses on portability, debugging and (some) reuse.
- Every CI tool has trade-offs; choose using your constraints.
- Thin workflows calling scripts give you portability and a fast feedback loop.

## Sources and further reading

- Original notes: `git show master:03_why_github_actions.md`. Vendor-specific claims from the notes (marketplace counts, "fewer than 500 GitLab components") were dropped because they could not be verified and go stale quickly.
- Official docs: [GitHub Actions](https://docs.github.com/en/actions); compare with the current documentation of each alternative before deciding.
