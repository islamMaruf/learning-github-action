# Chapter 5: Core Features - Building Blocks of GitHub Actions

## Overview

Now that your development environment is ready, it's time to dive into actually writing and running GitHub Actions workflows. This chapter covers the **core features**—the fundamental building blocks you'll use in every workflow. We'll explore concepts through both theory and hands-on examples, running real workflows and examining their outputs.

By the end of this chapter, you'll understand:
- The anatomy of a workflow (events, workflows, jobs, steps, runners)
- How to trigger workflows with different events
- How to pass data between steps and jobs
- How to manage secrets and environment variables
- How to control job execution order and parallelism

Let's begin by establishing a shared vocabulary.

## GitHub Actions Terminology: The Mental Model

Before looking at YAML configurations, we need a clear mental model of how GitHub Actions components fit together.

### The Five Core Concepts

```
[GitHub Event] → triggers → [Workflow] → contains → [Jobs] → contains → [Steps] → runs on → [Runner]
```

Let's explore each concept in detail:

#### 1. Events: The Triggers

**Definition**: Events are **things that happen in GitHub** that can trigger a workflow to run.

**Examples of events**:
- **push**: Code is pushed to a branch
- **pull_request**: A pull request is opened, updated, or merged
- **issues**: An issue is created, edited, or closed
- **schedule**: A scheduled time arrives (like a cron job)
- **workflow_dispatch**: Manual trigger from the GitHub UI
- **release**: A new release is published

**Think of events as**: The "when" of your automation. "When code is pushed" or "When a PR is opened" or "When it's midnight UTC."

**Key insight**: You can respond to virtually any GitHub API event. If you can see it happen in GitHub's interface, you can likely trigger a workflow from it.

#### 2. Workflows: The Top-Level Automation

**Definition**: A workflow is the **entire automation configuration**—one YAML file defining what happens when certain events occur.

**Location**: Workflows live in `.github/workflows/` directory of your repository.

**File naming**: Name workflow files descriptively: `test.yml`, `deploy-production.yml`, `nightly-cleanup.yml`.

**Think of workflows as**: Complete automation scripts. One workflow might handle "test on every PR" while another handles "deploy when merging to main."

**Key insight**: Workflows are **the unit of automation** in GitHub Actions. Each workflow file is independent and can be triggered by different events.

#### 3. Jobs: Units of Work

**Definition**: Jobs are **distinct units of work** within a workflow. Each job consists of multiple steps and runs on its own runner.

**Critical characteristic**: Jobs run in **parallel by default** unless you specify dependencies between them.

**Think of jobs as**: Different machines/environments performing work. One job might run tests on Linux, another on Windows, another builds a Docker image.

**Key insight**: Data does not automatically flow between jobs—they run in isolated environments. You must explicitly pass data between jobs using outputs.

#### 4. Steps: Individual Tasks

**Definition**: Steps are **individual tasks** within a job. They run sequentially in the order specified.

**Critical characteristic**: Steps run in the **same environment**—they share the file system, environment variables, and state.

**Types of steps**:
1. **Run commands**: Execute shell scripts directly
2. **Use actions**: Call pre-built or custom actions
3. **Conditional steps**: Run only when certain conditions are met

**Think of steps as**: Lines in a script. "First checkout code, then install dependencies, then run tests, then upload results."

**Key insight**: Steps within a job share the same runner and can easily pass data to each other.

#### 5. Runners: Execution Environments

**Definition**: Runners are the **virtual machines or servers** that actually execute your jobs.

**Types of runners**:
- **GitHub-hosted**: GitHub provides and maintains them (Ubuntu, Windows, macOS)
- **Self-hosted**: You provide and maintain your own runners
- **Third-party**: Services like Namespace provide optimized runners

**Specifications**: Each runner has:
- Operating system (Ubuntu, Windows, macOS)
- CPU and memory allocation
- Pre-installed software (Node.js, Python, Docker, etc.)

**Think of runners as**: The computers where your code runs. Like choosing "run on my laptop" vs. "run on that server" vs. "run on a fresh cloud VM."

