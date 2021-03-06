# K3S-remote

## Prerequisites

 - [kubectl]()
 - [k3sup](https://github.com/alexellis/k3sup)
 - [k9s](https://github.com/derailed/k9s)
 - [Helm v3](https://helm.sh/docs/intro/install/)

## What's K3S?

Find [here](https://k3s.io/) all you need

## Install K3S by k3sup

```sh
export IP=<HOST_IP>
k3sup install \
  --ip $IP \
  --user root \
  --ssh-key <SSH_PATH> \
  --merge 
  --local-path $HOME/.kube/config 
  --context my-k8s 
  --k3s-extra-args '--no-deploy traefik'
```

Options for `install`:

* `--cluster` - start this server in clustering mode using embdeed etcd (embedded HA)
* `--skip-install` - if you already have k3s installed, you can just run this command to get the `kubeconfig`
* `--ssh-key` - specify a specific path for the SSH key for remote login
* `--local-path` - default is `./kubeconfig` - set the file where you want to save your cluster's `kubeconfig`.  By default this file will be overwritten.
* `--merge` - Merge config into existing file instead of overwriting (e.g. to add config to the default kubectl config, use `--local-path ~/.kube/config --merge`).
* `--context` - default is `default` - set the name of the kubeconfig context.
* `--ssh-port` - default is `22`, but you can specify an alternative port i.e. `2222`
* `--k3s-extra-args` - Optional extra arguments to pass to k3s installer, wrapped in quotes, i.e. `--k3s-extra-args '--no-deploy traefik'` or `--k3s-extra-args '--docker'`. For multiple args combine then within single quotes `--k3s-extra-args '--no-deploy traefik --docker'`.
* `--k3s-version` - set the specific version of k3s, i.e. `v0.9.1`
- `--ipsec` - Enforces the optional extra argument for k3s: `--flannel-backend` option: `ipsec`
* `--print-command` - Prints out the command, sent over SSH to the remote computer
* `--datastore` - used to pass a SQL connection-string to the `--datastore-endpoint` flag of k3s. You must use [the format required by k3s in the Rancher docs](https://rancher.com/docs/k3s/latest/en/installation/ha/).

See even more install options by running `k3sup install --help`.

**NOTE**

Traefik can be configured by editing the `traefik.yaml` file. To prevent k3s from using or overwriting the modified version, deploy k3s with `--no-deploy traefik` and store the modified copy in the `k3s/server/manifests directory`. For more information, refer to the official [Traefik for Helm Configuration Parameters](https://github.com/helm/charts/tree/master/stable/traefik#configuration).

## Install Ingress Controller

`kubectl apply -f ./traefik.yml`

Verify that all is right by running `kubectl get pods --all-namespaces`

![image](https://user-images.githubusercontent.com/48289901/110213349-f4639980-7e9f-11eb-8a2d-e6a4c4720ebb.png)

In alternatives, browse to http://<HOST_IP>/ you should see a 404 page not found:

![image](https://user-images.githubusercontent.com/48289901/110215954-b0c35c80-7eac-11eb-8fcb-40ca50fce857.png)

## Deploy a dummy app

Deploy a dummy app, based on nginx image, and service by running:

`kubectl apply -f app.yml`

## cert-manager

[cert-manager](https://cert-manager.io/docs/) is a native Kubernetes certificate management controller. It can help with issuing certificates from a variety of sources, such as Let’s Encrypt, HashiCorp Vault, Venafi, a simple signing key pair, or self signed.

### Installing with Helm
As an alternative to the YAML manifests referenced above, we also provide an official Helm chart for installing cert-manager. Read more [here](https://cert-manager.io/docs/installation/kubernetes/#installing-with-helm).

#### Steps
In order to install the Helm chart, you must follow these steps:

Create the namespace for cert-manager:

`$ kubectl create namespace cert-manager`

Add the Jetstack Helm repository:

> Warning: It is important that this repository is used to install cert-manager. The version residing in the helm stable repository is deprecated and should not be used.

`$ helm repo add jetstack https://charts.jetstack.io`
  
Update your local Helm chart repository cache:

`$ helm repo update`

cert-manager requires a number of CRD resources to be installed into your cluster as part of installation.

This can either be done manually, using `kubectl`, or using the `installCRDs` option when installing the Helm chart.

> **Note**: If you’re using a helm version based on Kubernetes v1.18 or below (Helm v3.2) installCRDs will not work with cert-manager v0.16. For more info see the v0.16 upgrade notes

**Option 1: installing CRDs with `kubectl`**

Install the CustomResourceDefinition resources using kubectl:

`$ kubectl apply -f https://github.com/jetstack/cert-manager/releases/download/v1.2.0/cert-manager.crds.yaml`

**Option 2: install CRDs as part of the Helm release**

To automatically install and manage the CRDs as part of your Helm release, you must add the `--set installCRDs=true` flag to your Helm installation command.

Uncomment the relevant line in the next steps to enable this.

To install the cert-manager Helm chart:

```sh
$ helm install \
  cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --version v1.2.0 \
  --create-namespace \
  --set installCRDs=true
```

The default `cert-manager` configuration is good for the majority of users, but a full list of the available options can be found in the Helm chart README.

