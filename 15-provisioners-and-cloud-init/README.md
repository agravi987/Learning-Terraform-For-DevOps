# ⚙️ 15. Provisioners vs. Cloud-Init: The Right Way to Bootstrap Servers

> *"HashiCorp documentation explicitly says: Provisioners are a last resort! In modern DevOps, use Cloud-Init, User Data, and Packer instead!"* 🚫✅

When you launch an EC2 instance, you almost always need software installed on it: an Nginx web server, a Docker daemon, or application code. Beginners frequently reach for Terraform `provisioner` blocks (`remote-exec`, `local-exec`) to run shell scripts. In this chapter, you will learn why provisioners are an anti-pattern and how modern DevOps engineers use **Cloud-Init (User Data)** and **Golden AMIs** instead.

---

## 🎯 Learning Objectives

By the end of this topic, you will be able to:
1. ⚠️ **Understand what provisioners are** (`local-exec`, `remote-exec`) and why HashiCorp marks them as a **last resort**.
2. 🦹 **Identify the 4 major pitfalls of provisioners** (broken idempotency, open SSH ports, tainted resources).
3. 🚀 **Bootstrap servers cleanly using `user_data` and Cloud-Init**.
4. 🧱 **Embrace Immutable Infrastructure** with Packer and AWS Systems Manager (SSM).

---

## 📊 Visual Explanation

![Provisioners vs Cloud-Init Architecture](../images/15-provisioners-and-cloud-init.svg)

```mermaid
flowchart TD
    subgraph AntiPattern ["⚠️ Fragile Anti-Pattern: Provisioners"]
        PROV_EC2["aws_instance\n(Port 22 SSH open to internet)"]
        REMOTE["remote-exec provisioner\n(Runs SSH script from laptop)"]
        FAIL["Network timeout or script glitch\n$\rightarrow$ Resource marked TAINTED!"]
        
        PROV_EC2 --> REMOTE --> FAIL
    end

    subgraph BestPractice ["✅ Modern Best Practice: Cloud-Init / User Data"]
        USER_DATA["user_data = <<-EOF\n#!/bin/bash\napt-get update & install nginx\nEOF"]
        CLOUD_EC2["aws_instance\n(Zero SSH ports exposed!)"]
        BOOT["Cloud-Init runs natively\nduring OS kernel boot"]

        USER_DATA --> CLOUD_EC2 --> BOOT
    end

    style AntiPattern fill:#1e293b,stroke:#f43f5e,stroke-width:2px,color:#fff
    style BestPractice fill:#1e293b,stroke:#10b981,stroke-width:2px,color:#fff
```

---

## 👶 Explain Like I Am a Beginner

### 🎂 The Birthday Cake Analogy
Imagine ordering a custom birthday cake from a bakery:
- **The Provisioner Approach (Messy ❌):** The baker delivers an undecorated plain sponge cake to your house. Then, you hire a separate decorator to drive over with frosting bags. If the decorator gets stuck in traffic or their car breaks down, your party starts with a ruined, half-frosted cake.
- **The Cloud-Init / Golden Image Approach (Clean ✅):** The bakery bakes the sponge, frosts the icing, and writes the message in their professional kitchen. When the cake arrives at your door, it is **100% complete, fully baked, and ready to serve immediately**!

In Cloud computing:
- **Provisioners:** Launching a bare VM and having Terraform SSH into it over the public internet to run `apt-get install`. If the SSH connection drops, Terraform panics!
- **Cloud-Init / User Data:** Handing the server its startup recipe as it boots up, or using **Packer** to pre-bake a complete AMI with all software already installed.

---

## ⚡ The Essential 80/20 Concepts

| Concept | Simple Explanation | Practical Importance |
|---|---|---|
| **`remote-exec`** | Instructs Terraform to SSH or WinRM into a running VM and execute commands. | Anti-pattern: Requires open SSH ports and fragile network tunnels. |
| **`local-exec`** | Executes a shell script locally on the machine running Terraform. | Useful primarily for local notifications or running Ansible playbooks. |
| **`user_data` (Cloud-Init)** | A boot script passed to the cloud provider executed by the OS during instance initialization. | Industry standard: Native, reliable, requires zero SSH keys or open inbound ports. |
| **Tainted Resource** | A resource whose provisioner failed, marked by Terraform for automatic destruction on next apply. | Can cause unexpected downtime when an innocent script hiccup marks a database for deletion! |
| **Golden AMI (Packer)** | A pre-baked machine image containing the OS and application pre-installed. | Instances boot in 15 seconds with zero bootstrap latency. |

---

## 🚫 Why Terraform Provisioners are a Last Resort

HashiCorp's official docs explicitly state:
> *"Provisioners should only be used as a last resort of behavior of last resort... HashiCorp cannot guarantee that provisioners will be available in future versions."*

### The 4 Major Flaws of Provisioners:
1. **Breaks Idempotency:** Terraform cannot model changes made inside the operating system. If you run `terraform plan`, it has no idea if your installed packages changed.
2. **Security Risk:** `remote-exec` requires port 22 (SSH) open to the internet and requires storing SSH private keys on developer laptops or CI/CD runners.
3. **Fragile Network Dependencies:** If a transient network glitch drops your SSH session for 1 second, Terraform marks the entire cloud resource as **Tainted** and attempts to destroy it on the next run!
4. **Slow Execution:** Terraform blocks execution while waiting for the SSH script to finish, slowing down pipelines.

---

## ✅ The Modern Way: Cloud-Init & User Data

Instead of provisioners, use the native `user_data` argument. AWS executes this script via `cloud-init` directly on the machine during boot:

