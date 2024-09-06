# Provision a GKE cluster using Crossplane running in a kind management cluster

## Important Links:
* [kind](https://kind.sigs.k8s.io/)
* [Crossplane](https://docs.crossplane.io/)

## Prerequisites: </br>
* [Docker Desktop](https://docs.docker.com/desktop/install/mac-install/) </br>
* [kind](https://github.com/kubernetes-sigs/kind?tab=readme-ov-file)
* [Helm](https://helm.sh/docs/intro/install/)

## Create management cluster with kind
When you create your kind cluster, a `kubeconfig` file will get created which has your cluster connection credentials (among other things).

Tell kind to use this file to store its configuration -- `kubeconfig-kind.yaml`. That file has already been added to `.gitignore`. This will keep things tidy.

```bash
export KUBECONFIG=$PWD/kubeconfig-kind.yaml
```

Next, create a kind cluster. 

```bash
kind create cluster
```
## Install Crossplane
Install Crossplane into the kind cluster using Helm.

First, enable the Crossplane Helm Chart repository

```bash
helm repo add \
crossplane-stable https://charts.crossplane.io/stable
helm repo update
```

Then install the Crossplane components into a `crossplane-system` namespace.
```bash
helm install crossplane \
crossplane-stable/crossplane \
--namespace crossplane-system \
--create-namespace
```

Verify your Crossplane installation.
```bash
kubectl get pods -n crossplane-system
```

Check out all of the custom resources that got added to your cluster:
```bash
kubectl api-resources | grep crossplane
```
## Configure Crossplane to create and manage Google Cloud resources
