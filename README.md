# Red Hat Service Interconnect (Skupper v2) + EDB / Cloud Native PostgreSQL + Vault (External Secrets)

**Scenario:**
You have **two OpenShift 4.19 clusters**:

- **site-a** → AP Northeast 1 (Tokyo)
- **site-b** → AP Northeast 2 (Seoul)

You manage them with **RHACM** + **OpenShift GitOps (Argo CD)** and you want to:

1. Install **Red Hat Service Interconnect** (RHSI / Skupper) on both clusters using the **operator** (not the CLI).
2. Install **EDB / Cloud Native PostgreSQL** operator on both clusters.
3. Create Skupper **Sites** on both clusters using **CRDs** (`apiVersion: skupper.io/v2alpha1`) so this is **GitOpsable**.
4. Export a Postgres service from **site-a** to **site-b** using Skupper.
5. Keep **all sensitive secrets** (DB user/pass, Skupper link TLS certs) in **Vault** and let the **External Secrets Operator** (ESO) materialize them as Kubernetes secrets.
6. Drive **all of the above** from Git via Argo on the ACM hub.

This README explains how the repo is structured, what to apply, and what you must fill in.

---

## 1. What you get

- Skupper running on **both** clusters, defined by **CRDs** (`Site`, `Link`, `Connector`, `Listener`).
- EDB / CNPG running on **both** clusters.
- A DB on **site-a** exported over Skupper.
- A **local** ClusterIP on **site-b** called `site-a-db-rw` that points back to **site-a** over Skupper.
- No hardcoded passwords or Skupper tokens in Git — they come from **Vault** via ESO.

---

## 2. Repo layout (reference)

```text
service-interconnect-edb-vault/
├── README.md                        # ← this file
├── kustomization.yaml               # top-level; pulls everything together
├── cluster-config/
│   ├── kustomization.yaml
│   └── console-plugin/
│       ├── console-plugin.yaml      # enable RHSI console plugin
│       └── console-operator-patch.yaml
├── operators/
│   ├── rhs-interconnect/            # skupper operator subscription (stable-2.1)
│   ├── edb-cnpg/                    # cloud-native-postgresql (stable-v1.27)
│   └── external-secrets/            # External Secrets Operator from community-operators
├── vault/
│   ├── kustomization.yaml
│   ├── site-a-secretstore.yaml      # ClusterSecretStore → Vault for site-a
│   └── site-b-secretstore.yaml      # ClusterSecretStore → Vault for site-b
├── site-a/
│   ├── kustomization.yaml
│   ├── namespaces.yaml              # interconnect-demo + edb-demo
│   ├── site.yaml                    # Skupper Site (Tokyo)
│   ├── router-access.yaml           # expose router via LB
│   ├── access-grant.yaml            # token issuer for the other site
│   ├── edb-cluster.yaml             # EDB cluster, no inline creds
│   ├── connector-postgres.yaml      # exports DB on routingKey=postgres-site-a
│   └── external-secret-db-user.yaml # pulls DB creds from Vault, makes app-user-secret
├── site-b/
│   ├── kustomization.yaml
│   ├── namespaces.yaml
│   ├── site.yaml                    # Skupper Site (Seoul)
│   ├── link-to-site-a.yaml          # Skupper Link CR → uses TLS secret from ESO
│   ├── listener-postgres.yaml       # local service on site-b → forwards to site-a
│   ├── edb-cluster.yaml             # optional local DB
│   └── external-secret-skupper-link.yaml  # pulls TLS certs/key from Vault
└── argo-examples/
    ├── si-operators-site-a.yaml     # Argo app → installs operators on site-a
    ├── si-operators-site-b.yaml     # Argo app → installs operators on site-b
    ├── si-site-a.yaml               # Argo app → applies site-a/ to site-a
    └── si-site-b.yaml               # Argo app → applies site-b/ to site-b
```

So you have **5 big buckets**: `cluster-config/`, `operators/`, `vault/`, `site-a/`, `site-b/`, plus `argo-examples/`.

