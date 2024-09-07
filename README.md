# Provision a GKE cluster using Crossplane running in a kind management cluster

## Important Links:
* [kind](https://kind.sigs.k8s.io/)
* [Crossplane](https://docs.crossplane.io/)

## Prerequisites: </br>
* [Docker Desktop](https://docs.docker.com/desktop/install/mac-install/) </br>
* [kind](https://github.com/kubernetes-sigs/kind?tab=readme-ov-file)
* [Helm](https://helm.sh/docs/intro/install/)
* [gcloud CLI](https://cloud.google.com/sdk/docs/install)
* [GKE gcloud auth plugin](https://cloud.google.com/blog/products/containers-kubernetes/kubectl-auth-changes-in-gke)

## Create a management cluster with kind
When you create your kind cluster, a `kubeconfig` file will get created which has your cluster connection credentials (among other things).

Tell kind to use this file to store its configuration -- `kubeconfig-kind.yaml`. That file has already been added to `.gitignore`.

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

If you like, check out all of the custom resources that got added to your cluster as part of the Crossplane installation:
```bash
kubectl api-resources | grep crossplane
```
## Configure Google Cloud to allow Crossplane to create and manage GKE resources

Enable authentication to Google Cloud via CLI.
```bash
export USE_GKE_GCLOUD_AUTH_PLUGIN=True
```

Create a Google Cloud project.
```bash
export PROJECT_ID=wiggity-$(date +%Y%m%d%H%M%S)

gcloud projects create $PROJECT_ID
```

Enable Google Cloud Kubernetes API.
```bash
echo "https://console.cloud.google.com/marketplace/product/google/container.googleapis.com?project=$PROJECT_ID"

# Open the URL from the output and enable the Kubernetes API
```

Create a Google Cloud Service Account named `wiggitywhitney`.
```bash
export SA_NAME=wiggitywhitney

export SA="${SA_NAME}@${PROJECT_ID}.iam.gserviceaccount.com"

gcloud iam service-accounts create $SA_NAME --project $PROJECT_ID
```

Bind the `wiggitywhitney` service account to an admin role.
```bash
export ROLE=roles/admin

gcloud projects add-iam-policy-binding --role $ROLE $PROJECT_ID --member serviceAccount:$SA
```

Create credentials in a `gcp-creds.json` file (already added to `.gitignore`).
```bash
gcloud iam service-accounts keys create gcp-creds.json --project $PROJECT_ID --iam-account $SA
```

## Configure Crossplane to create and manage GKE resources
TO BE CONTINUED

```bash
```


## Resource Clean Up

Destroy kind cluster
```bash
kind delete cluster
```

Delete the kind Kubeconfig file
```bash
echo $KUBECONFIG

## MAKE SURE THIS IS THE RIGHT FILE

rm -rf $PWD/kubeconfig-kind.yaml
```

Delete the GCP project
```bash
gcloud projects delete $PROJECT_ID --quiet
```

TODO: Delete GCP Kubeconfig, once there is one
```bash
```


