# 🔍 09. Terraform Data Sources: Querying Existing Cloud Infrastructure

> *"Don't hardcode IDs that change every month. Query your cloud dynamically with Data Sources!"* 📡☁️

In real-world DevOps, you rarely build infrastructure in a blank vacuum. You often need to deploy a server into a VPC created by the networking team, attach a role created by the security team, or look up the newest official Ubuntu AMI. **Data Sources** allow Terraform to read and consume pre-existing cloud infrastructure without managing or modifying it.

---

## 🎯 Learning Objectives

By the end of this topic, you will be able to:
1. 📖 **Distinguish Data Sources (`data`) from Resources (`resource`)** with complete clarity.
2. 🔎 **Perform read-only queries** against live cloud providers (VPCs, Subnets, AMIs, IAM Roles).
3. 🚫 **Eliminate brittle hardcoded IDs** from your codebase forever.
4. 🔗 **Integrate externally managed infrastructure** seamlessly into your Terraform stacks.

---

## 📊 Visual Explanation

![Terraform Data Sources vs Resources](../images/09-data-sources.png)

```mermaid
flowchart LR
    subgraph TerraformCode ["💻 Your Terraform Configuration"]
        DATA_BLOCK["📖 data 'aws_vpc' 'existing' {\n  default = true\n}"]
        RES_BLOCK["🏗️ resource 'aws_subnet' 'new_subnet' {\n  vpc_id = data.aws_vpc.existing.id\n  cidr_block = '172.31.96.0/20'\n}"]
    end

    subgraph AWSCloud ["☁️ AWS Cloud Environment"]
        EXISTING_VPC["🌐 Pre-existing Default VPC\n(Created outside of this project)"]
        NEW_SUBNET["✨ Brand New Subnet\n(Provisioned by Terraform)"]
    end

    DATA_BLOCK -->|1. Read-Only Query (GET API)| EXISTING_VPC
    EXISTING_VPC -.->|2. Returns vpc_id: vpc-12345| DATA_BLOCK
    DATA_BLOCK -->|3. Injects ID into| RES_BLOCK
    RES_BLOCK -->|4. Provisions via POST API| NEW_SUBNET

    style TerraformCode fill:#1e293b,stroke:#38bdf8,stroke-width:2px,color:#fff
    style AWSCloud fill:#1e293b,stroke:#ff9900,stroke-width:2px,color:#fff
```

---

## 👶 Explain Like I Am a Beginner

### 📚 The Library Book Analogy
Think of the difference between buying a notebook and reading an encyclopedia at a library:
- **`resource` (Buying and writing a notebook):** You own it, you write on its pages, and when you no longer need it, you throw it in the trash (`terraform destroy`).
- **`data` (Reading an encyclopedia at the library):** You walk up to the shelf, open the book, look up a piece of information (e.g. *What is the capital of France?*), write the answer down on your notepad, and put the encyclopedia back on the shelf. **You do NOT own the library book, and you cannot tear its pages or throw it away!**

In Terraform:
- A `resource` is created, updated, and destroyed by Terraform.
- A `data` source is **read-only**. Terraform inspects it, reads its attributes, and uses them in your configuration. Running `terraform destroy` will **never** delete a data source!

---

## ⚡ The Essential 80/20 Concepts

| Concept | Simple Explanation | Practical Importance |
|---|---|---|
| **Data Block (`data`)** | A read-only query fetching information from the cloud provider. | Allows you to reference existing infrastructure without importing it into state. |
| **Data Source Type** | Provider query object (e.g., `aws_vpc`, `aws_ami`, `aws_caller_identity`). | Determines what API search Terraform executes. |
| **Data Source Reference** | `data.<TYPE>.<NAME>.<ATTRIBUTE>` (Notice the `data.` prefix!). | Injects fetched information into downstream resource blocks. |
| **Search Filter** | Criteria used to find specific cloud objects (e.g. `most_recent = true`, tag filters). | Dynamically selects the newest operating system image or correct subnet. |
| **Destroy Immunity** | Data sources are completely immune to `terraform destroy`. | Eliminates fear of accidentally deleting shared corporate VPCs or DNS zones. |