---

## 3. Assumptions

- You already have **ACM** and **OpenShift GitOps** installed on the **hub**.
- Both clusters are imported and appear as Argo cluster secrets:
  ```bash
  oc -n openshift-gitops get secret -l argocd.argoproj.io/secret-type=cluster
  # ...
  site-a-application-manager-cluster-secret
  site-b-application-manager-cluster-secret
  ```
- You will push this repo (or a fork) to somewhere like:
  **https://github.com/waynedovey/service-interconnect-edb-demo**
- Clusters can reach **Vault** (or Vault is reachable via a route / corporate DNS / VPN).
- You’re ok to edit a couple of **placeholders**:
  - Vault URL
  - Vault auth role
  - Skupper router public host on site-a

---

## 4. Operators (what gets installed where)

We install **3** operators to **both** clusters:

1. **Skupper / Red Hat Service Interconnect**
   - `operators/rhs-interconnect/subscription.yaml`
   - channel: `stable-2.1`
   - namespace: `openshift-operators`

2. **EDB / Cloud Native PostgreSQL**
   - `operators/edb-cnpg/subscription.yaml`
   - channel: `stable-v1.27`
   - source: `certified-operators`
   - namespace: `openshift-operators`

3. **External Secrets Operator**
   - `operators/external-secrets/subscription.yaml`
   - channel: `stable`
   - source: `community-operators`
   - namespace: `openshift-operators`

We install operators **first**, because the Skupper `Site` and the ExternalSecret CRDs must exist **before** we apply `site-a/` and `site-b/`.

---

## 5. Install order (important)

This is the order to apply the Argo Applications on the **hub**:

1. **Operators on site-a**
   ```bash
   oc apply -f argo-examples/si-operators-site-a.yaml
   ```

2. **Operators on site-b**
   ```bash
   oc apply -f argo-examples/si-operators-site-b.yaml
   ```

3. **Site-a resources**
   ```bash
   oc apply -f argo-examples/si-site-a.yaml
   ```

4. **Site-b resources**
   ```bash
   oc apply -f argo-examples/si-site-b.yaml
   ```

In the Argo UI this just means: create these 4 “Applications”, then **Sync** the operator ones first.

If you do it out of order, Argo will complain with messages like:

> no matches for kind "Site" in version "skupper.io/v2alpha1"

because the CRDs aren’t there yet.

---

## 6. Vault integration (ESO → Vault → K8s Secret)

We added **Vault** so we don’t have to commit:

- DB username/password
- Skupper TLS secret (the link secret)

to Git.

Instead, we let **External Secrets Operator** read from Vault and create the corresponding Kubernetes Secret.

### 6.1 The expected Vault paths

We expect **two** Vault paths:

#### (a) DB user for site-a

- Path: `ocp/site-a/edb/app-user`
- Keys:
  - `username`
  - `password`

Example:

```bash
vault kv put ocp/site-a/edb/app-user   username=appuser   password=AppUserP@ssw0rd
```

The ExternalSecret that consumes this is in:
`site-a/external-secret-db-user.yaml`

It creates the secret in **namespace** `edb-demo` with the name **`app-user-secret`**.

That’s the secret your **EDB operator** can use.

---

#### (b) Skupper link secret for site-b

- Path: `ocp/site-b/skupper/link-to-site-a`
- Keys:
  - `ca.crt`
  - `tls.crt`
  - `tls.key`

Example:

```bash
vault kv put ocp/site-b/skupper/link-to-site-a   ca.crt="$(cat ca.crt)"   tls.crt="$(cat tls.crt)"   tls.key="$(cat tls.key)"
```

The ExternalSecret that consumes this is in:
`site-b/external-secret-skupper-link.yaml`

It creates a k8s secret called **`link-to-site-a`** in namespace **`interconnect-demo`**.

And the **Skupper Link CR** (`site-b/link-to-site-a.yaml`) references **that** secret:

