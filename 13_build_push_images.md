# Chapter 13: Building and Pushing Container Images

## Introduction

After establishing a robust test workflow in the previous chapter, we now turn to building and publishing container images—a critical component of modern deployment pipelines. This chapter demonstrates how to automate the creation of Docker images for multiple services, intelligently filter which services need rebuilding, implement proper versioning strategies, and push images to container registries.

Building on the patterns established for testing, we'll create a workflow that handles multi-service image builds with dynamic filtering, environment-specific tagging, and support for both automated and manual deployments. This workflow exemplifies production-ready container image management in a CI/CD pipeline.

## Container Build Requirements

### Project Context

Our capstone project consists of five containerized services, each requiring its own Docker image:

1. **Node.js API** (`services/node/api-node`)
2. **Golang API** (`services/go/api-golang`)
3. **React Client** (`services/react/client-react`)
4. **Python Load Generator** (`services/python/load-generator`)
5. **Database Migrator** (`services/other/migrator`)

Each service directory contains:
- **Dockerfile**: Container image definition
- **Taskfile.yaml**: Build task definitions with standardized interface
- **Source code**: Application files to package into the image

### Versioning Strategy

We need different versioning approaches for different environments:

**Production Releases:**
- Format: `x.y.z` (semantic versioning)
- Triggered by: Git tags matching pattern `service-name@x.y.z`
- Example: `1.0.9` for a production release

**Non-Production Builds:**
- Format: `x.y.z-NNNN-githash` (extended versioning)
- Components:
  - `x.y.z`: Latest release version
  - `NNNN`: Zero-padded commit count since release (e.g., `0047`)
  - `githash`: Short commit hash (e.g., `188f7`)
- Example: `1.0.9-0047-188f7`

**Why zero-padding?** Without left-padding the commit count, version `1.0.9-11-abc` would sort before `1.0.9-3-def` in alphabetical ordering (common in many tools). Zero-padding to four digits ensures correct sorting up to 10,000 commits between releases.

### Build Conventions

Following the patterns established in our test workflow:

**Standardized Task Interface:**
- `task build`: Build single-architecture image
- `task build-multi`: Build multi-architecture image with buildx
- `task utils:generate-version`: Generate semantic version from git
- `task utils:generate-version-tag`: Generate full image tag with registry prefix

**Shared Configuration:**
Each service's Taskfile.yaml defines:
- `IMAGE_REPO`: DockerHub repository path (e.g., `sidpalas/demo-api-golang`)
- Version generation logic leveraging git describe and tags
- Build commands configured for the service's specific requirements

This consistency enables generic workflow logic that works across all services.

## Basic Single-Service Build

### Initial Workflow Structure

Let's start by building images for a single service—the Node.js API—then expand to all services using matrix strategies.

**Workflow File**: `.github/workflows/build-push.yaml`

```yaml
name: Build and Push Images

on:
  workflow_dispatch:

jobs:
  build-push:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@4.2.2  # v4.2.2 (compatible with Act/Node 20)
        with:
          fetch-depth: 0  # Full history needed for git describe
```

**Key configuration: fetch-depth: 0**

By default, GitHub Actions performs shallow clones (only the latest commit). Setting `fetch-depth: 0` clones the complete repository history including all tags. This is essential for `git describe`, which traverses commit history to find the nearest release tag and calculate the commit count.

**Why this matters:** Without full history, `git describe` fails or produces incorrect results, breaking our version generation logic.

### Installing Dependencies

```yaml
      - name: Setup dependencies
        uses: ./.github/actions/setup-dependencies
        with:
          install-task: 'true'
          task-version: v3.44.1
```

We're using the composite action created earlier to install Task. This demonstrates the reusability benefit of composite actions—the same setup step works across multiple workflows without code duplication.

**Creating the composite action:** `.github/actions/setup-dependencies/action.yaml`

