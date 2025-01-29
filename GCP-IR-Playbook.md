# 🚨 GCP IAM Security Incident Response Playbook

## 📌 Scope
This playbook is designed to respond to IAM security incidents involving leaked credentials, unauthorized access, or compromised service accounts. It ensures that access is revoked swiftly, affected accounts are locked down, and future attacks are mitigated.

---

## 🛑 1. Incident Detection & Initial Triage
### 🎯 Goal: Confirm the incident, assess its severity, and initiate response procedures.

### 🔹 Step 1: Detect the Incident
✅ **Security Alerts:**
- **Google Cloud Security Command Center (SCC)** alerts on leaked credentials.
- **IAM Activity Logs** show abnormal API calls.
- **GitHub Secret Scanners** detect hardcoded keys.
- **User Reports** about unauthorized access.

✅ **Confirm Severity:**
Run the following to check recent IAM key activity:
```bash
gcloud logging read \
'protoPayload.serviceName="iam.googleapis.com" AND protoPayload.methodName="google.iam.admin.v1.ServiceAccountKey.create"'
```
🔍 **Check if keys were used:**
```bash
gcloud logging read \
'protoPayload.serviceName="iam.googleapis.com" AND protoPayload.methodName="google.iam.admin.v1.ServiceAccountKey.use"'
```

✅ **Create an Incident Ticket**
- **Title:** “Leaked IAM Credentials - Urgent Action Required”
- **Severity Level:** High
- **Teams Notified:** SOC, IAM Admins, DevSecOps

---

## 🔐 2. Immediate Containment Actions
### 🎯 Goal: Prevent further access using the compromised credentials.

### 🔹 Step 2: Revoke Active Sessions
✅ **Revoke IAM User Sessions:**
```bash
gcloud auth revoke --all
```
✅ **Disable OAuth & Cloud API Access:**
```bash
gcloud organizations policies set-policy enforce-reauthentication.yaml
```
✅ **Force Logout for Google Workspace Users:**
1. **Go to Google Admin Console → Security → User Accounts**  
2. **Select "Force Logout" for all users**

---

## 🔹 Step 3: Disable Compromised Service Accounts
✅ **Identify All Active Keys**
```bash
gcloud iam service-accounts keys list --iam-account=<service-account-email>
```
✅ **Revoke & Delete Keys**
```bash
gcloud iam service-accounts keys delete <KEY_ID> --iam-account=<service-account-email>
```
✅ **Disable Service Account Temporarily**
```bash
gcloud iam service-accounts disable <service-account-email>
```

---

## 🔹 Step 4: Block Suspicious IPs & API Calls
✅ **Identify Suspicious IPs Accessing GCP APIs**
```bash
gcloud logging read \
'protoPayload.authenticationInfo.principalEmail:<COMPROMISED-EMAIL>'
```
✅ **Block Identified IPs Using Cloud Armor**
```bash
gcloud compute security-policies rules create 1000 \
  --action "deny(403)" \
  --rule "srcIpRanges=<SUSPICIOUS-IP>" \
  --security-policy <SECURITY_POLICY_NAME>
```

---

## 🕵️‍♂️ 3. Investigation & Forensics
### 🎯 Goal: Determine the attack vector, impacted resources, and potential lateral movement.

### 🔹 Step 5: Review IAM Audit Logs
✅ **Identify Who Used the Leaked Credentials**
```bash
gcloud logging read \
'protoPayload.authenticationInfo.principalEmail="<COMPROMISED-EMAIL>"'
```
✅ **List API Calls Made by the Attacker**
```bash
gcloud logging read \
'protoPayload.methodName:* AND protoPayload.authenticationInfo.principalEmail="<COMPROMISED-EMAIL>"'
```
✅ **Investigate If Data Was Exfiltrated**
```bash
gcloud logging read \
'protoPayload.serviceName="storage.googleapis.com" AND protoPayload.methodName="storage.objects.get"'
```

---

## 🛠️ 4. Remediation & Recovery
### 🎯 Goal: Restore secure access, rotate credentials, and patch security gaps.

### 🔹 Step 6: Rotate Credentials
✅ **Rotate All IAM Keys & OAuth Tokens**
```bash
gcloud iam service-accounts keys create new-key.json \
    --iam-account=<service-account-email>
```
✅ **Update All Workloads Using New Credentials**
- Deploy new service account credentials to Kubernetes, VMs, and CI/CD pipelines.

---

### 🔹 Step 7: Implement Preventative Measures
✅ **Enforce IAM Key Rotation Policy**
```terraform
resource "google_organization_policy" "enforce_key_rotation" {
  org_id     = "YOUR_ORG_ID"
  constraint = "iam.disableServiceAccountKeyCreation"
  boolean_policy {
    enforced = true
  }
}
```
✅ **Enable Organization Policy to Block IAM Key Creation**
```terraform
resource "google_organization_policy" "block_iam_key_creation" {
  org_id     = "YOUR_ORG_ID"
  constraint = "iam.disableServiceAccountKeyCreation"
  boolean_policy {
    enforced = true
  }
}
```
✅ **Apply Workload Identity Federation** (Prevent hardcoded keys)
```terraform
resource "google_iam_workload_identity_pool" "pool" {
  provider = google
  project  = "YOUR_PROJECT_ID"
  workload_identity_pool_id = "example-pool"
}
```

---

## 📜 5. Documentation & Post-Incident Analysis
### 🎯 Goal: Learn from the incident, update policies, and prevent future occurrences.

### 🔹 Step 8: Document the Incident
✅ **Incident Report**
- **What happened?**
- **What was affected?**
- **How was it detected?**
- **How was it mitigated?**

✅ **Root Cause Analysis (RCA)**
- **Why did the incident happen?**
- **What security gaps allowed this?**
- **What preventive measures are needed?**

✅ **Security Policy Updates**
- **IAM best practices**
- **New monitoring rules**
- **Team training on secret management**

---
