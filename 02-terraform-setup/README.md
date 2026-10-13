# 🛠️ 02. Terraform Installation, Editor Setup & Safe Cloud Authentication

> *"Hey Ravi! 🌸 Today we build your developer cockpit. Before we fly a rocket to the cloud, we must make sure our flight controls are tuned and our keys are locked in a safe!"* — **Mentor Rithu 👩‍🏫**

---

## 👫 Mentor & Student Check-In

- **Ravi 👨‍🎓:** *"Rithu, I have Terraform installed, but I'm terrified of accidentally leaking my AWS Access Keys to GitHub and getting a \$5,000 bill overnight! How do senior DevOps engineers protect their secrets?"*
- **Rithu 👩‍🏫:** *"I'm so glad you asked, Ravi! That fear is very healthy—everyone who has worked in DevOps has seen someone leak a key. Today, I'm going to teach you the 'Pocket Passport' rule so your code stays 100% clean and your credentials stay 100% private. Let's get your setup perfect!"*

---

## 🎯 What Ravi Will Learn Today

1. 💻 **Install and verify the Terraform CLI** across Windows, macOS, or Linux.
2. 🔌 **Configure VS Code with the official HashiCorp extension** and automatic formatting on save.
3. 🔐 **Authenticate with AWS safely** using named profiles and environment variables (zero hardcoding).
4. 🛡️ **Verify your cloud connection** with a read-only test that costs exactly \$0.00!

---

## 📊 Visual Explanation

![Terraform Setup and Architecture](../images/02-terraform-setup.png)

```mermaid
flowchart TD
    subgraph RaviMachine ["💻 Ravi's Developer Workstation"]
        subgraph Editor ["📝 VS Code IDE"]
            HCL_EXT["HashiCorp Terraform Extension\n• Autocomplete\n• Syntax Highlight\n• Auto-fmt on save"]
        end

        subgraph Terminal ["⌨️ Shell / CLI Environment"]
            TF_CLI["Terraform CLI Binary\n(terraform --version)"]
            AWS_CLI["AWS CLI (aws configure)\n~/.aws/credentials\n~/.aws/config"]
            ENV_VARS["Environment Variables\nAWS_ACCESS_KEY_ID\nAWS_SECRET_ACCESS_KEY"]
        end
    end

    subgraph AuthLayer ["🔐 Rithu's Safe Auth Pipeline"]
        CHECK["Terraform scans credentials in priority:\n1. Env Vars\n2. AWS Profile\n3. IAM Instance Profile"]
    end

    subgraph AWSCloud ["☁️ AWS Cloud"]
        IAM["AWS IAM (Identity & Access Management)\nVerifies Signature & Grants Least Privilege"]
    end

    TF_CLI --> CHECK
    ENV_VARS --> CHECK
    AWS_CLI --> CHECK
    CHECK -->|SigV4 Signed API Call| IAM

    style RaviMachine fill:#1e293b,stroke:#38bdf8,stroke-width:2px,color:#fff
    style AuthLayer fill:#1e293b,stroke:#a855f7,stroke-width:2px,color:#fff
    style AWSCloud fill:#1e293b,stroke:#ff9900,stroke-width:2px,color:#fff
```

---

## 👶 Rithu Explains Like You Are a Beginner

### 🛂 The Airport Passport Analogy
> *"Ravi, imagine traveling internationally at the airport:"*

- **The Wrong Way (The Billboard) ❌:** You print your credit card number, CVV, and passport number in giant bold letters on the front of your t-shirt. Anyone walking past at the terminal can steal your identity.
  *(This is what happens when someone writes `access_key = "AKIA..."` inside their `.tf` files and pushes to GitHub!)*
- **The Right Way (The Pocket Passport) ✅:** You keep your passport zipped safely in your jacket pocket. When you step up to the airline counter, you show it privately to the agent. They stamp your ticket, and you slip your passport right back into your pocket.

> *"In Terraform, Ravi, **your `.tf` files are public blueprints.** Your credentials stay in your private jacket pocket (`~/.aws/credentials` or terminal environment variables). Terraform finds them quietly in the background!"*

---

## 💻 Step 1: Installing Terraform CLI (Pick Ravi's OS)

### 🪟 On Windows (PowerShell):
```powershell
# Using winget (Fastest & cleanest)
winget install Hashicorp.Terraform

# Or using Chocolatey
choco install terraform
```

### 🍎 On macOS (Homebrew):
```bash
brew tap hashicorp/tap
brew install hashicorp/tap/terraform
```

### 🐧 On Linux (Ubuntu / Debian):
```bash
sudo apt-get update && sudo apt-get install -y gnupg software-properties-common curl
curl -fsSL https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list
sudo apt-get update && sudo apt-get install terraform
```

### 🔍 Rithu's Verification Check:
Open a fresh terminal and run:
```bash
terraform --version
```
*Expected Output:*
```text
Terraform v1.8.x (or newer)
on windows_amd64 / darwin_arm64 / linux_amd64
```
*If you see this, Ravi, give yourself a high-five! 🎉*

