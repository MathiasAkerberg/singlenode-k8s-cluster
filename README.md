# Single-node Kubernetes Cluster 

This project shows how to run and manage a single-node Kubernetes cluster using RancherOS and RKE. 

## Prerequisites: 
* Optional Hypervisor, e.g. Hyper-V 
* [RKE](https://rancher.com/docs/rke/latest/en/installation/)
* [Helm](https://helm.sh/docs/intro/install/)
* [Kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/)

## Setup RancherOS VM

Download RancherOS ISO from the Rancher website and setup the `cloud-config.yml` according to the [docs](https://rancher.com/docs/os/v1.2/en/configuration/). 

From your VM, validate and install the cloud-config to disk:

```
sudo ros config validate -i ./cloud-config.yml
sudo ros install -c ./cloud-config.yml -d /dev/sda
```

### Init/Update RKE cluster

(Optional) Get the RKE CLI configuration to upgrade an existing cluster:

```
kubectl --kubeconfig kube_config_cluster-minimal.yml -n kube-system get cm full-cluster-state -o json | jq -r .data.\"full-cluster-state\" | jq -r . > cluster-minimal.rkestate
```

Apply latest changes to cluster (etcd, control plane, worker):

```
rke up --config rke/config-minimal.yml
```

## Setup Rancher Server

Create namespace:

```
kubectl --kubeconfig rke/kube_config_cluster-minimal.yml create namespace cattle-system
```

(Optional) Create self-signed cert:

```
openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout tls.key -out tls.crt -subj "/CN=<HOST>/O=<HOST>"
```
*Note:* See custom ingress tls configurations [here](https://rancher.com/docs/rancher/v2.x/en/installation/k8s-install/helm-rancher/)

Add cert to Rancher ingress:

```
kubectl --kubeconfig rke/kube_config_cluster-minimal.yml -n cattle-system create secret tls tls-rancher-ingress \
  --cert=tls.crt \
  --key=tls.key
```

Install Rancher Server in Kubernetes cluster:

```
helm repo add rancher-stable https://releases.rancher.com/server-charts/stable
helm repo update
helm install --kubeconfig rke/kube_config_cluster-minimal.yml --namespace cattle-system -f helm/rancher-server/values.yaml rancher-server rancher-stable/rancher --version 2.4.3
```

### Update existing instance

Modify configuration values in `helm/rancher-server/values.yaml` then apply changes:

```
helm upgrade --kubeconfig rke/kube_config_cluster-minimal.yml --namespace cattle-system -f helm/rancher-server/values.yaml rancher-server rancher-stable/rancher --version 2.4.3 --timeout 2m0s --wait --atomic
```