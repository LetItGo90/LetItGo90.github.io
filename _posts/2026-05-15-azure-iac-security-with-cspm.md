---
title: "Building an Azure Cloud Security Platform from Scratch"
date: 2026-05-15
categories: [Azure, Security]
tags: [azure, terraform, cspm, prowler, cloud-custodian, logic-apps, devsecops]
render_with_liquid: false
---

I've been trying to build as much projects/lab's as possible. I've been super interested in transiting into a more cloud sec focused role and i've got all the fancy certs. So I thought its time to buidl stuff. I've done several simialr projects to this before , here: [Azure Vulnerability Scanning](https://github.com/LetItGo90/Azure-Vulnerability-Scanning-Compliance-Platform), [az_auth_project](https://github.com/LetItGo90/az_auth_project), and [Azure Secure Landing Zone](https://github.com/LetItGo90/Azure-Secure-Landing-Zone-Homelab-Project/). In this one I wanted to try and combine elements of them and expand on them. Iaac skills take a lot of muscle memory to remeber, what to do and why.

The repo is here if you want to follow along: [LetItGo90/Azure-Cloud-Security-Platform](https://github.com/LetItGo90/Azure-Cloud-Security-Platform)

## The Goal

The idea was to build a complete Azure security platform using Terraform with real infrastructure, real security controls, and real scanning, then actually run posture management tooling against it to see what comes back. I intentionally introduced some misconfigurations along the way because a perfectly clean environment doesn't teach you anything. You want findings. You want to see the tools catch something real.

## The Architecture

The foundation is a hub-spoke network topology. The hub holds Azure Firewall which controls all traffic flowing between spokes. Nothing communicates with anything it isn't supposed to. All the shared security services live in the hub and the spoke is where workloads sit.

For the architecture diagram I used mermaid. If you didn't know, you can generate architecture diagrams straight in github. Granted it can look a bit ugly and will take some tweaking but for small homelab projects it gets the job done.

<img width="1732" height="689" alt="image" src="https://github.com/user-attachments/assets/3ebcc853-19d6-487b-accc-682fea869382" />

For identity I went zero-trust all the way. GitHub Actions authenticates to Azure using OIDC federation so there are no long-lived secrets stored in GitHub anywhere. The pipeline gets a short-lived token, does its job, and that's it. Inside Azure everything uses Managed Identities for service-to-service auth. No passwords, no keys being passed around.

PaaS services like Key Vault and Storage are locked behind Private Endpoints so they aren't exposed to the public internet at all. Customer Managed Keys handle encryption, meaning Azure isn't holding the keys to your data, you are.

## Standing It Up With Terraform

Everything in this project is Terraform. Every resource, every role assignment, every policy definition. If it can't be expressed as code it doesn't exist in this environment.

The whole thing runs through GitHub Actions on push to main. Terraform plan runs on every PR so you can see exactly what's changing before it goes anywhere near the environment. On merge it applies automatically. No manual runs, no local state, no "it works on my machine."

<img width="1904" height="873" alt="image" src="https://github.com/user-attachments/assets/560b59eb-afa7-40ad-81ae-042a2bd1c7d5" />

<img width="1909" height="304" alt="image" src="https://github.com/user-attachments/assets/34131711-eb1e-4cb6-bb4d-a76d30970f2a" />

<img width="1915" height="290" alt="image" src="https://github.com/user-attachments/assets/3af87b3f-1fa4-4dee-84e2-95b031c8c7e7" />

The Terraform workflow handles plan, format check, validation, a Checkov IaC security scan, and then apply all in one pipeline:

```yaml
name: 'Terraform Plan/Apply'

on:
  push:
    branches:
      - main
    paths:
      - '**.tf'
      - '.terraform.lock.hcl'
  pull_request:
    branches:
      - main

permissions:
  id-token: write
  contents: read
  pull-requests: write

env:
  ARM_CLIENT_ID: ${{ secrets.AZURE_CLIENT_ID }}
  ARM_TENANT_ID: ${{ secrets.AZURE_TENANT_ID }}
  ARM_SUBSCRIPTION_ID: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
  ARM_USE_OIDC: true
  ARM_SKIP_PROVIDER_REGISTRATION: true
  TF_VAR_subscription_id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

defaults:
  run:
    working-directory: terraform/environments/dev

jobs:
  terraform-plan:
    name: 'Terraform Plan'
    runs-on: ubuntu-latest
    outputs:
      tfplanExitCode: ${{ steps.tf-plan.outputs.exitcode }}

    steps:
    - name: Checkout
      uses: actions/checkout@v4

    - name: Setup Terraform
      uses: hashicorp/setup-terraform@v3
      with:
        terraform_wrapper: false

    - name: Terraform Init
      run: terraform init

    - name: Terraform Format
      run: terraform fmt -check

    - name: Terraform Validate
      run: terraform validate

    - name: Run Checkov
      uses: bridgecrewio/checkov-action@v12
      with:
        directory: terraform/environments/dev
        framework: terraform
        soft_fail: true

    - name: Terraform Plan
      id: tf-plan
      run: |
        export exitcode=0
        terraform plan -detailed-exitcode -no-color -out tfplan || export exitcode=$?
        echo "exitcode=$exitcode" >> $GITHUB_OUTPUT
        if [ $exitcode -eq 1 ]; then
          echo Terraform Plan Failed!
          exit 1
        else
          exit 0
        fi

    - name: Publish Terraform Plan
      uses: actions/upload-artifact@v4
      with:
        name: tfplan
        path: terraform/environments/dev/tfplan

    - name: Create String Output
      id: tf-plan-string
      run: |
        TERRAFORM_PLAN=$(terraform show -no-color tfplan)
        delimiter="$(openssl rand -hex 8)"
        echo "summary<<${delimiter}" >> $GITHUB_OUTPUT
        echo "## Terraform Plan Output" >> $GITHUB_OUTPUT
        echo "<details><summary>Click to expand</summary>" >> $GITHUB_OUTPUT
        echo "" >> $GITHUB_OUTPUT
        echo '```terraform' >> $GITHUB_OUTPUT
        echo "$TERRAFORM_PLAN" >> $GITHUB_OUTPUT
        echo '```' >> $GITHUB_OUTPUT
        echo "</details>" >> $GITHUB_OUTPUT
        echo "${delimiter}" >> $GITHUB_OUTPUT

    - name: Publish Terraform Plan to Task Summary
      env:
        SUMMARY: ${{ steps.tf-plan-string.outputs.summary }}
      run: echo "$SUMMARY" >> $GITHUB_STEP_SUMMARY

    - name: Push Terraform Output to PR
      if: github.ref != 'refs/heads/main'
      uses: actions/github-script@v7
      env:
        SUMMARY: "${{ steps.tf-plan-string.outputs.summary }}"
      with:
        github-token: ${{ secrets.GITHUB_TOKEN }}
        script: |
          const body = `${process.env.SUMMARY}`;
          github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: body
          })

  terraform-apply:
    name: 'Terraform Apply'
    if: github.ref == 'refs/heads/main' && needs.terraform-plan.outputs.tfplanExitCode == '2'
    runs-on: ubuntu-latest
    needs: [terraform-plan]

    steps:
    - name: Checkout
      uses: actions/checkout@v4

    - name: Setup Terraform
      uses: hashicorp/setup-terraform@v3

    - name: Terraform Init
      run: terraform init

    - name: Download Terraform Plan
      uses: actions/download-artifact@v4
      with:
        name: tfplan
        path: terraform/environments/dev

    - name: Terraform Apply
      run: terraform apply -auto-approve tfplan