**Key insight**: The `runs-on` field determines which runner type executes your job. Different jobs can use different runner types.

### Visual Mental Model

```
Repository:
└── .github/workflows/
    ├── test.yml              ← Workflow #1
    └── deploy.yml            ← Workflow #2

test.yml Structure:
├── on: [push, pull_request]  ← Events trigger this workflow
├── jobs:
│   ├── job-1:                ← Job running on Ubuntu
│   │   ├── runs-on: ubuntu   ← Runner specification
│   │   └── steps:            ← Steps execute sequentially
│   │       ├── step-1
│   │       ├── step-2
│   │       └── step-3
│   └── job-2:                ← Job running on Windows
│       ├── runs-on: windows  ← Different runner
│       └── steps:
│           ├── step-1
│           └── step-2
```

**Key relationships**:
- **Events ↔ Workflows**: One-to-many (one push event can trigger multiple workflows)
- **Workflows ↔ Jobs**: One-to-many (one workflow contains multiple jobs)
- **Jobs ↔ Steps**: One-to-many (one job contains multiple steps)
- **Jobs ↔ Runners**: One-to-one (each job runs on exactly one runner)

## Your First Workflow: Hello World

Let's examine the simplest possible workflow to see these concepts in action.

### The Code

Create `.github/workflows/hello-world.yml`:

```yaml
name: Hello World

on: workflow_dispatch

jobs:
  say-hello:
    runs-on: ubuntu-24.04
    steps:
      - run: echo "Hello from an inline bash script in a GitHub Actions workflow!"
```

### Line-by-Line Breakdown

**Line 1: `name: Hello World`**
- The human-readable name displayed in GitHub UI
- Optional but recommended for clarity
- Shows up in the Actions tab and PR checks

**Line 3: `on: workflow_dispatch`**
- Defines the **event** that triggers this workflow
- `workflow_dispatch` means "manual trigger only"
- You'll click a button in GitHub UI to run it
- Great for learning/testing—you control when it runs

**Line 5: `jobs:`**
- Begins the jobs section
- Contains one or more job definitions

**Line 6: `say-hello:`**
- The **job ID** (used to reference this job)
- Can be any valid YAML key name
- Use descriptive names: `test`, `build`, `deploy-production`

**Line 7: `runs-on: ubuntu-24.04`**
- Specifies the **runner type**
- `ubuntu-24.04`: GitHub-hosted Ubuntu 24.04 LTS
- Alternatives: `ubuntu-22.04`, `windows-latest`, `macos-latest`
- Third-party runners specified differently (covered in advanced features)

**Line 8: `steps:`**
- Begins the steps section
- Contains one or more step definitions

**Line 9: `- run: echo "..."`**
- A single step that runs a shell command
- `run:` indicates this step executes shell commands
- Default shell on Ubuntu runners is bash
- The echo command prints to stdout (visible in logs)

### Running the Workflow

**Step 1: Navigate to Actions tab**
- Go to your forked repository on GitHub
- Click the "Actions" tab at the top

**Step 2: Find your workflow**
- Left sidebar lists all workflows
- Click "Hello World" (or whatever you named it)

**Step 3: Trigger manually**
- Click "Run workflow" button (top right)
- Select branch (usually `main`)
- Click green "Run workflow" button

**What happens**:
1. Workflow enters queue (status: ⏳ Queued)
2. GitHub allocates a runner
3. Runner starts workflow execution
4. Status changes to 🏃 Running
5. Steps execute
6. Status becomes ✅ Success or ❌ Failure

**Step 4: View results**
- Click on the workflow run to see details
- Click on the job name ("say-hello")
- Expand the step to see output
- You'll see: `Hello from an inline bash script in a GitHub Actions workflow!`

### What Just Happened?

Let's trace the execution:

1. **You clicked "Run workflow"** → Generated a `workflow_dispatch` event
2. **Event matched** `on: workflow_dispatch` → Triggered the workflow
3. **GitHub allocated** an Ubuntu 24.04 runner from its pool
4. **Runner executed** the single job `say-hello`
5. **Job executed** the single step with the echo command
6. **Output logged** to GitHub's UI for your review
7. **Runner destroyed** after job completion

