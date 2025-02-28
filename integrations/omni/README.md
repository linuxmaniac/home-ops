Prereq
======
download and install omnictl and omniconfig from omni


Create MachineClasses
=====================
```bash
omnictl apply -f main.yaml
```

Create cluster
==============
```bash
export CLUSTER=pk8s-dev
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

install flux
============

```bash
flux --context "omni-${CLUSTER}" bootstrap \
  github --owner=linuxmaniac --repository=home-ops \
  --path="clusters/${CLUSTER}" --personal
```

age config
==========
```bash
cat ~/.config/sops/age/keys.txt | \
kubectl --context "omni-${CLUSTER}" create secret generic sops-age \
  --namespace=flux-system \
  --from-file=age.agekey=/dev/stdin
```


TODO
====
https://isovalent.com/blog/post/migrating-from-metallb-to-cilium/
