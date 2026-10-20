---
domain: devops
tags: [devops, iac, terraform]
sources: []
updated: 2026-10-19
---

# Terraform — Learning Guide

## What problem does Terraform solve?

Imagine you need to create a database, a web app, and a storage account in Azure. You could:

- **Click in the portal** — works once, but how do you recreate it? How do you know if someone changed something without telling you?
- **Write shell scripts** — better, but run it twice and you get errors ("resource already exists"). And the order matters — you have to figure out what depends on what.
- **Write Terraform** — you describe *what you want to exist*. Terraform figures out the order, creates it, and remembers what it did. Run it again and nothing happens — it's already there.

Terraform is **Infrastructure as Code (IaC)**. By writing infrastructure as code, you get all the tools code already has: git history, pull request reviews, reproducibility, and the ability to know exactly what's deployed.

---

## Phase 1 — Reading the blueprint

### The file convention

Terraform reads **all `.tf` files** in a folder and merges them into one config. It doesn't care about filenames. But the community splits them by role:

| File | Role | Analogy |
|---|---|---|
| `variables.tf` | Inputs — things you can customize | Function parameters |
| `main.tf` | The actual infrastructure | Function body |
| `outputs.tf` | Values Terraform reports back | Function return value |

### The `terraform` block — settings and safety guards

```hcl
terraform {
  required_version = ">= 1.5"

  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 4.0"
    }
  }

  backend "azurerm" {
    resource_group_name  = "rg-demo-lda-knowledge-studio"
    storage_account_name = "stldaknowledgeapp"
    container_name       = "tfstate"
    key                  = "lda-knowledge.tfstate"
  }
}
```

Breaking it down:

- **`required_version = ">= 1.5"`** — "I need Terraform 1.5 or newer." If someone on the team has an old version, Terraform refuses to run instead of silently breaking things.

- **`version = "~> 4.0"`** — "I need the Azure provider version 4.x." The `~>` is the **pessimistic constraint**: 4.0 or higher, but not 5.0. Because 4.1 → 4.2 are backward-compatible updates, but 5.0 could break things. This is exactly like `"^4.0.0"` in `package.json`.

- **`backend "azurerm"`** — This is where the **state** lives. The state file is stored in an Azure Storage Account so that everyone on the team shares the same truth. Without this, each person would have a local state file and they'd overwrite each other's infrastructure.

**Why are the backend values hardcoded?** Terraform needs to know where the state is *before* it can read variables. Chicken-and-egg problem. Everything else can use variables; the backend can't.

### Provider — the connection to Azure

```hcl
provider "azurerm" {
  features {}
  subscription_id                 = var.subscription_id
  resource_provider_registrations = "none"
}
```

"Connect to Azure using this subscription." The `var.subscription_id` references a variable defined in `variables.tf`. The `features {}` block is required by the Azure provider even if empty.

### `resource` vs `data` — the key distinction

This is one of the most important concepts:

```hcl
# resource: Terraform CREATES and MANAGES this
resource "azurerm_resource_group" "mine" {
  name     = "rg-utopia"
  location = "westeurope"
}

# data: Terraform READS this (it already exists, someone else created it)
data "azurerm_resource_group" "existing" {
  name = "rg-already-there"
}
```

Think of it as: **`resource` = "create this for me"**, **`data` = "go find this for me"**.

Why use `data`? Because some things already exist and are shared. Your resource group might be shared with other teams. You don't want Terraform to own it (or worse, delete it on `terraform destroy`).

### Variables — the inputs

```hcl
variable "subscription_id" {
  description = "Azure subscription ID"
  type        = string
  default     = "76600b30-..."
}
```

Each variable has:
- **`description`** — documentation for humans
- **`type`** — Terraform validates this (string, number, bool, list, map...)
- **`default`** — if nobody provides a value, use this one

If a variable has **no `default`**, Terraform will ask you for it every time you run `plan` or `apply`.

**Sensitive variables** hide their value in output:

```hcl
variable "db_password" {
  type      = string
  sensitive = true    # shows (sensitive value) instead of the real password
}
```

You pass them via:
- `terraform plan -var="db_password=my-secret"`
- A `terraform.tfvars` file (never commit this to git)
- An environment variable `TF_VAR_db_password`

### Outputs — the return values

```hcl
output "resource_group_id" {
  value = data.azurerm_resource_group.main.id
}
```

Outputs serve **three purposes**, from simplest to most important:

**1. Print values after `apply`.** After Terraform finishes, it shows:

```
Outputs:
dev_app_url = "https://utopia-dev.azurewebsites.net"
```

You can also retrieve them anytime with `terraform output dev_app_url`.

