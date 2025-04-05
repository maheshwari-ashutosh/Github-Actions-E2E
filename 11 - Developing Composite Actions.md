# Chapter 11: Developing Composite Actions

Welcome to Chapter 11! In the previous chapters exploring custom action development, we delved into creating actions using JavaScript (Chapter 9) and Docker containers (Chapter 10). These methods provide powerful ways to encapsulate complex logic and manage dependencies. However, there are many scenarios where you simply want to reuse a sequence of common workflow steps without the overhead of building a container image or managing Node.js dependencies. This is precisely where **Composite Actions** shine.

Composite actions offer a lightweight yet effective way to bundle multiple workflow steps into a single, reusable action defined directly within an `action.yml` file. They allow you to combine shell scripts and, importantly, even call other actions, creating streamlined and maintainable workflows. Think of them as shareable functions or macros for your CI/CD processes.

This chapter provides an in-depth exploration of composite actions. We'll cover their core concepts, how to define them, handle inputs and outputs, understand their limitations, identify ideal use cases, and discuss testing strategies. By the end, you'll be equipped to create your own composite actions to reduce duplication and improve the clarity of your GitHub Actions workflows.

## A. Concept: Reusing Sequences of Steps

At its heart, workflow automation often involves repetitive sequences of tasks. Consider common CI/CD patterns:

1.  **Setup:** Check out code, set up a specific language runtime (Node.js, Python, Go, Java), install dependencies, configure authentication.
2.  **Build:** Compile code, package artifacts, build container images.
3.  **Test:** Run unit tests, integration tests, linting checks.
4.  **Deploy:** Push artifacts to a registry, deploy to staging/production environments.

You might find yourself copying and pasting the _exact same set_ of `steps` across multiple workflows in your repository or even across different repositories. For instance, the steps to set up Go, check out code, and run `go build` might appear in workflows for building binaries, running tests, and generating documentation.

This duplication leads to several problems:

- **Maintenance Burden:** If a step needs updating (e.g., changing a tool version, modifying a command flag), you have to find and update it in every workflow where it's used.
- **Inconsistency:** It's easy for copies to diverge over time, leading to subtle differences in how tasks are executed across workflows.
- **Reduced Readability:** Workflows become longer and harder to understand, obscuring the high-level process with low-level details.

Composite actions directly address this by allowing you to encapsulate such a sequence of steps into a single, reusable unit. Instead of copying multiple steps, you simply call your composite action using a single `uses:` statement.

```mermaid
graph TD
    subgraph Workflow A (Build)
        A1[Checkout Code] --> A2[Setup Go] --> A3[Install Deps] --> A4[Build Binary] --> A5[Upload Artifact]
    end

    subgraph Workflow B (Test)
        B1[Checkout Code] --> B2[Setup Go] --> B3[Install Deps] --> B4[Run Tests]
    end

    subgraph Workflow C (Lint)
        C1[Checkout Code] --> C2[Setup Go] --> C3[Install Deps] --> C4[Run Linter]
    end

    subgraph "Problem: Repetitive Steps"
        style A1 fill:#f9d,stroke:#333,stroke-width:2px
        style A2 fill:#f9d,stroke:#333,stroke-width:2px
        style A3 fill:#f9d,stroke:#333,stroke-width:2px
        style B1 fill:#f9d,stroke:#333,stroke-width:2px
        style B2 fill:#f9d,stroke:#333,stroke-width:2px
        style B3 fill:#f9d,stroke:#333,stroke-width:2px
        style C1 fill:#f9d,stroke:#333,stroke-width:2px
        style C2 fill:#f9d,stroke:#333,stroke-width:2px
        style C3 fill:#f9d,stroke:#333,stroke-width:2px
    end

    subgraph "Solution: Composite Action"
        CA[Composite Action: Setup & Prepare]
        style CA fill:#cfc,stroke:#333,stroke-width:2px
        CA --> |Contains| Steps(Checkout Code, Setup Go, Install Deps)
    end

    subgraph Workflow A' (Build - Using Composite)
        A1_C[uses: ./actions/setup-prepare] --> A4'[Build Binary] --> A5'[Upload Artifact]
        style A1_C fill:#cfc,stroke:#333,stroke-width:2px
    end

    subgraph Workflow B' (Test - Using Composite)
        B1_C[uses: ./actions/setup-prepare] --> B4'[Run Tests]
        style B1_C fill:#cfc,stroke:#333,stroke-width:2px
    end

    subgraph Workflow C' (Lint - Using Composite)
        C1_C[uses: ./actions/setup-prepare] --> C4'[Run Linter]
        style C1_C fill:#cfc,stroke:#333,stroke-width:2px
    end

```