**Cost**: ~5-10 seconds of runner time (free for public repos, counts against minutes for private repos)

## Three Types of Steps

Steps can take different forms depending on what you need to do. Let's explore the three main types.

### Expanded Hello World with Three Step Types

```yaml
name: Step Types

on: workflow_dispatch

jobs:
  bash-step:
    runs-on: ubuntu-24.04
    steps:
      - run: echo "Hello from an inline bash script!"

  python-step:
    runs-on: ubuntu-24.04
    steps:
      - shell: python
        run: |
          print("Hello from an inline Python script!")
          import sys
          print(f"Running Python {sys.version}")

  action-step:
    runs-on: ubuntu-24.04
    steps:
      - uses: actions/hello-world-javascript-action@main
        with:
          who-to-greet: 'GitHub Actions Student'
```

### Type 1: Bash/Shell Commands (Most Common)

**Syntax**:
```yaml
- run: echo "command here"
```

Or multi-line:
```yaml
- run: |
    echo "First command"
    echo "Second command"
    npm install
    npm test
```

**When to use**:
- Running command-line tools (npm, pip, docker, kubectl, etc.)
- Simple scripts
- Chaining multiple commands

**Default shell**: 
- Linux/macOS: `bash`
- Windows: `pwsh` (PowerShell)

**Example use cases**:
- `npm install && npm test` (install and test Node.js app)
- `docker build -t myapp:latest .` (build container image)
- `kubectl apply -f deployment.yml` (deploy to Kubernetes)

### Type 2: Alternative Shells

**Syntax**:
```yaml
- shell: python
  run: |
    print("Python code here")
```

**Available shells**:
- `bash`: Default on Linux/macOS
- `sh`: Basic POSIX shell
- `pwsh`: PowerShell (cross-platform)
- `python`: Python interpreter
- `node`: Node.js runtime

**When to use**:
- Need specific language features (Python for complex string manipulation, Node for JSON parsing)
- Platform-specific requirements (PowerShell on Windows)
- Existing scripts in a specific language

**Example: Python for complex processing**:
```yaml
- shell: python
  run: |
    import json
    import os
    
    # Read GitHub event data
    with open(os.environ['GITHUB_EVENT_PATH']) as f:
        event = json.load(f)
    
    # Extract PR number
    pr_number = event['pull_request']['number']
    print(f"Processing PR #{pr_number}")
```

This would be much more verbose in bash.

### Type 3: Using Actions

**Syntax**:
```yaml
- uses: owner/repo@version
  with:
    input-parameter: value
```

**What are actions**: Pre-built, reusable units of code that perform specific tasks. Think of them as functions you can call.

**Action sources**:
- **GitHub's official actions**: `actions/checkout`, `actions/setup-node`, etc.
- **Third-party marketplace**: Thousands from community and companies
- **Your own custom actions**: Build and use locally or publish

**Version pinning**: Three formats:
1. **Branch**: `@main`, `@dev` (gets latest on that branch—not recommended for stability)
2. **Tag**: `@v3`, `@v2.1.0` (semantic versioning)
3. **Commit SHA**: `@a1b2c3d4...` (most secure, immutable)

**Best practice**: Use full commit SHA for security-critical workflows. Tags can be moved (a malicious actor could retag v3 to point to malicious code), but commit SHAs are immutable.

**Example - Checkout code**:
```yaml
- uses: actions/checkout@v3  # Clones your repository
```

**Example - Setup Node.js**:
```yaml
- uses: actions/setup-node@v3
  with:
    node-version: '18'
```

**Example - Deploy to AWS**:
```yaml
- uses: aws-actions/configure-aws-credentials@v2
  with:
    aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
    aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
```

### Running the Multi-Step Workflow

When you run the "Step Types" workflow:

**Observation 1**: All three jobs run **in parallel**
- They don't have dependencies between them
- GitHub executes them simultaneously for speed
- Each gets its own isolated runner

