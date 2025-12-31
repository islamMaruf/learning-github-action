# GitHub Actions - Complete Learning Guide

A comprehensive learning resource for mastering GitHub Actions, from fundamental concepts to production-ready CI/CD pipelines. This guide covers everything you need to build automated workflows for modern software development.

## 📚 Course Overview

This learning path combines theory and hands-on practice to take you from beginner to proficient GitHub Actions practitioner. You'll learn to automate testing, building, deployment, and operational tasks using GitHub's native CI/CD platform.

## 🎯 What You'll Learn

- GitHub Actions fundamentals and core concepts
- Workflow authoring and automation patterns
- Testing strategies for multi-service applications
- Container image building and deployment
- GitOps and release automation
- Performance optimization and debugging techniques
- Production-ready best practices

## 📖 Course Structure

### [Chapter 1: Introduction to GitHub Actions](01_introduction.md)
Get started with GitHub Actions basics. Learn what GitHub Actions is, why automation matters in modern software development, and understand the course structure. Covers the learning approach, prerequisites, and sets expectations for the hands-on journey ahead.

**Key Topics:** Automation fundamentals, CI/CD concepts, course methodology, learning rhythm

---

### [Chapter 2: History and Motivation](02_history_and_motivation.md)
Explore the evolution of software delivery from punch cards to continuous deployment. Understand the historical context that led to modern CI/CD practices and see how GitHub Actions fits into the broader landscape of automation tools.

**Key Topics:** Software delivery evolution, CI/CD history, deployment frequency trends, infrastructure as code

---

### [Chapter 3: Why GitHub Actions?](03_why_github_actions.md)
Compare GitHub Actions with alternatives like Jenkins, GitLab CI, CircleCI, and Travis CI. Learn the strengths and trade-offs of each platform to make informed decisions about which tool fits your team's needs.

**Key Topics:** Platform comparison, GitHub Actions advantages, expressiveness limitations, debugging experience, cost analysis

---

### [Chapter 4: Development Environment Setup](04_development_environment.md)
Set up a reproducible development environment using Devbox and other essential tools. Learn to fork and clone the course repository, install dependencies, and configure local testing capabilities.

**Key Topics:** Devbox installation, repository setup, tool configuration, local workflow testing with Act

---

### [Chapter 5: Core Features](05_core_features.md)
Master the fundamental building blocks of GitHub Actions: events, workflows, jobs, steps, and runners. Learn to trigger workflows, pass data between components, manage secrets, and control execution flow.

**Key Topics:** Workflow anatomy, event triggers, job orchestration, context and expressions, secrets management, artifacts

---

### [Chapter 6: Advanced Features](06_advanced_features.md)
Extend your workflow capabilities with advanced GitHub Actions features. Explore runner types (GitHub-hosted, self-hosted, third-party), job containers, matrix strategies, concurrency control, and permissions management.

**Key Topics:** Runner infrastructure, container jobs, matrix builds, concurrency groups, GITHUB_TOKEN permissions, caching strategies

---

### [Chapter 7: GitHub Actions Marketplace](07_marketplace_actions.md)
Leverage thousands of pre-built actions from the GitHub Actions Marketplace. Learn to discover, evaluate, and integrate community-built solutions while understanding security considerations and versioning strategies.

**Key Topics:** Marketplace navigation, action evaluation, security best practices, popular actions, version pinning

---

### [Chapter 8: Authoring Actions](08_authoring_actions.md)
Create custom reusable logic with composite actions, reusable workflows, JavaScript actions, and container actions. Understand when to use each approach and how to structure, test, and publish your own actions.

**Key Topics:** Composite actions, reusable workflows, JavaScript/TypeScript actions, Docker actions, publishing strategies

---

### [Chapter 9: Common Workflows](09_common_workflows.md)
Explore real-world workflow patterns for testing, building, deploying, and releasing applications. See practical examples of CI/CD automation across different programming languages and deployment targets.

**Key Topics:** CI/CD patterns, deployment workflows, release automation, multi-environment deployments