```yaml
name: Setup Dependencies
description: Setup tools and dependencies

inputs:
  install-task:
    description: 'Whether to install Task runner'
    default: 'true'
  task-version:
    description: 'Version of Task to install'
    default: 'v3.44.1'

runs:
  using: composite
  steps:
    - name: Install Task
      if: inputs.install-task == 'true'
      uses: battila7/setup-binary@4.0.1
      with:
        binary: task
        version: ${{ inputs.task-version }}
        download-url: https://github.com/go-task/task/releases/download/${{ inputs.task-version }}/task_linux_amd64.tar.gz
```

**Note on input types:** GitHub Actions composite actions only support string inputs, not booleans. That's why we compare `inputs.install-task == 'true'` as a string comparison rather than treating it as a boolean.

### Docker Authentication

```yaml
      - name: Login to DockerHub
        if: env.ACT != 'true'
        uses: docker/login-action@9780b0c442fbb1117ed29e0efdff1e18412f7567  # v3.3.0
        with:
          username: ${{ vars.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}
```

**Conditional execution: `if: env.ACT != 'true'`**

When running workflows locally with Act, we typically don't want to push images to registries. The `ACT` environment variable is automatically set to `'true'` when running in Act, allowing us to skip authentication and push steps during local testing.

**Variables vs Secrets:**
- **DOCKERHUB_USERNAME**: Stored as a repository variable (not sensitive, visible in logs)
- **DOCKERHUB_TOKEN**: Stored as a secret (sensitive, masked in logs)

**Generating DockerHub access token:**
1. Visit hub.docker.com and sign in
2. Navigate to Account Settings → Security → Personal Access Tokens
3. Click "Generate New Token"
4. Name: `capstone-repo-token`
5. Permissions: Read & Write
6. Copy the generated token
7. Add to GitHub repository secrets as `DOCKERHUB_TOKEN`

### Multi-Architecture Support Setup

```yaml
      - name: Set up QEMU
        uses: docker/setup-qemu-action@49b3bc8e6bdd4a60e6116a5414239cba5943d3cf  # v3.2.0
        
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@988b5a0280414f521da01fcc63a27aeeb4b104db  # v3.7.1
```

**QEMU (Quick Emulator):**
Enables building images for architectures different from the runner's native architecture. On x86_64 runners, QEMU allows building ARM images by emulating ARM processors. This is slower than native builds but enables multi-architecture support from a single runner.

**Docker Buildx:**
Docker's extended build capabilities supporting:
- Multi-architecture image builds
- Advanced caching strategies
- Build secrets and SSH agent forwarding
- Parallel build stages
- Export to multiple formats

In production environments using remote builders (like Namespace), QEMU becomes unnecessary as you can schedule builds on native runners for each architecture. However, for GitHub-hosted runners, QEMU + Buildx provides a cost-effective multi-arch solution.

### Version Tag Generation (Placeholder)

```yaml
      - name: Generate image tag
        id: image-tag
        working-directory: services/node/api-node
        run: |
          # TODO: Generate actual version/tag
          echo "version=1.0.0-dev" >> $GITHUB_OUTPUT
          echo "image_tag=sidpalas/demo-api-node:1.0.0-dev" >> $GITHUB_OUTPUT
```

For now, we're using hardcoded placeholder values. We'll implement proper version generation shortly using Task commands and git metadata.

**Step outputs:** By writing to `$GITHUB_OUTPUT`, we make these values available to subsequent steps via `steps.image-tag.outputs.version` and `steps.image-tag.outputs.image_tag`.

### Building and Pushing the Image

```yaml
      - name: Build and push
        uses: docker/build-push-action@4f58ea79222b3b9dc2c8bbdd6debcef730109a75  # v6.9.0
        with:
          context: services/node/api-node
          push: ${{ env.ACT != 'true' }}
          tags: ${{ steps.image-tag.outputs.image_tag }}
          platforms: linux/amd64
          cache-from: type=gha
          cache-to: type=gha,mode=max
```

**Context:** Specifies the build context directory containing the Dockerfile and application files. Docker uses files in this directory during the build.

