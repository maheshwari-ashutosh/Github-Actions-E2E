# Chapter 6: Deployment Strategies with GitHub Actions

Welcome to Chapter 6, where we shift our focus from building and testing code to the critical final step: deploying applications into various environments using GitHub Actions. Automation is key to reliable, repeatable, and efficient deployments, and GitHub Actions provides a powerful and flexible platform to orchestrate these processes.

In this chapter, we'll explore the fundamentals of deployment automation, delve into specific strategies for different deployment targets (from traditional servers to modern serverless platforms), and examine crucial features like environment protection and rollback strategies. By the end, you'll understand how to leverage GitHub Actions to streamline your deployment pipelines, regardless of your infrastructure choices.

## A. Fundamentals of Deployment Automation

Before diving into specific techniques, let's establish a common understanding of the core concepts underpinning deployment automation, particularly within the context of GitHub Actions.

### 1. Deployment Targets (Servers, Containers, Serverless, PaaS)

Your deployment strategy is heavily influenced by _where_ you are deploying your application. Common targets include:

- **Servers (Virtual Machines / Bare Metal):** The traditional approach involves deploying code directly onto virtual machines (VMs) or physical servers. This often requires secure shell (SSH) access for file transfer (SCP/Rsync) and command execution, or the use of configuration management tools.
- **Containers (Kubernetes, ECS, Cloud Run, etc.):** Containerized applications (typically using Docker) are deployed to container orchestrators. Deployment involves building container images, pushing them to a registry (like Docker Hub, GitHub Container Registry, AWS ECR, etc.), and then instructing the orchestrator to pull and run the new image version.
- **Serverless (AWS Lambda, Azure Functions, Google Cloud Functions):** Deploying to serverless platforms usually involves packaging function code and its dependencies into an archive (e.g., a ZIP file) and uploading it to the cloud provider's service. Configuration updates might also be required.
- **Platform-as-a-Service (PaaS) (Heroku, Vercel, Netlify, Render):** PaaS providers abstract away much of the underlying infrastructure. Deployment often involves pushing code to a specific Git remote or using the provider's Command Line Interface (CLI) or a dedicated GitHub Action.

GitHub Actions can interact with all these targets by running CLI commands, using specialized Actions from the Marketplace, or making API calls.

### 2. Environment Management (Dev, Staging, Production)

Most software development lifecycles involve multiple environments to isolate development, testing, and live user traffic:

- **Development (Dev):** Used by developers for coding and initial testing. Often runs locally or in a shared cloud environment. Deployments are frequent and automated.
- **Staging (or QA, Pre-Prod):** A replica of the production environment used for final testing, user acceptance testing (UAT), and integration testing before release. Deployments are typically automated but might require approvals.
- **Production (Prod):** The live environment serving end-users. Deployments are carefully managed, often requiring manual approvals and strict controls.

GitHub Actions facilitates environment management through:

- **Environment-specific Secrets:** Storing sensitive data (API keys, passwords) scoped to specific environments (e.g., a production database password only available to production deployment workflows).
- **Environment Variables:** Defining configuration values that differ between environments (e.g., API endpoints, feature flags).
- **Environment Protection Rules:** Implementing deployment gates like manual approvals or wait timers specifically for sensitive environments like production (covered in detail in Section F).
- **Conditional Logic:** Using expressions (`if: ...`) in workflows to execute steps only when deploying to certain environments (often determined by the branch name or tags).

```mermaid
graph LR
    A[Code Commit] --> B{Build & Test};
    B -- Success --> C{Deploy to Dev};
    C --> D{Deploy to Staging};
    D -- Manual Approval? --> E[Deploy to Production];

    subgraph CI/CD Pipeline
        direction LR
        B
        C
        D
        E
    end

    style B fill:#f9f,stroke:#333,stroke-width:2px
    style C fill:#ccf,stroke:#333,stroke-width:2px
    style D fill:#ccf,stroke:#333,stroke-width:2px
    style E fill:#cfc,stroke:#333,stroke-width:2px
```

**Diagram Explanation:** This diagram illustrates a typical CI/CD (Continuous Integration/Continuous Deployment) flow. Code commits trigger automated build and test processes. Successful builds are often automatically deployed to a development environment. Deployment to staging might also be automated or triggered manually. Crucially, deployment to the production environment often involves stricter controls, such as manual approvals, representing a common pattern managed within GitHub Actions.

### 3. The Role of Secrets in Deployments

Deployments almost invariably require sensitive information:

- SSH keys for server access.
- API tokens for cloud providers (AWS, Azure, GCP).
- Database credentials.
- Access tokens for PaaS platforms (Heroku, Vercel).
- Credentials for container registries.

Exposing these secrets directly in workflow files is a major security risk. GitHub Actions provides **Secrets** as a secure storage mechanism.

- **Repository Secrets:** Available to actions running in any environment within that repository.
- **Environment Secrets:** Scoped specifically to a deployment environment (e.g., `production`, `staging`). These override repository secrets with the same name when deploying to that environment.
- **Organization Secrets:** Defined at the organization level and can be selectively exposed to specific repositories.

Secrets are injected as environment variables or inputs into workflow steps but are automatically redacted in logs, preventing accidental exposure. We covered the basics of secrets in Chapter 3, but their importance cannot be overstated in the context of deployment automation. **Always use GitHub Secrets for sensitive deployment credentials.**

## B. Deploying to Virtual Machines / Bare Metal

Deploying to traditional servers remains a common scenario, especially for legacy applications or specific infrastructure requirements. GitHub Actions can automate this using standard protocols or configuration management tools.

### 1. Using SSH and SCP/Rsync