```hcl
# 1. Store your shell script in a clean, separate file (e.g. scripts/install_nginx.sh)
# scripts/install_nginx.sh:
# #!/bin/bash
# dnf update -y
# dnf install -y nginx
# systemctl start nginx
# systemctl enable nginx
# echo "<h1>Hello from Terraform Cloud-Init! 🚀</h1>" > /usr/share/nginx/html/index.html

# 2. Reference the script cleanly using templatefile or file function
resource "aws_instance" "web" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t3.micro"

  # Clean Cloud-Init bootstrap!
  user_data = file("${path.module}/scripts/install_nginx.sh")

  # Optional: Replace instance automatically if user_data changes
  user_data_replace_on_change = true

  tags = {
    Name = "Nginx-Web-Server"
  }
}
```

---

## 🛠️ The Only Acceptable Use for `local-exec`

If there is one provisioner that remains occasionally useful in DevOps pipelines, it is `local-exec` for triggering local helper actions:

```hcl
resource "aws_instance" "web" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t3.micro"

  # Trigger a local notification or script on your workstation
  provisioner "local-exec" {
    command = "echo Server ${self.public_ip} deployed successfully! >> deployment-log.txt"
  }

  # Control failure behavior: continue even if the command errors
  provisioner "local-exec" {
    command    = "curl -X POST https://hooks.slack.com/... -d '{\"text\":\"Deployment complete!\"}'"
    on_failure = continue
  }
}
```

---

## 🧪 Hands-on Practice: Simulating Local-Exec & Script Generation

### 🎯 Objective
Use `local-exec` safely to run a local script, and explore the `templatefile` function for generating bootstrap scripts.

### 📋 Prerequisites
Terraform CLI installed.

### 🚀 Steps to Follow:

1. Create a lab directory:
   ```bash
   mkdir provisioner-lab
   cd provisioner-lab
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

   variable "environment" {
     type    = string
     default = "Production"
   }

   # Generates the bootstrap configuration file
   resource "local_file" "bootstrap_script" {
     filename = "${path.module}/bootstrap.sh"
     content  = <<-EOT
       #!/bin/bash
       echo "Initializing ${var.environment} Node..."
       echo "System Ready at $(date)"
     EOT

     # local-exec runs on the computer executing Terraform
     provisioner "local-exec" {
       command = "echo Bootstrap script generated for ${var.environment} >> audit.txt"
     }
   }
   ```

3. Initialize and apply:
   ```bash
   terraform init
   terraform apply -auto-approve
   ```

### 🔎 Expected Result:
Check `audit.txt` and `bootstrap.sh`:
```bash
cat audit.txt
cat bootstrap.sh
```
Notice that `local-exec` executed locally during the creation phase of the resource!

### 🧹 Cleanup Instructions:
```bash
terraform destroy -auto-approve
rm -f audit.txt bootstrap.sh
```

---

## ⚠️ Common Beginner Mistakes

### 1. Relying on `remote-exec` for Complex Software Installation
- **What it looks like:** Writing 50 lines of `apt-get` commands inside a `remote-exec` block.
- **Why it is bad:** If package 49 fails due to a network hiccup, the server is tainted and destroyed.
- **The Fix:** Use Cloud-Init (`user_data`) or pre-bake the machine with **Packer**.

### 2. Leaving Port 22 Open to the World for Provisioners
- **What it looks like:** Creating a security group with `0.0.0.0/0` on port 22 just so `remote-exec` can connect.
- **The Fix:** Never open port 22 to the public internet! Use **AWS Systems Manager (SSM) Session Manager** for secure shell access without open ports.

### 3. Forgetting `user_data_replace_on_change`
- **What it looks like:** Updating your `user_data` script in Terraform, running apply, and wondering why the server didn't run the new script!
- **The Reality:** By default, AWS EC2 **only runs `user_data` once on initial first boot**. It will not re-run on subsequent reboots.
- **The Fix:** Set `user_data_replace_on_change = true` on the `aws_instance` resource if you want Terraform to recreate the server when the bootstrap script changes.

---

## 💼 Interview Revision Questions

### Q1: Why does HashiCorp consider provisioners a "last resort"?
**Answer:** Provisioners break the declarative nature of Terraform by executing imperative shell commands outside of Terraform's state engine. They introduce fragile network dependencies, require opening sensitive inbound ports (like SSH 22 or WinRM 5985), and cannot guarantee idempotency or clean rollbacks upon failure.

### Q2: What happens when a provisioner fails during `terraform apply`?
**Answer:** By default (unless `on_failure = continue` is specified), Terraform halts execution and marks the resource as **"Tainted"** in the state file. On the next `terraform apply`, Terraform will automatically destroy and recreate the tainted resource from scratch.

### Q3: What is the recommended modern alternative to using `remote-exec` provisioners for server setup?
**Answer:** The recommended approaches are:
1. **Cloud-Init / User Data** scripts executed natively by the cloud provider during operating system boot.
2. **Immutable Machine Images (Golden AMIs)** pre-built using HashiCorp Packer with all dependencies pre-installed.
3. Post-boot configuration management using **Ansible** or cloud-native management agents like **AWS Systems Manager (SSM)**.

---

## ⚡ Quick Revision & Cheat Sheet

- 📌 **Provisioners:** `remote-exec` (SSH into VM), `local-exec` (Runs locally)
- 📌 **Why Avoid:** Non-declarative, fragile, opens port 22, taints resources on failure
- 📌 **Best Alternative:** `user_data = file("bootstrap.sh")`
- 📌 **Recreate on Script Update:** `user_data_replace_on_change = true`
- 📌 **Golden Standard:** Build AMIs with **Packer**, provision with **Terraform**

> [!WARNING]
> **Remember This:** If you catch yourself writing `provisioner "remote-exec"` in a code review, pause! 99% of the time, `user_data` or a pre-baked Packer image is the cleaner, safer solution!
