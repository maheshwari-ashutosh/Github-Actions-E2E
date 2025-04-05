# Chapter 16: Managing Self-Hosted Runners

While GitHub-hosted runners offer convenience and simplicity for many workflow automation tasks, certain scenarios demand more control, specific environments, or access to private resources. This is where self-hosted runners come into play. Self-hosted runners are machines that you provision and manage – whether physical hardware, virtual machines, or containers – that connect to GitHub Actions and execute workflow jobs.

This chapter provides a comprehensive guide to understanding when to use self-hosted runners, how to set them up securely, manage them effectively at scale, and maintain their operational health. Mastering self-hosted runners unlocks advanced capabilities and allows you to tailor your CI/CD environment precisely to your needs.

## A. When to Use Self-Hosted Runners

GitHub-hosted runners are excellent general-purpose execution environments, but they operate within GitHub's infrastructure and have predefined configurations. Self-hosted runners become necessary or advantageous in several key situations:

#### 1. Accessing Private Networks/Resources

Perhaps the most common reason for using self-hosted runners is the need for workflows to interact with resources within a private network, such as databases, internal APIs, artifact repositories, or staging environments that are not exposed to the public internet.

```mermaid
graph LR
    subgraph GitHub Cloud
        A[GitHub Actions Workflow] --> B{Job Request};
    end
    subgraph Your Private Network
        C(Self-Hosted Runner) -- Executes Job --> D[Internal Database];
        C -- Executes Job --> E[Private API];
        C -- Executes Job --> F[Staging Server];
    end
    B -- Assigned To --> C;
    C -- Sends Logs/Status --> A;

    style Your Private Network fill:#f9f,stroke:#333,stroke-width:2px
```

> **Diagram Explanation:** This diagram illustrates how a GitHub Actions workflow job is assigned to a self-hosted runner located within a private network, allowing the job to securely access internal resources like databases or APIs.

#### 2. Using Custom Hardware (GPUs, Specific Architectures)

Workflows requiring specialized hardware, such as:

- **GPUs:** For machine learning model training/testing or graphics processing.
- **Specific CPU Architectures:** ARM, PowerPC, s390x, etc., for building and testing software targeting those platforms.
- **Hardware Security Modules (HSMs):** For sensitive cryptographic operations.
- **Specialized Peripherals:** Connected test devices or hardware simulators.

GitHub-hosted runners offer standard CPU architectures (primarily x64) and limited specialized options. Self-hosted runners allow you to use precisely the hardware your workflows demand.

#### 3. Meeting Strict Compliance Requirements

Organizations operating under stringent regulatory or compliance frameworks (e.g., FedRAMP, HIPAA, PCI-DSS) may have specific requirements regarding data locality, network isolation, security hardening, and auditing that cannot be fully met by GitHub-hosted runners. Self-hosted runners allow you to enforce these controls within your own managed environment.

#### 4. Cost Optimization (Potential Trade-offs)

While GitHub-hosted runners have associated costs (especially for private repositories or parallel jobs beyond the free tier), running your own infrastructure also incurs costs (hardware/VMs, electricity, maintenance, personnel time).

However, for organizations with:

- **Existing Infrastructure:** Leveraging underutilized on-premises hardware or existing cloud commitments.
- **Very High Usage:** Running a large volume of long-duration jobs where the per-minute cost of hosted runners becomes significant.
- **Predictable Workloads:** Where dedicated hardware can be more cost-effective than pay-as-you-go hosted runners.

Self-hosting _can_ offer cost savings, but it requires careful analysis of the total cost of ownership (TCO), including operational overhead.

#### 5. Needing Specific Software/OS Configurations Not Available on Hosted Runners

GitHub-hosted runners come pre-installed with a wide range of common software, but you might need:

- **Licensed Software:** Requiring specific licenses installed on the runner machine.
- **Legacy Systems:** Dependencies on older operating systems or specific library versions not available on standard runner images.
- **Custom Kernel Modules:** For specialized hardware interaction or performance tuning.
- **Pre-configured Environments:** Complex software stacks that are time-consuming to set up dynamically in each workflow run.

Self-hosted runners give you complete control over the operating system, installed software, and environment configuration.

## B. Setting up Self-Hosted Runners

Setting up a self-hosted runner involves installing the GitHub Actions runner agent software on your chosen machine and registering it with your GitHub repository or organization.

#### 1. Installation Process (Linux, Windows, macOS)

The process is similar across supported operating systems, primarily involving downloading the runner agent archive, extracting it, and running a configuration script.

**General Steps:**

