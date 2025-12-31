# Chapter 11: Best Practices - Building Production-Grade Workflows

## Overview

Creating workflows that "just work" is one thing. Building **production-grade automation** that's performant, maintainable, and secure requires intentional design and adherence to best practices.

This chapter covers principles and patterns that separate amateur workflows from professional ones. You'll learn:
- **Performance optimization** - Making workflows fast and efficient
- **Maintainability patterns** - Keeping workflows manageable as they grow
- **Security practices** - Protecting credentials and preventing attacks
- **Monorepo vs. multirepo considerations** - Architectural tradeoffs

**Goal**: Apply these practices in your projects to build robust, scalable automation.

## Category 1: Performance Best Practices

### Principle 1: Measure Before Optimizing

**Rule**: Never optimize without data.

**Why**: Guessing where time is spent leads to wasted effort. Measure first.

**How to measure**:
1. **Export timing data** - Use Honeycomb (Chapter 10) or runner insights
2. **Identify bottlenecks** - Which steps take 80% of time?
3. **Set baseline** - Record current performance
4. **Optimize targeted** - Focus on slowest parts
5. **Validate improvement** - Measure again, confirm speedup

**Example workflow**:
```
Current state: Workflow takes 8 minutes
└─ Measure: Install dependencies = 5 minutes (62% of time)
   └─ Optimize: Add caching
      └─ New measure: Install dependencies = 30 seconds
         └─ Result: Workflow now 3.5 minutes (56% faster)
```

**Anti-pattern**: "Let's add caching everywhere" without knowing what's slow.

### Principle 2: Spend Less Time Waiting

**Subcategory A: Reduce Queue Time**

**Problem**: Jobs wait minutes for available runner.

**Solutions**:
- **Use faster runners** - Namespace, BuildJet (instant provisioning)
- **Add self-hosted runners** - No queue for dedicated hardware
- **Stagger workflows** - Use `concurrency` to limit simultaneous runs

**Subcategory B: Fail Fast**

**Problem**: Workflow runs 10 minutes, then fails on final step.

**Solution**: Run likely-to-fail tests first.

**Example ordering**:
```yaml
steps:
  # 1. Fastest checks first
  - name: Lint (10 seconds, often fails)
    run: npm run lint
  
  # 2. Unit tests (2 minutes, occasionally fail)
  - name: Unit tests
    run: npm test
  
  # 3. Build (3 minutes, rarely fails)
  - name: Build
    run: npm run build
  
  # 4. Integration tests (5 minutes, rarely fail)
  - name: Integration tests
    run: npm run test:integration
```

**Rationale**: Catch syntax errors in 10 seconds, not 10 minutes.

### Principle 3: Do Less Work

**Technique A: Conditional Execution**

**Path filters** (workflow level):
```yaml
on:
  push:
    paths:
      - 'src/**'
      - 'package.json'
```

**Effect**: Only runs when relevant files change.

**Path filters** (job level):
```yaml
jobs:
  frontend:
    if: contains(github.event.head_commit.message, '[frontend]')
    # ...
```

**Changed file detection** (step level):
```yaml
- name: Detect changes
  id: changes
  uses: dorny/paths-filter@v2
  with:
    filters: |
      frontend:
        - 'frontend/**'
      backend:
        - 'backend/**'

- name: Build frontend
  if: steps.changes.outputs.frontend == 'true'
  run: npm run build
```

**Technique B: Caching**

**Avoid repeating work from previous runs**.

**Dependency caching** (built into setup actions):
```yaml
- uses: actions/setup-node@v4
  with:
    node-version: '20'
    cache: 'npm'  # Caches node_modules
```

**Manual caching**:
```yaml
- name: Cache build artifacts
  uses: actions/cache@v3
  with:
    path: dist/
    key: build-${{ hashFiles('src/**') }}
```

**When cache hit**: Restores in seconds instead of rebuilding.

### Principle 4: Improve Resource Utilization

**Technique A: Parallelize Jobs**

