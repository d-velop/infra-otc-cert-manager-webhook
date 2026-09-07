# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A cert-manager ACME DNS01 webhook solver for Open Telekom Cloud (OTC) DNS, written in Go. It implements the `webhook.Solver` interface (`Present`/`CleanUp`) so cert-manager can create/delete the `_acme-challenge` TXT records needed for Let's Encrypt DNS validation, using OTC's DNS API via the `gophertelekomcloud` client.

Originally from `github.com/hpi-schul-cloud/infra-otc-cert-manager-webhook` (still the Go module path); deployed by d-velop as `infra-otc-cert-manager-webhook`.

## Commands

```bash
make                    # downloads kubebuilder test binaries, then runs the full test suite
go test -v ./otcdns     # OTC DNS client tests only (needs a local clouds.yaml, see below)
go test -v .            # cert-manager conformance test only (needs kubebuilder + testdata/otcdns/manifests/config.json)
make build              # docker build -t webhook:latest .
make rendered-manifest.yaml  # helm template the chart into _out/rendered-manifest.yaml (no Secrets included)
make clean              # removes _test/kubebuilder
```

There is no linter/formatter target beyond standard `go vet`/`gofmt`.

### Test setup (required before tests will pass)

Tests hit the real OTC API — there is no mock/fake client.

- **`otcdns/client_test.go`** (package `otcdns`): needs `~/.config/openstack/clouds.yaml` with an `otcaksk` and/or `otcuser` profile (see `otcdns/config.go` — `OtcProfileNameAkSk`/`OtcProfileNameUser`). Template: `_examples/clouds.yaml`.
- **`main_test.go`** (root, the cert-manager conformance suite via `dns.NewFixture`): needs `testdata/otcdns/manifests/config.json` with `accessKey`/`secretKey` set directly (the `...SecretRef` fields don't work outside Kubernetes). Template: `_examples/config.json`. This test also needs the kubebuilder binaries that `make` fetches into `_test/kubebuilder/bin`, and it resolves DNS against OTC's real nameservers (`80.158.48.19:53`), against the zone `hpi-schul-cloud.dev.` by default (override with `TEST_ZONE_NAME`).

## Architecture

```
main.go            → registers otcdns.NewSolver() with cert-manager's webhook server under GROUP_NAME
otcdns/solver.go    → OtcDnsSolver: Present()/CleanUp(), reads K8s Secrets for AK/SK, builds an OtcDnsClient per request
otcdns/config.go    → OtcDnsConfig (decoded from the ClusterIssuer webhook `config` JSON) + local clouds.yaml/OS-env auth helpers (used only by tests / NewDNSV2Client)
otcdns/client.go    → OtcDnsClient: thin wrapper around gophertelekomcloud's DNS v2 zones/recordsets API
```

Request flow: cert-manager calls `Present`/`CleanUp` with a `ChallengeRequest` → `getOtcDnsClientFromChallengeRequest` decodes the issuer's `config` JSON into `OtcDnsConfig` → resolves the access/secret key either inline (`config.AccessKey`/`SecretKey`, testing-only) or via `AccessKeySecretRef`/`SecretKeySecretRef` looked up in the challenge's `ResourceNamespace` using the solver's own in-cluster Kubernetes client (built in `Initialize`) → builds an `OtcDnsClient` via `NewDNSV2ClientWithAuth` (AK/SK auth) → looks up the hosted zone, then creates/updates/deletes the TXT recordset.

Key behaviors worth knowing before touching `client.go`/`solver.go`:
- All TXT challenge values are stored quoted (`getSafeTxtValue`) and a single recordset can hold multiple challenge values (needed for concurrent validation of the same FQDN, e.g. multiple SANs).
- The OTC API rejects updating a recordset to an empty `Records` list, so removing the last value means deleting the whole recordset instead (`DeleteTxtRecordValue`'s `deleteRecordsetIfEmpty` — always `true` from `CleanUp`).
- The DNS record name defaults to `_acme-challenge.<zone>`; `OtcDnsClient.Subdomain` (derived from `ChallengeRequest.ResolvedFQDN` minus `ResolvedZone`) overrides this for custom subdomain setups.
- `GetHostedZone`/`GetTxtRecordSet` both error out if the OTC API returns anything other than exactly one match — there's no pagination/multi-zone handling.

### Deployment

Helm chart lives in `deploy/infra-otc-cert-manager-webhook/`. It installs the webhook, an `APIService` for the cert-manager webhook API group, and RBAC/PKI (self-signed cert via `templates/pki.yaml`) needed for cert-manager to call it. `groupName` in the chart must match `GroupName` (env var `GROUP_NAME`, default `otc.acme.d-velop.de`) used when registering the solver in `main.go`, and must also match the `groupName` referenced in each `ClusterIssuer`/`Issuer`'s `webhook` stanza. See `_examples/` for a full worked example (Secret, ClusterIssuer, Certificate).

The Docker image (`Dockerfile`) is a static, CGO-disabled build run as a non-root user (uid 10000/gid 10001), matching the chart's default `properties.runAsUser`/`runAsGroup`/`fsGroup`.