- **Diagram Explanation:** This diagram illustrates the problem of duplicated setup steps (highlighted in red) across multiple workflows. It then shows how a composite action (highlighted in green) encapsulates these common steps, allowing each workflow to call the action with a single `uses:` statement, leading to cleaner and more maintainable workflows.

## B. Defining the Action (`action.yml`)

Like JavaScript and Docker actions, composite actions are defined using an `action.yml` metadata file. This file describes the action's inputs, outputs, and crucially, the sequence of steps it will execute.

The basic structure of an `action.yml` for a composite action looks like this:

```yaml
# .github/actions/my-composite-action/action.yml
name: 'My Composite Action'
description: 'A brief description of what this action does.'

# Define inputs the action accepts (optional)
inputs:
  input-parameter:
    description: 'Description of the input.'
    required: false
    default: 'default-value'

# Define outputs the action produces (optional)
outputs:
  output-result:
    description: 'Description of the output.'

# Specify that this is a composite action and define its steps
runs:
  using: 'composite'
  steps:
    # Sequence of steps to execute
    - name: First Step
      run: echo "Executing the first step..."
      shell: bash

    - name: Second Step using Input
      run: echo "Received input: ${{ inputs.input-parameter }}"
      shell: bash

    # Steps can also use other actions
    - name: Use Another Action
      uses: actions/setup-node@v4
      with:
        node-version: '20'

    # Step that potentially sets an output
    - name: Set Output Step
      id: set_output_step # Give the step an ID to reference its outputs
      run: echo "result=some-value" >> $GITHUB_OUTPUT
      shell: bash
```

Let's break down the key components specific to composite actions.

### B.1. Specifying `using: 'composite'`

The most critical part of defining a composite action is the `runs:` section. Within `runs:`, you **must** specify:

```yaml
runs:
  using: "composite"
  # ... steps defined below
```

The `using: 'composite'` declaration tells the GitHub Actions runner that this `action.yml` defines a sequence of steps to be executed directly, rather than pointing to a JavaScript file (`using: 'node16'` or `using: 'node20'`) or a Dockerfile/image (`using: 'docker'`).

### B.2. Defining the `steps:` within the Action

Nested under the `runs:` key (alongside `using: 'composite'`), you define a `steps:` list. This list contains the sequence of individual steps that the composite action will execute when called by a workflow.

```yaml
runs:
  using: "composite"
  steps:
    - name: Checkout Code # Example Step 1
      uses: actions/checkout@v4

    - name: Setup Go Environment # Example Step 2
      uses: actions/setup-go@v5
      with:
        go-version: "1.21" # This could use an input

    - name: Run a script # Example Step 3
      run: |
        echo "Running custom script..."
        ./scripts/my-script.sh
      shell: bash

    - name: Another Script Step # Example Step 4
      id: step_four # Give this step an ID
      run: echo "Processing complete."
      shell: bash
```

Key points about the `steps:` block in a composite action:

- **Structure:** It follows the same syntax as the `steps:` block within a workflow job. Each item in the list is a step object.
- **Step Types:** You can include steps that:
  - Run shell commands (`run:`).
  - Use other actions (`uses:`), including standard actions (`actions/checkout`, `actions/setup-node`), third-party actions, and even _other composite actions_ from the same repository. This is a powerful feature for building layered abstractions.
