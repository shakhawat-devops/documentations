
# Grafana with Azure AD SSO on EKS (via kube-prometheus-stack)

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
```

---

## ⚙️ Step 5: Configure `values.yaml`

### Grafana SSO and Role Mapping

```yaml
grafana:
  enabled: true

  envFromSecret: azuread-secret

  grafana.ini:
    server:
      root_url: https://<your-grafana-domain>

    auth.azuread:
      enabled: true
      allow_sign_up: true
      scopes: openid email profile
      auth_url: https://login.microsoftonline.com/<TENANT_ID>/oauth2/v2.0/authorize
      token_url: https://login.microsoftonline.com/<TENANT_ID>/oauth2/v2.0/token
      allowed_domains: yourdomain.com
      use_pkce: true
      role_attribute_path: >
        contains(groups, 'xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx') && 'Editor' ||
        contains(groups, 'yyyyyyyy-yyyy-yyyy-yyyy-yyyyyyyyyyyy') && 'Admin' ||
        'Viewer'
```

Replace:
- `<your-grafana-domain>`: your actual domain name
- `<TENANT_ID>`: your Azure AD tenant
- `xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx`: Object ID of `Grafana Editors`
- `yyyyyyyy-yyyy-yyyy-yyyy-yyyyyyyyyyyy`: Object ID of `Grafana Admins`

---

## 🔌 Step 6: Add Multiple Data Sources if you want to push data from differenet prometheus/loki servers (Optional)

```yaml
grafana:
  datasources:
    datasources.yaml:
      apiVersion: 1
      datasources:
        - name: Loki
          type: loki
          access: proxy
          url: http://loki:3100
          isDefault: true

        - name: Prometheus
          type: prometheus
          access: proxy
          url: http://prometheus-kube-prometheus-prometheus.monitoring.svc:9090

        - name: Tempo
          type: tempo
          access: proxy
          url: http://tempo.monitoring.svc:3100
```

---

## 📦 Step 7: Deploy the Stack

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

helm upgrade --install kube-prometheus-stack \
  prometheus-community/kube-prometheus-stack \
  -n monitoring --create-namespace \
  -f values.yaml
```

---

## 🔍 Step 8: Test Access

1. Open `https://<your-grafana-domain>`
2. Login via Microsoft
3. Based on group membership:
   - `Grafana Admins`: Full access
   - `Grafana Editors`: Can use Explore tab
   - Others: Viewer-only

---

## 🧭 Diagrams

### Azure AD SSO Login Flow
![Azure AD SSO Flow](./azure_ad_sso_flow.png)

### Grafana RBAC Role Mapping from Azure AD Groups
![Grafana RBAC Mapping](./grafana_rbac_mapping.png)

---

## 🧪 Troubleshooting

| Problem                         | Solution                                                                 |
|----------------------------------|--------------------------------------------------------------------------|
| No Explore tab                  | Ensure user is in the "Grafana Editors" group                            |
| Role mapping doesn't work       | Check `role_attribute_path`, ensure group Object IDs are correct         |
| Group claim not in token        | Ensure Group Claims were added under Token Configuration in Azure AD     |
| OAuth Login fails               | Check `client_id`, `client_secret`, `redirect_uri`, tenant consistency   |

---

## 📁 Resources

- [Grafana AzureAD OAuth](https://grafana.com/docs/grafana/latest/setup-grafana/configure-security/configure-authentication/azuread/)
- [Helm kube-prometheus-stack](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack)
- [Azure App Registration](https://portal.azure.com/#blade/Microsoft_AAD_IAM/ActiveDirectoryMenuBlade/RegisteredApps)