**Sequential** (slow):
```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - run: npm test
  
  lint:
    needs: test  # Waits for test
    runs-on: ubuntu-latest
    steps:
      - run: npm run lint
```

**Total time**: test time + lint time

**Parallel** (fast):
```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - run: npm test
  
  lint:
    runs-on: ubuntu-latest  # No dependency
    steps:
      - run: npm run lint
```

**Total time**: max(test time, lint time)

**Technique B: Parallelize Within Job (Matrix)**

**Sequential builds**:
```yaml
steps:
  - run: npm run build:linux
  - run: npm run build:mac
  - run: npm run build:windows
```

**Matrix builds** (parallel):
```yaml
strategy:
  matrix:
    os: [ubuntu-latest, macos-latest, windows-latest]
runs-on: ${{ matrix.os }}
steps:
  - run: npm run build
```

**Technique C: Avoid Emulation**

**Problem**: Building ARM64 image on AMD64 runner (or vice versa) uses QEMU emulation.

**Cost**: 5-10x slower than native builds.

**Solution A**: Use native architecture runners
```yaml
jobs:
  build-amd64:
    runs-on: ubuntu-latest  # AMD64
    steps:
      - run: docker build --platform linux/amd64 .
  
  build-arm64:
    runs-on: ubuntu-latest-arm  # ARM64
    steps:
      - run: docker build --platform linux/arm64 .
```

**Solution B**: Remote builders (Namespace, Depot)
- Build both architectures natively in parallel
- Merge into multi-arch image
- 10x faster than emulation

### Principle 5: Monitor Performance Over Time

**Problem**: Workflows get slower as project grows.

**Solution**: Track trends, catch regressions early.

**Approaches**:
1. **Weekly reviews** - Check Honeycomb dashboards
2. **Automated alerts** - Notify when p95 exceeds threshold
3. **Performance budgets** - "CI must complete <5 minutes"

**Example alert**:
```
⚠️ Performance Regression Detected
Workflow: Build and Test
Average duration: 6m 32s (was 4m 12s last week)
Slowest step: Install dependencies (4m 15s, +3 minutes)
Action: Investigate dependency caching
```

## Caching Strategies: What to Cache

### 1. Git Checkout

**Use case**: Large repositories (>1GB).

**Benefit**: Only pull delta, not full history.

**Implementation**:
```yaml
- uses: actions/cache@v3
  with:
    path: .git
    key: git-${{ github.sha }}
    restore-keys: git-
```

### 2. Language Toolchain

**Use case**: Custom version of Node.js, Python, Go, etc.

**Benefit**: Avoid downloading/installing each run.

**Example** (custom Node.js):
```yaml
- uses: actions/cache@v3
  with:
    path: ~/.nvm
    key: node-${{ hashFiles('.nvmrc') }}
```

### 3. Dependency Downloads

**Most impactful caching**.

**Built-in** (setup actions):
```yaml
- uses: actions/setup-node@v4
  with:
    cache: 'npm'

- uses: actions/setup-python@v4
  with:
    cache: 'pip'

- uses: actions/setup-go@v4
  with:
    cache: true
```

**Manual** (custom package managers):
```yaml
- uses: actions/cache@v3
  with:
    path: ~/.cargo
    key: cargo-${{ hashFiles('Cargo.lock') }}
```

### 4. Build and Test Artifacts

**Use case**: Expensive compilation outputs.

**Example**:
```yaml
- uses: actions/cache@v3
  with:
    path: target/release/
    key: build-${{ hashFiles('src/**', 'Cargo.toml') }}
```

### 5. Container Images and Layers

**A. Base Image Caching**

**Problem**: Pulling `node:18` every build.

**Solution**: Runner-level cache (automatic with most runners).

**B. Layer Caching**

**Problem**: Rebuilding unchanged layers.

**Solution**: Docker Build Cache.

**GitHub Actions cache**:
```yaml
- uses: docker/build-push-action@v5
  with:
    cache-from: type=gha
    cache-to: type=gha,mode=max
```