The most direct way to interact with servers is via Secure Shell (SSH). Workflows can use SSH to execute commands remotely (e.g., restart a service, run a deployment script) and Secure Copy (SCP) or Rsync to transfer build artifacts.

#### [Configuration Guide: Setting up SSH keys as secrets for deployment]

To allow your GitHub Actions workflow to securely connect to your server via SSH, you need to provide it with an SSH private key.

1.  **Generate an SSH Key Pair:** If you don't already have one dedicated for deployments, generate a new key pair _without a passphrase_ (as Actions cannot interactively enter one):

    ```bash
    ssh-keygen -t ed25519 -f deploy_key -N ""
    # This creates 'deploy_key' (private key) and 'deploy_key.pub' (public key)
    ```

    _Note: Using modern algorithms like Ed25519 is recommended over older RSA keys._

2.  **Add the Public Key to the Server:** Copy the contents of `deploy_key.pub` and add it to the `~/.ssh/authorized_keys` file on the target server for the user the workflow will connect as (e.g., `deploy_user`). Ensure correct file permissions (`chmod 600 ~/.ssh/authorized_keys`).

3.  **Add the Private Key as a GitHub Secret:**

    - Navigate to your repository on GitHub > Settings > Secrets and variables > Actions.
    - Click "New repository secret".
    - Name the secret descriptively, e.g., `SSH_PRIVATE_KEY`.
    - Paste the _entire content_ of the `deploy_key` file (the private key, including `-----BEGIN OPENSSH PRIVATE KEY-----` and `-----END OPENSSH PRIVATE KEY-----`) into the "Value" field.
    - Click "Add secret".

4.  **Add Server Hostname/IP as a Secret:** It's also good practice to store the server's hostname or IP address as a secret (e.g., `SSH_HOST`) and the SSH user as another (e.g., `SSH_USER`).

#### [Code Snippet: Workflow step to SSH into a server and deploy files]

This example uses the popular `appleboy/ssh-action` to connect via SSH, copy files using SCP, and run deployment commands.

```yaml
name: Deploy to VM via SSH

on:
  push:
    branches:
      - main # Trigger deployment on push to main

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Build application (example)
        run: |
          echo "Building the application..."
          # Replace with your actual build commands
          mkdir dist
          echo "<html><body><h1>Deployed via GitHub Actions!</h1></body></html>" > dist/index.html

      - name: Deploy to Server
        uses: appleboy/ssh-action@v1.0.3
        with:
          host: ${{ secrets.SSH_HOST }} # Server IP or hostname
          username: ${{ secrets.SSH_USER }} # Deployment user on the server
          key: ${{ secrets.SSH_PRIVATE_KEY }} # The private SSH key
          port: 22 # Default SSH port
          script: |
            # Commands executed on the remote server
            echo "Starting deployment..."
            # Example: Create target directory if it doesn't exist
            mkdir -p /var/www/myapp
            # Example: Remove old files (use with caution!)
            # rm -rf /var/www/myapp/*
            echo "Deployment commands finished."
          # Use scp_source for simple file/directory copying
          # Note: scp_target is relative to the user's home directory by default
          # Use absolute paths on the target if needed.
          # scp_source: "dist/*"
          # scp_target: "/var/www/myapp"

      - name: Transfer files using Rsync (Alternative/More Robust)
        uses: easingthemes/ssh-deploy@v5.0.0 # Another popular action supporting Rsync
        with:
          SSH_PRIVATE_KEY: ${{ secrets.SSH_PRIVATE_KEY }}
          ARGS: "-rlgoDzvc -i --delete" # Rsync arguments (recursive, links, owner, group, perms, verbose, compress, delete extraneous files)
          SOURCE: "dist/" # Source directory from the runner
          REMOTE_HOST: ${{ secrets.SSH_HOST }}
          REMOTE_USER: ${{ secrets.SSH_USER }}
          TARGET: "/var/www/myapp" # Target directory on the remote server

      - name: Post-deployment command (Example: Restart service)
        uses: appleboy/ssh-action@v1.0.3
        with:
          host: ${{ secrets.SSH_HOST }}
          username: ${{ secrets.SSH_USER }}
          key: ${{ secrets.SSH_PRIVATE_KEY }}
          script: |
            echo "Restarting application service..."
            # Example: sudo systemctl restart myapp.service
            # Ensure the SSH_USER has passwordless sudo rights for specific commands if needed,
            # or run the service under the deploy user.
            echo "Service restarted."

```

#### [Production Note: Security considerations for SSH-based deployments]

- **Least Privilege:** The SSH user (`deploy_user` in the example) should have only the minimum permissions necessary to deploy the application. Avoid using `root`. If `sudo` is needed, configure passwordless `sudo` _only_ for the specific required commands in the `/etc/sudoers` file.
- **Firewall Restrictions:** Configure the server's firewall (e.g., `ufw`, `firewalld`, cloud provider security groups) to allow SSH connections _only_ from GitHub Actions' IP address ranges. These ranges are published via the GitHub API and can change, so automating updates is recommended for robust security. See GitHub's documentation on "About GitHub-hosted runners" for IP ranges.
- **Dedicated Keys:** Use dedicated SSH keys solely for deployment automation. Do not reuse personal SSH keys.
- **Key Rotation:** Implement a policy for periodically rotating deployment SSH keys.
- **Secrets Security:** Protect your GitHub secrets. Limit who can modify repository/environment secrets.

### 2. Using Configuration Management Tools (Ansible, Chef, Puppet)

Instead of executing raw SSH commands, you can leverage configuration management tools like Ansible, Chef, or Puppet. Your GitHub Actions workflow can trigger these tools, which then handle the complexities of deployment and server configuration idempotently. Ansible is particularly popular for this due to its agentless nature (using SSH).

