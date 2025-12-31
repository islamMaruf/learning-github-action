# Chapter 10: Developer Experience - Debugging and Iterating on Workflows

## Overview

One of the **biggest pain points** with GitHub Actions (and CI/CD systems in general) is the slow iteration cycle: make a change → push → wait → see it fail → repeat.

This chapter addresses that frustration by teaching you techniques to **speed up development** and **debug workflows efficiently**.

You'll learn:
- **Iterating on actions** - Testing custom actions locally before pushing
- **Workflow development** - Tools and techniques to test workflows without GitHub
- **Debugging strategies** - SSH access to runners, debug logging, inspecting failures
- **Performance monitoring** - Tracking workflow timing and resource usage

**Goal**: Minimize the painful "push and pray" cycle by testing locally and debugging intelligently.

## The Developer Experience Problem

**Common workflow development cycle**:
1. Write workflow YAML
2. Push commit to GitHub
3. Wait for runner to pick up job
4. Watch workflow fail
5. Try to debug from logs
6. Make small tweak
7. Push again
8. Wait again
9. Repeat 5-10 times

**Problems**:
- **Slow feedback** - Each iteration takes minutes
- **Frustrating** - Small typos require full push/wait cycle
- **Expensive** - Wastes runner minutes on trivial mistakes
- **Confidence-breaking** - String of failures lowers morale

**Solution**: Test locally, debug intelligently, iterate quickly.

## Category 1: Iterating on Actions

When building custom actions (composite, JavaScript, container), you want to **test action logic locally** before integrating into workflows.

### Testing JavaScript/TypeScript Actions Locally

**Tool**: `@github/local-action` npm package

**Purpose**: Runs action code in a GitHub Actions-like environment on your machine.

#### Setup

**Add to package.json**:
```json
{
  "scripts": {
    "local-action": "npx @github/local-action . .env"
  },
  "devDependencies": {
    "@github/local-action": "^2.0.0"
  }
}
```

**Create `.env` file** (input values):
```bash
# Input format: INPUT_<NAME-IN-UPPERCASE>
INPUT_MILLISECONDS=2400
INPUT_WHO_TO_GREET=World
```

**Pattern**: `INPUT_` + uppercase input name (hyphens become underscores).

#### Running Locally

**Command**:
```bash
npm run local-action
```

**What happens**:
1. Loads inputs from `.env` file
2. Executes action entry point (e.g., `src/main.ts`)
3. Captures outputs
4. Shows results in terminal

**Example output**:
```
Loaded inputs:
  milliseconds: 2400

Running action...
Waiting 2400 milliseconds...

Outputs:
  time: 2023-10-15T10:30:45.123Z

✓ Action completed successfully
```

**Benefits**:
- **Fast feedback** - No push/wait cycle
- **Test with different inputs** - Modify `.env` and re-run immediately
- **Debug with breakpoints** - Use VS Code debugger
- **Validate behavior** - Ensure action works before integration

### Unit Testing Actions

**Principle**: Actions are code. Code should have tests.

**Framework**: Jest (Node.js), pytest (Python), go test (Go), etc.

**What to test**:
- Input validation (reject invalid inputs)
- Core logic (business logic correctness)
- Output generation (correct outputs produced)
- Error handling (failures handled gracefully)

#### Example: Jest Tests for TypeScript Action

**Test file** (`__tests__/main.test.ts`):
```typescript
import * as core from '@actions/core';
import { run } from '../src/main';

// Mock @actions/core functions
jest.mock('@actions/core');

describe('Action Tests', () => {
  beforeEach(() => {
    jest.clearAllMocks();
  });

  test('sets time output on success', async () => {
    const getInputMock = jest.spyOn(core, 'getInput').mockReturnValue('1000');
    const setOutputMock = jest.spyOn(core, 'setOutput');

    await run();

    expect(setOutputMock).toHaveBeenCalledWith('time', expect.any(String));
  });

  test('fails with invalid milliseconds input', async () => {
    const getInputMock = jest.spyOn(core, 'getInput').mockReturnValue('invalid');
    const setFailedMock = jest.spyOn(core, 'setFailed');

    await run();

    expect(setFailedMock).toHaveBeenCalledWith(expect.stringContaining('Invalid input'));
  });
});
```

