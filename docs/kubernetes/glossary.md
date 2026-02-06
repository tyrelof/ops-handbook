# Glossary & TL;DR

## Glossary
- **Pod**: The smallest deployable unit in Kubernetes.
- **Service**: An abstraction that defines a logical set of Pods and a policy by which to access them.
- **Ingress**: Manages external access to the services, typically HTTP.
- **EndpointSlice**: A scalable way to track network endpoints.

## TL;DR & Common Commands
- Check pod status: `kubectl get pods -n <ns>`
- Check service endpoints: `kubectl get endpointslice -n <ns>`
- Check ingress: `kubectl get ingress -n <ns> -o wide`
- View logs: `kubectl logs <pod> -n <ns> --tail=100`

## Publishing & Docs Workflow
- All changes must go through Git.
- No direct edits in production.
- Use PRs for visibility.
- Document every change.
