# 🛠️ 02. Terraform Installation, Editor Setup & Safe Cloud Authentication

> *"Set up your developer cockpit properly once, and never leak a cloud credential again!"* 🔐⚡

Before we write infrastructure code, we need our developer workbench set up properly. In this chapter, you will install the **Terraform CLI**, configure **VS Code** with syntax highlighting and autocompletion, and learn the enterprise-grade way to authenticate with **AWS** safely.

---

## 🎯 Learning Objectives

By the end of this topic, you will be able to:
1. 💻 **Install and verify the Terraform CLI** across Windows, macOS, or Linux.
2. 🔌 **Supercharge VS Code** with official HashiCorp extensions, linting, and auto-formatting.
3. 🛡️ **Configure safe cloud authentication** using the AWS CLI and Environment Variables without hardcoding sensitive secrets.
4. 🚫 **Avoid the #1 security disaster** that causes stolen cloud keys and massive unexpected bills.

---

## 📊 Visual Explanation

![Terraform Setup and Architecture](../images/02-terraform-setup.png)

```mermaid
flowchart TD
    subgraph LocalMachine ["💻 Developer Workstation"]
        subgraph Editor ["📝 VS Code IDE"]
            HCL_EXT["HashiCorp Terraform Extension\n• Autocomplete\n• Syntax Highlight\n• Auto-fmt on save"]
        end

        subgraph Terminal ["⌨️ Shell / CLI Environment"]
            TF_CLI["Terraform CLI Binary\n(terraform --version)"]
            AWS_CLI["AWS CLI (aws configure)\n~/.aws/credentials\n~/.aws/config"]
            ENV_VARS["Environment Variables\nAWS_ACCESS_KEY_ID\nAWS_SECRET_ACCESS_KEY"]
        end
    end

    subgraph AuthLayer ["🔐 Secure Authentication Pipeline"]
        CHECK["Provider scans credentials in priority:\n1. Env Vars\n2. AWS Profile\n3. IAM Instance Profile"]
    end

    subgraph AWSCloud ["☁️ AWS Cloud"]
        IAM["AWS IAM (Identity & Access Management)\nVerifies Signature & Grants Least Privilege"]
    end

    TF_CLI --> CHECK
    ENV_VARS --> CHECK
    AWS_CLI --> CHECK
    CHECK -->|SigV4 Signed API Call| IAM

    style LocalMachine fill:#1e293b,stroke:#38bdf8,stroke-width:2px,color:#fff
    style AuthLayer fill:#1e293b,stroke:#a855f7,stroke-width:2px,color:#fff
    style AWSCloud fill:#1e293b,stroke:#ff9900,stroke-width:2px,color:#fff
```

---

## 👶 Explain Like I Am a Beginner

### 🛂 The Airport Passport Analogy
When you travel internationally, you don't tattoo your credit card and national ID number onto your t-shirt for everyone to see. Instead, you keep your passport safely in your pocket, and when you reach the gate, you present it to security to stamp your temporary boarding pass.

In Terraform:
- **Tattooing your ID (Wrong way ❌):** Writing `access_key = "AKIA..."` inside your `.tf` files. The moment you push your code to GitHub, bots scrape your keys in under 60 seconds and spin up crypto-mining rigs on your account!
- **Keeping your passport in your pocket (Right way ✅):** Storing your credentials inside secure local configuration files (`~/.aws/credentials`) or temporary environment variables. Terraform automatically finds your credentials behind the scenes.

---

## ⚡ The Essential 80/20 Concepts

| Concept | Simple Explanation | Practical Importance |
|---|---|---|
| **Terraform Binary** | A single standalone executable file written in Go with zero external dependencies. | No bulky runtime (like Java JVM or Python virtualenv) required; runs anywhere. |
| **Language Server Protocol (LSP)** | Powers editor intelligence (autocomplete, jump to definition, documentation tooltips). | Saves hours looking up resource arguments in browser tabs. |
| **Credential Chain** | The ordered sequence in which Terraform's AWS Provider looks for valid authentication. | Allows the same code to run on your laptop, inside a GitHub Actions runner, or on an EC2 instance without modifying the code. |
| **IAM Least Privilege** | Giving your Terraform user only the exact cloud permissions it needs to build resources. | If credentials are ever compromised, the attacker cannot delete your entire cloud account. |

