# 1Password Operator

This set of manifests deployes the [1Password Connect Server and Kubernetes Operator](https://github.com/1Password/connect-helm-charts/tree/main/charts/connect) (and associated CRDs). These are then used for secret management by the rest of the cluster.

The Connect server and operator require a credentials file and token respectively. Since I haven't set up SOPS or similar at the time of writing, these are created manually:

```bash
# This will create 1password-credentials.json. DO NOT COMMIT IT
op connect server create hoyeslab --vaults infrastructure

# Create a token. You should probably echo this and save it to 1password.
export OP_CONNECT_TOKEN=$(op connect token create hoyeslab --server hoyeslab --vault infrastructure)

echo $OP_CONNECT_TOKEN
```

Save the created `1password-credentials.json` file and created token to 1password just in case. **Do not commit the `1password-credentials.json`** file.

We can now put these into the cluster:

```bash
# Create the namespace
kubectl apply -f namespace.yaml

# Create a secret from the credentials file, which needs to be base64 encoded
kubectl create secret generic op-credentials \
  --namespace 1password \
  --from-literal=1password-credentials.json=$(cat 1password-credentials.json | base64 --wrap=0)

# Create a secret for the token
kubectl create secret generic onepassword-token --from-literal=token=${OP_CONNECT_TOKEN} --namespace=1password
```
