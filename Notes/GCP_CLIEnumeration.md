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