1.  **Navigate to GitHub:** Go to the repository or organization settings where you want to add the runner.
    - **Repository:** `Settings` > `Actions` > `Runners` > `New self-hosted runner`.
    - **Organization:** `Settings` > `Actions` > `Runners` > `New runner`.
2.  **Choose OS & Architecture:** Select the target operating system (Linux, Windows, macOS) and architecture (x64, ARM, ARM64). GitHub will provide specific download and configuration commands.
3.  **Download the Agent:** Use `curl` or `wget` (Linux/macOS) or download manually (Windows) to get the latest runner agent package (`actions-runner-*.tar.gz` or `actions-runner-*.zip`).
4.  **Verify Hash (Optional but Recommended):** Check the SHA256 checksum of the downloaded file against the one provided by GitHub.
5.  **Extract the Agent:**
    - Linux/macOS: `tar xzf actions-runner-*.tar.gz`
    - Windows: Use File Explorer or PowerShell's `Expand-Archive`.
6.  **Configure the Agent:** Run the `config.sh` (Linux/macOS) or `config.cmd` (Windows) script. This script will prompt you for:
    - **GitHub URL:** The URL of your repository or organization (e.g., `https://github.com/your-org/your-repo` or `https://github.com/your-org`).
    - **Registration Token:** A short-lived token obtained from the GitHub UI (Step 1). **Treat this token like a password.**
    - **Runner Name:** A descriptive name for the runner (defaults to the hostname).
    - **Runner Group (Org Level):** Optionally assign the runner to a specific group (defaults to `Default`).
    - **Labels:** Assign custom labels (comma-separated) for targeting jobs (e.g., `linux,gpu,production`). The `self-hosted` label is automatically added.
    - **Work Folder:** The directory where the runner will execute jobs (defaults to `_work`).

**Example (Linux):**

```bash
# Create a directory for the runner
mkdir actions-runner && cd actions-runner

# Download the latest runner package (Replace URL with the one from GitHub UI)
curl -o actions-runner-linux-x64-2.3xx.x.tar.gz -L https://github.com/actions/runner/releases/download/v2.3xx.x/actions-runner-linux-x64-2.3xx.x.tar.gz

# Optional: Verify hash
# echo "<SHA256_checksum> actions-runner-linux-x64-2.3xx.x.tar.gz" | sha256sum -c

# Extract the installer
tar xzf ./actions-runner-linux-x64-2.3xx.x.tar.gz

# Create the runner and start the configuration experience (Replace URL and Token)
./config.sh --url https://github.com/your-org/your-repo --token YOUR_REGISTRATION_TOKEN

# Follow prompts for name, labels, etc.
```

#### 2. Configuring the Runner Agent (Service vs. Interactive Mode)

Once configured, the runner agent needs to run to listen for jobs from GitHub Actions.

- **Interactive Mode:**

  - **How:** Run `./run.sh` (Linux/macOS) or `.\run.cmd` (Windows) directly in your terminal.
  - **Use Case:** Useful for initial setup, testing, or debugging. The runner stops when you close the terminal or log out.
  - **Not Recommended for Production:** Requires a persistent user session.

- **Service Mode (Recommended for Production):**
  - **How:** The runner agent includes scripts to install itself as a system service (`svc.sh` for Linux/macOS systemd/launchd, `svc.cmd` for Windows Services).
    - Linux (systemd): `sudo ./svc.sh install`, `sudo ./svc.sh start`
    - macOS (launchd): `sudo ./svc.sh install`, `sudo ./svc.sh start`
    - Windows (Admin PowerShell): `.\svc.cmd install`, `.\svc.cmd start`
  - **Use Case:** Ensures the runner starts automatically on boot and runs reliably in the background without requiring a user to be logged in.
  - **Management:** Use standard service management commands (`systemctl`, `launchctl`, `services.msc`) to start, stop, restart, and check the status of the runner service.

##### [Configuration Guide: Step-by-step runner registration for a repository]

1.  **Navigate:** Go to your repository on GitHub.com.
2.  **Settings:** Click the `Settings` tab.
3.  **Actions > Runners:** In the left sidebar, click `Actions`, then `Runners`.
4.  **New Runner:** Click the `New self-hosted runner` button.
5.  **Select OS/Arch:** Choose the operating system and architecture of the machine where you will install the runner.
6.  **Follow Instructions:** GitHub displays commands to download, configure, and run the runner agent.
7.  **Download & Extract:** Execute the download and extraction commands on your target machine.
8.  **Configure:** Run the `config.sh` or `config.cmd` script.
    - When prompted for the URL, ensure it's the repository URL (e.g., `https://github.com/my-user/my-repo`).
    - Copy the registration token from the GitHub UI and paste it when prompted.
    - Provide a name and any desired labels.
