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

## install flux
```bash
flux install
```

### add initial flux secrets
```bash
sops --decrypt secrets/sidero/flux-system.enc.yaml | \
  kubectl apply -f -
```
### age config
```bash
cat ~/.config/sops/age/keys.txt | kubectl create secret generic sops-age \
  --namespace=flux-system \
  --from-file=age.agekey=/dev/stdin
```

### add flux-system kustomization
```bash
kubectl apply -f clusters/sidero/flux-system/gotk-sync.yaml
flux reconcile kustomization flux-system --with-source flux-system
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
talosctl --context pk8s -n ${PK8S_ENDPOINT} kubeconfig
kubectl config use-context admin@pk8s
```

## install cilium
```bash
#!/bin/bash
set -e

CILIUM_CLI_VERSION=$(curl -s https://raw.githubusercontent.com/cilium/cilium-cli/master/stable.txt)
CLI_ARCH=amd64
if [ "$(uname -m)" = "aarch64" ]; then CLI_ARCH=arm64; fi
curl -L --fail --remote-name-all https://github.com/cilium/cilium-cli/releases/download/${CILIUM_CLI_VERSION}/cilium-linux-${CLI_ARCH}.tar.gz{,.sha256sum}
sha256sum --check cilium-linux-${CLI_ARCH}.tar.gz.sha256sum
sudo tar xzvfC cilium-linux-${CLI_ARCH}.tar.gz ~/bin
rm cilium-linux-${CLI_ARCH}.tar.gz{,.sha256sum}
```

```bash
cilium install --helm-set-string securityContext.privileged=true \
  --helm-set-string kubeProxyReplacement=strict \
  --helm-set-string hubble.relay.enabled=true \
  --helm-set-string hubble.ui.enabled=true
```

check the status and wait for all OK
```bash
cilium status
```

```bash
cilium hubble enable --ui
```

```bash
#!/bin/bash
set -e

export HUBBLE_VERSION=$(curl -s https://raw.githubusercontent.com/cilium/hubble/master/stable.txt)
HUBBLE_ARCH=amd64
if [ "$(uname -m)" = "aarch64" ]; then HUBBLE_ARCH=arm64; fi
curl -L --fail --remote-name-all https://github.com/cilium/hubble/releases/download/$HUBBLE_VERSION/hubble-linux-${HUBBLE_ARCH}.tar.gz{,.sha256sum}
sha256sum --check hubble-linux-${HUBBLE_ARCH}.tar.gz.sha256sum
sudo tar xzvfC hubble-linux-${HUBBLE_ARCH}.tar.gz ~/bin
rm hubble-linux-${HUBBLE_ARCH}.tar.gz{,.sha256sum}
```

## install flux
```bash
flux install
```

### add initial flux secrets
sops --decrypt secrets/pk8s/flux-system.enc.yaml | \
  kubectl apply -f -

### age config
```bash
cat ~/.config/sops/age/keys.txt | kubectl create secret generic sops-age \
  --namespace=flux-system \
  --from-file=age.agekey=/dev/stdin
```

### add flux-system kustomization
```bash
kubectl apply -f clusters/pk8s/flux-system/gotk-sync.yaml
flux reconcile kustomization flux-system --with-source flux-system
```