- **Execution Environment:** The steps run directly on the runner machine assigned to the job that _calls_ the composite action. They share the same environment, filesystem, and context (with some nuances discussed later) as other steps in the calling job.
- **`id`:** You can assign an `id` to steps within the composite action if you need to reference their outputs later (e.g., to map them to the composite action's overall outputs).

```mermaid
graph LR
    subgraph Workflow Job Runner
        WF_Step1[Workflow Step 1] --> CA_Call{uses: ./my-composite-action} --> WF_Step3[Workflow Step 3]

        subgraph CA_Call [Composite Action Execution]
            direction TB
            Metadata[action.yml<br/>(using: composite)] -- Defines --> StepsList(steps:)
            StepsList --> CompStep1[Step 1: uses: actions/checkout@v4]
            CompStep1 --> CompStep2[Step 2: run: echo Hello]
            CompStep2 --> CompStep3[Step 3: uses: actions/setup-node@v4]
            CompStep3 --> CompStep4[Step 4: run: npm ci]
        end
    end

    style CA_Call fill:#eee,stroke:#333,stroke-width:1px
```

- **Diagram Explanation:** This diagram shows how a workflow job executes its steps. When it encounters a `uses:` statement calling a composite action, the runner essentially injects the sequence of steps defined within that composite action's `action.yml` directly into the job's execution flow on the same runner.

## C. Handling Inputs and Outputs

Just like JavaScript and Docker actions, composite actions can be made more flexible and useful by defining inputs and outputs.

- **Inputs:** Allow workflows calling the action to pass in data (e.g., versions, paths, flags).
- **Outputs:** Allow the composite action to return results or information (e.g., generated file paths, test summaries, status flags) back to the calling workflow.

Inputs and outputs are defined in the top-level `inputs:` and `outputs:` sections of the `action.yml`, respectively.

### C.1. Accessing Action Inputs within Composite Steps (`inputs.<input_name>`)

Inputs defined in the `inputs:` section of the `action.yml` are made available to the steps _within_ the composite action via the `inputs` context. You access a specific input using the expression syntax `${{ inputs.<input_name> }}`.

**Example `action.yml` defining and using an input:**

```yaml
# .github/actions/greet-user/action.yml
name: "Greet User"
description: "Greets a specified user."

inputs:
  user-name:
    description: "The name of the user to greet."
    required: true
    default: "World" # Default if not provided

runs:
  using: "composite"
  steps:
    - name: Greet the specified user
      run: echo "Hello, ${{ inputs.user-name }}!" # Accessing the input
      shell: bash
```

**Example workflow using this action:**

```yaml
# .github/workflows/greeting.yml
name: Greeting Workflow

on: push

jobs:
  say-hello:
    runs-on: ubuntu-latest
    steps:
      - name: Greet Alice
        uses: ./.github/actions/greet-user # Assuming action is in this path
        with:
          user-name: "Alice" # Passing the input value

      - name: Greet Bob (uses default)
        uses: ./.github/actions/greet-user
        with:
          user-name: "Bob" # Passing the input value
```

In this example, the `greet-user` composite action takes a `user-name` input. Inside its `run` step, `${{ inputs.user-name }}` resolves to the value provided in the `with:` block of the calling workflow (`'Alice'` or `'Bob'`).

### C.2. Mapping Step Outputs to Action Outputs

Composite actions don't automatically expose the outputs of their internal steps. You need to explicitly map the outputs generated by individual steps _within_ the composite action to the outputs defined in the top-level `outputs:` section of the `action.yml`.

To do this:

1.  **Define the Output:** Declare the output name and description in the main `outputs:` block of the `action.yml`.
2.  **Generate Output in a Step:** Have a step inside the `steps:` block generate an output. This typically involves:
    - Giving the step an `id` (e.g., `id: my_step`).
    - Using `echo "{output_name}={value}" >> $GITHUB_OUTPUT` within a `run` command to set the output.
3.  **Map the Output:** In the main `outputs:` block, set the `value` for your defined output to `${{ steps.<step_id>.outputs.<output_name> }}`.

#### [Code Snippet: Composite action wrapping setup, build, and test steps]

Let's create a composite action that sets up Go, builds a simple binary, runs a placeholder test, and outputs the path to the binary and a test status.

**Action Definition (`.github/actions/go-build-test/action.yml`):**

```yaml
name: "Go Build and Test"
description: "Sets up Go, builds a binary, runs tests, and outputs results."

inputs:
  go-version:
    description: "Version of Go to install (e.g., 1.21)"
    required: false
    default: "1.21"
  package-path:
    description: "Path to the Go package to build (relative to workspace root)"
    required: true
  output-binary-name:
    description: "Name for the output binary file"
    required: false
    default: "app"

outputs:
  binary-path:
    description: "The path to the built binary artifact."
    value: ${{ steps.build.outputs.artifact_path }} # Map from 'build' step output
  test-result:
    description: "Result status of the test execution (e.g., success/failure)."
    value: ${{ steps.test.outputs.status }} # Map from 'test' step output

runs:
  using: "composite"
  steps:
    - name: Setup Go
      uses: actions/setup-go@v5
      with:
        go-version: ${{ inputs.go-version }}

    - name: Build Go Binary
      id: build # Give this step an ID
      shell: bash
      run: |
        cd ${{ inputs.package-path }}
        echo "Building Go binary..."
        go build -o ${{ inputs.output-binary-name }}
        # Set the output for this step
        echo "artifact_path=$(pwd)/${{ inputs.output-binary-name }}" >> $GITHUB_OUTPUT
        echo "Built binary at $(pwd)/${{ inputs.output-binary-name }}"

    - name: Run Placeholder Tests
      id: test # Give this step an ID
      shell: bash
      run: |
        echo "Running tests (placeholder)..."
        # Simulate test success/failure (replace with actual test command)
        TEST_PASSED=true
        if $TEST_PASSED; then
          echo "status=success" >> $GITHUB_OUTPUT
          echo "Tests passed!"
        else
          echo "status=failure" >> $GITHUB_OUTPUT
          echo "Tests failed!"
          exit 1 # Fail the step if tests fail
        fi
```

**Example Workflow Using the Action (`.github/workflows/build-my-app.yml`):**

```yaml
name: Build My Go App

on: push

jobs:
  build-and-test:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout Code
        uses: actions/checkout@v4

      - name: Build and Test Go App
        id: build_test_action # Give the action step an ID in the workflow
        uses: ./.github/actions/go-build-test # Use the composite action
        with:
          go-version: "1.21.5"
          package-path: "./cmd/myapp" # Path to the main package
          output-binary-name: "my-application"

      - name: Show Outputs
        shell: bash
        run: |
          echo "Composite action finished."
          echo "Binary Path Output: ${{ steps.build_test_action.outputs.binary-path }}"
          echo "Test Result Output: ${{ steps.build_test_action.outputs.test-result }}"

      # Example: Upload the binary only if tests passed
      - name: Upload Binary Artifact
        if: steps.build_test_action.outputs.test-result == 'success'
        uses: actions/upload-artifact@v4
        with:
          name: ${{ inputs.output-binary-name }} # Use the same name
          path: ${{ steps.build_test_action.outputs.binary-path }}
```

In this example:

1.  The `go-build-test` action defines `binary-path` and `test-result` outputs.
2.  The `Build Go Binary` step (`id: build`) calculates the binary path and uses `$GITHUB_OUTPUT` to declare an output named `artifact_path`.
3.  The `Run Placeholder Tests` step (`id: test`) determines a status and uses `$GITHUB_OUTPUT` to declare an output named `status`.
4.  The top-level `outputs:` section maps `outputs.binary-path` to `steps.build.outputs.artifact_path` and `outputs.test-result` to `steps.test.outputs.status`.
5.  The calling workflow (`build-my-app.yml`) calls the composite action, gives that step an `id` (`build_test_action`), and can then access the composite action's outputs using `${{ steps.build_test_action.outputs.binary-path }}` and `${{ steps.build_test_action.outputs.test-result }}`.

## D. Shell Selection (`defaults.run.shell`)

By default, `run` steps within a composite action (and workflows) execute using the runner's default shell (usually `bash` on Linux/macOS, `pwsh` on Windows). However, you can specify a different default shell for _all_ `run` steps within your composite action using the `defaults.run.shell` key within the `runs:` block.

This is useful if your action primarily consists of scripts written for a specific shell (e.g., PowerShell, Python).

```yaml
# .github/actions/powershell-composite/action.yml
name: "PowerShell Composite Action"
description: "Runs steps using PowerShell Core by default."

inputs:
  computer-name:
    description: "Computer name to check."
    required: true

outputs:
  ping-status:
    description: "Result of pinging the computer."
    value: ${{ steps.ping_step.outputs.status }}

runs:
  using: "composite"
  defaults:
    run:
      shell: pwsh # Set default shell for all 'run' steps to PowerShell Core
  steps:
    - name: Check Computer Status
      id: ping_step
      # This script runs using pwsh because of the default setting
      run: |
        if (Test-Connection -ComputerName "${{ inputs.computer-name }}" -Count 1 -Quiet) {
          Write-Host "Computer '${{ inputs.computer-name }}' is reachable."
          # Use environment file syntax for pwsh
          echo "status=reachable" >> $env:GITHUB_OUTPUT
        } else {
          Write-Host "Computer '${{ inputs.computer-name }}' is unreachable."
          echo "status=unreachable" >> $env:GITHUB_OUTPUT
        }

    - name: Another PowerShell Step
      # No need to specify shell: pwsh here
      run: Get-Process | Sort-Object CPU -Descending | Select-Object -First 5
```

You can still override the default shell for individual steps by specifying the `shell:` key directly on that step:

```yaml
# ... inside runs: ... steps: ...
- name: Run a Bash command specifically
  run: echo "This runs in Bash!"
  shell: bash # Overrides the pwsh default for this step only
```

Common shell options include: `bash`, `pwsh`, `python`, `sh`, `cmd`, `powershell`.

## E. Limitations of Composite Actions

While powerful and convenient, composite actions have historically had some limitations compared to JavaScript or Docker actions. It's crucial to be aware of these, although the platform evolves, and limitations can be lifted.

### E.1. Cannot Use `uses:` within Composite Steps (Currently - check latest docs) [Note: Verify latest feature status]

**Update:** As of recent GitHub Actions updates, this limitation has been **lifted**. Composite actions **can now use other actions** (`uses:`) within their `steps:` block.

This was a significant previous limitation, forcing composite actions to rely solely on `run` commands for their logic. Now, you can freely compose actions by calling other existing actions (standard, third-party, or even other composite actions) from within your composite action's steps.

**Example (Now Valid):**

```yaml
# .github/actions/setup-and-build/action.yml
name: "Setup Node and Build Project"
description: "Checks out code, sets up Node.js, installs deps, and builds."

inputs:
  node-version:
    description: "Node.js version"
    default: "20"

runs:
  using: "composite"
  steps:
    # Using actions/checkout within a composite action
    - name: Checkout repository code
      uses: actions/checkout@v4

    # Using actions/setup-node within a composite action
    - name: Setup Node.js
      uses: actions/setup-node@v4
      with:
        node-version: ${{ inputs.node-version }}
        cache: "npm" # Example of using action features

    - name: Install Dependencies
      run: npm ci
      shell: bash

    - name: Build Project
      run: npm run build
      shell: bash
```

This ability makes composite actions significantly more powerful for creating reusable workflows that leverage the existing Actions ecosystem.

### E.2. Context Availability

Composite actions execute directly on the runner as part of the calling job. They generally have access to the same contexts as any other step in that job, including:

- `github`: Information about the repository, event payload, etc.
- `env`: Workflow-level and job-level environment variables.
- `job`: Information about the current job.
- `steps`: Outputs from _previous steps within the composite action itself_.
- `runner`: Information about the runner environment.
- `secrets`: Access to secrets (though passing secrets _into_ a composite action requires careful handling via inputs).
- `strategy`, `matrix`: If the calling job uses a matrix strategy.
- `needs`: Outputs from jobs the current job depends on.

However, there's a key distinction regarding the `inputs` context:

- **Inside the composite action's `steps:`:** `${{ inputs.<input_name> }}` refers to the inputs defined in the _composite action's_ `action.yml`.
- **Accessing workflow inputs:** A composite action _cannot_ directly access the inputs of the _calling workflow_ (defined under `on.workflow_call.inputs`). If a composite action needs data that originated as a workflow input, that data must be explicitly passed _into_ the composite action via its own defined inputs.

Similarly, while composite actions can access most environment variables, variables set directly in the `env:` block of the _calling workflow_ or _job_ are available, but variables set using `echo "VAR_NAME=value" >> $GITHUB_ENV` in a _previous step in the calling workflow_ (before the composite action is called) are also accessible.

## F. Use Cases for Composite Actions

Composite actions excel in scenarios where you need to reuse sequences of shell commands or standard actions without the complexity of JavaScript or Docker.

### F.1. Simplifying Repetitive Setup Tasks

This is one of the most common and effective uses. Instead of copying steps for checking out code, setting up language runtimes, installing dependencies, or configuring cloud credentials in every workflow, encapsulate them in a composite action.

**Example: AWS Credentials Setup Action**

```yaml
# .github/actions/configure-aws-creds/action.yml
name: "Configure AWS Credentials"
description: "Configures AWS credentials using OIDC or access keys."

inputs:
  aws-role-to-assume:
    description: "IAM Role ARN to assume via OIDC."
    required: false
  aws-region:
    description: "AWS Region."
    required: true
  aws-access-key-id:
    description: "AWS Access Key ID (use secrets!). Less preferred than OIDC."
    required: false
  aws-secret-access-key:
    description: "AWS Secret Access Key (use secrets!). Less preferred than OIDC."
    required: false

runs:
  using: "composite"
  steps:
    - name: Configure AWS Credentials using OIDC
      if: inputs.aws-role-to-assume != ''
      uses: aws-actions/configure-aws-credentials@v4
      with:
        role-to-assume: ${{ inputs.aws-role-to-assume }}
        aws-region: ${{ inputs.aws-region }}

    - name: Configure AWS Credentials using Keys
      if: inputs.aws-role-to-assume == '' && inputs.aws-access-key-id != ''
      uses: aws-actions/configure-aws-credentials@v4
      with:
        aws-access-key-id: ${{ inputs.aws-access-key-id }}
        aws-secret-access-key: ${{ inputs.aws-secret-access-key }}
        aws-region: ${{ inputs.aws-region }}

    - name: Check Credentials (Optional)
      shell: bash
      run: aws sts get-caller-identity
```

Workflows can now simply use `uses: ./.github/actions/configure-aws-creds` with appropriate inputs, abstracting the details of the `aws-actions/configure-aws-credentials` action and providing a consistent setup method.

### F.2. Creating High-Level Abstractions for Common Processes

Composite actions allow you to build higher-level "verbs" specific to your project or organization's processes. Instead of thinking in terms of individual `setup-go`, `go build`, `go test` steps, you can create a single action like `build-and-test-go-service`.

#### [Practical Example: Composite action to build and publish a Go binary]

Let's expand on the previous Go example to create a more complete action that builds a Go binary and prepares it for publishing (e.g., by placing it in a specific directory structure or creating a checksum). We won't include the actual publishing step here (as that often involves secrets and specific destinations better handled in the workflow), but we'll prepare the artifact.

**Action Definition (`.github/actions/build-go-binary/action.yml`):**

```yaml
name: "Build Go Binary for Release"
description: "Builds a Go binary, creates a checksum, and prepares it in a dist folder."

inputs:
  go-version:
    description: "Version of Go to install"
    required: false
    default: "1.21"
  package-path:
    description: "Path to the Go package to build (relative to workspace root)"
    required: true
    default: "."
  output-binary-name:
    description: "Name for the output binary file"
    required: true
  target-os:
    description: "Target OS for cross-compilation (e.g., linux, windows, darwin)"
    required: false
    default: "linux"
  target-arch:
    description: "Target architecture (e.g., amd64, arm64)"
    required: false
    default: "amd64"

outputs:
  dist-path:
    description: "Path to the directory containing the binary and checksum."
    value: ${{ steps.prepare_dist.outputs.dist_dir }}
  binary-path:
    description: "Path to the built binary within the dist directory."
    value: ${{ steps.prepare_dist.outputs.bin_path }}
  checksum-path:
    description: "Path to the checksum file within the dist directory."
    value: ${{ steps.prepare_dist.outputs.checksum_path }}

runs:
  using: "composite"
  steps:
    - name: Setup Go
      uses: actions/setup-go@v5
      with:
        go-version: ${{ inputs.go-version }}

    - name: Build Go Binary
      id: build
      shell: bash
      env:
        GOOS: ${{ inputs.target-os }}
        GOARCH: ${{ inputs.target-arch }}
      run: |
        echo "Building Go binary for $GOOS/$GOARCH..."
        cd ${{ inputs.package-path }}
        go build -v -o ${{ inputs.output-binary-name }} .
        echo "Build complete: $(pwd)/${{ inputs.output-binary-name }}"
        # Output the build directory for the next step
        echo "build_dir=$(pwd)" >> $GITHUB_OUTPUT
        echo "binary_name=${{ inputs.output-binary-name }}" >> $GITHUB_OUTPUT

    - name: Prepare Distribution Files
      id: prepare_dist
      shell: bash
      run: |
        BUILD_DIR="${{ steps.build.outputs.build_dir }}"
        BINARY_NAME="${{ steps.build.outputs.binary_name }}"
        DIST_DIR="${{ github.workspace }}/dist_${{ inputs.target-os }}_${{ inputs.target-arch }}" # Create unique dist dir
        BINARY_PATH="$DIST_DIR/$BINARY_NAME"
        CHECKSUM_PATH="$DIST_DIR/checksums.txt"

        echo "Creating distribution directory: $DIST_DIR"
        mkdir -p "$DIST_DIR"

        echo "Moving binary to $BINARY_PATH"
        mv "$BUILD_DIR/$BINARY_NAME" "$BINARY_PATH"

        echo "Generating checksum..."
        cd "$DIST_DIR"
        sha256sum "$BINARY_NAME" > checksums.txt
        cd "$OLDPWD" # Go back to previous directory

        echo "Distribution prepared in $DIST_DIR"
        echo "dist_dir=$DIST_DIR" >> $GITHUB_OUTPUT
        echo "bin_path=$BINARY_PATH" >> $GITHUB_OUTPUT
        echo "checksum_path=$CHECKSUM_PATH" >> $GITHUB_OUTPUT
```

**Example Workflow Using the Action (`.github/workflows/release.yml`):**

```yaml
name: Release Go Binary

on:
  push:
    tags:
      - "v*.*.*" # Trigger on version tags

jobs:
  build-release:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        goos: [linux, windows, darwin]
        goarch: [amd64, arm64]
        exclude: # Example: Exclude combinations if needed
          - goos: windows
            goarch: arm64
          - goos: darwin
            goarch: arm64 # Assuming no M1 runner/build support setup here

    steps:
      - name: Checkout Code
        uses: actions/checkout@v4

      - name: Build Go Binary using Composite Action
        id: build_binary
        uses: ./.github/actions/build-go-binary # Call the composite action
        with:
          package-path: "./cmd/mytool" # Adjust to your project structure
          output-binary-name: "mytool-${{ matrix.goos }}-${{ matrix.goarch }}" # Include platform in name
          target-os: ${{ matrix.goos }}
          target-arch: ${{ matrix.goarch }}

      - name: Upload Release Artifact
        uses: actions/upload-artifact@v4
        with:
          name: mytool-${{ matrix.goos }}-${{ matrix.goarch }} # Artifact name
          path: |
            ${{ steps.build_binary.outputs.binary-path }}
            ${{ steps.build_binary.outputs.checksum-path }}
          retention-days: 7 # Optional: How long to keep artifact

      # In a real release workflow, you might use outputs to:
      # - Create a GitHub Release (e.g., using softprops/action-gh-release)
      # - Upload assets to the release
      # - Push to other registries
      - name: Display Artifact Paths (for verification)
        run: |
          echo "Binary Path: ${{ steps.build_binary.outputs.binary-path }}"
          echo "Checksum Path: ${{ steps.build_binary.outputs.checksum-path }}"
          echo "Dist Path: ${{ steps.build_binary.outputs.dist-path }}"
```

This example demonstrates how a composite action can encapsulate a multi-step build and preparation process, making the main release workflow much cleaner and focused on the higher-level goal (building for multiple platforms and uploading artifacts). The workflow simply calls the action for each platform in the matrix.

## G. Testing Composite Actions

Since composite actions are essentially reusable parts of workflows, testing them involves invoking them within a workflow context.

### G.1. Testing within Workflows that Call the Action

The most common and practical way to test a composite action is to create one or more test workflows _within the same repository_ where the action is defined (or a dedicated testing repository that checks out the action's code).

**Strategy:**

1.  **Create Test Workflows:** Add new workflow files (e.g., `.github/workflows/test-composite-action.yml`) specifically designed to exercise your composite action.
2.  **Call the Action:** Use a `uses:` statement pointing to the local path of your action (e.g., `uses: ./path/to/your/action`).
3.  **Trigger Conditions:** Set up triggers for these test workflows (e.g., `on: push` to specific branches, `on: workflow_dispatch` for manual testing).
4.  **Test Scenarios:** Create different jobs or steps within the test workflow to cover various scenarios:
    - Test with different input values (required, optional, defaults).
    - Verify expected outputs are generated correctly.
    - Check for expected side effects (e.g., files created, commands executed successfully).
    - Test edge cases and potential failure conditions.
5.  **Assertions:** Use `run` steps with shell commands to assert expected outcomes. For example:
    - Check if an output file exists (`test -f <expected_file_path>`).
    - Compare an output value with an expected value (`if [ "${{ steps.my_action.outputs.result }}" != "expected" ]; then exit 1; fi`).
    - Check the exit code of the step calling the composite action (it should fail if an internal step fails).

**Example Test Workflow:**

```yaml
# .github/workflows/test-go-build-binary-action.yml
name: Test Go Build Binary Action

on:
  push:
    paths: # Trigger only if the action or test workflow changes
      - ".github/actions/build-go-binary/**"
      - ".github/workflows/test-go-build-binary-action.yml"
      - "cmd/mytool/**" # Or the source code the action builds
  workflow_dispatch: # Allow manual triggering

jobs:
  test-linux-amd64:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout Code
        uses: actions/checkout@v4

      - name: Run Build Action (Linux/amd64)
        id: build_linux
        uses: ./.github/actions/build-go-binary # Use local path
        with:
          package-path: "./cmd/mytool"
          output-binary-name: "mytool-test-linux-amd64"
          target-os: "linux"
          target-arch: "amd64"

      - name: Verify Outputs and Artifacts (Linux/amd64)
        shell: bash
        run: |
          echo "--- Verifying Outputs ---"
          echo "Dist Path: ${{ steps.build_linux.outputs.dist-path }}"
          echo "Binary Path: ${{ steps.build_linux.outputs.binary-path }}"
          echo "Checksum Path: ${{ steps.build_linux.outputs.checksum-path }}"

          echo "--- Verifying Files ---"
          if [ ! -f "${{ steps.build_linux.outputs.binary-path }}" ]; then
            echo "ERROR: Binary file not found!"
            exit 1
          fi
          if [ ! -f "${{ steps.build_linux.outputs.checksum-path }}" ]; then
            echo "ERROR: Checksum file not found!"
            exit 1
          fi

          echo "--- Verifying Checksum ---"
          cd $(dirname "${{ steps.build_linux.outputs.checksum-path }}")
          sha256sum -c checksums.txt || exit 1 # Verify checksum, exit if fails
          cd "$OLDPWD"

          echo "--- Verifying Binary Execution (Basic) ---"
          # Make executable if needed (might depend on Go build flags)
          chmod +x "${{ steps.build_linux.outputs.binary-path }}"
          # Run a basic command like --version, assumes the binary supports it
          "${{ steps.build_linux.outputs.binary-path }}" --version || exit 1

          echo "--- Linux/amd64 Test Passed ---"

  # Add more jobs for other platforms or input variations as needed
  # test-windows-amd64:
  #   runs-on: windows-latest
  #   steps:
  #     # ... similar steps, adjusting verification commands for Windows (pwsh)
```

This iterative process of developing the action and running test workflows helps ensure your composite action behaves as expected before using it in critical production workflows.

---

Composite actions provide a fantastic middle ground in custom action development. They offer much of the power of reusable steps without the packaging overhead of JavaScript or Docker actions, especially now that they can incorporate `uses:` steps themselves. By mastering composite actions, you can significantly reduce redundancy, improve maintainability, and create clearer, more abstract workflows. In the next chapter, we'll shift our focus to the crucial aspects of publishing, versioning, and maintaining the custom actions you've learned to build.