9.  **Run:** Start the runner either interactively (`./run.sh` or `.\run.cmd`) or install and start it as a service (`sudo ./svc.sh install`, `sudo ./svc.sh start` or `.\svc.cmd install`, `.\svc.cmd start`).
10. **Verify:** Refresh the Runners page in GitHub Settings. Your new runner should appear with an "Idle" status.

##### [Configuration Guide: Step-by-step runner registration for an organization]

1.  **Navigate:** Go to your organization's page on GitHub.com (e.g., `https://github.com/my-org`).
2.  **Settings:** Click the `Settings` tab.
3.  **Actions > Runners:** In the left sidebar, under the "Actions" section, click `Runners`.
4.  **New Runner:** Click the `New runner` button. You might see options like `New self-hosted runner` or potentially platform-specific setup guides if enabled. Choose `New self-hosted runner`.
5.  **Select OS/Arch:** Choose the operating system and architecture.
6.  **Follow Instructions:** GitHub displays the necessary commands.
7.  **Download & Extract:** Execute the download and extraction commands on your target machine.
8.  **Configure:** Run the `config.sh` or `config.cmd` script.
    - When prompted for the URL, ensure it's the organization URL (e.g., `https://github.com/my-org`).
    - Copy the registration token from the GitHub UI.
    - Provide a name.
    - **Runner Group:** Choose the runner group to assign this runner to (defaults to `Default`). You can create custom groups beforehand (see Section D).
    - Provide any desired labels.
9.  **Run:** Start the runner interactively or as a service.
10. **Verify:** Refresh the Runners page in Organization Settings. Your new runner should appear in the selected group with an "Idle" status.

#### 3. Using Labels for Runner Targeting (`runs-on: [self-hosted, label1, label2]`)

Labels are the mechanism used in your workflow YAML files to specify _which_ runner(s) should execute a job.

- **`self-hosted`:** All self-hosted runners automatically receive this label. You _must_ include `self-hosted` in your `runs-on` key when targeting any self-hosted runner.
- **Default Labels:** Runners automatically get labels for their operating system (e.g., `linux`, `windows`, `macos`) and architecture (e.g., `x64`, `arm`, `arm64`).
- **Custom Labels:** You define these during the runner configuration (`./config.sh --labels mylabel,anotherlabel`) or later via the GitHub UI (Organization Settings > Actions > Runners > Edit Runner). Custom labels allow you to categorize runners based on capabilities, environment, or purpose.

The `runs-on` key in your workflow job definition accepts a single label or an array of labels. A job will only be assigned to a runner that possesses _all_ the labels specified in the `runs-on` key.

##### [Practical Example: Targeting specific runner groups based on OS or capabilities]

```yaml
name: Build and Test Application

on: [push]

jobs:
  build-linux:
    name: Build on Linux x64
    # Requires a self-hosted runner with BOTH 'linux' and 'x64' labels
    # (These are usually added automatically, but explicitly listing is clear)
    runs-on: [self-hosted, linux, x64]
    steps:
      - uses: actions/checkout@v4
      - name: Build
        run: make build

  test-windows-gpu:
    name: Test on Windows with GPU
    # Requires a self-hosted runner explicitly labeled 'windows' and 'gpu'
    runs-on: [self-hosted, windows, gpu]
    steps:
      - uses: actions/checkout@v4
      - name: Run GPU-intensive tests
        run: .\run-gpu-tests.ps1

  deploy-production:
    name: Deploy to Production Environment
    needs: [build-linux, test-windows-gpu]
    # Requires a self-hosted runner specifically designated for production deployments
    runs-on: [self-hosted, linux, production-deployer]
    environment: production # Optional: Link to a GitHub Environment
    steps:
      - name: Deploy application
        run: ./deploy.sh --env production
```

In this example:

- `build-linux` targets any self-hosted Linux x64 runner.
- `test-windows-gpu` targets a specific self-hosted Windows runner equipped with a GPU and labeled accordingly.
- `deploy-production` targets a runner designated solely for production deployments, likely with specific network access and credentials configured, identified by the `production-deployer` label.

## C. Security Considerations for Self-Hosted Runners

Using self-hosted runners shifts the security responsibility from GitHub to you. Securing these runners is **critical** as they often have access to sensitive environments and execute code from your repository.

#### 1. Runner Host Security Hardening

Treat your runner machines like any production server:

