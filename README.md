# K3S-remote

A guide to install K3s with a custom Traefik Ingress Control.

## Prerequisites

 - [kubectl]()
 - [k3sup](https://github.com/alexellis/k3sup)
 - [k9s](https://github.com/derailed/k9s)
 - [Helm v3](https://helm.sh/docs/intro/install/)

## What's K3S?

Find [here](https://k3s.io/) all you need

## 1. Install K3S by k3sup

```sh
export IP=<HOST_IP>
k3sup install \
  --ip $IP \
  --user root \
  --ssh-key <SSH_PATH> \
  --merge \
  --local-path $HOME/.kube/config \
  --context my-k8s \
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

[Traefik](https://github.com/traefik/traefik) can be configured by editing the `traefik.yaml` file. To prevent k3s from using or overwriting the modified version, deploy k3s with `--no-deploy traefik` and store the modified copy in the `k3s/server/manifests directory`. For more information, refer to the official [Traefik for Helm Configuration Parameters](https://github.com/helm/charts/tree/master/stable/traefik#configuration).

### Extra

Let's build a 3-node Kubernetes cluster with Rancher's k3s project and k3sup, which uses `ssh` to make the whole process quick and painless.

Running k3s/MicroK8s on some ARM hardware may run into difficulties because `cgroups` are not enabled by default. 
- This can be remedied on the Rasberry Pi by editing the boot parameters:

  `sudo vi /boot/firmware/cmdline.txt`

**Note:** In some *Raspberry Pi Linux distributions* the boot parameters are in `/boot/firmware/nobtcmd.txt`.

And adding the following:

`cgroup_enable=cpuset cgroup_memory=1 cgroup_enable=memory` to address disk performance issues often present on Raspberry Pi see the troubleshooting section.

More details are available at: https://microk8s.io/docs/install-alternatives#heading--arm

**Create the server**
In Kubernetes terminology, the server is often called the master.

```sh
$ export IP=<HOST_IP>

$ k3sup install \
  --ip $IP \
  --user root \
  --ssh-key <SSH_PATH> \
  --merge \
  --local-path $HOME/.kube/config \
  --context my-k8s \
  --k3s-extra-args '--no-deploy traefik'
```

k3s is so fast to start up, that it may be ready for use after the command has completed.

Test it out:
```sh
$ export KUBECONFIG=`pwd`/kubeconfig

$ kubectl get node -o wide
NAME    STATUS   ROLES    AGE   VERSION         INTERNAL-IP     EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION      CONTAINER-RUNTIME
master  Ready    master   15h   v1.19.15+k3s2   192.168.1.45    <none>        Ubuntu 20.04.3 LTS   5.4.0-1045-raspi    containerd://1.4.11-k3s1
```

**Extend the cluster**
You can add additional hosts in order to expand our available capacity.

Run the following:
```sh
$ k3sup join --ip <WORKER_1_IP> --server-ip <SERVER_IP> --user root --ssh-key <SSH_PATH>

$ k3sup join --ip <WORKER_2_IP> --server-ip <SERVER_IP> --user root --ssh-key <SSH_PATH>
```

```sh
kubectl taint nodes <SERVER_NAME> node-role.kubernetes.io/master=true:NoSchedule
```

## 2. Install Ingress Controller

```sh
$ kubectl apply -f ./traefik.yml
```

Verify that all is right by running `kubectl get pods --all-namespaces`

![image](https://user-images.githubusercontent.com/48289901/110213349-f4639980-7e9f-11eb-8a2d-e6a4c4720ebb.png)

In alternatives, browse to http://<HOST_IP>/ you should see a 404 page not found:

![image](https://user-images.githubusercontent.com/48289901/110215954-b0c35c80-7eac-11eb-8fcb-40ca50fce857.png)

## 3. Deploy a dummy app

Deploy a dummy app, based on `nicomincuzzi/go-webapp` image, and service by running:

```sh
$ kubectl apply -f dummy_app.yml
```

Verify your app responds correctly:

```sh
$ kubectl port-forward pod/<POD_NAME> <YOUR_LOCAL_PORT>:<POD_PORT>
```

## 4. cert-manager

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

### Verifying the installation

Once you’ve installed `cert-manager`, you can verify it is deployed correctly by checking the cert-manager namespace for running pods:

```sh
$ kubectl get pods --namespace cert-manager

NAME                                       READY   STATUS    RESTARTS   AGE
cert-manager-5c6866597-zw7kh               1/1     Running   0          2m
cert-manager-cainjector-577f6d9fd7-tr77l   1/1     Running   0          2m
cert-manager-webhook-787858fcdb-nlzsq      1/1     Running   0          2m
```

You should see the `cert-manager`, `cert-manager-cainjector`, and `cert-manager-webhook` pod in a Running state. It may take a minute or so for the TLS assets required for the webhook to function to be provisioned. This may cause the webhook to take a while longer to start for the first time than other pods. If you experience problems, please check the FAQ guide.

### Configuration
In order to configure cert-manager to begin issuing certificates, first `Issuer` or `ClusterIssuer` resources must be created. These resources represent a particular signing authority and detail how the certificate requests are going to be honored. You can read more on the concept of `Issuers` [here](https://cert-manager.io/docs/concepts/issuer/).

cert-manager supports multiple ‘in-tree’ issuer types that are denoted by being in the `cert-manager.io` group. cert-manager also supports external issuers than can be installed into your cluster that belong to other groups. These external issuer types behave no different and are treated equal to in tree issuer types.

When using `ClusterIssuer` resource types, ensure you understand the [Cluster Resource Namespace](https://cert-manager.io/docs/faq/cluster-resource/) where other Kubernetes resources will be referenced from.

Create `ClusterIssuer` resource:

`kubectl apply -f cluster-issuer.yml`

Verify that all it's ok running:

`kubectl get clusterissuer`

![image](https://user-images.githubusercontent.com/48289901/111067158-2dfd5b80-84c3-11eb-94e9-70b77d0fb2e2.png)

## 5. Expose app to extern via Ingress

Finally, expose your app to extern running the following command:

`kubectl apply -f ingress.yml`

## How to Contribute
Make a pull request...

## License
Distributed under Apache-2.0 License, please see license file within the code for more details.
