# Chapter 6: Advanced Features - Extending Your Workflow Capabilities

## Overview

You've mastered the core features—events, workflows, jobs, steps, and runners. Now it's time to unlock GitHub Actions' advanced capabilities that will transform your workflows from simple scripts into sophisticated CI/CD systems.

This chapter covers features that **extend your ability to utilize the platform** rather than being essential from the start. While not all are difficult to use, they provide powerful mechanisms for:

- Choosing the right runner infrastructure for performance and cost
- Persisting data beyond ephemeral job execution
- Managing permissions and authentication securely
- Running jobs with multiple configurations simultaneously
- Controlling workflow execution flow dynamically

These are **distilled versions** capturing the essence of each feature without getting bogged down in minutiae. You'll see them applied in real-world examples throughout the capstone project later in this course.

## Runner Types: Choosing Your Execution Environment

When we examined the simplest "Hello World" workflow, we still used the `runs-on` field to specify a runner type. Now let's explore the **three categories of runners** available and when to use each.

### The Three Runner Categories

#### 1. GitHub-Hosted Runners

**What they are**: Virtual machines provided and maintained by GitHub.

**Available operating systems**:
- **Ubuntu**: `ubuntu-latest`, `ubuntu-24.04`, `ubuntu-22.04`, `ubuntu-20.04`
- **Windows**: `windows-latest`, `windows-2022`, `windows-2019`
- **macOS**: `macos-latest`, `macos-13`, `macos-12`

**Specifications** (standard GitHub-hosted):
- 2-core CPU
- 7 GB RAM
- 14 GB SSD disk space
- Pre-installed software (Docker, Node.js, Python, Go, Java, etc.)

**When to use**:
- Public repositories (free unlimited minutes)
- Standard build requirements
- No special infrastructure needs
- Quick setup without configuration

**Pricing**:
- **Public repos**: Free
- **Private repos**: Included minutes vary by plan, then $0.008/minute (Linux), $0.016/minute (Windows), $0.08/minute (macOS)

**Example**:
```yaml
jobs:
  test:
    runs-on: ubuntu-24.04  # GitHub-hosted Ubuntu
```

#### 2. Third-Party Hosted Runners

**What they are**: Runners provided by companies like **Namespace**, offering GitHub Actions-compatible infrastructure.

**Why use them**:
- **Performance**: Significantly faster compute and storage
- **Cost**: Often half the price of GitHub-hosted runners
- **Features**: Advanced caching, better observability, improved multi-architecture builds
- **Speed**: Faster startup times and network performance

**Namespace example specifications**:
- Customizable CPU and memory (e.g., 4-core, 16 GB)
- Fast SSD storage
- ARM64 and x86-64 architectures
- Custom VM images with pre-installed dependencies

**Example**:
```yaml
jobs:
  build:
    runs-on: namespace-profile-default  # Third-party runner
```

**Or with custom configuration**:
```yaml
jobs:
  build:
    runs-on:
      - namespace-profile-my-custom-image
      - nscloud-cpu-4-x64  # 4 CPU cores, x86-64
      - nscloud-ram-16-x64  # 16 GB RAM
```

**Performance comparison**: According to benchmarks by runs-on across providers:
- **Namespace**: Fastest x86-64 runners, 50% cost of GitHub-hosted
- **GitHub-hosted**: Standard performance, predictable but slower
- **Self-hosted (via runs-on)**: Cost-effective at scale

**Additional benefits**:
- **Observability**: CPU, memory, network, storage metrics for each job
- **Insights dashboard**: Execution timing trends over time
- **Filtering**: Drill down to specific jobs and view performance data
- **Logs retention**: Access logs from completed runners

#### 3. Self-Hosted Runners

**What they are**: Runners you install and maintain on your own infrastructure.

**The runner agent**: Open-source project you can install on:
- Physical servers in your data center
- VMs in your private cloud
- Kubernetes clusters (using Actions Runner Controller/ARC)
- On-demand cloud instances (using tools like runs-on)

**Runs-on project**: Deploys infrastructure into your AWS account that spins up on-demand EC2 instances for each runner job. Benefits:
- Reduces costs significantly (pay only for execution time)
- Keeps data within your VPC (compliance/security)
- Automatic scaling based on demand