**Observation 2**: Each job completes independently
- bash-step completes first (fastest—just an echo)
- python-step second (loads Python interpreter)
- action-step third (needs to clone action repository)

**Observation 3**: Output varies by type
- bash-step: Direct stdout
- python-step: Python's print() output
- action-step: Action's predefined output format

## Controlling Job Order with Dependencies

By default, jobs run in parallel. But often you need specific ordering—for example, "build before deploy" or "test before merge."

### The `needs` Keyword

**Purpose**: Create dependencies between jobs, forming a Directed Acyclic Graph (DAG).

**Syntax**:
```yaml
jobs:
  job-a:
    # ...
  
  job-b:
    needs: job-a  # Wait for job-a to complete
    # ...
  
  job-c:
    needs: [job-a, job-b]  # Wait for both
    # ...
```

### Example: Sequential Execution

```yaml
name: Jobs and Steps

on: workflow_dispatch

jobs:
  job-1:
    runs-on: ubuntu-latest
    steps:
      - run: echo "Step 1 of Job 1"
      - run: echo "Step 2 of Job 1"
      - run: echo "Step 3 of Job 1"

  job-2:
    runs-on: ubuntu-latest
    steps:
      - run: echo "Single step of Job 2"

  job-3:
    needs: [job-1, job-2]
    runs-on: ubuntu-latest
    steps:
      - run: echo "Job 3 waits for jobs 1 and 2"

  job-4:
    needs: job-3
    runs-on: ubuntu-latest
    steps:
      - run: echo "Job 4 is last"
```

### Execution Flow Visualization

```
Time →

job-1 ████████
job-2 ████████
              job-3 ████████
                            job-4 ████████
```

**What happens**:
1. **job-1** and **job-2** start immediately (no dependencies)
2. They run **in parallel**
3. **job-3** waits until **both** job-1 and job-2 complete
4. **job-4** waits until job-3 completes

**In GitHub UI**: You'll see a graph showing these dependencies with arrows connecting jobs.

### Important: Acyclic Constraint

**You cannot create loops**:
```yaml
# ❌ THIS WILL FAIL
jobs:
  job-a:
    needs: job-b
  job-b:
    needs: job-a  # Circular dependency!
```

**Why**: Workflows must have a clear start and end. Cycles would create infinite loops.

**Valid patterns**:
- Linear: A → B → C → D
- Fan-out: A → [B, C, D]
- Fan-in: [A, B, C] → D
- Diamond: A → [B, C] → D
- Complex DAG: Any combination without cycles

## Workflow Events: Triggers That Matter

`workflow_dispatch` is useful for learning, but real workflows trigger automatically based on repository events.

### The Four Most Common Events

Let's examine a workflow that uses all four:

```yaml
name: Common Events

on:
  workflow_dispatch:  # Manual trigger
  
  push:
    branches:
      - main
      - 'release/**'  # Glob patterns supported
  
  pull_request:
    types: [opened, synchronize, reopened]
    paths:
      - 'src/**'  # Only if src/ files change
      - '!src/**/*.md'  # Exclude markdown files
  
  schedule:
    - cron: '0 0 * * *'  # Daily at midnight UTC

jobs:
  show-event:
    runs-on: ubuntu-latest
    steps:
      - run: cat $GITHUB_EVENT_PATH  # Dump event JSON
```

### Event 1: workflow_dispatch (Manual Trigger)

**When it triggers**: When you click "Run workflow" in GitHub UI or use GitHub CLI.

**Use cases**:
- Testing workflows during development
- Manually deploying to production
- Running maintenance tasks on-demand
- Debugging workflow issues

**How to trigger**:
- UI: Actions tab → Select workflow → "Run workflow"
- CLI: `gh workflow run workflow-name.yml`

**Optional: Inputs**:
```yaml
on:
  workflow_dispatch:
    inputs:
      environment:
        description: 'Environment to deploy to'
        required: true
        type: choice
        options:
          - staging
          - production
```

Then access in workflow: `${{ inputs.environment }}`

### Event 2: push (Code Pushed to Branch)

**When it triggers**: Whenever code is pushed to specified branches.