---

## 💻 Step-by-Step Installation

### 1. Installing Terraform CLI

#### 🪟 Windows (via winget or Chocolatey):
```powershell
# Using Windows Package Manager (winget) - Recommended
winget install Hashicorp.Terraform

# OR using Chocolatey:
choco install terraform
```

#### 🍎 macOS (via Homebrew):
```bash
brew tap hashicorp/tap
brew install hashicorp/tap/terraform
```

#### 🐧 Linux (Ubuntu / Debian):
```bash
sudo apt-get update && sudo apt-get install -y gnupg software-properties-common curl
curl -fsSL https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list
sudo apt-get update && sudo apt-get install terraform
```

### 2. Verify Your Installation
Open a fresh terminal window and run:
```bash
terraform --version
```
**Expected Output:**
```text
Terraform v1.8.x (or newer)
on windows_amd64 / darwin_arm64 / linux_amd64
```

---

## 🎨 Supercharging VS Code for Terraform

To make writing HCL effortless:

1. Open **VS Code**.
2. Press `Ctrl+P` (or `Cmd+P` on Mac) and paste:
   ```text
   ext install HashiCorp.terraform
   ```
3. Enable **Format on Save** so your code is automatically aligned when you save:
   - Open Settings (`Ctrl+,` or `Cmd+,`).
   - Search for `editor.formatOnSave` and check the box.
   - Set the default formatter to `HashiCorp Terraform`.

```json
// In settings.json:
{
  "[terraform]": {
    "editor.defaultFormatter": "hashicorp.terraform",
    "editor.formatOnSave": true
  }
}
```

---

## 🔐 Safe AWS Authentication (Zero Hardcoding)

### Method 1: AWS CLI Named Profiles (Best for Local Development)

