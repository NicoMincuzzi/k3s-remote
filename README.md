# K3S-remote

## Prerequisites

 - [kubectl]()
 - [k3sup](https://github.com/alexellis/k3sup)
 - [k9s](https://github.com/derailed/k9s)

## What's K3S?

Read [here](https://k3s.io/)

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
