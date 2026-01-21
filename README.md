# AWS IAM Identity Center Integration with Microsoft Entra ID

[![AWS](https://img.shields.io/badge/AWS-IAM%20Identity%20Center-FF9900?style=flat&logo=amazon-aws)](https://aws.amazon.com/iam/identity-center/)
[![Azure](https://img.shields.io/badge/Microsoft-Entra%20ID-0078D4?style=flat&logo=microsoft-azure)](https://www.microsoft.com/en-us/security/business/identity-access/microsoft-entra-id)
[![License](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE)

A production-grade enterprise identity integration between **Microsoft Entra ID** and **AWS IAM Identity Center**, implementing comprehensive IAM controls across a multi-account AWS Organization.

## 🎯 Project Overview

This project demonstrates enterprise-grade Identity and Access Management (IAM) implementation, including:

- **SAML 2.0 Federation** — Single sign-on from Entra ID to AWS
- **SCIM Provisioning** — Automated user and group synchronization
- **Lifecycle Workflows** — Automated Joiner/Mover/Leaver processes
- **Access Reviews** — Quarterly certification with auto-revocation
- **Privileged Identity Management (PIM)** — Just-in-time admin access
- **Attribute-Based Access Control (ABAC)** — Dynamic authorization based on user attributes
- **Service Control Policies (SCPs)** — Organization-wide security guardrails
- **CLI Access** — Secure programmatic access with temporary credentials
- **CloudTrail Auditing** — Comprehensive API logging and monitoring

## 🏗️ Architecture

### AWS Organization Structure

```
Root
├── Management Account
│   └── IAM Identity Center, Break-Glass Access
├── DEV OU
│   └── Dev Account (Development workloads)
├── PROD OU
│   └── Prod Account (Production workloads)
├── Sandbox OU
│   └── Sandbox Account (Testing)
└── Security OU
    ├── Audit (Audit logging)
    └── Log Archive (Centralized logs)
```

### Identity Flow

```
┌─────────────────┐      SAML 2.0       ┌─────────────────────┐
│                 │ ←─────────────────→ │                     │
│  Microsoft      │                     │  AWS IAM            │
│  Entra ID       │      SCIM 2.0       │  Identity Center    │
│                 │ ───────────────────→│                     │
└─────────────────┘                     └──────────┬──────────┘
        │                                          │
        │                                          ▼
        │                               ┌─────────────────────┐
        │                               │   AWS Accounts      │
        │                               │  ┌───┐ ┌───┐ ┌───┐  │
        └── Session Tags (ABAC) ──────→│  │Dev│ │Prd│ │Sbx│  │
                                        │  └───┘ └───┘ └───┘  │
                                        └─────────────────────┘
```

### Defense in Depth Model

| Layer | Control | Purpose |
|-------|---------|---------|
| 1 | Service Control Policies | Hard ceiling — what NOBODY can do |
| 2 | Permission Sets | What groups CAN do |
| 3 | ABAC Policies | Dynamic restrictions based on attributes |
| 4 | PIM | Just-in-time elevation with approval |
| 5 | Access Reviews | Periodic certification |
| 6 | CloudTrail | Complete audit trail |

## 🔐 Security Controls Implemented

### Service Control Policies (SCPs)

| SCP | Purpose | Attached To |
|-----|---------|-------------|
| [`RestrictToUSEast1`](policies/scps/RestrictToUSEast1.json) | Block services outside us-east-1 | DEV, PROD, Sandbox OUs |
| [`DenyRootUser`](policies/scps/DenyRootUser.json) | Block root user actions | All member account OUs |
| [`DenyLeaveOrganization`](policies/scps/DenyLeaveOrganization.json) | Prevent account escape | Root (all accounts) |

### Permission Sets

| Permission Set | Policy | Use Case |
|---------------|--------|----------|
| `SalesReadOnly` | ViewOnlyAccess | Sales team read-only to Prod |
| `EngineeringPower` | PowerUserAccess | Engineering team Dev/Sandbox |
| `EmergencyAdmin` | AdministratorAccess | PIM-controlled emergency access |
| [`DepartmentRestrictedEC2`](policies/permission-sets/DepartmentRestrictedEC2.json) | Custom ABAC | Department-based EC2 access |

### ABAC Policy Example

Users can only manage EC2 instances tagged with their own department:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "AllowEC2WhenDepartmentMatches",
            "Effect": "Allow",
            "Action": ["ec2:StartInstances", "ec2:StopInstances", "ec2:RebootInstances"],
            "Resource": "*",
            "Condition": {
                "StringEquals": {
                    "ec2:ResourceTag/Department": "${aws:PrincipalTag/Department}"
                }
            }
        }
    ]
}
```

## 🔄 Lifecycle Automation

### Joiner Workflow
- **Trigger:** `employeeHireDate`
- **Action:** Add to department group → SCIM syncs → AWS access granted
- **Result:** Day-one access without manual intervention

### Mover Workflow
- **Trigger:** Department attribute change
- **Action:** Remove from old group, add to new group
- **Result:** Access automatically adjusts to new role

### Leaver Workflow
- **Trigger:** `employeeLeaveDateTime`
- **Action:** Remove from all groups, disable account
- **Result:** All access revoked immediately

## ⚡ Just-in-Time Access (PIM)

```
┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐
│  User    │───→│  Request │───→│  Approve │───→│  Active  │───→│  Expire  │
│ Eligible │    │ + Reason │    │ (Manager)│    │ (4 hours)│    │ (Auto)   │
└──────────┘    └──────────┘    └──────────┘    └──────────┘    └──────────┘
```

- No standing admin privileges
- Requires justification and approval
- Automatic expiration after 4 hours
- Full audit trail of all activations

## 🖥️ CLI Access

Secure programmatic access without long-term credentials:

```bash
# AWS CLI config (~/.aws/config)
[profile dev-engineer]
sso_start_url = https://d-xxxxxxxxxx.awsapps.com/start
sso_region = us-east-1
sso_account_id = 123456789012
sso_role_name = EngineeringPower
region = us-east-1

# Authenticate
aws sso login --profile dev-engineer

# Verify identity
aws sts get-caller-identity --profile dev-engineer
```

See [`cli/aws-config-example`](cli/aws-config-example) for full configuration template.

## 📊 Audit & Compliance

### CloudTrail Configuration
- Organization-wide trail (all accounts)
- Multi-region logging
- CloudWatch Logs integration for real-time queries

### Sample CloudWatch Logs Insights Query

```sql
fields @timestamp, eventName, userIdentity.onBehalfOf.userId, 
    sourceIPAddress, serviceEventDetails.account_id, serviceEventDetails.role_name
| filter eventSource = "sso.amazonaws.com"
| filter eventName = "Federate"
| sort @timestamp desc
| limit 50
```

## 📁 Repository Structure

```
├── README.md
├── LICENSE
├── docs/
│   ├── AWS-SSO-Integration-Guide-Final.docx
│   └── AWS-SSO-Integration-Runbooks-Final.docx
├── diagrams/
│   └── AWS-SSO-Architecture-Diagrams.html
├── policies/
│   ├── scps/
│   │   ├── RestrictToUSEast1.json
│   │   ├── DenyRootUser.json
│   │   └── DenyLeaveOrganization.json
│   └── permission-sets/
│       └── DepartmentRestrictedEC2.json
└── cli/
    └── aws-config-example
```

## 🚀 Key Achievements

- ✅ Eliminated standing privileged access with PIM
- ✅ Automated user lifecycle (Joiner/Mover/Leaver)
- ✅ Implemented ABAC for dynamic authorization
- ✅ Enforced regional restrictions via SCPs
- ✅ Configured quarterly access reviews with auto-revocation
- ✅ Established break-glass procedure for emergencies
- ✅ Full audit trail with CloudTrail and CloudWatch

## 📈 Business Impact

| Metric | Before | After |
|--------|--------|-------|
| Onboarding Time | 2-3 days | < 1 hour (automated) |
| Offboarding Time | 1-2 days | Immediate (automated) |
| Standing Admin Accounts | Multiple | Zero (PIM) |
| Access Review Frequency | Ad-hoc | Quarterly (automated) |
| Credential Exposure Risk | High (long-term keys) | Minimal (1-hour temp creds) |

## 🛠️ Technologies Used

- **Identity Provider:** Microsoft Entra ID (Azure AD)
- **AWS Services:** IAM Identity Center, Organizations, Control Tower, CloudTrail, CloudWatch
- **Protocols:** SAML 2.0, SCIM 2.0, OAuth 2.0
- **Governance:** Entra ID Governance, PIM, Access Reviews

## 📚 Documentation

| Document | Description |
|----------|-------------|
| [Implementation Guide](docs/AWS-SSO-Integration-Guide-Final.docx) | Complete technical documentation (25+ pages) |
| [Operational Runbooks](docs/AWS-SSO-Integration-Runbooks-Final.docx) | 24 step-by-step procedures |
| [Architecture Diagrams](https://htmlpreview.github.io/?https://github.com/arashfariv/aws-sso-entra-integration/blob/main/diagrams/AWS-SSO-Architecture-Diagrams.html) | 11 visual diagrams (click to view) |

> **Note:** Download the `.docx` files to view in Microsoft Word or Google Docs.

## 👤 Author

**Arash Farivar**  
IAM & DevSecOps Engineer

[![LinkedIn](https://img.shields.io/badge/LinkedIn-Connect-0A66C2?style=flat&logo=linkedin)](https://www.linkedin.com/in/arashfarivar)

## 📄 License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

---

*This project was built as a portfolio demonstration of enterprise IAM capabilities. All sensitive values have been sanitized.*