- **Minimal Installation:** Install only the necessary OS components and software required for the runner agent and your build/test tasks. Avoid unnecessary services or tools.
- **Regular Patching:** Keep the operating system and all installed software up-to-date with security patches. Automate this process where possible.
- **Principle of Least Privilege:** Run the runner agent service using a dedicated, non-privileged user account. Avoid running as `root` or `Administrator`. Ensure the runner user has only the permissions necessary to perform its tasks (e.g., clone code, run build tools, access specific directories).
- **Access Control:** Limit administrative access (SSH, RDP) to the runner host to authorized personnel only. Use strong authentication methods (SSH keys, MFA).
- **Endpoint Security:** Install and maintain endpoint security software (antivirus/anti-malware) if appropriate for your environment and compliance needs.

#### 2. Network Security (Firewalls, Access Controls)

Control network traffic to and from your runner hosts:

- **Firewalls:** Implement host-based firewalls (like `ufw` on Linux or Windows Firewall) and network firewalls.
- **Egress Control (Outbound):** Restrict outbound connections to only what is necessary:
  - GitHub.com (for registering, receiving jobs, sending logs/status). See GitHub Docs for specific required domains/IPs.
  - Package registries (npm, PyPI, Maven Central, Docker Hub, etc.).
  - Internal resources required by your workflows (databases, APIs).
  - _Deny all other outbound traffic by default._
- **Ingress Control (Inbound):** Self-hosted runners initiate connections _outbound_ to GitHub. They generally do _not_ require any inbound connections from the internet. Block all unnecessary inbound traffic. If administrative access (SSH/RDP) is needed, restrict it to specific trusted IP ranges (e.g., your corporate VPN).
- **Network Segmentation:** Place runners in isolated network segments, separate from highly sensitive production systems, if possible.

#### 3. Running Workflows from Forks (Security Risks)

By default, workflows triggered by `pull_request` events from forks of public repositories run on GitHub-hosted runners with read-only permissions and no access to secrets. **This is a critical security boundary.**

Enabling self-hosted runners for workflows from forks introduces significant risks:

- **Malicious Code Execution:** A contributor could submit a pull request with code designed to exploit the runner environment, steal credentials stored on the runner, or attack other resources accessible from the runner's network.
- **Secret Exposure:** Although GitHub Actions has some safeguards, malicious PR code could potentially try to exfiltrate secrets available to the runner.

> **Production Note:** **Never run untrusted code from forks on self-hosted runners without strict controls.** If absolutely necessary, consider:
>
> - Using dedicated, isolated, ephemeral runners specifically for forks.
> - Requiring manual approval for all workflows triggered by first-time contributors or specific fork events (`pull_request_target` requires careful handling).
> - Implementing robust code scanning and review processes _before_ allowing a fork's workflow to run on a self-hosted runner.
> - Disabling fork runs on self-hosted runners entirely at the repository or organization level (`Settings > Actions > General > Fork pull request workflows`).

#### 4. Ephemeral Runners vs. Persistent Runners

- **Persistent Runners:** The runner agent runs continuously on the same machine (physical or VM). The workspace might be reused between jobs (though cleanup usually occurs). If compromised, the attacker has persistent access until the runner is cleaned or rebuilt. State can accidentally leak between jobs if cleanup fails.
- **Ephemeral Runners:** A new, clean runner instance is provisioned for each job (or a small number of jobs) and then destroyed immediately after completion. This significantly enhances security.

##### [Deep Dive: Implementing ephemeral runners for enhanced security]

Ephemeral runners provide a clean, isolated environment for every job run, drastically reducing the risk of state leakage between jobs and limiting the window of opportunity for an attacker if a job compromises the runner instance.

**Common Implementation Strategies:**

1.  **Containerized Runners:**

    - Run the runner agent inside a Docker container.
    - Use an orchestrator (like Kubernetes with `actions-runner-controller`, Docker Swarm, AWS ECS, Azure Container Instances) to launch a new container instance for each job request.
    - The container is destroyed after the job finishes.
    - **Benefits:** Fast startup, lightweight, excellent isolation.
    - **Tools:** `actions-runner-controller` (ARC) is a popular Kubernetes operator for managing ephemeral, auto-scaled runners.

2.  **Virtual Machine-Based Runners:**
    - Use cloud provider APIs (AWS EC2, Azure VM, GCP Compute Engine) or virtualization tools (vSphere) to provision a fresh VM for each job.
    - Install the runner agent dynamically during VM startup (using user data/cloud-init) or use pre-built VM images (AMIs, VHDs).
    - Terminate the VM after the job completes.
    - **Benefits:** Full OS-level isolation, suitable for jobs needing kernel access or specific OS configurations not easily containerized.
    - **Drawbacks:** Slower startup times and potentially higher cost compared to containers.
    - **Tools:** Philips-labs/terraform-aws-github-runner, `actions-runner-controller` (can manage VM-based runners too, though less common).

**Key Advantages of Ephemeral Runners:**

