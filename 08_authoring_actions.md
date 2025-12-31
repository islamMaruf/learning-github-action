# Chapter 8: Authoring Actions - Building Custom Reusable Logic

## Overview

In Chapter 7, you learned how to consume actions from the GitHub Marketplace. But what happens when the action you need doesn't exist? Or when you have organization-specific logic that needs to be reused across multiple workflows?

This chapter explores **authoring your own actions**—creating custom, reusable units of logic that can be shared across workflows, repositories, and even organizations.

GitHub Actions provides **four mechanisms** for writing and reusing logic:
1. **Composite actions** - Bundle steps into reusable components
2. **Reusable workflows** - Share entire workflow definitions
3. **JavaScript/TypeScript actions** - Full-featured actions with GitHub's native runtime
4. **Container actions** - Actions in any language, packaged as Docker containers

By the end of this chapter, you'll understand:
- When to use each authoring approach
- How to structure and build custom actions
- Input/output mechanisms for passing data
- Publishing strategies (public marketplace vs. private repos)
- Decision frameworks for choosing the right tool

## Composite Actions: The Easiest Starting Point

**Definition**: Composite actions are the **simplest way** to bundle multiple steps into a single reusable action.

**Use case**: You have a sequence of steps you're copying across workflows and want to centralize the logic.

### Anatomy of a Composite Action

**Required file**: `action.yml` (or `action.yaml`)

**Structure**:
```yaml
name: 'My Composite Action'
description: 'Description of what this action does'
author: 'Your Name'

inputs:
  who-to-greet:
    description: 'Person to greet'
    required: true
    default: 'World'

outputs:
  random-number:
    description: 'A random number'
    value: ${{ steps.random-step.outputs.random-number }}

runs:
  using: 'composite'
  steps:
    - name: Greet someone
      run: echo "Hello, ${{ inputs.who-to-greet }}!"
      shell: bash
    
    - name: Generate random number
      id: random-step
      run: echo "random-number=$RANDOM" >> $GITHUB_OUTPUT
      shell: bash
```

**Key sections**:
- **`inputs:`** - Define parameters the action accepts
- **`outputs:`** - Define values the action produces
- **`runs.using: composite`** - Declares this as a composite action
- **`runs.steps:`** - The actual steps to execute

### Creating a Composite Action

**Step 1: Create directory structure**

```
.github/
└── actions/
    └── hello-world/
        └── action.yml
```

**Convention**: Store actions in `.github/actions/` directory for discoverability.

**Step 2: Define action.yml**

```yaml
name: 'Hello World Composite'
description: 'Greets someone and generates a random number'

inputs:
  who-to-greet:
    description: 'Who to greet'
    required: true

outputs:
  random-number:
    description: 'Random number generated'
    value: ${{ steps.generate-random.outputs.random-number }}

runs:
  using: 'composite'
  steps:
    - name: Greet
      run: echo "Hello, ${{ inputs.who-to-greet }}!"
      shell: bash
    
    - name: Generate random
      id: generate-random
      run: echo "random-number=$RANDOM" >> $GITHUB_OUTPUT
      shell: bash
```

**Step 3: Use in workflow**

```yaml
name: Use Composite Action

on: workflow_dispatch

jobs:
  job1:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4  # REQUIRED to access action code
      
      - name: Call composite action
        id: my-action
        uses: ./.github/actions/hello-world
        with:
          who-to-greet: 'from Job 1'
      
      - name: Show output
        run: echo "Random: ${{ steps.my-action.outputs.random-number }}"

  job2:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Call same action with different input
        uses: ./.github/actions/hello-world
        with:
          who-to-greet: 'from Job 2'
```

**Critical requirement**: Must run `actions/checkout` **before** using composite action, otherwise the code won't be available on the runner.

### Composite Action Features

**Inputs**:
- Access via `${{ inputs.input-name }}`
- Can have default values
- Can be required or optional
- Type-checked at workflow syntax level

**Outputs**:
- Must reference output from a specific step (via `steps.step-id.outputs.output-name`)
- Available to calling workflow via `steps.action-step-id.outputs.output-name`

**Limitations**:
- Cannot use `secrets` context directly (must be passed as inputs)
- Cannot define environment variables at action level
- All steps must specify `shell` (bash, pwsh, python, etc.)