**Registry cache**:
```yaml
- uses: docker/build-push-action@v5
  with:
    cache-from: type=registry,ref=ghcr.io/user/app:buildcache
    cache-to: type=registry,ref=ghcr.io/user/app:buildcache,mode=max
```

**C. Cache Mounts** (BuildKit)

**Use case**: npm cache within Dockerfile.

**Dockerfile**:
```dockerfile
RUN --mount=type=cache,target=/root/.npm \
    npm install
```

**Effect**: npm cache persists across builds, speeds up `npm install` dramatically.

## Category 2: Maintainability Best Practices

### Principle 1: Define Standard CI/CD API

**Problem**: Each service has different build/test commands.

**Solution**: Standardize interface with task runner.

**Example** (taskfile.yml):
```yaml
version: '3'

tasks:
  install:
    desc: Install dependencies
    cmds:
      - npm ci
  
  test:
    desc: Run tests
    cmds:
      - npm test
  
  build:
    desc: Build application
    cmds:
      - npm run build
  
  dev:
    desc: Start development server
    cmds:
      - npm run dev
```

**Every service has same commands**:
- `task install`
- `task test`
- `task build`
- `task dev`

**Workflow benefits**:
```yaml
steps:
  - uses: actions/checkout@v4
  - run: task install
  - run: task test
  - run: task build
```

**Works for any service** - frontend, backend, microservice, library.

**Developer benefits**:
- Clone any repo
- Run `task dev`
- Immediately productive

### Principle 2: Use Composite Actions and Reusable Workflows

**Avoid copy-paste**.

**Composite action** (`.github/actions/setup-dependencies/action.yml`):
```yaml
name: Setup Dependencies
runs:
  using: composite
  steps:
    - uses: actions/setup-node@v4
      with:
        node-version-file: '.nvmrc'
        cache: 'npm'
    
    - name: Install task
      shell: bash
      run: |
        wget -qO- https://taskfile.dev/install.sh | sh
        sudo mv ./bin/task /usr/local/bin/
    
    - run: task install
      shell: bash
```

**Usage** (any workflow):
```yaml
steps:
  - uses: actions/checkout@v4
  - uses: ./.github/actions/setup-dependencies
  - run: task test
```

**Benefit**: Change setup once, applies everywhere.

### Principle 3: Set Up Local Development Tooling

**From Chapter 10**:
- VS Code extensions (GitHub Actions, YAML, Rainbow Indent)
- Act for local workflow testing
- Taskfile for extracting logic from YAML

**Why this matters for maintainability**:
- Faster iteration = more time for quality improvements
- Easier to refactor with quick feedback
- Lower barrier for team members to contribute

## Decision Frameworks (Revisited)

### Reusability Framework

```
Do I need to reuse logic across workflows/repos?
├─ NO → Standard workflow (no abstraction)
└─ YES → Does it need to enforce standardization?
    ├─ YES → Reusable workflow
    └─ NO → Composite action
```

### Implementation Framework

```
Found reputable marketplace action?
├─ YES → Use it
└─ NO → Is logic simple (1-2 commands)?
    ├─ YES → Inline bash
    └─ NO → Complex logic needed?
        ├─ BASH SUFFICIENT → External task file
        └─ NEED EXPRESSIVITY → Need dependencies?
            ├─ NO → In-repo script (Python/Node)
            └─ YES → Custom action
                ├─ JAVASCRIPT → JS/TS action
                └─ OTHER → Container action
```

## Category 3: Monorepo vs. Multirepo Considerations

### Comparison Matrix