---

## 🚫 The Anti-Pattern: Why Hardcoding AMIs is Terrible

Look at this common beginner snippet:

```hcl
# ❌ FRAGILE & BAD PRACTICE:
resource "aws_instance" "web" {
  ami           = "ami-0c55b159cbfafe1f0" # Hardcoded Ubuntu 22.04 AMI in us-east-1
  instance_type = "t3.micro"
}
```

### Why this fails in the real world:
1. **Region Brittle:** `ami-0c55b159cbfafe1f0` only exists in `us-east-1`! If someone deploys your code to `eu-west-1` (Europe), Terraform crashes because that AMI ID doesn't exist there.
2. **Security Vulnerability:** In 6 months, this AMI will be outdated, lacking security patches and kernel updates.

---

## ✅ The Pro Solution: Dynamic AMI Lookup with Data Sources

Let Terraform query AWS for the **latest official Ubuntu 22.04 AMI** automatically:

```hcl
# 1. Query AWS for the newest official Ubuntu AMI
data "aws_ami" "ubuntu" {
  most_recent = true
  owners      = ["099720109477"] # Canonical (Official Ubuntu Owner ID)

  filter {
    name   = "name"
    values = ["ubuntu/images/hvm-ssd/ubuntu-jammy-22.04-amd64-server-*"]
  }

  filter {
    name   = "virtualization-type"
    values = ["hvm"]
  }
}

# 2. Query the existing Default VPC
data "aws_vpc" "default" {
  default = true
}

# 3. Provision an EC2 instance dynamically in any region!
resource "aws_instance" "web" {
  ami           = data.aws_ami.ubuntu.id       # 👈 Dynamic AMI ID!
  instance_type = "t3.micro"

  tags = {
    Name = "Dynamic-Ubuntu-Server"
    VPC  = data.aws_vpc.default.id
  }
}
```

> 🎯 **Why this is awesome:** You can run this exact same code in New York (`us-east-1`), Frankfurt (`eu-central-1`), or Tokyo (`ap-northeast-1`). Terraform automatically finds the correct regional AMI ID every time!

---

## 🔍 Top 5 Most Useful AWS Data Sources in DevOps

1. **`aws_caller_identity`**: Get the current AWS Account ID and IAM ARN.
   ```hcl
   data "aws_caller_identity" "current" {}
   # Access: data.aws_caller_identity.current.account_id
   ```

2. **`aws_region`**: Get the currently configured AWS Region.
   ```hcl
   data "aws_region" "current" {}
   # Access: data.aws_region.current.name
   ```

3. **`aws_availability_zones`**: Fetch all available AZs in the current region.
   ```hcl
   data "aws_availability_zones" "available" {
     state = "available"
   }
   # Access: data.aws_availability_zones.available.names[0]
   ```

4. **`aws_subnets`**: Find IDs of existing subnets by VPC or tags.
   ```hcl
   data "aws_subnets" "public" {
     filter {
       name   = "vpc-id"
       values = [data.aws_vpc.default.id]
     }
   }
   ```

5. **`aws_secretsmanager_secret_version`**: Safely read passwords from AWS Secrets Manager.

---

## 🧪 Hands-on Practice: Local File Data Source Lab

### 🎯 Objective
Use a `data` source to read an existing local system file and pass its contents into a new resource.

### 📋 Prerequisites
Terraform CLI installed.

### 🚀 Steps to Follow:

1. Create a lab directory:
   ```bash
   mkdir data-source-lab
   cd data-source-lab
   ```

2. Create an external file manually (simulating pre-existing infrastructure):
   ```bash
   echo "Corporate-Security-Policy-v4" > policy.txt
   ```

