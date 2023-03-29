### Create an Anthos Team Cluster and Namespace

Link: https://cloud.google.com/anthos/fleet-management/docs/team-management

0. Create cluster in GUI with default settings

0. Register cluster in Anthos

0. Initial setup stuff:
      ```
      gcloud projects add-iam-policy-binding PROJECT_ID \
         --member user:USERNAME@YOUR_DOMAIN \
         --role='roles/gkehub.admin'
      gcloud services enable --project=PROJECT_ID \
         gkehub.googleapis.com \
         container.googleapis.com \
         connectgateway.googleapis.com \
         cloudresourcemanager.googleapis.com \
         iam.googleapis.com \
         anthos.googleapis.com
      ```

0. Create scope and add cluster to it
      ```
      gcloud alpha container fleet scopes create scope-1
      gcloud alpha container fleet memberships bindings create cluster-1-scope-1 \
         --membership cluster-1 \
         --scope  scope-1 \
         --location global
      ```

0. Create a namespace in the scope that will be added to all clusters in the scope
      ```
      gcloud alpha container fleet namespaces create test-1 --scope=scope-1
      ```

0. Create a google group for Team 1 called `PROJECT_ID-team-1@YOUR_DOMAIN`

0. Add `PROJECT_ID-team-1@YOUR_DOMAIN` to the `gke-security-groups@YOUR_DOMAIN` group

0. Add the `gke-security-groups@YOUR_DOMAIN` to the cluster
      ```
      gcloud container clusters update cluster-1 \
         --region=us-central1-c \
         --security-group="gke-security-groups@YOUR_DOMAIN"
      ```

0. Give the `PROJECT_ID-team-1@YOUR_DOMAIN` group proper permissions to access the cluster
      ```
      gcloud projects add-iam-policy-binding PROJECT_ID \
         --member=group:PROJECT_ID-team-1@YOUR_DOMAIN \
         --role=roles/gkehub.viewer
      gcloud projects add-iam-policy-binding PROJECT_ID \
         --member=group:PROJECT_ID-team-1@YOUR_DOMAIN \
         --role=roles/gkehub.gatewayEditor
      gcloud alpha container fleet namespaces rbacrolebindings create cluster-1-scope-1 \
         --namespace test-1 \
         --role=admin \
         --group=PROJECT_ID-team-1@YOUR_DOMAIN
      ```

0. Create `user-a@YOUR_DOMAIN` and add them to the `PROJECT_ID-team-1@YOUR_DOMAIN` group

0. Open an incognito window, log into console.cloud.google.com as `user-a@YOUR_DOMAIN`, open the Cloud Shell and run: 
      ```
      gcloud config set project PROJECT_ID
      gcloud container fleet memberships get-credentials cluster-1
      ```

0. Try to deploy an app to the default namespace (it will fail):
      ```
      kubectl apply -f hello-world.yaml -n default
      ```

0. Try to deploy an app to the test-1 namespace (it will succeed):
      ```
      kubectl apply -f hello-world.yaml -n test-1
      ```

### Create Anthos Team Namespace Across Projects

Link: https://cloud.google.com/anthos/fleet-management/docs/before-you-begin/gke#gke-cross-project

0. Create a new project

0. In the new project, create cluster in GUI with default settings and enable Workload Identity (different name from first cluster)

0. Verify Workload Identity is installed `gcloud container clusters describe cluster-3 --format="value(workloadIdentityConfig.workloadPool)" --zone=us-central1-c`

0. In the fleet hub project, ensure GKE Hub service account has been provisioned in host project `gcloud projects get-iam-policy HUB_PROJECT_NAME | grep gcp-sa-gkehub`

0. In the new project, give the GKE Hub service account the proper permissions to connect the cluster to the hub fleet
    ```
    gcloud projects add-iam-policy-binding NEW_PROJECT_NAME \
        --member=serviceAccount:SVC_ACCOUNT_NAME@gcp-sa-gkehub.iam.gserviceaccount.com \
        --role=roles/gkehub.serviceAgent
    ```

0. Register the cluster to the hub project
    ```
    gcloud container fleet memberships register NEW_CLUSTER_NAME \
        --gke-uri=//container.googleapis.com/projects/NEW_PROJECT_NAME/locations/ZONE/clusters/NEW_CLUSTER_NAME \
        --project=HUB_PROJECT_NAME
    ```
0. Add the new cluster to scope-1 to provision the test-1 namespace automatically
    ```
    gcloud alpha container fleet memberships bindings create cluster-2-scope-1 \
         --membership cluster-2 \
         --scope  scope-1 \
         --location global
    ```
0. Run the following kubectl command and the namespace should pop up in a few moments
    ```
    kubectl get namespace --watch
    ```