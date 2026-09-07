## Image

Pull the container image used by this chart:

```bash
docker pull ghcr.io/d-velop/infra-otc-cert-manager-webhook:<tag>
```

Replace `<tag>` with the chart's `image.tag` value in `values.yaml` (defaults to the
version matching this chart's `appVersion` in `Chart.yaml`), or override it with
`--set image.tag=<tag>` on install/upgrade.

## Install

```bash
helm repo add otcdnswebhook https://d-velop.github.io/infra-otc-cert-manager-webhook/
helm repo update
helm install --namespace cert-manager otcdns-release otcdnswebhook/infra-otc-cert-manager-webhook
```