**When to use**:
- Need access to internal resources (databases, private networks)
- Compliance requirements (data sovereignty)
- Very high build volumes (cost optimization)
- Custom hardware requirements (GPUs, specific CPU architectures)

**Example**:
```yaml
jobs:
  build:
    runs-on: my-self-hosted-runner-name
```

**Or with labels**:
```yaml
jobs:
  build:
    runs-on: [self-hosted, linux, x64, gpu]
```

### Running Jobs Inside Containers

Beyond choosing a runner type, you can **execute jobs within container images** for even more control over dependencies.

**Syntax**:
```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    container:
      image: alpine:latest
```

**What happens**:
1. GitHub allocates the specified runner (Ubuntu)
2. Runner pulls the container image (Alpine)
3. All steps execute inside the container
4. Container provides additional dependencies not in base image

**Use cases**:
- Specific language versions not pre-installed
- Custom toolchains or build environments
- Isolation between jobs
- Reproducibility (same container = same environment)

**Complete example**:
```yaml
jobs:
  alpine-test:
    runs-on: ubuntu-latest
    container:
      image: alpine:3.18
    steps:
      - run: uname -a  # Shows Alpine kernel info
      - run: cat /etc/os-release  # Shows Alpine OS
```

### Practical Example: Multiple Runner Types

```yaml
name: Runner Types

on: workflow_dispatch

jobs:
  github-ubuntu:
    runs-on: ubuntu-24.04
    steps:
      - run: |
          echo "Runner: $(uname -s) $(uname -m)"
          echo "OS: $(cat /etc/os-release | grep PRETTY_NAME)"

  github-windows:
    runs-on: windows-latest
    steps:
      - run: |
          echo "Runner: Windows"
          systeminfo | findstr /C:"OS Name"

  github-macos:
    runs-on: macos-latest
    steps:
      - run: |
          echo "Runner: $(uname -s) $(uname -m)"
          sw_vers

  container-alpine:
    runs-on: ubuntu-latest
    container: alpine:latest
    steps:
      - run: |
          echo "Container: Alpine Linux"
          cat /etc/os-release

  namespace-runner:
    runs-on: namespace-profile-default
    steps:
      - run: |
          echo "Third-party runner"
          echo "Runner type: NSC (Namespace Cloud)"
```

**Execution characteristics**:
- All jobs run **in parallel** (no dependencies)
- Each gets its own isolated runner
- GitHub-hosted jobs complete in 10-30 seconds
- Namespace job often faster due to optimized infrastructure

## Artifacts: Persisting Data Beyond Job Lifecycle

By default, jobs run in **ephemeral environments**—created at job start, destroyed at job end. Artifacts provide a mechanism to **persist data beyond that lifecycle**.

### What Are Artifacts?

**Definition**: Files or directories uploaded to GitHub's storage that persist after workflow completion.

**Retention**:
- Default: 90 days (configurable per repository)
- Maximum: 400 days
- Public repos: Free storage
- Private repos: Counts against storage quota

**Common use cases**:
- Build outputs (compiled binaries, Docker images metadata)
- Test results and coverage reports
- Log files for debugging
- Documentation generated during build

### Uploading Artifacts

**Action**: `actions/upload-artifact@v3`

**Basic syntax**:
```yaml
- uses: actions/upload-artifact@v3
  with:
    name: my-artifact  # Name shown in UI
    path: path/to/file.txt  # File or directory to upload
```

**Multiple files/patterns**:
```yaml
- uses: actions/upload-artifact@v3
  with:
    name: test-results
    path: |
      test-results/**/*.xml
      coverage/**/*.html
      logs/*.log
```

### Downloading Artifacts

**Action**: `actions/download-artifact@v3`

**Basic syntax**:
```yaml
- uses: actions/download-artifact@v3
  with:
    name: my-artifact  # Must match upload name
```

**Where it downloads**: Current working directory by default.

**Custom path**:
```yaml
- uses: actions/download-artifact@v3
  with:
    name: my-artifact
    path: ./downloaded-artifacts/
```

### Complete Example: Producer-Consumer Pattern

