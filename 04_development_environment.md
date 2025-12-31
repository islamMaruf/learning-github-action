# Chapter 4: Setting Up Your Development Environment

## Overview

Before diving into writing and running GitHub Actions workflows, we need to ensure your development environment is properly configured with all the necessary tools. This chapter will guide you through setting up everything you need to follow along with the course examples, experiment with workflows, and develop GitHub Actions effectively. We'll take a methodical approach, explaining not just *what* to install, but *why* each tool matters and how it fits into your workflow.

## Why Development Environment Setup Matters

**The goal**: Create a **reproducible**, **isolated** development environment that:
- Matches the instructor's setup (avoiding "works on my machine" issues)
- Doesn't interfere with other projects on your computer
- Provides all necessary tools at correct versions
- Enables local testing before pushing to GitHub

**The problem we're solving**: Without proper setup, you'll encounter:
- Version mismatches causing mysterious errors
- Missing dependencies breaking examples
- Conflicts between project requirements
- Slow iteration due to inability to test locally

**Investment**: 30-60 minutes of setup will save hours of debugging later.

## Prerequisites Review

Before starting, ensure you have:
- **Operating system**: macOS, Linux, or Windows with WSL (Windows Subsystem for Linux)
- **Git installed**: Check with `git --version`
- **Terminal access**: Comfortable running shell commands
- **GitHub account**: Free account is sufficient

## Step 1: Fork and Clone the Course Repository

### Why Fork Instead of Just Clone?

**Forking** creates your own copy of the repository under your GitHub account. This is essential because:

1. **You become the owner**: Only repository owners can execute GitHub Actions workflows
2. **You can modify freely**: Experiment without affecting the original repository
3. **Workflows will run in your account**: Using your free GitHub Actions minutes, not the instructor's
4. **You can push changes**: Test your own workflow modifications

**If you just cloned** (without forking), workflows wouldn't run because GitHub Actions only executes workflows in repositories you own or have appropriate permissions for.

### Forking Process

**Step-by-step**:

1. **Navigate to the course repository**:
   ```
   https://github.com/sidpalas/devops-directive-github-actions-course
   ```

2. **Click the "Fork" button** (top right of the page)

3. **Choose destination**:
   - Select your personal account or an organization you own
   - Keep the repository name or customize it
   - Add a description (optional)

4. **Click "Create fork"**

**What happens**: GitHub creates a complete copy of the repository under your account, including all branches, commits, and history.

### Cloning Your Fork with Submodules

**Why submodules matter**: The course repository includes a **git submodule** for the capstone project. Submodules are separate repositories embedded within the main repository.

**The command**:
```bash
git clone --recurse-submodules https://github.com/YOUR-USERNAME/devops-directive-github-actions-course
```

**Breaking down the command**:
- `git clone`: Creates a local copy of the repository
- `--recurse-submodules`: Automatically clones submodules too
- Replace `YOUR-USERNAME` with your actual GitHub username

**What you'll see**:
```
Cloning into 'devops-directive-github-actions-course'...
remote: Enumerating objects: 1234, done.
remote: Counting objects: 100% (1234/1234), done.
remote: Compressing objects: 100% (567/567), done.
Receiving objects: 100% (1234/1234), 5.67 MiB | 2.34 MiB/s, done.
Resolving deltas: 100% (890/890), done.
Submodule 'capstone' (https://github.com/sidpalas/devops-directive-docker-course-example) registered for path 'capstone'
Cloning into 'capstone'...
```

**Navigate into the directory**:
```bash
cd devops-directive-github-actions-course
```

**Verify submodule loaded**:
```bash
ls -la capstone/
```

You should see files, not an empty directory.

### What if You Forget --recurse-submodules?

If you already cloned without the flag:
```bash
git submodule update --init --recursive
```

This initializes and updates all submodules after the fact.

## Step 2: Install Devbox for Reproducible Environments

### What is Devbox?

**Devbox** is a tool that creates **isolated, reproducible development environments** using Nix under the hood.

**The problem it solves**: Different projects often require different tool versions. For example:
- Project A needs Node.js 16
- Project B needs Node.js 18
- This course needs specific versions of multiple tools

**Traditional approach problems**:
- Installing globally pollutes your system
- Version conflicts between projects
- "Works on my machine" syndrome (different team members have different versions)
- Manual tracking of required versions

**Devbox approach**:
- Each project specifies required tools and versions in `devbox.json`
- Running `devbox shell` enters an isolated environment with those exact versions
- Exiting the shell returns to your normal environment
- Other projects are unaffected

### Devbox Architecture

**Under the hood**: Devbox uses **Nix**, a powerful package manager, but provides a simpler interface.

