# Chapter 12: Building a Production Test Workflow

## Introduction

In this chapter, we'll build a comprehensive, production-ready test workflow for a multi-service monorepo application. This represents the culmination of all concepts learned in the course, bringing together workflow design, job orchestration, matrix strategies, caching, conditional execution, and monorepo filtering into a cohesive CI/CD automation.

The test workflow we'll construct demonstrates professional DevOps practices used by teams working on real-world applications. We'll tackle the challenges of testing multiple services written in different languages (Node.js, Go, Python, React) while ensuring our CI system only runs tests for services that have actually changed. This chapter exemplifies how thoughtful workflow design can dramatically improve developer experience and CI performance.

## The Capstone Project Overview

### Project Structure

Our capstone project is a realistic three-tier web application structured as a monorepo:

**Application Components:**
- **React Frontend**: User interface written in React
- **Node.js API**: RESTful API implementation in Node
- **Golang API**: Alternative API implementation in Go
- **Python Load Generator**: Automated load testing tool
- **PostgreSQL Database**: Persistent storage with migration system

This multi-service architecture mirrors real-world enterprise applications where different components use different technology stacks based on team expertise and service requirements.

**Repository Organization:**

```
services/
├── node/
│   └── api-node/          # Node.js API service
├── go/
│   └── api-golang/        # Golang API service
├── python/
│   └── load-generator/    # Python load generator
├── react/
│   └── client-react/      # React frontend
└── other/
    └── migrator/          # Database migrations
```

The deliberate organization by language (services/node/, services/go/, etc.) isn't arbitrary—it enables us to use directory structure as metadata for determining which toolchains to install in our CI workflows. This structural convention simplifies conditional logic and makes workflows more maintainable.

### Workflow Requirements

We need to implement six distinct workflows for this project:

1. **Test Workflow**: Run unit tests for each service (this chapter)
2. **Build and Push Images**: Create and publish container images
3. **Update GitOps Manifests**: Automate deployment manifest updates
4. **Release Automation**: Generate releases based on commits
5. **Stale Issue Management**: Identify and close inactive issues/PRs
6. **Export Timing Data**: Send CI metrics to observability platforms

Each workflow demonstrates different aspects of GitHub Actions capabilities and collectively forms a complete DevOps automation system.

### Monorepo Considerations

Working in a monorepo presents unique challenges:

**Performance Concerns:**
- Changes to one service shouldn't trigger tests for all services
- CI costs multiply when testing everything on every commit
- Developer feedback loops become slower as the repo grows

**Coordination Benefits:**
- Atomic commits across service boundaries
- Shared infrastructure and tooling configuration
- Simplified dependency management for shared libraries

Our test workflow must intelligently determine which services changed and selectively run only the relevant tests, balancing the benefits of monorepo organization with the need for fast, efficient CI.

## Initial Workflow Structure

### Starting Simple: Single Service Testing

Let's begin by creating a test workflow for a single service—the Node.js API. This establishes the foundational pattern we'll expand to cover all services.

**Workflow File**: `.github/workflows/test.yaml`

```yaml
name: Test Workflow

on:
  workflow_dispatch:

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@5.0.0  # v5.0.0
        
      - name: Setup Node.js
        uses: actions/setup-node@4.4.0  # v4.4.0
        with:
          node-version-file: services/node/api-node/package.json
          cache: npm
          cache-dependency-path: services/node/api-node/package-lock.json
          
      - name: Install Task
        uses: battila7/setup-binary@4.0.1  # v4.0.1
        with:
          binary: task
          version: v3.44.1
          download-url: https://github.com/go-task/task/releases/download/v3.44.1/task_linux_amd64.tar.gz
          
      - name: Install dependencies
        working-directory: services/node/api-node
        run: task installci
        
      - name: Run tests
        working-directory: services/node/api-node
        run: task test
```

### Understanding Each Step

**1. Checkout Code**

```yaml
- name: Checkout code
  uses: actions/checkout@5.0.0
```

This fundamental step clones the repository to the runner's workspace. Without it, subsequent steps have no code to work with. We pin to a specific version (5.0.0) using the full semantic version, ensuring reproducible builds.