### When to Use Composite Actions

**Good fit**:
- Reusing 3-10 steps across multiple workflows
- Organization-specific setup sequences (install dependencies, configure tools)
- Common validation patterns (lint, test, security scan)
- Pre/post job setup routines

**Not ideal**:
- Single-step operations (just use inline `run:`)
- Complex logic requiring external dependencies (use JavaScript/container actions)
- Need for fine-grained control over execution environment

## Reusable Workflows: Standardizing Entire Workflows

**Definition**: Reusable workflows allow you to call an entire workflow as a **job** within another workflow.

**Key distinction**: While composite actions are **steps**, reusable workflows are **jobs**.

### Anatomy of a Reusable Workflow

**Source workflow** (the one being called):

```yaml
name: Reusable Deploy Workflow

on:
  workflow_call:  # This makes it reusable
    inputs:
      environment:
        description: 'Environment to deploy to'
        required: true
        type: string
    secrets:
      deploy-key:
        description: 'Deployment key'
        required: true

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: ${{ inputs.environment }}
    steps:
      - uses: actions/checkout@v4
      - name: Deploy
        run: |
          echo "Deploying to ${{ inputs.environment }}"
          echo "Using key: ${{ secrets.deploy-key }}"
```

**Caller workflow** (the one calling it):

```yaml
name: Call Reusable Workflow

on: workflow_dispatch

jobs:
  deploy-staging:
    uses: ./.github/workflows/deploy-reusable.yml
    with:
      environment: staging
    secrets:
      deploy-key: ${{ secrets.STAGING_DEPLOY_KEY }}
  
  deploy-production:
    uses: ./.github/workflows/deploy-reusable.yml
    with:
      environment: production
    secrets:
      deploy-key: ${{ secrets.PRODUCTION_DEPLOY_KEY }}
```

### Key Features of Reusable Workflows

#### The `workflow_call` Trigger

**Purpose**: Marks a workflow as reusable (callable from other workflows).

**Syntax**:
```yaml
on:
  workflow_call:
    inputs:
      # ... input definitions
    secrets:
      # ... secret definitions
```

**Without this trigger**: Workflow cannot be called from another workflow.

#### Inputs for Reusable Workflows

**Definition**:
```yaml
on:
  workflow_call:
    inputs:
      environment-name:
        description: 'Target environment'
        required: true
        type: string  # string, boolean, number
      debug-mode:
        description: 'Enable debug logging'
        required: false
        type: boolean
        default: false
```

**Types**: `string`, `boolean`, `number`

**Accessing inputs**:
```yaml
steps:
  - run: echo "Environment: ${{ inputs.environment-name }}"
```

#### Secrets for Reusable Workflows

**Definition** (in source workflow):
```yaml
on:
  workflow_call:
    secrets:
      api-key:
        description: 'API key for service'
        required: true
```

**Passing secrets** (from caller):
```yaml
jobs:
  call-workflow:
    uses: ./.github/workflows/reusable.yml
    secrets:
      api-key: ${{ secrets.MY_API_KEY }}
```

**Important limitation**: Only **repository-level** and **organization-level** secrets can be passed explicitly.

#### Environment Secret Inheritance

**Problem**: Environment secrets cannot be passed like repository secrets.

**Solution**: Use `inherit` keyword.

**Example**:
```yaml
jobs:
  deploy-staging:
    uses: ./.github/workflows/deploy.yml
    with:
      environment: staging
    secrets: inherit  # Inherits ALL secrets from staging environment
```

**Without `secrets: inherit`**: Environment secrets are NOT available to reusable workflow.

### Referencing Reusable Workflows

**Option 1: Relative path** (same repository):
```yaml
jobs:
  call-local:
    uses: ./.github/workflows/reusable.yml
```

**Behavior**: Uses workflow definition from the **current commit**.

**Use case**: Development/testing, workflows that evolve together.

**Option 2: Repository reference** (same or different repository):
```yaml
jobs:
  call-versioned:
    uses: organization/repo/.github/workflows/reusable.yml@v1.2.3
```

**Behavior**: Uses workflow definition from **specified commit/tag/branch**.

**Use case**: Production, external shared workflows, version pinning.

### When to Use Reusable Workflows