**Running tests**:
```bash
npm test
```

**Benefits**:
- **Prevent regressions** - Tests catch broken changes
- **Document behavior** - Tests serve as usage examples
- **Faster development** - Confidence to refactor
- **CI integration** - Run tests in pull requests

### Testing Workflow Integration

**taskfile.yml** example:
```yaml
version: '3'

tasks:
  test-local:
    desc: Run action locally with test inputs
    dir: .github/actions/my-action
    cmds:
      - npm run local-action

  test-unit:
    desc: Run unit tests
    dir: .github/actions/my-action
    cmds:
      - npm test
```

**Usage**:
```bash
task test-local   # Quick manual test
task test-unit    # Full test suite
```

## Category 2: Iterating on Workflows

Most development time is spent on **workflows**, not actions. Here's how to speed up that process.

### VS Code Extensions for Workflow Development

#### 1. GitHub Actions Extension

**ID**: `GitHub.vscode-github-actions`

**Features**:
- Syntax highlighting for workflow YAML
- IntelliSense for action names/inputs
- Validation of workflow syntax
- Quick links to action documentation

**Installation**:
```bash
code --install-extension GitHub.vscode-github-actions
```

#### 2. YAML Extension

**ID**: `redhat.vscode-yaml`

**Features**:
- YAML formatting
- Schema validation
- Error detection
- Auto-indentation

**Installation**:
```bash
code --install-extension redhat.vscode-yaml
```

#### 3. Rainbow Indent Extension

**ID**: `oderwat.indent-rainbow`

**Features**:
- Color-codes indentation levels
- Makes YAML nesting visible
- Catches indentation errors instantly

**Why it helps**: YAML is indent-sensitive. Rainbow Indent makes structure obvious.

**Visual example**:
```yaml
jobs:           # No color
  build:        # Color 1 (e.g., red)
    runs-on:    # Color 2 (e.g., yellow)
      ubuntu    # Color 3 (e.g., green)
    steps:      # Color 2
      - name:   # Color 3
```

**Installation**:
```bash
code --install-extension oderwat.indent-rainbow
```

### Technique 1: Extract Logic from Workflow YAML

**Problem**: Inline bash in workflows is hard to test.

**Solution**: Pull logic into external scripts/task files.

#### Anti-Pattern: Inline Bash

```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Deploy application
        run: |
          echo "Deploying..."
          if [ "$ENVIRONMENT" == "production" ]; then
            kubectl apply -f prod-config.yaml
            kubectl rollout status deployment/app
          else
            kubectl apply -f staging-config.yaml
          fi
          echo "Deployment complete"
```

**Problems**:
- Can't test without pushing
- No syntax highlighting for bash
- Hard to maintain
- Can't run independently

#### Better Pattern: External Task File

**taskfile.yml**:
```yaml
version: '3'

tasks:
  deploy:
    desc: Deploy application
    vars:
      ENVIRONMENT: '{{.ENVIRONMENT | default "staging"}}'
    cmds:
      - echo "Deploying to {{.ENVIRONMENT}}..."
      - |
        if [ "{{.ENVIRONMENT}}" == "production" ]; then
          kubectl apply -f prod-config.yaml
          kubectl rollout status deployment/app
        else
          kubectl apply -f staging-config.yaml
        fi
      - echo "Deployment complete"
```

**Workflow (simplified)**:
```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Install task
        run: |
          wget -qO- https://taskfile.dev/install.sh | sh
          sudo mv ./bin/task /usr/local/bin/
      
      - name: Deploy
        run: task deploy
        env:
          ENVIRONMENT: ${{ inputs.environment }}
```

**Benefits**:
- **Test locally**: `task deploy ENVIRONMENT=staging`
- **Fast iteration**: Modify task file, re-run immediately
- **Proper editor support**: Syntax highlighting, linting
- **Reusable**: Call from multiple workflows or manually