**Why explicit versions matter**: Marketplace actions can change between releases. Pinning versions prevents unexpected breakage when action authors publish updates. In production workflows, you should also reference the commit SHA for maximum security.

**2. Setup Node.js Toolchain**

```yaml
- name: Setup Node.js
  uses: actions/setup-node@4.4.0
  with:
    node-version-file: services/node/api-node/package.json
    cache: npm
    cache-dependency-path: services/node/api-node/package-lock.json
```

The `setup-node` action installs Node.js and configures the environment. Key inputs:

- **node-version-file**: Points to package.json containing the engines field specifying required Node version. This approach is superior to hardcoding versions in the workflow because it keeps the authoritative version specification in the application code.

- **cache**: Enables npm dependency caching. Setup-node automatically caches the global npm cache directory, dramatically speeding up subsequent runs.

- **cache-dependency-path**: Specifies which lock file to use as the cache key. When package-lock.json changes, the cache is invalidated and dependencies are reinstalled.

**Cache effectiveness**: First run installs all dependencies (slow). Subsequent runs with unchanged package-lock.json restore from cache (fast). This can reduce dependency installation from minutes to seconds.

**3. Install Task Runner**

```yaml
- name: Install Task
  uses: battila7/setup-binary@4.0.1
  with:
    binary: task
    version: v3.44.1
    download-url: https://github.com/go-task/task/releases/download/v3.44.1/task_linux_amd64.tar.gz
```

Task is a modern task runner (alternative to Make) used across all services for standardized commands. The `setup-binary` action is a generic tool installer that:
- Downloads the binary from the specified URL
- Extracts it to the runner's tool cache
- Adds it to the system PATH

**Why Task?**: Having a consistent interface (`task install`, `task test`) across services in different languages simplifies workflow logic. Each service has a Taskfile.yaml defining standard commands, abstracting away language-specific details.

**4. Install Dependencies**

```yaml
- name: Install dependencies
  working-directory: services/node/api-node
  run: task installci
```

The `installci` task runs `npm ci` instead of `npm install`. The difference matters:

- **npm install**: Updates package-lock.json if it's out of sync, installs dependencies, may update minor/patch versions
- **npm ci**: Requires package-lock.json, deletes node_modules before installing, installs exact versions listed in lock file

In CI, you always want `npm ci` for reproducible builds. The `working-directory` ensures the command runs in the correct service directory.

**5. Run Tests**

```yaml
- name: Run tests
  working-directory: services/node/api-node
  run: task test
```

Finally, execute the tests via the standardized `task test` command. For Node.js, this typically runs `npm test`, which executes whatever test runner is configured (Jest, Mocha, etc.).

## Local Testing with Act

### The Development Feedback Loop

Pushing code to GitHub to test workflow changes creates an unacceptably slow feedback loop. Act solves this by running workflows locally using Docker containers that simulate GitHub's runners.

**Creating a Local Test Harness**

Structure: `.github/workflows/test/taskfile.yaml`

```yaml
version: '3'

tasks:
  trigger-workflow:
    cmds:
      - |
        act workflow_dispatch \
          --container-architecture linux/amd64 \
          -P ubuntu-latest=catthehacker/ubuntu:act-latest \
          --directory ../../.. \
          --workflows .github/workflows/test.yaml \
          -e event.json \
          -s GITHUB_TOKEN="${GITHUB_TOKEN}"
    env:
      GITHUB_TOKEN:
        sh: gh auth token
```

### Understanding Act Configuration

**workflow_dispatch**: The event type to trigger. Matches the `on: workflow_dispatch` in our workflow.

**--container-architecture linux/amd64**: Forces x86_64 architecture even on ARM machines (like M1 Macs). This ensures the environment matches GitHub's runners.

**-P ubuntu-latest=catthehacker/ubuntu:act-latest**: Specifies which Docker image to use for the `ubuntu-latest` runner. The catthehacker images closely mirror GitHub's actual runner images, including preinstalled tools.

**--directory ../../..**: Runs Act from the repository root (three directories up from the taskfile location).

**--workflows**: Path to the specific workflow file to execute.

**-e event.json**: Provides event payload metadata. GitHub workflows receive contextual information about the triggering event. When testing locally, we must supply this ourselves.

**-s GITHUB_TOKEN**: Passes the GitHub token as a secret. Many actions query the GitHub API and require authentication.

