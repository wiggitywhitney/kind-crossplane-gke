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
* [yq](https://github.com/mikefarah/yq)

## Create a management cluster with kind
When you create your kind cluster, a `kubeconfig` file will get created which has your cluster connection credentials (among other things).

Tell kind to use this file to store its configuration -- `kubeconfig-kind-gke.yaml`. That file has already been added to `.gitignore`.

```bash
export KUBECONFIG=$PWD/kubeconfig-kind-gke.yaml
```

Next, create a kind cluster. 

```bash
kind create cluster
```
## Install Crossplane
Install Crossplane into the kind cluster using Helm.

First, enable the Crossplane Helm Chart repository.

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

Create a secret named `gcp-secret` that contains the GCP credentials that we just created and add it to the `crossplane-system` namespace.
```bash
kubectl --namespace crossplane-system \
    create secret generic gcp-secret \
    --from-file creds=./gcp-creds.json
```

To see your new secret, run the following:
```bash
kubectl get secret gcp-secret -n crossplane-system -o yaml
```

View the `provider-gcp-container` manifest. When applied, it will install the Crossplane infrastructure provider for GCP. 
```bash
cat crossplane-config/provider-gcp-container.yaml
```

Providers extend Crossplane by installing controllers for new kinds of managed resources.

Apply `provider-gcp-container` to your cluster to add [three new custom resource definitions](https://marketplace.upbound.io/providers/upbound/provider-gcp-container/v1.8.0) to your cluster. Each of these CRDs is called a `Managed Resource`, and each one is Crossplane's representation of a GCP resource. 

Once this Provider is installed, you will have the ability to manage external cloud resources via the Kubernetes API.
```bash
kubectl apply -f crossplane-config/provider-gcp-container.yaml
```
To your three new Crossplane custom resource definitions, run the following:
```bash
kubectl api-resources | grep "container.gcp.upbound.io"
```
(Spoiler alert: later we're going to create a `Cluster` resource!)

Next we need to teach Crossplane how to connect to our Google Cloud project with the permissions that we created in the last step. We do that using a Crossplane `ProviderConfig` resource. 

Run this command to add your project name to the `providerconfig.yaml` file that is already in this repo:
```bash
yq --inplace ".spec.projectID = \"$PROJECT_ID\"" crossplane-config/providerconfig.yaml
```

As you can see, our `ProviderConfig` references both our GCP project name and the `gcp-secret` Kubernetes secret that we created earlier.
```bash
cat crossplane-config/providerconfig.yaml
```
Let's apply it to the cluster.
```bash
kubectl apply -f crossplane-config/providerconfig.yaml
```
Great! Now we can use Crossplane and Kubernetes to create a GKE cluster!

## Use Crossplane and Kuberentes to create a GKE cluster

[API Documentation for the Crossplane `Cluster` Managed Resource](https://marketplace.upbound.io/providers/upbound/provider-gcp-container/v1.8.0/resources/container.gcp.upbound.io/Cluster/v1beta1)

Apply this minimal `Cluster` resource to make a GKE cluster!
```bash
kubectl apply -f cluster-definitions/bare-minimum.yaml
```

View the minimal `Cluster` manifest. See that it uses [GKE Autopilot](https://cloud.google.com/kubernetes-engine/docs/concepts/autopilot-overview) to manage your cluster configuration, so in this example, there are very few decisions that are made as part of the resource.

```bash
cat cluster-definitions/bare-minimum.yaml
```

You can `get` and `describe` your Crossplane `Cluster` resource just like any other Kubernetes resource.
```bash
kubectl get cluster heckyesyoudiditgoodjob
```
```bash
kubectl describe cluster heckyesyoudiditgoodjob
```
If you like, view your cluster in Google Cloud Console:
```bash
echo "https://console.cloud.google.com/kubernetes/list/overview?project=$PROJECT_ID"

# Open the URL from the output
```

Once your GKE cluster is ready, connect to it!

```bash
gcloud container clusters get-credentials heckyesyoudiditgoodjob --region us-central1 --project $PROJECT_ID

# The gke_gcloud_auth_plugin_cache file has been added to .gitignore
```
TO BE CONTINUED 



</br>
TODO: Show that you can manipulate the Cluster Crossplane resource and see the cluster change in GKE

```bash
```

## Resource Clean Up

Destroy kind cluster
```bash
kind delete cluster
```

Delete the GCP project
```bash
gcloud projects delete $PROJECT_ID --quiet
```

Delete the kubeconfig file
```bash
echo $KUBECONFIG

rm -rf $PWD/kubeconfig-kind-gke.yaml -i

## MAKE SURE THIS IS THE RIGHT FILE
```