1. Install the [AWS CLI](https://aws.amazon.com/cli/).
2. Run interactive setup:
   ```bash
   aws configure --profile terraform-dev
   ```
3. Enter your IAM credentials:
   ```text
   AWS Access Key ID [None]: AKIAIOSFODNN7EXAMPLE
   AWS Secret Access Key [None]: wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
   Default region name [None]: us-east-1
   Default output format [None]: json
   ```
4. Tell your terminal to use this profile:
   ```bash
   # Windows PowerShell:
   $env:AWS_PROFILE="terraform-dev"

   # Linux / macOS / Git Bash:
   export AWS_PROFILE="terraform-dev"
   ```

### Method 2: Environment Variables (Best for CI/CD Pipelines)

```bash
# Windows PowerShell:
$env:AWS_ACCESS_KEY_ID="AKIAIOSFODNN7EXAMPLE"
$env:AWS_SECRET_ACCESS_KEY="wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY"
$env:AWS_DEFAULT_REGION="us-east-1"

# Linux / macOS / Git Bash:
export AWS_ACCESS_KEY_ID="AKIAIOSFODNN7EXAMPLE"
export AWS_SECRET_ACCESS_KEY="wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY"
export AWS_DEFAULT_REGION="us-east-1"
```

### 🛑 What Your Provider Block Should Look Like:

Notice that your provider block is completely clean and **contains zero secrets**:

```hcl
# providers.tf - Clean, Safe, and Portable!
terraform {
  required_version = ">= 1.8.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = "us-east-1"
  # Notice: NO access_key or secret_key here!
  # Terraform automatically uses your AWS CLI profile or environment variables.
}
```

---

## 🧪 Hands-on Practice: Verifying Cloud Auth

### 🎯 Objective
Verify that your Terraform environment can securely query your AWS account without hardcoding any keys.

### 📋 Prerequisites
1. Terraform CLI installed.
2. AWS CLI configured with active IAM permissions.

### 🚀 Steps to Follow:

1. Create a verification directory:
   ```bash
   mkdir terraform-auth-check
   cd terraform-auth-check
   ```

2. Create `main.tf`:
   ```hcl
   terraform {
     required_providers {
       aws = {
         source  = "hashicorp/aws"
         version = "~> 5.0"
       }
     }
   }

   provider "aws" {
     region = "us-east-1"
   }

   # Read-only query: returns information about the authenticated caller
   data "aws_caller_identity" "current" {}

   output "account_id" {
     description = "Your authenticated AWS Account ID"
     value       = data.aws_caller_identity.current.account_id
   }

   output "caller_arn" {
     description = "Your IAM User or Role ARN"
     value       = data.aws_caller_identity.current.arn
   }
   ```

3. Run initialization and plan:
   ```bash
   terraform init
   terraform apply -auto-approve
   ```

### 🔎 Expected Result:
```text
Apply complete! Resources: 0 added, 0 changed, 0 destroyed.

Outputs:
account_id = "123456789012"
caller_arn = "arn:aws:iam::123456789012:user/terraform-learner"
```
🎉 If you see your Account ID printed, your developer workstation is securely connected to AWS!

### 🧹 Cleanup Instructions:
No resources were created (only read-only data queries), but running `terraform destroy` will keep your workspace clean:
```bash
terraform destroy -auto-approve
```

---

## ⚠️ Common Beginner Mistakes

### 1. Hardcoding Credentials inside `main.tf`
- **What it looks like:**
  ```hcl
  # ❌ NEVER DO THIS!
  provider "aws" {
    access_key = "AKIA..."
    secret_key = "secret..."
  }
  ```
- **Why it happens:** Beginners think this is the simplest way to authenticate.
- **The Fix:** Delete those two lines immediately. Use `aws configure` or environment variables instead.

### 2. Forgetting to Set the AWS Region
- **What it looks like:** Error: `No valid credential sources found for AWS Provider. Please see https://registry.terraform.io/providers/hashicorp/aws/latest/docs#authentication-and-configuration-options` or `Missing region in configuration`.
- **Why it happens:** AWS requires a target region (e.g., `us-east-1`, `eu-west-1`, `ap-south-1`) for almost every API call.
- **The Fix:** Explicitly set `region = "us-east-1"` in your `provider "aws"` block or set `export AWS_DEFAULT_REGION="us-east-1"`.

### 3. Multiple Conflicting AWS Profiles
- **What it looks like:** Terraform creates resources in your personal AWS account instead of your work/sandbox account.
- **Why it happens:** Unset or mismatched `AWS_PROFILE` variables in your terminal.
- **The Fix:** Run `aws sts get-caller-identity` in your terminal *before* running Terraform to verify which account you are logged into!

---

## 💼 Interview Revision Questions

### Q1: What is the credential search order for the Terraform AWS Provider?
**Answer:** The AWS provider scans for credentials in this priority order:
1. Parameters directly in the provider block (not recommended).
2. Environment variables (`AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_SESSION_TOKEN`).
3. Shared credentials and config files (`~/.aws/credentials`, `~/.aws/config` or specified by `AWS_PROFILE`).
4. Container credentials (Amazon ECS task roles).
5. Instance metadata service (Amazon EC2 IAM Instance Profile).

### Q2: Why should you avoid committing `.terraform/` and `.tfstate` files to version control?
**Answer:** The `.terraform/` directory contains downloaded provider binaries (hundreds of megabytes), and `.tfstate` files often contain sensitive plain-text values (passwords, certificates, private keys). Both should always be listed in `.gitignore`.

### Q3: How do automated CI/CD pipelines (e.g. GitHub Actions) authenticate to AWS safely without static keys?
**Answer:** Modern CI/CD pipelines use **OIDC (OpenID Connect)**. The CI/CD runner requests a temporary, short-lived token from AWS STS via an IAM Role trust policy, completely eliminating the need for long-lived, leakable access keys.

---

## ⚡ Quick Revision & Cheat Sheet

- 📌 **Verify Install:** `terraform --version`
- 📌 **Format Code:** `terraform fmt`
- 📌 **Check Who You Are:** `aws sts get-caller-identity`
- 📌 **Safe Auth:** Store credentials in `~/.aws/credentials` or `AWS_ACCESS_KEY_ID`.
- 📌 **Git Hygiene:** Always include `.terraform/`, `*.tfstate`, and `*.tfvars` in your `.gitignore` file.

> [!CAUTION]
> **Remember This:** If an AWS Access Key is committed to a public GitHub repository, automated bots find it in seconds and can rack up thousands of dollars in unauthorized infrastructure within minutes. Never hardcode credentials in `.tf` files!