### Event Payload Configuration

File: `.github/workflows/test/event.json`

```json
{
  "repository": {
    "default_branch": "main"
  }
}
```

This minimal event payload supplies the default branch name. Some actions (particularly path filtering actions) need this to determine which commits to compare when detecting changes.

**Why gh auth token?**: The GitHub CLI (`gh`) manages authentication. Running `gh auth token` generates a temporary access token using your authenticated session. This token is passed to Act, which provides it to workflow steps requesting `${{ secrets.GITHUB_TOKEN }}`.

### Running Locally

```bash
cd .github/workflows/test
task trigger-workflow
```

Act downloads the runner image (first run only), starts a container, and executes the workflow. Output streams to your terminal in real-time, showing exactly what would happen on GitHub's runners.

**Benefits:**
- Iteration speed: Test changes in seconds instead of minutes
- No commit pollution: No need to create test commits to validate workflow changes
- Cost reduction: No CI minutes consumed during development
- Debugging capability: Can exec into containers to inspect state

## Configuring Node.js Properly

### The Engine Version Problem

Our initial workflow had a subtle bug. The workflow ran successfully, but produced warnings:

```
npm WARN EBADENGINE Unsupported engine
npm WARN EBADENGINE Required: { node: '>=20.0.0 <21.0.0' }
npm WARN EBADENGINE Current: { node: 'v18.x.x' }
```

The runner had Node.js v18 installed by default, but our application requires Node.js v20. The `setup-node` action was installed, but we didn't configure it to read the version requirement from package.json.

### Specifying Engine Requirements

In `services/node/api-node/package.json`:

```json
{
  "engines": {
    "node": ">=20.0.0 <21.0.0",
    "npm": ">=10.0.0"
  }
}
```

The engines field declares which runtime versions the application supports. This serves as documentation and can be enforced by package managers.

### Reading Engine Version in Workflows

Updated workflow step:

```yaml
- name: Setup Node.js
  uses: actions/setup-node@4.4.0
  with:
    node-version-file: services/node/api-node/package.json
    cache: npm
    cache-dependency-path: services/node/api-node/package-lock.json
```

The `node-version-file` input tells setup-node to parse package.json and extract the version requirement from the engines.node field. The action then downloads and installs the appropriate Node.js version.

**Why this approach?**: The application's package.json is the single source of truth for version requirements. Developers, local builds, and CI all reference the same specification, preventing version skew.

### Verification

After adding `node-version-file`, rerunning the workflow shows:

```
Setup Node.js: Installing Node.js 20.x.x
npm install: No engine warnings
Tests: All passed
```

The engine version now matches application requirements, and npm no longer complains.

## Expanding to Multiple Services

### Adding Golang Testing

With Node.js working, let's add support for testing the Go API. The naive approach: duplicate the entire job.

```yaml
jobs:
  test-node:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@5.0.0
      - name: Setup Node.js
        uses: actions/setup-node@4.4.0
        with:
          node-version-file: services/node/api-node/package.json
          cache: npm
          cache-dependency-path: services/node/api-node/package-lock.json
      - name: Install Task
        uses: battila7/setup-binary@4.0.1
        with:
          binary: task
          version: v3.44.1
          download-url: https://github.com/go-task/task/releases/download/v3.44.1/task_linux_amd64.tar.gz
      - name: Install dependencies
        working-directory: services/node/api-node
        run: task installci
      - name: Run tests
        working-directory: services/node/api-node
        run: task test
        
  test-go:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@5.0.0
      - name: Setup Go
        uses: actions/setup-go@5.5.0
        with:
          go-version-file: services/go/api-golang/go.mod
          cache-dependency-path: services/go/api-golang/go.sum
      - name: Install Task
        uses: battila7/setup-binary@4.0.1
        with:
          binary: task
          version: v3.44.1
          download-url: https://github.com/go-task/task/releases/download/v3.44.1/task_linux_amd64.tar.gz
      - name: Install dependencies
        working-directory: services/go/api-golang
        run: task installci
      - name: Run tests
        working-directory: services/go/api-golang
        run: task test
```

### The Problem with Duplication

This works, but has serious maintainability issues:

