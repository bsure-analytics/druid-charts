# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Is

Helm chart repository for deploying Apache Druid on Kubernetes via the Druid Operator. Contains two charts:

- **`charts/druid`** — Production chart. Encapsulates a Druid Operator CRD (`Druid` custom resource) with supporting RBAC, secrets, and services. No subchart dependencies.
- **`charts/druid-dev`** — Development umbrella chart. Depends on `druid` (local) plus Druid Operator, MinIO, PostgreSQL, and Metrics Server as subcharts.

## Architecture

The `druid` chart does not deploy Druid pods directly. It creates a `Druid` CRD instance that the Druid Operator reconciles into the actual cluster. Key design choices:
- Kubernetes API replaces ZooKeeper for coordination
- Kubernetes Jobs replace Middle Manager for ingestion tasks
- S3 (or MinIO in dev) for segment and log storage
- PostgreSQL for metadata storage

Node types defined in `spec.nodes`: `brokers`, `coordinators`, `routers`, `historicals`, `hot` (optional), `cold` (optional). The `_helpers.tpl` includes a `druid.properties` template that recursively converts nested YAML dicts into Java properties format.

The `druid-dev` chart wires everything together with preconfigured MinIO credentials (`minio`/`minio123`), a PostgreSQL database, and broker autoscaling.

## Common Commands

The root `Makefile` delegates all targets to `charts/druid-dev/Makefile`. Default namespace/release is `druid`.

```bash
# Development workflow (from repo root or charts/druid-dev)
make upgrade          # (or: make up) helm upgrade --install
make diff             # preview changes (requires helm-diff plugin)
make template         # render manifests without applying
make test             # run helm test
make uninstall        # (or: make down) helm uninstall
make dependency-update  # (or: make init) sync druid-dev version with druid chart, then helm dependency update

# Override namespace and release
make upgrade NAMESPACE=my-ns RELEASE=my-release

# Pass extra helm flags
make upgrade HELM_OPTS="--set druid.spec.nodes.brokers.replicas=2"
```

For the `druid` chart alone, use the same targets from `charts/druid/`.

## Versioning and Releases

Both charts share the same version number. The `druid` chart's `Chart.yaml` is the source of truth — `make dependency-update` in `druid-dev` syncs from it using `yq`.

Release process:
1. Bump version in `charts/druid/Chart.yaml` (and `appVersion` when Druid itself is bumped)
2. Run `make dependency-update` from `charts/druid-dev` (or root) to sync
3. Tag with `v*.*.*` — GitHub Actions packages both charts and publishes via chart-releaser to GitHub Pages

During development, the `druid` dependency uses `repository: file://../druid`. The `make release` target switches this to the GitHub Pages URL for non-SNAPSHOT versions.

## Values Structure

`charts/druid/values.yaml` — top-level keys:
- `spec.nodes.<name>` — per-node-type config (replicas, JVM, resources, storage, runtimeProperties)
- `spec.extensions` — Druid extension lists
- `spec.metadataStorage` — database connector config
- `spec.s3` — segment and log bucket configuration
- `secret.stringData` — AWS/S3 credentials
- `service` — ClusterIP service for brokers/routers

`charts/druid-dev/values.yaml` — overrides for dev environment (MinIO endpoint, PostgreSQL connection, autoscaling settings, subchart toggles via `operator.enabled`, `minio.enabled`, etc.)

## Druid Version Support

The chart tracks the upstream Apache Druid version via `appVersion` in `charts/druid/Chart.yaml`. The image tag defaults to appVersion but can be overridden with `spec.image.tag`.

The `druid.yaml` template contains version-conditional logic (e.g. `ge (.image.tag | default $.Chart.AppVersion | semver).Major 33` for `DRUID_SET_HOST_IP`). When adding support for a new Druid version, check the release notes for:
- Extension changes — the extension list in `spec.extensions` must match what's available in that Druid version (e.g. `druid-exact-count-bitmap` was added in 35.0.0, `druid-multi-stage-query` was moved to core in 35.0.0 and must not be in the load list)
- New mandatory environment variables or JVM flags — add version-conditional blocks in the template
- Kubernetes extension changes — the coordinator node's runtime properties are partially hardcoded in the template (`druid.indexer.runner.type=k8s`, `druid.indexer.runner.namespace`, `druid.indexer.task.encapsulatedTask=true`); additional K8s runner properties go through `runtimeProperties` in values

New opt-in Druid features (e.g. `druid.indexer.runner.useK8sSharedInformers`, `druid.indexer.task.buildV10`) do not require template changes — users configure them via `spec.nodes.<name>.runtimeProperties` or `spec.commonRuntimeProperties` in values.