**Good fit**:
- Enforcing standardized deployment processes
- Organization-wide CI/CD templates
- Multi-environment workflows (staging, production)
- Workflows that must execute in specific order with specific configuration

**Not ideal**:
- Simple step sequences (use composite actions)
- Need to mix steps from reusable logic with custom steps
- Workflows that differ significantly between uses

## Composite Action vs. Reusable Workflow: Decision Framework

Use this flowchart to decide:

```
Do you need to reuse logic across multiple repos/workflows?
├─ NO → Use standard workflow (no abstraction needed)
└─ YES → Do you need to enforce standardization across entire workflow?
    ├─ YES → Use reusable workflow
    │        (Entire job definition is controlled centrally)
    └─ NO → Use composite action
             (Flexible mix of shared steps and custom steps)
```

**Example scenarios**:

**Reusable workflow**:
- "All deployments must run these exact steps in this exact order"
- "Every release workflow must include approval gates and rollback procedures"

**Composite action**:
- "Many workflows need to install Node.js and cache npm modules"
- "Several workflows lint code, but with different subsequent steps"

## JavaScript and TypeScript Actions: Native GitHub Integration

**Definition**: Actions written in JavaScript or TypeScript that run directly on the Node.js runtime provided by GitHub runners.

**Why use them**: 
- GitHub's **most natively supported** action type
- Official `@actions/core` npm package for GitHub integration
- Most marketplace actions use this approach
- Fast execution (no container build overhead)

### Anatomy of a JavaScript Action

**Required files**:
- `action.yml` - Action metadata
- `index.js` - Entrypoint JavaScript file (or compiled from TypeScript)
- `package.json` - npm dependencies
- `node_modules/` or bundled output

**action.yml**:
```yaml
name: 'JavaScript Action'
description: 'Example JS action'

inputs:
  name:
    description: 'Name to greet'
    required: true

outputs:
  greeting:
    description: 'The greeting message'

runs:
  using: 'node20'  # Node.js runtime version
  main: 'dist/index.js'  # Entrypoint file
```

**Key differences from composite action**:
- `runs.using:` specifies Node.js version (`node20`, `node16`)
- `runs.main:` points to JavaScript file
- No `steps:` section

### The @actions/core Package

**Installation**:
```bash
npm install @actions/core
```

**Purpose**: Official GitHub package providing utilities for:
- Reading inputs
- Setting outputs
- Logging messages
- Creating annotations
- Failing workflow
- Accessing environment variables

**Common methods**:

```javascript
const core = require('@actions/core');

// Read inputs
const name = core.getInput('name', { required: true });

// Set outputs
core.setOutput('greeting', `Hello, ${name}!`);

// Logging
core.debug('Debug message');
core.info('Info message');
core.warning('Warning message');
core.error('Error message');

// Fail the action
core.setFailed('Something went wrong!');

// Annotations
core.notice('This is a notice');
core.error('Missing semicolon', {
  file: 'app.js',
  startLine: 42,
  startColumn: 10
});

// Group logs
const group = core.startGroup('Build output');
// ... build logs
core.endGroup();
```

### Building a TypeScript Action

**Why TypeScript over JavaScript**:
- Type safety reduces bugs
- Better IDE support and autocomplete
- Compile-time error checking
- Industry standard for GitHub Actions

**Required compilation**: Must transpile TypeScript → JavaScript before use.

**GitHub provides templates**:
- TypeScript action template: github.com/actions/typescript-action
- JavaScript action template: github.com/actions/javascript-action

### TypeScript Action Structure

**File layout**:
```
action-name/
├── action.yml
├── src/
│   ├── main.ts
│   └── index.ts
├── dist/
│   ├── index.js          # Compiled output
│   └── index.js.map
├── package.json
├── tsconfig.json
├── rollup.config.js      # Bundler configuration
└── node_modules/
```

**src/main.ts** (logic):
```typescript
import * as core from '@actions/core';

export async function run(): Promise<void> {
  try {
    const name = core.getInput('name', { required: true });
    
    core.info(`Hello, ${name}!`);
    
    const greeting = `Hello, ${name}!`;
    core.setOutput('greeting', greeting);
    
  } catch (error) {
    core.setFailed((error as Error).message);
  }
}
```