- **Clean Slate:** Each job starts in a known, clean environment.
- **Reduced Attack Surface:** Compromise is contained to a single job run; the instance is destroyed afterward.
- **Simplified Maintenance:** Base images/containers are updated, and new jobs automatically use the updated version. No need to patch long-running persistent runners.

#### 5. Runner Process Isolation

Even on persistent runners, ensure jobs cannot interfere with each other or the host system:

- **Runner Agent:** The agent itself attempts to isolate job processes.
- **Containers (Docker):** If jobs run within containers (e.g., using container actions or `docker run` steps), this provides strong process and filesystem isolation. Configure the Docker daemon securely on the host.
- **User Accounts:** On Linux/macOS, consider configuring the runner to execute jobs as different, less privileged users if strict isolation between concurrent jobs on the _same_ runner is needed (though ephemeral runners are generally preferred).

## D. Managing Runner Groups (Organization Level)

For organizations managing multiple repositories and runner types, runner groups provide centralized control and administration. Runner groups are configured at the organization level (`Settings > Actions > Runner groups`).

#### 1. Creating and Assigning Runners to Groups

- **Creation:** Organization owners or admins can create new runner groups (e.g., "Production Deployers", "GPU Testers", "Windows Builders").
- **Assignment:** When registering a new organization-level runner (`./config.sh --url https://github.com/my-org ...`), you can specify the target group using the `--runnergroup` flag or select it during the interactive configuration. Existing runners can also be moved between groups via the GitHub UI.
- **Default Group:** Runners not assigned to a specific group belong to the "Default" group.

#### 2. Controlling Repository Access to Runner Groups

This is the primary benefit of runner groups: granular control over which repositories can utilize which runners.

- **Access Policies:** For each runner group, you can configure which repositories (or all repositories) within the organization are allowed to send jobs to runners in that group.
  - **All Repositories:** Any repository in the organization can use runners in this group.
  - **Selected Repositories:** Explicitly list the repositories granted access. This is crucial for restricting access to specialized or sensitive runners (e.g., production deployment runners).
- **Permissions:** You need organization owner or admin privileges to manage runner groups and their access policies.

##### [Configuration Guide: Setting up org-level runner groups and access policies]

1.  **Navigate:** Go to your organization's `Settings`.
2.  **Actions > Runner groups:** In the left sidebar, click `Runner groups` under the "Actions" section.
3.  **New Runner Group:** Click the `New runner group` button.
4.  **Name Group:** Enter a descriptive name (e.g., `linux-builders`, `secure-deploy`).
5.  **Configure Access:** Choose the repository access policy:
    - `Allow public repositories`: Controls access for public repos (use with caution).
    - `Repository access`: Select `All repositories` or `Selected repositories`. If selecting, search for and add the specific repositories that should have access to this group.
    - _Recommendation:_ Use `Selected repositories` for groups with specialized hardware or access to sensitive environments.
6.  **Create Group:** Click `Create group`.
7.  **Assign Runners:**
    - When registering _new_ runners, specify this group name during configuration.
    - For _existing_ runners, go to `Settings > Actions > Runners`, find the runner, click the `...` menu, choose `Edit runner`, and select the desired `Runner group`.
8.  **Target in Workflow:** In your repository's workflow file, use the appropriate labels associated with the runners in that group. Note that you target runners via _labels_, not group names directly in the `runs-on` key. Ensure runners within the group have consistent, descriptive labels.

```yaml
# Example: Targeting a runner in the 'secure-deploy' group
# Assume runners in this group are labeled 'self-hosted', 'linux', 'deploy'

jobs:
  deploy:
    # This job will only run on runners that:
    # 1. Belong to a group accessible by this repository.
    # 2. Have ALL the labels: 'self-hosted', 'linux', 'deploy'.
    runs-on: [self-hosted, linux, deploy]
    steps:
      - name: Deploy step
        run: echo "Deploying..."
```

## E. Scaling Self-Hosted Runners

As your organization's use of GitHub Actions grows, you'll likely need to scale your self-hosted runner fleet to handle the increased load and avoid long job queue times.

#### 1. Manual Scaling Strategies

The simplest approach involves monitoring runner utilization and manually adding or removing runner instances as needed.

- **Adding Runners:** Provision a new VM or physical machine, install the runner agent, and register it with the appropriate repository or organization group.
- **Removing Runners:** Stop the runner service (`sudo ./svc.sh stop`, `.\svc.cmd stop`), remove the service (`sudo ./svc.sh uninstall`, `.\svc.cmd uninstall`), and remove the runner configuration from GitHub (`./config.sh remove --token REMOVAL_TOKEN`). Finally, decommission the host machine.
- **Pros:** Simple to understand and implement for small-scale needs.
- **Cons:** Reactive, requires manual intervention, inefficient for fluctuating loads, prone to bottlenecks or over-provisioning.

