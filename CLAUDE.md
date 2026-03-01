# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

K3S-Remote is a documentation and configuration repository for deploying lightweight Kubernetes (K3s) clusters on remote machines with Traefik ingress and SSL certificate management via cert-manager/Let's Encrypt. It is not a traditional application codebase — it contains Kubernetes manifests and a comprehensive installation guide.

## Repository Structure

- **README.md** — Complete step-by-step installation guide (K3s setup, Traefik, cert-manager, app deployment)
- **traefik.yml** — HelmChart resource for Traefik v3 ingress controller (chart v39.0.2, deployed to `kube-system` namespace)
- **cluster-issuer.yml** — Let's Encrypt ClusterIssuer using ACME HTTP-01 challenge with Traefik
- **dummy_app.yml** — Sample Go webapp Deployment + ClusterIP Service (image: `nicomincuzzi/go-webapp:0.1.0`, port 3030)
- **ingress.yml** — Ingress resource routing `gowebapp.dev.pettycashmate.co` to the dummy app with TLS

## Required Tools

- `k3sup` — K3s cluster installer
- `kubectl` — Kubernetes CLI
- `helm` v3 — Package manager (used for cert-manager installation)
- `k9s` — Optional TUI for cluster management

## Applying Manifests

```bash
kubectl apply -f traefik.yml
kubectl apply -f dummy_app.yml
kubectl apply -f cluster-issuer.yml
kubectl apply -f ingress.yml
```

## Architecture Flow

K3s cluster → Traefik ingress controller → Ingress rules (with TLS via cert-manager/Let's Encrypt) → ClusterIP Service → Go webapp pods