```yaml
spec:
  tlsCredentials: link-to-site-a
```

So:
**Vault → ExternalSecret → Secret/link-to-site-a → Skupper Link**

---

### 6.2 SecretStores

We made **two** ClusterSecretStores in `vault/`:

- `vault/site-a-secretstore.yaml`
- `vault/site-b-secretstore.yaml`

You **must** update:

- `spec.provider.vault.server: https://REPLACE_WITH_VAULT_URL`
- `spec.provider.vault.auth.kubernetes.role: ocp-site-a` or `ocp-site-b`
- `spec.provider.vault.auth.kubernetes.mountPath:` to whatever you configured in Vault

Example for site-a:

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ClusterSecretStore
metadata:
  name: vault-site-a
spec:
  provider:
    vault:
      server: https://vault.mycorp.example
      path: ocp
      version: v2
      auth:
        kubernetes:
          mountPath: /v1/auth/kubernetes
          role: ocp-site-a
          serviceAccountRef:
            name: external-secrets-operator
            namespace: openshift-operators
```

Do the same for site-b.

---

## 7. Skupper resources (CRD-based, no CLI)

You said: **“I don’t want to use the skupper CLI; I want to do it via GitOps and CRDs.”**
That’s exactly what this does. We use these CRDs:

- `Site`
- `RouterAccess`
- `AccessGrant` (token issuer)
- `Link`
- `Connector` (exports a service)
- `Listener` (imports a service)

### 7.1 Site (both clusters)

**site-a/site.yaml**:

```yaml
apiVersion: skupper.io/v2alpha1
kind: Site
metadata:
  name: site-a
  namespace: interconnect-demo
spec:
  routerMode: interior
  metadata:
    displayName: "Tokyo site (site-a)"
```

**site-b/site.yaml** is the same but name `site-b`.

---

### 7.2 RouterAccess (site-a only)

We need site-b to **reach** site-a. On AWS/ROSA simplest is LB:

```yaml
apiVersion: skupper.io/v2alpha1
kind: RouterAccess
metadata:
  name: site-a-router
  namespace: interconnect-demo
spec:
  ingress: loadbalancer
```

This gives you an ELB hostname.
**You must copy that hostname** into `site-b/link-to-site-a.yaml` as:

```yaml
host: REPLACE_WITH_SITE_A_ROUTER_HOST
```

This is the **one** manual piece if you don’t also store the host in Vault.

---

### 7.3 AccessGrant (site-a)

We also added an `AccessGrant` — this is how you can generate a redeemable token:

```yaml
apiVersion: skupper.io/v2alpha1
kind: AccessGrant
metadata:
  name: site-a-to-site-b
  namespace: interconnect-demo
spec:
  redemptionsAllowed: 1
  expirationWindow: 1h
```

You can later use this to issue a token; in this demo we just show the object so GitOps owns it.

---

### 7.4 Link (site-b)

**site-b/link-to-site-a.yaml**:

```yaml
apiVersion: skupper.io/v2alpha1
kind: Link
metadata:
  name: link-to-site-a
  namespace: interconnect-demo
spec:
  tlsCredentials: link-to-site-a
  endpoints:
    - group: skupper-router
      host: REPLACE_WITH_SITE_A_ROUTER_HOST
      port: "55671"
      name: inter-router
```

- `tlsCredentials: link-to-site-a` → this is the secret ESO will create from Vault.
- `host:` → must be your site-a router LB hostname.

When the secret exists **and** the host is reachable, this link should go **Connected**.

---

### 7.5 Connector (site-a)

We want to **export** the Postgres RW service from site-a.
EDB creates a service like `site-a-db-rw` in `edb-demo` (or similar, depending on version).
We make a Connector that says: “expose that over the Skupper network.”

**site-a/connector-postgres.yaml**:

```yaml
apiVersion: skupper.io/v2alpha1
kind: Connector
metadata:
  name: site-a-postgres
  namespace: interconnect-demo