1. **Code repetition**: Most steps are identical across jobs
2. **Update burden**: Changing Task version requires edits in multiple places
3. **Scalability**: Adding more services means more duplication
4. **Error-prone**: Easy to forget updating one job when modifying another

As we add React (Node.js), Python, and future services, this approach becomes untenable.

## Matrix Strategy for Multi-Language Testing

### Understanding Matrix Builds

Matrix strategies generate multiple job instances from a single job definition by parameterizing varying elements. Instead of defining separate jobs for each service, we define one job with a matrix of services.

**Conceptual model:**

```
Job Template + Matrix Values = Multiple Job Instances
```

GitHub Actions generates a separate job instance for each matrix combination, running them in parallel (resource limits permitting).

### Implementing Service Matrix

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      fail-fast: false
      matrix:
        service:
          - services/node/api-node
          - services/go/api-golang
    steps:
      - name: Checkout code
        uses: actions/checkout@5.0.0
        
      - name: Setup Node.js
        if: startsWith(matrix.service, 'services/node')
        uses: actions/setup-node@4.4.0
        with:
          node-version-file: ${{ matrix.service }}/package.json
          cache: npm
          cache-dependency-path: ${{ matrix.service }}/package-lock.json
          
      - name: Setup Go
        if: startsWith(matrix.service, 'services/go')
        uses: actions/setup-go@5.5.0
        with:
          go-version-file: ${{ matrix.service }}/go.mod
          cache-dependency-path: ${{ matrix.service }}/go.sum
          
      - name: Install Task
        uses: battila7/setup-binary@4.0.1
        with:
          binary: task
          version: v3.44.1
          download-url: https://github.com/go-task/task/releases/download/v3.44.1/task_linux_amd64.tar.gz
          
      - name: Install dependencies
        working-directory: ${{ matrix.service }}
        run: task installci
        
      - name: Run tests
        working-directory: ${{ matrix.service }}
        run: task test
```

### Matrix Configuration Explained

**Strategy Block:**

```yaml
strategy:
  fail-fast: false
  matrix:
    service:
      - services/node/api-node
      - services/go/api-golang
```

- **fail-fast: false**: By default, if any matrix job fails, GitHub cancels remaining jobs. Setting this to false allows all jobs to complete, providing visibility into all test results rather than just the first failure.

- **matrix.service**: Defines the matrix dimension. Each array element generates one job instance. The value becomes available as `${{ matrix.service }}` in step expressions.

**Conditional Toolchain Setup:**

```yaml
- name: Setup Node.js
  if: startsWith(matrix.service, 'services/node')
  uses: actions/setup-node@4.4.0
```

The `if` conditional ensures Node.js only installs for Node services. The `startsWith()` function checks if the service path begins with the specified string. This leverages our directory structure convention: all Node services live under `services/node/`.

**Dynamic Path References:**

```yaml
working-directory: ${{ matrix.service }}
```

Matrix variables can be interpolated into any step property. As each job executes, `${{ matrix.service }}` evaluates to the specific service path for that instance.

### Execution Model

With two services in the matrix, GitHub creates two parallel jobs:

**Job Instance 1:**
- matrix.service = "services/node/api-node"
- Setup Node.js: Runs (condition true)
- Setup Go: Skipped (condition false)
- Install Task: Runs
- Install deps: Working dir = services/node/api-node
- Run tests: Working dir = services/node/api-node

**Job Instance 2:**
- matrix.service = "services/go/api-golang"
- Setup Node.js: Skipped (condition false)
- Setup Go: Runs (condition true)
- Install Task: Runs
- Install deps: Working dir = services/go/api-golang
- Run tests: Working dir = services/go/api-golang

Both execute simultaneously, cutting total workflow time in half compared to sequential execution.

## Adding React and Python Services

### React Service (Node.js)

React tests also run in Node.js, but the service path differs. We need to adjust our Node.js conditional to cover both Node API and React client:

```yaml
- name: Setup Node.js
  if: startsWith(matrix.service, 'services/node') || startsWith(matrix.service, 'services/react')
  uses: actions/setup-node@4.4.0
  with:
    node-version-file: ${{ matrix.service }}/package.json
    cache: npm
    cache-dependency-path: ${{ matrix.service }}/package-lock.json