#### 2. Autoscaling Solutions (e.g., `actions-runner-controller` for Kubernetes)

Autoscaling automatically adjusts the number of active runners based on the demand (number of queued jobs). This provides elasticity, ensuring jobs are processed promptly while minimizing costs during idle periods.

- **Concept:** An external controller monitors the job queue for specific labels/groups via the GitHub API. When jobs are queued, it provisions new runner instances (often containers or VMs). When runners become idle, the controller terminates them after a configurable period.
- **`actions-runner-controller` (ARC):** A popular open-source Kubernetes operator specifically designed for autoscaling GitHub Actions runners.
  - **How it Works:** You deploy ARC to your Kubernetes cluster. You define `RunnerDeployment` or `RunnerSet` custom resources, specifying the runner image, scope (repo/org), labels, scaling parameters (min/max replicas), etc. ARC watches for jobs matching these definitions and scales runner pods accordingly.
  - **Benefits:** Leverages Kubernetes for orchestration, scheduling, and scaling; supports ephemeral container-based runners; integrates well with cloud Kubernetes services (EKS, AKS, GKE).

##### [Deep Dive: Implementing ARC for autoscaled Kubernetes-based runners]

1.  **Prerequisites:** A running Kubernetes cluster (e.g., EKS, AKS, GKE, k3s, Kind). `kubectl` configured to access the cluster. Helm (optional, but recommended for installation).
2.  **Install Cert-Manager:** ARC requires cert-manager for webhook certificate management.
    ```bash
    kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.13.x/cert-manager.yaml
    ```
    _(Wait for cert-manager pods to be ready)_
3.  **Install ARC:** Use Helm (recommended) or apply manifests directly.
    ```bash
    helm repo add actions-runner-controller https://actions-runner-controller.github.io/actions-runner-controller
    helm upgrade --install --namespace actions-runner-system --create-namespace \
      --set=authSecret.create=true \
      --set=authSecret.github_token="YOUR_GITHUB_PAT" \ # Or use github_app options
      actions-runner-controller actions-runner-controller/actions-runner-controller
    ```
    - **Authentication:** ARC needs a GitHub Personal Access Token (PAT) with `repo` (for repo runners) or `admin:org` (for org runners) scope, or preferably, credentials for a dedicated GitHub App. Store this securely in a Kubernetes secret (`authSecret`).
4.  **Define RunnerDeployment/RunnerSet:** Create a YAML manifest for your runners.
    ```yaml
    # runner-deployment.yaml
    apiVersion: actions.summerwind.dev/v1alpha1
    kind: RunnerDeployment
    metadata:
      name: example-runner-deployment
      namespace: actions-runner-system # Or your preferred namespace
    spec:
      replicas: 1 # Start with 1, HorizontalRunnerAutoscaler will manage this
      template:
        spec:
          repository: your-org/your-repo # Or 'organization: your-org'
          labels:
            - self-hosted
            - linux
            - x64
            - k8s-runner # Custom label for these runners
          # Use a standard runner image or build your own
          image: summerwind/actions-runner:latest
          # Define resource requests/limits for runner pods
          resources:
            limits:
              cpu: "1"
              memory: "2Gi"
            requests:
              cpu: "500m"
              memory: "1Gi"
    ```
5.  **Define HorizontalRunnerAutoscaler (HRA):** Create a YAML manifest for autoscaling.
    ```yaml
    # runner-hra.yaml
    apiVersion: actions.summerwind.dev/v1alpha1
    kind: HorizontalRunnerAutoscaler
    metadata:
      name: example-runner-hra
      namespace: actions-runner-system
    spec:
      scaleTargetRef:
        name: example-runner-deployment # Must match RunnerDeployment name
      minReplicas: 1 # Minimum number of runners always available
      maxReplicas: 10 # Maximum number of runners to scale up to
      # Scale up when queued jobs > 0 (adjust metrics as needed)
      metrics:
        - type: TotalNumberOfQueuedAndInProgressWorkflowRuns
          repositoryNames:
            - your-org/your-repo # Scope scaling to specific repos if needed
    ```
6.  **Apply Manifests:**
    ```bash
    kubectl apply -f runner-deployment.yaml -n actions-runner-system
    kubectl apply -f runner-hra.yaml -n actions-runner-system
    ```
7.  **Trigger Workflow:** Push a change to your repository that triggers a workflow targeting the labels defined (e.g., `runs-on: [self-hosted, linux, k8s-runner]`). Observe ARC scaling the runner pods up and down in Kubernetes.

