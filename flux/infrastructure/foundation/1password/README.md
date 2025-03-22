# 1Password Operator

This set of manifests deployes the [1Password Connect Server and Kubernetes Operator](https://github.com/1Password/connect-helm-charts/tree/main/charts/connect) (and associated CRDs). These are then used for secret management by the rest of the cluster.

The Connect server and operator require a credentials file and token respectively. Since I haven't set up SOPS or similar at the time of writing, these are created manually:

```bash
# This will create 1password-credentials.json. DO NOT COMMIT IT
op connect server create hoyeslab --vaults infrastructure

# Create a token. Save it to 1password.
op connect token create hoyeslab --server hoyeslab --vault infrastructure
```

Save the created `1password-credentials.json` file and created token to 1password just in case. **Do not commit the `1password-credentials.json`** file.

We can now put these into the cluster:

```bash
# Create the namespace. If this cluster is already bootstrapped with configs, this will already exist.
kubectl apply -f namespace.yaml

# 1Password vault and item name with the saved token and config file
CONNECT_ITEM_PATH="op://infrastructure/1Password Connect Token"

# Create a secret from the credentials file, which needs to be base64 encoded
kubectl create secret generic op-credentials \
  --namespace 1password \
  --from-literal=1password-credentials.json=$(op read "${CONNECT_ITEM_PATH}/1password-credentials.json" | base64 --wrap=0)

# Create a secret for the token
kubectl create secret generic onepassword-token \
  --namespace 1password \
  --from-literal=token=$(op read "${CONNECT_ITEM_PATH}/credential")
```
