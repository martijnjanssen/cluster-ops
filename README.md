# cluster-ops

## Setup

GitHub PAT with **repo** and **read:packages** scopes.

```sh
flux bootstrap github \
--components-extra=image-reflector-controller,image-automation-controller \
--owner=martijnjanssen \
--repository=cluster-ops \
--branch=main \
--path=clusters/production \
--read-write-key \
--personal
```

The **read:packages** scope, `--components-extra=` and `--read-write-key` are optional, but are required for performing image updates. These image updates write and commit the image updates back to the repository.

## GitHub secret for pulling from [ghcr.io](https://ghcr.io):

```sh
kubectl create secret -n flux-system docker-registry ghcr-cred --docker-username=<GH_USERNAME> --docker-password=<GH_PAT> --dry-run=client -o yaml > ghcr-cred.yaml
kubeseal --cert pub-sealed-secrets.pem -o yaml < ghcr-cred.yaml > ghcr-cred-sealed.yaml
```

## Creating application secrets

```sh
kubectl create secret generic <secret-name> -n <namespace> --from-literal=key1=supersecret --from-literal=key2=topsecret --dry-run=client -o yaml > secret-name.yaml
kubeseal --cert pub-sealed-secrets.pem -o yaml < secret-name.yaml > secret-name-sealed.yaml
```