**2. Feed values into scripts and CI/CD:**

```bash
APP_URL=$(terraform output -raw dev_app_url)
curl "$APP_URL/health"
```

This is how Terraform connects to the rest of your pipeline. It creates the infrastructure, then *tells* the next step where things are.

**3. Connect modules together (the main reason they exist).** When you split Terraform into modules (networking, database, app), they share data through outputs. Module A outputs a value, module B consumes it. Without outputs, modules can't talk to each other.

**Variables go *into* a config. Outputs come *out*.** Like a function signature.

### The referencing syntax

References follow a consistent pattern — `type.local_name.attribute`:

- `azurerm_resource_group.main.name` → resource attribute
- `data.azurerm_resource_group.existing.id` → data source attribute (note the `data.` prefix)
- `var.location` → variable value
- `module.db.connection_string` → output from another module

### String interpolation

Works like JavaScript template literals:

```hcl
value = "https://${azurerm_linux_web_app.dev.default_hostname}"
```

The `${}` inside quotes inserts a dynamic value into a string.

### Reading a `plan` output

When you run `terraform plan`, the output has two sections:

```
# Section 1: real infrastructure changes
Terraform will perform the following actions:
  + azurerm_storage_container.docs will be created
  ~ azurerm_resource_group.main will be updated
  - azurerm_redis_cache.old will be destroyed

Plan: 1 to add, 1 to change, 1 to destroy.

# Section 2: output bookkeeping
Changes to Outputs:
  + resource_group_id = "/subscriptions/..."
```

**"Changes to Outputs"** does NOT mean infrastructure is being created. The `+` on an output just means "saving a new value to the state file." Terraform itself clarifies this with the line: *"without changing any real infrastructure."*

The `+`, `~`, `-` symbols mean "add/change/remove" — but *what* is being added depends on the block type. On resources, it's real infrastructure. On outputs, it's just bookkeeping.

---

## Phase 2 — The mental model

Four concepts. Once these click, you can read any Terraform config and reason about what it does.

### 1. Declarative vs Imperative

| Approach | You describe | Run it twice |
|---|---|---|
| **Imperative** (shell script) | The steps to execute | Error: "already exists" |
| **Declarative** (Terraform) | The desired end state | No changes — already matches |

You never say *how* to create a resource. You say *what* should exist. Terraform figures out the API calls, the ordering, and whether anything needs to change at all.

This makes Terraform **idempotent** — you can run it 100 times and the result is always the same. This is the single most important property of IaC.

### 2. The dependency graph

When one resource references another, Terraform sees the dependency:

```hcl
resource "azurerm_resource_group" "main" {
  name     = "rg-utopia"
  location = "westeurope"
}

resource "azurerm_storage_account" "main" {
  resource_group_name = azurerm_resource_group.main.name   # ← reference = dependency
}
```

Terraform builds a **directed graph** automatically:

```
        resource_group
           /       \
          v         v
  storage_account   postgresql_server    ← these two run in parallel
```

- You never write "step 1, step 2, step 3."
- Terraform parallelizes independent resources automatically.
- If you remove storage_account from your config, Terraform knows it can delete it without affecting postgresql_server (no dependency between them).

**One rule**: if resource A references resource B, then A depends on B. That's it.

### 3. The CRUD lifecycle

Every resource goes through a lifecycle. When you run `terraform apply`, Terraform compares your `.tf` files against the state and decides, for each resource, one of four actions:

| State file | Your .tf config | Action |
|---|---|---|
| Doesn't exist | Exists | **Create** (green `+`) |
| Exists | Exists, same values | **No change** |
| Exists | Exists, different values | **Update** (yellow `~`) |
| Exists | Doesn't exist | **Delete** (red `-`) |

`terraform plan` shows you this diff. It's like `git diff` but for infrastructure.

**Deleting a block from your `.tf` file is dangerous** — it doesn't just "stop managing" the resource, it **destroys** it. Remove the postgres block → Terraform deletes your database. This catches people off guard exactly once.

**To stop managing a resource without deleting it**, use `terraform state rm`:

```bash
terraform state rm azurerm_postgresql_server.main
```

Or the newer `removed` block (Terraform 1.7+), which is better because it's in code and reviewable:

```hcl
removed {
  from = azurerm_postgresql_server.main
  lifecycle {
    destroy = false
  }
}
```

### 4. Mutable vs immutable changes

