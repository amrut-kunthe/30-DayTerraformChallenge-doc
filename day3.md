# Day 3 — Deploying Your First Server with Terraform: A Beginner's Guide

> **Day 3 — 30-Day Terraform Challenge**
> **Communities:** AWS AI/ML UserGroup Kenya · Meru HashiCorp User Group 
> **Date:** March 19, 2026
> **Machine:** MacBook (macOS · Apple Silicon)

---

## Table of Contents

- [Overview](#overview)
- [Project Structure](#project-structure)
- [The Complete Terraform Code](#the-complete-terraform-code)
- [Breaking Down Each Block](#breaking-down-each-block)
  - [Block 1 — The Provider](#block-1--the-provider)
  - [Block 2 — The Security Group](#block-2--the-security-group)
  - [Block 3 — The EC2 Instance](#block-3--the-ec2-instance)
- [The Three Commands That Make It All Work](#the-three-commands-that-make-it-all-work)
  - [terraform init](#terraform-init)
  - [terraform plan](#terraform-plan)
  - [terraform apply](#terraform-apply)
- [Verifying the Deployment](#verifying-the-deployment)
- [Key Learnings from Day 3](#key-learnings-from-day-3)
- [Cleaning Up](#cleaning-up)

---

## Overview

Days 1 and 2 were about understanding concepts and setting up the environment. Day 3 is where Terraform stops being theoretical and becomes real. Today I wrote my first complete Terraform configuration, deployed an actual EC2 instance on AWS running a live web server, and curled it from my terminal to see "Hello, World" come back.

The goal was simple: provision an EC2 instance on AWS that serves a basic web page on port 8080 — all from code, without touching the AWS console once the infrastructure is defined.

Here is what I built:

- An **AWS Security Group** that opens port 8080 to incoming traffic
- An **EC2 instance** (t2.micro — free tier) running a minimal web server via a bash startup script
- Everything wired together using **Terraform resource references**

---

## Project Structure

Before writing any code, I created a dedicated folder for the project:

```bash
mkdir 30DayTerraformChallenge
cd 30DayTerraformChallenge
```

Inside this folder I created a single file called `main.tf`. This is the standard entry point for a Terraform configuration. Terraform reads all `.tf` files in the working directory, so you can split your configuration across multiple files as projects grow — but for this exercise, everything lives in one file.

```
30DayTerraformChallenge/
└── main.tf
```

---

## The Complete Terraform Code

Here is the full `main.tf` I wrote for Day 3:

```hcl
# Configure the AWS provider
provider "aws" {
  region = "us-east-1"
}

# Create a Security Group
resource "aws_security_group" "firewall" {
  name = "instanceFirewall"

  ingress {
    from_port   = 8080
    to_port     = 8080
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

# Create an EC2 instance
resource "aws_instance" "myFirstInstance" {
  ami                    = "ami-0ec10929233384c7f"
  instance_type          = "t2.micro"
  vpc_security_group_ids = [aws_security_group.firewall.id]

  user_data = <<-EOF
              #!/bin/bash
              echo "Hello, World" > index.html
              nohup busybox httpd -f -p 8080 &
              EOF

  user_data_replace_on_change = true

  tags = {
    Name = "myFirstInstance"
  }
}
```

---

## Breaking Down Each Block

### Block 1 — The Provider

```hcl
provider "aws" {
  region = "us-east-1"
}
```

The provider block tells Terraform which cloud platform to work with and how to connect to it. In this case:

- `provider "aws"` — tells Terraform to use the AWS provider
- `region = "us-east-1"` — all resources will be created in the N. Virginia region

Terraform does not connect to AWS directly on its own. It uses provider plugins — small pieces of code that know how to translate your HCL into AWS API calls. The provider block is what initialises that connection. Without it, Terraform has no idea where to create anything.

**Why us-east-1?** It is the most feature-complete AWS region. For a learning project, it is the safest choice — every service we need is available here, and the free tier applies.

---

### Block 2 — The Security Group

```hcl
resource "aws_security_group" "firewall" {
  name = "instanceFirewall"

  ingress {
    from_port   = 8080
    to_port     = 8080
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
}
```

By default, AWS blocks all incoming and outgoing traffic to EC2 instances. A security group is AWS's way of defining firewall rules — what traffic is allowed in and out.

Breaking this down:

- `resource "aws_security_group" "firewall"` — `aws_security_group` is the resource type (defined by the AWS provider), and `firewall` is my local name for it within this Terraform configuration
- `name = "instanceFirewall"` — the name that appears in the AWS console
- `ingress` block — defines an inbound traffic rule:
  - `from_port = 8080` and `to_port = 8080` — only allow traffic on port 8080
  - `protocol = "tcp"` — TCP traffic only
  - `cidr_blocks = ["0.0.0.0/0"]` — allow traffic from any IP address

**Why port 8080 and not port 80?** Ports below 1024 require root-level operating system privileges to bind to. Running a web server as root is a security risk. Port 8080 is a common alternative that avoids this problem while still demonstrating the concept.

**Note on egress:** I did not define an egress (outgoing) rule. By default, AWS allows all outbound traffic from an EC2 instance, so no explicit rule is needed for this example.

---

### Block 3 — The EC2 Instance

```hcl
resource "aws_instance" "myFirstInstance" {
  ami                    = "ami-0ec10929233384c7f"
  instance_type          = "t2.micro"
  vpc_security_group_ids = [aws_security_group.firewall.id]

  user_data = <<-EOF
              #!/bin/bash
              echo "Hello, World" > index.html
              nohup busybox httpd -f -p 8080 &
              EOF

  user_data_replace_on_change = true

  tags = {
    Name = "myFirstInstance"
  }
}
```

This is the heart of the configuration — the actual server. Let me go through each argument:

**`ami`**
The Amazon Machine Image ID specifies which operating system and base configuration to use for the instance. Think of it as the blueprint for the virtual machine. The AMI `ami-0ec10929233384c7f` is an Ubuntu Server 24.04 LTS image available in us-east-1. AMI IDs are region-specific — the same image has a different ID in every region.

**`instance_type`**
Defines the hardware specifications of the server. `t2.micro` gives us 1 vCPU and 1 GB of memory. It is part of the AWS free tier, so it costs nothing to run during the challenge.

**`vpc_security_group_ids`**
This is where things get interesting. Instead of hardcoding the security group ID (which I would not know until after it was created), I used a **resource reference**:

```hcl
vpc_security_group_ids = [aws_security_group.firewall.id]
```

The syntax is `<resource_type>.<resource_name>.<attribute>`. Terraform reads this, understands that `myFirstInstance` depends on `firewall`, and automatically creates the security group first before attempting to create the EC2 instance. This dependency resolution is one of Terraform's most powerful features — I did not have to tell it the order. It figured it out from the reference.

**`user_data`**
This is a bash script that runs automatically when the EC2 instance starts up for the first time:

```bash
#!/bin/bash
echo "Hello, World" > index.html
nohup busybox httpd -f -p 8080 &
```

Line by line:
- `echo "Hello, World" > index.html` — writes the text "Hello, World" into a file called `index.html`
- `nohup busybox httpd -f -p 8080 &` — starts a lightweight HTTP server using `busybox` on port 8080, serving `index.html`. The `nohup` and `&` keep it running in the background even after the script exits

The `<<-EOF ... EOF` syntax is called a **heredoc**. It lets me write a multiline string directly in the HCL without needing escape characters. Clean and readable.

**`user_data_replace_on_change = true`**
User data scripts run only on the very first boot of an instance. If I change the script and apply, Terraform's default behaviour would be to update the instance in place — but the script would not re-run because the instance has already booted. Setting this to `true` tells Terraform to destroy the old instance and create a fresh one whenever the user data changes, ensuring the new script always gets executed.

**`tags`**
Tags are key-value pairs attached to AWS resources for identification and organisation. The `Name` tag is special — it is what appears as the instance name in the AWS EC2 console.

---

## The Three Commands That Make It All Work

### terraform init

```bash
terraform init
```

**My output:**
```
terraform init 
Initializing the backend...
Initializing provider plugins...
- Finding latest version of hashicorp/aws...
- Installing hashicorp/aws v6.37.0...
- Installed hashicorp/aws v6.37.0 (signed by HashiCorp)
Terraform has created a lock file .terraform.lock.hcl to record the provider
selections it made above. Include this file in your version control repository
so that Terraform can guarantee to make the same selections by default when
you run "terraform init" in the future.

Terraform has been successfully initialized!

You may now begin working with Terraform. Try running "terraform plan" to see
any changes that are required for your infrastructure. All Terraform commands
should now work.

If you ever set or change modules or backend configuration for Terraform,
rerun this command to reinitialize your working directory. If you forget, other
commands will detect it and remind you to do so if necessary.
```

`terraform init` is always the first command to run in any new Terraform project. It does three things:

1. **Downloads provider plugins** — reads your configuration, identifies which providers you're using (`aws` in this case), and downloads the corresponding plugin code into a `.terraform` directory
2. **Creates a lock file** — generates `.terraform.lock.hcl` which records the exact provider versions used, so future runs are consistent
3. **Sets up the backend** — configures where Terraform state will be stored (local by default)

Think of `init` like `npm install` or `pip install` — it prepares the working directory before any real work begins. You only need to run it once per project, or whenever you add a new provider.

**Important:** Add `.terraform/` to your `.gitignore`. It contains downloaded binaries that should not be committed to version control.

---

### terraform plan

```bash
terraform plan
```

**My output:**
```
Terraform will perform the following actions:

  # aws_instance.myFirstInstance will be created
  + resource "aws_instance" "myFirstInstance" {
      + ami                    = "ami-0ec10929233384c7f"
      + instance_type          = "t2.micro"
      + tags                   = {
          + "Name" = "myFirstInstance"
        }
      + vpc_security_group_ids = (known after apply)
      (...)
    }

  # aws_security_group.firewall will be created
  + resource "aws_security_group" "firewall" {
      + name = "instanceFirewall"
      + ingress = [
          + {
              + cidr_blocks = ["0.0.0.0/0"]
              + from_port   = 8080
              + protocol    = "tcp"
              + to_port     = 8080
            },
        ]
      (...)
    }

Plan: 2 to add, 0 to change, 0 to destroy.
```

`terraform plan` is the preview step. It reads your configuration, compares it to the current state of your infrastructure, and prints a detailed diff of exactly what will be created, changed, or destroyed — without touching anything in AWS.

The symbols in the output mean:
- `+` — resource will be created
- `~` — resource will be modified in place
- `-` — resource will be destroyed
- `-/+` — resource will be destroyed and recreated

Notice `vpc_security_group_ids = (known after apply)` — Terraform knows the security group ID will come from the resource being created, but it cannot know the actual value until after `apply` runs. It shows this honestly rather than guessing.

**This is Terraform's most important safety feature.** Always run `plan` before `apply`. Review the output carefully. In a production environment, this output would go into a pull request for a colleague to review before anything is applied.

---

### terraform apply

```bash
terraform apply
```

`apply` runs the plan one more time, shows the same output, and then asks for confirmation:

```
Do you want to perform these actions?
  Terraform will perform the actions described above.
  Only 'yes' will be accepted to approve.

  Enter a value: yes
```

After typing `yes`:

```
aws_security_group.firewall: Creating...
aws_security_group.firewall: Creation complete after 3s [id=sg-0xxxxxxxxxxxxxxxxx]
aws_instance.myFirstInstance: Creating...
aws_instance.myFirstInstance: Still creating... [10s elapsed]
aws_instance.myFirstInstance: Still creating... [20s elapsed]
aws_instance.myFirstInstance: Still creating... [30s elapsed]
aws_instance.myFirstInstance: Creation complete after 38s [id=i-0xxxxxxxxxxxxxxxxx]

Apply complete! Resources: 2 added, 0 changed, 0 destroyed.
```

Two things to notice in the output:
1. The security group was created **before** the EC2 instance — Terraform resolved the dependency automatically from the resource reference
2. The EC2 instance takes about 30–40 seconds to provision — Terraform waits and reports progress

---

## Verifying the Deployment

After `apply` completed, I got the public IP address from the AWS EC2 console and tested the web server:

```bash
curl http://<EC2_PUBLIC_IP>:8080
```

**Output:**
```
Hello, World
```

It worked. A real web server, running on real AWS infrastructure, provisioned entirely from a 30-line Terraform file.

---

## Key Learnings from Day 3

**Resource references create implicit dependencies**

Using `aws_security_group.firewall.id` inside the EC2 instance block did more than just pass a value — it told Terraform that the instance depends on the security group. Terraform resolved this automatically and created resources in the correct order. No explicit `depends_on` was needed.

**The plan → apply workflow is a genuine safety net**

Reading the plan output before applying is not just a formality. It clearly showed me that two resources would be created and nothing would be destroyed. In a production environment, this output is what you put in a pull request for review. A colleague can read it and catch mistakes without needing to understand every line of HCL.

**User data runs once — and only once**

The `user_data_replace_on_change = true` setting was an important insight. Without it, changing the startup script and re-applying would silently do nothing, because the instance has already booted. The flag forces a replacement — destroy the old instance, create a new one — so the new script always runs. Understanding when Terraform modifies in-place versus replaces is critical for predictable deployments.

**Terraform's state file tracks everything**

After `apply`, a `terraform.tfstate` file appeared in the project directory. This JSON file records every resource Terraform created, along with the IDs and metadata AWS assigned. It is how Terraform knows, on the next `plan` or `apply`, what already exists. Do not delete it, do not edit it by hand, and do not commit it to a public repository — it can contain sensitive values.

**`terraform destroy` is a first-class command**

At the end of the day I ran:

```bash
terraform destroy
```

Which cleanly removed both resources — the EC2 instance and the security group — with no manual cleanup required. No orphaned resources, no forgotten security groups, no surprise AWS bill. For a learning environment this is invaluable.

---

## Cleaning Up

Always run `terraform destroy` when you are finished with a learning exercise:

```bash
terraform destroy
```

**Output:**
```
aws_instance.myFirstInstance: Destroying... [id=i-0xxxxxxxxxxxxxxxxx]
aws_instance.myFirstInstance: Still destroying... [10s elapsed]
aws_instance.myFirstInstance: Destruction complete after 32s
aws_security_group.firewall: Destroying... [id=sg-0xxxxxxxxxxxxxxxxx]
aws_security_group.firewall: Destruction complete after 1s

Destroy complete! Resources: 2 destroyed.
```

Everything is gone. Nothing running in AWS means nothing on the bill.

---

*Day 3 complete. First server deployed. Hello, World received. 27 days to go.*

---

**Tags:**

`Terraform` · `IaC` · `InfrastructureAsCode` · `HashiCorp` · `HCL` · `TerraformCLI` · `TerraformSetup` · `TerraformRegistry` · `30DayTerraformChallenge` · `TerraformChallenge` · `AWS` · `AWSEC2` · `AWSCloud` · `AWSUG` · `AWSUserGroupKenya` · `EveOps` · `MeruHashiCorpUserGroup` · `EC2` · `SecurityGroup` · `IAM` · `VPC` · `AMI` · `UserData` · `DevOps` · `DevSecOps` · `GitOps` · `SRE` · `PlatformEngineering` · `CloudEngineering` · `CloudComputing` · `CloudNative` · `CloudArchitecture` · `CloudSecurity` · `MultiCloud` · `Serverless` · `Docker` · `Kubernetes` · `CI_CD` · `CICD` · `Automation` · `VersionControl` · `Git` · `GitHub` · `VSCode` · `macOS` · `Homebrew` · `AppleSilicon` · `OpenSource` · `SoftwareEngineering` · `BackendEngineering` · `SystemDesign` · `Programming` · `Coding` · `Tech` · `Technology` · `Engineering` · `LearningInPublic` · `100DaysOfCode` · `TechCommunity` · `TechAfrica` · `AfricaInTech` · `KenyaTech` · `WomenInTech` · `Developer` · `CloudDeveloper` · `DevOpsEngineer` · `CloudEngineer` · `Tutorial` · `TechBlog` · `TechEducation` · `LearnToCode` · `CodeNewbie` · `BeginnerDeveloper`