spec:
  routingKey: postgres-site-a
  host: site-a-db-rw.edb-demo.svc.cluster.local
  port: 5432
```

Key points:

- `routingKey: postgres-site-a` → this is how the other side subscribes
- `host:` is the internal service DNS
- `port: 5432` because Postgres

---

### 7.6 Listener (site-b)

On site-b we want a **local** K8s service called `site-a-db-rw` that actually talks to Tokyo over Skupper.

**site-b/listener-postgres.yaml**:

```yaml
apiVersion: skupper.io/v2alpha1
kind: Listener
metadata:
  name: postgres-from-site-a
  namespace: interconnect-demo
spec:
  routingKey: postgres-site-a
  host: site-a-db-rw
  port: 5432
```

This will create (under the covers) a K8s service in **site-b**:

```bash
oc -n interconnect-demo get svc site-a-db-rw
```

You can then `psql` to that service from **site-b**.

---

## 8. Testing

After Argo has synced **operators → site-a → site-b**:

1. **Check ExternalSecrets:**
   ```bash
   oc -n edb-demo get secret app-user-secret
   oc -n interconnect-demo get secret link-to-site-a
   ```

2. **Check Skupper site / link:**
   ```bash
   oc -n interconnect-demo get site
   oc -n interconnect-demo get link
   ```

3. **Check service on site-b:**
   ```bash
   oc -n interconnect-demo get svc | grep site-a-db-rw
   ```

4. **Run a psql pod on site-b:**
   ```bash
   oc -n interconnect-demo run psql --image=postgres:16 -- sleep 3600
   oc -n interconnect-demo exec -it deploy/psql --      psql -h site-a-db-rw -p 5432 -U appuser
   ```

5. **Set password from Vault** (if you used that value):
   ```bash
   export PGPASSWORD=AppUserP@ssw0rd
   ```

If that connects → your Skupper overlay + Vault-backed secrets are working.

---

## 9. Argo / ACM specifics

You already saw these secrets:

```bash
oc -n openshift-gitops get secret -l argocd.argoproj.io/secret-type=cluster
# ...
site-a-application-manager-cluster-secret
site-b-application-manager-cluster-secret
```

That means in your Argo `Application` manifests we can do:

```yaml
destination:
  name: site-a-application-manager-cluster-secret
  namespace: openshift-operators
```

and

```yaml
destination:
  name: site-b-application-manager-cluster-secret
  namespace: openshift-operators
```

So we’re **deploying to the spokes**, not the hub.

---

## 10. Console plugin (optional)

We included:

- `cluster-config/console-plugin/console-plugin.yaml`
- `cluster-config/console-plugin/console-operator-patch.yaml`

This enables the **RHSI console plugin** so you can see sites/links in the OpenShift console.

You can apply that only to the hub or to both clusters — up to you.

---

## 11. Things still manual (on purpose)

- **Filling in Vault details**:
  we can’t know your actual Vault URL, auth mount, or role name → please edit:
  - `vault/site-a-secretstore.yaml`
  - `vault/site-b-secretstore.yaml`

- **Router hostname from site-a**:
  after `RouterAccess` is created on site-a, get the LB/hostname:
  ```bash
  oc -n interconnect-demo get svc -o wide | grep skupper-router
  ```
  put that hostname in:
  - `site-b/link-to-site-a.yaml` under `spec.endpoints[0].host`

If you want **zero** manual changes, you can store **that** hostname in Vault too and use ESO `template:` support, but I’ve kept it simple here.

---

## 12. Variations / next steps

- Add **EDB replica** on site-b and do real Postgres async replication across Skupper.
- Switch from **ClusterSecretStore** to **SecretStore** if you want per-namespace isolation.
- Add **SealedSecrets** only for the Vault auth JWT if you can’t use k8s auth.
- Add **policy** to Skupper objects once you tighten cluster-to-cluster exposure.
- Drive **site-a** and **site-b** from two **separate** repos and use ACM **GRC** to control which cluster gets what.

---

That’s the full README.