**src/index.ts** (entrypoint):
```typescript
import { run } from './main';

run();
```

**Build process**:
```bash
# Install dependencies
npm install

# Compile TypeScript + bundle dependencies
npm run bundle
```

**package.json** scripts:
```json
{
  "scripts": {
    "build": "tsc",
    "bundle": "rollup -c",
    "package": "npm run build && npm run bundle"
  }
}
```

**Bundler (rollup)**: Combines TypeScript output and node_modules into single `dist/index.js` file.

**Why bundle**: 
- Single file to maintain in repo
- No need to commit node_modules (large, messy)
- Faster action startup (no dependency resolution)

### Using a JavaScript Action

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Call JavaScript action
        id: js-action
        uses: ./.github/actions/my-js-action
        with:
          name: 'GitHub Actions User'
      
      - name: Show output
        run: echo "${{ steps.js-action.outputs.greeting }}"
```

**No special requirements**: Action code already bundled as JavaScript.

### When to Use JavaScript Actions

**Good fit**:
- Complex logic requiring control flow, data structures
- Need GitHub API integration (issues, PRs, releases)
- Multiple external npm dependencies
- Organization-standard tooling
- Potential for publishing to marketplace

**Not ideal**:
- Simple shell commands (use composite actions)
- Need language other than JavaScript/TypeScript
- Heavy dependencies (container actions may be better)

## Container Actions: Any Language, Dockerized

**Definition**: Actions that run inside Docker containers, allowing you to use **any programming language**.

**Use case**: When your team's expertise is in Python, Go, Rust, etc., and JavaScript would be awkward.

### Anatomy of a Container Action

**Required files**:
- `action.yml` - Action metadata
- `Dockerfile` - Container image definition
- `entrypoint.py` (or other language) - Action logic

**action.yml**:
```yaml
name: 'Python Container Action'
description: 'Example container action'

inputs:
  who-to-greet:
    description: 'Who to greet'
    required: true

outputs:
  random-number:
    description: 'Random number'

runs:
  using: 'docker'
  image: 'Dockerfile'  # Or 'docker://user/image:tag'
```

**Key field**: `runs.using: docker`

**Image options**:
1. **`image: 'Dockerfile'`** - Build from Dockerfile in repository
2. **`image: 'docker://owner/repo:tag'`** - Pull pre-built image from registry

### Building a Python Container Action

**Dockerfile**:
```dockerfile
FROM python:3.11-slim

# Copy action code
COPY entrypoint.py /entrypoint.py

# Install dependencies (if needed)
# COPY requirements.txt /requirements.txt
# RUN pip install --no-cache-dir -r /requirements.txt

# Make entrypoint executable
RUN chmod +x /entrypoint.py

# Set entrypoint
ENTRYPOINT ["/entrypoint.py"]
```

**entrypoint.py**:
```python
#!/usr/bin/env python3
import os
import sys
import random

# Read input from environment variable
# GitHub formats as: INPUT_<NAME-IN-UPPERCASE>
who_to_greet = os.environ.get('INPUT_WHO-TO-GREET', 'World')

# Or read from sys.argv
# who_to_greet = sys.argv[1] if len(sys.argv) > 1 else 'World'

# Print notice message (workflow command)
print(f'::notice::Hello, {who_to_greet}!')

# Set output (write to GITHUB_OUTPUT file)
random_number = random.randint(1, 10000)
output_file = os.environ.get('GITHUB_OUTPUT')
if output_file:
    with open(output_file, 'a') as f:
        f.write(f'random-number={random_number}\n')

print(f'Generated random number: {random_number}')
```

### Accessing Inputs in Container Actions

**Option 1: Environment variables** (recommended):
```python
import os

# Input name: who-to-greet
# Environment variable: INPUT_WHO-TO-GREET
value = os.environ.get('INPUT_WHO-TO-GREET')
```

**Pattern**: `INPUT_<NAME-IN-UPPERCASE-WITH-HYPHENS>`

**Option 2: Command-line arguments**:
```python
import sys

value = sys.argv[1]  # First argument
```

**action.yml configuration**:
```yaml
runs:
  using: 'docker'
  image: 'Dockerfile'
  args:
    - ${{ inputs.who-to-greet }}
