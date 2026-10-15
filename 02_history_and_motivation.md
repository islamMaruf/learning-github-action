# Chapter 2: Why CI/CD Exists

> **Level:** beginner · **Time:** 40 minutes · **Prerequisites:** [Chapter 1](01_introduction.md)
> **Next:** [Chapter 3: Why GitHub Actions?](03_why_github_actions.md)

## Learning objectives

After this chapter you can:

- Explain why shipping software faster forced teams to automate testing and deployment.
- Place the major CI tools in time and say what each one added.
- Name the four kinds of work CI/CD systems automate.
- Contrast push-based deployment with GitOps (pull-based) deployment.

## 1. Problem statement

Understanding *why* a tool looks the way it does makes its rules easier to remember. GitHub Actions did not appear from nowhere: it combines ideas from twenty years of build servers. And the four kinds of work it automates (check, build, deploy, maintain) are the same in every CI/CD system, so this vocabulary transfers to any tool you meet later.

## 2. Core explanation

### How delivery speed created the need for automation

| Era | How software reached users | Typical release rhythm | Consequence for testing |
|---|---|---|---|
| Early mainframes | Programs run by the people who wrote them (punch cards, batch jobs). | Weeks to months, limited by access to machines. | Debugging meant waiting for the next machine slot. |
| Boxed software | Floppies, CDs, DVDs, shipped by mail. | Months to years. A bug fix meant manufacturing new media. | Mistakes were expensive, so testing was careful but largely manual. |
| Web and downloads | Servers updated centrally; downloads over the internet. | Weekly or faster. | Manual testing became the bottleneck. Automated tests became essential. |
| Cloud era | Continuous delivery to servers and app stores. | Many times per day for strong teams. | Automation is mandatory: people cannot keep pace with that volume of changes. |

The pattern: **every time the cost of releasing dropped, the amount of checking per release had to be automated**, otherwise quality collapsed or releases stalled.

### Before CI: "integration hell"

Developers worked on long-lived branches for weeks, then tried to merge everything at once. Conflicts were large, bugs surfaced long after they were introduced, and "it works on my machine" was a daily problem because there was no shared, repeatable build.

**Continuous Integration** is the opposite habit: integrate small changes often, and verify each one automatically.

### A short timeline of CI tools

| Year (approx.) | Tool | What it added |
|---|---|---|
| 1997 | Tinderbox (Netscape/Mozilla) | Automated builds on multiple platforms with a web page showing which commit broke the build. |
| 2001 | CruiseControl (ThoughtWorks) | Build + test as one automated loop for Java projects; plugins; failure notifications. |
| 2005 | Hudson, later forked as Jenkins | A friendly web UI, a huge plugin ecosystem, distributed builds. Flexible, but you run and patch the server yourself. |
| 2006–2007 | Bamboo (Atlassian), TeamCity (JetBrains) | CI tied closely to the issue tracker, code host and IDE. |
| 2011 | Travis CI, CircleCI | CI as a hosted service. Configuration lives in the repository (`.travis.yml`, `config.yml`); free for open source. |
| 2015+ | GitLab CI/CD; cloud-provider build services (AWS, Google Cloud, Azure) | CI built into the code platform or the cloud platform. |
| 2018–2019 | GitHub Actions | Announced in 2018 and generally available in late 2019. Automation built into the world's largest code host, with a public marketplace of reusable actions. |

The trend: from *a server you operate* to *a service you configure with a file in your repository*.

### The four kinds of work CI/CD automates

1. **Validation.** Linting, unit/integration/end-to-end tests, type checks, static analysis, dependency and security scans. Goal: catch defects before they are merged.
2. **Building artifacts.** Compile code, bundle dependencies, build container images, tag them with a version. Goal: a reproducible, versioned thing that can be deployed.
3. **Deployment.** Get the artifact running somewhere. (See below.)
4. **Repository automation.** Releases and changelogs, stale-issue cleanup, dependency-update pull requests, issue labeling.

### Two deployment models

| | Push-based | GitOps (pull-based) |
|---|---|---|
| Who acts | The CI system authenticates to the target and applies the change. | CI only updates a Git repository that describes the desired state. An agent inside the target (for example Argo CD or Flux for Kubernetes) notices the change and applies it. |
| Credentials | CI holds credentials that can change production. | Production credentials stay inside production; CI needs only Git access. |
| Audit and rollback | Depends on your scripts and logs. | Every change is a Git commit; rollback is a revert. |
| Cost | Simple to set up. | An extra component to run and understand. |