**Iteration workflow**:
1. Develop logic in task file
2. Test: `task deploy`
3. Iterate until working
4. Push workflow once

**Composite action approach**: For repeated logic across workflows, convert task file into composite action.

### Technique 2: Run Workflows Locally with Act

**Tool**: [Act](https://github.com/nektos/act) - Run GitHub Actions workflows locally in Docker containers.

**Use case**: Test entire workflow before pushing.

#### Installation

**Using Homebrew** (macOS/Linux):
```bash
brew install act
```

**Using Devbox** (reproducible):
```bash
devbox add act
```

**Verify installation**:
```bash
act --version
```

#### Basic Usage

**Simple workflow** (`.github/workflows/test.yml`):
```yaml
name: Test

on: workflow_dispatch

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: echo "Hello, World!"
```

**Run locally**:
```bash
act workflow_dispatch
```

**What happens**:
1. Act reads workflow file
2. Pulls Docker image matching `runs-on`
3. Mounts your repository into container
4. Executes workflow steps
5. Shows output in terminal

#### Advanced Act Configuration

**Full command with options**:
```bash
act workflow_dispatch \
  --container-architecture linux/amd64 \
  -P ubuntu-latest=node:18 \
  --directory /path/to/repo \
  --workflows .github/workflows/specific-workflow.yml
```

**Options explained**:

**`workflow_dispatch`** - Trigger event name (also: `push`, `pull_request`, etc.)

**`--container-architecture linux/amd64`** - Force AMD64 on ARM machines (Apple Silicon)

**`-P ubuntu-latest=node:18`** - Platform mapping
- Maps workflow runner (`ubuntu-latest`) to Docker image (`node:18`)
- Use smaller images for faster startup

**`--directory /path/to/repo`** - Working directory (repository root)

**`--workflows path/to/workflow.yml`** - Specific workflow file (otherwise runs all matching trigger)

#### Choosing Runner Images

**Act provides three image sizes**:

| Size | Image | Use Case |
|------|-------|----------|
| **Micro** | `node:18` | Simple workflows, no runner dependencies |
| **Medium** | `catthehacker/ubuntu:act-latest` | Most workflows |
| **Large** | `catthehacker/ubuntu:full-latest` | Workflows requiring specific runner tools |

**Trade-offs**:
- **Micro** - Fast (small), but missing most runner tools
- **Medium** - Balanced, covers 80% of workflows
- **Large** - Slow (multi-GB), but complete runner environment

**Recommendation**: Start with medium, upgrade to large only if encountering missing dependencies.

#### Act with Taskfile

**taskfile.yml**:
```yaml
version: '3'

tasks:
  test-workflow:
    desc: Run workflow locally with act
    cmds:
      - |
        act workflow_dispatch \
          --container-architecture linux/amd64 \
          -P ubuntu-latest=catthehacker/ubuntu:act-latest \
          --directory {{.ROOT_DIR}} \
          --workflows .github/workflows/test.yml
```

**Usage**:
```bash
task test-workflow
```

#### Act Limitations

**Works well for**:
- Checking workflow syntax
- Testing step execution order
- Validating environment variables
- Running basic CI tasks (build, test, lint)

**Doesn't work well for**:
- Workflows modifying git state (commits, tags, releases)
- Workflows requiring GitHub API access (issues, PRs)
- Workflows with `GITHUB_TOKEN` authentication
- Workflows using GitHub-specific contexts (`github.event`, etc.)

**Workaround**: Use `act --secret-file .secrets` to provide credentials.

#### Workflow with Act

**Development process**:
1. Write workflow in `.github/workflows/`
2. Test locally: `act workflow_dispatch`
3. Fix issues, iterate
4. When working: Push to GitHub
5. Validate in real environment

**Time saved**: 5-10 iterations locally = 30-60 minutes of CI time saved.

## Category 3: Debugging Failing Workflows

When workflows fail in GitHub, use these techniques to diagnose.

### Technique 1: SSH Access to Runner

**Tool**: `namespacelabs/breakpoint-action`

**Use case**: Get shell access to runner environment at failure point.

**Workflow** (`.github/workflows/debug.yml`):
```yaml
name: Debug with SSH

on: workflow_dispatch

jobs:
  debug:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Run failing step
        run: |
          echo "Attempting something..."
          exit 1  # Simulate failure
      
      - name: Breakpoint on failure
        if: failure()
        uses: namespacelabs/breakpoint-action@v1
        with:
          duration: 10m
          authorized-users: your-github-username
```

**Key configuration**:
- **`if: failure()`** - Only runs when previous step fails
- **`duration: 10m`** - How long to keep SSH session open
- **`authorized-users`** - GitHub usernames allowed to connect

**Usage**:
1. Trigger workflow
2. Wait for failure
3. Breakpoint step runs, shows SSH command
4. Copy command: `ssh <connection-string>`
5. Explore runner environment

**Example SSH session**:
```bash
$ ssh breakpoint@ns-12345.ssh.namespace.so

Welcome to breakpoint shell!
Commands:
  breakpoint-extend - Add 10 more minutes
  breakpoint-resume - Exit and continue workflow

runner@ns-12345:~/work/repo$ ls
README.md  src/  .github/

runner@ns-12345:~/work/repo$ task deploy
task: command not found

# Aha! Task not installed in runner. Need to add installation step.

runner@ns-12345:~/work/repo$ breakpoint-resume
```

**Benefits**:
- **Inspect file system** - See what files exist, their contents
- **Check environment** - Verify environment variables, PATH
- **Test commands** - Try commands to see what works
- **Diagnose tool issues** - Check if dependencies installed correctly

**Works with**:
- GitHub-hosted runners
- Self-hosted runners
- Third-party runners (Namespace, etc.)

### Technique 2: Enable Debug Logging

GitHub Actions supports two debug modes.

#### Step Debug Logging

**Purpose**: Show `core.debug()` messages from actions.

**Enable**: Set repository variable/secret

**Location**: Settings → Secrets and variables → Actions → Variables

**Variable name**: `ACTIONS_STEP_DEBUG`

**Value**: `true`

**Effect**: Debug messages appear in workflow logs.

**Example action code**:
```javascript
const core = require('@actions/core');

core.debug('This only shows when step debug enabled');
core.info('This always shows');
```

**Without `ACTIONS_STEP_DEBUG`**:
```
This always shows
```

**With `ACTIONS_STEP_DEBUG=true`**:
```
This only shows when step debug enabled
This always shows
```

**Use case**: Action behaving unexpectedly, author provided debug logs.

#### Runner Debug Logging

**Purpose**: Show internal runner diagnostics.

**Enable**: Set repository variable

**Variable name**: `ACTIONS_RUNNER_DEBUG`

**Value**: `true`

**Effect**: Massive amount of runner internals logged.

**Access**: Download log archive (includes runner diagnostic logs)

**Use case**: Extremely rare. Only for debugging runner-level issues (not action issues).

**How to access**:
1. Enable `ACTIONS_RUNNER_DEBUG`
2. Run workflow
3. Go to workflow run page
4. Click "⋯" → "Download log archive"
5. Extract ZIP
6. Open `runner-diagnostic-logs/` folder

**Example logs**:
```
[2023-10-15 10:30:45] Runner.Listener: Starting runner listener
[2023-10-15 10:30:45] Terminal.ParseCommand: Parsing command '/bin/bash'
[2023-10-15 10:30:46] ProcessInvoker: Starting process '/bin/bash'
```

### Technique 3: Re-run with Debug Logging

**Convenience feature**: Enable debug logging for single run without changing repository settings.

**How**:
1. Go to failed workflow run
2. Click "Re-run jobs" dropdown
3. Select "Re-run jobs with debug logging"
4. Run executes with both step and runner debugging enabled

**Benefits**:
- No need to modify repository settings
- Debug logs only for this run (doesn't clutter subsequent runs)

## Category 4: Performance Monitoring

**Goal**: Understand where time is spent, identify optimization opportunities.

### Problem with Default GitHub UI

**Current experience**:
- List of workflow runs
- Durations shown per run
- No trends over time
- No step-level breakdown visible at-a-glance

**Limitation**: Hard to spot performance regressions or optimization opportunities.

### Solution 1: Third-Party Runner Insights

**Providers with analytics**:
- **Namespace** - Built-in insights dashboard
- **BuildJet** - Performance metrics
- **Blacksmith** - Timing and resource data

**Example: Namespace Insights**

**Access**: Dashboard → Insights

**Features**:
- **Timing trends** - Workflow duration over time (graph)
- **Step-level breakdown** - See which steps are slowest
- **Resource usage** - CPU and memory utilization per job
- **Filtering** - By workflow, job, date range

**Use cases**:
- **Identify slow steps** - Which step takes 80% of runtime?
- **Detect regressions** - Did recent change slow workflow?
- **Resource constraints** - Is job CPU or memory-bound?
- **Optimization validation** - Did caching improvement work?

**Example insights**:
```
Workflow: Build and Test
Last 30 days

Average duration: 4m 32s
Slowest step: Install dependencies (2m 15s)
CPU usage: 40% average
Memory usage: 2.1 GB peak
```

**Action**: Add dependency caching to speed up "Install dependencies" step.

### Solution 2: Export to Observability Platform (Honeycomb)

**Advanced approach**: Treat workflow runs as traces, analyze with APM tools.

**Tool**: [kvrhdn/gha-buildevents](https://github.com/kvrhdn/gha-buildevents)

**Purpose**: Converts GitHub Actions timing data into OpenTelemetry traces, sends to Honeycomb.

#### Setup

**Create Honeycomb account** (free tier available)

**Get API key**: Settings → API Keys

**Add to repository secrets**: `HONEYCOMB_API_KEY`

**Create export workflow** (`.github/workflows/export-timing.yml`):
```yaml
name: Export Timing Data

on:
  workflow_run:
    workflows:
      - Build and Test
      - Deploy
    types: [completed]

jobs:
  export:
    runs-on: ubuntu-latest
    steps:
      - uses: kvrhdn/gha-buildevents@v1
        with:
          apikey: ${{ secrets.HONEYCOMB_API_KEY }}
          dataset: github-actions
          job-status: ${{ job.status }}
```

**What happens**:
1. Target workflow completes
2. Export workflow triggers
3. Fetches timing data from GitHub API
4. Converts to trace format (spans = steps)
5. Sends to Honeycomb

#### Analyzing in Honeycomb

**View traces**:
- Navigate to dataset "github-actions"
- See workflow runs as distributed traces
- Each step = span with duration

**Example trace**:
```
Workflow: Build and Test (1m 23s total)
├─ Queue (7s)
├─ Setup job (5s)
├─ Checkout code (3s)
├─ Setup Node.js (8s)
├─ Install dependencies (45s) ← SLOWEST
├─ Build (12s)
├─ Run tests (18s)
└─ Upload artifacts (5s)
```

**Visualizations**:
- **Heatmap** - Spot outliers (which runs were slowest?)
- **Time series** - Trend over days/weeks
- **Percentiles** - p50, p95, p99 durations
- **Breakdown** - Which step contributes most to total time?

**Queries** (Honeycomb query builder):
- "Show me all runs where Install dependencies > 1 minute"
- "Compare Build step duration before/after caching"
- "What's the p95 duration for Deploy workflow?"

**Benefits**:
- **Step-level visibility** - See exactly where time goes
- **Historical trends** - Track performance over weeks/months
- **Anomaly detection** - Spot unusual spikes
- **Team collaboration** - Share queries and dashboards

**Cost**: Fits within Honeycomb free tier for most projects.

## Developer Experience Best Practices

### 1. Invest in Tooling Setup

**One-time effort, continuous benefit**.

**Recommended setup**:
- VS Code with GitHub Actions + YAML + Rainbow Indent extensions
- Act installed and configured
- Taskfile for common operations
- Breakpoint action in common workflow template

**Time investment**: 30 minutes
**Time saved**: Hours per month

### 2. Extract Logic Early

**When writing workflow**:
- Inline bash for simple 1-liners (echo, env var)
- Task file for anything complex (loops, conditionals, multi-command)
- Composite action when logic used >3 workflows

**Rule of thumb**: If you're not 100% sure it'll work, pull it out into a task file.

### 3. Test Locally First

**Development workflow**:
1. Write/modify workflow
2. Run with act locally
3. Iterate until working
4. Push to GitHub
5. Validate in real environment

**Avoid**: Push → wait → fail → push → wait → fail (repeat 10x)

### 4. Use Debug Logging Strategically

**When to enable**:
- Action behaves unexpectedly
- Need to see internal state
- Debugging third-party actions

**When NOT to enable**:
- Debugging own bash scripts (add `set -x` instead)
- Performance issues (use timing data, not logs)
- Always-on (creates log noise)

### 5. Monitor Performance from Day One

**Don't wait for slowness complaints**.

**Set up**:
- Third-party runner insights (if using Namespace, etc.)
- OR Honeycomb export workflow
- Review trends monthly

**Identify optimization opportunities early** before they become bottlenecks.

### 6. Create Reusable Debug Workflows

**Template workflow** (`.github/workflows/_debug-template.yml`):
```yaml
name: Debug Template

on: workflow_dispatch

jobs:
  debug:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Your debugging steps here
        run: echo "Test something"
      
      - name: Breakpoint on failure
        if: failure()
        uses: namespacelabs/breakpoint-action@v1
        with:
          duration: 15m
          authorized-users: ${{ github.actor }}
```

**Use**: Copy/paste when need to debug new workflow.

## Summary: Developer Experience Techniques

| Category | Technique | Tool | Benefit |
|----------|-----------|------|---------|
| **Action Development** | Local execution | `@github/local-action` | Test actions without pushing |
| **Action Development** | Unit tests | Jest, pytest, etc. | Prevent regressions |
| **Workflow Development** | Extract to task files | Taskfile | Iterate locally |
| **Workflow Development** | Run locally | Act | Test full workflow before push |
| **Workflow Development** | Extensions | GitHub Actions, YAML, Rainbow Indent | Better editing experience |
| **Debugging** | SSH access | Breakpoint action | Inspect runner environment |
| **Debugging** | Step debug logs | `ACTIONS_STEP_DEBUG` | See action internals |
| **Debugging** | Runner debug logs | `ACTIONS_RUNNER_DEBUG` | Diagnose runner issues (rare) |
| **Performance** | Runner insights | Namespace, BuildJet | Job-level metrics |
| **Performance** | Trace export | Honeycomb | Step-level trends over time |

## Key Takeaways

**The problem**: Default workflow development is slow (push/wait/fail cycle).

**The solution**: Test locally, debug intelligently.

**Best practices**:
1. **Set up tooling once** - Extensions, act, taskfile
2. **Extract logic early** - Task files, not inline bash
3. **Test before pushing** - Act for workflows, local-action for actions
4. **Debug with SSH** - Breakpoint action for mysterious failures
5. **Monitor performance** - Runner insights or Honeycomb exports
6. **Iterate quickly** - Tight feedback loops = faster development

**Time saved**: Converting 10-iteration push/fail cycles into 1-2 iterations saves 30-60 minutes per workflow.

**Improved experience**: Less frustration, more confidence, faster delivery.

## Next Chapter

You've mastered the tools and techniques for efficient workflow development. In Chapter 11, we'll explore **Best Practices**—the principles and patterns that separate amateur workflows from production-grade automation. You'll learn about security, maintainability, performance optimization, and organizational standards that ensure your workflows are robust, secure, and sustainable at scale. These practices prevent technical debt and set your team up for long-term success with GitHub Actions.