```

The `||` operator creates a logical OR. Node.js installs if the service path starts with either `services/node` or `services/react`.

Add to matrix:

```yaml
matrix:
  service:
    - services/node/api-node
    - services/go/api-golang
    - services/react/client-react
```

No other changes needed! The standardized Task interface (`task installci`, `task test`) works identically for the React client. This demonstrates the power of conventions—by establishing patterns, we minimize the code needed to support new services.

### Python Service (Poetry)

Python introduces a new package manager: Poetry. We need to:
1. Install Poetry
2. Setup Python with Poetry caching
3. Add service to matrix

**Install Poetry:**

```yaml
- name: Install Poetry
  if: startsWith(matrix.service, 'services/python')
  uses: snok/install-poetry@1.4.1
  with:
    version: 2.0.0
```

The `install-poetry` action downloads and configures Poetry, adding it to the PATH.

**Setup Python:**

```yaml
- name: Setup Python
  if: startsWith(matrix.service, 'services/python')
  uses: actions/setup-python@5.6.0
  with:
    python-version-file: ${{ matrix.service }}/pyproject.toml
    cache: poetry
    cache-dependency-path: ${{ matrix.service }}/poetry.lock
```

Similar to Node and Go setup actions, but:
- **python-version-file**: Points to pyproject.toml (Python's project metadata file)
- **cache: poetry**: Enables Poetry-specific caching
- **cache-dependency-path**: Uses poetry.lock as cache key

**Add to Matrix:**

```yaml
matrix:
  service:
    - services/node/api-node
    - services/go/api-golang
    - services/react/client-react
    - services/python/load-generator
