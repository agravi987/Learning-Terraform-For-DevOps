# ⚙️ 11. Terraform Meta-Arguments & Dependency Control

> *"Meta-arguments give you superpowers: scale resources dynamically, govern destruction rules, and take absolute control of dependency graphs!"* 🚀🛡️

By default, every `resource` block in Terraform defines a single cloud resource. But what if you want to spawn 5 servers? Or create a resource only if a feature flag is enabled? Or prevent anyone from ever accidentally deleting a production database? 

These superpowers are provided by **Meta-Arguments**: special built-in keywords available inside *every* resource and module block in Terraform.

---

## 🎯 Learning Objectives

By the end of this topic, you will be able to:
1. ⛓️ **Enforce explicit dependencies** using `depends_on` when implicit references don't cut it.
2. 🔢 **Use `count` for scaling and toggles** (`count = var.enabled ? 1 : 0`).
3. 🗺️ **Master `for_each`** and understand why it is far safer than `count` for managing lists.
4. 🛡️ **Harness `lifecycle` rules** (`prevent_destroy`, `create_before_destroy`, `ignore_changes`) to safeguard production infrastructure.

---

## 📊 Visual Explanation

![Terraform Meta-Arguments Architecture](../images/11-meta-arguments.svg)

```mermaid
flowchart TD
    subgraph MetaArguments ["⚙️ Core Meta-Arguments"]
        DEP["1. depends_on\nExplicit Execution Order"]
        COUNT["2. count\nNumerical Index-based Scaling"]
        FOR_EACH["3. for_each\nKey-based Set/Map Iteration"]
        LIFE["4. lifecycle\nSafety & Deployment Hooks"]
        PROV["5. provider\nTargeting Aliased Providers"]
    end

    subgraph LifecycleHooks ["🛡️ Lifecycle Directives"]
        CBD["create_before_destroy\nZero downtime replacement"]
        PD["prevent_destroy\nHard stop on accidental deletion"]
        IC["ignore_changes\nIgnore external drift on specified tags"]
    end

    LIFE --> CBD & PD & IC

    style MetaArguments fill:#1e293b,stroke:#38bdf8,stroke-width:2px,color:#fff
    style LifecycleHooks fill:#1e293b,stroke:#10b981,stroke-width:2px,color:#fff
```

---

## 👶 Explain Like I Am a Beginner

### 🚦 The Construction Foreman Analogy
Imagine you are the general contractor managing a construction site:
- **`depends_on` (Safety barrier):** You tell the electricians: *"Do NOT turn on the power to the building until the city safety inspector has signed the document."* Even though the wires don't physically touch the paperwork, you enforce an **explicit sequence**.
- **`count` (Duplicate copies):** You order 3 identical concrete pillars (`Pillar #0`, `Pillar #1`, `Pillar #2`).
- **`for_each` (Named assignments):** Instead of numbered pillars, you label them specifically: `"North-Pillar"`, `"South-Pillar"`, and `"East-Pillar"`. If the North pillar is removed, the other two don't get renamed!
- **`lifecycle` (Safety rules):**
  - `prevent_destroy`: *"Under no circumstances may anyone knock down the load-bearing foundation wall!"*
  - `create_before_destroy`: *"Build the new bridge first, route the traffic onto it, and only THEN demolish the old bridge!"*

---

## ⚡ The Essential 80/20 Concepts

| Meta-Argument | Syntax Example | When to Use in DevOps |
|---|---|---|
| **`depends_on`** | `depends_on = [aws_iam_role_policy.x]` | Hidden dependencies where Terraform cannot infer relationship automatically (e.g. IAM policies). |
| **`count`** | `count = 3` or `count = var.enable ? 1 : 0` | Simple scaling or conditional resource feature toggles. |
| **`for_each`** | `for_each = toset(["web", "api"])` | Scaling resources using unique keys. Much safer than `count`! |
| **`lifecycle`** | `lifecycle { prevent_destroy = true }` | Production safeguards (prevent database drops, enable zero-downtime swaps). |
| **`provider`** | `provider = aws.dr_region` | Routing a specific resource to a non-default provider alias. |

---

## ⚔️ The Fatal Flaw of `count` vs. The Glory of `for_each`

This is one of the most critical lessons in all of Terraform!