```

Worth calling out that `ARM_USE_OIDC: true` is doing the heavy lifting here. No client secret anywhere in the pipeline. Terraform picks up the OIDC token that GitHub Actions mints for the job and exchanges it directly with Azure AD. The first time this actually works it genuinely feels like something is broken.

## Cloud Custodian — Policy as Code

Once the infrastructure was up I wired in Cloud Custodian running on a GitHub Actions schedule. The idea with Custodian is less "here's a report" and more "here's something that will actually take action when it finds a problem." Policies are written in YAML and run against your subscription on whatever cadence you set.

The workflow is straightforward. Install Custodian, authenticate via OIDC the same as Terraform, run the policies, and upload the findings as an artifact:

```yaml
name: Cloud Custodian CSPM Scan

on:
  workflow_dispatch:

permissions:
  id-token: write
  contents: read

jobs:
  custodian-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.x'

      - name: Install Cloud Custodian
        run: pip install c7n c7n-azure

      - name: Azure Login
        uses: azure/login@v2
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

      - name: Run Custodian Scan
        run: bash scripts/run-custodian.sh

      - name: Upload findings
        uses: actions/upload-artifact@v4
        with:
          name: custodian-findings
          path: cspm/cloud-custodian/output/dry-run/
```

The policies cover things like NSGs with port 22 or 3389 open to the internet, storage accounts missing required tags, and overly permissive network access rules. Not every policy does the same thing when it finds a violation and that's intentional. The NSG policies ship findings to an Azure Storage Queue for visibility. The tag remediation policy actually goes and fixes the resource directly. That's also the one wired up to the Logic App, which I'll get into in the next section. The rest is notify-only for now, which is honestly how you'd want to roll this out in a real environment anyway. Notify first, understand what you're catching, then flip to remediate once you're confident in the policy.

Here's what the NSG policies look like:

```yaml
policies:
  - name: nsg-open-ssh
    description: NSGs with inbound port 22 open to the internet.
    resource: azure.networksecuritygroup
    filters:
      - type: ingress
        ports: '22'
        access: 'Allow'
        source: '*'
    actions:
      - type: notify
        transport:
          type: azure-storage-queue
          queue: https://storageaccount.queue.core.windows.net/custodian-findings

  - name: close-rdp-access
    description: NSGs with inbound port 3389 open to the internet.
    resource: azure.networksecuritygroup
    filters:
      - type: ingress
        ports: '3389'
        access: 'Allow'
        source: '*'
    actions:
      - type: notify
        transport:
          type: azure-storage-queue
          queue: https://storageaccount.queue.core.windows.net/custodian-findings