---

## 🎨 Step 2: Rithu's Pro VS Code Setup

Ravi, let's configure VS Code so it formats your code automatically every time you hit `Ctrl+S` (or `Cmd+S`):

1. In VS Code, open Extensions (`Ctrl+Shift+X` or `Cmd+Shift+X`).
2. Search for: `HashiCorp Terraform` and click **Install**.
3. Enable **Format on Save**:
   - Open Settings (`Ctrl+,`).
   - Search for `Format on Save` and check the box.
   - Set Default Formatter to `HashiCorp Terraform`.

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

## 🔐 Step 3: Rithu's Safe AWS Authentication

### Option A: AWS CLI Named Profiles (Rithu's Recommended Way!)

1. Run the interactive configuration in your terminal:
   ```bash
   aws configure --profile terraform-ravi
   ```
2. Enter your IAM credentials when prompted:
   ```text
   AWS Access Key ID [None]: AKIAIOSFODNN7EXAMPLE
   AWS Secret Access Key [None]: wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
   Default region name [None]: us-east-1
   Default output format [None]: json
   ```
3. Set your active profile:
   ```bash
   # Windows PowerShell:
   $env:AWS_PROFILE="terraform-ravi"

   # Linux / macOS / Git Bash:
   export AWS_PROFILE="terraform-ravi"
   ```

### 🛑 What Ravi's `providers.tf` Looks Like:
Notice how clean this is, Ravi! **Not a single credential in sight:**

```hcl
# providers.tf
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
  # Terraform picks up your profile or environment variables automatically!
}
```

---

## 🧪 Hands-on Practice: Ravi's Zero-Cost Auth Verification

> *"Ravi, let's make sure your computer can communicate with your AWS account without provisioning any paid resources!"* — **Rithu**

### 🎯 Objective
Run a read-only query using `data "aws_caller_identity"` to verify credentials.

### 🚀 Steps for Ravi:

1. Create a verification folder:
   ```bash
   mkdir ravi-auth-check
   cd ravi-auth-check
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

   # Read-only query: checks who is currently authenticated
   data "aws_caller_identity" "current" {}

   output "my_aws_account_id" {
     description = "Ravi's AWS Account ID"
     value       = data.aws_caller_identity.current.account_id
   }

   output "my_iam_identity" {
     description = "Ravi's IAM User / Role ARN"
     value       = data.aws_caller_identity.current.arn
   }
   ```

3. Run Terraform:
   ```bash
   terraform init
   terraform apply -auto-approve
   ```

### 🔎 Expected Result:
```text
Apply complete! Resources: 0 added, 0 changed, 0 destroyed.

Outputs:
my_aws_account_id = "123456789012"
my_iam_identity = "arn:aws:iam::123456789012:user/ravi-devops"
```
🎉 If you see your Account ID printed, you are connected and 100% secure!

---

## ⚠️ Rithu's Top 3 Setup Traps

### 1. Hardcoding Keys Directly in `provider "aws"`
- **Rithu says:** *"If you ever see `access_key = "..."` inside your code, erase it immediately! It's a ticking time-bomb."*

### 2. Forgetting to Set the Region
- **Symptom:** `Error: Missing region in configuration`.
- **Rithu says:** *"AWS needs to know which geographic region you want to talk to. Always set `region = "us-east-1"` in your provider block or export `AWS_DEFAULT_REGION`."*

### 3. Committing `.terraform/` to Git
- **Symptom:** Your Git repository becomes hundreds of megabytes in size!
- **Rithu says:** *"The `.terraform` folder holds OS-specific compiled binaries. Always list `.terraform/` and `*.tfstate` in your `.gitignore`!"*

---

## 💬 Ravi's Doubt Corner

### Ravi: *"Rithu, what if I work on multiple client AWS accounts on the same laptop?"*
**Rithu:** *"Great question! That's why AWS Named Profiles are so powerful. You configure `aws configure --profile client-a` and `aws configure --profile client-b`. Then in your terminal, simply switch by setting `$env:AWS_PROFILE="client-a"` or `export AWS_PROFILE="client-b"` before running Terraform!"*

### Ravi: *"What does IAM Least Privilege mean?"*
**Rithu:** *"It means only giving your user the exact permissions needed. Don't use `AdministratorAccess` if your project only needs to touch S3 buckets! That way, even if a key is somehow compromised, the damage is strictly contained."*

---

## ⚡ Quick Revision with Rithu

- 📌 **Check Version:** `terraform --version`
- 📌 **Check Identity:** `aws sts get-caller-identity`
- 📌 **Safe Auth:** Store keys in `~/.aws/credentials` or `AWS_ACCESS_KEY_ID`
- 📌 **Format Automatically:** VS Code official HashiCorp extension
- 📌 **Clean Git:** Always ignore `.terraform/` and `*.tfstate`

> [!CAUTION]
> **Rithu's Rule #1:** Never commit `.tfvars` or `.tfstate` files containing credentials to GitHub. Protect your cloud like you protect your bank account! 🔐
