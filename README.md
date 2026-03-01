# K3S-Remote

A guide to install [K3s](https://k3s.io/) on a VM or Raspberry Pi with a custom Traefik Ingress Controller, SSL certificate management via cert-manager, and Let's Encrypt.

K3s is a lightweight, fully conformant Kubernetes distribution designed for resource-constrained environments like edge computing, IoT, and ARM devices. This repository provides step-by-step instructions and ready-to-use manifests to get a production-ready cluster running with HTTPS ingress.

### Architecture

```
┌─────────────┐     ┌────────────────────┐     ┌─────────────────┐     ┌──────────────┐     ┌────────────┐
│   Client    │────▶│  Traefik Ingress   │────▶│  Ingress Rules  │────▶│  ClusterIP   │────▶│  Go Webapp │
│  (Browser)  │     │  Controller        │     │  (TLS via LE)   │     │  Service     │     │  Pods      │
└─────────────┘     └────────────────────┘     └─────────────────┘     └──────────────┘     └────────────┘
                           │                           │
                      K3s Cluster              cert-manager /
                                              Let's Encrypt
```

## Table of Contents

- [Prerequisites](#prerequisites)
- [Repository Structure](#repository-structure)
- [What's K3s?](#whats-k3s)
- [1. Install K3s with k3sup](#1-install-k3s-with-k3sup)
  - [1.1 Multi-Node Cluster Setup](#11-multi-node-cluster-setup)
- [2. Install Ingress Controller](#2-install-ingress-controller)
- [3. Deploy a Dummy App](#3-deploy-a-dummy-app)
- [4. cert-manager](#4-cert-manager)
- [5. Expose App Externally via Ingress](#5-expose-app-externally-via-ingress)
- [Troubleshooting](#troubleshooting)
- [How to Contribute](#how-to-contribute)
- [License](#license)

## Prerequisites

- [kubectl](https://kubernetes.io/docs/tasks/tools/) - Kubernetes CLI
- [k3sup](https://github.com/alexellis/k3sup) - K3s installer over SSH
- [k9s](https://github.com/derailed/k9s) - Optional TUI for cluster management
- [Helm v3](https://helm.sh/docs/intro/install/) - Package manager for Kubernetes

## Repository Structure

| File | Description |
|------|-------------|
| `traefik.yml` | HelmChart resource for Traefik v3 ingress controller (chart v39.0.2, deployed to `kube-system` namespace) |
| `cluster-issuer.yml` | Let's Encrypt ClusterIssuer using ACME HTTP-01 challenge with Traefik |
| `dummy_app.yml` | Sample Go webapp Deployment + ClusterIP Service (`nicomincuzzi/go-webapp:0.1.0`, port 3030) |
| `ingress.yml` | Ingress resource routing `gowebapp.dev.pettycashmate.co` to the dummy app with TLS |

## What's K3s?

Find [here](https://k3s.io/) all you need.

## 1. Install K3s with k3sup

This step installs K3s on a remote machine via SSH. The `--disable traefik` flag prevents the default Traefik installation so we can deploy a custom version in the next step.

```sh
export IP=<HOST_IP>
k3sup install \
  --ip $IP \
  --user root \
  --ssh-key <SSH_PATH> \
  --merge \
  --local-path $HOME/.kube/config \
  --context my-k8s \
  --k3s-extra-args '--disable traefik'
```

Options for `install`:

- `--cluster` - start this server in clustering mode using embedded etcd (embedded HA)
- `--skip-install` - if you already have k3s installed, you can just run this command to get the `kubeconfig`
- `--ssh-key` - specify a specific path for the SSH key for remote login
- `--local-path` - default is `./kubeconfig` - set the file where you want to save your cluster's `kubeconfig`. By default this file will be overwritten.
- `--merge` - Merge config into existing file instead of overwriting (e.g. to add config to the default kubectl config, use `--local-path ~/.kube/config --merge`).
- `--context` - default is `default` - set the name of the kubeconfig context.
- `--ssh-port` - default is `22`, but you can specify an alternative port i.e. `2222`
- `--k3s-extra-args` - Optional extra arguments to pass to k3s installer, wrapped in quotes, i.e. `--k3s-extra-args '--disable traefik'` or `--k3s-extra-args '--docker'`. For multiple args combine them within single quotes `--k3s-extra-args '--disable traefik --docker'`.
- `--k3s-version` - set the specific version of k3s, i.e. `v0.9.1`
- `--ipsec` - Enforces the optional extra argument for k3s: `--flannel-backend` option: `ipsec`
- `--print-command` - Prints out the command, sent over SSH to the remote computer
- `--datastore` - used to pass a SQL connection-string to the `--datastore-endpoint` flag of k3s. You must use [the format required by k3s in the Rancher docs](https://rancher.com/docs/k3s/latest/en/installation/ha/).

See even more install options by running `k3sup install --help`.

> **Note:** [Traefik](https://github.com/traefik/traefik) can be configured by editing the `traefik.yml` file. To prevent k3s from using or overwriting the modified version, deploy k3s with `--disable traefik` and store the modified copy in the `k3s/server/manifests` directory. For more information, refer to the official [Traefik Helm Chart](https://github.com/traefik/traefik-helm-chart).

### 1.1 Multi-Node Cluster Setup

Build a 3-node Kubernetes cluster with k3s and k3sup, which uses SSH to make the whole process quick and painless.

> **Note:** Running k3s/MicroK8s on some ARM hardware may run into difficulties because `cgroups` are not enabled by default.
>
> This can be remedied on Ubuntu by editing the boot parameters:
>
> `sudo vi /boot/firmware/cmdline.txt`
>
> **Note:** In some Raspberry Pi Linux distributions the boot parameters are in `/boot/cmdline.txt` or `/boot/firmware/nobtcmd.txt`.
>
> Add the following:
>
> `cgroup_enable=cpuset cgroup_memory=1 cgroup_enable=memory`
>
> More details: https://microk8s.io/docs/install-alternatives#heading--arm

**1. Create the server**

In Kubernetes terminology, the server is often called the master.

```sh
export IP=<HOST_IP>

k3sup install \
  --ip $IP \
  --user root \
  --ssh-key <SSH_PATH> \
  --merge \
  --local-path $HOME/.kube/config \
  --context my-k8s \
  --k3s-extra-args '--disable traefik'
```

k3s is so fast to start up, that it may be ready for use after the command has completed.

Test it out:

```sh
export KUBECONFIG=`pwd`/kubeconfig

kubectl get node -o wide
NAME    STATUS   ROLES    AGE   VERSION         INTERNAL-IP     EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION      CONTAINER-RUNTIME
master  Ready    master   15h   v1.19.15+k3s2   192.168.1.45    <none>        Ubuntu 20.04.3 LTS   5.4.0-1045-raspi    containerd://1.4.11-k3s1
```

**2. Extend the cluster**

You can add additional hosts in order to expand available capacity.

```sh
k3sup join --ip <WORKER_X_IP> --server-ip <SERVER_IP> --user root --ssh-key <SSH_PATH>
```

Replace `<WORKER_X_IP>` with each worker IP address.

**3. Control plane node isolation: `taint`**

Unlike [k8s](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/#control-plane-node-isolation), the master node here is eligible to run containers destined for worker nodes as it does not have the `node-role.kubernetes.io/master=true:NoSchedule` taint that's typically present.

Tainting your master node is recommended to prevent workloads from being scheduled on it, unless you are only running a single-node k3s cluster on a Raspberry Pi.

```sh
kubectl taint nodes <SERVER_NAME> node-role.kubernetes.io/master=true:NoSchedule
```

Replace `<SERVER_NAME>` with your k3s server node `NAME` shown in the `kubectl get nodes` output.

**4. Optional labels**

By default, k3s does not label agent nodes with the `worker` role (unlike k8s). You can label them manually for clarity:

```sh
kubectl label node <WORKER_NAME> node-role.kubernetes.io/worker=''
```

Replace `<WORKER_NAME>` with the hostname of your nodes.

## 2. Install Ingress Controller

Deploy the custom Traefik v3 ingress controller from the official Helm chart repository. This replaces the default K3s Traefik installation (which we disabled in step 1) with a version we can configure.

```sh
kubectl apply -f traefik.yml
```

Verify that everything is working by running `kubectl get pods --all-namespaces`:

![image](https://user-images.githubusercontent.com/48289901/110213349-f4639980-7e9f-11eb-8a2d-e6a4c4720ebb.png)

Alternatively, browse to `http://<HOST_IP>/` — you should see a 404 page not found, which confirms Traefik is running and listening:

![image](https://user-images.githubusercontent.com/48289901/110215954-b0c35c80-7eac-11eb-8fcb-40ca50fce857.png)

## 3. Deploy a Dummy App

Deploy a dummy app, based on the `nicomincuzzi/go-webapp` image, and its service:

```sh
kubectl apply -f dummy_app.yml
```

Verify your app responds correctly:

```sh
kubectl port-forward pod/<POD_NAME> <YOUR_LOCAL_PORT>:<POD_PORT>
```

## 4. cert-manager

[cert-manager](https://cert-manager.io/docs/) is a native Kubernetes certificate management controller. It can help with issuing certificates from a variety of sources, such as Let's Encrypt, HashiCorp Vault, Venafi, a simple signing key pair, or self signed.

### Installing with Helm

As an alternative to the YAML manifests referenced above, we also provide an official Helm chart for installing cert-manager. Read more [here](https://cert-manager.io/docs/installation/kubernetes/#installing-with-helm).

#### Steps

In order to install the Helm chart, you must follow these steps:

Create the namespace for cert-manager:

```sh
kubectl create namespace cert-manager
```

Add the Jetstack Helm repository:

> **Warning:** It is important that this repository is used to install cert-manager. The version residing in the helm stable repository is deprecated and should not be used.

```sh
helm repo add jetstack https://charts.jetstack.io
```

Update your local Helm chart repository cache:

```sh
helm repo update
```

cert-manager requires a number of CRD resources to be installed into your cluster as part of installation.

This can either be done manually, using `kubectl`, or using the `installCRDs` option when installing the Helm chart.

> **Note:** If you're using a Helm version based on Kubernetes v1.18 or below (Helm v3.2), `installCRDs` will not work with cert-manager v0.16. For more info see the v0.16 upgrade notes.

**Option 1: installing CRDs with `kubectl`**

Install the CustomResourceDefinition resources using kubectl:

```sh
kubectl apply -f https://github.com/jetstack/cert-manager/releases/download/v1.2.0/cert-manager.crds.yaml
```

**Option 2: install CRDs as part of the Helm release**

To automatically install and manage the CRDs as part of your Helm release, you must add the `--set installCRDs=true` flag to your Helm installation command.

Uncomment the relevant line in the next steps to enable this.

To install the cert-manager Helm chart:

```sh
helm install \
  cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --version v1.2.0 \
  --create-namespace \
  --set installCRDs=true
```

The default `cert-manager` configuration is good for the majority of users, but a full list of the available options can be found in the Helm chart README.

### Verifying the installation

Once you've installed `cert-manager`, you can verify it is deployed correctly by checking the cert-manager namespace for running pods:

```sh
kubectl get pods --namespace cert-manager

NAME                                       READY   STATUS    RESTARTS   AGE
cert-manager-5c6866597-zw7kh               1/1     Running   0          2m
cert-manager-cainjector-577f6d9fd7-tr77l   1/1     Running   0          2m
cert-manager-webhook-787858fcdb-nlzsq      1/1     Running   0          2m
```

You should see the `cert-manager`, `cert-manager-cainjector`, and `cert-manager-webhook` pod in a Running state. It may take a minute or so for the TLS assets required for the webhook to function to be provisioned. This may cause the webhook to take a while longer to start for the first time than other pods. If you experience problems, please check the FAQ guide.

### Configuration

In order to configure cert-manager to begin issuing certificates, first `Issuer` or `ClusterIssuer` resources must be created. These resources represent a particular signing authority and detail how the certificate requests are going to be honored. You can read more on the concept of `Issuers` [here](https://cert-manager.io/docs/concepts/issuer/).

cert-manager supports multiple 'in-tree' issuer types that are denoted by being in the `cert-manager.io` group. cert-manager also supports external issuers that can be installed into your cluster that belong to other groups. These external issuer types behave no differently and are treated equal to in-tree issuer types.

When using `ClusterIssuer` resource types, ensure you understand the [Cluster Resource Namespace](https://cert-manager.io/docs/faq/cluster-resource/) where other Kubernetes resources will be referenced from.

Create `ClusterIssuer` resource:

```sh
kubectl apply -f cluster-issuer.yml
```

Verify that it's ready:

```sh
kubectl get clusterissuer
```

![image](https://user-images.githubusercontent.com/48289901/111067158-2dfd5b80-84c3-11eb-94e9-70b77d0fb2e2.png)

## 5. Expose App Externally via Ingress

Finally, expose your app externally by applying the Ingress resource. This creates routing rules that direct traffic from your domain to the app service, and triggers cert-manager to provision a TLS certificate.

```sh
kubectl apply -f ingress.yml
```

Verify the Ingress was created:

```sh
kubectl get ingress
```

Check that the TLS certificate has been issued:

```sh
kubectl get certificate
```

The certificate status should show `Ready: True` once Let's Encrypt has successfully issued it. This may take a minute or two.

## Troubleshooting

### cgroups not enabled on ARM / Raspberry Pi

If K3s fails to start on ARM hardware, `cgroups` may not be enabled. Edit your boot parameters:

```sh
sudo vi /boot/firmware/cmdline.txt
```

Add: `cgroup_enable=cpuset cgroup_memory=1 cgroup_enable=memory`

Reboot the device after saving.

### Certificate not becoming Ready

If `kubectl get certificate` shows the certificate is not ready:

1. Check the certificate status for details: `kubectl describe certificate <name>`
2. Check the challenge status: `kubectl get challenges`
3. Ensure your domain's DNS A record points to your server's public IP
4. Ensure port 80 is open and reachable from the internet (required for HTTP-01 challenge)

### Traefik returning 404 for all routes

- Verify the Ingress resource exists: `kubectl get ingress`
- Check the Ingress host matches your DNS: `kubectl describe ingress <name>`
- Ensure the backend Service and Pods are running: `kubectl get svc,pods`

## How to Contribute

Contributions are welcome! To get started:

1. Fork the repository
2. Create a feature branch: `git checkout -b feature/my-change`
3. Commit your changes: `git commit -m 'Add my change'`
4. Push to the branch: `git push origin feature/my-change`
5. Open a Pull Request

For bug reports or feature requests, please [open an issue](../../issues).

## License

Distributed under Apache-2.0 License, please see license file within the code for more details.