```yaml
name: Artifacts

on: workflow_dispatch

jobs:
  producer:
    runs-on: ubuntu-latest
    steps:
      - name: Create artifact
        run: |
          mkdir -p output
          echo "Hello from producer job at $(date)" > output/artifact.txt
          echo "Build ID: ${{ github.run_id }}" >> output/artifact.txt
      
      - name: Upload artifact
        uses: actions/upload-artifact@v3
        with:
          name: my-artifact
          path: output/artifact.txt

  consumer:
    needs: producer  # Wait for producer to finish
    runs-on: ubuntu-latest
    steps:
      - name: Download artifact
        uses: actions/download-artifact@v3
        with:
          name: my-artifact
      
      - name: Display artifact contents
        run: cat artifact.txt
```

**Execution flow**:
1. **Producer** creates `artifact.txt` with timestamp
2. **Producer** uploads file to GitHub's artifact storage
3. **Consumer** waits for producer to complete (via `needs`)
4. **Consumer** downloads artifact from storage
5. **Consumer** displays contents (same data from producer)

**In GitHub UI**:
- Go to workflow run summary page
- See "Artifacts" section at bottom
- Download as ZIP file
- Persists for 90 days (or your configured retention)

## Caching: Avoiding Repetitive Work

Similar to artifacts, but serving a different purpose: **caching** optimizes workflows by storing frequently-used dependencies that don't change often.

### When to Use Caching

**Problem**: Jobs are ephemeral. If you need to install dependencies, you do it **every single run**.

**Example without caching**:
```
Run 1: Download Node.js modules (45 seconds)
Run 2: Download Node.js modules (45 seconds)  ← Repetitive!
Run 3: Download Node.js modules (45 seconds)  ← Repetitive!
```

**Solution**: Cache dependencies after first run, restore them in subsequent runs.

```
Run 1: Download Node.js modules (45 seconds) → Cache them
Run 2: Restore from cache (3 seconds)  ← 15x faster!
Run 3: Restore from cache (3 seconds)  ← 15x faster!
```

### GitHub's Official Cache Action

**Action**: `actions/cache@v3`

**How it works**:
1. Specify **paths** to cache (directories or files)
2. Specify **cache key** (unique identifier)
3. First run: Cache miss → Populate cache
4. Subsequent runs: Cache hit → Restore from cache

**Basic syntax**:
```yaml
- uses: actions/cache@v3
  with:
    path: ~/.npm  # Directory to cache
    key: npm-${{ hashFiles('package-lock.json') }}  # Unique key
```

### Cache Keys: The Critical Detail

**Cache key determines**: When to invalidate and regenerate cache.

**Best practice**: Include hash of dependency lock file.

**Examples**:
- Node.js: `npm-${{ hashFiles('package-lock.json') }}`
- Python: `pip-${{ hashFiles('requirements.txt') }}`
- Go: `go-${{ hashFiles('go.sum') }}`

**Why this works**:
- Lock file changes → Hash changes → New cache key → Cache miss → Re-download dependencies
- Lock file unchanged → Same hash → Same cache key → Cache hit → Restore quickly

### Complete Caching Example

```yaml
name: Caching Example

on: workflow_dispatch

jobs:
  cache-demo:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v3
      
      - name: Restore cache
        id: restore-cache
        uses: actions/cache@v3
        with:
          path: /tmp/my-cache
          key: my-cache-key-v1
      
      - name: Check cache status
        run: |
          if [ "${{ steps.restore-cache.outputs.cache-hit }}" == "true" ]; then
            echo "Cache HIT - file exists!"
          else
            echo "Cache MISS - need to generate file"
          fi
      
      - name: Generate file (only on cache miss)
        if: steps.restore-cache.outputs.cache-hit != 'true'
        run: |
          mkdir -p /tmp/my-cache
          echo "Generated at $(date)" > /tmp/my-cache/data.txt
      
      - name: Display cached file
        run: cat /tmp/my-cache/data.txt
```

**First execution**:
- `cache-hit` output: `false`
- Executes generation step
- Automatically uploads `/tmp/my-cache` to cache storage

**Second execution**:
- `cache-hit` output: `true`
- Skips generation step (conditional)
- File already exists from cache restore

### Language-Specific Caching

Most setup actions include built-in caching support.

#### Node.js Example

```yaml
- uses: actions/setup-node@v3
  with:
    node-version: '18'
    cache: 'npm'  # Enables npm caching
    cache-dependency-path: 'package-lock.json'

- run: npm install  # Automatically uses cache
```

