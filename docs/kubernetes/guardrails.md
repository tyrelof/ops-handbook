# Platform Guardrails

These are **rules**, not suggestions.

## ❌ App teams MUST NOT:
- Edit Ingress resources
- Edit Services
- Edit ExternalSecrets / SecretStores
- Touch IRSA / IAM
- Expose ports other than 80/443
- Change probes without review

## ✅ App teams MAY:
- Read logs
- Describe pods
- Change app‑level env vars (via values)
- Scale replicas (if allowed)

> [!CAUTION]
> Violation of guardrails = platform instability.
