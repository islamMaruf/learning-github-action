# Chapter 9: Common Workflows - Real-World CI/CD Patterns

## Overview

Now that you understand the features and capabilities GitHub Actions provides, it's time to see how they combine into real-world workflows. This chapter walks through **common workflow patterns** you'll encounter when automating software delivery.

We'll explore:
- **Linting** - Enforcing code quality standards
- **Testing** - Running automated test suites
- **Static analysis** - Security scanning and code analysis
- **Building** - Compiling applications and container images
- **Deploying** - Push-based and GitOps deployment strategies
- **Repository automation** - Releases, stale issue management, dependency updates

For each pattern, we'll start with a **naive baseline** implementation, then explore **optimizations** using marketplace actions and advanced features.

## Approaching Workflow Design

**Starting principle**: Break down your task into individual steps.

**Process**:
1. **Identify the goal** - What needs to happen?
2. **List the steps** - What individual actions achieve this?
3. **Implement baseline** - Get it working first
4. **Optimize** - Improve performance, user experience, maintainability

**Philosophy**: Start simple, iterate toward excellence.

## Linting: Enforcing Code Quality

### Naive Implementation

**Goal**: Run linters to catch code quality issues.

**Basic approach**:
```yaml
name: Lint Code

on: [push, pull_request]

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
      
      - name: Install linter
        run: npm install -g eslint
      
      - name: Run linter
        run: eslint .
```

**What this does**:
- Checks out repository code
- Installs ESLint globally
- Runs linter on all files

**Problems**:
- Lints **all files** every time (slow on large codebases)
- No reporting of results in GitHub UI
- Doesn't cache dependencies
- Only supports one language

### Optimized Implementation

**Improvements we want**:
1. **Detect changed files** - Only lint what changed
2. **Report status** - Show results as PR checks/annotations
3. **Multi-language support** - Lint Python, JavaScript, Go, etc.
4. **Efficient execution** - Cache dependencies, parallel execution

**Solution**: Use **Super-Linter** action from marketplace.

```yaml
name: Lint Code (Optimized)

on: [push, pull_request]

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
        with:
          fetch-depth: 0  # Fetch full history for diff analysis
      
      - name: Run Super-Linter
        uses: github/super-linter@v5
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          VALIDATE_ALL_CODEBASE: false  # Only lint changed files
```

**What Super-Linter provides**:
- **40+ language support** - JavaScript, TypeScript, Python, Go, Ruby, Shell, Docker, YAML, Markdown, etc.
- **Changed file detection** - Only runs on modified files (requires `fetch-depth: 0`)
- **GitHub integration** - Annotations appear on PR diffs
- **Configured by default** - Sensible defaults, customizable via `.github/linters/` config files

**Key configuration**:
- `fetch-depth: 0` - Downloads full git history (needed to calculate diffs)
- `GITHUB_TOKEN` - Enables status check integration
- `VALIDATE_ALL_CODEBASE: false` - Optimizes by only validating changes

**Result**: Four lines of code replace a multi-step custom solution while gaining performance, multi-language support, and better UX.

## Testing: Running Automated Test Suites

### Naive Implementation

**Goal**: Run unit/integration tests on every push.

**Basic approach**:
```yaml
name: Test

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
      
      - name: Install Node.js
        run: |
          curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
          sudo apt-get install -y nodejs
      
      - name: Install dependencies
        run: npm install
      
      - name: Build application
        run: npm run build
      
      - name: Run tests
        run: npm test
```

**Problems**:
- No dependency caching (re-downloads on every run)
- Test results not preserved
- Manual Node.js installation (fragile)
- No test result reporting

### Optimized Implementation

**Improvements**:
1. **Use official setup actions** - Reliable, maintained installers
2. **Enable caching** - Restore dependencies from cache
3. **Upload artifacts** - Preserve test results
4. **Report status** - Integrate test results with PR UI

```yaml
name: Test (Optimized)

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
      
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'  # Automatically caches node_modules
      
      - name: Install dependencies
        run: npm ci  # Faster, more reliable than npm install
      
      - name: Build application
        run: npm run build
      
      - name: Run tests
        run: npm test -- --coverage
      
      - name: Upload test results
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: test-results
          path: coverage/
```

**Key improvements**:
- **`actions/setup-node@v4`** - Official Node.js installer with built-in caching
- **`cache: 'npm'`** - Automatically caches `node_modules` based on `package-lock.json` hash
- **`npm ci`** - Clean install (faster, more reproducible than `npm install`)
- **`if: always()`** - Uploads results even if tests fail
- **Artifacts** - Test coverage reports available for download in workflow UI