**Basic syntax**:
```yaml
on:
  push:  # Triggers on push to ANY branch
```

**Filter by branch**:
```yaml
on:
  push:
    branches:
      - main  # Only main
      - develop  # Only develop
      - 'release/*'  # Any branch starting with release/
      - '!staging/*'  # Exclude staging branches
```

**Common patterns**:
- `main` - Production deployments
- `develop` - Development environment deployments
- `release/**` - Release candidate testing
- `feature/**` - Feature branch CI

**Use cases**:
- Run tests on every push
- Deploy to development on push to develop
- Deploy to production on push to main

**Example - Real-world CI**:
```yaml
on:
  push:
    branches: [main, develop]

jobs:
  test:
    # Run tests
  
  deploy-dev:
    needs: test
    if: github.ref == 'refs/heads/develop'
    # Deploy to development
  
  deploy-prod:
    needs: test
    if: github.ref == 'refs/heads/main'
    # Deploy to production
```

### Event 3: pull_request (PR Opened or Updated)

**When it triggers**: When pull requests are opened, updated, or closed.

**Default types** (if not specified):
- `opened` - PR is created
- `synchronize` - PR is updated with new commits
- `reopened` - Closed PR is reopened

**All available types**:
`opened`, `edited`, `closed`, `reopened`, `synchronize`, `converted_to_draft`, `ready_for_review`, `labeled`, `unlabeled`, `auto_merge_enabled`, `auto_merge_disabled`

**Basic syntax**:
```yaml
on:
  pull_request:  # opened, synchronize, reopened
```

**Filter by type**:
```yaml
on:
  pull_request:
    types: [opened, synchronize]  # Skip reopened
```

**Filter by paths**:
```yaml
on:
  pull_request:
    paths:
      - 'src/**'  # Only if src/ changes
      - 'package.json'  # Or package.json
      - '!**/*.md'  # Exclude markdown files
```

**Why path filtering matters**: In monorepos with multiple services, you don't want to run Service A's tests when Service B's code changes.

**Example - Monorepo**:
```
repo/
├── service-a/  # Node.js API
├── service-b/  # Python worker
└── shared/     # Shared libraries
```

```yaml
# .github/workflows/service-a-tests.yml
on:
  pull_request:
    paths:
      - 'service-a/**'
      - 'shared/**'  # Shared code affects service-a

# .github/workflows/service-b-tests.yml
on:
  pull_request:
    paths:
      - 'service-b/**'
      - 'shared/**'  # Shared code affects service-b
```

**Use cases**:
- Run tests on every PR
- Check code quality/linting
- Verify PR description format
- Comment on PR with test results

### Event 4: schedule (Cron-Based Timing)

**When it triggers**: At specified times on a schedule.

**Syntax**:
```yaml
on:
  schedule:
    - cron: '0 0 * * *'  # Daily at midnight UTC
    - cron: '0 */6 * * *'  # Every 6 hours
```

**Cron format**: `minute hour day month weekday`
- `0 0 * * *` - Every day at midnight
- `0 12 * * *` - Every day at noon
- `0 9 * * 1` - Every Monday at 9 AM
- `*/15 * * * *` - Every 15 minutes
- `0 0 1 * *` - First day of every month

**Important**: Times are always **UTC**.

**Use cases**:
- Nightly builds
- Daily dependency updates check
- Weekly cleanup of old resources
- Monthly reports generation

**Example - Nightly cleanup**:
```yaml
on:
  schedule:
    - cron: '0 2 * * *'  # 2 AM UTC daily

jobs:
  cleanup:
    runs-on: ubuntu-latest
    steps:
      - name: Delete old artifacts
        run: # ...
      - name: Cleanup old branches
        run: # ...
```

### Practical Example: Path Filtering

Let's see path filtering in action:

**Setup**:
```
repo/
└── module-3/
    └── filters/
        ├── included.md
        └── excluded.txt
```

**Workflow**:
```yaml
on:
  pull_request:
    paths:
      - '**.md'  # Include all markdown
      - '!**/*.txt'  # Exclude all txt
```