##### [Case Study: Setting up scalable self-hosted runners on AWS/Azure/GCP]

- **AWS:** Use EKS (Elastic Kubernetes Service) to host `actions-runner-controller`. ARC will create runner pods scheduled onto EC2 instances within the EKS node groups. Use EC2 Autoscaling Groups or Karpenter for the underlying EKS nodes to scale the cluster infrastructure itself based on pod resource requests. For VM-based scaling, use tools like `philips-labs/terraform-aws-github-runner` which leverages Lambda, SQS, and EC2 Auto Scaling Groups.
- **Azure:** Use AKS (Azure Kubernetes Service) to host `actions-runner-controller`. ARC manages runner pods on AKS nodes (backed by Virtual Machine Scale Sets - VMSS). Configure cluster autoscaler on AKS to scale the node pools. Alternatively, use Azure VM Scale Sets directly with custom scripts or tools like `dapr/github-actions-runner` for VM-based autoscaling triggered by queue messages.
- **GCP:** Use GKE (Google Kubernetes Engine) to host `actions-runner-controller`. ARC manages runner pods on GKE nodes (backed by Managed Instance Groups - MIGs). Enable GKE cluster autoscaler to manage node pool scaling. For VM-based scaling, use Compute Engine MIGs with custom startup scripts triggered by Pub/Sub messages or a dedicated controller monitoring the GitHub API.

In all cloud scenarios, leverage managed Kubernetes services and their native node autoscaling features alongside `actions-runner-controller` for robust, container-based runner autoscaling.

#### 3. Monitoring Runner Fleet Health and Utilization

Effective scaling requires monitoring:

- **Runner Status:** Check the runner status (Idle, Active, Offline) in the GitHub UI (Repository/Organization Settings > Actions > Runners).
- **Job Queue Length:** Monitor how many jobs are waiting for a runner. Long queues indicate a need for more runners. GitHub doesn't expose this directly via UI easily, but autoscalers monitor it via API.
- **Runner Utilization:** Track CPU, memory, disk, and network usage on your runner hosts or within runner containers. High utilization might necessitate more powerful runner instances or more numerous instances.
- **Autoscaler Metrics (if applicable):** Monitor the autoscaler's own metrics (e.g., number of active runners, scaling decisions, errors). Tools like Prometheus/Grafana are often used with ARC.
- **Costs:** Track the infrastructure costs associated with your runner fleet.

## F. Maintaining and Updating Self-Hosted Runners

Self-hosted runners require ongoing maintenance to ensure security, stability, and compatibility.

#### 1. Updating the Runner Agent Software

- **Automatic Updates (Default):** By default, the runner agent periodically checks for new versions and attempts to update itself automatically when idle. This is generally recommended.
- **Manual Updates:** You can check the current version (`./config.sh --version`) and compare it to the latest release on `github.com/actions/runner/releases`. To manually trigger an update check (if auto-update is disabled or you want to force it), you might need to restart the runner service or consult specific runner documentation. In some cases, downloading and re-running the configuration might be needed for major updates, but typically the agent handles it.
- **Monitoring:** Keep an eye on runner logs for any update failures.

#### 2. Patching the Runner Host OS and Software

This is a critical security responsibility.

- **Operating System:** Regularly apply OS security patches and updates (e.g., `sudo apt update && sudo apt upgrade -y` on Debian/Ubuntu, Windows Update on Windows).
- **Installed Software:** Update any software installed on the runner host that your workflows depend on (e.g., Docker, Node.js, Python, compilers, build tools).
- **Base Images (Ephemeral Runners):** If using ephemeral runners (containers/VMs), regularly rebuild your base images with the latest OS patches and software versions. Your autoscaler will then launch new runners based on the updated images.

##### [Production Note: Automation strategies for runner maintenance]

Manually patching a large fleet of runners is time-consuming and error-prone. Automation is key:

- **Configuration Management:** Use tools like Ansible, Chef, Puppet, or SaltStack to automate OS patching, software updates, and configuration consistency across your persistent runner fleet.
- **Golden Images:** For VM-based runners (persistent or ephemeral), maintain a "golden image" (e.g., AWS AMI, Azure Image, GCP Image) that includes the OS, runner agent, and common dependencies. Regularly update this image using tools like Packer and roll it out to your fleet or use it as the base for new ephemeral VMs.
- **Container Image Scanning:** Integrate container image vulnerability scanning (e.g., Trivy, Clair, cloud provider tools like ECR scanning, ACR scanning, GCR scanning) into your CI/CD pipeline for building runner container images. Regularly rebuild and redeploy runner images with patched base layers and dependencies.
- **Scheduled Maintenance:** Implement scheduled tasks or cron jobs for routine updates during low-usage periods.

