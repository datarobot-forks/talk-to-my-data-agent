# Using GitHub-Hosted Larger Runners with Static IP Addresses

For many teams, managing self-hosted runners in AWS VPC (via EC2, ECS, or EKS) can be complex and require DevOps expertise. If your primary goal is to run GitHub Actions jobs that need to be whitelisted by IP address (for example, to access resources behind a firewall or in a VPC), GitHub offers a simpler alternative: **GitHub-hosted larger runners with static (fixed) egress IP addresses**.

This guide explains how to use GitHub-hosted larger runners with static IPs, what to expect, and how to configure your environment to leverage this feature.

---

## Table of Contents

1. [Overview](#overview)
1. [When to Use GitHub-Hosted Larger Runners with Static IPs](#when-to-use-github-hosted-larger-runners-with-static-ips)
1. [How Static IPs Work](#how-static-ips-work)
1. [Enabling and Configuring Larger Runners with Static IPs](#enabling-and-configuring-larger-runners-with-static-ips)
1. [Finding and Using the Static IP Address List](#finding-and-using-the-static-ip-address-list)
1. [Configuring Your Systems to Allow GitHub Runner Traffic](#configuring-your-systems-to-allow-github-runner-traffic)
1. [Limitations and Considerations](#limitations-and-considerations)
1. [FAQ](#faq)

---

## Overview

**GitHub-hosted runners** are managed by GitHub and run on GitHub's infrastructure. Traditionally, their outbound IP addresses change frequently, which can be problematic for accessing private resources that require IP allowlisting.

**GitHub now offers larger runners with static egress IPs** (sometimes called "fixed IP ranges" or "static IP pools"). You can use these to configure your firewalls or VPCs to allow only GitHub Actions jobs from these known IPs.

---

## When to Use GitHub-Hosted Larger Runners with Static IPs

- When you do **not** want to manage your own infrastructure (EC2, ECS, EKS, etc.).
- When your private resources (APIs, databases, internal services) are behind a firewall or in a VPC and require IP allowlisting.
- When you want to keep the simplicity and scalability of GitHub-hosted runners.

---

## How Static IPs Work

- GitHub provides **larger runners** with static egress IP addresses in select regions.
- You can configure your organization or repository to use these runners.
- GitHub publishes and maintains a list of the static IP addresses used by these runners.
- You whitelist these IPs on your infrastructure/firewall.
- All outbound traffic from your workflows will originate from these static IP addresses.

---

## Enabling and Configuring Larger Runners with Static IPs

### 1. **Check Your GitHub Plan**

- Larger runners with static IPs are available to organizations on the **GitHub Enterprise Cloud** plan. [See details](https://docs.github.com/en/actions/using-github-hosted-runners/using-larger-runners).

### 2. **Provision a Larger Runner**

- Go to your organization settings:  
  `Organization Settings > Actions > Runners > New GitHub-hosted runner`
- Select your desired hardware size (e.g., 2-64 vCPUs, higher RAM).
- In the configuration, you can **request static IP addresses** for your runner(s).  
  [Managing larger runners](https://docs.github.com/en/actions/using-github-hosted-runners/using-larger-runners/managing-larger-runners)

### 3. **Update Your Workflow File**

Use the provided runner group label or the name you assigned to your larger runner in your workflow YAML:

```yaml
jobs:
  build:
    runs-on: my-org/ubuntu-20.04-16core
    steps:
      - uses: actions/checkout@v4
      - name: Run your build
        run: |
          echo "Hello from a larger runner!"
```

> The exact `runs-on` value will be shown in your organization's runner settings.

### 4. **Wait for Runner Provisioning**

- It may take a few minutes for the runner to be ready.
- Once online, it will have egress traffic from the assigned static IP range.

---

## Finding and Using the Static IP Address List

- For each larger runner you provision with static IPs, GitHub assigns a specific range.
- You can view your organization's assigned static IP ranges in the **runner management UI**:  
  `Organization Settings > Actions > Runners > [Your Larger Runner]`
- For general information, see:  
  [Managing larger runners - GitHub Docs](https://docs.github.com/en/actions/using-github-hosted-runners/using-larger-runners/managing-larger-runners)
- Announcement and details about dual static IP ranges:  
  [Changelog: Dual Static IP ranges for GitHub-hosted Larger runners](https://github.blog/changelog/2023-09-06-dual-static-ip-ranges-for-github-hosted-larger-runners/)
- **Note**: The global list of GitHub's public IPs (not specific to your static range) is available at [GitHub's IP addresses](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/about-githubs-ip-addresses) and via the [GitHub Meta API](https://api.github.com/meta).

---

## Configuring Your Systems to Allow GitHub Runner Traffic

1. **Obtain the Assigned Static IP Range**
   - From your organization’s runner settings as above.

2. **Whitelist the IPs**
   - Update your firewall, API gateway, or VPC security group to allow inbound connections from your assigned GitHub Actions static IP range(s).

3. **Automate IP Updates (Recommended)**
   - While static, GitHub may update assigned ranges in rare circumstances; set a process to review.

---

## Limitations and Considerations

- **Availability:**  
  Static IP runners are only available with certain GitHub plans (Enterprise Cloud).
- **No Inbound Connections:**  
  GitHub-hosted runners cannot receive inbound connections—they can only make outbound requests.
- **Billing:**  
  Larger runners are billed per-minute and may be more expensive than standard runners.
- **Egress Only:**  
  Only outbound connections from the runner use the static IP; inbound is not supported.
- **IP Rotation:**  
  Static IP ranges are assigned to your organization but may change with notice.
- **Capacity:**  
  Larger runners are subject to regional capacity; plan accordingly for peak usage.

## FAQ

**Q: What if my organization needs more control or guaranteed capacity?**  
A: Larger runners offer higher capacity and static IPs. For even more control, consider [self-hosted runners](https://docs.github.com/en/actions/hosting-your-own-runners/about-self-hosted-runners).

**Q: How do I know which runner labels to use?**  
A: See your runner group and label in your organization's runner settings after provisioning.

**Q: What happens if GitHub changes the static IP addresses?**  
A: GitHub will notify you in advance; monitor your runner settings and update your allowlists as needed.

**Q: Can I use these runners to access resources inside my AWS VPC?**  
A: Yes, if your AWS security group/firewall allows inbound traffic from your assigned static IP range.

**Q: Can I use static IP runners for on-prem resources?**  
A: Yes, as long as your firewall allows outbound connections from the static IP range.
