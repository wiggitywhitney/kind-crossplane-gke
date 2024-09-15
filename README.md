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

gcloud iam service-accounts create $SA_NAME \
    --project $PROJECT_ID
```

Bind the `wiggitywhitney` service account to an admin role.
```bash
export ROLE=roles/admin

gcloud projects add-iam-policy-binding \
    --role $ROLE $PROJECT_ID \
    --member serviceAccount:$SA
```

Create credentials in a `gcp-creds.json` file (already added to `.gitignore`).
```bash
gcloud iam service-accounts keys create gcp-creds.json \
    --project $PROJECT_ID \
    --iam-account $SA
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
kubectl --namespace crossplane-system \
    get secret gcp-secret \
    --output yaml
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
(Spoiler alert: later we're going to create `Cluster` and `NodePool` resources!)

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

* [API Documentation for the Crossplane GCP `Cluster` Managed Resource](https://marketplace.upbound.io/providers/upbound/provider-gcp-container/v1.8.0/resources/container.gcp.upbound.io/Cluster/v1beta1)
* [API Documentation for the Crossplane GCP `NodePool` Managed Resource](https://marketplace.upbound.io/providers/upbound/provider-gcp-container/v1.8.0/resources/container.gcp.upbound.io/NodePool/v1beta1)

Apply this minimal `clusterandnodepool` resource definition to make a GKE cluster!
```bash
kubectl apply -f cluster-definitions/clusterandnodepool.yaml
```

To see the Crossplane resources that got created, run the following:
```bash
kubectl get managed
```

You made a Crossplane `Cluster` resource and a Crossplane `NodePool` resource! 

```bash
cat cluster-definitions/clusterandnodepool.yaml
```

You can `get` and `describe` your Crossplane `Cluster` and `NodePool` resources just like any other Kubernetes resource.

```bash
kubectl get cluster newclusterwhodis
```
```bash
kubectl describe nodepool newnodepoolwhodis
```

## View your cluster in GCP

Let's view our newly created external resources! Which method do you prefer?
* [Google Cloud Console](#View-your-new-cluster-in-the-Google-Cloud-Console-web-UI) web UI
* [gcloud CLI](#View-your-new-cluster-via-the-gcloud-CLI)

## View your new cluster in the Google Cloud Console web UI
```bash
echo "https://console.cloud.google.com/kubernetes/list/overview?project=$PROJECT_ID"

# Open the URL from the output
```
Click around and try and find the easter egg label set on the machine that our `NodePool` resource created!

Do you give up? Find more detailed instructions [here](easter-egg-hunt/gcp-console-ui.md).

## View your new cluster via the gcloud CLI

If needed, authorize `gcloud` to access the Google Cloud Platform:
```bash
gcloud auth login
```

Set the `project` property in your gcloud configuration
```bash
gcloud config set project $PROJECT_ID
```

See the cluster that you and Crossplane made!
```bash
gcloud container clusters list
```

Describe the cluster that you and Crossplane made!
```bash
gcloud container clusters describe newclusterwhodis \
    --region us-central1-b
```

Explore! Try and find the easter egg label set on the machine that our `NodePool` resource created!

Do you give up? Find more detailed instructions [here](easter-egg-hunt/gcloud-cli.md).


TO BE CONTINUED IN PART 2 - Compositions

</br>

TODO: GCP Container Provider 1.8 has a [bug](https://github.com/crossplane-contrib/provider-upjet-gcp/issues/607) where default nodepool isn't being deleted. Refactor to use provider v1.7?

</br>
TODO: Delete Crossplane resources & see that the corresponding GKE resources disappear

</br>
TODO: Is USE_GKE_GCLOUD_AUTH_PLUGIN is necessary any more? Try without it.

</br>
TODO: Add intro & outro

</br>
TODO: Add command outputs

</br>
TODO: Add console screenshots

</br>
TODO: Complete Easter Egg solutions

</br>
Then it is finished!!!


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

rm -rf -i $PWD/kubeconfig-kind.yaml

## MAKE SURE THIS IS THE RIGHT FILE
```