#### [Practical Example: Triggering an Ansible playbook from a workflow]

This example assumes you have an Ansible playbook (`deploy.yml`) in your repository that defines the deployment steps.

1.  **Prerequisites:**

    - Ansible playbook (`deploy.yml`) committed to your repository.
    - An Ansible inventory file (e.g., `inventory.ini`) defining your target hosts. This could also be committed or dynamically generated.
    - SSH access configured as described previously (SSH key in secrets).

2.  **Workflow:**

```yaml
name: Deploy via Ansible

on:
  push:
    branches:
      - main

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      # Optional: Build step if artifacts need to be available to Ansible
      # - name: Build application
      #   run: make build

      - name: Set up Python for Ansible
        uses: actions/setup-python@v5
        with:
          python-version: "3.x"

      - name: Install Ansible
        run: pip install ansible

      # Create a temporary SSH key file from the secret
      - name: Set up SSH key
        run: |
          mkdir -p ~/.ssh
          echo "${{ secrets.SSH_PRIVATE_KEY }}" > ~/.ssh/id_deploy
          chmod 600 ~/.ssh/id_deploy
          # Optional: Add server host key to known_hosts to avoid interactive prompt
          # ssh-keyscan -H ${{ secrets.SSH_HOST }} >> ~/.ssh/known_hosts

      - name: Run Ansible Playbook
        run: |
          ansible-playbook \
            -i inventory.ini \ # Path to your inventory file
            --private-key ~/.ssh/id_deploy \
            -u ${{ secrets.SSH_USER }} \
            deploy.yml # Path to your playbook
        env:
          ANSIBLE_HOST_KEY_CHECKING: "False" # Disable host key checking (less secure, use known_hosts or configure Ansible properly for production)
          # Pass other variables to Ansible if needed
          # ANSIBLE_EXTRA_VARS: "app_version=${{ github.sha }}"

      # Clean up the temporary SSH key
      - name: Clean up SSH key
        if: always() # Ensure cleanup runs even if previous steps fail
        run: rm -f ~/.ssh/id_deploy
```

```mermaid
graph LR
    subgraph GitHub Actions Runner
        direction LR
        A[Workflow Trigger (e.g., Push)] --> B(Checkout Code);
        B --> C(Setup Ansible);
        C --> D{Run Ansible Playbook};
    end

    subgraph Target Server
        direction TB
        F[SSH Connection] --> G(Execute Playbook Tasks);
        G --> H(Deploy Application Files);
        H --> I(Configure/Restart Services);
    end

    D -- SSH --> F;

    style A fill:#f9f,stroke:#333,stroke-width:2px
    style D fill:#ccf,stroke:#333,stroke-width:2px
    style F fill:#ffcc99,stroke:#333,stroke-width:2px
    style I fill:#cfc,stroke:#333,stroke-width:2px
```

**Diagram Explanation:** This diagram shows how a GitHub Actions workflow orchestrates deployment using Ansible. The workflow checks out code, sets up Ansible on the runner, and then executes the `ansible-playbook` command. Ansible connects to the target server(s) via SSH (using the provided key) and executes the tasks defined in the playbook to deploy the application and configure the server.

Using configuration management tools centralizes deployment logic within the tool's framework (playbooks, recipes, manifests), often leading to more maintainable and idempotent deployment processes compared to raw SSH scripts.

## C. Deploying Containerized Applications

Containers have become the standard for packaging and deploying modern applications. GitHub Actions excels at building container images (as seen in Chapter 5) and deploying them to various orchestrators.

### 1. Deploying to Kubernetes (EKS, AKS, GKE, OpenShift, K3s)

Kubernetes (K8s) is the leading container orchestration platform. Deploying to K8s from GitHub Actions involves authenticating with the cluster and then using tools like `kubectl` or Helm to apply configurations or deploy packaged applications.

#### a. Authenticating with Kubernetes Clusters (kubeconfig, Service Accounts, OIDC)

Securely connecting your workflow to your K8s cluster is paramount. Common methods include:

- **`kubeconfig` File:** A configuration file containing cluster details and credentials. You can store the base64-encoded content of a `kubeconfig` file as a GitHub Secret and decode it in the workflow to configure `kubectl`. This is simple but less secure, as the credentials in the file might be long-lived or overly permissive.
- **Service Account Tokens:** Create a dedicated Service Account within your K8s cluster with specific, limited permissions (RBAC). Generate a token for this Service Account and store it as a GitHub Secret. The workflow uses this token to authenticate. This is more secure than a full `kubeconfig` but still involves managing static tokens.
- **OpenID Connect (OIDC):** **(Recommended for Cloud Providers)** This is the most secure method for authenticating with managed Kubernetes services on major cloud platforms (AWS EKS, Azure AKS, GCP GKE). It allows GitHub Actions workflows to exchange a short-lived OIDC token (automatically generated by GitHub) for temporary cloud credentials, which are then used to access the cluster _without_ storing any long-lived keys or tokens in GitHub Secrets.

#### [Deep Dive: Using OIDC for secure, keyless authentication to cloud K8s]

OIDC authentication eliminates the need to manage static cloud credentials (like AWS Access Keys or Azure Service Principal secrets) within GitHub Secrets for accessing cloud resources, including managed Kubernetes clusters.

**Conceptual Workflow:**