**Test 1**: Create PR modifying `included.md`
- **Result**: Workflow triggers ✅
- **Why**: File matches `**.md`

**Test 2**: Create PR modifying `excluded.txt`
- **Result**: Workflow does NOT trigger ❌
- **Why**: File matches `!**/*.txt` (excluded)

**Test 3**: Create PR modifying both
- **Result**: Workflow triggers ✅
- **Why**: At least one file (`included.md`) matches

## Environment Variables: Scoped Data

Environment variables allow you to store configuration and data that steps can access. GitHub Actions supports three scopes:

### The Three Scopes

```yaml
name: Environment Variables

on: workflow_dispatch

env:
  WORKFLOW_VAR: "Available in all jobs"  # Workflow scope

jobs:
  job-1:
    runs-on: ubuntu-latest
    env:
      JOB_VAR: "Available in all steps of job-1"  # Job scope
    
    steps:
      - name: Step 1
        env:
          STEP_VAR: "Available only in this step"  # Step scope
        run: |
          echo "WORKFLOW_VAR: $WORKFLOW_VAR"
          echo "JOB_VAR: $JOB_VAR"
          echo "STEP_VAR: $STEP_VAR"
      
      - name: Step 2
        run: |
          echo "WORKFLOW_VAR: $WORKFLOW_VAR"
          echo "JOB_VAR: $JOB_VAR"
          echo "STEP_VAR: $STEP_VAR"  # Will be empty!

  job-2:
    runs-on: ubuntu-latest
    steps:
      - run: |
          echo "WORKFLOW_VAR: $WORKFLOW_VAR"
          echo "JOB_VAR: $JOB_VAR"  # Will be empty!
          echo "STEP_VAR: $STEP_VAR"  # Will be empty!
```

### Scope Visibility Table

| Variable | Step 1 | Step 2 | Job 2 |
|----------|--------|--------|-------|
| WORKFLOW_VAR | ✅ | ✅ | ✅ |
| JOB_VAR | ✅ | ✅ | ❌ |
| STEP_VAR | ✅ | ❌ | ❌ |

### When to Use Each Scope

**Workflow scope**: Values needed across all jobs
- Build version numbers
- Repository metadata
- Global configuration

**Job scope**: Values specific to one job's execution
- Job-specific configuration
- Database connection strings for test job
- Build artifact names

**Step scope**: Values for a single operation
- Temporary values
- Step-specific flags
- Credentials used once

### Best Practice: Principle of Least Privilege

Use the narrowest scope possible:
- Only one step needs it? Use step scope.
- Multiple steps in one job? Use job scope.
- Multiple jobs? Use workflow scope.

This makes workflows easier to understand and debug.

## Passing Data: Outputs and Environment Variables

Steps and jobs run in isolated environments. To share data, you must explicitly pass it.

### Within a Job: Outputs Between Steps

**Method 1: Step Outputs** (Recommended)

```yaml
jobs:
  example:
    runs-on: ubuntu-latest
    steps:
      - name: Generate data
        id: generate  # ID required to reference later
        run: |
          echo "foo=bar" >> $GITHUB_OUTPUT
          echo "timestamp=$(date +%s)" >> $GITHUB_OUTPUT
      
      - name: Use data
        run: |
          echo "Foo value: ${{ steps.generate.outputs.foo }}"
          echo "Timestamp: ${{ steps.generate.outputs.timestamp }}"
```

**How it works**:
1. First step writes to `$GITHUB_OUTPUT` file
2. Format: `key=value`
3. Later steps reference: `steps.<step-id>.outputs.<key>`

**Method 2: Runtime Environment Variables**

```yaml
steps:
  - name: Set environment variable
    run: echo "FOO=bar" >> $GITHUB_ENV
  
  - name: Use environment variable
    run: echo "Foo value: $FOO"
```

**How it works**:
1. First step writes to `$GITHUB_ENV` file
2. GitHub injects it as environment variable for subsequent steps
3. Access as normal shell variable: `$FOO`

**When to use each**:
- **Step outputs**: Explicit, clear what's being passed, better for complex data
- **Environment variables**: Simpler syntax, good for simple values

