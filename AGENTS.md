# AGENTS.md

This file provides guidance to coding agents (e.g. Claude Code, claude.ai/code) when working with code in this repository.

## Repository purpose

Go module `open-cluster-management.io/cluster-proxy` — an OCM addon that automates the install/operation of [apiserver-network-proxy](https://github.com/kubernetes-sigs/apiserver-network-proxy) (ANP) so the hub cluster can reach services in managed clusters even when they sit in isolated VPCs. The agent on each managed cluster opens a **reverse-proxy tunnel** back to the hub; the hub-side proxy-server terminates those tunnels and routes hub-originated traffic out through them.

Two binaries:
- `addon-manager` — runs on the hub. Installs the hub-side proxy ingress (the proxy-servers) and registers the addon with OCM.
- `addon-agent` — runs on each managed cluster. Stays connected to the hub's proxy-server and serves proxied requests.

Fork is mirrored to `kluster-manager/cluster-proxy`; **upstream is `open-cluster-management-io/cluster-proxy`** and this repo tracks it.

## Architecture

- `cmd/addon-manager/main.go`, `cmd/addon-agent/main.go` — the two entry points.
- `cmd/Dockerfile` — a single Dockerfile produces both binaries (`make images`).
- `pkg/apis/proxy/v1alpha1/` — Kubernetes API types. Key kinds (from `PROJECT`):
  - `ManagedProxyConfiguration` — defines the proxy install (TLS, signing keys, hub endpoint, agent strategy).
  - `ManagedProxyServiceResolver` — declares which hub-side services should be proxied into a managed cluster.
- `pkg/proxyserver/` — hub-side:
  - `controllers/managedproxyconfiguration_controller.go` — reconciles `ManagedProxyConfiguration`.
  - `controllers/service_resolver_controller.go` — reconciles `ManagedProxyServiceResolver`.
  - `controllers/manifests.go` — manifest templates for the deployed ANP components.
  - `operator/` — addon manager wiring (registers as an OCM addon via `addon-framework`).
- `pkg/proxyagent/agent/` — spoke-side agent code (manages the tunnel back to the hub).
- `pkg/common/`, `pkg/config/`, `pkg/util/` — shared.
- `pkg/generated/` — generated typed clientset/listers/informers (`client-gen`). Do not hand-edit.
- `charts/cluster-proxy/` — Helm chart for the addon-manager install.
- `examples/`, `docs/`, `FQA.md` — usage docs and examples.
- `test/` — e2e tests.
- `hack/` — codegen helpers.
- `Makefile` — Kubebuilder-style harness (local Go toolchain, no AppsCode Docker wrapper). `controller-gen` / `kustomize` / `client-gen` install into `bin/` on first use.

## Common commands

This repo uses a **local Go toolchain**, not the AppsCode Docker harness.

- `make build` (alias `make all`) — `generate fmt vet`, then build.
- `make generate` — controller-gen DeepCopy generation.
- `make manifests` — controller-gen CRDs/RBAC/webhooks.
- `make client-gen` — regenerate `pkg/generated/`.
- `make fmt`, `make vet`, `make golint` — standard.
- `make verify` — `fmt vet golint`.
- `make test` — `manifests generate fmt vet`, then Go tests.
- `make install` / `make uninstall` — `kustomize` apply/remove CRDs against the current kube context.
- `make deploy` / `make undeploy` — install/remove the controller via kustomize.
- `make docker-build` — `test`, then docker build the manager image.
- `make docker-push` — push the built image.
- `make images` — build the all-in-one image from `cmd/Dockerfile` (both binaries).
- `make controller-gen` / `make kustomize` — install the tools into `bin/`.
- `make help` — list all targets with descriptions.

Run a single Go test:

```
go test ./pkg/proxyserver/controllers/... -run TestName -v
```

## Conventions

- Module path is `open-cluster-management.io/cluster-proxy` (**upstream**); imports must use that, not the GitHub URL.
- **Upstream-tracking** fork (mirrored as `kluster-manager/cluster-proxy`). Prefer rebasing onto upstream over diverging; isolate AppsCode-only patches so they replay cleanly.
- License: Apache-2.0 (`LICENSE`, `CODE_OF_CONDUCT.md`, `OWNERS`).
- Sign off commits (`git commit -s`); contributions follow the DCO (`DCO`, `CONTRIBUTING.md`).
- CRD API group is `proxy.open-cluster-management.io`. Domain in `PROJECT` is `open-cluster-management.io`; do not change it without a Kubebuilder migration.
- Do not hand-edit `zz_generated.*.go`, anything under `pkg/generated/`, or the controller-gen-generated CRD YAMLs — change `pkg/apis/proxy/v1alpha1/*_types.go` and re-run `make generate manifests client-gen`.
- Two binaries, one Docker image — `cmd/addon-manager/` is hub-side, `cmd/addon-agent/` is spoke-side. Don't merge them; they have different RBAC/runtime surfaces.
- Reverse-proxy tunnels are the architectural foundation here. The agent must always initiate the connection to the hub (so VPC-isolated clusters can still be reached) — don't reorder that direction.
- `Makefile` requires Go matching `go.mod`'s `go 1.21` line; bump `go.mod` and update CI together.