**How it works**:
- Action generates cache key from `package-lock.json` hash
- Restores `node_modules/` if cache hit
- `npm install` skips already-installed packages
- Significant speed improvement on cache hits

#### Go Example

```yaml
- uses: actions/setup-go@v4
  with:
    go-version: '1.21'
    cache: true  # Enables Go module caching

- run: go build ./...
```

#### Python Example

```yaml
- uses: actions/setup-python@v4
  with:
    python-version: '3.11'
    cache: 'pip'
    cache-dependency-path: 'requirements.txt'

- run: pip install -r requirements.txt
```

### GitHub Cache Limits

**Storage limit**: 10 GB per repository

**Eviction policy**: When limit exceeded, GitHub evicts **least recently used (LRU)** cache entries.

**Result**: Rolling 10 GB window of your most recent caches.

**Best practices**:
- Cache only what you need
- Use specific cache keys (avoid caching everything with one key)
- Consider cache size when deciding what to cache

### Namespace Cache Volumes: A Different Approach

While GitHub uses **object storage** (upload/download model), Namespace uses **volume snapshots** (mount model).

**How Namespace caching works**:
1. Specify a cache volume size and identifier
2. Volume mounts at job start (like a disk drive)
3. Read/write directly to volume during job
4. Volume snapshots automatically at job end
5. Next job mounts snapshot copy (instant access)

**Advantages**:
- **No upload/download time** (volume is already there)
- **Much faster** for large datasets or many small files
- **Simpler workflow** (no explicit save/restore steps)

**Trade-offs**:
- **Less deterministic** (snapshots may not be from exact cache key)
- **Good enough** for most use cases (dependencies rarely need exact matching)

**Example**:
```yaml
jobs:
  build:
    runs-on:
      - namespace-profile-default
      - nscloud-cache-size-20gb
      - nscloud-cache-tag-node-modules
    steps:
      - uses: namespacelabs/nscloud-cache-action@v1
        with:
          path: /home/runner/.npm
      
      - run: npm install  # Uses cached /home/runner/.npm
```

**Performance comparison** (from benchmark):
- **Single large file**: 
  - GitHub cache: 28 seconds
  - Namespace cache: 3 seconds (9x faster)
- **100,000 small files**:
  - GitHub cache: 72 seconds
  - Namespace cache: 4 seconds (18x faster)

**When cache misses occur**: Just slower execution, not a failure. The penalty is minimal compared to consistent speedups.

## Permissions: Controlling GitHub API Access

Workflows can interact with GitHub's API to:
- Read repository contents
- Create issues and PRs
- Add labels and comments
- Publish packages
- Manage releases

**By default**, workflows have **limited permissions**. You must explicitly grant additional access.

### Default Permissions

**Contents**: `read` (can clone repository code)  
**Packages**: `read` (can pull from GitHub Package Registry)

**Everything else**: `none`

### Available Permission Scopes

| Scope | Purpose | Actions |
|-------|---------|---------|
| `contents` | Repository code and releases | Clone, create releases |
| `issues` | Issue management | Create, edit, label issues |
| `pull-requests` | PR management | Create, edit, label PRs |
| `packages` | Package registry | Push/pull packages |
| `deployments` | Deployment tracking | Create deployment statuses |
| `actions` | Workflow management | Cancel runs, approve runs |
| `checks` | Check runs | Create check runs |
| `statuses` | Commit statuses | Set commit status |

**Full list**: https://docs.github.com/en/actions/security-guides/automatic-token-authentication#permissions-for-the-github_token

### Permission Levels

- **`read`**: Query/view data
- **`write`**: Create/modify data
- **`none`**: No access (default for most scopes)

### Specifying Permissions

**Workflow level** (applies to all jobs):
```yaml
name: My Workflow

permissions:
  contents: read
  pull-requests: write

jobs:
  # All jobs have these permissions
```

**Job level** (overrides workflow permissions):
```yaml
jobs:
  deploy:
    permissions:
      contents: write  # This job needs write access
      packages: write
```

**Step level**: Not supported (permissions apply at job level).

### Practical Example: PR Automation