**Official setup actions** (use these instead of manual installation):
- `actions/setup-node` - Node.js with npm/yarn caching
- `actions/setup-python` - Python with pip caching
- `actions/setup-java` - Java/JVM with Maven/Gradle caching
- `actions/setup-go` - Go with module caching

## Static Analysis: Security and Code Quality

### Implementation Pattern

**Goal**: Scan code for security vulnerabilities, code smells, technical debt.

**Common tools**:
- **CodeQL** - GitHub's semantic code analysis (finds security issues)
- **Snyk** - Dependency vulnerability scanning
- **SonarQube** - Code quality metrics
- **Trivy** - Container image vulnerability scanning

**Baseline approach**:
```yaml
name: Static Analysis

on: [push, pull_request]

jobs:
  analyze:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
      
      - name: Install analysis tool
        run: |
          # Install tool (e.g., Bandit for Python security)
          pip install bandit
      
      - name: Run analysis
        run: bandit -r . -f json -o report.json
      
      - name: Upload report
        uses: actions/upload-artifact@v4
        with:
          name: analysis-report
          path: report.json
```

**Pattern**:
1. Checkout code
2. Install analysis tool
3. Run scan
4. Upload/report results

**Optimization opportunities** (left as exercise):
- Use marketplace action for tool (easier, cached)
- Only scan changed files
- Integrate results as PR comments/annotations
- Fail workflow on critical findings

## Building: Compiling Applications

### Building Executables

**Goal**: Compile source code into deployable artifacts.

**Pattern**:
```yaml
name: Build Application

on: [push]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
      
      - name: Setup language toolchain
        uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Build application
        run: npm run build
      
      - name: Publish to registry
        run: |
          npm config set //registry.npmjs.org/:_authToken ${{ secrets.NPM_TOKEN }}
          npm publish
```

**Steps**:
1. Checkout code
2. Setup language toolchain (Node.js, Go, Rust, etc.)
3. Install dependencies
4. Build application
5. Publish to registry (npm, PyPI, Maven Central, etc.)

### Building Container Images

**Goal**: Build Docker image and push to registry.

**Pattern**:
```yaml
name: Build Container Image

on: [push]

jobs:
  build-image:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
      
      - name: Setup Docker Buildx
        uses: docker/setup-buildx-action@v3
      
      - name: Login to GitHub Container Registry
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      
      - name: Build and push
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ghcr.io/${{ github.repository }}:${{ github.sha }}
          cache-from: type=registry,ref=ghcr.io/${{ github.repository }}:buildcache
          cache-to: type=registry,ref=ghcr.io/${{ github.repository }}:buildcache,mode=max
```

**Key components**:
- **Docker Buildx** - Advanced build features (multi-platform, caching)
- **Login action** - Authenticates to registry
- **Build-push action** - Builds and pushes in one step
- **Layer caching** - Dramatically speeds up builds by reusing layers

**Registries supported**:
- **GitHub Container Registry** (ghcr.io) - Free for public repos
- **Docker Hub** - Most popular
- **AWS ECR** - Elastic Container Registry
- **Azure ACR** - Azure Container Registry
- **Google Artifact Registry** - GCP registry

## Deploying: Getting Code to Production

### Push-Based Deployment

**Definition**: Workflow authenticates to target environment and pushes code directly.

**Use case**: Traditional deployments where GitHub Actions has credentials for production.

**Pattern**:
```yaml
name: Deploy to Production

on:
  workflow_run:
    workflows: ["Build Container Image"]
    types: [completed]
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: production  # Requires approval
    steps:
      - name: Checkout deployment configs
        uses: actions/checkout@v4
      
      - name: Configure kubectl
        run: |
          mkdir -p $HOME/.kube
          echo "${{ secrets.KUBECONFIG }}" > $HOME/.kube/config
      
      - name: Deploy to Kubernetes
        run: |
          kubectl set image deployment/my-app \
            app=ghcr.io/${{ github.repository }}:${{ github.sha }}
          kubectl rollout status deployment/my-app
      
      - name: Validate deployment
        run: |
          # Wait for pods to be ready
          kubectl wait --for=condition=ready pod -l app=my-app --timeout=300s
          # Run smoke tests
          curl -f https://my-app.production.example.com/health
```

