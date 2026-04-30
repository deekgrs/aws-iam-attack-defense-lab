# 🔐 AWS IAM Attack & Defense Lab

> A hands-on cloud security simulation demonstrating IAM privilege escalation, persistence attacks, and least-privilege defense — built on AWS CloudShell.

---

## 📋 Overview

This lab simulates a realistic AWS IAM attack-defense scenario. Starting from a low-privileged IAM user, the lab walks through how an attacker can enumerate the IAM environment, establish persistence via access keys, and escalate privileges to full admin access — then shows how to mitigate every step using least-privilege policies and explicit deny rules.

All attacks and defenses were executed live in AWS CloudShell using the AWS CLI.

---

## 🎯 Objectives

- Simulate IAM-based attack scenarios in a real AWS environment
- Demonstrate privilege escalation via misconfigured IAM permissions
- Show how attackers maintain persistence using multiple access keys
- Implement and validate least-privilege security controls
- Prove mitigation effectiveness through repeated real attack attempts

---

## 🛠️ Environment

| Component | Details |
|-----------|---------|
| **Platform** | AWS CloudShell |
| **Services** | AWS IAM, AWS STS |
| **CLI** | AWS CLI v2 |
| **Resources** | IAM Users, Managed Policies, Inline Policies, Access Keys |

---

## ⚔️ Attack Phase

### 1. Insecure IAM User Creation
Created a low-privileged IAM user `lab-insecure-user` to simulate an attacker's initial foothold. An access key was generated immediately to enable programmatic CLI access.

### 2. Over-Permissive Policy (LabInsecurePolicy)
A custom managed policy was created with excessive IAM permissions:
- `iam:List*` / `iam:Get*` — full enumeration
- `iam:CreateAccessKey` — credential creation
- `iam:PutUserPolicy` — inline policy injection

### 3. Policy Attachment
`LabInsecurePolicy` was attached to `lab-insecure-user`, enabling all attack capabilities.

### 4. Access Key Generation
A second access key was created to simulate credential theft and persistence setup.

### 5. IAM Enumeration
Using attacker credentials, the full IAM environment was mapped:
```bash
aws sts get-caller-identity --profile lab-insecure
aws iam list-users --profile lab-insecure
```
**Result:** Attacker gained visibility into all 5 IAM users in the account.

### 6. Persistence via Access Keys
Two active access keys were maintained simultaneously, ensuring continued access even if one key is revoked. The AWS limit of 2 keys per user was reached, confirming full slot occupancy.

```bash
aws iam list-access-keys --user-name lab-insecure-user
# Output: 2 Active keys
```

### 7. 🔥 Privilege Escalation (Critical)
The attacker exploited `iam:PutUserPolicy` to inject an inline policy granting full `AdministratorAccess`:

```bash
aws iam put-user-policy \
  --user-name lab-insecure-user \
  --policy-name escalate \
  --policy-document file://escalate.json \
  --profile lab-insecure
```

**Result:** A low-privileged user achieved full admin control of the AWS account.

---

## 🛡️ Defense Phase

### 1. Remove Malicious Inline Policy
```bash
aws iam delete-user-policy \
  --user-name lab-insecure-user \
  --policy-name escalate
# Verified: PolicyNames: []
```

### 2. Detach Insecure Managed Policy
```bash
aws iam detach-user-policy \
  --user-name lab-insecure-user \
  --policy-arn arn:aws:iam::ACCOUNT_ID:policy/LabInsecurePolicy
# Verified: AttachedPolicies: []
```

### 3. Implement Least-Privilege Policy (LabSecurePolicy)
A minimal policy was created with:
- Read-only permissions scoped to required resources only
- **Explicit Deny** on all sensitive IAM actions (`iam:CreateAccessKey`, `iam:PutUserPolicy`, etc.)

```bash
aws iam create-policy \
  --policy-name LabSecurePolicy \
  --policy-document file://secure-policy.json

aws iam attach-user-policy \
  --user-name lab-insecure-user \
  --policy-arn arn:aws:iam::ACCOUNT_ID:policy/LabSecurePolicy
```

### 4. ✅ Validation
All previous attack commands were re-run under the new policy:

```bash
aws iam create-access-key --profile lab-insecure
# → An error occurred (AccessDenied): ...explicit deny in identity-based policy

aws iam put-user-policy --user-name lab-insecure-user ...
# → An error occurred (AccessDenied): ...explicit deny in identity-based policy
```

**Both attack vectors fully blocked.**

---

## 📊 Key Findings

| Finding | Impact |
|---------|--------|
| Over-permissive IAM policies | Enable privilege escalation by any low-privileged user |
| Multiple active access keys | Attackers maintain persistence even after partial remediation |
| `iam:PutUserPolicy` exposure | Single permission leads to full account compromise |
| Explicit Deny rules | Override all Allow statements — critical for hard security boundaries |
| Least-privilege model | Completely mitigates all demonstrated attack vectors |

---

## 💡 Key Takeaways

- **Never grant `iam:PutUserPolicy` or `iam:AttachUserPolicy`** to non-admin users — these are effectively admin-equivalent permissions.
- **Explicit Deny is your safety net.** Unlike Allow, Deny cannot be overridden by any other policy in the account.
- **Access key hygiene matters.** Enforce key rotation and monitor for unusual creation events via CloudTrail.
- **Enumerate like an attacker.** Run `aws iam list-users` and `aws iam simulate-principal-policy` regularly to audit your own posture.

---

## 📁 Project Structure

```
aws-iam-attack-defense-lab/
│
├── escalate.json          # Inline policy used in privilege escalation attack
├── secure-policy.json     # Least-privilege policy with explicit deny rules
└── README.md
```

---

## 🔒 Security Note

> All AWS Access Key IDs and Secret Access Keys shown in screenshots have been **redacted** (`****REDACTED****`). The AWS account used for this lab has been cleaned up — all insecure policies detached and deleted, and all lab users deprovisioned after completion.

---

## 💼 Resume Bullet

> **AWS IAM Attack & Defense Lab** — Built a cloud security lab to simulate IAM privilege escalation and persistence attacks by exploiting misconfigured policies; implemented least-privilege controls with explicit deny rules and validated mitigation using real attack attempts.

---

## 🔗 Related Concepts

- [AWS IAM Best Practices](https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html)
- [AWS IAM Policy Evaluation Logic](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_evaluation-logic.html)
- [Privilege Escalation in AWS (Rhino Security Labs)](https://rhinosecuritylabs.com/aws/aws-privilege-escalation-methods-mitigation/)
- [AWS CloudTrail for IAM Monitoring](https://docs.aws.amazon.com/awscloudtrail/latest/userguide/cloudtrail-user-guide.html)
