# Red Hat Service Interconnect (Skupper v2) + EDB / Cloud Native PostgreSQL
**GitOps + RHACM + Argo CD for two OpenShift 4.19 clusters (ap-northeast-1 + ap-northeast-2)**

This repo shows how to:
1. Install RHSI (Skupper) operator on both clusters
2. Install Cloud Native PostgreSQL / EDB operator on both clusters
3. Create a Skupper **Site** on site-a (Tokyo) and site-b (Seoul)
4. Export Postgres from site-a via a Skupper **Connector**
5. Import it on site-b via a Skupper **Listener** so pods on site-b can reach it as a normal ClusterIP
6. Drive everything via Argo CD on the ACM hub

### Repo layout
- `cluster-config/` → console plugin + console patch for RHSI UI
- `operators/` → two operator subscriptions (RHSI + CNPG/EDB)
- `site-a/` → site, router access, access grant, EDB primary, connector
- `site-b/` → site, link, listener, optional EDB
- `argo-examples/` → 4 Argo Applications pointing to ACM-imported clusters

### Assumptions
- You imported two clusters into ACM → Argo shows these secrets in `openshift-gitops`:
  - `site-a-application-manager-cluster-secret`
  - `site-b-application-manager-cluster-secret`
- You will push this repo to **https://github.com/waynedovey/service-interconnect-edb-demo** and use that URL in Argo.

### Install order (important)
1. Apply `argo-examples/si-operators-site-a.yaml` (operators on Tokyo)
2. Apply `argo-examples/si-operators-site-b.yaml` (operators on Seoul)
3. Apply `argo-examples/si-site-a.yaml` (Skupper Site + EDB + connector on Tokyo)
4. Apply `argo-examples/si-site-b.yaml` (Skupper Site + Link + Listener on Seoul)
5. Replace placeholders in `site-b/access-token.yaml` and `site-b/link-to-site-a.yaml` with the real token + endpoint from site-a

### Manual bit
- On site-a, grab the Skupper access token (via UI or CLI)
- On site-b, replace the `REPLACE_ME` values in the Secret and the `REPLACE_WITH_SITE_A_ROUTER_HOST` in the Link
- Re-sync Argo → link becomes Connected

### Test
```bash
oc -n interconnect-demo run psql --image=postgres:16 -- sleep 3600
oc -n interconnect-demo exec -it deploy/psql -- psql -h site-a-db-rw -p 5432 -U appuser
```
Set `PGPASSWORD=AppUserP@ssw0rd` if you used the demo secret.