1.  **Trust Configuration (One-time Setup):**

    - **In GitHub:** No specific configuration needed beyond using the standard OIDC token generation.
    - **In Cloud Provider (e.g., AWS):**
      - Create an OIDC Identity Provider in IAM, trusting `token.actions.githubusercontent.com`.
      - Create an IAM Role (e.g., `GitHubActionsEKSRole`) that workflows will assume.
      - Define a Trust Policy on this IAM Role that allows principals (federated users) from the GitHub OIDC provider to assume the role, typically restricted to specific repositories, branches, or environments using OIDC claims (like `repo`, `ref`, `environment`).
      - Grant this IAM Role the necessary permissions to interact with your EKS cluster (e.g., using `aws-auth` ConfigMap in EKS or Kubernetes RBAC bound to the IAM role).
    - **In Cloud Provider (e.g., Azure):**
      - Similar concept using Federated Identity Credentials associated with an Azure AD Application / Service Principal and granting that principal roles on the AKS cluster.
    - **In Cloud Provider (e.g., GCP):**
      - Similar concept using Workload Identity Federation, linking a Google Cloud Service Account to GitHub Actions identities and granting that service account roles on the GKE cluster.

2.  **Workflow Execution:**
    - The GitHub Actions workflow requests an OIDC token from GitHub.
    - The workflow uses an official action (e.g., `aws-actions/configure-aws-credentials`, `azure/login`, `google-github-actions/auth`) to exchange this OIDC token for temporary cloud credentials by assuming the pre-configured IAM Role (AWS) or authenticating the Service Principal/Service Account (Azure/GCP). This action typically configures the runner's environment so subsequent CLI commands (like `aws`, `az`, `gcloud`, `kubectl`) automatically use these temporary credentials.
    - The workflow uses `kubectl` or Helm, now authenticated via the temporary cloud credentials, to interact with the EKS/AKS/GKE cluster.

**Benefits:**

- **Keyless:** No long-lived cloud secrets stored in GitHub.
- **Short-lived Credentials:** Tokens and temporary credentials expire quickly, reducing the impact of potential leaks.
- **Fine-grained Permissions:** Cloud IAM and K8s RBAC provide granular control over what each workflow can do.

Setting up OIDC requires initial configuration in your cloud provider but significantly enhances security for cloud-based deployments. Refer to Chapter 19 and the official documentation from GitHub and your cloud provider for detailed setup instructions.

#### b. Using `kubectl` or Helm Charts

Once authenticated, you deploy using standard Kubernetes tools:

- **`kubectl`:** The primary Kubernetes CLI. Workflows can run `kubectl apply -f <manifest-file.yaml>` to apply configuration changes or `kubectl set image deployment/<deployment-name> <container-name>=<new-image-tag>` to update a running deployment's image.
- **Helm:** A package manager for Kubernetes. Helm Charts bundle application configurations. Workflows typically run `helm upgrade --install <release-name> <chart-path> --set image.tag=<new-image-tag>` to deploy or update an application.

#### [Code Snippet: Deploying an application using Helm]

This example assumes you have a Helm chart in your repository (`./charts/my-app`) and are deploying to a cluster where authentication is handled (e.g., via OIDC setup or a `kubeconfig` secret).

```yaml
name: Deploy to Kubernetes using Helm

on:
  push:
    tags:
      - 'v*.*.*' # Trigger deployment on version tags

jobs:
  deploy:
    runs-on: ubuntu-latest
    # Add permissions for OIDC token if using keyless auth
    permissions:
      id-token: write # Required for OIDC federation
      contents: read  # Required to checkout code

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Configure Cloud Credentials (Example: AWS OIDC)
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789012:role/GitHubActionsEKSRole # Replace with your IAM role ARN
          aws-region: us-east-1 # Replace with your cluster region
          # OIDC token is automatically fetched by the action

      - name: Set up Kubeconfig for EKS (if using AWS OIDC)
        run: aws eks update-kubeconfig --name my-eks-cluster --region us-east-1 # Replace with your cluster name and region

      # Alternative: Set up Kubeconfig from Secret (Less Secure)
      # - name: Set up Kubeconfig from Secret
      #   run: |
      #     mkdir -p $HOME/.kube
      #     echo "${{ secrets.KUBECONFIG_BASE64 }}" | base64 -d > $HOME/.kube/config
      #     chmod 600 $HOME/.kube/config

      - name: Set up Helm
        uses: azure/setup-helm@v4 # Installs Helm CLI

      - name: Build and Push Docker Image (Example - assumes previous chapter knowledge)
        # Replace with your actual image build/push steps
        # Ensure the image tag is determined (e.g., from Git tag)
        id: build_image
        run: |
          IMAGE_TAG=${GITHUB_REF_NAME#v} # Extract version from tag (e.g., v1.2.3 -> 1.2.3)
          echo "Building and pushing image: my-registry/my-app:${IMAGE_TAG}"
          # docker build ...
          # docker push ...
          echo "image_tag=${IMAGE_TAG}" >> $GITHUB_OUTPUT

      - name: Deploy with Helm
        run: |
          helm upgrade --install my-app-release ./charts/my-app \
            --namespace my-namespace \
            --create-namespace \
            --set image.repository=my-registry/my-app \
            --set image.tag=${{ steps.build_image.outputs.image_tag }} \
            --wait # Optional: wait for resources to become ready
```

#### [Case Study: Implementing a GitOps workflow with Argo CD / Flux triggered by Actions]

GitOps flips the deployment model. Instead of GitHub Actions directly pushing changes to the cluster, Actions pushes changes _to a Git repository_ that defines the desired state of the cluster. A GitOps agent running in the cluster (like Argo CD or Flux) continuously monitors this repository and automatically syncs the cluster state to match the Git repository.

**Workflow:**