---

### [Chapter 10: Developer Experience](10_developer_experience.md)
Speed up development and debug workflows efficiently. Learn techniques for local testing, SSH debugging, performance monitoring, and iteration strategies that minimize the "push and pray" cycle.

**Key Topics:** Local action testing, Act workflow runner, debugging strategies, VS Code extensions, performance monitoring with Honeycomb

---

### [Chapter 11: Best Practices](11_best_practices.md)
Build production-grade workflows with performance optimization, maintainability patterns, and security practices. Learn monorepo vs. multirepo considerations and how to scale automation as your team grows.

**Key Topics:** Performance optimization, caching strategies, security practices, workflow maintainability, monorepo patterns, cost optimization

---

### [Chapter 12: Test Workflow](12_test_workflow.md)
Build a comprehensive test workflow for a multi-service monorepo application. Implement intelligent service filtering, language-specific testing, and conditional execution to optimize CI performance.

**Key Topics:** Monorepo testing, path filtering, matrix strategies, conditional execution, multi-language support

---

### [Chapter 13: Build and Push Images](13_build_push_images.md)
Automate container image building and publishing for multiple services. Implement semantic versioning, environment-specific tagging, and multi-architecture builds with proper registry authentication.

**Key Topics:** Docker image building, semantic versioning, container registries, multi-architecture builds, Taskfile conventions

---

### [Chapter 14: GitOps and Automation](14_remaining_workflows_conclusion.md)
Complete your CI/CD pipeline with GitOps manifest updates, release automation, stale issue management, and performance monitoring. Integrate all course concepts into a production-ready DevOps platform.

**Key Topics:** GitOps deployment, Kubernetes manifest updates, release automation, issue management, observability

---

## 🚀 Getting Started

1. **Fork the repository** to your GitHub account
2. **Clone with submodules**: `git clone --recurse-submodules <your-fork-url>`
3. **Set up development environment** following Chapter 4
4. **Start with Chapter 1** and progress sequentially
5. **Practice with examples** in each chapter

## 🛠️ Prerequisites

- **Git** installed and basic familiarity
- **GitHub account** (free tier sufficient)
- **Terminal/command-line** experience
- **Code editor** (VS Code recommended)
- **Operating system**: macOS, Linux, or Windows with WSL

## 📦 What's Included

- Comprehensive written documentation for each chapter
- Hands-on workflow examples and templates
- Capstone project with multi-service application
- Best practices and production patterns
- Debugging and optimization techniques

## 🎓 Learning Path

**Beginner Track** (Chapters 1-5): Start here if you're new to GitHub Actions. Learn fundamentals, core features, and basic workflow creation.

**Intermediate Track** (Chapters 6-9): Build on fundamentals with advanced features, marketplace actions, custom action authoring, and common patterns.

**Advanced Track** (Chapters 10-14): Master professional practices including debugging, performance optimization, and building production CI/CD pipelines.

## 💡 Course Philosophy

This course follows a **theory-then-practice** approach:
- **Theory sections** explain concepts, features, and mental models
- **Practice sections** apply knowledge through hands-on examples
- **Real-world examples** demonstrate production usage patterns
- **Iterative learning** builds complexity progressively

## 🔗 Additional Resources

- [GitHub Actions Documentation](https://docs.github.com/en/actions)
- [GitHub Actions Marketplace](https://github.com/marketplace?type=actions)
- [Course Repository Issues](https://github.com/sidpalas/devops-directive-github-actions-course/issues)

## 📝 Notes

- All examples are tested and production-ready
- Workflows can be adapted to your specific projects
- Community contributions and feedback welcome
- Regular updates with new features and best practices

## 🙏 Acknowledgments

This comprehensive guide represents months of research, practical experience, and refinement to create the definitive resource for learning GitHub Actions effectively.

---

**Ready to automate everything?** Start with [Chapter 1: Introduction to GitHub Actions](01_introduction.md) and begin your journey to CI/CD mastery!
