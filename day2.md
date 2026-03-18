# Day 2 — Step-by-Step Guide to Setting Up Terraform, AWS CLI, and Your AWS Environment

> **Day 2 — 30-Day Terraform Challenge**
> **Communities:** AWS AI/ML UserGroup Kenya · Meru HashiCorp User Group · EveOps
> **Date:** March 18, 2026
> **Machine:** MacBook (macOS · Apple Silicon)

---

## Table of Contents

- [My Setup at a Glance](#my-setup-at-a-glance)
- [Step 1 — Create and Secure Your AWS Account](#step-1--create-and-secure-your-aws-account)
- [Step 2 — Install Homebrew](#step-2--install-homebrew)
- [Step 3 — Install Terraform](#step-3--install-terraform)
- [Step 4 — Install the AWS CLI](#step-4--install-the-aws-cli)
- [Step 5 — Install VS Code and Extensions](#step-5--install-vs-code-and-extensions)
- [Step 6 — Verify the Full Setup](#step-6--verify-the-full-setup)
- [Issues I Ran Into](#issues-i-ran-into)
- [Key Decisions I Made and Why](#key-decisions-i-made-and-why)
- [Setup Checklist](#setup-checklist)
- [Key Takeaway from Day 2](#key-takeaway-from-day-2)

---

## My Setup at a Glance

Day 1 was about understanding the *why* behind Terraform and Infrastructure as Code. Day 2 is where the hands-on work begins. Before writing a single `.tf` file, I needed a fully working local environment — Terraform installed, AWS connected, and everything verified end to end.

Here is exactly what I was working with going into this:

| Item | Details |
|---|---|
| Machine | MacBook (macOS · Apple Silicon) |
| AWS Account | Brand new — created specifically for this challenge |
| Installation method | Homebrew for all CLI tools |
| Default AWS region | us-east-1 (N. Virginia) |
| IAM User | `terraform-admin` with `AdministratorAccess` |

One decision I made early on: I created a brand new AWS account dedicated entirely to this challenge rather than reusing an existing account. The reasons were straightforward — clean isolation, no risk of touching existing resources, and a billing dashboard that would reflect only what I was building. For anyone starting this challenge, I'd recommend doing the same.

---

## Step 1 — Create and Secure Your AWS Account

If you are starting fresh, go to [aws.amazon.com](https://aws.amazon.com) and create a new account. You will need an email address, a phone number for identity verification, and a credit card. AWS requires the card even for free tier usage, but the resources we provision throughout this challenge comfortably fall within the free tier limits.

Once the account is active, the very first priority — before installing anything locally — is to secure it properly.

### Enable MFA on the root account

The root account has unrestricted access to every resource and setting in your AWS environment. It cannot be scoped or limited by IAM policies. Losing control of it would be catastrophic, so the first thing I did was enable multi-factor authentication:

1. Sign in to the AWS Console using your root email
2. Click your account name at the top right and go to **Security credentials**
3. Under **Multi-factor authentication (MFA)**, select **Assign MFA device**
4. Choose **Authenticator app** and follow the on-screen steps — I used **Twilio Authy**

From this point on, the root account stays locked away. All day-to-day work happens through an IAM user.

### Create an IAM user for everyday use

1. Navigate to **IAM → Users → Create user**
2. Set the username — I used `terraform-admin`
3. Enable console access if you also want to log in via the browser
4. On the permissions screen, choose **Attach policies directly** and select `AdministratorAccess`
5. Once the user is created, go to its **Security credentials** tab
6. Under **Access keys**, click **Create access key** and select **CLI** as the intended use
7. Download the `.csv` file immediately — this is the only time AWS will display the secret access key

Store the `.csv` somewhere secure — a password manager works well. You will need the Access Key ID and Secret Access Key in Step 4.

---

## Step 2 — Install Homebrew

Homebrew is the standard package manager for macOS and the cleanest way to manage CLI tools on a Mac. If it is not already installed, open Terminal and run:

```bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```

Once the installation finishes, confirm it is working:

```bash
brew --version
```

**My output:**
```
Homebrew 5.1.0
```

### Apple Silicon (M1/M2/M3) note

On Apple Silicon Macs, Homebrew installs to `/opt/homebrew` rather than `/usr/local`. At the end of the installation, the script will prompt you to run two commands to add Homebrew to your shell PATH. Run both of them — they look like this:

```bash
echo 'eval "$(/opt/homebrew/bin/brew shellenv)"' >> ~/.zshrc
eval "$(/opt/homebrew/bin/brew shellenv)"
```

Skipping this step means every tool installed through Homebrew will return `command not found` when you try to use it. This was the first wall I hit, and it costs you five minutes you don't need to spend.

---

## Step 3 — Install Terraform

With Homebrew ready, installing Terraform is two commands. The first adds HashiCorp's official tap — their package repository — and the second installs Terraform from it:

```bash
brew tap hashicorp/tap
brew install hashicorp/tap/terraform
```

Using the official HashiCorp tap ensures the binary comes directly from HashiCorp rather than a third-party source. Once it finishes, verify the installation:

```bash
terraform version
```

**My output:**
```
Terraform v1.14.7
on darwin_arm64
```

Terraform is a single self-contained binary with no runtime dependencies. That clean output — version number plus OS and architecture — is all you need to see to know it is ready.

To upgrade Terraform in the future:

```bash
brew update
brew upgrade hashicorp/tap/terraform
```

---

## Step 4 — Install the AWS CLI

Install the AWS CLI through Homebrew:

```bash
brew install awscli
```

Verify the installation:

```bash
aws --version
```

**My output:**
```
aws-cli/2.30.4 Python/3.13.7 Darwin/25.3.0 exe/arm64
```

Now connect it to your AWS account using the IAM credentials from Step 1:

```bash
aws configure
```

The CLI will ask for four values:

```
AWS Access Key ID [None]: YOUR_ACCESS_KEY_ID
AWS Secret Access Key [None]: YOUR_SECRET_ACCESS_KEY
Default region name [None]: us-east-1
Default output format [None]: json
```

**Why us-east-1?** It is the oldest and most feature-complete AWS region, and new services almost always become available here first. For a learning environment it is the safest default — you are unlikely to encounter a situation where a needed service is not yet available in this region.

Your credentials are stored locally at `~/.aws/credentials`. Terraform reads from this location automatically when authenticating with AWS, so no additional configuration is needed.

To verify your stored credentials at any time:

```bash
cat ~/.aws/credentials
```

---

## Step 5 — Install VS Code and Extensions

Download and install Visual Studio Code from [code.visualstudio.com](https://code.visualstudio.com).

After installation, set up the `code` shell command so you can open any project folder directly from Terminal:

1. Open VS Code
2. Press `Cmd+Shift+P` to open the Command Palette
3. Type **Shell Command: Install 'code' command in PATH** and press Enter

Test it:

```bash
code .
```

This should open VS Code in your current directory. You will use this constantly throughout the challenge.

### Required extensions

Open the Extensions panel with `Cmd+Shift+X` and install both of the following:

**HashiCorp Terraform**
Adds syntax highlighting, intelligent auto-complete, and real-time validation for `.tf` files. As you type, it suggests valid resource arguments and underlines configuration errors before you run a single command. Working with Terraform files without this extension is like writing code with no IDE — technically possible, but unnecessarily painful.

**AWS Toolkit**
Embeds AWS directly into VS Code. You can browse S3 buckets, EC2 instances, Lambda functions, and more from the VS Code sidebar without switching to the browser. When Terraform creates a resource, this is where you go to verify it looks exactly as expected.

---

## Step 6 — Verify the Full Setup

With all components installed, I ran the two verification commands that confirm the environment is fully ready:

**Terraform version check:**

```bash
terraform version
```

**My output:**
```
Terraform v1.14.7
on darwin_arm64
```

**AWS identity check:**

```bash
aws sts get-caller-identity
```

**My output:**
```json
{
    "UserId": "AIDA1234567890EXAMPLE",
    "Account": "123456789012",
    "Arn": "arn:aws:iam::123456789012:user/terraform-admin"
}
```

Three things to read from that second command:

- **UserId** — the AWS CLI found and loaded valid credentials
- **Account** — you are pointing at the correct AWS account
- **Arn** — critically, this shows `user/terraform-admin`, not `root`. That confirms you are working as your IAM user, not the root account

When both commands return clean output, the environment is ready. This `aws sts get-caller-identity` command is worth memorising — any time something authentication-related breaks later in the challenge, this is the first thing to run.

---

## Issues I Ran Into

### Issue 1 — `command not found: terraform` after Homebrew install

**Problem:**
```
zsh: command not found: terraform
```

**Root cause:** On Apple Silicon, Homebrew installs to `/opt/homebrew/bin`, which was not in my shell PATH after the initial install.

**Fix:**
```bash
echo 'export PATH="/opt/homebrew/bin:$PATH"' >> ~/.zshrc
source ~/.zshrc
terraform version
```

**Tip for others:** Always reload your shell after a Homebrew installation on Apple Silicon. If `brew --version` works but a tool you just installed does not, a missing PATH entry is almost always the reason.

---

### Issue 2 — `aws sts get-caller-identity` returned an auth error

**Problem:**
```
An error occurred (InvalidClientTokenId) when calling the
GetCallerIdentity operation: The security token included
in the request is invalid.
```

**Root cause:** Trailing whitespace was added when pasting the secret access key during `aws configure`.

**Fix:**
```bash
aws configure
# Re-entered both keys carefully, character by character
aws sts get-caller-identity
# Returned clean output
```

**Tip for others:** If you are copying credentials from a browser or PDF, paste them into a plain text editor first to strip any invisible characters before entering them into `aws configure`.

---

### Common macOS-specific issues

**VS Code `code` command not working after setup:**
Close Terminal completely and reopen it. The PATH change takes effect only in new sessions.

**Terraform blocked by macOS Gatekeeper (manual install only):**
If you downloaded the binary manually rather than through Homebrew, macOS may quarantine it. Fix:
```bash
xattr -d com.apple.quarantine /usr/local/bin/terraform
```
This is one more reason to use the Homebrew install path — it handles Gatekeeper automatically.

---

## Key Decisions I Made and Why

**Homebrew over a manual binary download**
Homebrew manages upgrades, handles PATH configuration, and pulls directly from HashiCorp's official tap. A manually downloaded binary is one more thing to maintain and one more source of version drift.

**A dedicated AWS account for this challenge**
Every resource in this account exists because I created it. No shared credentials, no pre-existing infrastructure to accidentally modify, and a billing dashboard that maps cleanly to what the challenge is consuming.

**us-east-1 as the default region**
The most feature-complete and widely-supported AWS region. New services launch here first. The safest default for a learning context.

**IAM user over root for all daily work**
Root credentials cannot be scoped, restricted, or audited with IAM policies. Building the habit of working through a scoped IAM user from day one is the right practice — it is what every production team does, and this challenge is good training ground.

---

## Setup Checklist

| Task | Status |
|---|---|
| New AWS account created | ✅ Done |
| Root account MFA enabled | ✅ Done |
| IAM user `terraform-admin` created | ✅ Done |
| `AdministratorAccess` policy attached | ✅ Done |
| Access key and secret downloaded securely | ✅ Done |
| Homebrew installed and verified | ✅ Done |
| Terraform installed via `brew install hashicorp/tap/terraform` | ✅ Done |
| `terraform version` returns clean output | ✅ Done |
| AWS CLI installed via `brew install awscli` | ✅ Done |
| AWS CLI configured via `aws configure` with `us-east-1` | ✅ Done |
| `aws sts get-caller-identity` returns IAM user ARN | ✅ Done |
| VS Code installed | ✅ Done |
| `code` command working in Terminal | ✅ Done |
| HashiCorp Terraform extension installed | ✅ Done |
| AWS Toolkit extension installed | ✅ Done |

---

## Key Takeaway from Day 2

Setup days feel slow. Nothing gets built, no infrastructure gets provisioned, and it is tempting to rush through and skip the verification steps. That is a mistake. Every item skipped here becomes a harder-to-diagnose problem later, when you are deep in a configuration and cannot tell whether the error is in your Terraform code or your environment.

`aws sts get-caller-identity` is the single most useful diagnostic command in this entire setup. Run it now, remember it, and run it again any time authentication breaks later in the challenge. Two seconds, definitive answer.

---

*Day 2 complete. Environment fully configured and verified on macOS. Day 3: writing the first Terraform configuration file and provisioning the first real AWS resource entirely from code.*

---

**Tags:**

`Terraform` · `IaC` · `InfrastructureAsCode` · `HashiCorp` · `HCL` · `TerraformSetup` · `TerraformCLI` · `30DayTerraformChallenge` · `TerraformChallenge` · `AWS` · `AWSCLI` · `AWSCloud` · `AWSUG` · `AWSUserGroupKenya` · `EveOps` · `MeruHashiCorpUserGroup` · `IAM` · `DevOps` · `DevSecOps` · `CloudEngineering` · `CloudComputing` · `CloudNative` · `PlatformEngineering` · `SRE` · `GitOps` · `Automation` · `Git` · `GitHub` · `VSCode` · `macOS` · `Homebrew` · `AppleSilicon` · `OpenSource` · `SoftwareEngineering` · `LearningInPublic` · `100DaysOfCode` · `TechCommunity` · `TechAfrica` · `AfricaInTech` · `KenyaTech` · `LearnToCode` · `CodeNewbie` · `Tutorial` · `TechBlog` · `CloudDeveloper` · `DevOpsEngineer` · `Tech` · `Engineering`  