**Push:** Controlled by the ACT environment variable—push to registry when running in GitHub Actions, skip when running locally.

**Tags:** References the image tag generated in the previous step.

**Platforms:** Specifies target architecture(s). Using only `linux/amd64` for faster builds since GitHub-hosted runners are x86_64. For production, you might specify `linux/amd64,linux/arm64` to build both architectures.

**Cache configuration:**

- **cache-from: type=gha**: Restore build cache from GitHub Actions cache service
- **cache-to: type=gha,mode=max**: Save all build layers to cache, not just the final image layers

**Mode=max explained:** By default, Docker only caches layers from the final image. With `mode=max`, intermediate build stages are also cached. This is crucial for multi-stage Dockerfiles where build dependencies (compilers, build tools) are in separate stages from the final runtime image.

**Cache effectiveness:** First build is slow (no cache). Subsequent builds with unchanged dependencies complete in seconds rather than minutes, particularly impactful for Node.js/Python installs or Go compilations.

## Local Testing Configuration

### Act Setup for Build Workflow

Create `.github/workflows/build-push/taskfile.yaml`:

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
          --workflows .github/workflows/build-push.yaml \
          -e event.json \
          -s GITHUB_TOKEN="${GITHUB_TOKEN}"
    env:
      GITHUB_TOKEN:
        sh: gh auth token
```

**Event payload:** `.github/workflows/build-push/event.json`

```json
{
  "repository": {
    "default_branch": "main"
  }
}
```

Similar to the test workflow setup, we provide minimal event metadata required by the path filtering action.

### Running Locally

```bash
cd .github/workflows/build-push
task trigger-workflow
```

**Expected behavior:**
1. Checkout skipped (Act mounts local code by default)
2. Task installation
3. **Authentication skipped** (ACT environment variable is 'true')
4. QEMU and Buildx setup
5. Image tag generation
6. Image build
7. **Push skipped** (controlled by ACT variable)

**Verification:** The workflow should complete successfully without pushing anything to DockerHub. Logs confirm the image was built and tagged locally.

## Dynamic Version Generation

### Understanding Git Describe

The `git describe` command generates version strings from git history:

```bash
git describe --tags --always --dirty
```

**Output example:** `v1.0.9-47-g188f7a2`

**Components:**
- `v1.0.9`: Most recent tag
- `47`: Number of commits since that tag
- `g188f7a2`: Short commit hash (prefixed with 'g' for git)

**Flags explained:**
- `--tags`: Consider lightweight tags, not just annotated tags
- `--always`: If no tags exist, output commit hash
- `--dirty`: Append `-dirty` if working directory has uncommitted changes

This provides a unique, sortable version identifier that encodes distance from the last release.

### Task-Based Version Generation

In the shared utils Taskfile (`.github/utils/taskfile.yaml`):

```yaml
version: '3'