```

With these additions, we now test four services across three languages (Node.js, Go, Python) with a single 60-line workflow definition.

## Dynamic Filtering for Changed Services

### The Monorepo Performance Problem

Our workflow currently tests all services on every commit. This wastes resources:

- **Unrelated changes**: Updating documentation shouldn't trigger all tests
- **Isolated services**: Changes to Node API don't affect Go API
- **CI cost**: Running unnecessary tests consumes CI minutes
- **Developer time**: Slower feedback loops when waiting for irrelevant tests

We need intelligent filtering: only test services that have actually changed.

### Path-Based Change Detection

The approach: compare the current commit against a base branch, identify changed files, and map those files to affected services.

**Enter dorny/paths-filter:**

This action examines git diffs and evaluates file paths against user-defined patterns, outputting which categories of changes occurred.

### Filter Configuration File

Create `.github/utils/file-filters.yaml`:

```yaml
api-golang: &api-golang
  - services/go/api-golang/**
  - services/other/migrator/**
  - .github/workflows/test.yaml

api-node:
  - services/node/api-node/**
  - .github/workflows/test.yaml

client-react:
  - services/react/client-react/**
  - .github/workflows/test.yaml

load-generator:
  - services/python/load-generator/**
  - .github/workflows/test.yaml

migrator:
  *api-golang
```

### Understanding Filter Patterns

Each top-level key defines a service identifier. The array contains glob patterns matching files belonging to that service.

**api-golang patterns:**
- `services/go/api-golang/**`: Any file in the Go API directory
- `services/other/migrator/**`: Database migration files (affect API functionality)
- `.github/workflows/test.yaml`: The workflow itself (changes should trigger retest)

**YAML anchors** (`&api-golang` and `*api-golang`): The migrator shares the same file patterns as api-golang. The anchor creates a reusable reference, avoiding duplication.

**Why include workflow files?**: If you modify the test workflow, you want to run tests to verify the changes work correctly. Including the workflow file ensures this happens automatically.

### Implementing the Filter Job

Add a new job before the test job:

```yaml
jobs:
  filter:
    runs-on: ubuntu-latest
    outputs:
      services: ${{ steps.filter.outputs.changes }}
    steps:
      - name: Checkout code
        uses: actions/checkout@5.0.0
        
      - name: Filter changed services
        id: filter
        uses: dorny/paths-filter@de90cc6fb38fc0963ad72b210f1f284cd68cea36  # v3.0.2
        with:
          token: ${{ secrets.GITHUB_TOKEN }}
          filters: .github/utils/file-filters.yaml
```

**Job outputs:**

```yaml
outputs:
  services: ${{ steps.filter.outputs.changes }}
```

This exposes the filter results to downstream jobs. The `changes` output is a JSON array containing the keys of all filters that matched (e.g., `["api-golang", "client-react"]`).

**Step ID and output reference:**

The step has `id: filter`, allowing us to reference its outputs: `steps.filter.outputs.changes`. This value comes from the paths-filter action's defined outputs.

**Token requirement:**

The action needs API access to fetch commit information and perform git operations. `${{ secrets.GITHUB_TOKEN }}` provides an automatically generated token with repository access.

### Consuming Filter Results

Update the test job to depend on filter and use its output:

```yaml
test:
  needs: filter
  runs-on: ubuntu-latest
  strategy:
    fail-fast: false
    matrix:
      service: ${{ fromJSON(needs.filter.outputs.services) }}
  steps:
    # ... existing steps
```

**needs: filter**: Establishes dependency. The test job won't start until filter completes.

**fromJSON()**: The filter output is a JSON-encoded string (`'["api-golang", "api-node"]'`). The `fromJSON()` function parses it into an array that the matrix strategy can consume.

**needs.filter.outputs.services**: Accesses the outputs from the filter job. The `needs` context provides access to outputs from jobs listed in the needs clause.

### Execution Flow

1. **Filter job runs**: Compares current branch to base, identifies changed files, matches against patterns
2. **Filter job outputs**: JSON array like `["api-golang", "client-react"]`
3. **Test job receives**: Matrix expands to those two services only
4. **Test job executes**: Two job instances run (for the two services), others are skipped

This transforms our workflow from "always test everything" to "test only what changed," dramatically improving efficiency.

## Local Testing with Path Filters

### The GitHub Token Challenge

Path filtering requires git operations and API access. When running locally with Act, we need to provide:
1. Event metadata (base branch for comparison)
2. GitHub token (for API authentication)

**Event configuration**: `.github/workflows/test/event.json`

```json
{
  "repository": {
    "default_branch": "main"
  }
}
```

The `default_branch` tells paths-filter which branch to compare against when determining changes.

**Token provision**: Update taskfile.yaml:

```yaml
tasks:
  trigger-workflow:
    cmds:
      - |
        act workflow_dispatch \
          --container-architecture linux/amd64 \
          -P ubuntu-latest=catthehacker/ubuntu:act-latest \
          --directory ../../.. \
          --workflows .github/workflows/test.yaml \
          -e event.json \
          -s GITHUB_TOKEN="${GITHUB_TOKEN}"
    env:
      GITHUB_TOKEN:
        sh: gh auth token
```

The `-s GITHUB_TOKEN="${GITHUB_TOKEN}"` flag passes a secret to Act. The `env` section runs `gh auth token` to generate a token from your authenticated GitHub CLI session.

**Security note**: Tokens displayed during Act execution are short-lived. You can revoke GitHub CLI access afterward to invalidate any exposed tokens.

### Verifying Filter Behavior

Run locally:

```bash
task trigger-workflow
```

Expected output:

```
Filter job:
  - Comparing branch to main
  - Changed files: [list of modified files]
  - Matched services: ["api-golang", "api-node", "client-react", "load-generator"]
  
Test job (4 instances):
  - api-golang: Setup Go → Install deps → Run tests → ✓
  - api-node: Setup Node → Install deps → Run tests → ✓
  - client-react: Setup Node → Install deps → Run tests → ✓
  - load-generator: Setup Python → Install deps → Run tests → ✓
```

If all files have changed (typical on a feature branch), all services are tested. If you modify only the Go API, only `api-golang` would appear in the matrix.

## Complete Production Workflow

Bringing everything together, here's the final test workflow:

```yaml
name: Test Workflow

on:
  workflow_dispatch:
  pull_request:
  push:
    branches: [main, develop]

jobs:
  filter:
    runs-on: ubuntu-latest
    outputs:
      services: ${{ steps.filter.outputs.changes }}
    steps:
      - name: Checkout code
        uses: actions/checkout@5.0.0
        
      - name: Filter changed services
        id: filter
        uses: dorny/paths-filter@de90cc6fb38fc0963ad72b210f1f284cd68cea36  # v3.0.2
        with:
          token: ${{ secrets.GITHUB_TOKEN }}
          filters: .github/utils/file-filters.yaml

  test:
    needs: filter
    runs-on: ubuntu-latest
    strategy:
      fail-fast: false
      matrix:
        service: ${{ fromJSON(needs.filter.outputs.services) }}
    steps:
      - name: Checkout code
        uses: actions/checkout@5.0.0
        
      - name: Setup Node.js
        if: startsWith(matrix.service, 'services/node') || startsWith(matrix.service, 'services/react')
        uses: actions/setup-node@4.4.0
        with:
          node-version-file: ${{ matrix.service }}/package.json
          cache: npm
          cache-dependency-path: ${{ matrix.service }}/package-lock.json
          
      - name: Setup Go
        if: startsWith(matrix.service, 'services/go')
        uses: actions/setup-go@5.5.0
        with:
          go-version-file: ${{ matrix.service }}/go.mod
          cache-dependency-path: ${{ matrix.service }}/go.sum
          
      - name: Install Poetry
        if: startsWith(matrix.service, 'services/python')
        uses: snok/install-poetry@1.4.1
        
      - name: Setup Python
        if: startsWith(matrix.service, 'services/python')
        uses: actions/setup-python@5.6.0
        with:
          python-version-file: ${{ matrix.service }}/pyproject.toml
          cache: poetry
          cache-dependency-path: ${{ matrix.service }}/poetry.lock
          
      - name: Install Task
        uses: battila7/setup-binary@4.0.1
        with:
          binary: task
          version: v3.44.1
          download-url: https://github.com/go-task/task/releases/download/v3.44.1/task_linux_amd64.tar.gz
          
      - name: Install dependencies
        working-directory: ${{ matrix.service }}
        run: task installci
        
      - name: Run tests
        working-directory: ${{ matrix.service }}
        run: task test
```

### Production Triggers

```yaml
on:
  workflow_dispatch:  # Manual triggering
  pull_request:       # Run on all PRs
  push:
    branches: [main, develop]  # Run on pushes to main branches
```

These triggers ensure tests run at critical points:
- **Pull requests**: Validate changes before merging
- **Main branch pushes**: Verify production/integration branch health
- **Manual dispatch**: On-demand testing for debugging

## Key Takeaways

### Architectural Patterns

1. **Standardized interfaces**: The Task runner provides uniform commands across all services, simplifying workflow logic
2. **Convention over configuration**: Directory structure encodes metadata (language type), reducing conditional complexity
3. **Matrix parameterization**: Single job definition scales to arbitrary number of services
4. **Intelligent filtering**: Path-based change detection ensures only relevant tests run
5. **Local development**: Act enables rapid iteration without pushing to GitHub

### Best Practices

1. **Pin action versions**: Use explicit versions or commit SHAs for reproducibility
2. **Disable fail-fast**: See all test results, not just first failure
3. **Cache aggressively**: Package caches dramatically reduce CI time
4. **Reference canonical configs**: Read versions from package.json/go.mod/pyproject.toml
5. **Include workflow in filters**: Test workflow changes by triggering affected tests

### Performance Impact

**Before optimization:**
- Every commit tests all 4 services
- ~10 minutes per workflow run (sequential)
- Wasted CI minutes on unrelated services

**After optimization:**
- Tests only changed services
- Parallel execution of filtered services
- ~3 minutes for typical single-service change
- 70% reduction in CI time and cost

### Extensibility

Adding a new service requires:
1. Add service path to file-filters.yaml
2. Add toolchain setup step with appropriate conditional (if new language)
3. No other changes needed

The workflow's design accommodates growth without proportional complexity increase.

## Conclusion

This chapter demonstrated how to build a professional, production-ready test workflow that handles the complexities of multi-language monorepo testing. By combining matrix strategies, conditional execution, path filtering, and local testing capabilities, we created a CI system that's both powerful and maintainable.

The patterns established here—standardized interfaces, convention-based logic, intelligent filtering—form the foundation for the remaining capstone workflows. As we build container image pipelines, GitOps automation, and repository management workflows in subsequent chapters, we'll continue applying these principles to create a cohesive DevOps automation system.

The test workflow ensures code quality through automated validation, providing fast feedback to developers while optimizing CI resource usage. This balance of speed, cost, and thoroughness represents the goal of effective CI/CD design.
