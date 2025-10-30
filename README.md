# RHSI (Skupper v2) + EDB/CNPG + Vault (External Secrets) Demo

This repo shows how to:
- install 3 operators (RHSI, Cloud Native PostgreSQL, External Secrets);
- define Skupper Sites for site-a (Tokyo) and site-b (Seoul);
- pull the Skupper link secret and the DB app-user secret from **Vault** using **External Secrets Operator**;
- drive everything from ACM/Argo.

See `vault/` for SecretStore templates.
