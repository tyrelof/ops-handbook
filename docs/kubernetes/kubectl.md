# Safe kubectl Patterns

- Always use `--namespace=<ns>` for clarity (or set context contextually).
- Use `kubectl get ... -o wide` to reveal more info (IPs, Nodes).
- Prefer `kubectl describe` over `kubectl edit` for diagnostics.
- Use `kubectl logs -f` for live log tailing.
- For resource diffs, use `kubectl diff -f <file>`.