**Key concepts**:

**devbox.json**: Specifies which packages you need
```json
{
  "packages": [
    "nodejs@latest",
    "docker@latest",
    "kubectl@latest",
    "gh@latest"
  ]
}
```

**devbox.lock**: Locks packages to specific versions (like `package-lock.json` for npm)
```json
{
  "packages": {
    "nodejs": {
      "version": "18.17.0",
      "system": "x86_64-linux",
      "url": "..."
    }
  }
}
```

**The workflow**:
```
Run `devbox shell`
    ↓
Devbox reads devbox.json
    ↓
Checks devbox.lock for exact versions
    ↓
Downloads/installs any missing packages
    ↓
Starts shell with packages in PATH
    ↓
You work with correct versions
    ↓
Exit shell → back to normal environment
```

### Installing Devbox

**Official installation instructions**: https://www.jetpack.io/devbox/docs/installing_devbox/

**For macOS**:
```bash
curl -fsSL https://get.jetpack.io/devbox | bash
```

**For Linux**:
```bash
curl -fsSL https://get.jetpack.io/devbox | bash
```

**For Windows**: You must use WSL (Windows Subsystem for Linux). Install WSL first, then follow Linux instructions.

**Important note about Windows**: The instructor explicitly states:
> "I haven't tested any of this on Windows directly. You'll have a much better time if you work within a Linux environment under WSL."

**Why WSL is required for Windows users**:
- GitHub Actions runners are Linux-based
- Many commands and scripts assume Linux environment
- Devbox/Nix requires Unix-like system
- Testing locally will match GitHub's environment

**Setting up WSL** (Windows users only):
```powershell
# In PowerShell as Administrator
wsl --install
```

Then restart and follow Linux installation instructions inside WSL.

### Using Devbox in the Course Repository

**Navigate to the cloned repository**:
```bash
cd devops-directive-github-actions-course
```

**Inspect the devbox configuration**:
```bash
cat devbox.json
```

**You'll see something like**:
```json
{
  "packages": [
    "nodejs@latest",
    "python3@latest",
    "go@latest",
    "docker@latest",
    "kubectl@latest",
    "act@latest",
    "gh@latest",
    "jq@latest"
  ]
}
```

**What each tool is for**:
- `nodejs`: JavaScript runtime (needed for some actions and examples)
- `python3`: Python runtime (similar reasons)
- `go`: Go runtime (for building some tools and examples)
- `docker`: Container runtime (for local testing with Act, building images)
- `kubectl`: Kubernetes CLI (for capstone deployment examples)
- `act`: Run GitHub Actions locally
- `gh`: GitHub CLI (interact with GitHub from terminal)
- `jq`: JSON processor (parsing workflow outputs)

**Start the Devbox shell**:
```bash
devbox shell
```

**First run**: This will take several minutes as Devbox downloads and installs all packages.

**What you'll see**:
```
Installing packages... [this may take a few minutes]
✓ nodejs@18.17.0
✓ python3@3.11.4
✓ go@1.21.0
✓ docker@24.0.5
✓ kubectl@1.27.4
✓ act@0.2.48
✓ gh@2.32.1
✓ jq@1.6

Starting devbox shell...
```

**Verify installation**:
```bash
node --version    # Should show v18.17.0 or similar
python3 --version # Should show Python 3.11.4 or similar
go version        # Should show go1.21.0 or similar
docker --version  # Should show Docker version 24.0.5 or similar
act --version     # Should show act version 0.2.48 or similar
```

**Important**: These versions are **locked** by `devbox.lock`, ensuring everyone using the course has identical tool versions.

### Devbox Shell Lifecycle

**Enter the shell**:
```bash
devbox shell
```
Your prompt may change to indicate you're in a Devbox environment.

**Work normally**: All commands run with Devbox-provided tools.

**Exit the shell**:
```bash
exit
# or press Ctrl+D
```

**Back to normal environment**: Your system's default tool versions are active again.

**Example**:
```bash
# Outside Devbox
node --version
# v14.17.0 (your system version)

devbox shell
# Inside Devbox
node --version
# v18.17.0 (Devbox version)

exit
# Outside Devbox again
node --version
# v14.17.0 (back to system version)
```

## Step 3: Install Docker Desktop

### Why Docker Desktop?

Docker Desktop provides a **container runtime** necessary for:

1. **Building container images** in workflows
2. **Running containerized actions** (actions that execute inside containers)
3. **Local workflow testing with Act** (Act runs workflows inside containers)
4. **Capstone project** (involves building and deploying containerized applications)