| Aspect | Monorepo | Multirepo |
|--------|----------|-----------|
| **Reusing Logic** | Easy (relative paths) | Harder (requires shared repo) |
| **Discoverability** | Easy (one codebase search) | Harder (document patterns) |
| **Performance (Running Only What's Needed)** | Hard (path filtering required) | Easy (repo boundaries) |
| **Caching** | Hard (scoped keys needed) | Easy (automatic isolation) |
| **Artifacts** | Easy (within repo) | Hard (external registry) |
| **Permissions** | Easier (one repo) | Granular (per-repo) |
| **Required Checks** | Hard (granularity limited) | Easy (per-repo config) |
| **Scale** | Risk of runner contention | Natural isolation |
| **Dependency Updates** | Atomic (one PR) | Scattered (many PRs) |
| **Auditing** | Easy (one place) | Hard (many places) |
| **Ownership** | Requires CODEOWNERS | Clear by repo |

### Monorepo: Key Considerations

**Reusing logic**: ✅ Easy
```yaml
- uses: ./.github/actions/shared-setup
```

**Path filtering**: ⚠️ Critical
```yaml
on:
  push:
    paths:
      - 'services/api/**'
      - '.github/workflows/api.yml'
```

**Without filtering**: Every push triggers all workflows.

**Caching**: ⚠️ Needs scoping
```yaml
- uses: actions/cache@v3
  with:
    path: services/api/node_modules
    key: api-deps-${{ hashFiles('services/api/package-lock.json') }}
```

**Without scoping**: Cache collisions between services.

**Dependency updates**: ✅ Atomic
- Single Dependabot PR updates all services
- Test all together, merge atomically

### Multirepo: Key Considerations

**Reusing logic**: ⚠️ Requires shared repo
```yaml
- uses: org/devops-actions/.github/actions/setup@main
```

**Discoverability**: ⚠️ Needs documentation
- Maintain catalog of shared workflows/actions
- Onboarding docs for new services

**Caching**: ✅ Automatic isolation
- Each repo has own cache namespace
- No key collision risk

**Dependency updates**: ⚠️ Scattered
- Dependabot creates PRs in each repo
- Need tooling to track rollout across repos

**Permissions**: ✅ Granular
- Production repo: strict approvals
- Staging repo: auto-deploy
- Easy to configure per-repo

## Category 4: Security Best Practices

### Principle 1: Minimum Permissions

**Default** (too permissive):
```yaml
# Implicit: permissions: read-write for all scopes
```

**Better** (explicit minimum):
```yaml
permissions:
  contents: read        # Read repo
  pull-requests: write  # Comment on PRs
```

**Best** (per-job permissions):
```yaml
permissions: {}  # Workflow default: none

jobs:
  build:
    permissions:
      contents: read  # Only read access
    steps:
      - uses: actions/checkout@v4
  
  deploy:
    permissions:
      contents: read
      id-token: write  # OIDC token
    steps:
      # Deploy steps
```

### Principle 2: Avoid Long-Lived Credentials

**Anti-pattern**: Static AWS keys
```yaml
- uses: aws-actions/configure-aws-credentials@v4
  with:
    aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
    aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
```

**Problems**:
- Keys don't expire
- Rotation requires secret updates
- Broader blast radius if leaked

**Better**: OIDC (short-lived tokens)
```yaml
- uses: aws-actions/configure-aws-credentials@v4
  with:
    role-to-assume: arn:aws:iam::123456789012:role/GitHubActions
    aws-region: us-east-1
```

**Benefits**:
- Tokens expire after job
- No secrets to rotate
- Audit trail in AWS CloudTrail

### Principle 3: Action Allow Lists

**Prevent unapproved actions**.

**Organization settings**:
1. Settings → Actions → General
2. "Allow select actions"
3. Specify:
   - `actions/*` (GitHub official)
   - `org/repo/*` (your organization)
   - `docker/*` (verified publishers)

**Effect**: Block unknown/unvetted actions.

### Principle 4: Pin Actions to Commit SHA

**Vulnerable** (tag can be moved):
```yaml
- uses: some-action@v1.2.3
```

**Attack**: Maintainer or attacker moves tag to malicious code.

**Secure** (immutable commit):
```yaml
- uses: some-action@abc123def456...  # Full commit SHA
```

**Effect**: Always runs exact code, even if tag moved.

**Maintenance**: Use Dependabot to update SHAs automatically.

### Principle 5: Restrict Self-Hosted Runner Access

**Danger**: Fork PRs running on self-hosted runners.

**Attack scenario**:
1. Attacker forks public repo
2. Modifies workflow to exfiltrate secrets
3. Opens PR
4. Workflow runs on your self-hosted runner
5. Secrets exposed

**Solution**: Disable fork PR access

**Settings → Actions → General**:
- ☐ Run workflows from fork pull requests

**For public repos with self-hosted runners**: Critical security control.

### Principle 6: Use Environment Protection

**Protect production deployments**.

**Environment configuration**:
1. Settings → Environments → New environment
2. Name: "production"
3. Protection rules:
   - ✅ Required reviewers: @senior-eng, @ops-team
   - ✅ Wait timer: 5 minutes
   - ✅ Allowed branches: `main` only

**Workflow**:
```yaml
jobs:
  deploy-production:
    runs-on: ubuntu-latest
    environment: production  # Requires approval
    steps:
      - name: Deploy
        run: kubectl apply -f production/
```

**Effect**: Deploy to production only after human approval.

## Best Practices Summary Checklist

### Performance
- [ ] Measure workflow performance (Honeycomb or runner insights)
- [ ] Enable dependency caching (setup action `cache:` parameter)
- [ ] Use path filters to skip unnecessary runs
- [ ] Fail fast (run linting before slow tests)
- [ ] Parallelize independent jobs
- [ ] Use matrix strategies for similar jobs
- [ ] Avoid emulation (use native architecture runners)
- [ ] Monitor trends over time

### Maintainability
- [ ] Define standard task commands (install, test, build, dev)
- [ ] Extract repeated logic into composite actions
- [ ] Use reusable workflows for standardized processes
- [ ] Set up local development tools (Act, VS Code extensions)
- [ ] Document shared actions/workflows (if multirepo)
- [ ] Version shared workflows (tag releases)

### Security
- [ ] Set minimum permissions per job
- [ ] Use OIDC instead of static credentials
- [ ] Pin actions to commit SHAs
- [ ] Set up action allow lists (organizations)
- [ ] Disable fork PR access for self-hosted runners
- [ ] Require approvals for production environments
- [ ] Rotate secrets regularly (or eliminate with OIDC)
- [ ] Audit third-party action code before using

### Architecture
- [ ] Choose monorepo or multirepo based on team size/structure
- [ ] If monorepo: implement robust path filtering
- [ ] If monorepo: scope cache keys carefully
- [ ] If multirepo: create shared DevOps repo
- [ ] If multirepo: document reusable workflows
- [ ] Define ownership (CODEOWNERS file)

## Key Takeaways

**Performance**:
1. Measure before optimizing
2. Fail fast, cache aggressively
3. Parallelize everything possible

**Maintainability**:
1. Standardize CI/CD interface (task commands)
2. DRY principle (composite actions, reusable workflows)
3. Invest in developer tooling

**Security**:
1. Minimum permissions, always
2. Short-lived credentials (OIDC)
3. Pin and vet all actions
4. Protect production with approvals

**Architecture**:
- Monorepo: Great for simplicity, needs careful path filtering
- Multirepo: Better isolation, needs shared workflow repo

**Philosophy**: Start simple, add complexity only when needed. Optimize for team velocity and security.

## Next Chapter

You've learned the principles for building production-grade workflows. Now it's time to **apply everything**. 

Chapters 12-19 form the **Capstone Project**—a comprehensive real-world application demonstrating all concepts from this course. You'll build complete CI/CD automation for a multi-service application, including:
- Test workflows with caching and parallelization
- Container image builds with multi-arch support
- GitOps-based deployments
- Automated releases with conventional commits
- Repository automation (stale issues)
- Performance monitoring
- All while following best practices for security and maintainability

This hands-on project ties together GitHub Actions features, marketplace actions, custom actions, performance optimization, and security practices into a cohesive, production-ready system. Let's build something real.