```

### Workflow Commands in Container Actions

**Problem**: No official GitHub package for non-JavaScript languages.

**Solution**: Use **workflow commands** - special strings printed to stdout that GitHub Actions recognizes.

**Format**: `::command parameter1={value1},parameter2={value2}::{message}`

**Common commands**:

```python
# Notice annotation
print('::notice::This is a notice message')

# Warning annotation
print('::warning file=app.py,line=10::Missing type hint')

# Error annotation
print('::error file=app.py,line=15::Syntax error')

# Set output (DEPRECATED - use GITHUB_OUTPUT file instead)
# print(f'::set-output name=result::{value}')

# Add to PATH
print(f'::add-path::/usr/local/custom/bin')

# Set environment variable
print(f'::set-env name=API_KEY::secret-value')
```

**Setting outputs** (modern approach):
```python
import os

output_file = os.environ['GITHUB_OUTPUT']
with open(output_file, 'a') as f:
    f.write(f'output-name={value}\n')
```

### Pre-built vs. Dockerfile Image

**Option 1: Build from Dockerfile** (development):
```yaml
runs:
  using: 'docker'
  image: 'Dockerfile'
```

**Behavior**: Builds container image **every workflow run**.

**Advantages**:
- Always uses latest code
- No separate build/publish step
- Great for testing

**Disadvantages**:
- **Slow** (build time adds 30s-2min per run)
- Wasted compute rebuilding unchanged images
- Not suitable for production

**Option 2: Pre-built image** (production):
```yaml
runs:
  using: 'docker'
  image: 'docker://ghcr.io/user/action:v1.2.3'
```

**Behavior**: Pulls existing image from registry.

**Advantages**:
- **Fast** (pull vs. build)
- Production-ready
- Versioned and immutable

**Disadvantages**:
- Requires separate build/push workflow
- More complex release process

**Build and push workflow**:
```yaml
name: Build Action Image

on:
  release:
    types: [published]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      
      - uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ghcr.io/${{ github.repository }}:${{ github.ref_name }}
```

### When to Use Container Actions

**Good fit**:
- Team expertise in non-JavaScript language
- Existing codebase in Python, Go, Rust, etc.
- Dependencies with better ecosystem in another language
- Need specific system libraries or tools

**Not ideal**:
- Simple logic (overhead not worth it)
- Speed-critical workflows (container startup adds latency)
- Actions that run frequently (build overhead compounds)

## Workflow Commands Reference

**Definition**: Special strings that, when printed to stdout, control GitHub Actions behavior.

**Format**: `::command parameter1=value1,parameter2=value2::message`

### Complete Workflow Commands

| Command | Purpose | Example |
|---------|---------|---------|
| `::debug::` | Debug message (only visible with debug logging enabled) | `::debug::Variable value: $VAR` |
| `::notice::` | Notice annotation (info level) | `::notice::Build completed` |
| `::warning::` | Warning annotation | `::warning file=app.js,line=10::Deprecated API` |
| `::error::` | Error annotation (doesn't fail workflow) | `::error file=app.js,line=15::Syntax error` |
| `::group::/::endgroup::` | Collapsible log group | `::group::Build logs` ... `::endgroup::` |
| `::add-mask::` | Mask value in logs | `::add-mask::$SECRET_VALUE` |
| `::stop-commands::` | Temporarily disable command processing | `::stop-commands::token` ... `::token::` |

### Modern Output/Environment Setting

**Setting outputs** (write to file):
```bash
echo "output-name=value" >> $GITHUB_OUTPUT
```

**Setting environment variables** (write to file):
```bash
echo "VAR_NAME=value" >> $GITHUB_ENV
```

**Adding to PATH** (write to file):
```bash
echo "/usr/local/custom/bin" >> $GITHUB_PATH
```

**Why file-based**: More secure, prevents log injection attacks.

## Decision Framework: Choosing the Right Tool

Here's a comprehensive flowchart for deciding which approach to use:

```
Was I able to find a reputable public action in the marketplace?
├─ YES → Use marketplace action
└─ NO → Is the logic simple (1-2 shell commands)?
    ├─ YES → Inline shell script in workflow
    └─ NO → Is bash sufficient, or do I need more expressivity?
        ├─ BASH SUFFICIENT → External bash file (makefile, taskfile)
        └─ NEED MORE → Do I need external dependencies?
            ├─ NO → In-repo Python/Node script called from workflow
            └─ YES → JavaScript/TypeScript action or Container action?
                ├─ JAVASCRIPT → Use JavaScript/TypeScript action
                │              (better GitHub integration)
                └─ OTHER LANGUAGE → Use container action
                                   (flexibility for any language)