### The Trap with `count`:
Suppose you create 3 subnets using a list:
```hcl
variable "subnets" {
  default = ["subnet-alpha", "subnet-beta", "subnet-gamma"]
}

resource "aws_subnet" "example" {
  count      = length(var.subnets)
  cidr_block = "10.0.${count.index}.0/24"
}
```
Terraform assigns addresses by index:
- `aws_subnet.example[0]` $\rightarrow$ `subnet-alpha`
- `aws_subnet.example[1]` $\rightarrow$ `subnet-beta`
- `aws_subnet.example[2]` $\rightarrow$ `subnet-gamma`

💥 **The Disaster:** Tomorrow, you delete `"subnet-alpha"` from the list!
Now:
- Index `0` becomes `subnet-beta` (Terraform **destroys and recreates** it!).
- Index `1` becomes `subnet-gamma` (Terraform **destroys and recreates** it!).
- Index `2` is deleted!
Deleting ONE item caused Terraform to destroy and replace all other subnets!

### The Solution: Use `for_each`!
```hcl
variable "subnets" {
  type    = set(string)
  default = ["subnet-alpha", "subnet-beta", "subnet-gamma"]
}

resource "aws_subnet" "example" {
  for_each   = var.subnets
  cidr_block = "10.0.1.0/24" # or lookup from map
  tags = {
    Name = each.key  # 👈 Directly accesses each item!
  }
}
```
Now Terraform addresses them by their unique key name:
- `aws_subnet.example["subnet-alpha"]`
- `aws_subnet.example["subnet-beta"]`
- `aws_subnet.example["subnet-gamma"]`

If you delete `"subnet-alpha"`, Terraform **only destroys `subnet-alpha`**! The others remain completely untouched.

---

## 🛡️ Mastering the `lifecycle` Block

### 1. `create_before_destroy = true` (Zero-Downtime)
By default, when replacing a resource, Terraform **destroys the existing one first**, then builds the new one. This causes downtime!
Setting `create_before_destroy = true` reverses this order:
```hcl
resource "aws_instance" "web" {
  ami           = "ami-new"
  instance_type = "t3.micro"

  lifecycle {
    create_before_destroy = true
  }
}
```

### 2. `prevent_destroy = true` (Production Armor)
Prevents catastrophic accidents on critical assets like production databases:
```hcl
resource "aws_db_instance" "production_database" {
  allocated_storage = 100
  engine            = "postgres"

  lifecycle {
    prevent_destroy = true  # 👈 terraform destroy will exit with an ERROR!
  }
}
```

### 3. `ignore_changes` (Stop Unwanted Drift)
If external tools (like Kubernetes autoscalers or security tagging bots) modify attributes in the cloud, tell Terraform to ignore those fields:
```hcl
resource "aws_instance" "app" {
  ami           = "ami-123"
  instance_type = "t3.micro"

  lifecycle {
    ignore_changes = [
      tags["LastScannedByGuardDuty"],
      user_data
    ]
  }
}
```

---

## 🔗 When to Use `depends_on`

Terraform handles 95% of dependencies implicitly. But occasionally, a cloud resource depends on another resource via a **background permission** that isn't referenced directly in arguments.

The classic AWS example: An EC2 instance profile needs an IAM Policy attached before the instance can download files from S3:

```hcl
resource "aws_iam_role_policy" "s3_access" {
  name   = "s3-read-policy"
  role   = aws_iam_role.app_role.id
  policy = data.aws_iam_policy_document.s3.json
}

resource "aws_instance" "app" {
  ami                  = "ami-123"
  iam_instance_profile = aws_iam_instance_profile.app_profile.name

  # Explicit dependency!
  depends_on = [
    aws_iam_role_policy.s3_access
  ]
}
```

---

## 🧪 Hands-on Practice: `for_each` and `lifecycle` Lab

### 🎯 Objective
Use `for_each` to create multiple named local files and protect one with `prevent_destroy`.

### 📋 Prerequisites
Terraform CLI installed.

### 🚀 Steps to Follow:

1. Create a lab directory:
   ```bash
   mkdir meta-arguments-lab
   cd meta-arguments-lab
   ```