```yaml
name: Permissions Example

on: workflow_dispatch

jobs:
  readonly-pr:
    permissions:
      pull-requests: read  # Can only READ PRs
    runs-on: ubuntu-latest
    steps:
      - name: List PRs
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: gh pr list --repo ${{ github.repository }}
      
      - name: Try to add label (WILL FAIL)
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: |
          PR_NUMBER=$(gh pr list --json number --jq '.[0].number')
          gh pr edit $PR_NUMBER --add-label "documentation"

  readwrite-pr:
    permissions:
      pull-requests: write  # Can READ and WRITE PRs
    runs-on: ubuntu-latest
    steps:
      - name: List PRs
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: gh pr list --repo ${{ github.repository }}
      
      - name: Add label (WILL SUCCEED)
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: |
          PR_NUMBER=$(gh pr list --json number --jq '.[0].number')
          gh pr edit $PR_NUMBER --add-label "documentation"
```

**Execution results**:
- **First job**: Lists PRs successfully, but fails when trying to add label (403 Forbidden)
- **Second job**: Lists PRs successfully AND adds label successfully

**The `GITHUB_TOKEN`**: Automatically created secret providing API authentication with specified permissions.

### Best Practice: Principle of Least Privilege

**Always use minimum permissions required**:
- Most workflows: `contents: read`
- Publishing packages: Add `packages: write`
- Interacting with PRs/issues: Add `pull-requests: write` or `issues: write`

**Why this matters**: Compromised workflows (supply chain attacks, malicious PRs) have limited blast radius.

## Authentication to Third-Party Systems

Workflows often need to interact with external services: AWS, Azure, GCP, Docker Hub, etc. Two methods exist for authentication.

### Method 1: Static Credentials (API Keys)

**What they are**: Long-lived credentials stored as GitHub secrets.

**Characteristics**:
- **Security**: Less secure (credentials live indefinitely)
- **Compatibility**: Works with every service
- **Rotation**: Manual process
- **Risk**: If leaked, valid until rotated

**Setup process**:
1. Generate API key in third-party service
2. Add as GitHub Actions secret
3. Reference secret in workflow

**Example - AWS static credentials**:

**In AWS Console**:
1. IAM → Users → Create user
2. Generate access key
3. Save access key ID and secret access key

**In GitHub**:
1. Settings → Secrets and variables → Actions
2. New repository secret: `AWS_ACCESS_KEY_ID`
3. New repository secret: `AWS_SECRET_ACCESS_KEY`

**In workflow**:
```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: aws-actions/configure-aws-credentials@v2
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: us-east-1
      
      - run: aws s3 ls  # Authenticated!
```

### Method 2: OIDC Tokens (Recommended)

**What they are**: Short-lived credentials generated at runtime via OpenID Connect (OIDC).

**Characteristics**:
- **Security**: More secure (tokens expire quickly)
- **Compatibility**: Requires OIDC support in service
- **Rotation**: Automatic (new token each run)
- **Risk**: If leaked, expires in minutes

**How it works**:
1. GitHub generates JWT (JSON Web Token) signed by GitHub
2. Workflow presents JWT to third-party service
3. Service validates JWT signature with GitHub's public keys
4. Service issues short-lived credentials
5. Workflow uses credentials
6. Credentials expire after job completes

**Supported services**:
- AWS (via IAM role assumption)
- Azure (via workload identity federation)
- GCP (via workload identity federation)
- HashiCorp Vault
- Many others

### OIDC with AWS: Complete Setup

**Step 1: Add OIDC provider in AWS IAM**

1. IAM → Identity providers → Add provider
2. Provider type: OpenID Connect
3. Provider URL: `https://token.actions.githubusercontent.com`
4. Audience: `sts.amazonaws.com`
5. Click "Add provider"

**Step 2: Create IAM role with trust policy**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::ACCOUNT_ID:oidc-provider/token.actions.githubusercontent.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
        },
        "StringLike": {
          "token.actions.githubusercontent.com:sub": "repo:YOUR_ORG/YOUR_REPO:*"
        }
      }
    }
  ]
}
```

**What this trust policy does**:
- Allows GitHub Actions to assume this role
- **Only** from specified repository
- Only when audience matches
- Prevents other repos from assuming your role

**Step 3: Use in workflow**

```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    permissions:
      id-token: write  # REQUIRED for OIDC
      contents: read
    steps:
      - uses: aws-actions/configure-aws-credentials@v2
        with:
          role-to-assume: arn:aws:iam::ACCOUNT_ID:role/GitHubActionsRole
          aws-region: us-east-1
      
      - run: aws sts get-caller-identity  # Verify authentication
      - run: aws s3 ls  # Perform actual work
