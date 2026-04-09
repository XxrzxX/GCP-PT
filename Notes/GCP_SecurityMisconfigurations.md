## 🔴 Security Misconfigurations

### High-Risk Scenarios

| # | Misconfiguration | Risk | Check |
|---|-----------------|------|-------|
| 1 | **Public Cloud Storage Buckets** | Data breach / Exfiltration | Look for `allUsers` via `gcloud storage buckets get-iam-policy` |
| 2 | **Default Service Account on Compute Engine** | Broad project compromise | Check if VMs have the default SA (often possesses `Editor` role) |
| 3 | **Broad Use of `allAuthenticatedUsers`** | Access to ANY Google account globally | Review IAM policies (`gcloud projects get-iam-policy`) |
| 4 | **Service Account Keys in Code/Storage** | Long-term credential theft | Scan environments/buckets for `*.json` SA keys |
| 5 | **Use of Legacy Basic Roles** | Overprivilege (`Owner/Editor/Viewer`) | Review IAM bindings for broad legacy roles |
| 6 | **Unrestricted Impersonation Permissions** | Privilege Escalation | Check who holds `roles/iam.serviceAccountTokenCreator` / `iam.serviceAccounts.actAs` |
| 7 | **Disabled OS Login (Metadata SSH Keys)** | Easy lateral movement | Check Compute Metadata if `enable-oslogin=TRUE` is missing |
| 8 | **No Organization Policies / SCPs limits** | Cross-project hop / Lack of guardrails | Review Org level restrictions |
| 9 | **Unauthenticated Cloud Functions/Run** | Unauthorized code execution | Check serverless IAM for `allUsers` |
| 10 | **Exposed Web Apps (SSRF)** | Metadata token extraction | Hit `http://[IP_ADDRESS]` with `Metadata-Flavor: Google` |

---

### Red Team Targets (Priority Order in GCP)

```
🥇 Organization Admin        - Highest level organizational control
🥈 Project Owner/Editor      - Full control over resources inside a project
🥉 Service Account Admins    - Can create keys or impersonate other Service Accounts
4️⃣  Default Service Accounts  - Often assigned automatically with broad project scopes
5️⃣  Storage Admin             - Access to sensitive buckets, Terraform states, DB dumps
6️⃣  CI/CD Pipelines           - JSON SA Keys stored insecurely as secrets or env vars
7️⃣  Compute Instances (VMs)   - SSRF entry points to hit the Metadata Server
```

### Critical Attack Checks

```bash
# Check for Overly Permissive Project Bindings 
gcloud projects get-iam-policy <ProjectID> --flatten="bindings[].members"

# Search for any Service Account impersonation roles
gcloud projects get-iam-policy <ProjectID> --filter="bindings.role:roles/iam.serviceAccountTokenCreator"

# Check Compute Engines for attached Service Accounts & their emails
gcloud compute instances list --format="table(name, zone, serviceAccounts[].email)"

# Find out what resources a compromised Service Account can access (Asset Inventory needed)
gcloud asset search-all-iam-policies --scope=projects/<ProjectID> --query="policy:<Service_Account_Email>"
```
