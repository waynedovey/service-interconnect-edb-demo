# Red Hat Service Interconnect (Skupper v2) + EDB / Cloud Native PostgreSQL
This repo is meant to be used by OpenShift GitOps (Argo CD) running on the ACM hub to roll out
to two managed clusters:
- site-a → AP Northeast 1 (Tokyo)
- site-b → AP Northeast 2 (Seoul)

Repo URL you will actually use in Argo: **https://github.com/waynedovey/service-interconnect-edb-demo**

Folders:
- cluster-config/ → console plugin for RHSI
- operators/ → skupper-operator + cloud-native-postgresql subscriptions
- site-a/ → Skupper Site, RouterAccess, AccessGrant, EDB primary, Connector
- site-b/ → Skupper Site, Link, Listener, optional EDB
- argo-examples/ → Argo Applications targeting your ACM-imported clusters

Apply order:
1. argo-examples/si-operators-site-a.yaml
2. argo-examples/si-operators-site-b.yaml
3. argo-examples/si-site-a.yaml
4. argo-examples/si-site-b.yaml (will stay OutOfSync until you put real Skupper token on site-b)
