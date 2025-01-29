manage certs
============

https://www.talos.dev/v1.9/talos-guides/howto/cert-management/#generating-new-client-configuration

We use secrets.yml for sidero

```bash
sops --decrypt integrations/sidero/secrets.enc.yaml > integrations/sidero/secrets.yaml

talosctl gen config sidero https://sidero.lab:6443 --with-secrets integrations/sidero/secrets.yaml \
  --config-patch @integrations/sidero/common.yaml \
  --config-patch-worker @integrations/sidero/sidero-node0.lab.yaml \
  --config-patch-control-plane @integrations/sidero/sidero.lab.yaml \
  --output  integrations/sidero/
```
copy ca crt key values to ``~/.talosctl/config`` at sidero context

For pk8s we need to fetch ``pk8s-talos`` secret on ``sidero-system`` namespace

```bash
kubectl --context admin@sidero -n sidero-system get secret pk8s-talos -o jsonpath='{.data.bundle}' | base64 --decode > ~/tmp/pk8s-talos.txt
```
and generate a config

```bash
talosctl gen config --with-secrets ~/tmp/pk8s-talos.txt --output-types talosconfig -o ~/tmp/talosconfig pk8s https://pk8s.lab:6443
```
copy ca crt key values to ``~/.talosctl/config`` at sidero context

Then just generate kubeconfig for both clusters

```bash
talosctl --context sidero kubeconfig
talosctl --context pk8s kubeconfig
```

Have fun.