```

**Critical detail**: `id-token: write` permission is **required**. Without it, workflow cannot request JWT from GitHub.

**Security benefits**:
- No long-lived credentials to leak
- Scoped to specific repository
- Automatic expiration
- Audit trail via CloudTrail

## Matrix Strategies: Run Jobs with Multiple Configurations

Often you need to test code across multiple environments:
- Different OS versions (Ubuntu 20.04, 22.04, 24.04)
- Different language versions (Node 16, 18, 20)
- Different database versions (PostgreSQL 13, 14, 15)

**Matrix strategies** let you define configurations once and run jobs for all combinations.

### Basic Matrix Syntax

```yaml
jobs:
  test:
    strategy:
      matrix:
        os: [ubuntu-latest, windows-latest, macos-latest]
        node: [16, 18, 20]
    
    runs-on: ${{ matrix.os }}
    steps:
      - uses: actions/setup-node@v3
        with:
          node-version: ${{ matrix.node }}
      - run: npm test
```

**What happens**: GitHub creates **9 jobs** (3 OS × 3 Node versions):
1. ubuntu-latest + Node 16
2. ubuntu-latest + Node 18
3. ubuntu-latest + Node 20
4. windows-latest + Node 16
5. windows-latest + Node 18
6. windows-latest + Node 20
7. macos-latest + Node 16
8. macos-latest + Node 18
9. macos-latest + Node 20

All jobs run **in parallel** (subject to runner availability).

### Excluding Combinations

**Problem**: Sometimes certain combinations don't make sense or are known to fail.

**Solution**: Use `exclude` to skip specific combinations.

```yaml
strategy:
  matrix:
    os: [ubuntu-latest, windows-latest]
    node: [16, 18, 20]
    exclude:
      - os: windows-latest
        node: 16  # Skip Windows + Node 16
```

**Result**: 5 jobs instead of 6 (all except Windows + Node 16).

### Including Additional Combinations

**Use case**: Add one specific configuration not covered by matrix.

```yaml
strategy:
  matrix:
    os: [ubuntu-latest]
    node: [18, 20]
    include:
      - os: macos-latest  # Add one macOS job
        node: 18
```

**Result**: 3 jobs (ubuntu+18, ubuntu+20, macos+18).

### Practical Matrix Example

```yaml
name: Matrix and Conditionals

on: workflow_dispatch

jobs:
  matrix-job:
    strategy:
      matrix:
        number: [1, 2]
        letter: ['a', 'b', 'c']
        exclude:
          - number: 1
            letter: 'c'  # Skip 1-c combination
    
    runs-on: ubuntu-latest
    steps:
      - name: Step 1
        run: echo "Number=${{ matrix.number }}, Letter=${{ matrix.letter }}"
      
      - name: Step 2 (conditional)
        if: matrix.number == 2 && matrix.letter == 'c'
        run: echo "This only runs for 2-c combination"
```

**Execution**:
- **5 jobs created**: 1-a, 1-b, 2-a, 2-b, 2-c (1-c excluded)
- **All run in parallel**
- **In 2-c job**: Step 2 executes
- **In other jobs**: Step 2 skipped

## Conditionals: Dynamic Execution Control

Conditionals allow steps or jobs to run only when certain conditions are met.

### Step-Level Conditionals

**Syntax**:
```yaml
- name: Conditional step
  if: <expression>
  run: echo "This runs conditionally"
```

**Common expressions**:

**Check job status**:
```yaml
if: success()  # Previous steps succeeded
if: failure()  # Previous step failed
if: always()   # Run regardless of status
```

**Check variables**:
```yaml
if: github.ref == 'refs/heads/main'  # Only on main branch
if: github.event_name == 'pull_request'  # Only for PRs
if: matrix.os == 'ubuntu-latest'  # Matrix value check
```

**Check secrets/variables**:
```yaml
if: secrets.DEPLOY_KEY != ''  # Secret is set
```

**Combine conditions**:
```yaml
if: github.ref == 'refs/heads/main' && success()
if: failure() || cancelled()
```

### Job-Level Conditionals

Same syntax at job level:

```yaml
jobs:
  deploy-staging:
    if: github.ref == 'refs/heads/develop'
    # Only runs when pushing to develop branch

  deploy-production:
    if: github.ref == 'refs/heads/main'
    # Only runs when pushing to main branch