**Components**:
1. **Checkout configs** - Get deployment manifests/scripts
2. **Install tooling** - kubectl, helm, AWS CLI, etc.
3. **Authenticate** - Use OIDC or static credentials
4. **Deploy** - Apply changes to environment
5. **Validate** - Health checks, smoke tests

**Authentication options**:

**OIDC (recommended)**:
```yaml
- name: Configure AWS credentials
  uses: aws-actions/configure-aws-credentials@v4
  with:
    role-to-assume: arn:aws:iam::123456789012:role/GithubActionsRole
    aws-region: us-east-1
```

**Static credentials** (less secure):
```yaml
- name: Configure AWS credentials
  uses: aws-actions/configure-aws-credentials@v4
  with:
    aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
    aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
    aws-region: us-east-1
```

### GitOps Deployment

**Definition**: Workflow updates git repository; agent in cluster pulls changes.

**Use case**: Modern cloud-native deployments (Argo CD, Flux, Weaveworks).

**Advantages**:
- **No cluster credentials in GitHub** - More secure
- **Git as single source of truth** - Audit trail, rollback via git
- **Declarative** - Describe desired state, agent reconciles
- **Separation of concerns** - CI builds, CD deploys

**Pattern**:
```yaml
name: Update Deployment Manifest

on:
  workflow_run:
    workflows: ["Build Container Image"]
    types: [completed]
    branches: [main]

jobs:
  update-manifest:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout deployment repo
        uses: actions/checkout@v4
        with:
          repository: my-org/k8s-manifests
          token: ${{ secrets.MANIFEST_REPO_TOKEN }}
      
      - name: Update image tag
        run: |
          # Update Kubernetes manifest with new image
          sed -i "s|image: ghcr.io/.*|image: ghcr.io/${{ github.repository }}:${{ github.sha }}|" \
            manifests/production/deployment.yaml
      
      - name: Commit and push
        run: |
          git config user.name "github-actions[bot]"
          git config user.email "github-actions[bot]@users.noreply.github.com"
          git add manifests/production/deployment.yaml
          git commit -m "Update production image to ${{ github.sha }}"
          git push
```

**Flow**:
1. **Build workflow** completes successfully
2. **GitOps workflow** triggers
3. Updates manifest in deployment repository
4. Commits and pushes change
5. **GitOps agent** (Argo CD/Flux) detects change
6. Agent applies updates to cluster
7. Application deploys automatically

**Why this works**:
- GitHub Actions only needs access to git repositories
- No production credentials stored in GitHub
- Deployment history preserved in git
- Easy rollback (revert commit)

## Repository Automation: Release Management

### Automated Releases

**Goal**: Automatically create versioned releases based on commit history.

**Workflow**:
1. Determine what changed since last release
2. Calculate version bump (major, minor, patch)
3. Update version references in codebase
4. Generate changelog
5. Commit changes
6. Create GitHub release

**Implementation strategy**:

**Conventional Commits**:
```
feat: add user authentication (minor version bump)
fix: resolve memory leak (patch version bump)
feat!: redesign API (major version bump - breaking change)
chore: update dependencies (no version bump)
```

**Pattern**: Prefix commits with type to indicate semantic version change.

**Baseline implementation**:
```yaml
name: Release

on:
  push:
    branches: [main]

jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
        with:
          fetch-depth: 0  # Need full history
      
      - name: Determine new version
        id: version
        run: |
          # Parse commits since last tag
          # Extract conventional commit types (feat, fix, etc.)
          # Calculate semver bump
          echo "new_version=v1.2.3" >> $GITHUB_OUTPUT
      
      - name: Update package.json
        run: |
          npm version ${{ steps.version.outputs.new_version }} --no-git-tag-version
      
      - name: Generate changelog
        run: |
          # Parse commits and generate CHANGELOG.md
          echo "## ${{ steps.version.outputs.new_version }}" >> CHANGELOG.md
      
      - name: Commit changes
        run: |
          git config user.name "github-actions[bot]"
          git config user.email "github-actions[bot]@users.noreply.github.com"
          git add package.json CHANGELOG.md
          git commit -m "chore: release ${{ steps.version.outputs.new_version }}"
          git push
      
      - name: Create GitHub release
        uses: actions/create-release@v1
        with:
          tag_name: ${{ steps.version.outputs.new_version }}
          release_name: Release ${{ steps.version.outputs.new_version }}
```

**Marketplace solution**: **Release Please** (by Google)

```yaml
name: Release

on:
  push:
    branches: [main]

jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - uses: google-github-actions/release-please-action@v3
        with:
          release-type: node
          package-name: my-package
```