tasks:
  generate-version:
    desc: Generate semantic version from git describe
    cmds:
      - |
        VERSION=$(git describe --tags --always --dirty)
        # Remove 'v' prefix if present
        VERSION=${VERSION#v}
        echo "$VERSION"

  generate-version-extended:
    desc: Generate extended version with zero-padded commit count
    cmds:
      - |
        VERSION=$(git describe --tags --always --dirty)
        VERSION=${VERSION#v}
        # Extract parts: 1.0.9-47-g188f7a2 -> 1.0.9 0047 188f7a2
        if [[ $VERSION =~ ^([0-9]+\.[0-9]+\.[0-9]+)-([0-9]+)-g([a-f0-9]+)$ ]]; then
          BASE_VERSION="${BASH_REMATCH[1]}"
          COMMIT_COUNT="${BASH_REMATCH[2]}"
          COMMIT_HASH="${BASH_REMATCH[3]}"
          # Zero-pad commit count to 4 digits
          PADDED_COUNT=$(printf "%04d" $COMMIT_COUNT)
          echo "${BASE_VERSION}-${PADDED_COUNT}-${COMMIT_HASH}"
        else
          echo "$VERSION"
        fi

  generate-version-tag:
    desc: Generate full image tag with registry prefix
    cmds:
      - |
        VERSION=$(task generate-version)
        IMAGE_TAG="${IMAGE_REPO}:${VERSION}"
        echo "$IMAGE_TAG"
```

Each service imports this utils Taskfile and defines its `IMAGE_REPO` variable, allowing these shared tasks to generate appropriate tags.

### Implementing in Workflow

Replace the placeholder version generation:

```yaml
      - name: Generate image tag
        id: image-tag
        working-directory: ${{ matrix.service }}
        env:
          ENVIRONMENT: ${{ needs.filter.outputs.environment }}
        run: |
          if [ "$ENVIRONMENT" = "production" ]; then
            VERSION=$(task utils:generate-version)
            IMAGE_TAG=$(task utils:generate-version-tag)
          else
            VERSION=$(task utils:generate-version-extended)
            IMAGE_TAG=$(task utils:generate-version-tag-extended)
          fi
          echo "version=$VERSION" >> $GITHUB_OUTPUT
          echo "image_tag=$IMAGE_TAG" >> $GITHUB_OUTPUT
```

**Environment-based logic:**
- **Production**: Use semantic version only (`1.0.9`)
- **Non-production**: Use extended version with commit count (`1.0.9-0047-188f7a2`)

This differentiation ensures production images have clean version numbers while development/staging images include additional metadata for traceability.

## Expanding to Multiple Services with Matrix

### Adding Filter Job

Like the test workflow, we need intelligent filtering to avoid building unchanged services:

```yaml
jobs:
  filter:
    runs-on: ubuntu-latest
    outputs:
      services: ${{ steps.set-output.outputs.services }}
      environment: ${{ steps.set-output.outputs.environment }}
    steps:
      - name: Checkout code
        uses: actions/checkout@4.2.2
        
      - name: Filter changed services
        id: filter
        if: github.ref_type != 'tag'
        uses: dorny/paths-filter@de90cc6fb38fc0963ad72b210f1f284cd68cea36  # v3.0.2
        with:
          token: ${{ secrets.GITHUB_TOKEN }}
          filters: .github/utils/file-filters.yaml
```

**Conditional filtering:** The path filter action only runs when the triggering event is NOT a tag push. For tag-based deployments, we'll use different logic.

### Multi-Environment Output Logic

```yaml
      - name: Set output
        id: set-output
        run: |
          if [ "${{ github.ref_type }}" = "tag" ]; then
            # Extract service from tag: services/go/api-golang@1.0.9 -> services/go/api-golang
            TAG_NAME="${{ github.ref_name }}"
            SERVICE="${TAG_NAME%@*}"
            echo "services=[\"$SERVICE\"]" >> $GITHUB_OUTPUT
            echo "environment=production" >> $GITHUB_OUTPUT
          elif [ "${{ github.event_name }}" = "workflow_dispatch" ]; then
            # Manual trigger: use input values
            echo "services=[\"${{ github.event.inputs.service }}\"]" >> $GITHUB_OUTPUT
            echo "environment=development" >> $GITHUB_OUTPUT
          else
            # Standard push to main: use path filter results
            echo "services=${{ steps.filter.outputs.changes }}" >> $GITHUB_OUTPUT
            echo "environment=staging" >> $GITHUB_OUTPUT
          fi
```

**Three triggering scenarios:**

1. **Tag push (production):**
   - Parse tag name to extract service path
   - Tag format: `services/go/api-golang@1.0.9`
   - Bash expansion `${TAG_NAME%@*}` removes everything after @ symbol
   - Single-service matrix: `["services/go/api-golang"]`
   - Environment: `production`

2. **Workflow dispatch (manual):**
   - Use service specified in workflow input
   - Environment: `development` (arbitrary choice for manual builds)

3. **Push to main (automated staging):**
   - Use path filter results (potentially multiple services)
   - Environment: `staging`

**Output format:** Services must be a JSON array even for single values, ensuring compatibility with matrix strategy.

### Workflow Dispatch Inputs

```yaml
on:
  workflow_dispatch:
    inputs:
      service:
        description: 'Service to build'
        required: true
        type: choice
        options:
          - services/node/api-node
          - services/go/api-golang
          - services/react/client-react
          - services/python/load-generator
          - services/other/migrator
      version:
        description: 'Version tag (optional, defaults to git describe)'
        required: false
```

**Type: choice** creates a dropdown in the GitHub Actions UI, preventing typos and providing discoverability. Users select from predefined service paths rather than typing them manually.

**Optional version input:** Allows overriding automatic version generation for special cases (hotfixes, re-releases, etc.). If not provided, workflow uses git describe as normal.

### Matrix Build Job

```yaml
  build-push:
    needs: filter
    if: |
      needs.filter.outputs.services != '[]' &&
      needs.filter.outputs.services != ''
    runs-on: ubuntu-latest
    strategy:
      fail-fast: false
      matrix:
        service: ${{ fromJSON(needs.filter.outputs.services) }}
    steps:
      # ... existing steps using matrix.service
```

**Empty matrix protection:**

The conditional `if` clause prevents the job from running when no services changed. Without this, an empty JSON array `[]` or empty string would cause the job to execute once with an empty service value, leading to cryptic errors.

**Why both checks?** The paths-filter action may return either `[]` or `''` depending on circumstances, so we guard against both.

### Complete Matrix Configuration

```yaml
      - name: Build and push
        uses: docker/build-push-action@4f58ea79222b3b9dc2c8bbdd6debcef730109a75
        with:
          context: ${{ matrix.service }}
          push: ${{ env.ACT != 'true' }}
          tags: ${{ steps.image-tag.outputs.image_tag }}
          platforms: linux/amd64
          cache-from: type=gha
          cache-to: type=gha,mode=max
```

By using `matrix.service` as the context, the same step definition builds images for all filtered services in parallel. Each matrix job instance processes one service completely independently.

## Production Triggers and Tag-Based Deploys

### Trigger Configuration

```yaml
on:
  workflow_dispatch:  # Manual triggering
  push:
    branches:
      - main
    paths:
      - 'services/**'  # Only trigger if services changed
  push:
    tags:
      - '*@[0-9]+.[0-9]+.[0-9]+'  # Match tags like service-name@1.0.0
```

**Path filtering on push:** The `paths` filter ensures the workflow only runs when relevant files change. Updating documentation, workflow files in other directories, or configuration won't trigger unnecessary image builds.

**Tag pattern matching:** The glob pattern `*@[0-9]+.[0-9]+.[0-9]+` matches our release tag convention. Any tag containing an @ symbol followed by a semantic version triggers the workflow.

**Examples:**
- ✅ `services/go/api-golang@1.0.9` - matches
- ✅ `services/node/api-node@2.1.0` - matches
- ❌ `v1.0.9` - doesn't match (no @)
- ❌ `services/go/api-golang@1.0` - doesn't match (incomplete version)

### Testing Tag-Based Builds Locally

Create `.github/workflows/build-push/event-tag.json`:

```json
{
  "repository": {
    "default_branch": "main"
  },
  "ref": "refs/tags/services/go/api-golang@1.0.9",
  "ref_type": "tag"
}
```

Add a task in `.github/workflows/build-push/taskfile.yaml`:

```yaml
  trigger-workflow-tag:
    cmds:
      - |
        act push \
          --container-architecture linux/amd64 \
          -P ubuntu-latest=catthehacker/ubuntu:act-latest \
          --directory ../../.. \
          --workflows .github/workflows/build-push.yaml \
          -e event-tag.json \
          -s GITHUB_TOKEN="${GITHUB_TOKEN}" \
          --no-skip-checkout
```

**Key difference: `--no-skip-checkout`**

By default, Act mounts your local code directory into the container, skipping the checkout step. For tag-based testing, we want Act to perform a real checkout at the specified tag, ensuring the workflow processes the correct version of code.

**Testing:**

```bash
task trigger-workflow-tag
```

**Expected output:**
1. Checkout at tag `services/go/api-golang@1.0.9`
2. Filter job extracts service path: `services/go/api-golang`
3. Environment set to `production`
4. Single matrix job for Golang API
5. Version generated without commit count (clean semantic version)

This validates the tag-parsing logic without pushing to production registries.

## File Filters Configuration

Reuse the same filters from the test workflow: `.github/utils/file-filters.yaml`

```yaml
api-golang: &api-golang
  - services/go/api-golang/**
  - services/other/migrator/**  # Migrations affect API
  - .github/workflows/build-push.yaml  # Workflow changes should trigger rebuild

api-node:
  - services/node/api-node/**
  - .github/workflows/build-push.yaml

client-react:
  - services/react/client-react/**
  - .github/workflows/build-push.yaml

load-generator:
  - services/python/load-generator/**
  - .github/workflows/build-push.yaml

migrator:
  *api-golang  # Shares same trigger paths as api-golang
```

**Shared patterns:** The YAML anchor for `api-golang` is reused for `migrator` since changes to migrations require rebuilding both the migrator image and the API that uses those migrations. This ensures consistency when database schemas change.

## Complete Production Workflow

Bringing all components together:

```yaml
name: Build and Push Images

on:
  workflow_dispatch:
    inputs:
      service:
        description: 'Service to build'
        required: true
        type: choice
        options:
          - services/node/api-node
          - services/go/api-golang
          - services/react/client-react
          - services/python/load-generator
          - services/other/migrator
      version:
        description: 'Version tag (optional)'
        required: false
  push:
    branches: [main]
    paths: ['services/**']
  push:
    tags: ['*@[0-9]+.[0-9]+.[0-9]+']

jobs:
  filter:
    runs-on: ubuntu-latest
    outputs:
      services: ${{ steps.set-output.outputs.services }}
      environment: ${{ steps.set-output.outputs.environment }}
    steps:
      - name: Checkout code
        uses: actions/checkout@4.2.2
        
      - name: Filter changed services
        id: filter
        if: github.ref_type != 'tag'
        uses: dorny/paths-filter@de90cc6fb38fc0963ad72b210f1f284cd68cea36
        with:
          token: ${{ secrets.GITHUB_TOKEN }}
          filters: .github/utils/file-filters.yaml
          
      - name: Set output
        id: set-output
        run: |
          if [ "${{ github.ref_type }}" = "tag" ]; then
            TAG_NAME="${{ github.ref_name }}"
            SERVICE="${TAG_NAME%@*}"
            echo "services=[\"$SERVICE\"]" >> $GITHUB_OUTPUT
            echo "environment=production" >> $GITHUB_OUTPUT
          elif [ "${{ github.event_name }}" = "workflow_dispatch" ]; then
            echo "services=[\"${{ github.event.inputs.service }}\"]" >> $GITHUB_OUTPUT
            echo "environment=development" >> $GITHUB_OUTPUT
          else
            echo "services=${{ steps.filter.outputs.changes }}" >> $GITHUB_OUTPUT
            echo "environment=staging" >> $GITHUB_OUTPUT
          fi

  build-push:
    needs: filter
    if: |
      needs.filter.outputs.services != '[]' &&
      needs.filter.outputs.services != ''
    runs-on: ubuntu-latest
    strategy:
      fail-fast: false
      matrix:
        service: ${{ fromJSON(needs.filter.outputs.services) }}
    steps:
      - name: Checkout code
        uses: actions/checkout@4.2.2
        with:
          fetch-depth: 0
          
      - name: Setup dependencies
        uses: ./.github/actions/setup-dependencies
        
      - name: Login to DockerHub
        if: env.ACT != 'true'
        uses: docker/login-action@9780b0c442fbb1117ed29e0efdff1e18412f7567
        with:
          username: ${{ vars.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}
          
      - name: Set up QEMU
        uses: docker/setup-qemu-action@49b3bc8e6bdd4a60e6116a5414239cba5943d3cf
        
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@988b5a0280414f521da01fcc63a27aeeb4b104db
          
      - name: Generate image tag
        id: image-tag
        working-directory: ${{ matrix.service }}
        env:
          ENVIRONMENT: ${{ needs.filter.outputs.environment }}
        run: |
          if [ "$ENVIRONMENT" = "production" ]; then
            VERSION=$(task utils:generate-version)
            IMAGE_TAG=$(task utils:generate-version-tag)
          else
            VERSION=$(task utils:generate-version-extended)
            IMAGE_TAG=$(task utils:generate-version-tag-extended)
          fi
          echo "version=$VERSION" >> $GITHUB_OUTPUT
          echo "image_tag=$IMAGE_TAG" >> $GITHUB_OUTPUT
          
      - name: Build and push
        uses: docker/build-push-action@4f58ea79222b3b9dc2c8bbdd6debcef730109a75
        with:
          context: ${{ matrix.service }}
          push: ${{ env.ACT != 'true' }}
          tags: ${{ steps.image-tag.outputs.image_tag }}
          platforms: linux/amd64
          cache-from: type=gha
          cache-to: type=gha,mode=max
```

## Key Takeaways

### Architecture Patterns

1. **Environment-aware versioning**: Production gets clean semantic versions, non-production includes commit metadata
2. **Flexible triggering**: Support automated (push), manual (workflow_dispatch), and release (tag) workflows
3. **Intelligent filtering**: Build only changed services using path analysis
4. **Local development**: Act enables full workflow testing without pushing to registries
5. **Composite actions**: Shared setup logic reduces duplication across workflows

### Version Management Strategy

**Git describe advantages:**
- Automatic version generation from repository state
- No manual version file maintenance
- Sortable, unique version identifiers
- Traceable to specific commits

**Extended versioning benefits:**
- Clear indication of distance from last release
- Alphabetical sorting works correctly (via zero-padding)
- Commit hash enables quick repository navigation
- Different formats for production vs non-production distinguish environment

### Build Optimization

**Caching strategy:**
- GitHub Actions cache stores Docker layers between runs
- `mode=max` caches intermediate stages, crucial for multi-stage builds
- First build slow (no cache), subsequent builds fast (cached layers)
- Cache invalidates automatically when relevant files change

**Parallel execution:**
- Matrix strategy builds multiple services simultaneously
- Each service builds independently (no cross-dependencies)
- Failures in one service don't block others (fail-fast: false)
- Total workflow time equals slowest individual build, not sum of all builds

### Production Readiness

1. **Security**: DockerHub credentials stored as secrets, masked in logs
2. **Traceability**: Every image tagged with exact git commit
3. **Rollback capability**: Clean versions enable easy rollback to any release
4. **Audit trail**: Tags and workflow runs provide complete deployment history
5. **Environment isolation**: Different tagging for prod vs non-prod prevents confusion

## Conclusion

This chapter demonstrated building a production-grade container image pipeline that handles complex requirements: multi-service builds, environment-specific versioning, intelligent filtering, and flexible deployment models. By combining matrix strategies, conditional logic, git-based versioning, and caching optimizations, we created a system that's both efficient and maintainable.

The build-push workflow works in concert with the test workflow from the previous chapter, forming the foundation of our CI/CD pipeline. Tests validate code quality, image builds package validated code into deployable artifacts, and—as we'll see in the next chapter—GitOps workflows automate deployment of those artifacts to Kubernetes clusters.

The patterns established here—Task-based abstractions, environment-aware configuration, local testing with Act, composite actions for reusability—continue to pay dividends as we build additional workflows. A well-architected initial workflow makes adding new capabilities incrementally straightforward rather than requiring wholesale rewrites.
