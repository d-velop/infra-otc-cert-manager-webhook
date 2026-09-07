# Migration Guide

This release renames the webhook's `groupName` and moves the published Docker
image to a new registry. Both changes require action in every cluster running
this webhook — a plain `helm upgrade` is not enough on its own.

## 1. groupName change

The webhook's `groupName` changed from

```
infra-otc-cert-manager-webhook.hpi-schul-cloud.github.com
```

to

```
otc.acme.d-velop.de
```

`groupName` is the routing label cert-manager uses to find this webhook. It
must be identical in three places at all times:

- the chart's `groupName` value (`values.yaml`, or `--set groupName=...`)
- the `GROUP_NAME` environment variable the webhook pod actually starts with
  (set by the chart from the same value — you don't set this separately)
- every `ClusterIssuer`/`Issuer`'s `spec.acme.solvers[].dns01.webhook.groupName`

If any of these three disagree, cert-manager silently fails to find the
webhook and DNS01 challenges stop resolving — there is no obvious error
pointing at the mismatch, and it will not affect certificates that are already
issued, only future issuance/renewal.

**Steps, in order:**

1. Upgrade the Helm release with the new chart (new default `groupName:
   otc.acme.d-velop.de`). The webhook pod restarts and starts advertising
   itself under the new group name via its `APIService`.
2. Update `webhook.groupName` in **every** `ClusterIssuer`/`Issuer` that
   references this webhook, in every cluster, to `otc.acme.d-velop.de`.
   See `_examples/clusterissuer-solver-dns01-webhook.yaml` and
   `_examples/clusterissuer-staging-solver-dns01-webhook.yaml` for the
   updated examples.
3. Confirm challenges still resolve, e.g. by forcing a renewal
   (`kubectl cert-manager renew <cert>` or deleting the cert's `Secret`) and
   watching the `Certificate`/`CertificateRequest`/`Challenge` resources go
   `Ready`.

Steps 1 and 2 should happen close together — certificates already in flight
between the two steps will fail to validate until both sides agree again.

## 2. Image registry change

The image moved from Docker Hub to GitHub Container Registry:

```
schulcloud/infra-otc-cert-manager-webhook   →   ghcr.io/d-velop/infra-otc-cert-manager-webhook
```

Before upgrading any installation to a chart version that defaults to the new
`image.repository`:

- Make sure an image has actually been published to `ghcr.io/d-velop/infra-otc-cert-manager-webhook`
  (triggered by pushing a `v*` tag or publishing a GitHub release — see
  `.github/workflows/main.yml`).
- Make sure the GHCR package is public, or configure `image.pullSecrets` /
  `image.imagePullSecrets` with credentials that can pull it. GHCR packages
  are private by default.
- Pick an `image.tag` that actually exists at the new registry — old tags
  (e.g. `1.0.1`) only exist on Docker Hub and were not copied over.

If you don't override `image.repository` in your own values, upgrading the
chart to a version with the new default will point the Deployment at the new
registry immediately; without the above in place, pods will fail with
`ImagePullBackOff`.
