# 📊 Grafana with Azure AD SSO on EKS (via kube-prometheus-stack)

This guide explains how to deploy Grafana using the `kube-prometheus-stack` Helm chart on Amazon EKS with:

- **Azure AD (Office 365) Single Sign-On (SSO)**
- **Role-based access control (RBAC)** for `Admin`, `Editor`, and `Viewer`
- **Multiple data sources configured via `values.yaml`**

---

## 🚀 Prerequisites

- EKS Cluster with Ingress configured
- `helm`, `kubectl`, `aws` CLI installed
- HTTPS DNS name for Grafana
- Azure AD Admin access

---

## 🔐 Step 1: Register Grafana in Azure AD

1. Go to **Azure Portal → Azure Active Directory → App registrations → New registration**
2. Fill out:
   - **Name**: `Grafana`
   - **Redirect URI**: `https://<your-grafana-domain>/login/azuread`
3. Click **Register**

### Collect These:
- **Tenant ID**
- **Client ID**
- **Client Secret** (under *Certificates & secrets*)

### Add API Permissions:
- Under **API permissions** → Add Microsoft Graph → Delegated permissions:
  - `openid`, `profile`, `email`, `User.Read`
- Click **Grant admin consent**

---

## 👥 Step 2: Create Azure AD Groups for RBAC

1. Go to **Azure Active Directory → Groups → New Group**
2. Create two groups:
   - `Grafana Admins`
   - `Grafana Editors`

3. Add appropriate users to each group.
4. For each group, copy the **Object ID** (you’ll use it for mapping roles in Grafana).

---

## 🧾 Step 3: Add Group Claims to Tokens

1. Go to your Azure AD App → **Token Configuration**
2. Click **+ Add Group Claim**
3. Choose:
   - ✅ Security groups
   - ✅ ID Token
   - ✅ Group Object ID
4. Save

---

## 🔐 Step 4: Create Kubernetes Secret for Grafana OAuth

```bash
kubectl create secret generic azuread-secret \
  --from-literal=GF_AUTH_AZUREAD_CLIENT_ID=<CLIENT_ID> \
  --from-literal=GF_AUTH_AZUREAD_CLIENT_SECRET=<CLIENT_SECRET> \
  -n monitoring
