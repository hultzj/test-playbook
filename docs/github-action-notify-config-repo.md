# GitHub Action Workflow: notify-config-repo.yml

## Overview

The `notify-config-repo.yml` workflow automates the process of notifying a Configuration as Code (CaC) repository when Ansible playbooks are updated. This enables automated deployment pipelines where playbook changes automatically trigger configuration updates in Ansible Automation Platform (AAP).

## Workflow Purpose

This workflow is designed for a **playbook repository** and serves two main functions:

1. **Validate playbooks** - Ensures changes pass syntax checks and linting before deployment
2. **Trigger downstream automation** - Notifies the configuration repository (`hello-world-CaC`) to update AAP with the latest playbook changes

## Key Features

- **Tests on all branches** - Feature branches get validated without triggering deployments
- **Deploys only from protected branches** - `main` and `develop` are the only branches that trigger AAP updates
- **Early validation** - Catch syntax errors and linting issues before merging
- **Automated deployment** - No manual intervention needed for `main`/`develop` changes

## Trigger Conditions

### Automatic Triggers

The workflow runs automatically when:

```yaml
on:
  push:
    paths:
      - '*.yml'
      - '*.yaml'
      - 'roles/**'
      - 'playbooks/**'
```

- **Branch**: Triggers on **any branch** when playbook files change
- **File paths**: Only triggers when specific files change:
  - Root-level `.yml` or `.yaml` files
  - Any files in `roles/` directory
  - Any files in `playbooks/` directory

This filtering prevents the workflow from running on changes to documentation, CI configs, or other non-playbook files.

**Important**: While the workflow runs on all branches, the notification to the config repository only happens on `main` and `develop` branches (see Job 2 details below).

### Manual Trigger

```yaml
workflow_dispatch:
```

The workflow can also be triggered manually from the GitHub Actions UI for testing purposes.

## Workflow Jobs

### Job 1: test-playbooks

**Purpose**: Validate playbook syntax and quality before notifying downstream systems.

**Runs on**: `ubuntu-latest`

**Steps**:

1. **Checkout code** - Retrieves the repository content
   
2. **Set up Python 3.11** - Installs Python environment

3. **Install Ansible** - Installs required tools:
   - `ansible` - Core Ansible package
   - `ansible-lint` - Best practices linter
   - `yamllint` - YAML syntax validator

4. **Validate YAML syntax** - Runs `yamllint` across all files
   - Note: Uses `|| true` to prevent pipeline failure (continues on errors)

5. **Lint Ansible playbooks** - Runs `ansible-lint` for Ansible-specific best practices
   - Also non-blocking (`|| true`)

6. **Syntax check playbooks** - Performs `ansible-playbook --syntax-check` on all `.yml` files
   - Validates each playbook individually
   - Non-blocking to ensure notification still occurs

7. **Test Summary** - Adds results to GitHub's step summary for easy review

**Note**: All test steps use `|| true`, meaning they won't fail the workflow. This is a design choice to ensure the notification job always runs, even if linting finds issues.

### Job 2: notify-config-repo

**Purpose**: Trigger the Configuration as Code repository to deploy changes.

**Runs on**: `ubuntu-latest`

**Dependencies**: 
```yaml
needs: test-playbooks
if: github.ref == 'refs/heads/main' || github.ref == 'refs/heads/develop'
```

This job:
- Only runs after `test-playbooks` completes successfully
- **Only executes on `main` or `develop` branches** (feature branches skip this job)
- This allows playbook validation on all branches while limiting deployments to protected branches

**Steps**:

1. **Checkout code** - Retrieves repository content

2. **Get changed files** - Uses `tj-actions/changed-files@v41`
   - Identifies which `.yml`/`.yaml` files were modified
   - Stores results in `steps.changed-files.outputs.all_changed_files`

3. **Display changed playbooks** - Adds a formatted list to the GitHub step summary

4. **Trigger Config Repository Deployment** - The core step
   
   Uses `peter-evans/repository-dispatch@v2` to send a webhook event:
   
   ```yaml
   repository: hultzj/hello-world-CaC  # Target repository
   token: ${{ secrets.CONFIG_REPO_PAT }}  # Authentication
   event-type: playbook-updated  # Custom event identifier
   ```
   
   **Client Payload** - JSON data sent with the event:
   - `repository` - Source repository name
   - `ref` - Git reference (e.g., `refs/heads/develop`)
   - `branch` - Branch name (e.g., `develop`)
   - `sha` - Commit SHA that triggered the workflow
   - `actor` - GitHub user who made the change
   - `event` - Event type (`push` or `workflow_dispatch`)
   - `changed_files` - List of modified files

5. **Summary** - Creates comprehensive workflow summary explaining:
   - Notification success
   - Next steps in the deployment pipeline
   - Link to monitor the config repo actions

## Required Secrets

### CONFIG_REPO_PAT

A GitHub Personal Access Token (PAT) with the following:

- **Scope**: `repo` (full repository access)
- **Purpose**: Allows this repository to trigger workflows in `hultzj/hello-world-CaC`
- **Configuration**: Set in repository Settings → Secrets and variables → Actions

