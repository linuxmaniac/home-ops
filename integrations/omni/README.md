Prereq
======
download and install omnictl and omniconfig from omni

kubectl and kubelogin (kubectl_oidc_login)
```bash
cd /tmp
wget https://github.com/int128/kubelogin/releases/download/v1.34.1/kubelogin_linux_amd64.zip
unzip kubelogin_linux_amd64.zip
mv kubelogin ~/bin/kubectl_oidc_login
```

Create MachineClasses
=====================
```bash
omnictl apply -f main.yaml
```

Create cluster
==============
```bash
export CLUSTER=pk8s-dev
export CLUSTER_BRANCH=dev
omnictl cluster template sync --file cluster-${CLUSTER}.yaml
```
Download talosconfig and kubeconfig from omni and merge it to
``~/.talos/config`` and ``~/.kube/config``

Cilium CLI
==========
```bash
CILIUM_CLI_VERSION=$(curl -s https://raw.githubusercontent.com/cilium/cilium-cli/main/stable.txt)
CLI_ARCH=amd64
if [ "$(uname -m)" = "aarch64" ]; then CLI_ARCH=arm64; fi
curl -L --fail --remote-name-all "https://github.com/cilium/cilium-cli/releases/download/${CILIUM_CLI_VERSION}/cilium-linux-${CLI_ARCH}.tar.gz{,.sha256sum}"
sha256sum --check "cilium-linux-${CLI_ARCH}.tar.gz.sha256sum"
tar xzvfC "cilium-linux-${CLI_ARCH}.tar.gz" ~/bin/
rm "cilium-linux-${CLI_ARCH}.tar.gz" "cilium-linux-${CLI_ARCH}.tar.gz.sha256sum"
```

workers lable
============
worker nodes need ``node-role.kubernetes.io/worker: 'true'`` label

install flux
============

```bash
flux --context "omni-${CLUSTER}" bootstrap \
  github --owner=linuxmaniac --repository=home-ops \
  --path="clusters/${CLUSTER}" --branch=${CLUSTER_BRANCH} --personal \
  --components-extra image-reflector-controller,image-automation-controller
```

age config
==========
```bash
cat ~/.config/sops/age/keys.txt | \
kubectl --context "omni-${CLUSTER}" create secret generic sops-age \
  --namespace=flux-system \
  --from-file=age.agekey=/dev/stdin
```

Update Cilium manifest
======================

```bash
helm repo add cilium https://helm.cilium.io/
# if you already have it
helm repo update
# produce cilium manifest using our choices
helm template cilium/cilium -f infrastructure/controllers/cilium-values.yaml --namespace kube-system > infrastructure/controllers/cilium.yaml
```

Cilium status
=============

```bash
cilium  status --context omni-${CLUSTER}
```

TODO
====
https://isovalent.com/blog/post/migrating-from-metallb-to-cilium/