```

### Practical Recommendations

**Most workflows**: Combination of:
1. Marketplace actions for common tasks
2. Bash scripts in makefiles/taskfiles for custom logic
3. Occasional composite actions for repeated step sequences

**When to invest in full actions**:
- Logic used across 10+ workflows
- Organization-wide standardization needed
- Complex interactions with GitHub API
- Publishing to marketplace for community

## Publishing Actions

### Publishing to Marketplace

**Requirements**:
1. Public repository
2. `action.yml` in repository root
3. Valid action metadata (name, description, author)
4. README.md with usage documentation

**Process**:
1. Create action in public repository
2. Push `action.yml` to repository root
3. Create GitHub release
4. Check "Publish this action to GitHub Marketplace" in release UI
5. Select category for action
6. Publish release

**Release interface**: When drafting release, GitHub detects `action.yml` and offers marketplace publication checkbox.

**Marketplace listing**: Action appears at `github.com/marketplace/actions/<action-name>`

### Using Private Actions

**Same repository**:
```yaml
steps:
  - uses: actions/checkout@v4  # Required!
  - uses: ./.github/actions/my-action
```

**Different private repository** (same organization):

**Step 1: Grant access** (in source repo):
1. Settings → Actions → General
2. Scroll to "Access"
3. Select "Accessible from repositories in 'organization' organization"

**Step 2: Reference in workflow**:
```yaml
steps:
  - uses: organization/private-action-repo/.github/actions/my-action@main
```

**Important limitations**:
- Source and consumer must **both be private**, OR
- Source and consumer must **both be public**
- **Cannot** use private action from public repo

## Complete Example: All Action Types

Here's a workflow demonstrating all action types:

```yaml
name: Action Types Demo

on: workflow_dispatch

jobs:
  composite:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: ./.github/actions/composite-hello
        with:
          who-to-greet: 'Composite User'
  
  javascript:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: ./.github/actions/js-hello
        with:
          name: 'JS User'
  
  container-dockerfile:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: ./.github/actions/container-hello
        with:
          who-to-greet: 'Container User'
  
  container-prebuilt:
    runs-on: ubuntu-latest
    steps:
      - uses: docker://ghcr.io/user/action:v1.0.0
        with:
          who-to-greet: 'Prebuilt Container User'

  reusable:
    uses: ./.github/workflows/reusable-deploy.yml
    with:
      environment: staging
    secrets:
      deploy-key: ${{ secrets.STAGING_KEY }}
```

## Key Takeaways

**Composite actions**:
- Easiest to create (just bundle steps in action.yml)
- No build process required
- Limited to step-level reuse
- Best for 3-10 step sequences

**Reusable workflows**:
- Job-level reuse (entire workflow definitions)
- Enforce standardization across organization
- Support inputs and secrets (with inheritance)
- Best for complete CI/CD templates

**JavaScript/TypeScript actions**:
- GitHub's most natively supported type
- Official @actions/core package for integration
- Requires build/bundle step
- Best for complex logic with npm dependencies

**Container actions**:
- Any language supported
- Requires Docker knowledge
- Slower startup (container build/pull)
- Best when team expertise is in non-JS language

**Decision factors**:
1. Can I find a marketplace action? (use that first)
2. Is it simple? (inline bash)
3. Need to reuse? (composite action or reusable workflow)
4. Need dependencies? (JavaScript or container action)
5. JavaScript viable? (prefer JS action over container)

**Publishing**:
- Public actions: Marketplace via GitHub release UI
- Private actions: Grant access in repository settings
- Always pin to commit SHA when consuming actions

## Next Chapter

You've learned how to build custom actions for any need. In Chapter 9, we'll explore **Common Workflows**—real-world CI/CD patterns you'll encounter in production. You'll see how to combine marketplace actions, custom actions, and GitHub's features to build complete automation systems for testing, building, deploying, and maintaining applications. These battle-tested patterns form the foundation of modern software delivery.
