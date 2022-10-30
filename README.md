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

## Automated image updates

All code for automated image updates resides in [`clusters/production/automation`](https://github.com/martijnjanssen/cluster-ops/tree/main/clusters/production/automation). The `<name>-registry.yaml` file registers the image to fetch tags for, while the `<name>-policy.yaml` specifies which tags to filter on. This filtering action allows you to specify what the latest image is you want the ImagePolicy to use. The results from this can then be used to auto-update the images in the deployment with a `Kustomization` as seen in [`apps/production/bite/kustomization.yaml`](https://github.com/martijnjanssen/cluster-ops/blob/main/apps/production/bite/kustomization.yaml).

```yaml
images:
  - name: ghcr.io/martijnjanssen/<someimage>
    newName: ghcr.io/martijnjanssen/<someimage>
    newTag: "423" # {"$imagepolicy": "flux-system:<someimage>:tag"}
```

The [`flux-system-automation.yaml`](https://github.com/martijnjanssen/cluster-ops/blob/main/clusters/production/automation/flux-system-automation.yaml) provides templating for the commits made by flux when doing the automated image updates.

## Secrets

To avoid having secrets magically appear in your cluster, [`sealed-secrets`](https://github.com/bitnami-labs/sealed-secrets) is used. SealedSecrets generates a private key, for which the public key can be stored in this git repository. With this public key, as seen in [`pub-sealed-secrets.pem`](https://github.com/martijnjanssen/cluster-ops/blob/main/pub-sealed-secrets.pem), you can create a kubernetes secret with the commands listed below and seal the secret with the public key. This sealed secret will be applied to the cluster and unsealed with the private key by the SealedSecret container in your cluster.

By doing this, you can commit your **SEALED** secrets to version control, whatever you do, **DO NOT COMMIT THE PLAIN SECRET**.

### GitHub secret for pulling from [ghcr.io](https://ghcr.io)

```sh
kubectl create secret -n flux-system docker-registry ghcr-cred --docker-username=<GH_USERNAME> --docker-password=<GH_PAT> --dry-run=client -o yaml > ghcr-cred.yaml
kubeseal --cert pub-sealed-secrets.pem -o yaml < ghcr-cred.yaml > ghcr-cred-sealed.yaml
```

### Creating application secrets

```sh
kubectl create secret generic <secret-name> -n <namespace> --from-literal=key1=supersecret --from-literal=key2=topsecret --dry-run=client -o yaml > secret-name.yaml
kubeseal --cert pub-sealed-secrets.pem -o yaml < secret-name.yaml > secret-name-sealed.yaml
```