**What is Docker Desktop**: A desktop application that includes:
- Docker Engine (the core container runtime)
- Docker CLI (command-line interface)
- Docker Compose (multi-container orchestration)
- Kubernetes integration (optional)
- Graphical management interface

### Installation Process

**Official documentation**: https://docs.docker.com/desktop/

**For macOS**:
1. Download Docker Desktop for Mac (Intel or Apple Silicon depending on your hardware)
2. Install the .dmg file
3. Launch Docker Desktop
4. Complete the setup wizard

**For Linux**:
Docker Desktop is available for some Linux distributions, or install Docker Engine directly:
```bash
# Ubuntu/Debian example
sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io
sudo usermod -aG docker $USER
# Log out and back in for group changes to take effect
```

**For Windows with WSL**:
1. Download Docker Desktop for Windows
2. During installation, ensure "Use WSL 2 instead of Hyper-V" is selected
3. Docker will integrate with your WSL environments

### Verifying Docker Installation

**Check Docker is running**:
```bash
docker version
```

**Expected output**:
```
Client: Docker Engine - Community
 Version:           24.0.5
 API version:       1.43
 ...

Server: Docker Engine - Community
 Engine:
  Version:          24.0.5
  API version:      1.43 (minimum version 1.12)
  ...
```

**Test Docker works**:
```bash
docker run hello-world
```

**Expected output**:
```
Hello from Docker!
This message shows that your installation appears to be working correctly.
```

### Docker Concepts Refresher

Since Docker knowledge is an optional prerequisite, here's a quick refresher:

**Containers**: Isolated environments running applications with their dependencies
**Images**: Templates for containers (like a class in OOP)
**Dockerfile**: Recipe for building an image
**Registry**: Storage for images (Docker Hub, GitHub Container Registry, etc.)

**Why containers matter for CI/CD**:
- **Consistency**: Same environment locally and in CI
- **Isolation**: Dependencies don't conflict
- **Portability**: Build once, run anywhere

## Step 4: Configure VS Code for GitHub Actions Development

### Why VS Code?

The instructor uses **Visual Studio Code** throughout the course. While you can use any editor, VS Code provides:
- Excellent YAML editing support
- GitHub Actions-specific extensions
- Integrated terminal
- Git integration
- Free and cross-platform

### Essential Extensions

#### Extension 1: YAML by Red Hat

**What it provides**:
- **Syntax highlighting**: Colors for better readability
- **Validation**: Catches YAML syntax errors
- **Auto-completion**: Suggests valid keys
- **Formatting**: Auto-indents and formats YAML

**Installation**:
1. Open VS Code
2. Click Extensions icon (or Cmd/Ctrl+Shift+X)
3. Search "YAML"
4. Find "YAML" by Red Hat
5. Click "Install"

**Why YAML validation matters**: YAML is **whitespace-sensitive**. A missing space or wrong indentation breaks your workflow. This extension catches errors before you push.

**Example error it catches**:
```yaml
# Invalid YAML (wrong indentation)
jobs:
build:  # This line is wrong!
  runs-on: ubuntu-latest
```

The extension will show a red squiggle and error: "Incorrect indentation"

#### Extension 2: GitHub Actions by GitHub

**What it provides**:
- **Workflow visualization**: See workflows in a tree view
- **Action documentation**: Hover over actions for inline docs
- **Workflow execution history**: View past runs from VS Code
- **Workflow triggers**: Manually trigger workflows

**Installation**:
Same process as YAML extension, search "GitHub Actions"

**Note from instructor**: "I don't use them that much but it's nice to have installed."

### Configuration Gotcha: File Associations

**The problem**: When both extensions are installed, the GitHub Actions extension takes ownership of workflow YAML files, preventing the YAML extension from working properly.

**Symptoms**:
- YAML syntax highlighting doesn't work in `.github/workflows/*.yml`
- No auto-completion or validation
- Files treated as generic text instead of YAML

**The fix**: Configure VS Code to associate these files with YAML language.

**Method 1: Edit settings.json directly**:

Press Cmd/Ctrl+Shift+P, type "Open Settings (JSON)", press Enter.

Add this:
```json
{
  "files.associations": {
    ".github/workflows/*.yml": "yaml",
    ".github/workflows/*.yaml": "yaml",
    "taskfile.yml": "yaml"
  }
}
```

**Method 2: Use Settings GUI**:
1. Open Settings (Cmd/Ctrl+,)
2. Search "associations"
3. Find "Files: Associations"
4. Click "Add Item"
5. Pattern: `.github/workflows/*.yml`
6. Value: `yaml`
7. Repeat for other patterns

**Why the taskfile.yml entry**: The course uses **Task** (a task runner tool) which stores commands in `taskfile.yml`. Without this association, YAML features won't work for task files.

