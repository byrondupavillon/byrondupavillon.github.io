---
title: "📦 How to Import Existing Terraform Resources with Import Blocks"
date: 2025-08-11
description: "Easily bring existing AWS resources under Terraform management using import blocks and generated configuration—no more manual reverse-engineering."
categories:
  - terraform
tags:
  - aws
  - infrastructure-as-code
  - terraform
  - import
  - devops
---

Since Terraform v1.5, you can use **import blocks** to bring existing resources into Terraform state—**and** generate matching configuration automatically. This replaces the old, manual `terraform import` process.

Here’s a step-by-step example importing an **AWS Security Group**.

## 🧾 1. Prerequisites

You’ll need:

- **Terraform** v1.5+
- AWS provider configured (credentials + region)
- A Terraform working directory

Example provider config:

```hcl
# versions.tf
terraform {
  required_version = ">= 1.5.0"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

# provider.tf
provider "aws" {
  region = "eu-west-1"
}
```

Initialize Terraform:

```bash
terraform init
```

## 📜 2. Create the Import Block

Declare what you want to import in a new `import.tf` file:

```hcl
# import.tf
import {
  to = aws_security_group.security_group_name
  id = "sg-903004f8"
}
```

- `to` = the **Terraform resource address** you want to use  
- `id` = the **real AWS resource ID**

> 💡 If importing into a module, use the full module address, e.g.:
> `to = module.network.aws_security_group.web`

## 🛠️ 3. Generate Starter Configuration

Run:

```bash
terraform plan -generate-config-out=output.tf
```

Terraform will:

1. Read your `import` block  
2. Query AWS for the live resource  
3. Write a **proposed configuration** to `output.tf`

This step **does not** import anything yet—it just scaffolds the config.

## ✏️ 4. Move and Refactor the Generated Config

Open `output.tf`, review it, and move it into your actual Terraform files (e.g., `security-group.tf`).

While doing so:

- Keep the **same resource address** as in `import.tf`
- Replace hard-coded values with variables or data sources
- Match your project’s style (inline rules vs. separate resources)

Example (simplified):

```hcl
# security-group.tf
resource "aws_security_group" "security_group_name" {
  name        = "web-sg"
  description = "Web security group"
  vpc_id      = data.aws_vpc.main.id

  ingress {
    description = "HTTP"
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name = "web-sg"
  }
}

data "aws_vpc" "main" {
  default = true
}
```

## 📥 5. Import the Resource

With your final config in place, run:

```bash
terraform apply
```

Terraform will:

1. Execute the `import` block to create a **state entry**
2. Compare your config with the imported resource
3. Plan any changes (ideally **no changes** right after import)

If you see unexpected changes, adjust your config to match the live resource.

## 🧹 6. Clean Up

Once the import is complete:

1. Remove the `import.tf` file
2. Run:

```bash
terraform plan
```

You should see:

```
No changes. Your infrastructure matches the configuration.
```

## ⚠️ Common Gotchas

- **Terraform <1.5** → use legacy `terraform import`
- **Wrong region/account** → provider must point to the correct target
- **Modules** → use full module address in `to = ...`
- **`for_each` / `count`** → include key/index in the address
- **Style mismatch** → refactor generated code before applying
- **Drift** → expected if you intentionally change config after import

## ✅ Done!

You’ve now:

- Declared an **import block**
- Generated **matching configuration**
- Imported the resource into **state**
- Cleaned up the temporary import block

This workflow makes importing resources **repeatable, reviewable, and safer** than ever.