**What Release Please does**:
- Parses conventional commits
- Creates "release PR" with version bumps and changelog
- When PR merged, creates GitHub release automatically
- Handles multiple packages in monorepos

## Repository Automation: Stale Issues

**Goal**: Automatically mark and close stale issues/PRs.

**Why**: Keeps issue tracker tidy, encourages activity.

**Baseline implementation**:
```yaml
name: Close Stale Issues

on:
  schedule:
    - cron: '0 0 * * *'  # Daily at midnight

jobs:
  stale:
    runs-on: ubuntu-latest
    steps:
      - name: Fetch issues
        run: |
          # Use GitHub API to list all issues
          # Filter for issues inactive > 30 days
      
      - name: Mark as stale
        run: |
          # Add "stale" label
          # Post comment warning of closure
      
      - name: Close stale issues
        run: |
          # Find issues stale > 7 days
          # Close them
```

**Marketplace solution**: **Stale action**

```yaml
name: Close Stale Issues

on:
  schedule:
    - cron: '0 0 * * *'

jobs:
  stale:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/stale@v8
        with:
          repo-token: ${{ secrets.GITHUB_TOKEN }}
          stale-issue-message: 'This issue is stale because it has been open 30 days with no activity. Remove stale label or comment or this will be closed in 7 days.'
          stale-pr-message: 'This PR is stale because it has been open 30 days with no activity.'
          close-issue-message: 'This issue was closed because it has been stalled for 7 days with no activity.'
          days-before-stale: 30
          days-before-close: 7
          stale-issue-label: 'stale'
          stale-pr-label: 'stale'
```

**Configuration options**:
- `days-before-stale` - Inactivity threshold
- `days-before-close` - Grace period after stale label
- Custom messages for issues/PRs
- Exempt labels (e.g., "pinned" issues never go stale)

## Repository Automation: Dependency Updates

**Goal**: Automatically update dependencies to latest versions.

**Reasons**:
- **Security patches** - Critical vulnerabilities fixed in newer versions
- **New features** - Benefit from improvements
- **Compatibility** - Stay current with ecosystem

**Baseline implementation**:
```yaml
name: Update Dependencies

on:
  schedule:
    - cron: '0 0 * * 1'  # Weekly on Monday

jobs:
  update:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
      
      - name: Check for updates
        run: |
          # Read package-lock.json
          # Query registry for newer versions
          npm outdated
      
      - name: Update dependencies
        run: |
          npm update
      
      - name: Create PR
        run: |
          git checkout -b update-dependencies-$(date +%Y%m%d)
          git add package.json package-lock.json
          git commit -m "chore: update dependencies"
          git push origin update-dependencies-$(date +%Y%m%d)
          # Use GitHub API to create PR
```

**Marketplace/service solution**: **Dependabot** (built into GitHub)

**Setup**: Create `.github/dependabot.yml`

```yaml
version: 2
updates:
  - package-ecosystem: "npm"
    directory: "/"
    schedule:
      interval: "weekly"
    open-pull-requests-limit: 10
    
  - package-ecosystem: "docker"
    directory: "/"
    schedule:
      interval: "weekly"
  
  - package-ecosystem: "github-actions"
    directory: "/"
    schedule:
      interval: "weekly"
```

**What Dependabot does**:
- Monitors dependencies (npm, pip, Maven, Docker, GitHub Actions, etc.)
- Creates PR for each update
- Includes changelog and compatibility notes
- Triggers your test workflow automatically
- Closes PR if tests fail

**Result**: Dependency updates become automatic, reviewable, testable PRs.

## GitHub's Template Workflows

**Location**: Actions tab → "New workflow" button

**What they are**: Pre-built workflow templates for common scenarios.

**Categories**:
- **Deployment** - AWS ECS, Azure, Google Cloud, Kubernetes
- **Continuous Integration** - Node.js, Python, Go, Java, .NET
- **Security** - CodeQL analysis, dependency scanning
- **Automation** - Stale issues, greeting new contributors

**How to use**:
1. Navigate to Actions tab in repository
2. Click "New workflow"
3. Browse suggested workflows (GitHub suggests based on repository languages)
4. Click "Configure" on relevant template
5. Customize and commit

### Example: Docker Image Template

```yaml
name: Docker Image CI

on:
  push:
    branches: [ "main" ]
  pull_request:
    branches: [ "main" ]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3
    - name: Build the Docker image
      run: docker build . --file Dockerfile --tag my-image-name:$(date +%s)
```