```

### Practical Example

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - run: npm run build
      - run: npm test
  
  deploy:
    needs: build
    if: github.ref == 'refs/heads/main' && success()
    runs-on: ubuntu-latest
    steps:
      - run: echo "Deploying to production..."
```

**Logic**:
- Build job: Always runs
- Deploy job: Only if build succeeds AND on main branch

## Concurrency: Controlling Multiple Workflow Runs

**Problem**: Multiple commits pushed quickly trigger multiple workflow runs. Often, only the latest run matters.

**Example scenario**:
```
Commit A pushed → Workflow run #1 starts (takes 5 minutes)
Commit B pushed → Workflow run #2 starts (takes 5 minutes)
Commit C pushed → Workflow run #3 starts (takes 5 minutes)
```

Without concurrency control, all three run in parallel, wasting resources. Run #1 and #2 are obsolete the moment #3 starts.

### Concurrency Configuration

**Syntax**:
```yaml
concurrency:
  group: <group-name>
  cancel-in-progress: <true|false>
```

**Group name**: Workflows with same group name are considered concurrent.

**cancel-in-progress**:
- `true`: Cancel older runs when new run starts
- `false`: Queue new runs (wait for old ones to finish)

### Common Concurrency Patterns

**Pattern 1: One workflow run per branch**

```yaml
concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true
```

**Result**: New push to same branch cancels in-progress run from previous push.

**Pattern 2: One workflow run per PR**

```yaml
concurrency:
  group: ${{ github.workflow }}-${{ github.event.pull_request.number }}
  cancel-in-progress: true
```

**Result**: New commits to same PR cancel previous PR checks.

**Pattern 3: Never cancel production deploys**

```yaml
concurrency:
  group: deploy-production
  cancel-in-progress: false
```

**Result**: Production deploys queue up and run sequentially (never interrupt mid-deploy).

### Complete Example

```yaml
name: Matrix and Conditionals

on:
  push:
    branches: [main, develop]

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

jobs:
  matrix-job:
    strategy:
      matrix:
        number: [1, 2]
        letter: ['a', 'b', 'c']
    runs-on: ubuntu-latest
    steps:
      - run: echo "Matrix: ${{ matrix.number }}-${{ matrix.letter }}"
  
  sleep-job:
    runs-on: ubuntu-latest
    steps:
      - run: sleep 100  # Deliberate delay to test cancellation
```

**Test scenario**:
1. Trigger workflow (it starts sleeping)
2. While sleeping, trigger again
3. **Result**: First run cancels, second run takes over

**In GitHub UI**: First run shows "Status: Canceled" with explanation that newer run superseded it.

## Key Takeaways

**Runner types**:
- GitHub-hosted: Easy, pre-configured, good for standard workloads
- Third-party (Namespace): Faster, cheaper, better observability
- Self-hosted: Maximum control, best for special requirements

**Artifacts**: Persist files across jobs and after workflow completion (build outputs, test results, logs)

**Caching**: Speed up workflows by avoiding repetitive dependency downloads (language packages, build tools)

**Permissions**: Grant minimum necessary GitHub API access per job (contents, issues, pull-requests, packages)

**Authentication**:
- Static credentials: Simple, universally compatible, less secure
- OIDC tokens: Complex setup, more secure, automatic rotation

**Matrix strategies**: Test across multiple configurations simultaneously (OS versions, language versions, architectures)

**Conditionals**: Run steps/jobs only when conditions met (branch checks, success/failure status, variable values)

**Concurrency**: Prevent wasted resources by canceling obsolete workflow runs (per-branch, per-PR grouping)

## Next Chapter

You've now mastered both core and advanced GitHub Actions features. In Chapter 7, we'll explore the **GitHub Actions Marketplace**—a ecosystem of thousands of pre-built actions for common tasks. You'll learn how to find, evaluate, and integrate marketplace actions into your workflows, dramatically accelerating development by leveraging community-built solutions. From deploying to cloud providers to sending notifications, the marketplace has actions for almost every CI/CD task imaginable.
