# Security, Privacy & Compliance

- **Segreti**: gestiti con vault; mai nel codice/CI logs.
- **PII**: classificazione dati, minimizzazione, mascheramento in log.
- **SBOM**: generare per ogni build (Syft), scan CVE (Grype).
- **Supply Chain**: pinning versioni, firma immagini (cosign), policy admission.
- **Accessi**: SSO, RBAC, audit trail.