### Between Jobs: Job Outputs

**Jobs run on different runners**, so data doesn't persist. You must use outputs:

```yaml
jobs:
  producer:
    runs-on: ubuntu-latest
    outputs:
      foo: ${{ steps.generate.outputs.foo }}  # Expose step output as job output
    steps:
      - name: Generate data
        id: generate
        run: echo "foo=bar" >> $GITHUB_OUTPUT
  
  consumer:
    needs: producer  # Dependency required!
    runs-on: ubuntu-latest
    steps:
      - name: Use data
        run: echo "From producer: ${{ needs.producer.outputs.foo }}"
```

**Critical steps**:
1. **Step generates output**: Write to `$GITHUB_OUTPUT`
2. **Job exposes output**: Reference step output in job's `outputs:`
3. **Consumer declares dependency**: Use `needs:`
4. **Consumer accesses output**: Reference `needs.<job-id>.outputs.<key>`

**Why the dependency**: Without `needs`, consumer might start before producer finishes, and outputs wouldn't be available.

### What About Environment Variables Across Jobs?

**They don't work**:
```yaml
jobs:
  producer:
    steps:
      - run: echo "FOO=bar" >> $GITHUB_ENV
  
  consumer:
    steps:
      - run: echo "$FOO"  # Empty! Different runner!
```

**Why**: Each job runs on a fresh runner with a fresh environment.

**Solution**: Use job outputs instead.

## Secrets and Variables: Managing Configuration

Hard-coding credentials in workflows is a security disaster waiting to happen. GitHub provides **Secrets** and **Variables** for secure, manageable configuration.

### Secrets vs. Variables

| Aspect | Secrets | Variables |
|--------|---------|-----------|
| **Sensitivity** | Sensitive (passwords, API keys) | Non-sensitive (config values) |
| **Visibility** | Hidden in UI, masked in logs | Visible in UI and logs |
| **Where stored** | Encrypted in GitHub | Plain text in GitHub |
| **Use case** | AWS keys, database passwords | Environment names, URLs |

### Setting Secrets

**Repository level**:
1. Go to repository → Settings
2. Secrets and variables → Actions
3. Click "New repository secret"
4. Name: `AWS_ACCESS_KEY_ID`
5. Value: `AKIAIOSFODNN7EXAMPLE`
6. Click "Add secret"

**Organization level** (for sharing across repos):
1. Go to organization → Settings
2. Secrets and variables → Actions
3. Click "New organization secret"
4. Choose which repositories can access it

**Environment level** (for staging vs. production):
1. Repository → Settings → Environments
2. Create environment (e.g., "production")
3. Add secrets specific to that environment

### Setting Variables

Same process as secrets, but under the "Variables" tab instead of "Secrets" tab.

### Using Secrets and Variables in Workflows

```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: production  # Links to environment secrets
    steps:
      - name: Configure AWS
        run: |
          aws configure set aws_access_key_id ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws configure set aws_secret_access_key ${{ secrets.AWS_SECRET_ACCESS_KEY }}
      
      - name: Show environment
        run: echo "Deploying to: ${{ vars.ENVIRONMENT_NAME }}"
```

**Syntax**:
- Secrets: `${{ secrets.SECRET_NAME }}`
- Variables: `${{ vars.VARIABLE_NAME }}`

### Secret Masking

GitHub automatically **masks secrets in logs**:

```yaml
steps:
  - run: echo "Secret: ${{ secrets.MY_SECRET }}"
```

**Output in logs**:
```
Secret: ***
```

**Why this matters**: Prevents accidental credential leaks. Even if you explicitly try to print secrets, GitHub hides them.

**Caveat**: Masking isn't perfect. Avoid:
```yaml
- run: echo "${{ secrets.MY_SECRET }}" | base64  # Base64 isn't masked!
```

### Environment-Specific Secrets

**Setup**:
- Repository → Settings → Environments
- Create "staging" environment
  - Secret: `DB_PASSWORD` = "staging_password"
- Create "production" environment
  - Secret: `DB_PASSWORD` = "production_password"

