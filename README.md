# Provision a GKE cluster using Crossplane running in a kind management cluster

## Important Links:
* [kind](https://kind.sigs.k8s.io/)
* [Crossplane](https://docs.crossplane.io/)

## Prerequisites: </br>
* [Docker Desktop](https://docs.docker.com/desktop/install/mac-install/) </br>
* [kind](https://github.com/kubernetes-sigs/kind?tab=readme-ov-file)
* [Helm](https://helm.sh/docs/intro/install/)
* [gcloud CLI](https://cloud.google.com/sdk/docs/install)
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
```bash
NAME                                       READY   STATUS    RESTARTS   AGE
crossplane-7b47779878-jv9kk                1/1     Running   0          115s
crossplane-rbac-manager-7f8b68c844-4lz2f   1/1     Running   0          115s
```

If you like, check out all of the custom resources that got added to your cluster as part of the Crossplane installation:
```bash
kubectl api-resources | grep crossplane
```
```bash
compositeresourcedefinitions        xrd,xrds     apiextensions.crossplane.io/v1         false        CompositeResourceDefinition
compositionrevisions                comprev      apiextensions.crossplane.io/v1         false        CompositionRevision
compositions                        comp         apiextensions.crossplane.io/v1         false        Composition
environmentconfigs                  envcfg       apiextensions.crossplane.io/v1alpha1   false        EnvironmentConfig
usages                                           apiextensions.crossplane.io/v1alpha1   false        Usage
configurationrevisions                           pkg.crossplane.io/v1                   false        ConfigurationRevision
configurations                                   pkg.crossplane.io/v1                   false        Configuration
controllerconfigs                                pkg.crossplane.io/v1alpha1             false        ControllerConfig
deploymentruntimeconfigs                         pkg.crossplane.io/v1beta1              false        DeploymentRuntimeConfig
functionrevisions                                pkg.crossplane.io/v1                   false        FunctionRevision
functions                                        pkg.crossplane.io/v1                   false        Function
locks                                            pkg.crossplane.io/v1beta1              false        Lock
providerrevisions                                pkg.crossplane.io/v1                   false        ProviderRevision
providers                                        pkg.crossplane.io/v1                   false        Provider
storeconfigs                                     secrets.crossplane.io/v1alpha1         false        StoreConfig
```

## Configure Google Cloud to allow Crossplane to create and manage GKE resources

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
```bash
apiVersion: pkg.crossplane.io/v1
kind: Provider
metadata:
  name: provider-gcp-container
spec:
  package: xpkg.upbound.io/upbound/provider-gcp-container:v1.7.0
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
```bash
clusters                                         container.gcp.upbound.io/v1beta2       false        Cluster
nodepools                                        container.gcp.upbound.io/v1beta2       false        NodePool
registries                                       container.gcp.upbound.io/v1beta1       false        Registry
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
```bash
apiVersion: gcp.upbound.io/v1beta1
kind: ProviderConfig
metadata:
  name: default
spec:
  projectID: <your $PROJECT_ID>
  credentials:
    source: Secret
    secretRef:
      namespace: crossplane-system
      name: gcp-secret
      key: creds
```

Let's apply it to the cluster.
```bash
kubectl apply -f crossplane-config/providerconfig.yaml
```
Great! Now we can use Crossplane and our local kind cluster to create a GKE cluster!

## Use Crossplane and our local kind cluster to create a GKE cluster

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
This is creating an external Google Cloud Kubernetes cluster, so it may take some minutes for the resources to become ready. Mine took about 12 minutes.
```bash
NAME                                                SYNCED   READY   EXTERNAL-NAME      AGE
cluster.container.gcp.upbound.io/newclusterwhodis   True     True    newclusterwhodis   12m

NAME                                                  SYNCED   READY   EXTERNAL-NAME       AGE
nodepool.container.gcp.upbound.io/newnodepoolwhodis   True     True    newnodepoolwhodis   12m
```

You made a Crossplane `Cluster` resource and a Crossplane `NodePool` resource! Let's view the manifests.
```bash
cat cluster-definitions/clusterandnodepool.yaml
```
```bash
apiVersion: container.gcp.upbound.io/v1beta1
kind: Cluster
metadata:
  name: newclusterwhodis
  labels:
   cluster-name: newclusterwhodis
spec:
  forProvider:
    deletionProtection: false
    removeDefaultNodePool: true
    initialNodeCount: 1
    location: us-central1-b

---

apiVersion: container.gcp.upbound.io/v1beta1
kind: NodePool
metadata:
  name: newnodepoolwhodis
  labels:
    cluster-name: newclusterwhodis
spec:
  forProvider:
    clusterSelector:
      matchLabels:
        cluster-name: newclusterwhodis
    nodeCount: 1
    nodeConfig:
      - preemptible: true
        machineType: e2-medium
        oauthScopes:
          - https://www.googleapis.com/auth/cloud-platform
        labels:
          heckyesyoudidit: yourefrigginawesome
```

You can `get` and `describe` your Crossplane `Cluster` and `NodePool` resources just like any other Kubernetes resource.

```bash
kubectl get cluster newclusterwhodis
```
```bash
NAME               SYNCED   READY   EXTERNAL-NAME      AGE
newclusterwhodis   True     True    newclusterwhodis   14m
```
```bash
kubectl describe nodepool newnodepoolwhodis
```
```bash
# This has too long of an output to display here. But you should run it!
```

## View your cluster in GCP

Let's view our newly created external resources! Which method do you prefer?
* [Google Cloud Console web UI](#View-your-new-cluster-in-the-Google-Cloud-Console-web-UI)
* [gcloud CLI](#View-your-new-cluster-via-the-gcloud-CLI)

## View your new cluster in the Google Cloud Console web UI
```bash
echo "https://console.cloud.google.com/kubernetes/list/overview?project=$PROJECT_ID"

# Open the URL from the output
```
Click around and try and find the Easter egg label set on the machine that our `NodePool` resource created!

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
```bash
NAME              LOCATION       MASTER_VERSION      MASTER_IP        MACHINE_TYPE  NODE_VERSION        NUM_NODES  STATUS
newclusterwhodis  us-central1-b  1.30.3-gke.1639000  104.155.132.170  e2-medium     1.30.3-gke.1639000  1          RUNNING
```

Describe the cluster that you and Crossplane made!
```bash
gcloud container clusters describe newclusterwhodis \
    --region us-central1-b
```

Explore! Try and find the Easter egg label set on the machine that our `NodePool` resource created!

Do you give up? Find more detailed instructions [here](easter-egg-hunt/gcloud-cli.md).

## Delete your local Crossplane resources, which will delete the GKE cluster

Because your GKE cluster is being managed by the instance of Crossplane that is running in your kind cluster, when you delete your local `newclusterwhodis` Crossplane `Cluster` resources and your `newnodepoolwhodis` Crossplane `NodePool` resource, it will also delete the associated GKE resources that are running in Google Cloud. 

But you don't have to take my word for it! Let's do it!

```bash
kubectl delete cluster newclusterwhodis

kubectl delete nodepool newnodepoolwhodis
```
In a few minutes, once the commands resolve, use your preferred method (web ui or CLI) to see that the GKE resources have disappeared.




TO BE CONTINUED IN PART 2 - Compositions


</br>
TODO: Add intro & outro

</br>
TODO: Add console screenshots

</br>
TODO: Complete Easter Egg solutions

</br>
Then it is finished!!!




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