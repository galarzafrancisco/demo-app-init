# Deployment Guide

This repository deploys with the same GitOps pattern used by `ai-monorepo`.

## Overview

1. Application code lives in this repository.
2. Kubernetes manifest templates live in `install/manifests/`.
3. GitHub Actions builds and pushes `ghcr.io/galarzafrancisco/demo-app-init`.
4. GitHub Actions copies `install/manifests/` into `spark-ops/k8s/demo-app-init`.
5. The deployment PR in `spark-ops` updates the image tag with Kustomize.
6. ArgoCD in `spark-ops` watches `k8s/*/overlays/main` and syncs the cluster.

## Container

- `Dockerfile` builds the Vite app with Node 24 and serves the static output from `nginxinc/nginx-unprivileged`.
- `nginx.conf` listens on port `8080` and rewrites unknown routes to `index.html` for SPA routing.

## Kubernetes Layout

Manifest root: `install/manifests/`

- `base/namespace.yaml`: namespace `demo-app-init`
- `base/deployment.yaml`: single replica deployment using `ghcr.io/galarzafrancisco/demo-app-init:main`
- `base/service.yaml`: ClusterIP service on port `80` to container port `8080`
- `base/listener-set.yaml`: Gateway API HTTPS listener with TLS
- `base/route.yaml`: Gateway API HTTPS application route
- `base/https-redirect.yaml`: Gateway API HTTP-to-HTTPS redirect
- `base/certificate.yaml`: cert-manager certificate
- `overlays/main/env.env`: environment-specific values
- `overlays/main/kustomization.yaml`: Kustomize replacements for overlay values

## Overlay Values

Current overlay values:

- `HOST`: public hostname for the Gateway API resources and certificate

The `main` overlay replaces:

- `ListenerSet.spec.listeners[0].hostname`
- `HTTPRoute.spec.hostnames[0]` in `route.yaml`
- `HTTPRoute.spec.hostnames[0]` in `https-redirect.yaml`
- `Certificate.spec.dnsNames[0]`

## GitHub Actions

Workflow: `.github/workflows/deploy.yml`

On pull requests to `main`:

- runs `npm ci`
- runs `npm run lint`
- runs `npm run build`

On pushes to `main`:

- builds and pushes the container image to GHCR
- copies manifests to `spark-ops/k8s/demo-app-init`
- runs `kustomize edit set image` in the copied base
- opens a deployment PR in `spark-ops`

## Required GitHub Secrets

- `SPARK_OPS_TOKEN`: token with access to push branches and open PRs in `galarzafrancisco/spark-ops`

`GITHUB_TOKEN` is used for publishing to GHCR.

## Values To Replace For Production

Before first production deployment, confirm:

- `install/manifests/overlays/main/env.env` has the real public hostname
- cert-manager issuer name is correct for the cluster
- the `public` Gateway in the `gateways` namespace is available

## spark-ops Expectations

ArgoCD auto-discovers `spark-ops/k8s/*/overlays/main` via the ApplicationSet in `spark-ops/ops/applications/base/application-set.yaml`.

That means this app must be copied to:

- `spark-ops/k8s/demo-app-init/base`
- `spark-ops/k8s/demo-app-init/overlays/main`
