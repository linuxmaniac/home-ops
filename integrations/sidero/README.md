# 🪙 Sidero

The following applies to sidero v1.3.0
## Dependencies

- [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/)
- [clusterctl](https://cluster-api.sigs.k8s.io/user/quick-start.html#install-clusterctl)
- [talosctl](https://www.talos.dev/v1.3/introduction/getting-started/#talosctl)

## Install Talos on RPI4

Flash USB SSD with Talos image, (found here.)[https://github.com/siderolabs/talos/releases/latest/download/metal-rpi_4-arm64.img.xz]

## Creating management cluster
```bash
export SIDERO_ENDPOINT=192.168.4.4

sops --decrypt integrations/sidero/secrets.enc.yaml > integrations/sidero/secrets.yaml

talosctl gen config sidero https://sidero.lab:6443 --with-secrets integrations/sidero/secrets.yaml --config-patch-control-plane @integrations/sidero/sidero.lab.yaml --output  integrations/sidero/

mkdir -p ~/.talos/
cp integrations/sidero/talosconfig ~/.talos/config

talosctl --context sidero -n ${SIDERO_ENDPOINT} kubeconfig

# bootstrap single node management cluster
talosctl --context sidero apply-config -e ${SIDERO_ENDPOINT} -n ${SIDERO_ENDPOINT} --file integrations/sidero/controlplane.yaml --insecure

talosctl --context sidero -e ${SIDERO_ENDPOINT} -n ${SIDERO_ENDPOINT} bootstrap
```

## Install sidero
```bash
SIDERO_CONTROLLER_MANAGER_AUTO_BMC_SETUP=false \
SIDERO_CONTROLLER_MANAGER_HOST_NETWORK=true \
SIDERO_CONTROLLER_MANAGER_API_ENDPOINT=${SIDERO_ENDPOINT} \
clusterctl init -b talos -c talos -i sidero
```

## Prepare rpi4 customization
```bash
kubectl -n sidero-system scale deployment sidero-controller-manager --replicas 0

kubectl -n sidero-system patch deployments.apps sidero-controller-manager \
--patch "$(cat integrations/sidero/deployment.patch.yaml)"

kubectl -n sidero-system scale deployment sidero-controller-manager --replicas 1
```