2. Create `main.tf`:
   ```hcl
   terraform {
     required_providers {
       local = {
         source  = "hashicorp/local"
         version = "~> 2.5"
       }
     }
   }

   # Define multiple environments using a map
   locals {
     environments = {
       dev     = "DEBUG_MODE=true\nLOG_LEVEL=verbose"
       staging = "DEBUG_MODE=false\nLOG_LEVEL=info"
       prod    = "DEBUG_MODE=false\nLOG_LEVEL=error"
     }
   }

   # Iterate with for_each
   resource "local_file" "configs" {
     for_each = locals.environments

     filename = "${path.module}/${each.key}.env"
     content  = each.value
   }

   output "generated_files" {
     value = [for f in local_file.configs : f.filename]
   }
   ```

3. Initialize and apply:
   ```bash
   terraform init
   terraform apply -auto-approve
   ```

4. Inspect the generated files:
   ```bash
   ls *.env
   cat dev.env
   cat prod.env
   ```

5. **Test `prevent_destroy`:**
   Add a lifecycle block inside `resource "local_file" "configs"`:
   ```hcl
   lifecycle {
     prevent_destroy = true
   }
   ```
   Now attempt to run:
   ```bash
   terraform destroy
   ```
   **Output:**
   ```text
   Error: Instance cannot be destroyed
   ... Resource local_file.configs["dev"] has lifecycle.prevent_destroy set, but the plan calls for it to be destroyed.
   ```
   Terraform blocked the destruction! 🛡️

### 🧹 Cleanup Instructions:
Remove `prevent_destroy = true` from `main.tf`, then run:
```bash
terraform destroy -auto-approve
```

---

## ⚠️ Common Beginner Mistakes

### 1. Using `count` and `for_each` on the Same Block
- **What it looks like:** Declaring both `count = 3` and `for_each = var.servers` inside one resource.
- **The Reality:** Terraform will throw a syntax error. A resource can use `count` OR `for_each`, but never both.

### 2. Overusing `depends_on`
- **What it looks like:** Adding `depends_on = [aws_vpc.main]` on a subnet that already has `vpc_id = aws_vpc.main.id`.
- **Why it is bad:** It's redundant. Terraform's graph engine already knows the subnet depends on the VPC. Overusing `depends_on` cripples Terraform's ability to run operations in parallel.

### 3. Forgetting `toset()` with `for_each` on Lists
- **What it looks like:** `for_each = ["us-east-1", "us-west-2"]`
- **Why it errors:** `for_each` requires a `map` or a `set(string)`.
- **The Fix:** Wrap the list in `toset()`: `for_each = toset(["us-east-1", "us-west-2"])`.

---

## 💼 Interview Revision Questions

### Q1: Why is `for_each` generally preferred over `count` when scaling resources?
**Answer:** `count` identifies resources by integer indices (e.g., `[0]`, `[1]`, `[2]`). If an item in the middle or beginning of the list is removed, all subsequent resource indices shift, causing Terraform to recreate untouched resources. `for_each` tracks resources by immutable unique keys (e.g., `["web"]`, `["api"]`), allowing individual resources to be added or removed without affecting siblings.

### Q2: What does `lifecycle { create_before_destroy = true }` do?
**Answer:** It alters Terraform's default replacement behavior. Normally, when replacing a resource, Terraform destroys the old one first, causing downtime. With `create_before_destroy = true`, Terraform provisions the replacement resource first, verifies its health, and only then terminates the original resource.

### Q3: When is `depends_on` strictly necessary?
**Answer:** When there is an invisible dependency that cannot be inferred from resource argument references—such as an IAM policy granting permissions that a resource needs at creation time, or an S3 bucket needing an access policy applied before an external service can write logs to it.

---

## ⚡ Quick Revision & Cheat Sheet

- 📌 **Iteration by Index:** `count = 3` $\rightarrow$ `count.index`
- 📌 **Iteration by Key:** `for_each = toset([...])` $\rightarrow$ `each.key`, `each.value`
- 📌 **Zero-Downtime Swaps:** `lifecycle { create_before_destroy = true }`
- 📌 **Production Armor:** `lifecycle { prevent_destroy = true }`
- 📌 **Explicit Dependencies:** `depends_on = [aws_resource.name]`

> [!CAUTION]
> **Remember This:** Always favor `for_each` over `count` when iterating over collections of items that might change in the future!