**What this configuration does**:
- Tells VS Code to treat files matching these patterns as YAML
- Enables YAML extension features for these files
- Both extensions can now coexist peacefully

### Verifying Your Setup

**Create a test workflow**:
1. In VS Code, navigate to `.github/workflows/` (create if doesn't exist)
2. Create file `test.yml`
3. Start typing:
```yaml
name: Test
on: push
jobs:
```

**You should see**:
- ✅ Syntax highlighting (colors)
- ✅ Auto-completion suggestions
- ✅ Indentation guides
- ✅ No errors yet (we haven't finished the workflow)

## Tool Reference: Complete Development Environment

At this point, your environment includes:

| Tool | Purpose | Verify Command |
|------|---------|----------------|
| Git | Version control | `git --version` |
| GitHub CLI (gh) | GitHub API access from terminal | `gh --version` |
| Node.js | JavaScript runtime for actions | `node --version` |
| Python | Python runtime for actions | `python3 --version` |
| Go | Go runtime for actions | `go version` |
| Docker | Container runtime | `docker --version` |
| kubectl | Kubernetes CLI | `kubectl version --client` |
| Act | Run workflows locally | `act --version` |
| jq | JSON parsing | `jq --version` |
| VS Code | Code editor | `code --version` |

**Quick verification script**:
```bash
# Run inside Devbox shell
echo "Git: $(git --version)"
echo "GitHub CLI: $(gh --version)"
echo "Node: $(node --version)"
echo "Python: $(python3 --version)"
echo "Go: $(go version)"
echo "Docker: $(docker --version)"
echo "kubectl: $(kubectl version --client --short 2>/dev/null)"
echo "Act: $(act --version)"
echo "jq: $(jq --version)"
echo "VS Code: $(code --version | head -n1)"
```

All should output version numbers without errors.

## Troubleshooting Common Issues

### Issue: Devbox shell fails to start

**Symptoms**: Errors about Nix or unable to download packages

**Solutions**:
1. Ensure you have internet connection (Devbox downloads packages)
2. Check if Nix is properly installed: `nix --version`
3. Try clearing Devbox cache: `devbox cache clear`
4. Reinstall Devbox using official instructions

### Issue: Docker commands fail with permission errors

**Symptoms**: "permission denied while trying to connect to Docker daemon"

**Solutions (Linux)**:
```bash
# Add your user to docker group
sudo usermod -aG docker $USER
# Log out and back in (or restart)
```

**Solutions (macOS/Windows)**: Ensure Docker Desktop is running (check system tray icon)

### Issue: Git submodules not loaded

**Symptoms**: `capstone/` directory is empty

**Solution**:
```bash
git submodule update --init --recursive
```

### Issue: VS Code extensions not working

**Symptoms**: No syntax highlighting or autocomplete

**Solutions**:
1. Verify extensions are installed and enabled
2. Check file associations in settings
3. Reload VS Code window: Cmd/Ctrl+Shift+P → "Reload Window"
4. Check for extension conflicts

## Best Practices for Development Environment

### Use Devbox Shell for All Course Work

**Always start with**:
```bash
cd devops-directive-github-actions-course
devbox shell
```

This ensures you're using correct tool versions.

### Keep Your Fork Updated

Periodically sync your fork with the original repository:
```bash
# Add original repo as upstream (one-time)
git remote add upstream https://github.com/sidpalas/devops-directive-github-actions-course

# Sync with upstream
git fetch upstream
git merge upstream/main
```

### Organize Your Experiments

Create branches for experimentation:
```bash
git checkout -b experiment-with-caching
# Make changes, test workflows
# Keep main branch clean
```

## Next Steps: You're Ready!

With your development environment fully configured, you now have:
- ✅ Course repository cloned and ready
- ✅ Reproducible tool environment via Devbox
- ✅ Container runtime for testing
- ✅ Optimized editor with extensions

**You're prepared to**:
- Write workflows locally with validation
- Test workflows locally with Act
- Push changes and run on GitHub
- Follow along with all course examples

## Key Takeaways

- **Forking is essential** for running workflows in your own account
- **Devbox provides reproducibility** by locking tool versions
- **Docker enables local testing** through Act and container actions
- **VS Code extensions enhance productivity** with validation and autocomplete
- **File associations matter** for extension functionality
- **Environment isolation prevents conflicts** between projects

## Next Chapter

Now that your development environment is ready, Chapter 5 will introduce the **Core Features** of GitHub Actions. We'll examine the fundamental building blocks—workflows, jobs, steps, runners, and events—through both conceptual explanations and hands-on examples. This is where we start writing and executing actual GitHub Actions workflows!