3. Create `main.tf`:
   ```hcl
   terraform {
     required_providers {
       local = {
         source  = "hashicorp/local"
         version = "~> 2.5"
       }
     }
   }

   # Read the existing file (read-only query)
   data "local_file" "security_policy" {
     filename = "${path.module}/policy.txt"
   }

   # Create a new file incorporating the data from the existing file
   resource "local_file" "server_banner" {
     filename = "${path.module}/welcome-banner.txt"
     content  = <<-EOT
       === WELCOME TO SECURE SERVER ===
       Enforced Policy: ${data.local_file.security_policy.content}
       Audited On:      ${timestamp()}
     EOT
   }

   output "fetched_policy" {
     value = data.local_file.security_policy.content
   }
   ```

4. Run Terraform:
   ```bash
   terraform init
   terraform apply -auto-approve
   ```

### 🔎 Expected Result:
```text
Outputs:
fetched_policy = "Corporate-Security-Policy-v4"
```
Check `welcome-banner.txt`:
```bash
cat welcome-banner.txt
```

### 🧹 Cleanup Instructions:
```bash
terraform destroy -auto-approve
```
Notice: `welcome-banner.txt` is destroyed, but your original `policy.txt` remains completely untouched!

---

## ⚠️ Common Beginner Mistakes

### 1. Forgetting the `data.` Prefix When Referencing
- **What it looks like:** `vpc_id = aws_vpc.default.id` instead of `vpc_id = data.aws_vpc.default.id`.
- **Why it happens:** Habit from referencing `resource` blocks.
- **The Fix:** Whenever referencing a `data` block, you must prefix it with `data.`:
  `data.<TYPE>.<NAME>.<ATTRIBUTE>`.

### 2. Overly Broad Filters Returning Multiple Matches
- **What it looks like:** Error: `Your query returned more than one result. Please try a more specific search criteria.`
- **Why it happens:** Searching for an AMI or subnet with filters that match 10 different objects.
- **The Fix:** Add `most_recent = true` for AMIs, or use plural data sources like `data "aws_subnets"` if you intend to retrieve multiple items.

### 3. Querying Non-Existent Resources
- **What it looks like:** Error: `No matching subnet found.`
- **Why it happens:** Typo in tag names or searching in the wrong AWS region.
- **The Fix:** Double-check your AWS region and tag keys in the cloud console.

---

## 💼 Interview Revision Questions

### Q1: What is the key difference between a Resource block and a Data Source block?
**Answer:** A `resource` block instructs Terraform to manage the complete lifecycle (create, update, destroy) of an infrastructure component. A `data` source block is strictly read-only; it queries and fetches information about existing infrastructure without managing or modifying it.

### Q2: What happens to a Data Source when you run `terraform destroy`?
**Answer:** Nothing happens to the underlying cloud resource. Data sources are read-only queries, so `terraform destroy` simply removes the local reference from state; the actual cloud object is never deleted.

### Q3: Why is querying AMIs via data sources preferred over hardcoding AMI IDs?
**Answer:** Hardcoded AMI IDs are region-specific and become obsolete quickly as new security patches are released. A data source dynamically retrieves the newest official AMI in whatever region the template is executed, making configurations portable and secure.

---

## ⚡ Quick Revision & Cheat Sheet

- 📌 **Syntax:** `data "<TYPE>" "<NAME>" { ... }`
- 📌 **Reference:** `data.<TYPE>.<NAME>.<ATTRIBUTE>`
- 📌 **Lifecycle:** Read-only; completely safe from `terraform destroy`!
- 📌 **Dynamic AMIs:** Use `data "aws_ami"` with `most_recent = true`
- 📌 **Current Account Info:** `data "aws_caller_identity" "current" {}`

> [!TIP]
> **Remember This:** Use `resource` when you want Terraform to build and own it. Use `data` when someone else already built it and you just need to borrow its ID!