1.  **Application Code Repo:** Developer pushes code changes. A GitHub Actions workflow runs build, test, and container image push steps.
2.  **Manifest Update:** Upon successful image push, the _same_ workflow (or a subsequent one) checks out a separate **Configuration Repository**. It updates Kubernetes manifests (e.g., Deployment YAML or a Helm chart's `values.yaml`) in this repo to point to the newly built container image tag. It then commits and pushes these changes.
3.  **Configuration Repo:** This Git repository holds the declarative definition (YAML manifests, Helm charts, Kustomize overlays) of the desired state for one or more environments.
4.  **Kubernetes Cluster:** Argo CD or Flux is installed in the cluster and configured to monitor the Configuration Repository.
5.  **Sync:** When Argo CD/Flux detects changes in the Configuration Repository (the updated image tag), it automatically applies these changes to the cluster, pulling the new container image and updating the relevant Kubernetes resources.

**GitHub Actions Role:** In a GitOps model, GitHub Actions is responsible for CI (build, test, image push) and updating the _source of truth_ (the Configuration Repository), but _not_ for direct interaction with the Kubernetes API for deployment.

**Benefits:**

- **Declarative:** Git is the single source of truth for cluster state.
- **Auditable:** All changes to the cluster state are Git commits.
- **Consistency:** Easily replicate environments or recover from disasters using Git history.
- **Automation:** Continuous reconciliation ensures the cluster matches the desired state.

Implementing GitOps requires setting up Argo CD or Flux in your cluster, but it provides a robust and scalable deployment paradigm, especially for complex Kubernetes environments.

### 2. Deploying to Container Orchestration Services (ECS, Fargate, Cloud Run)

Managed container services from cloud providers offer simpler alternatives to full Kubernetes.

- **AWS Elastic Container Service (ECS) / Fargate:** Deployments typically involve updating an ECS Task Definition with the new container image URI and then updating the ECS Service to use the new Task Definition revision. Fargate is a serverless compute engine for ECS, removing the need to manage underlying EC2 instances.
- **Google Cloud Run:** A fully managed serverless platform for containers. Deployment involves submitting a new container image, and Cloud Run handles scaling and infrastructure automatically.
- **Azure Container Apps:** A serverless application-centric hosting service built on Kubernetes primitives.

#### [Configuration Guide: Authenticating with AWS/GCP/Azure]

Similar to Kubernetes, authentication is crucial.

- **OIDC (Recommended):** Use the official actions (`aws-actions/configure-aws-credentials`, `google-github-actions/auth`, `azure/login`) with OIDC federation for secure, keyless authentication. This is the preferred method.
- **Static Credentials (Less Secure):** Store cloud provider access keys/service principal credentials as GitHub Secrets. Use the same official actions, configuring them with the secrets instead of OIDC. Manage these secrets carefully (rotation, least privilege).

Refer to Chapter 3 for secrets management and Chapter 19 for detailed cloud provider integration patterns.

#### [Practical Example: Updating an ECS service or deploying to Cloud Run]

**Example 1: Updating an AWS ECS Service**

This workflow assumes OIDC is configured for AWS access.

```yaml
name: Deploy to AWS ECS

on:
  push:
    branches:
      - main

permissions:
  id-token: write # for OIDC
  contents: read

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Configure AWS Credentials (OIDC)
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789012:role/GitHubActionsECSRole # Your IAM Role ARN
          aws-region: us-west-2 # Your AWS Region

      - name: Build, tag, and push image to Amazon ECR
        id: build-image
        env:
          ECR_REGISTRY: 123456789012.dkr.ecr.us-west-2.amazonaws.com # Your ECR Registry
          ECR_REPOSITORY: my-app-repo # Your ECR Repo
          IMAGE_TAG: ${{ github.sha }}
        run: |
          # Login to ECR
          aws ecr get-login-password --region $AWS_REGION | docker login --username AWS --password-stdin $ECR_REGISTRY
          # Build and push
          docker build -t $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG .
          docker push $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG
          echo "image=$ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG" >> $GITHUB_OUTPUT

      - name: Download existing task definition
        run: |
          aws ecs describe-task-definition --task-definition my-app-task-def \
            --query taskDefinition > task-definition.json

      - name: Fill in the new image ID in the Amazon ECS task definition
        id: task-def
        uses: aws-actions/amazon-ecs-render-task-definition@v1
        with:
          task-definition: task-definition.json
          container-name: my-app-container # Container name in your task definition
          image: ${{ steps.build-image.outputs.image }}

      - name: Deploy Amazon ECS task definition
        uses: aws-actions/amazon-ecs-deploy-task-definition@v1
        with:
          task-definition: ${{ steps.task-def.outputs.task-definition }}
          service: my-app-service # Name of the ECS service to update
          cluster: my-ecs-cluster # Name of the ECS cluster
          wait-for-service-stability: true # Wait for deployment to complete
```

**Example 2: Deploying to Google Cloud Run**

This workflow assumes OIDC (Workload Identity Federation) is configured for GCP access.

```yaml
name: Deploy to Google Cloud Run

on:
  push:
    branches:
      - main

permissions:
  id-token: write # for OIDC
  contents: read

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Authenticate to Google Cloud (OIDC)
        uses: google-github-actions/auth@v2
        with:
          workload_identity_provider: "projects/YOUR_PROJECT_NUMBER/locations/global/workloadIdentityPools/YOUR_POOL_ID/providers/YOUR_PROVIDER_ID"
          service_account: "your-service-account@your-project-id.iam.gserviceaccount.com" # Service account email

      - name: Set up Cloud SDK
        uses: google-github-actions/setup-gcloud@v2

      - name: Build and Push Docker Image to Google Artifact Registry
        id: build-image
        run: |
          IMAGE_TAG=${GITHUB_SHA}
          IMAGE_NAME="us-central1-docker.pkg.dev/your-project-id/your-repo/my-app:${IMAGE_TAG}" # Update with your GAR location/repo
          # Configure Docker auth
          gcloud auth configure-docker us-central1-docker.pkg.dev # Update region if needed
          # Build and push
          docker build -t ${IMAGE_NAME} .
          docker push ${IMAGE_NAME}
          echo "image_name=${IMAGE_NAME}" >> $GITHUB_OUTPUT

      - name: Deploy to Cloud Run
        id: deploy
        uses: google-github-actions/deploy-cloudrun@v2
        with:
          service: my-cloud-run-service # Your Cloud Run service name
          region: us-central1 # Your Cloud Run region
          image: ${{ steps.build-image.outputs.image_name }}
          # Optional: Set environment variables, traffic allocation, etc.
          # env_vars: |
          #   API_URL=https://api.example.com
          #    GITHUB_SHA=${{ github.sha }}
          # traffic: |
          #   latest=true

      - name: Show Output URL
        run: echo "Deployed to ${{ steps.deploy.outputs.url }}"
```

These examples leverage official GitHub Actions provided by cloud vendors, simplifying the interaction with their respective services.

## D. Deploying to Serverless Platforms (AWS Lambda, Azure Functions, Google Cloud Functions)

Serverless functions require packaging code and dependencies and uploading them to the cloud provider.

### 1. Packaging and Deploying Function Code

The basic process involves:

1.  **Install Dependencies:** Gather necessary libraries (e.g., using `pip install -r requirements.txt -t ./package` for Python, `npm install --production` for Node.js).
2.  **Package:** Create an archive (usually a ZIP file) containing your function code and its dependencies.
3.  **Deploy:** Use the cloud provider's CLI or SDK to upload the package and update the function configuration.

#### [Code Snippet: Deploying a Python Lambda function using AWS CLI]

This assumes OIDC is configured for AWS access and the Lambda function `my-lambda-function` already exists.

```yaml
name: Deploy Python Lambda Function

on:
  push:
    branches:
      - main
      paths: # Only run if function code or requirements change
        - 'src/my_lambda/**'
        - 'src/requirements.txt'

permissions:
  id-token: write # for OIDC
  contents: read

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.11'

      - name: Configure AWS Credentials (OIDC)
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789012:role/GitHubActionsLambdaRole # Your IAM Role ARN
          aws-region: eu-west-1 # Your AWS Region

      - name: Install dependencies and package function
        working-directory: ./src # Assuming function code is in src/my_lambda and requirements in src/
        run: |
          pip install -r requirements.txt -t ./package
          cp -r my_lambda/* ./package/
          cd package
          zip -r ../lambda_function.zip .

      - name: Deploy to AWS Lambda
        run: |
          aws lambda update-function-code \
            --function-name my-lambda-function \
            --zip-file fileb://src/lambda_function.zip \
            --region eu-west-1 # Ensure region matches function's region
          # Optional: Publish a new version
          # aws lambda publish-version --function-name my-lambda-function --region eu-west-1
          # Optional: Update function configuration (e.g., environment variables)
          # aws lambda update-function-configuration --function-name my-lambda-function --environment "Variables={API_KEY=${{ secrets.API_KEY }}}"
```

### 2. Using Frameworks (Serverless Framework, SAM)

Frameworks like the Serverless Framework or AWS Serverless Application Model (SAM) simplify defining, packaging, and deploying serverless applications (including functions, APIs, databases, etc.) using a declarative configuration file.

#### [Practical Example: Deploying an application using the Serverless Framework]

This example assumes you have a `serverless.yml` file defining your application and OIDC configured for your cloud provider (AWS in this case).

```yaml
name: Deploy with Serverless Framework

on:
  push:
    branches:
      - main

permissions:
  id-token: write # for OIDC
  contents: read

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Set up Node.js (for Serverless Framework CLI)
        uses: actions/setup-node@v4
        with:
          node-version: "18" # Or your preferred version

      - name: Install Serverless Framework
        run: npm install -g serverless

      - name: Configure AWS Credentials (OIDC)
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789012:role/GitHubActionsServerlessRole # Your IAM Role ARN
          aws-region: us-east-1 # Your AWS Region

      - name: Install project dependencies
        run: npm install # Or yarn install, pip install, etc. depending on your project

      - name: Deploy Serverless Application
        run: serverless deploy --stage prod # Or your target stage (e.g., dev, staging)
        env:
          # Pass secrets or config needed by serverless.yml via environment variables
          MY_API_KEY: ${{ secrets.MY_API_KEY_PROD }}
```

Using frameworks like Serverless or SAM is highly recommended for managing anything beyond trivial serverless functions, as they handle packaging, resource provisioning (via CloudFormation/Terraform), and deployment complexities.

## E. Deploying to Platform-as-a-Service (PaaS) (Heroku, Vercel, Netlify, Render)

PaaS platforms abstract infrastructure management, offering streamlined deployment workflows, often tightly integrated with Git.

### 1. Using Platform-Specific CLIs or Actions

Most PaaS providers offer a CLI tool and/or dedicated GitHub Actions to facilitate deployment. Authentication usually involves storing an API token from the PaaS provider as a GitHub Secret.

#### [Practical Example: Deploying a Node.js app to Heroku]

Heroku deployments can often be done via `git push`, but using the Heroku CLI or dedicated actions provides more control. This example uses the `akhileshns/heroku-deploy` action.

```yaml
name: Deploy Node.js App to Heroku

on:
  push:
    branches:
      - main

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      # Optional: Setup Node.js and build if needed before deployment
      # - name: Setup Node.js
      #   uses: actions/setup-node@v4
      #   with:
      #     node-version: '18'
      # - name: Install Dependencies & Build
      #   run: |
      #     npm install
      #     npm run build

      - name: Deploy to Heroku
        uses: akhileshns/heroku-deploy@v3.13.15 # Use the latest version
        with:
          heroku_api_key: ${{ secrets.HEROKU_API_KEY }}
          heroku_app_name: "my-heroku-app-name" # Replace with your Heroku app name
          heroku_email: "user@example.com" # Replace with your Heroku account email
          # Optional: Specify branch, buildpacks, healthcheck, etc.
          # branch: "main"
          # buildpack: "https://github.com/heroku/heroku-buildpack-nodejs"
          # healthcheck: "https://my-heroku-app-name.herokuapp.com/"
```

#### [Code Snippet: Using the Vercel CLI for deployment]

Vercel deployments are often automated via their Git integration, but the CLI allows for more explicit control within a workflow.

```yaml
name: Deploy to Vercel

on:
  push:
    branches:
      - main

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Set up Node.js
        uses: actions/setup-node@v4
        with:
          node-version: "18"

      - name: Install Vercel CLI
        run: npm install -g vercel

      # Optional: Build step if your project requires it
      # - name: Build Project
      #   run: npm run build
      #   env:
      #     # Set build-time environment variables if needed
      #     NEXT_PUBLIC_API_URL: "https://api.example.com"

      - name: Deploy to Vercel Production
        run: vercel --prod --token ${{ secrets.VERCEL_TOKEN }} --yes
        # '--prod' promotes the deployment to production alias
        # '--yes' skips confirmation prompts
        # Vercel CLI automatically detects project settings, or you can specify them
        env:
          # Set Vercel environment variables if needed (can also be set in Vercel UI)
          VERCEL_ORG_ID: ${{ secrets.VERCEL_ORG_ID }}
          VERCEL_PROJECT_ID: ${{ secrets.VERCEL_PROJECT_ID }}
```

Always consult the specific PaaS provider's documentation for their recommended deployment methods and GitHub Actions integration options.

## F. Environment Protection Rules and Approvals

For critical environments like `staging` or `production`, simply triggering a deployment on every push to `main` is often too risky. GitHub Actions allows you to define **Environments** and apply protection rules to control deployments.

### 1. Defining Environments in GitHub

Environments are configured in your repository settings:

- Go to your repository > Settings > Environments.
- Click "New environment".
- Give it a name (e.g., `production`, `staging`). This name will be referenced in your workflow files using the `environment:` key in a job.

### 2. Configuring Required Reviewers for Deployment

Within an environment's settings, you can require specific users or teams to approve a deployment before it proceeds.

- Edit the environment settings.
- Check "Required reviewers".
- Search for and add the GitHub users or teams who must approve deployments to this environment.
- When a workflow job targeting this environment starts, it will pause, and notifications will be sent to the reviewers. The workflow only continues once the required number of reviewers approve it via the GitHub UI.

### 3. Setting Wait Timers

You can configure a delay before a job targeting a protected environment begins.

- Edit the environment settings.
- Check "Wait timer".
- Enter the number of minutes (0-43,200, which is 30 days) the workflow should wait after being triggered before starting the deployment job.
- This can be useful for allowing time to observe monitoring dashboards after a related deployment or to cancel a deployment if issues are detected quickly.

### 4. Restricting Branches/Tags that Can Deploy

You can specify which branches or tags are allowed to deploy to a particular environment.

- Edit the environment settings.
- Under "Deployment branches", select:
  - **All branches:** Any branch can deploy (least restrictive).
  - **Protected branches only:** Only branches configured with branch protection rules can deploy.
  - **Selected branches:** Specify exact branch names or patterns (e.g., `main`, `release/*`, `v*.*.*`) that are allowed to deploy.

#### [Configuration Guide: Setting up a protected 'production' environment with manual approval]

1.  **Navigate:** Go to your GitHub repository > Settings > Environments.
2.  **Create Environment:** Click "New environment", name it `production`, and click "Configure environment".
3.  **Add Reviewers:** Check the "Required reviewers" box. Search for and add at least one user or team who must approve production deployments.
4.  **Restrict Branches:** Under "Deployment branches", select "Selected branches". Click "Add deployment branch rule". Enter `main` (or your primary production branch/tag pattern like `refs/tags/v*`).
5.  **Add Secrets (Optional but Recommended):** Add any secrets specific to production (e.g., `DATABASE_PASSWORD`, `API_KEY`) under "Environment secrets". These will only be available to jobs deploying to the `production` environment.
6.  **Save:** Click "Save protection rules".
7.  **Update Workflow:** In your deployment workflow file, add the `environment` key to the job that performs the production deployment:

    ```yaml
    jobs:
      deploy_production:
        runs-on: ubuntu-latest
        environment:
          name: production # References the configured environment
          url: https://myapp.example.com # Optional: Sets deployment URL shown in GitHub UI
        # Add permissions for OIDC if needed
        permissions:
          id-token: write
          contents: read
          deployments: write # Required for environment URL and status updates
        if: github.ref == 'refs/heads/main' # Or use tag trigger: starts_with(github.ref, 'refs/tags/v')
        steps:
          # ... checkout, build, configure credentials ...

          - name: Deploy to Production Environment
            run: |
              echo "Deploying to production..."
              # Add your production deployment commands here
              # e.g., helm upgrade ..., aws ecs deploy ..., serverless deploy --stage prod ...
    ```

Now, when this workflow runs on a push/tag matching the trigger and branch/tag rules, the `deploy_production` job will pause, waiting for approval from the configured reviewers before executing the deployment steps.

#### [Production Note: Implementing robust deployment gates]

Environment protection rules are essential guardrails for production stability and security.

- **Use for Critical Environments:** Always protect `production` and potentially `staging`.
- **Combine Rules:** Use a combination of required reviewers, branch restrictions, and potentially wait timers for layered protection.
- **Least Privilege Reviewers:** Only grant review permissions to necessary personnel.
- **Audit Trail:** Approvals and deployments are logged in the GitHub UI, providing an audit trail.
- **Beyond GitHub:** These rules control the _workflow execution_. Ensure your deployment _scripts/tools_ also have appropriate safeguards and confirmations, especially for destructive actions.

## G. Rollback Strategies

Despite careful testing and deployment gates, issues can still occur in production. Having a plan to quickly revert to a previous working state (rollback) is crucial.

### 1. Manual Rollback Procedures

The simplest form of rollback involves manual intervention:

- **Re-run Previous Workflow:** Find the last successful deployment workflow run in the GitHub Actions UI and re-run it. This redeploys the previous stable version of the code/image. This requires the workflow to be designed for re-runnability.
- **Manual CLI/UI Commands:** Use the deployment tool's CLI (e.g., `helm rollback <release-name> <revision>`, `kubectl rollout undo deployment/<name>`, cloud provider console) to revert the changes directly on the infrastructure.
- **Revert Git Commit:** Revert the Git commit that triggered the faulty deployment and push the revert commit. This will trigger the CI/CD pipeline again, deploying the previous code version (assuming the pipeline deploys the latest commit on the target branch).

Manual rollbacks require operator intervention and can be slower, but they are often the starting point. Documenting the specific manual rollback steps for your application is essential.

### 2. Automating Rollbacks on Failure

GitHub Actions can help automate rollbacks, typically by adding steps that run conditionally if the main deployment step fails.

- **Conditional Steps:** Use `if: failure()` on a subsequent step in the deployment job to trigger rollback commands if an earlier step (like the main deployment command) fails.
- **Dedicated Rollback Job:** Have a separate job that runs `if: failure()` and depends on the deployment job (`needs: deploy_job`).
- **Tool-Specific Rollback:** Leverage features within deployment tools. For example, `helm upgrade` has an `--atomic` flag that attempts to automatically roll back the release if the upgrade fails. Similarly, some cloud deployment services offer automatic rollback capabilities.

#### [Deep Dive: Designing workflows for automated canary or blue-green deployment rollbacks]

More advanced deployment strategies like Canary or Blue-Green deployments inherently involve managing multiple versions and provide smoother rollback paths. GitHub Actions can orchestrate these, although the complexity increases significantly.

- **Blue-Green Deployment:**

  1.  **Deploy New Version (Green):** Deploy the new application version to a separate, identical environment ("Green") alongside the currently live version ("Blue").
  2.  **Test Green:** Run automated health checks and tests against the Green environment.
  3.  **Shift Traffic:** If tests pass, update the load balancer/router to direct live traffic to the Green environment. Green becomes the new Blue.
  4.  **(Rollback Path):** If tests fail on Green _or_ issues arise after shifting traffic, simply switch the router back to the original Blue environment. The faulty Green environment can be investigated or terminated.

  - **Actions Role:** The workflow would contain steps to provision/update the Green environment, run tests, trigger the traffic switch (e.g., via cloud provider CLI/API), and potentially trigger a switch-back (`if: failure()`) or tear down the old Blue environment after success.

- **Canary Deployment:**
  1.  **Deploy Canary:** Deploy the new version alongside the stable version but initially route only a small percentage of traffic (e.g., 1%, 5%) to the new "Canary" version.
  2.  **Monitor:** Closely monitor error rates, latency, and business metrics for the Canary traffic. Automated health checks are crucial.
  3.  **Gradual Rollout:** If the Canary performs well, gradually increase the traffic percentage routed to it (e.g., 10%, 25%, 50%, 100%).
  4.  **(Rollback Path):** If monitoring detects issues at any stage, immediately shift all traffic back to the stable version by setting the Canary traffic weight to 0%.
  - **Actions Role:** The workflow would deploy the new version, then make calls (e.g., via CLI/API to a service mesh like Istio, a load balancer, or a PaaS feature) to configure the initial traffic split. Subsequent steps (potentially manual triggers or scheduled workflows) could increase the traffic percentage based on monitoring data (or integrate with monitoring tools). A failure condition (`if: failure()`) in monitoring steps or the deployment itself could trigger an immediate traffic shift back to 100% stable.

**Automation Complexity:** Fully automating rollbacks, especially for Canary/Blue-Green, often requires sophisticated health checks, monitoring integration, and careful state management within the workflow or reliance on underlying platform capabilities (like service meshes or PaaS deployment features). GitHub Actions acts as the orchestrator, triggering commands and reacting to outcomes, but the core logic often resides in specialized deployment tools or scripts.

#### [Note: Further author detail needed to fully scope advanced deployment patterns]

While this chapter introduces the concepts, fully detailing the implementation of automated Blue-Green or Canary rollbacks with GitHub Actions, including integration with specific service meshes (Istio, Linkerd), cloud provider load balancers (ALB, Application Gateway), or PaaS features (Heroku Preboot, Vercel/Netlify features), requires deeper dives that are beyond the scope of this introductory chapter. These advanced patterns often involve complex workflow logic, API interactions, and robust monitoring integration.

---

This chapter provided a comprehensive overview of deploying applications using GitHub Actions. We covered fundamental concepts, explored strategies for various deployment targets (VMs, Kubernetes, Serverless, PaaS), emphasized the importance of security through secrets and environment protection rules, and introduced rollback strategies.

Armed with this knowledge, you can now begin designing and implementing robust, automated deployment pipelines tailored to your specific infrastructure and application needs. The next chapter will delve into advanced workflow techniques, building upon the foundations laid so far to unlock even more powerful automation possibilities.
