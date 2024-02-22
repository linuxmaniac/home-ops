# 🪙 Sidero

The following applies to sidero v1.6.4
## Dependencies

- [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/)
- [clusterctl](https://cluster-api.sigs.k8s.io/user/quick-start.html#install-clusterctl)
- [talosctl](https://www.talos.dev/v1.6/introduction/getting-started/#talosctl)

## Install Talos on RPI4

Flash USB SSD with Talos image, (found here.)[https://github.com/siderolabs/talos/releases/download/v1.6.4/metal-rpi_generic-arm64.raw.xz]

## Creating management cluster
```bash
export SIDERO_ENDPOINT=192.168.4.4
```

Be sure ``iscssi-tools`` extension is installed
```bash
talosctl -e ${SIDERO_ENDPOINT} -n ${SIDERO_ENDPOINT} get extensions
```

```bash
sops --decrypt integrations/sidero/secrets.enc.yaml > integrations/sidero/secrets.yaml

talosctl gen config sidero https://sidero.lab:6443 --with-secrets integrations/sidero/secrets.yaml --config-patch-control-plane @integrations/sidero/sidero.lab.yaml --output  integrations/sidero/

mkdir -p ~/.talos/
cp integrations/sidero/talosconfig ~/.talos/config

talosctl --context sidero -n ${SIDERO_ENDPOINT} kubeconfig

# bootstrap single node management cluster
talosctl --context sidero apply-config -e ${SIDERO_ENDPOINT} -n ${SIDERO_ENDPOINT} --file integrations/sidero/controlplane.yaml --insecure

talosctl --context sidero -e ${SIDERO_ENDPOINT} -n ${SIDERO_ENDPOINT} bootstrap
```

Get ``iscssi-tools`` extension installed
```bash
talosctl --context sidero upgrade --image factory.talos.dev/installer/c9078f9419961640c712a8bf2bb9174933dfcf1da383fd8ea2b7dc21493f8bac:v1.6.5 --preserve --force
```

## install flux

```bash
flux --context admin@sidero bootstrap github --owner=linuxmaniac --repository=home-ops --path=clusters/sidero --personal
```

### age config
```bash
cat ~/.config/sops/age/keys.txt | \
kubectl --context admin@sidero create secret generic sops-age \
  --namespace=flux-system \
  --from-file=age.agekey=/dev/stdin
```

### add flux-system kustomization
```bash
kubectl --context admin@sidero apply -f clusters/sidero/flux-system/gotk-sync.yaml
flux reconcile kustomization flux-system --with-source flux-system
```

## Install sidero
```bash
SIDERO_CONTROLLER_MANAGER_AUTO_BMC_SETUP=false \
SIDERO_CONTROLLER_MANAGER_DEPLOYMENT_STRATEGY=Recreate \
SIDERO_CONTROLLER_MANAGER_HOST_NETWORK=true \
SIDERO_CONTROLLER_MANAGER_API_ENDPOINT=${SIDERO_ENDPOINT} \
clusterctl init -b talos -c talos -i sidero
```

## Prepare rpi4 customization
```bash
kubectl --context admin@sidero -n sidero-system scale deployment sidero-controller-manager --replicas 0

kubectl --context admin@sidero -n sidero-system patch deployments.apps sidero-controller-manager \
--patch "$(cat integrations/sidero/deployment.patch.yaml)"

kubectl --context admin@sidero -n sidero-system scale deployment sidero-controller-manager --replicas 1
```
## Initial pk8s controlplane
```bash
export PK8S_ENDPOINT=192.168.4.7

kubectl --context=admin@sidero --namespace sidero-system get secret \
    pk8s-talosconfig -o jsonpath='{.data.talosconfig}' \
  | base64 -d > integrations/sidero/pk8s-talosconfig
```
Put that info at ~/.talos/config
```bash
talosctl --context pk8s -e ${PK8S_ENDPOINT} -n ${PK8S_ENDPOINT} bootstrap
talosctl --context pk8s -e ${PK8S_ENDPOINT} -n ${PK8S_ENDPOINT} kubeconfig
kubectl config use-context admin@pk8s
```

## install flux
```bash
flux --context admin@pk8s bootstrap \
  --components-extra=image-reflector-controller,image-automation-controller \
  github --owner=linuxmaniac --repository=home-ops --path=clusters/pk8s \
  --read-write-key \
  --personal
```

### age config
```bash
cat ~/.config/sops/age/keys.txt | kubectl --context admin@pk8s create secret generic sops-age \
  --namespace=flux-system \
  --from-file=age.agekey=/dev/stdin
```

### add flux-system kustomization
```bash
kubectl --context admin@pk8s apply -f clusters/pk8s/flux-system/gotk-sync.yaml
flux --context admin@pk8s reconcile kustomization flux-system --with-source flux-system
```