Neither is "right". Push-based is fine for many small systems with strong protection on production credentials; GitOps shines when many services deploy to a cluster.

## 3. Mental model

CI/CD is an **assembly line with quality inspectors at each station**. Early manufacturing inspected the finished product once, at the end. Modern lines inspect at every station, so a defect is caught where it is cheapest to fix. Pull requests are the stations; workflows are the inspectors.

## 4. Practical example: the same pipeline in four stages

A conceptual flow (not yet code) that you will build in the capstone:

```
developer pushes ──▶ validate (lint, test) ──▶ build image ──▶ deploy to staging ──▶ (approval) ──▶ deploy to production
        │                    │                      │                 │
        └─ event             └─ fails fast          └─ versioned      └─ verify health
```

## 5. Execution walkthrough

Take one change through the line:

1. A developer opens a pull request. Validation workflows run; a failing test blocks the merge.
2. After the merge to `main`, a build workflow creates an image tagged with the commit identifier.
3. A deployment step (push-based) or a manifest update (GitOps) puts that exact version in staging.
4. A health check confirms the new version is serving traffic.
5. A person approves, and the same image (not a rebuild) is promoted to production.

Step 5 matters: **build once, promote the same artifact**, so what you tested is what you ship.

## 6. Common mistakes

| Mistake | Why it hurts | Better |
|---|---|---|
| Rebuilding the artifact for each environment | Production may run something that was never tested. | Build once, deploy the same image everywhere. |
| Treating CI as only "run the tests" | Misses build, release and maintenance automation. | Think in the four categories above. |
| Long-lived feature branches | Recreates integration hell. | Merge small changes often, behind feature flags if needed. |
| Choosing a tool by popularity statistics alone | Surveys differ by audience and age quickly. | Choose by where your code lives, your team's skills and your constraints (Chapter 3). |

## 7. Security considerations

- Push-based deployment means CI holds credentials that can change production. Limit them (Chapter 6 covers short-lived credentials via OIDC), scope them per environment, and require approval for production.
- GitOps shrinks that exposure but moves trust to the repository: protect the manifest repository's branches as strictly as production itself.

## 8. Hands-on exercise

**Goal:** classify real automation in a project you know.

1. Pick any repository you work on, or a well-known open-source project that has a `.github/workflows` directory.
2. List every workflow file and write, next to each, which of the four categories it belongs to.
3. For one deployment workflow, decide whether it is push-based or GitOps-style and write down which credentials it needs.

**Success looks like:** a table with one row per workflow, a category, and the trigger event. Workflows that fit none of the four categories deserve a note: they are usually repository chores.

## 9. Verification checklist

- [ ] You can name at least three tools from the timeline and what each introduced.
- [ ] Your table names the trigger for every workflow.
- [ ] You can explain in one sentence why "build once, promote the same artifact" is safer.

## 10. Knowledge check

1. Why did faster releases make automated testing unavoidable?
2. What problem did Travis CI and CircleCI remove compared with running Jenkins yourself?
3. Give one advantage and one cost of GitOps compared with push-based deployment.
4. Which of the four categories does "close issues with no activity for 90 days" belong to?

<details>
<summary>Answers</summary>

1. Manual testing cannot keep pace with many releases per week or day.
2. Operating and patching a build server; they provided hosted runners and in-repository configuration.
3. Advantage: production credentials stay inside the environment, and every change is an auditable Git commit. Cost: an additional agent and repository to operate.
4. Repository automation.
</details>

## 11. Key takeaways

- CI/CD exists because releasing got cheap and manual checking did not scale.
- CI tools moved from self-operated servers to configuration files in the repository.
- Four categories of work: validate, build, deploy, maintain.
- Push-based and GitOps deployment trade simplicity against credential exposure and auditability.

## Sources and further reading

- Original notes: `git show master:02_history_and_motivation.md`. The unsourced statistics from those notes (deployment-frequency claims, survey rankings) were removed rather than repeated; the tool timeline was kept and checked for plausibility, but dates are approximate.
- Official docs: [Understanding GitHub Actions](https://docs.github.com/en/actions) (start page).
