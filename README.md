# Automated SEC/FINRA Regulatory Compliance Log Vault (WORM)

[![Provider](https://img.shields.io/badge/Storage-AWS_S3_Glacier-orange.svg)](https://aws.amazon.com/s3/)
[![Category](https://img.shields.io/badge/Architecture-Immutable_Log_Vault-blue.svg)](https://aws.amazon.com/s3/)
[![Governance](https://img.shields.io/badge/Governance-Object_Lock_Compliance-green.svg)](https://docs.aws.amazon.com/AmazonS3/latest/userguide/object-lock.html)
[![Compliance](https://img.shields.io/badge/Compliance-SEC_17a--4_/_FINRA_4511-red.svg)](https://www.sec.gov/)

## 📋 Project Overview
This repository contains the infrastructure-as-code (IaC) templates, configuration files, and deployment blueprints for an enterprise-grade, automated log aggregation and retention vault. 

Engineered specifically to satisfy strict financial sector mandates, this architecture guarantees **WORM (Write Once, Read Many)** data integrity. It provides a tamper-proof repository for systemic logs, audit trails, and transactional records, ensuring that once data is ingested, it can neither be modified, overwritten, nor deleted by any user—including administrative and root accounts—for the duration of the mandated retention period.

---

## 🏗️ System Architecture & Data Lifecycle

The architecture leverages native AWS cloud storage controls and cryptographic verification to build a secure pipeline from log generation to immutable archiving:

1. **Ingestion Layer (Stream & Forward):** System logs from distributed applications, on-premises infrastructure, and cloud environments are securely streamed via TLS-encrypted delivery streams (e.g., Amazon Kinesis Data Firehose or CloudWatch Logs) into a centralized landing zone.
2. **Immutable Storage Tier (S3 WORM Vault):** Data lands in a dedicated Amazon S3 bucket configured with **S3 Object Lock in Compliance Mode**. Legal hold capability is enabled by default. This tier prevents any data mutation or premature deletion, ensuring absolute compliance with SEC Rule 17a-4.
3. **Long-Term Glacier Archiving:** Automated lifecycle policies systematically transition older logs into Amazon S3 Glacier Flexible or Deep Archive. The strict Object Lock parameters are retained during migration, maintaining immutable characteristics at a highly optimized cost structure.

---

## 🔒 Threat Modeling & Risk Mitigation

| Threat Vector | Architectural Mitigation Strategy |
| :--- | :--- |
| **Ransomware / Malicious Deletion** | Enforced S3 Object Lock in **Compliance Mode**. Not even the AWS root account or administrator roles can override or bypass the retention period. |
| **Data Alteration & Tampering** | Continuous cryptographic hashing (SHA-256) and S3 Inventory verification to ensure log file integrity remains uncompromised. |
| **Unauthorized Internal Access** | Strict Identity and Access Management (IAM) policies coupled with an explicit S3 Bucket Policy enforcing the principle of least privilege. |
| **Eavesdropping / Interception** | Mandatory transport layer security (HTTPS/TLS 1.3) required for all API calls and data ingestion paths, blocking unencrypted traffic. |
| **Credential Compromise** | Storage encryption enforced at-rest using customer-managed AWS KMS keys (SSE-KMS) with dual-layer envelope encryption and asymmetric key rotation. |

---

## 💻 Core Infrastructure Policy (`worm-vault-policy.json`)

This JSON definition outlines the declarative IAM and Bucket Policy configuration that enforces secure transport and blocks any attempts to delete or alter objects within the compliance vault:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "EnforceSSLRequestsOnly",
      "Effect": "Deny",
      "Principal": "*",
      "Action": "s3:*",
      "Resource": [
        "arn:aws:s3:::regulatory-compliance-log-vault",
        "arn:aws:s3:::regulatory-compliance-log-vault/*"
      ],
      "Condition": {
        "Bool": {
          "aws:SecureTransport": "false"
        }
      }
    },
    {
      "Sid": "DenyBypassOfComplianceRetention",
      "Effect": "Deny",
      "Principal": "*",
      "Action": [
        "s3:BypassGovernanceMode",
        "s3:DeleteObject",
        "s3:DeleteObjectVersion"
      ],
      "Resource": [
        "arn:aws:s3:::regulatory-compliance-log-vault/*"
      ]
    },
    {
      "Sid": "EnforceKMSKeyEncryption",
      "Effect": "Deny",
      "Principal": "*",
      "Action": "s3:PutObject",
      "Resource": "arn:aws:s3:::regulatory-compliance-log-vault/*",
      "Condition": {
        "StringNotEquals": {
          "s3:x-amz-server-side-encryption": "aws:kms"
        }
      }
    }
  ]
}
