# Minimal RBAC for "deployer" and "accessor" users in a namespace

This folder contains namespace-scoped Roles/RoleBindings:

- **Deployer**: can install/upgrade/uninstall the Helm release in *their* namespace.
- **Accessor**: can `kubectl port-forward` to the WordPress Service (and do basic reads).

## Recommended approach
Bind RBAC to **groups** rather than individual users so you can map identities via:
- Kubernetes CSRs (client cert CN + O=groups), or
- OIDC (e.g., Teleport), or
- another authn mechanism that can assert group membership.

These manifests default to two groups:
- `wp-deployers`
- `wp-accessors`

Edit subjects as needed.

## Apply
```bash
kubectl -n <namespace> apply -f rbac/10-role-wp-deployer.yaml \
  -f rbac/11-rolebinding-wp-deployer.yaml \
  -f rbac/20-role-wp-accessor.yaml \
  -f rbac/21-rolebinding-wp-accessor.yaml
```
