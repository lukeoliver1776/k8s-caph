# Setting up a CAPI management cluster in Hetzner

## Overall Process

Boostrap kind cluster -> managment cluster in hetzner -> n number of workload clusters you want to manage

Loosely based on 

https://community.hetzner.com/tutorials/kubernetes-on-hetzner-with-cluster-api

and

https://syself.com/docs/caph/getting-started/quickstart/management-cluster-setup


## Set up Hetzner

### Copy the example env file 

The `mgmt.env` file gives an example configuration for your management cluster

```
cp mgmt.env .env
```

### Create a project in Hetzner


Create a project in hetzner to provide for seperation of interests. We called it "mgmt cluster".

### Security

#### SSH Keys

Create ssh key and apply it to the project in the security section of the dashboard. 


```_
ssh-keygen -t ed25519 -C "mgmt-cluster" -f ~/.ssh/mgmt-cluster-ssh-key
```

Add the public key into the SSH Keys area of the Security dashboardh

 ```
 cat ~/.ssh/mgmt-cluster-ssh-key.pub
 ```

#### Create API Token

While still inside the security panel, click the API Tokens link.

Create an API token called mgmt-cluster and give it read/write permissions.

Place the created token into the .env file using the HCLOUD_TOKEN variable.


## Set up developer machine

Install pre-requisite software

```
brew install kubectl kind clusterctl helm hcloud kubectx kubernetes-cli
```

If you have some of these components installed already, makek sure they are the most recent version

```
brew upgrade
```

## Active the variables in env

Turn your .env file into real exported variables accessable to our system

```
set -a
source .env
set +a
```


## Create and boostrap a local management cluster

Create a local bootstrap cluster using kind.


```
kind create cluster --name bootstrap-cluster

```

Now install CAPI into the newly created boostrap-cluster. This basically turns the bootstrap cluster into a temporary capi management cluster.

```
clusterctl init --core cluster-api --bootstrap kubeadm --control-plane kubeadm --infrastructure hetzner

```

Creating a secret based on the api token we created
```
kubectl create secret generic hetzner --from-literal=hcloud=$HCLOUD_TOKEN
```

Add a special label to allow this secret to be moved when we pivot our mgmt cluster.
```
kubectl patch secret hetzner -p '{"metadata":{"labels":{"clusterctl.cluster.x-k8s.io/move":""}}}'
```

## Generate manifests for management cluster

Generate the manifests for your management cluster. To see all of the required and optional variables run this command

```
clusterctl generate cluster my-cluster --list-variables
```

Now create the manifest files for our management cluster

```
clusterctl generate cluster mgmt-cluster --kubernetes-version v1.31.6 --control-plane-machine-count=3 --worker-machine-count=1  > mgmt.yaml
```

IGNORE FOR NOW: remove cloud-provider from extraargs in the resulting manifest file.

Then apply the cluster manifests

```
kubectl apply -f mgmt.yaml
```

Once the load balancer becomes healty, get the kubeconfig and store it in /tmp
```
clusterctl get kubeconfig mgmt-cluster > /tmp/mgmt-cluster

```


Install Hetzner CCM and CSI

```
KUBECONFIG=/tmp/mgmt-cluster helm repo add hcloud https://charts.hetzner.cloud
KUBECONFIG=/tmp/mgmt-cluster helm repo update hcloud

KUBECONFIG=/tmp/mgmt-cluster helm upgrade --install hccm hcloud/hcloud-cloud-controller-manager \
        --namespace kube-system \
        --set env.HCLOUD_TOKEN.valueFrom.secretKeyRef.name=hetzner \
        --set env.HCLOUD_TOKEN.valueFrom.secretKeyRef.key=hcloud


KUBECONFIG=/tmp/mgmt-cluster helm upgrade --install hcloud-csi hcloud/hcloud-csi \
        --namespace kube-system \
        --set env.HCLOUD_TOKEN.valueFrom.secretKeyRef.name=hetzner \
        --set env.HCLOUD_TOKEN.valueFrom.secretKeyRef.key=hcloud

```
validate csi

```
KUBECONFIG=/tmp/mgmt-cluster k get csidrivers
```


Install the Flannel CNI

```
KUBECONFIG=/tmp/mgmt-cluster kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
```

Now turn this newly created cluster into our management cluster where we can create n number of workload clusters

```
KUBECONFIG=/tmp/mgmt-cluster clusterctl init --core cluster-api --bootstrap kubeadm --control-plane kubeadm --infrastructure hetzner
```

Now move the cluster definitions to the new mgmt cluster

```
clusterctl move --to-kubeconfig=/tmp/mgmt-cluster --namespace=default

```
delete the bootstrap cluster
```
kind delete cluster --name capi-mgmt
```



## Generate a cluster manifest for our dev cluster

```
KUBECONFIG=/tmp/mgmt-cluster clusterctl generate cluster dev-cluster --kubernetes-version v1.31.6 --control-plane-machine-count=3 --worker-machine-count=3 -n dev-cluster > dev.yaml
```

Create the namespace
```
KUBECONFIG=/tmp/mgmt-cluster kubectl create ns dev-cluster
```


Creating a secret based on the api token we created
```
KUBECONFIG=/tmp/mgmt-cluster kubectl create secret -n dev-cluster generic hetzner --from-literal=hcloud=$HCLOUD_TOKEN
```

Add a special label to allow this secret to be moved when we pivot our mgmt cluster.
```
KUBECONFIG=/tmp/mgmt-cluster  kubectl patch secret -n dev-cluster hetzner -p '{"metadata":{"labels":{"clusterctl.cluster.x-k8s.io/move":""}}}'
```
Apply the dev cluster yaml


```

KUBECONFIG=/tmp/mgmt-cluster kubectl apply -f dev.yaml

```


Get the k8s config for the dev cluster

```
KUBECONFIG=/tmp/mgmt-cluster clusterctl -n dev-cluster get kubeconfig dev-cluster > /tmp/dev-cluster
```

## Install Hetzner helper software

Install Hetzner CCM and CSI

```
KUBECONFIG=/tmp/dev-cluster helm repo add hcloud https://charts.hetzner.cloud
KUBECONFIG=/tmp/dev-cluster helm repo update hcloud

KUBECONFIG=/tmp/dev-cluster helm upgrade --install hccm hcloud/hcloud-cloud-controller-manager \
        --namespace kube-system \
        --set env.HCLOUD_TOKEN.valueFrom.secretKeyRef.name=hetzner \
        --set env.HCLOUD_TOKEN.valueFrom.secretKeyRef.key=hcloud


KUBECONFIG=/tmp/dev-cluster helm upgrade --install hcloud-csi hcloud/hcloud-csi \
        --namespace kube-system \
        --set env.HCLOUD_TOKEN.valueFrom.secretKeyRef.name=hetzner \
        --set env.HCLOUD_TOKEN.valueFrom.secretKeyRef.key=hcloud
```

validate csi

```
KUBECONFIG=/tmp/dev-cluster k get csidrivers
```


Install the Flannel CNI

```
KUBECONFIG=/tmp/dev-cluster k apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
```


