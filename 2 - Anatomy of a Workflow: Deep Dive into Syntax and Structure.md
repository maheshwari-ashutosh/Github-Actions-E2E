# Chapter 2: Anatomy of a Workflow: Deep Dive into Syntax and Structure

Welcome to the engine room of GitHub Actions. Chapter 1 introduced the _why_ and _what_ of workflow automation; this chapter dives deep into the _how_. We'll dissect the fundamental building blocks of a GitHub Actions workflow, exploring the YAML syntax, key components, and the logic that binds them together. Understanding this anatomy is crucial for crafting effective, reliable, and maintainable automation for your projects.

Workflows are defined in YAML files stored within a special directory in your repository: `.github/workflows/`. GitHub automatically detects files in this location and attempts to execute them based on their defined triggers. Let's break down the structure of these files piece by piece.

## A. Workflow Definition (`*.yml` files)

At its core, a workflow is a configuration file written in YAML (YAML Ain't Markup Language), a human-friendly data serialization standard. Each `.yml` or `.yaml` file in the `.github/workflows/` directory typically defines one distinct automated process.

```mermaid
graph LR
    A[Code Push] --> B{GitHub Actions};
    B -- Reads --> C[`.github/workflows/ci.yml`];
    C -- Defines --> D[Workflow: CI Pipeline];
    D -- Executes --> E[Job: Build];
    E -- Executes --> F[Job: Test];
    F -- Executes --> G[Job: Deploy (Optional)];
```

**Diagram Explanation:** This simple diagram illustrates a common CI/CD (Continuous Integration/Continuous Deployment) flow triggered by a code push. GitHub Actions reads the workflow definition file (`ci.yml`) and executes the defined jobs (Build, Test, potentially Deploy) in sequence. This chapter focuses on the structure and syntax within that `.yml` file.

### 1. The `name` Key: Naming Your Workflow

The optional `name` key provides a human-readable name for your workflow. This name is displayed in the GitHub UI (e.g., on the Actions tab of your repository), making it easier to identify and manage your workflows.

```yaml
# .github/workflows/ci-pipeline.yml
name: Continuous Integration Pipeline

on: [push] # Triggers (more on this next)

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - run: echo "Building the project..."
```

- **Purpose:** Improves clarity in the GitHub interface.
- **Optional:** If omitted, GitHub uses the workflow file's path relative to the repository root (e.g., `.github/workflows/ci-pipeline.yml`).
- **Best Practice:** Use a clear, descriptive name that reflects the workflow's purpose.

### 2. The `on` Key: Triggering Workflows with Events

The `on` key is fundamental; it defines what event(s) will trigger the execution of your workflow. Without this key, your workflow will never run automatically. GitHub Actions supports a wide variety of events originating from repository activity, schedules, manual triggers, and external systems.

```yaml
# .github/workflows/simple-trigger.yml
name: Simple Push Trigger

# Trigger this workflow on every push to any branch
on: push

jobs:
  say_hello:
    runs-on: ubuntu-latest
    steps:
      - run: echo "Workflow triggered by a push event!"
```

We will explore the diverse range of event types and their configurations in the next section.

## B. Events (`on`) In Detail

The `on` key is the gatekeeper for your automation. Mastering its configuration allows you to precisely control _when_ your workflows execute.

### 1. Repository Events (`push`, `pull_request`, `release`, `fork`, etc.)

These are the most common triggers, tied directly to activity within your GitHub repository.

- `push`: Triggered when commits are pushed to a repository branch or tag.
- `pull_request`: Triggered when a pull request is opened, updated (synchronized), closed, reopened, assigned, labeled, etc. You can specify activity types using the `types` keyword (e.g., `on: pull_request: types: [opened, synchronize, reopened]`).
- `release`: Triggered when a release is published, unpublished, created, edited, deleted, or prereleased. Use `types` to specify which actions trigger the workflow.
- `issues`: Triggered when an issue is opened, edited, closed, etc.
- `issue_comment`: Triggered when a comment is created, edited, or deleted on an issue or pull request.
- `fork`: Triggered when a repository is forked.
- `watch`: Triggered when someone stars a repository (`started`).
- `create`: Triggered when a branch or tag is created.
- `delete`: Triggered when a branch or tag is deleted.
- ...and many more (e.g., `page_build`, `project_card`, `registry_package`). Refer to the official GitHub Actions documentation for the exhaustive list.

#### a. Filtering Events: Branches, Tags, Paths

Often, you don't want a workflow to run on _every_ push or pull request. You can add filters to the event configuration to make triggers more specific.

- `branches` / `branches-ignore`: Trigger only for pushes/PRs targeting specific branches (e.g., `main`, `develop`, `feature/*`) or ignore specific branches. Wildcards (`*`, `**`) are supported.
- `tags` / `tags-ignore`: Trigger only for pushes creating specific tags (e.g., `v*.*.*`) or ignore specific tags. Wildcards are supported.
- `paths` / `paths-ignore`: Trigger only if at least one modified file in the push/PR matches the specified path patterns (e.g., `src/**`, `docs/*.md`) or ignore changes in specific paths. Wildcards are supported.

> **Note:** When using multiple filters (`branches`, `paths`, etc.) for the _same_ event, the workflow triggers if _all_ conditions are met (logical AND). For example, `branches` _and_ `paths` filters must both match. However, if you list multiple _events_ (e.g., `on: [push, pull_request]`), the workflow triggers if _any_ of the events occur (logical OR).

##### [Code Snippet: Filtering `push` events to the `main` branch and specific paths]

```yaml
# .github/workflows/filtered-push.yml
name: Filtered Push Workflow

on:
  push:
    branches:
      - main # Only run on pushes to the main branch
    paths:
      - "src/**" # Run if files in src/ change
      - "package.json" # Also run if package.json changes
      - "!docs/**" # But ignore changes *only* in the docs/ folder

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4 # Checkout code to access files
      - run: echo "Push to main involving src/ or package.json detected."
      # Add build steps here...
```

#### b. Understanding Event Payloads (`github.event` context)

When an event triggers a workflow, GitHub sends a _payload_ containing detailed information about that specific event. This payload is accessible within your workflow via the `github.event` context object. The structure of this payload varies depending on the event type.

- For a `push` event, `github.event` might contain information about the commits, the pusher, the repository, etc.
- For a `pull_request` event, `github.event` will contain details about the pull request itself (number, title, body, head/base branches, user who opened it, etc.).

You can use expressions (covered in Chapter 3) to access data within this payload.

##### [Practical Example: Accessing PR details from the event payload]

```yaml
# .github/workflows/pr-comment.yml
name: Comment on New Pull Requests

on:
  pull_request:
    types: [opened] # Trigger only when a PR is opened

jobs:
  add_comment:
    runs-on: ubuntu-latest
    steps:
      - name: Post PR Comment
        uses: actions/github-script@v7
        with:
          script: |
            github.rest.issues.createComment({
              issue_number: context.issue.number, // Access PR number via context shortcut
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: `👋 Thanks for opening this PR! The number is ${{ github.event.pull_request.number }}. Title: "${{ github.event.pull_request.title }}"`
            })
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }} # Required for API calls
```

This example uses the `github.event` context to extract the PR number and title and post them as a comment on the newly opened pull request.

### 2. Scheduled Events (`schedule` with cron syntax)

Workflows can be triggered on a schedule using POSIX cron syntax. This is useful for recurring tasks like nightly builds, dependency updates checks, or generating reports.

```yaml
# .github/workflows/nightly-build.yml
name: Nightly Build

on:
  schedule:
    # Runs "At 03:15 on Monday" (UTC time)
    - cron: "15 3 * * 1"

jobs:
  nightly_job:
    runs-on: ubuntu-latest
    steps:
      - run: echo "Running the scheduled nightly job..."
      # Add steps for nightly tasks...
```

- **Syntax:** Uses standard cron syntax (`minute hour day(month) month day(week)`). Online cron generators can be helpful.
- **Timing:** Schedules run based on UTC time.

##### [Production Note: Considerations for scheduled job frequency and reliability]

- **Frequency:** GitHub Actions schedules may be delayed during periods of high load. They are not guaranteed to run at the _exact_ specified time but will run at the _next available opportunity_. Avoid overly frequent schedules (e.g., every minute) unless absolutely necessary, as this consumes runner minutes.
- **Branch:** Scheduled workflows run on the latest commit on the _default branch_ (usually `main` or `master`) of the repository.
- **Reliability:** While generally reliable, treat scheduled workflows as "best effort" for exact timing. If precise timing is critical, consider external scheduling solutions that trigger workflows via `repository_dispatch` or `workflow_dispatch`.

### 3. Manual Triggers (`workflow_dispatch`)

This event type allows you to trigger a workflow manually from the GitHub UI (Actions tab) or via the GitHub API. It's excellent for tasks that require human intervention to initiate, such as deployments to production or running specific maintenance scripts.

```yaml
# .github/workflows/manual-deploy.yml
name: Manual Deployment

on:
  workflow_dispatch: # Allows manual triggering

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - run: echo "Manual deployment initiated..."
      # Add deployment steps...
```

#### a. Defining Inputs for Manual Workflows

You can define input parameters that users must (or can optionally) provide when manually triggering the workflow. This makes manual workflows much more flexible. Supported input types include `string`, `choice`, `boolean`, and `environment`.

- `description`: Explains the input's purpose in the UI.
- `required`: Specifies if the input must be provided (`true` or `false`).
- `default`: Provides a default value if the user doesn't specify one.
- `type`: The data type (`string`, `choice`, `boolean`, `environment`).
- `options` (for `choice` type): A list of allowed values.

Inputs are accessible within the workflow via the `github.event.inputs` or `inputs` context (more on contexts in Chapter 3).

##### [Configuration Guide: Setting up a workflow with user-defined inputs]

```yaml
# .github/workflows/manual-task-with-inputs.yml
name: Manual Task with Inputs

on:
  workflow_dispatch:
    inputs:
      environment:
        description: "Target Environment"
        required: true
        type: choice
        options:
          - staging
          - production
        default: "staging"
      logLevel:
        description: "Log Level"
        required: false
        type: string
        default: "info"
      forceDeploy:
        description: "Force deployment?"
        required: false
        type: boolean
        default: false

jobs:
  run_task:
    runs-on: ubuntu-latest
    steps:
      - name: Display Inputs
        run: |
          echo "Environment: ${{ github.event.inputs.environment }}"
          echo "Log Level: ${{ inputs.logLevel }}" # Shorter context syntax
          echo "Force Deploy: ${{ inputs.forceDeploy }}"

      - name: Conditional Step based on Input
        if: ${{ github.event.inputs.forceDeploy == 'true' }}
        run: echo "Forcing the task because forceDeploy was true!"

      # Add steps that use these inputs...
```

When triggering this workflow via the UI, the user will be presented with fields to select the environment, enter a log level, and check a box for force deployment.

### 4. External Events (`repository_dispatch`)

This event allows you to trigger a workflow from outside of GitHub, typically via a REST API call. This is useful for integrating GitHub Actions with external systems, CI/CD orchestrators, or custom tooling.

To trigger a `repository_dispatch` event, you send a POST request to a specific GitHub API endpoint for your repository, including a custom `event_type` name and an optional JSON `client_payload` containing data you want to pass to the workflow.

```yaml
# .github/workflows/external-trigger.yml
name: Handle External Trigger

on:
  repository_dispatch:
    types: [deploy_app, run_integration_test] # Only trigger for these specific event_types

jobs:
  handle_event:
    runs-on: ubuntu-latest
    steps:
      - name: Print Event Details
        run: |
          echo "Workflow triggered by repository_dispatch"
          echo "Event Type: ${{ github.event.action }}" # The event_type sent in the API call
          echo "Client Payload:"
          echo "${{ toJSON(github.event.client_payload) }}" # Access the custom payload

      - name: Deploy Step (Conditional)
        if: ${{ github.event.action == 'deploy_app' }}
        run: |
          echo "Deploying application..."
          echo "Target environment from payload: ${{ github.event.client_payload.environment }}"
          # Add deployment logic here...

      - name: Test Step (Conditional)
        if: ${{ github.event.action == 'run_integration_test' }}
        run: |
          echo "Running integration tests..."
          echo "Test suite from payload: ${{ github.event.client_payload.suite }}"
          # Add testing logic here...
```

#### a. Triggering Workflows via API Calls

You would typically use `curl` or a script/application to send a POST request like this:

```bash
curl -L \
  -X POST \
  -H "Accept: application/vnd.github+json" \
  -H "Authorization: Bearer <YOUR_GITHUB_TOKEN>" \
  -H "X-GitHub-Api-Version: 2022-11-28" \
  https://api.github.com/repos/YOUR_USERNAME/YOUR_REPO/dispatches \
  -d '{"event_type":"deploy_app","client_payload":{"environment":"staging","version":"1.2.3"}}'
```

Replace placeholders with your username, repository name, a GitHub token with `repo` scope, and the desired `event_type` and `client_payload`.

#### [Practical Example: Integrating an external system to trigger a workflow]

Imagine a custom deployment dashboard outside of GitHub. When an operator clicks "Deploy to Staging" on the dashboard, the dashboard's backend makes the API call shown above with `event_type: deploy_app` and relevant details in the `client_payload`. The `external-trigger.yml` workflow catches this event, extracts the environment and version from the payload, and executes the appropriate deployment steps.

### 5. Workflow Events (`workflow_run`, `workflow_call`)

These events allow workflows to interact with each other.

- `workflow_run`: Triggers your workflow when another specified workflow _run_ completes. You can filter based on the triggered workflow's name or file, branches, and completion status (`completed`, `success`, `failure`, `cancelled`). This is useful for creating post-processing workflows (e.g., generating reports after a CI build finishes).
- `workflow_call`: Allows you to create _reusable_ workflows that can be called by other workflows within the same organization or enterprise (or public repositories). This promotes DRY (Don't Repeat Yourself) principles by centralizing common sequences of jobs. `workflow_call` workflows can accept inputs and produce outputs.

#### [Deep Dive: Chaining workflows and understanding dependencies]

`workflow_run` creates a _loose coupling_ between workflows. Workflow A triggers Workflow B _after_ A finishes. They run independently, though B can access artifacts produced by A.

```yaml
# .github/workflows/main-ci.yml
name: Main CI Pipeline
on: push
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: echo "Building..." > build_output.txt
      - uses: actions/upload-artifact@v4
        with:
          name: build-artifact
          path: build_output.txt

# .github/workflows/post-ci-report.yml
name: Post-CI Report
on:
  workflow_run:
    workflows: ["Main CI Pipeline"] # Trigger when 'Main CI Pipeline' completes
    types:
      - completed # Trigger regardless of success or failure

jobs:
  report:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/download-artifact@v4 # Download artifact from the triggering run
        with:
          name: build-artifact
          # Note: Need github-token with actions:read scope if triggering repo is different
          # github-token: ${{ secrets.OTHER_REPO_TOKEN }}
          # run-id: ${{ github.event.workflow_run.id }} # Specify the run ID
      - run: |
          echo "Generating report based on workflow run status: ${{ github.event.workflow_run.conclusion }}"
          cat build_output.txt
```

`workflow_call`, on the other hand, creates a _tight coupling_. Workflow A explicitly _calls_ Workflow B (the reusable workflow) as if it were a function. Workflow A waits for Workflow B to complete before proceeding. This is ideal for shared logic like setting up environments, running linters, or performing standard deployment steps. We will explore `workflow_call` in more detail in Chapter 7 (Advanced Techniques) and Part III (Custom Actions).

## C. Jobs (`jobs`)

A workflow run is composed of one or more _jobs_. By default, jobs run in parallel. Each job runs in a fresh instance of a virtual environment specified by the `runs-on` key.

### 1. Defining Single and Multiple Jobs

Jobs are defined under the top-level `jobs:` key. Each key under `jobs:` is a unique identifier for that job (e.g., `build`, `test`, `deploy`). This ID is used for referencing the job elsewhere (e.g., in `needs` or the UI).

```yaml
# .github/workflows/multiple-jobs.yml
name: Build and Test

on: [push]

jobs:
  # First job definition
  build: # Job ID
    runs-on: ubuntu-latest
    steps:
      - run: echo "Building..."

  # Second job definition
  test: # Job ID
    runs-on: ubuntu-latest
    steps:
      - run: echo "Testing..."

  # Third job definition
  lint: # Job ID
    runs-on: windows-latest # Can run on different OS
    steps:
      - run: echo "Linting..."
```

In this example, the `build`, `test`, and `lint` jobs will start simultaneously and run in parallel on separate runners.

### 2. Job Execution Order and Dependencies (`needs`)

While parallel execution is efficient, many workflows require jobs to run sequentially. For example, you must build code before you can test it, and test it before you can deploy it. The `needs` key establishes dependencies between jobs.

A job with a `needs` key will only start after _all_ the jobs listed in `needs` have completed _successfully_. If any prerequisite job fails, the dependent job (and any jobs that depend on it) will be skipped by default.

##### [Practical Example: Creating a build -> test -> deploy job sequence]

```yaml
# .github/workflows/sequential-jobs.yml
name: Sequential Build, Test, Deploy

on:
  push:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - run: echo "Building..."
      # Imagine steps that produce build artifacts

  test:
    runs-on: ubuntu-latest
    needs: build # test will only start after build succeeds
    steps:
      # Imagine steps that download build artifacts and run tests
      - run: echo "Testing..."

  deploy:
    runs-on: ubuntu-latest
    needs: [build, test] # deploy needs both build and test to succeed
    steps:
      # Imagine steps that download artifacts and deploy
      - run: echo "Deploying..."
```

Here, `test` waits for `build`, and `deploy` waits for both `build` and `test`. This creates a linear execution flow: `build` -> `test` -> `deploy`. You can create complex dependency graphs by having jobs depend on multiple other jobs.

### 3. Conditional Execution of Jobs (`if`)

You can control whether a job runs based on a condition using the `if` key at the job level. The condition is evaluated _before_ the job starts. If the condition evaluates to `false`, the job is skipped entirely.

#### a. Using Contexts and Expressions in Conditions

Conditions typically involve checking values from contexts like `github` (event details, ref, SHA), `inputs` (for `workflow_dispatch`), `secrets`, or the status of previous jobs (`needs.<job_id>.result`). Chapter 3 will cover contexts and expressions in detail.

A common use case is running a deployment job only on pushes to the main branch.

##### [Code Snippet: Running a job only on the `main` branch]

```yaml
# .github/workflows/conditional-deploy.yml
name: Conditional Deploy

on: [push] # Trigger on any push

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - run: echo "Building on every push..."

  deploy_to_prod:
    runs-on: ubuntu-latest
    needs: build
    # This job only runs if the push was to the 'main' branch
    if: github.ref == 'refs/heads/main'
    steps:
      - run: echo "Deploying to production..."
```

Even though the workflow triggers on any push, the `deploy_to_prod` job will only execute if the `github.ref` context (which holds the reference that triggered the workflow) matches the full reference name for the `main` branch.

## D. Runners (`runs-on`)

Every job needs an environment to execute in. This environment is provided by a _runner_. The `runs-on` key specifies the type of runner machine to use for a job.

### 1. GitHub-Hosted Runners

GitHub provides and manages runners for you. These are convenient, require no setup, and come pre-installed with a wide range of common software tools.

#### a. Available Operating Systems (Ubuntu, Windows, macOS)

You can choose from several operating systems:

- `ubuntu-latest` (or specific versions like `ubuntu-22.04`, `ubuntu-20.04`)
- `windows-latest` (or specific versions like `windows-2022`, `windows-2019`)
- `macos-latest` (or specific versions like `macos-13`, `macos-12`)

Using `*-latest` is convenient as it always points to the latest stable version supported by GitHub, but specifying a version ensures greater consistency over time.

#### b. Runner Specifications and Limitations

- **Specs:** Typically offer 2-core CPUs, 7 GB RAM, 14 GB SSD (specs can vary, especially for larger runners available on paid plans). macOS runners generally have higher specs (e.g., 3-core CPU, 14 GB RAM).
- **Software:** Come pre-installed with many tools (Git, Docker, various programming language runtimes like Node.js, Python, Java, .NET, Go, Ruby, etc.). You can install additional software during your job run if needed.
- **Ephemeral:** Each job runs on a fresh, clean runner instance that is discarded after the job completes. No state persists between jobs unless explicitly passed using artifacts.
- **Usage Limits:** Free plans have limits on concurrent jobs and total execution minutes per month. Paid plans offer higher limits and potentially faster hardware.

#### [Production Note: Cost implications of GitHub-hosted runners]

- **Free Tier:** GitHub provides a generous free tier for public repositories and a limited amount for private repositories.
- **Minute Multipliers:** Execution time is measured in minutes. Linux runners consume minutes at a 1x rate. Windows runners typically consume minutes at a 2x rate, and macOS runners at a 10x rate.
- **Billing:** Usage beyond the free tier is billed per minute, based on the runner OS multiplier. Costs can add up, especially with frequent builds or long-running jobs on Windows/macOS. Optimizing workflow duration (Chapter 17) is important for cost management.

### 2. Self-Hosted Runners (Overview - Deep Dive in Part V)

Alternatively, you can host your own runners on your infrastructure (physical servers, VMs, cloud instances, containers).

#### a. When to Use Self-Hosted Runners

- **Custom Hardware:** Need more powerful CPUs, GPUs, more RAM, or specific hardware.
- **Custom Software/OS:** Require a specific operating system version or specialized software not available on GitHub-hosted runners.
- **Network Access:** Need access to private network resources (databases, internal services) behind a firewall.
- **Cost Control:** Can potentially be more cost-effective for very high usage volumes, especially if you have existing infrastructure.
- **Compliance/Security:** Need full control over the execution environment for compliance or security reasons.

#### b. Basic Concepts

You install the GitHub Actions runner application on your machine and register it with your repository, organization, or enterprise. The runner then polls GitHub for available jobs assigned to its labels (e.g., `self-hosted`, `linux`, `x64`, `gpu`). When it picks up a job, it executes the steps locally. You are responsible for maintaining the runner machine, its operating system, and installed software. Part V of this book provides a comprehensive guide to managing self-hosted runners.

## E. Steps (`steps`)

Jobs are composed of a sequence of individual tasks called _steps_. Steps are executed in order within a job, one after the other, on the same runner. If any step fails (typically by exiting with a non-zero code), subsequent steps in that job are skipped by default, and the job fails.

Steps are defined as a list under the `steps:` key within a job.

```yaml
jobs:
  example_job:
    runs-on: ubuntu-latest
    steps:
      # Step 1: Use an action
      - name: Check out repository code
        uses: actions/checkout@v4

      # Step 2: Run a shell command
      - name: List files
        run: ls -l

      # Step 3: Run another shell command
      - name: Say Goodbye
        run: echo "Finished!"
```

### 1. Defining Sequential Tasks within a Job

As shown above, steps are simply listed under the `steps:` key. They execute sequentially from top to bottom.

### 2. Running Shell Commands (`run`)

The `run` keyword allows you to execute command-line programs directly on the runner's operating system shell.

#### a. Single-line vs. Multi-line Scripts

- **Single-line:** `run: command arg1 arg2`
- **Multi-line:** Use the YAML literal block scalar `|` to write multi-line scripts. Each line is executed as a separate command in the shell, but they run within the same shell process, preserving state (like environment variables or current directory) between lines.

```yaml
steps:
  - name: Multi-line script example
    run: |
      echo "First command"
      pwd
      echo "Second command, continuing in the same shell"
      export MY_VAR="Hello" # Set an env var for subsequent lines in this step
      echo $MY_VAR
```

#### b. Working with Shells (Bash, PowerShell, Cmd, Python, etc.)

By default, GitHub Actions uses:

- `bash` on Linux and macOS runners.
- `pwsh` (PowerShell Core) on Windows runners.

You can explicitly specify a different shell using the `shell:` keyword for a specific step or set a default shell for the entire job or workflow.

```yaml
steps:
  - name: Run Python script
    shell: python {0} # Use python interpreter
    run: |
      import os
      print(f"Home directory: {os.getenv('HOME')}")

  - name: Run PowerShell on Linux
    shell: pwsh
    run: Get-Location # PowerShell command

  - name: Run Bash on Windows
    shell: bash
    run: pwd # Bash command
```

#### c. Error Handling and Exit Codes

- **Default Behavior:** If any command in a `run` step exits with a non-zero status code, the step is marked as failed. This usually causes the entire job to fail, and subsequent steps are skipped.
- **Bash `set -e`:** By default, multi-line `run` steps using `bash` execute with the `-e` option enabled. This means the script will exit immediately if any command fails.
- **Checking Exit Codes:** You can manually check exit codes (`$?` in bash/sh, `$LASTEXITCODE` in PowerShell) if you need finer-grained error handling within a script, often combined with `continue-on-error: true` for the step (see section E.7).

#### [Troubleshooting Section: Debugging failed shell commands]

When a `run` step fails, debugging involves:

1.  **Examine Logs:** Carefully read the workflow logs in the GitHub Actions UI. The output (stdout and stderr) of the failed command is usually printed just before the error message.
2.  **Add Debug Output:** Insert `echo` statements or commands to print variable values, directory contents (`ls`, `dir`), or environment variables (`env`, `printenv`) within your `run` script to understand the state just before the failure.
3.  **Run with Verbose Flags:** For shell scripts, add `set -x` (bash/sh) at the beginning of your multi-line script. This prints each command before it's executed, showing expansions and substitutions.
4.  **Check Exit Codes:** If a command might fail non-critically, check its exit code explicitly in the script (e.g., `command || echo "Command failed but continuing"`).
5.  **Simplify:** Temporarily simplify the failing command or script to isolate the problem.
6.  **Reproduce Locally (if possible):** If using standard tools, try running the same commands in a local environment matching the runner OS (or a Docker container based on the runner image) to see if the issue is environment-specific.

### 3. Using Actions (`uses`)

Instead of writing shell commands for everything, you can leverage pre-built, reusable units of code called _Actions_. Actions encapsulate complex tasks, interact with the GitHub API, or set up specific tools. The `uses` keyword specifies which action to run.

Actions are identified by a string in the format `{owner}/{repo}@{ref}` or `./path/to/local/action`.

#### a. Specifying Action Versions (SHA, Tag, Branch)

- `{owner}/{repo}@{ref}`:
  - **SHA:** `actions/checkout@a5ac7e51b41094c92402da3b24376905380afc29` (Full commit SHA - **Most Secure & Stable**)
  - **Tag:** `actions/checkout@v4` (Points to a specific release tag, e.g., v4.1.1 - Good balance)
  - **Branch:** `actions/checkout@main` (Points to the latest commit on a branch - Use with caution, potentially unstable)
- `./path/to/local/action`: Runs an action defined within your own repository (covered in Part III).
- `docker://image:tag`: Runs a Docker container action from Docker Hub.
- `docker://ghcr.io/owner/image:tag`: Runs a Docker container action from GitHub Container Registry.

##### [Production Note: Best practices for action version pinning for stability and security]

- **Pin to SHA:** For maximum stability and security, pin actions to a specific commit SHA. This guarantees you are always running the exact same code and protects against tag mutation or malicious code being injected into a branch. The downside is you won't automatically get updates. Dependabot can help manage SHA updates.
- **Pin to Major Version Tag (e.g., `v4`):** A common compromise is pinning to a major version tag (like `v4`). This allows you to receive non-breaking bug fixes and feature updates within that major version (e.g., v4.1.1 -> v4.1.2) automatically. However, it relies on the action maintainer following semantic versioning correctly.
- **Avoid Branches (`main`, `master`):** Using a branch reference like `@main` is generally discouraged for production workflows. The code on the branch can change at any time, potentially introducing breaking changes or instabilities without warning. Use branches only for testing development versions of actions.

#### b. Passing Inputs to Actions (`with`)

Actions often require or accept input parameters to customize their behavior. These are provided using the `with` keyword, which takes a map of key-value pairs. The available inputs are defined by the action itself in its `action.yml` file.

```yaml
steps:
  - name: Setup Node.js environment
    uses: actions/setup-node@v4
    with:
      node-version: "20" # Input 'node-version' set to '20'
      cache: "npm" # Input 'cache' set to 'npm'
```

#### c. Accessing Outputs from Actions (`steps.<step_id>.outputs`)

Actions can also produce output values that subsequent steps in the same job can use. To access an output, the step using the action must have an `id`, and the subsequent step references it using the expression syntax `${{ steps.<step_id>.outputs.<output_name> }}`.

```yaml
jobs:
  outputs_example:
    runs-on: ubuntu-latest
    steps:
      - name: Generate value
        id: generator # Assign an ID to this step
        run: echo "my_output_value=HelloFromAction" >> $GITHUB_OUTPUT

      - name: Use value
        run: echo "The generated value was: ${{ steps.generator.outputs.my_output_value }}"
```

_Note: The example above uses a `run` step to simulate an action producing an output via `$GITHUB_OUTPUT`. Real actions define outputs in their `action.yml` and set them internally._

##### [Practical Example: Using `actions/checkout` and `actions/setup-node`]

This is a very common pattern for Node.js projects:

```yaml
# .github/workflows/node-ci.yml
name: Node.js CI

on: [push, pull_request]

jobs:
  build_and_test:
    runs-on: ubuntu-latest
    steps:
      # Step 1: Check out the repository code
      - name: Checkout Code
        uses: actions/checkout@v4
        # No 'with' needed for basic checkout

      # Step 2: Setup Node.js environment
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: "18.x" # Specify Node.js version
          cache: "npm" # Enable caching for npm dependencies

      # Step 3: Install dependencies
      - name: Install Dependencies
        run: npm ci # Use 'ci' for clean installs in CI

      # Step 4: Run tests
      - name: Run Tests
        run: npm test

      # Step 5: Build project (if applicable)
      - name: Build Project
        run: npm run build --if-present # Run build script if it exists
```

This workflow checks out the code, sets up the specified Node.js version (caching dependencies for speed), installs packages using `npm ci`, runs tests, and finally runs a build script if one is defined in `package.json`.

### 4. Naming Steps (`name`) and Setting IDs (`id`)

- `name`: Provides a human-readable name for the step, displayed in the GitHub Actions UI logs. This makes it much easier to follow the workflow's progress and identify where failures occur. If omitted, GitHub displays the text from the `uses` or `run` key.
- `id`: Assigns a unique identifier to the step within the job. This ID is primarily used to reference the step's outputs (`steps.<id>.outputs`) or its outcome/conclusion (`steps.<id>.outcome`, `steps.<id>.conclusion`) in conditional expressions (`if:`) in subsequent steps.

```yaml
steps:
  - name: Step with an ID
    id: my_step_id
    run: echo "This step has an ID"

  - name: Step using the ID's outcome
    if: steps.my_step_id.conclusion == 'success'
    run: echo "my_step_id succeeded!"
```

### 5. Conditional Execution of Steps (`if`)

Similar to jobs, individual steps can be executed conditionally using the `if` keyword. The condition is evaluated just before the step runs. If it evaluates to `false`, the step is skipped. This allows for dynamic behavior within a single job.

Common use cases include:

- Running a step only on a specific branch.
- Running a step only if a previous step succeeded or failed.
- Running a step based on the event type (`github.event_name`).
- Running a step based on manual inputs (`inputs.*`).

```yaml
steps:
  - name: Always run this
    run: echo "Executing step 1"

  - name: Run only on main branch
    if: github.ref == 'refs/heads/main'
    run: echo "This only runs on main"

  - name: Check previous step status
    id: check_status
    run: exit 1 # Simulate failure
    continue-on-error: true # Allow workflow to continue

  - name: Run if check_status failed
    if: steps.check_status.conclusion == 'failure'
    run: echo "Handling the failure of check_status step."
```

### 6. Environment Variables (`env`)

You can set custom environment variables for use within your workflow steps using the `env` keyword.

#### a. Workflow, Job, and Step Level Environment Variables

Environment variables can be defined at three levels:

- **Workflow Level:** Defined at the top level of the workflow file. Available to all jobs and all steps within those jobs.
- **Job Level:** Defined within a specific job. Available to all steps within that job. Overrides workflow-level variables with the same name.
- **Step Level:** Defined within a specific step. Only available to that particular step. Overrides job and workflow-level variables with the same name.

```yaml
name: Environment Variables Demo

on: workflow_dispatch

env:
  WORKFLOW_VAR: "I am defined at the workflow level"
  SHARED_VAR: "Workflow version"

jobs:
  job1:
    runs-on: ubuntu-latest
    env:
      JOB_VAR: "I am defined at the job level for job1"
      SHARED_VAR: "Job1 version" # Overrides workflow SHARED_VAR
    steps:
      - name: Access env vars in job1
        env:
          STEP_VAR: "I am defined only for this step"
          SHARED_VAR: "Step version" # Overrides job and workflow SHARED_VAR
        run: |
          echo "WORKFLOW_VAR: $WORKFLOW_VAR"
          echo "JOB_VAR: $JOB_VAR"
          echo "STEP_VAR: $STEP_VAR"
          echo "SHARED_VAR: $SHARED_VAR" # Will print "Step version"

      - name: Access env vars in another step (job1)
        run: |
          echo "WORKFLOW_VAR: $WORKFLOW_VAR"
          echo "JOB_VAR: $JOB_VAR"
          # echo "STEP_VAR: $STEP_VAR" # This would be empty/error
          echo "SHARED_VAR: $SHARED_VAR" # Will print "Job1 version"

  job2:
    runs-on: ubuntu-latest
    steps:
      - name: Access env vars in job2
        run: |
          echo "WORKFLOW_VAR: $WORKFLOW_VAR"
          # echo "JOB_VAR: $JOB_VAR" # This would be empty/error
          echo "SHARED_VAR: $SHARED_VAR" # Will print "Workflow version"
```

#### b. Default Environment Variables

GitHub automatically sets several default environment variables for every workflow run. These provide useful context about the execution environment and the event that triggered the workflow. Some key examples include:

- `GITHUB_TOKEN`: An automatically generated token used to authenticate on behalf of GitHub Actions. Its permissions are typically limited to the repository containing the workflow. Can be used for API calls or interacting with the repository (e.g., by `actions/checkout`).
- `GITHUB_WORKFLOW`: The name of the workflow.
- `GITHUB_RUN_ID`: A unique number for each run of a particular workflow in a repository.
- `GITHUB_RUN_NUMBER`: A unique number for each run of a particular workflow in a repository, incrementing with each run.
- `GITHUB_ACTION`: The name of the action currently running, or the step `id`.
- `GITHUB_ACTOR`: The username of the user that triggered the workflow run.
- `GITHUB_REPOSITORY`: The owner and repository name (e.g., `octocat/Hello-World`).
- `GITHUB_EVENT_NAME`: The name of the event that triggered the workflow (e.g., `push`, `pull_request`).
- `GITHUB_SHA`: The commit SHA that triggered the workflow.
- `GITHUB_REF`: The branch or tag ref that triggered the workflow (e.g., `refs/heads/main`, `refs/tags/v1.0`).
- `GITHUB_WORKSPACE`: The default working directory on the runner where the repository code is checked out (e.g., `/home/runner/work/my-repo-name/my-repo-name`).
- `RUNNER_OS`: The operating system of the runner (e.g., `Linux`, `Windows`, `macOS`).
- `CI`: Always set to `true`. Indicates the code is running in a CI environment.

Refer to the official GitHub Actions documentation for the complete list of default environment variables.

#### [Code Snippet: Defining and using custom environment variables]

```yaml
name: Using Custom Env Vars

on: push

env:
  GLOBAL_API_KEY: ${{ secrets.PRODUCTION_API_KEY }} # Using secrets is common

jobs:
  use_vars:
    runs-on: ubuntu-latest
    env:
      BUILD_CONFIG: "Release"
    steps:
      - name: Use Env Vars in Script
        env:
          LOCAL_MESSAGE: "Hello from step env!"
        run: |
          echo "Global API Key starts with: ${GLOBAL_API_KEY:0:4}..." # Access workflow env var
          echo "Build configuration: $BUILD_CONFIG" # Access job env var
          echo "Local message: $LOCAL_MESSAGE" # Access step env var
          echo "Accessing job env var via context: ${{ env.BUILD_CONFIG }}" # Alternative access method

      - name: Use Job Var in Action Input
        uses: actions/github-script@v7
        with:
          script: |
            console.log("Build config from env context:", process.env.BUILD_CONFIG)
            core.setOutput('config_used', process.env.BUILD_CONFIG)
        env:
          # Env vars are automatically available to actions
          # We could also explicitly pass:
          # ACTION_INPUT_CONFIG: ${{ env.BUILD_CONFIG }}
```

### 7. Timeouts and Error Handling (`timeout-minutes`, `continue-on-error`)

You can control how long jobs and steps are allowed to run and how failures are handled.

- `timeout-minutes` (Job Level): Specifies the maximum number of minutes a job can run before GitHub automatically cancels it. The default is 360 minutes (6 hours). Set a lower value to prevent runaway jobs from consuming excessive runner time.
- `timeout-minutes` (Step Level): Specifies the maximum number of minutes a single step can run before being cancelled. This applies only to the individual step, not the entire job.
- `continue-on-error` (Step Level): If set to `true`, a failure in this step (non-zero exit code) will _not_ cause the job to fail. The step itself will be marked with a warning/failure status, but subsequent steps in the job will still execute. This is useful for non-critical steps (like uploading optional test reports) or when you want to perform cleanup actions even if the main task failed.

```yaml
jobs:
  resilient_job:
    runs-on: ubuntu-latest
    timeout-minutes: 15 # Max runtime for the entire job
    steps:
      - name: Critical Step 1
        run: ./run_critical_task.sh
        timeout-minutes: 5 # Max runtime for this specific step

      - name: Optional Reporting Step
        run: ./generate_report.sh || echo "Report generation failed but continuing"
        continue-on-error: true # Job continues even if this step fails

      - name: Always Run Cleanup
        # This step runs regardless of whether previous steps passed or failed
        # (as long as the job wasn't cancelled by timeout or a non-continue-on-error failure)
        if: always() # Use the always() status check function
        run: echo "Performing cleanup..."
```

---

This chapter has dissected the core components and syntax of a GitHub Actions workflow file. You now understand how workflows are triggered (`on`), how work is organized into parallel or sequential `jobs`, the role of `runners`, and how individual `steps` execute commands (`run`) or use pre-built `actions` (`uses`). We've also covered essential concepts like conditional execution (`if`), environment variables (`env`), and basic error handling.

With this foundational knowledge of the workflow anatomy, you are ready to explore the powerful expression syntax and context variables that enable dynamic and intelligent automation, which is the focus of Chapter 3.
