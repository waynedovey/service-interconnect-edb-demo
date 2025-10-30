# Red Hat Service Interconnect (Skupper v2) + EDB / Cloud Native PostgreSQL
GitOps + RHACM example for two OpenShift 4.19 clusters:

- **site-a** = ap-northeast-1 (Tokyo)
- **site-b** = ap-northeast-2 (Seoul)

Repo (this one): **https://github.com/waynedovey/service-interconnect-edb-demo**

## What this provides

1. Operator installs:
   - Red Hat Service Interconnect (skupper-operator) from `redhat-operators`, channel `stable-2.1`
   - EDB / Cloud Native PostgreSQL (cloud-native-postgresql) from `certified-operators`, channel `stable-v1.27`
2. Console plugin to surface RHSI in the OpenShift web console
3. Skupper v2 declarative objects (`skupper.io/v2alpha1`):
   - Site (site-a / site-b)
   - RouterAccess (site-a) → creates LB so site-b can link
   - AccessGrant (site-a) → to issue token for site-b
   - Link (site-b) → consumes the token
   - Connector (site-a) → exports Postgres
   - Listener (site-b) → imports Postgres as a ClusterIP service
4. EDB cluster on site-a (`site-a-db`) and optional EDB cluster on site-b
5. Argo CD / ACM examples to apply to **both clusters under ACM management**

---

## Layout

```text
service-interconnect-edb-demo/
├── README.md
├── kustomization.yaml
├── cluster-config/
│   ├── kustomization.yaml
│   └── console-plugin/
│       ├── console-plugin.yaml
│       └── console-operator-patch.yaml
├── operators/
│   ├── rhs-interconnect/
│   │   ├── namespace.yaml
│   │   ├── operator-group.yaml
│   │   └── subscription.yaml
│   └── edb-cnpg/
│       ├── namespace.yaml
│       ├── operator-group.yaml
│       └── subscription.yaml
├── site-a/
│   ├── kustomization.yaml
│   ├── namespaces.yaml
│   ├── site.yaml
│   ├── router-access.yaml
│   ├── access-grant.yaml
│   ├── edb-cluster.yaml
│   └── connector-postgres.yaml
├── site-b/
│   ├── kustomization.yaml
│   ├── namespaces.yaml
│   ├── site.yaml
│   ├── access-token.yaml
│   ├── link-to-site-a.yaml
│   ├── listener-postgres.yaml
│   └── edb-cluster.yaml
└── argo-examples/
    ├── app-operators.yaml
    ├── app-site-a.yaml
    ├── app-site-b.yaml
    └── appset-multicluster.yaml
```

---

## Argo (OpenShift GitOps) – use with ACM-imported clusters

### 1. Operators (sync first)

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: si-operators
  namespace: openshift-gitops
  annotations:
    argocd.argoproj.io/sync-wave: "-1"
spec:
  project: default
  source:
    repoURL: https://github.com/waynedovey/service-interconnect-edb-demo
    targetRevision: main
    path: operators
  destination:
    server: https://kubernetes.default.svc
    namespace: openshift-operators
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

### 2. site-a (Tokyo)

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: si-site-a
  namespace: openshift-gitops
  annotations:
    argocd.argoproj.io/sync-wave: "0"
spec:
  project: default
  source:
    repoURL: https://github.com/waynedovey/service-interconnect-edb-demo
    targetRevision: main
    path: site-a
  destination:
    # if deploying to THIS cluster:
    server: https://kubernetes.default.svc
    namespace: interconnect-demo
    # if deploying to an ACM-managed remote cluster, use:
    # name: site-a
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

### 3. site-b (Seoul)

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: si-site-b
  namespace: openshift-gitops
  annotations:
    argocd.argoproj.io/sync-wave: "1"
spec:
  project: default
  source:
    repoURL: https://github.com/waynedovey/service-interconnect-edb-demo
    targetRevision: main
    path: site-b
  destination:
    server: https://kubernetes.default.svc
    namespace: interconnect-demo
    # or: name: site-b
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

### 4. ApplicationSet for ACM

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: si-edb-multicluster
  namespace: openshift-gitops
spec:
  generators:
    - clusters:
        selector:
          matchLabels:
            env: demo
  template:
    metadata:
      name: si-{{name}}
    spec:
      project: default
      destination:
        name: "{{name}}"
        namespace: interconnect-demo
      source:
        repoURL: https://github.com/waynedovey/service-interconnect-edb-demo
        targetRevision: main
        path: '{{if eq name "site-a"}}site-a{{else}}site-b{{end}}'
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
        syncOptions:
          - CreateNamespace=true
```

---

## The Skupper token step

This stays manual/secret-backed:

- `site-b/access-token.yaml` → fill in values from AccessGrant on site-a
- `site-b/link-to-site-a.yaml` → set `host:` to the LB/route created by `RouterAccess` on site-a

Use SealedSecrets or ExternalSecrets if you want to stay fully GitOps.

---

## Testing

```bash
oc --context site-b -n interconnect-demo get svc site-a-db-rw
oc --context site-b -n interconnect-demo run psql --image=postgres:16 -- sleep 3600
oc --context site-b -n interconnect-demo exec -it deploy/psql -- psql -h site-a-db-rw -p 5432 -U appuser
```

---

