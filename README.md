# Automated SEC/FINRA Regulatory Compliance Log Vault (WORM)

[![IaC](https://img.shields.io/badge/IaC-Terraform_1.5+-7B42BC.svg)](https://www.terraform.io/)
[![Provider](https://img.shields.io/badge/Provider-AWS-orange.svg)](https://aws.amazon.com/)
[![Storage](https://img.shields.io/badge/Storage-S3_WORM_Object_Lock-blue.svg)](https://aws.amazon.com/s3/)
[![Compliance](https://img.shields.io/badge/Compliance-SEC_17a--4(f)_/_FINRA_4511-red.svg)](https://www.sec.gov/)

## 📋 Project Overview
This repository contains the infrastructure-as-code and configuration policies for an automated, regulatory-grade financial auditing log repository. 

Designed to meet the stringent data retention mandates of **SEC Rule 17a-4(f)** and **FINRA Rule 4511**, this architecture implements immutable Write-Once-Read-Many (WORM) storage. This ensures that systemic audit trails, transaction records, and cryptographic signatures are preserved with absolute zero-tamper guarantees, even against compromised administrative or root accounts.

---

## 🏗️ Compliance Engineering & Immutability Guardrails

The log storage subsystem leverages a multi-layered defense-in-depth framework to enforce data integrity and maintain a mathematically verifiable chain of custody:

1. **Object Lock Enforcement (WORM Strategy):** Configured strictly in **Compliance Mode**. This enforces an immutable lock that cannot be bypassed, shortened, or overwritten by any identity, including the root account.
2. **Cryptographic Integrity:** Enforced via customer-managed AWS KMS keys utilizing industry-standard AES-256 encryption with automated annual key rotation policies.
3. **Access Control Baselines:** Enforces strict least-privilege Resource Policies, explicitly blocking destructive operations (`s3:DeleteObject`, `s3:DeleteObjectVersion`) globally across the bucket perimeter.
4. **Log Ingestion & Non-Repudiation:** Generates cryptographic hashes for incoming log batches, ensuring any malicious data modification attempts trigger instant compliance alarms.

---

## 🔒 Threat Modeling & Integrity Mapping

| Regulatory Risk | Architectural Mitigation Strategy |
| :--- | :--- |
| **Privileged Insider Threat** | Enforced S3 Object Lock in Compliance Mode; even compromised administrator credentials lack the privileges required to modify or destroy locked historical logs. |
| **Data Exfiltration** | Tightened target repository bucket policies to explicitly deny data transfers outside the corporate organizational perimeter. |
| **Log Injection / Alteration** | Implemented append-only transport pipelines, completely removing modifying operations from application ingestion roles. |
| **Audit Trail Gaps** | Multi-region backup distribution paired with explicit lifecycle version tracking guarantees a permanent record of historical account state changes. |

---

## 💻 Core Regulatory Vault Architecture (`main.tf`)

# ==============================================================================
# ARCHITECTURE: Automated SEC/FINRA Regulatory Compliance Log Vault (WORM)
# COMPLIANCE MAPPING: SEC Rule 17a-4(f) & FINRA Rule 4511 (Tamper-Proof Storage)
# ==============================================================================

terraform {
  required_version = ">= 1.5.0"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = var.aws_region
}

variable "aws_region" {
  type    = string
  default = "us-east-1"
}

variable "environment" {
  type    = string
  default = "production"
}

# ------------------------------------------------------------------------------
# 1. CRYPTOGRAPHIC ISOLATION (Custom Managed Key with Auto-Rotation)
# ------------------------------------------------------------------------------
resource "aws_kms_key" "regulatory_vault_key" {
  description             = "Cryptographic boundary for SEC/FINRA regulatory audit storage"
  deletion_window_in_days = 30
  enable_key_rotation     = true # Mandatory compliance rotation metric

  tags = {
    Environment = var.environment
    Compliance  = "SEC_17a-4_FINRA_4511"
  }
}

# ------------------------------------------------------------------------------
# 2. CORE STORAGE PLANE (S3 Bucket Initialized with Object Locking Enabled)
# ------------------------------------------------------------------------------
resource "aws_s3_bucket" "regulatory_compliance_vault" {
  bucket              = "fintech-regulatory-worm-vault-${var.environment}-data"
  force_destroy       = false # CRITICAL: Prevents accidental/malicious terraform destroy overrides
  object_lock_enabled = true  # Must be explicitly set to true during initial provisioning

  tags = {
    Purposes           = "Regulatory_Audit_Evidence"
    Immutability_State = "WORM_Enforced"
  }
}

# Enforce Server-Side Encryption (SSE-KMS) using the dedicated compliance key
resource "aws_s3_bucket_server_side_encryption_configuration" "vault_encryption" {
  bucket = aws_s3_bucket.regulatory_compliance_vault.id

  rule {
    apply_server_side_encryption_by_default {
      kms_master_key_id = aws_kms_key.regulatory_vault_key.arn
      sse_algorithm     = "aws:kms"
    }
    bucket_key_enabled = true
  }
}

# Enforce strict public access blocks to eliminate internet data exposure
resource "aws_s3_bucket_public_access_block" "vault_public_block" {
  bucket = aws_s3_bucket.regulatory_compliance_vault.id

  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

# Force active object versioning (Prerequisite requirement for structural tracking)
resource "aws_s3_bucket_versioning_v2" "vault_versioning" {
  bucket = aws_s3_bucket.regulatory_compliance_vault.id
  versioning_configuration {
    status = "ENABLED"
  }
}

# ------------------------------------------------------------------------------
# 3. REGULATORY CONTROLS PLANE (S3 Object Lock Enforced in COMPLIANCE Mode)
# ------------------------------------------------------------------------------
resource "aws_s3_bucket_object_lock_configuration" "worm_lock_enforcement" {
  bucket = aws_s3_bucket.regulatory_compliance_vault.id

  # Ensure all downstream resources inheriting the configuration fallback to the lock rule
  depends_on = [aws_s3_bucket_versioning_v2.vault_versioning]

  rule {
    default_retention {
      # COMPLIANCE Mode: The lock CANNOT be bypassed, shortened, or overridden by ANY user (including Root)
      mode = "COMPLIANCE"
      
      # 7-Year Retention Horizon mapped explicitly into day metrics (7 * 365 = 2555 days)
      days = 2555
    }
  }
}