**What this provides**:
- Basic structure
- Trigger events
- Build command

**What you'd add**:
- Push to registry
- Multi-stage builds
- Caching
- Tagging strategy

### Example: AWS ECS Deployment Template

**Before using**, you must:
1. Create ECR repository
2. Create ECS task definition (store in repo as `task-definition.json`)
3. Configure IAM permissions for workflow

**Workflow structure**:
```yaml
name: Deploy to Amazon ECS

on:
  push:
    branches: [ "main" ]

env:
  AWS_REGION: us-east-1
  ECR_REPOSITORY: my-app
  ECS_SERVICE: my-service
  ECS_CLUSTER: my-cluster
  CONTAINER_NAME: my-app

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4
      
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ env.AWS_REGION }}
      
      - name: Login to Amazon ECR
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v2
      
      - name: Build, tag, and push image to Amazon ECR
        id: build-image
        env:
          ECR_REGISTRY: ${{ steps.login-ecr.outputs.registry }}
          IMAGE_TAG: ${{ github.sha }}
        run: |
          docker build -t $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG .
          docker push $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG
          echo "image=$ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG" >> $GITHUB_OUTPUT
      
      - name: Fill in the new image ID in the Amazon ECS task definition
        id: task-def
        uses: aws-actions/amazon-ecs-render-task-definition@v1
        with:
          task-definition: task-definition.json
          container-name: ${{ env.CONTAINER_NAME }}
          image: ${{ steps.build-image.outputs.image }}
      
      - name: Deploy Amazon ECS task definition
        uses: aws-actions/amazon-ecs-deploy-task-definition@v1
        with:
          task-definition: ${{ steps.task-def.outputs.task-definition }}
          service: ${{ env.ECS_SERVICE }}
          cluster: ${{ env.ECS_CLUSTER }}
          wait-for-service-stability: true
```

**Improvements to make**:
- **Change authentication** - Use OIDC instead of static keys
- **Add build caching** - Speed up Docker builds
- **Add rollback** - On validation failure
- **Add smoke tests** - Validate deployment health

## Key Workflow Design Principles

**1. Start simple, optimize later**
- Get baseline working first
- Measure performance before optimizing
- Don't prematurely optimize

**2. Leverage marketplace actions**
- Don't reinvent the wheel
- Use well-maintained, popular actions
- Pin to commit SHAs for security

**3. Make workflows observable**
- Upload artifacts (test results, build outputs)
- Use annotations for errors/warnings
- Set meaningful job names and step names

**4. Design for failure**
- Use `if: always()` to ensure cleanup steps run
- Add validation steps after deployments
- Include rollback procedures

**5. Optimize for speed**
- Enable caching (dependencies, build outputs)
- Run jobs in parallel where possible
- Only run on relevant file changes (path filters)

**6. Keep workflows DRY**
- Use reusable workflows for repeated logic
- Create composite actions for step sequences
- Use matrix strategies for similar jobs

## Summary: Workflow Patterns

| Workflow Type | Key Steps | Common Tools |
|--------------|-----------|--------------|
| **Linting** | Checkout → Detect changes → Run linters → Report | Super-Linter, ESLint, Pylint |
| **Testing** | Checkout → Setup toolchain → Install deps → Build → Test → Upload results | setup-node/python/go, Jest, pytest |
| **Static Analysis** | Checkout → Install tool → Scan → Report | CodeQL, Snyk, Trivy |
| **Build Executable** | Checkout → Setup toolchain → Install deps → Build → Publish | setup-node/go/rust, npm/cargo |
| **Build Image** | Checkout → Setup Docker → Build → Push | docker/build-push-action |
| **Deploy (Push)** | Checkout configs → Authenticate → Deploy → Validate | AWS/Azure/GCP actions |
| **Deploy (GitOps)** | Checkout manifest repo → Update version → Commit → Push | git commands |
| **Release** | Parse commits → Bump version → Update changelog → Create release | Release Please |
| **Stale Issues** | Fetch issues → Mark stale → Close old | Stale action |
| **Dependencies** | Check for updates → Create PR → Run tests | Dependabot |

## Next Chapter

You've learned how to design and build real-world workflows for common CI/CD scenarios. In Chapter 10, we'll explore **Developer Experience**—the tools and techniques for debugging workflows, testing actions locally, and iterating quickly. Great workflows aren't just functional; they're also easy to develop and maintain. You'll learn how to optimize your development loop to build better automation faster.