**Workflow**:
```yaml
jobs:
  deploy-staging:
    environment: staging
    steps:
      - run: echo "DB Password: ${{ secrets.DB_PASSWORD }}"
        # Uses staging_password

  deploy-production:
    environment: production
    steps:
      - run: echo "DB Password: ${{ secrets.DB_PASSWORD }}"
        # Uses production_password
```

**Same secret name, different values**. The `environment:` field determines which value is used.

## GitHub Actions Contexts: Accessing Runtime Data

Throughout this chapter, we've used syntax like `${{ secrets.MY_SECRET }}` and `${{ steps.step-id.outputs.key }}`. This double-curly-brace syntax accesses **contexts**—data structures available at runtime.

### What Are Contexts?

Contexts are **objects containing information about the workflow run**, repository, event, and more. Think of them as global variables you can access in your workflow.

**Syntax**: `${{ context.property }}`

### Most Useful Contexts

**1. `github` - Repository and run information**
```yaml
- run: |
    echo "Repository: ${{ github.repository }}"
    echo "Ref: ${{ github.ref }}"
    echo "SHA: ${{ github.sha }}"
    echo "Actor: ${{ github.actor }}"
    echo "Event: ${{ github.event_name }}"
```

**2. `env` - Environment variables**
```yaml
env:
  MY_VAR: "value"

steps:
  - run: echo "${{ env.MY_VAR }}"
```

**3. `secrets` - Secret values**
```yaml
- run: echo "${{ secrets.MY_SECRET }}"
```

**4. `vars` - Variable values**
```yaml
- run: echo "${{ vars.MY_VARIABLE }}"
```

**5. `steps` - Step outputs**
```yaml
- id: step1
  run: echo "foo=bar" >> $GITHUB_OUTPUT
- run: echo "${{ steps.step1.outputs.foo }}"
```

**6. `needs` - Outputs from dependent jobs**
```yaml
jobs:
  job1:
    outputs:
      foo: ${{ steps.generate.outputs.foo }}
  
  job2:
    needs: job1
    steps:
      - run: echo "${{ needs.job1.outputs.foo }}"
```

**7. `runner` - Runner information**
```yaml
- run: |
    echo "OS: ${{ runner.os }}"
    echo "Arch: ${{ runner.arch }}"
    echo "Temp: ${{ runner.temp }}"
```

**8. `matrix` - Matrix values** (covered in Advanced Features)
```yaml
strategy:
  matrix:
    node: [14, 16, 18]

steps:
  - run: echo "Node version: ${{ matrix.node }}"
```

**9. `inputs` - Workflow dispatch inputs**
```yaml
on:
  workflow_dispatch:
    inputs:
      name:
        description: 'Your name'
        required: true

jobs:
  greet:
    steps:
      - run: echo "Hello, ${{ inputs.name }}!"
```

### Full Context Reference

For complete documentation of all contexts and their properties:
https://docs.github.com/en/actions/learn-github-actions/contexts

**Pro tip**: To see all available data in a context, dump it as JSON:
```yaml
- run: echo '${{ toJson(github) }}'
```

## Key Takeaways

- **Events trigger workflows** - push, PR, schedule, or manual
- **Workflows contain jobs** - defined in `.github/workflows/*.yml`
- **Jobs contain steps** - executed sequentially on a runner
- **Steps are tasks** - shell commands or actions
- **Runners execute jobs** - GitHub-hosted, self-hosted, or third-party
- **Jobs run in parallel** unless dependencies specified with `needs`
- **Data passes via outputs** between steps and jobs
- **Environment variables** can be scoped to workflow, job, or step
- **Secrets and variables** store configuration securely
- **Contexts provide runtime data** accessible via `${{ }}`  syntax

## Next Chapter

You now understand the core building blocks of GitHub Actions workflows. In Chapter 6, we'll explore **Advanced Features** including matrix strategies for running jobs with multiple configurations, caching to speed up workflows, artifacts for passing files between jobs, and sophisticated conditional logic for complex workflows. These advanced techniques will transform you from writing basic workflows to architecting sophisticated CI/CD systems.
