# Setting Up GitHub Actions Self-Hosted Runners in an AWS VPC

Many organizations run critical workloads inside AWS Virtual Private Clouds (VPCs) for security and compliance reasons. When using GitHub Actions, the default runners are hosted by GitHub in Azure, which means they won't have network access to resources inside your private AWS VPC.

To run GitHub Actions workflows that interact with services or software inside your VPC, you need to configure **self-hosted runners** inside your AWS environment.

This guide provides a detailed, step-by-step process for deploying and managing self-hosted GitHub Actions runners inside your AWS VPC—covering both EC2 and containerized (ECS/EKS) approaches.

---

## Table of Contents

1. [Overview](#overview)
1. [Prerequisites](#prerequisites)
1. [Architecture Options](#architecture-options)
1. [EC2 Approach: Self-Hosted Runner on EC2](#ec2-approach-self-hosted-runner-on-ec2)
1. [ECS/EKS Approach: Self-Hosted Runners in Containers](#ecseks-approach-self-hosted-runners-in-containers)
   - [ECS (Fargate/EC2)](#ecs-fargateec2)
   - [EKS (Kubernetes)](#eks-kubernetes)
1. [Security Considerations](#security-considerations)
1. [Managing and Monitoring Runners](#managing-and-monitoring-runners)

---

## Overview

- **Why self-hosted runners?**  
  Self-hosted runners allow you to execute GitHub Actions workflows on machines you control—inside your AWS VPC—giving them secure access to VPC-only resources.
- **Where do they run?**  
  On EC2 instances, ECS/EKS-managed containers, or on-prem VMs inside your VPC.

---

## Prerequisites

- **AWS Account** with permissions to launch compute resources (EC2, ECS, EKS).
- **GitHub Repository** or Organization where you want to register the runner.
- **IAM Role/User** with permissions to manage resources.
- **Basic familiarity** with AWS Console or CLI.

---

## Architecture Options

1. **EC2 Instance(s) as Runners:**  
   The simplest and most common approach—spin up EC2 instances inside your VPC and install the runner agent.

2. **ECS/EKS/Containers:**  
   For larger scale or dynamic workloads, you can run runners in containers using ECS (Elastic Container Service) or EKS (Elastic Kubernetes Service).

3. **On-premises Machines:**  
   If you have on-prem resources connected via Direct Connect or VPN, you can host runners there too.

Below, we describe both the EC2 and ECS/EKS approaches.

---

## EC2 Approach: Self-Hosted Runner on EC2

### 1. Launch an EC2 Instance

- Choose an AMI (Amazon Linux 2, Ubuntu, etc.).
- Place the instance **inside your VPC** and **subnet** with access to your private resources.
- Assign appropriate **security groups** (SSH access for admin, outbound HTTPS for GitHub connectivity).
- Attach an **IAM role** if the runner needs to interact with AWS services.

### 2. Install GitHub Runner Software

#### a. Connect to your EC2 instance

Use SSH or AWS SSM to connect.

#### b. Install prerequisites

Most runners require `curl`, `tar`, and `unzip`. On Ubuntu:

```sh
sudo apt-get update
sudo apt-get install -y curl tar
```

#### c. Download the runner package

Go to your GitHub repo > Settings > Actions > Runners > "Add runner", choose your OS, and copy the download link.

Or run:

```sh
# Replace with the latest version and your preferred directory
mkdir actions-runner && cd actions-runner
curl -o actions-runner-linux-x64.tar.gz -L https://github.com/actions/runner/releases/download/v2.XXX.X/actions-runner-linux-x64-2.XXX.X.tar.gz
tar xzf actions-runner-linux-x64.tar.gz
```

#### d. Configure the runner

Still in the GitHub repo > Settings > Actions > Runners > "Add runner", you'll see a command like:

```sh
./config.sh --url https://github.com/OWNER/REPO --token <TOKEN>
```

Run this on your EC2 instance, replacing `<TOKEN>` with the one provided by GitHub.

#### e. Start the runner

```sh
./run.sh
```

For background/service mode (recommended for production):

```sh
sudo ./svc.sh install
sudo ./svc.sh start
```

### 3. Verify Runner Registration

- Go back to the `GitHub repo > Settings > Actions > Runners`.
- You should see your new runner listed as `online`.

---

## ECS/EKS Approach: Self-Hosted Runners in Containers

Running GitHub Actions runners in containers (ECS/EKS) enables you to scale runners dynamically, automate their lifecycle, and manage them as code.

### ECS (Fargate/EC2)

#### 1. Prepare a Runner Container Image

- Use an official or community-provided Docker image for the GitHub Actions runner. Example: [myoung34/docker-github-actions-runner](https://hub.docker.com/r/myoung34/github-runner).
- You can build your own image if you need custom tools.

#### 2. Store the Runner Registration Token

- Runner tokens expire quickly and must be fetched dynamically.
- Use AWS Secrets Manager, SSM Parameter Store, or environment variables to pass tokens securely.

#### 3. Create an ECS Task Definition

- Define a container using your runner image.
- Pass environment variables for repo/org URL and token.
- Mount volumes if persistent storage is needed.

Example task definition snippet:

```json
{
  "containerDefinitions": [
    {
      "name": "github-actions-runner",
      "image": "myoung34/github-runner:latest",
      "environment": [
        { "name": "REPO_URL", "value": "https://github.com/OWNER/REPO" },
        { "name": "RUNNER_TOKEN", "value": "<TOKEN>" }
      ]
    }
  ]
}
```

- Place the ECS service in the desired VPC/subnet for private resource access.

#### 4. Launch the Runner

- Start the ECS task (Fargate or EC2 launch type).
- The runner registers itself with GitHub and polls for jobs.

#### 5. Scale

- Use ECS Service Auto Scaling to adjust runner count based on demand.
- Consider one-runner-per-job (ephemeral) models for security and isolation.

---

### EKS (Kubernetes)

#### 1. Use GitHub Actions Runner Controller

- [actions-runner-controller](https://github.com/actions/actions-runner-controller) is an open source project for managing GitHub runners as Kubernetes resources.
- It spins up runners on-demand as Kubernetes pods.

#### 2. Install actions-runner-controller

- Deploy the controller with Helm or kubectl.
- Store runner registration tokens as Kubernetes Secrets.

#### 3. Configure Runner Deployment

- Create a `RunnerDeployment` or `RunnerSet` manifest specifying:
  - The GitHub repository or organization.
  - Number of replicas (runners).
  - Docker image for the runner.
- Place the pods in subnets with VPC access to your resources.

Example manifest:

```yaml
apiVersion: actions.summerwind.dev/v1alpha1
kind: RunnerDeployment
metadata:
  name: example-runner-deployment
spec:
  replicas: 2
  template:
    spec:
      repository: your-org/your-repo
```

#### 4. Monitor and Scale

- The controller scales runners based on job demand.
- Runners can be ephemeral (one job per pod) for security.

References:

- [actions-runner-controller docs](https://github.com/actions/actions-runner-controller)
- [GitHub Docs: Self-hosted runners with Kubernetes](https://docs.github.com/en/actions/hosting-your-own-runners/using-a-kubernetes-cluster)
- [AWS blog: GitHub Actions runners on EKS](https://aws.amazon.com/blogs/opensource/ephemeral-github-actions-runners-with-eks/)

---

## Security Considerations

- **Runner Permissions:**  
  Limit runner instance/container IAM permissions to only what's necessary.
- **Network Access:**  
  Security groups should only allow required inbound access (e.g., SSH/K8s API from your IP) and allow outbound HTTPS to `github.com`.
- **Runner Isolation:**  
  Consider using dedicated or ephemeral runners for sensitive workflows.
- **Reset/Rotate Runner Tokens:**  
  Tokens expire after a short time; do not share or hard-code them.
- **Secrets Management:**  
  Use AWS Secrets Manager, SSM Parameter Store, or Kubernetes Secrets for tokens and credentials.

## Managing and Monitoring Runners

- **Scaling:**  
  Use ECS/EKS scaling policies or automation for dynamic workloads.
- **Updates:**  
  Keep the runner agent/container updated for security and features.
- **Logs:**  
  GitHub Actions logs are viewable in the GitHub UI. System/container logs are in CloudWatch/EKS logs.

## References

- [GitHub Docs: Adding self-hosted runners](https://docs.github.com/en/actions/hosting-your-own-runners/adding-self-hosted-runners)
- [GitHub Docs: Self-hosted runner security](https://docs.github.com/en/actions/hosting-your-own-runners/using-self-hosted-runners-in-a-workflow)
- [actions-runner-controller (Kubernetes)](https://github.com/actions/actions-runner-controller)
- [myoung34/docker-github-actions-runner](https://github.com/myoung34/docker-github-actions-runner)

---

## FAQ

**Q: Does the runner need internet access?**  
A: Yes, at minimum the runner must be able to reach `github.com` over HTTPS.

**Q: Can I use private runners at the organization level?**  
A: Yes, follow the same process but register the runner at the organization scope.

**Q: Can I make the runner auto-scale?**  
A: Yes, ECS/EKS approaches are designed for scaling. Use controller or automation scripts.