## G. Networking for Self-Hosted Runners

Self-hosted runners need reliable network connectivity to GitHub.com and potentially other resources.

#### 1. Proxy Configuration

If your runners are behind an HTTP or HTTPS proxy server, you need to configure the runner agent to use it.

- **Environment Variables:** The standard way is to set the `http_proxy`, `https_proxy`, and `no_proxy` environment variables _before_ running the configuration script (`config.sh`/`config.cmd`) and the runner agent (`run.sh`/`run.cmd` or the service).
  - Example (Linux/macOS):
    ```bash
    export http_proxy="http://proxy.example.com:8080"
    export https_proxy="http://proxy.example.com:8080"
    export no_proxy="localhost,127.0.0.1,.example.internal" # Domains to bypass proxy
    ./config.sh ...
    ./run.sh # Or configure service to inherit these variables
    ```
  - Example (Windows PowerShell):
    ```powershell
    $env:http_proxy = "http://proxy.example.com:8080"
    $env:https_proxy = "http://proxy.example.com:8080"
    $env:no_proxy = "localhost,127.0.0.1,.example.internal"
    .\config.cmd ...
    .\run.cmd # Or configure service environment
    ```
- **.env File:** Alternatively, create a `.env` file in the runner agent directory with the proxy settings:
  ```
  http_proxy=http://proxy.example.com:8080
  https_proxy=http://proxy.example.com:8080
  no_proxy=localhost,127.0.0.1,.example.internal
  ```
  The runner agent will automatically load these variables.
- **Authentication:** If the proxy requires authentication, include credentials in the URL: `http://user:password@proxy.example.com:8080`. Securely manage these credentials.

#### 2. Firewall Rules

Ensure your firewall allows the necessary outbound connections from the runner host. The specific domains and IPs can change, so it's best to consult the official GitHub documentation for the most up-to-date list. Generally, runners need access to:

- `github.com` (HTTPS/443)
- `api.github.com` (HTTPS/443)
- `*.actions.githubusercontent.com` (HTTPS/443) - For downloading actions.
- `*.blob.core.windows.net` (HTTPS/443) - For log and artifact storage (Azure Blob Storage).
- `*.pkg.github.com` (HTTPS/443) - For GitHub Packages.
- Package manager repositories (e.g., `registry.npmjs.org`, `pypi.org`, `docker.io`).
- Any internal resources your workflows need to access.

**Restrict inbound traffic** as much as possible, ideally allowing only established/related connections initiated by the runner.

##### [Troubleshooting Section: Common network connectivity issues]

- **Runner Appears Offline:**
  - **Check Service/Process:** Ensure the runner agent service (`svc.sh status` / `services.msc`) or process (`run.sh`/`run.cmd`) is running on the host. Check runner logs (`_diag` folder) for errors.
  - **Firewall Blocks:** Verify outbound firewall rules allow access to the required GitHub domains (see above) on port 443. Use tools like `curl -v https://api.github.com` or `openssl s_client -connect api.github.com:443` from the runner host.
  - **Proxy Issues:** If using a proxy, double-check proxy environment variables (`http_proxy`, `https_proxy`, `no_proxy`) are correctly set and accessible by the runner process/service. Test proxy connectivity independently. Check proxy authentication.
  - **DNS Resolution:** Ensure the runner host can resolve GitHub domain names. Use `nslookup github.com` or `dig github.com`. Check DNS server configuration (`/etc/resolv.conf` on Linux).
  - **Token/Registration Issues:** An invalid or expired registration token during setup can prevent connection. Try re-registering the runner.
- **Action/Tool Download Failures:**
  - **Firewall/Proxy:** Often caused by blocked access to `*.actions.githubusercontent.com` or package registries (npm, Docker Hub, etc.). Verify firewall/proxy rules for these domains.
  - **SSL/TLS Interception:** If your network performs SSL/TLS inspection, ensure the runner host trusts the proxy's certificate authority (CA). You might need to install the corporate root CA certificate on the runner host or configure the runner agent/tools (Node.js, Git, Docker) to use it. See runner documentation for configuring custom CAs.
- **Slow Performance:**
  - **Network Latency:** High latency between the runner and GitHub or other resources.
  - **Bandwidth Constraints:** Insufficient network bandwidth on the runner host or through the proxy.
  - **Resource Exhaustion:** Runner host CPU, memory, or disk I/O is bottlenecked. Monitor host resources.

By understanding these networking requirements and potential issues, you can ensure your self-hosted runners maintain reliable communication with GitHub Actions and execute your workflows efficiently.