**To create this token**:
1. Go to GitHub Settings → Developer settings → Personal access tokens
2. Generate new token (classic)
3. Select `repo` scope
4. Copy token and add as `CONFIG_REPO_PAT` secret in this repository

## Integration with Configuration Repository

The config repository (`hello-world-CaC`) must have a workflow that listens for `repository_dispatch` events:

```yaml
on:
  repository_dispatch:
    types: [playbook-updated]
```

When this workflow triggers the event, the config repository receives the payload and can:

1. Update the AAP project (pulls latest playbooks from this repo)
2. Refresh job templates
3. Deploy changes through environments (Dev → Staging → Production)

## Workflow Execution Flow

```
┌─────────────────────────────────────┐
│  Developer pushes playbook changes  │
│  to ANY branch                      │
└─────────────────┬───────────────────┘
                  │
                  ▼
┌─────────────────────────────────────┐
│  GitHub detects changes to          │
│  *.yml, roles/**, or playbooks/**   │
└─────────────────┬───────────────────┘
                  │
                  ▼
┌─────────────────────────────────────┐
│  JOB 1: test-playbooks              │
│  ✓ YAML validation                  │
│  ✓ Ansible linting                  │
│  ✓ Syntax checking                  │
│  (Runs on ALL branches)             │
└─────────────────┬───────────────────┘
                  │
                  ▼
         ┌────────┴─────────┐
         │  Branch check    │
         └────────┬─────────┘
                  │
      ┌───────────┴───────────┐
      │                       │
      ▼                       ▼
┌──────────┐          ┌─────────────────┐
│ Feature  │          │ main/develop    │
│ branch   │          │ branch          │
│ (SKIP)   │          │ (CONTINUE)      │
└──────────┘          └────────┬────────┘
                               │
                               ▼
                ┌─────────────────────────────────────┐
                │  JOB 2: notify-config-repo          │
                │  ✓ Identify changed files           │
                │  ✓ Send repository_dispatch event   │
                │  ✓ Include metadata & changed files │
                └─────────────────┬───────────────────┘
                                  │
                                  ▼
                ┌─────────────────────────────────────┐
                │  Config repo (hello-world-CaC)      │
                │  receives event and triggers        │
                │  AAP deployment workflow            │
                └─────────────────────────────────────┘
```

## Monitoring and Debugging

### View Workflow Runs
- Navigate to **Actions** tab in this repository
- Select "Notify Config Repo on Playbook Update"
- View individual run details

### Check Step Summaries
Each run creates a summary showing:
- Which playbooks were tested
- Which files changed
- Confirmation of notification sent
- Link to config repo to monitor deployment

### Common Issues

**Workflow doesn't trigger**:
- Confirm modified files match path filters (`.yml`, `.yaml`, `roles/**`, `playbooks/**`)
- Check that workflow file is committed to `.github/workflows/`
- Note: Tests run on all branches, but notification only happens on `main` or `develop`

**Notification fails**:
- Verify `CONFIG_REPO_PAT` secret exists and is valid
- Confirm PAT has `repo` scope
- Check token hasn't expired
- Verify target repository name is correct: `hultzj/hello-world-CaC`

**Tests fail but workflow continues**:
- This is intentional (all tests use `|| true`)
- Review step logs to see linting/syntax issues
- Fix issues in subsequent commits

## Customization Options

### Add/modify branches for deployment
Update the conditional in the `notify-config-repo` job:
```yaml
if: github.ref == 'refs/heads/main' || github.ref == 'refs/heads/develop' || github.ref == 'refs/heads/staging'
```

### Add more path filters
```yaml
paths:
  - '*.yml'
  - '*.yaml'
  - 'roles/**'
  - 'playbooks/**'
  - 'inventory/**'  # Add inventory tracking
```

### Make tests blocking
Remove `|| true` from test commands to fail workflow on errors:
```yaml
- name: Validate YAML syntax
  run: yamllint .  # Will fail workflow on errors
```

### Change event type
Modify `event-type` to trigger different workflows in config repo:
```yaml
event-type: playbook-updated-dev  # Different event for dev branch
```

## Best Practices

1. **Token Security**: Never commit PAT tokens; always use secrets
2. **Branch Strategy**: The workflow tests on all branches but only deploys from `main`/`develop` - this ensures feature branches are validated without triggering production deployments
3. **Test Locally**: Run `ansible-lint` and `ansible-playbook --syntax-check` before pushing
4. **Monitor Deployments**: Check config repo actions after pushing changes
5. **Payload Documentation**: Document expected payload format in config repo
6. **Feature Branch Testing**: Push feature branches to see test results before merging to `main`/`develop`

## Related Documentation

- [GitHub Actions - repository_dispatch](https://docs.github.com/en/actions/using-workflows/events-that-trigger-workflows#repository_dispatch)
- [peter-evans/repository-dispatch action](https://github.com/peter-evans/repository-dispatch)
- [tj-actions/changed-files](https://github.com/tj-actions/changed-files)