```

And the Key Vault compliance policy, which uses notify rather than auto-remediate:

```yaml
  - name: keyvault-missing-deletion-protection
    description: Find Key Vaults missing soft delete or purge protection
    resource: azure.keyvault
    filters:
      - or:
          - type: value
            key: properties.enableSoftDelete
            value: false
          - type: value
            key: properties.enablePurgeProtection
            value: false
    actions:
      - type: notify
        transport:
          type: azure-storage-queue
          queue: https://storageaccount123131.queue.core.windows.net/custodian-findings
```

Custodian does support full auto-remediation. For example you can mark unused VMs for deletion based on CPU metrics:

```yaml
policies:
  - name: delete-unused-vms
    resource: azure.vm
    filters:
      - type: metric
        metric: Percentage CPU
        op: le
        aggregation: average
        threshold: 1
        timeframe: 72
    actions:
      - type: mark-for-op
        op: delete
        days: 7
```

My reason for not leaning on auto-remediation heavily is that out of the box, Custodian gives you no visibility into what changes it made. You'd need to build alerting on top of it before you'd ever want it autonomously deleting or modifying resources in a real subscription. For a learning environment it makes more sense to keep most policies on notify and use something like a Logic App to handle the response side properly.

## Cloud Custodian — The Output

After each run Custodian creates an artifact in GitHub Actions containing a zip of all findings. Here's what that looks like:

<img width="1896" height="757" alt="image" src="https://github.com/user-attachments/assets/c9831377-0b9e-4423-91f5-558a7921a03c" />

And here's an example of what one of the result files actually contains:

<img width="1197" height="50" alt="image" src="https://github.com/user-attachments/assets/2dc8430c-48c0-4ab8-9446-625798d710f1" />

Each policy gets its own folder with a `resources.json` of what it matched and a log of the run. Not pretty, but it gets the job done and it's completely free. You could easily build a small HTML dashboard on top of the JSON to make it presentable — I've had Claude knock one of those out before and it's not a complicated ask.

That said, from a business perspective it's hard to imagine many companies running Custodian this way at scale. Azure has native policy tracking built in, and tools like Orca and Wiz are mainstream for a reason. They're just expensive and impossible to run at home, which is exactly why Custodian is a good fit for this kind of project.

## Logic App — Closing the Loop

The Logic App is what ties detection to remediation. When the tag compliance policy in Custodian finds a resource missing required tags, it writes a message to the Azure Storage Queue. The Logic App sits on the other end of that queue, reads the message, and takes action.

The flow works like this: Logic App triggers on a new message in the Storage Queue, parses the JSON payload that Custodian dropped there, calls the Azure Resource Manager API to apply the missing tags, and then sends a notification confirming what was changed and on which resource.

This was the piece of the project I was most interested in getting right because it's the full cycle. Custodian is the detection layer. The queue is the handoff point. The Logic App is the enforcement layer. Nothing is blocking the pipeline and nothing requires a human to be sitting there approving each change.

The reason I didn't wire up every policy this way is the same reason I kept most policies on notify only. Until you've run a policy for a while and you trust what it's catching, automated remediation on every finding is a bad idea. You can cause outages. Getting the tagging policy on the Logic App was enough to prove the pattern works. The rest can follow once there's confidence in what the policies are actually matching.

## Prowler — The Big Scan

Prowler is one of the most widely used open source cloud security scanners out there. I'd used it before in Pwned Labs pentesting courses so it felt like a natural fit to pull into this project. They have a web app version that gives you one free scan. You connect your Azure account, create a service principal with the right permissions, and let it run. I had dependency issues trying to run it locally so the web app was an easy call.

<img width="1915" height="859" alt="image" src="https://github.com/user-attachments/assets/19a7ac9f-b75a-4108-8daa-50ff4cebbb06" />

The overall score came back at **49.83% — Moderate Risk**. Exactly what I was hoping for. A 100% clean score would mean either the tool isn't working or the environment is too simple to be interesting.

The score breakdown by pillar tells the real story though.

<img width="1581" height="138" alt="image" src="https://github.com/user-attachments/assets/7c60b632-5a12-4234-a10d-b42a66d61ab6" />

IAM came back at 100% which is expected given the zero-trust identity setup with OIDC and Managed Identities. Attack Surface was 96.9%. Then Logging and Monitoring came in at 9.7% and Encryption sat at 50%.

The logging score is honestly the most realistic finding in the whole project. In almost every real Azure environment I've seen, logging and monitoring is the worst performing category. It's boring to configure, it costs money to retain, and it's easy to deprioritize until something actually goes wrong. Even in a lab environment built with security in mind it tanked the score, and that tells you something.

<img width="1891" height="922" alt="image" src="https://github.com/user-attachments/assets/d1f49210-a156-47cc-9e27-9a6f89511e7d" />

50 total findings across the subscription. The highlights were storage accounts not enforcing Customer Managed Keys, Network Watcher flow logs not being shipped to a Log Analytics workspace, and storage default network access rules being too permissive. All intentional, all sitting there waiting to be fixed.

## What I Took Away

Getting OIDC federation working between GitHub Actions and Azure is one of those things that feels almost wrong the first time. You keep expecting it to fail because you haven't put a secret anywhere. But once it works you genuinely can't go back to storing client secrets in GitHub.

Terraform remote state is another one. Storing state locally works right up until it doesn't. Getting the backend configured with a storage account and state locking early means you never have to deal with corrupted state or two people running applies at the same time.

Cloud Custodian was new territory for me and I like it a lot. It sits in a different space to Prowler. Prowler tells you what's wrong, Custodian can actually do something about it. The Logic App then turns that "doing something" into a proper auditable workflow rather than a blind automated action. The three together cover a lot of ground for a zero-cost home lab setup.

## Part 2

The findings are sitting there and they need fixing. In the next post I'll go through remediating the Prowler results one by one — flow logs, CMK enforcement, storage network rules — and show what the score looks like after. I'll also flesh out the Logic App remediation pipeline so more of the Custodian policies feed into it rather than just tagging.

Same repo, same environment, rebuilt clean. See you there.