Not all updates are equal. Some changes Terraform can do **in-place** (like changing a tag). Others **force a destroy-and-recreate** (like renaming a storage account — the cloud API literally doesn't support it).

`terraform plan` always tells you which:

```
~ tags = { "env" = "dev" -> "staging" }              # in-place update, safe
~ name = "stold" -> "stnew"    # forces replacement  # destroys and recreates
```

The words **"forces replacement"** are your danger signal. They mean downtime and potential data loss. Always read the plan.

### Drift — when reality diverges from code

If someone goes to the Azure portal and manually changes a tag on a resource Terraform manages, here's what happens on the next `plan`:

- Your .tf file says: `tag = "production"`
- Azure reality says: `tag = "testing"` (your colleague changed it)
- Terraform's plan: `~ tag = "testing" → "production"` — **it reverts the manual change**

Terraform doesn't care about who changed it or why. It only knows: "the code says X, reality says Y, I'll make reality match the code." The `.tf` files are the **single source of truth**. Manual changes get overwritten.

---

## Phase 3 — The commands

### `terraform init`

Downloads providers, connects to the backend. Like `pnpm install`.

Run it when:
- First time in a new Terraform directory
- You add a new provider
- You change the backend config
- You pull code that adds a new provider

Creates `.terraform/` (like `node_modules/` — never commit) and `.terraform.lock.hcl` (like `pnpm-lock.yaml` — always commit).

### `terraform plan`

The most important command. **Never modifies anything.** Always safe to run.

Does three things in order:
1. **Refresh** — calls Azure's API to check real resource state
2. **Diff** — compares real state against your .tf config
3. **Output** — shows you exactly what it would do

The summary line at the bottom is what you always read:

```
Plan: 1 to add, 0 to change, 0 to destroy.
```

### `terraform apply`

Two ways:

```bash
terraform apply              # shows the plan, asks "yes/no?"
terraform apply plan.out     # runs a saved plan, no question
```

The safer workflow (what teams do in CI/CD):

```bash
terraform plan -out=plan.out    # save the plan to a file
# review it, maybe in a PR
terraform apply plan.out        # apply exactly what was reviewed
```

Why save the plan? Because between `plan` and `apply`, someone could change the infrastructure. The saved plan guarantees you apply *exactly* what you reviewed.

### `terraform destroy`

Deletes **everything** Terraform manages. The nuclear option. In practice, you almost never use it — maybe to tear down a dev environment.

---

## Self-assessment quiz

### Block 1 — Concepts

1. What's the difference between imperative (shell scripts) and declarative (Terraform)?
2. What does "idempotent" mean? Why is it important?
3. What is "state" in Terraform and where does it live?

### Block 2 — Reading config

4. What's the difference between a `resource` block and a `data` block?
5. Why is the `backend` block hardcoded instead of using variables?
6. You see `resource_group_name = azurerm_resource_group.main.name` in a resource. What does this tell Terraform about ordering?

### Block 3 — Reasoning about behavior

7. You delete a resource block from your .tf file and run `apply`. What happens?
8. A colleague manually changes a tag in the Azure portal. What does `terraform plan` show next time you run it?
9. You see "forces replacement" in a plan. What does that mean and why is it dangerous?
10. The plan shows `Changes to Outputs: + app_url = "..."`. Is Azure infrastructure being created?

### Answers

1. Imperative: you describe the steps ("create A, then B"). Declarative: you describe the end state ("I want A and B to exist"). Terraform figures out the steps.
2. Running it multiple times produces the same result. Important because you can run Terraform safely without worrying about duplicating resources.
3. A JSON file recording what infrastructure Terraform manages. It lives in a remote backend (Azure Storage, S3) so the team shares a single source of truth.
4. `resource` = "create and manage this for me." `data` = "go find this thing that already exists." Terraform owns resources (can update/delete them) but only reads data sources.
5. Terraform needs the state location *before* it can read variables. Chicken-and-egg problem.
6. It creates an implicit dependency. Terraform knows it must create the resource group before this resource. You never specify ordering manually.
7. Terraform destroys the real resource in Azure. Removing from code = removing from reality.
8. Terraform proposes to **revert** the manual change back to what the .tf file says. The code is the single source of truth.
9. Terraform can't update the resource in-place (the cloud API doesn't support it), so it will **destroy** the old one and **create** a new one. Dangerous because it means downtime and potential data loss.
10. No. The `+` on an output means "saving a new value to the state file." Outputs are bookkeeping, not infrastructure. Terraform even says "without changing any real infrastructure."

## Relations

- [[ci-cd-pipeline]] — Terraform runs inside CI/CD to automate infrastructure changes
- [[environments]] — Terraform can manage multiple environments via workspaces or separate configs
- [[deployment-strategies]] — infrastructure changes (like swapping slots) are part of deployment

## Tags

#devops #iac #terraform
