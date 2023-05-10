# Introducing the ArgoCD Application Pull Controller for Open Cluster Management

Authors: Mike Ng, Christian Stark

Abstract:

Argo CD is a CNCF project that utilizes a `GitOps approach` for managing and deploying applications on Kubernetes clusters. On the other hand, Open Cluster Management (OCM) is a CNCF Sandbox project that focuses on managing a fleet of Kubernetes clusters at scale.

Argo CD currently utilizes a push model architecture where workload is pushed from a centralized cluster to remote clusters, requiring a connection between the control-plane and the remote destination. However, the pull model offers several advantages over the push model, and `Open Cluster Management's architecture` aligns better with the pull model.

One advantage of the pull model is decentralized control, where each cluster has its own copy of the configuration and is responsible for pulling updates on its own. This eliminates the need for a centralized system to be aware of all the target clusters and their configurations, making the system more scalable and easier to manage. Additionally, the pull model offers better security by reducing the risk of unauthorized access and eliminating the need for remote cluster credentials to be stored in a centralized environment. The pull model also provides more flexibility, allowing clusters to pull updates on their own schedule and reducing the risk of conflicts or disruptions.

The goal of this blog is to highlight the benefits of using a pull model for managing multiple Kubernetes clusters in a CD system like ArgoCD. It also explains how to enhance Argo CD without significant code changes to existing applications and how to migrate existing OCM AppSubscription users to ArgoCD. The blog mainly focuses on the upstream version of RHACM but also explains how to use it with RHACM 2.8, where the pull model feature will be introduced as Tech-Preview.

## OCM key components used in ArgoCD pull model Integration

### Managed Clusters

Here you get a list of Managed-Clusters

```
oc get managedclusters -l cloud=AWS
NAME                        HUB ACCEPTED     JOINED   AVAILABLE   AGE
demo-managed-0   true                         True         True              17h
demo-managed-1   true                         True         True              16h
```

### Placement: dynamically select a set of managed clusters


See [here](https://open-cluster-management.io/concepts/placement/) for a detailed description on the `Placement-CRD` which is a central component of the MultiCluster solution.

## Architecture and Dependencies

This ArgoCD pull model controller on the Hub cluster will create `ManifestWork objects` wrapping `Application objects` as payload.
See [here](https://open-cluster-management.io/concepts/manifestwork/) for more info regarding ManifestWork which a central concepts for delivering workloads to the Spoke-Clusters.
The `Open Cluster Management` agent on the Managed cluster will notice the ManifestWork on the Hub cluster and pull the Application from there.


## Setting up the solution using Open Cluster Management (OCM) 

The Open Cluster Management (OCM) multi-cluster environment needs to be setup. 
Note: Currently OCM as the upstream version of RHACM does not have a UI.

See the [OCM website](https://open-cluster-management.io/getting-started/quick-start/#setup-hub-and-managed-cluster) on how to set up a quick demo environment based on kind.

In the overall solution OCM-Hub will provide the cluster inventory and ability to deliver workload to the remote/managed clusters.

The Hub cluster and remote/managed clusters need to have ArgoCD Application installed. See the [ArgoCD website](https://argo-cd.readthedocs.io/en/stable/) for more details.


#### ArgoCD-Installation:

```
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

Set the property [skip-reconcile](https://argo-cd.readthedocs.io/en/latest/user-guide/skip_reconcile/). 

```
metadata:
  annotations:
    argocd.argoproj.io/skip-reconcile: "true"
```

Clone this project and connect to the Hub cluster and start the Pull controller:

```
git clone https://github.com/open-cluster-management-io/argocd-pull-integration
cd argocd-pull-integration
export KUBECONFIG=/path/to/<hub-kubeconfig>
make deploy
```

If your controller starts successfully, you should see:

```
$ kubectl -n argocd get deploy | grep pull
argocd-pull-integration-controller-manager   1/1     1            1           106s
```

On the Hub cluster, create an ArgoCD cluster secret that represents the managed cluster. 
Note replace the cluster-name with the registered managed cluster name.

```
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Secret
metadata:
  name: <cluster-name>-secret # cluster1-secret
  namespace: argocd
  labels:
    argocd.argoproj.io/secret-type: cluster
type: Opaque
stringData:
  name: <cluster-name> # cluster1
  server: https://<cluster-name>-control-plane:6443 # https://cluster1-control-plane:6443
EOF
```

On the Hub cluster, apply the manifests in example/hub:

```
kubectl apply -f example/hub
```

On the Managed cluster, apply the manifests in example/managed:

```
kubectl apply -f example/managed
```

On the Hub cluster, apply the guestbook-app-set manifest:


```
kubectl apply -f example/guestbook-app-set.yaml
```

Note: The Application template inside the ApplicationSet must contain the following content:
```     
     labels:
        apps.open-cluster-management.io/pull-to-ocm-managed-cluster: 'true'
      annotations:
        argocd.argoproj.io/skip-reconcile: 'true'
        apps.open-cluster-management.io/ocm-managed-cluster: '{{name}}'
```

The label allows the pull model controller to select the Application for processing.
The `skip-reconcile` annotation is to prevent the Application from reconciling on the Hub cluster.
The `ocm-managed-cluster` annotation is for the ApplicationSet to generate multiple Application based on each cluster generator targets.
When this guestbook ApplicationSet reconciles, it will generate an Application for the registered ManagedCluster. 
For example:

```
$ kubectl -n argocd get appset
NAME            AGE
guestbook-app   84s
$ kubectl -n argocd get app
NAME                     SYNC STATUS   HEALTH STATUS
cluster1-guestbook-app     
```

On the Hub cluster, the pull controller will wrap the Application with a ManifestWork. For example:

```
$ kubectl -n cluster1 get manifestwork
NAME                          AGE
cluster1-guestbook-app-d0e5   2m41s
On the Managed cluster, you should see the Application is pulled down successfully. For example:
$ kubectl -n argocd get app
NAME                     SYNC STATUS   HEALTH STATUS
cluster1-guestbook-app   Synced        Healthy
$ kubectl -n guestbook get deploy
NAME           READY   UP-TO-DATE   AVAILABLE   AGE
guestbook-ui   1/1     1            1           7m36s
```

On the Hub cluster, the status controller will sync the Application with the ManifestWork status feedback. For example:

```
$ kubectl -n argocd get app
NAME                     SYNC STATUS   HEALTH STATUS
cluster1-guestbook-app   Synced        Healthy
```


Setting up instructions for RHACM 2.8

1. Install OpenShift GitOps operator on the hub and all the target managed clusters. We recommend the installed namespace: `openshift-gitops`.


2. Every managed cluster needs to have a cluster secret in the ArgoCD server namespace on the hub cluster. This is required by the ArgoCD application set controller to 
   propagate the ArgoCD application template for a managed cluster. We recommend that users create a gitOpsCluster resource that contains a reference to a placement resource. The placement resource selects all the managed clusters that need to support the pull model.  As a result, managed cluster secrets will be created in the ArgoCD server namespace.

```
apiVersion: apps.open-cluster-management.io/v1beta1
kind: GitOpsCluster
metadata:
  name: argo-acm-importer
  namespace: openshift-gitops
spec:
  argoServer:
    cluster: notused
    argoNamespace: openshift-gitops
  placementRef:
    kind: Placement
    apiVersion: cluster.open-cluster-management.io/v1beta1
    name: bgdk-app-placement       # the placement can select all clusters
    namespace: openshift-gitops
```


3.  For all namespaces on each managed cluster where the ArgoCD application will be deployed, They have to be managed by the 
    Argocd Application controller SA as you see in the following example.

```
apiVersion: v1
kind: Namespace
metadata:
 name: mortgage2
 labels:
argocd.argoproj.io/managed-by: openshift-gitops
```

4. Let ArgoCD Application controller ignore these ArgoCD applications propagated by pull model
-
For OpenShift GitOps operator 1.9.0+,  specify the annotation in the ArgoCD ApplicationSet template
annotations:

```
 argocd.argoproj.io/skip-reconcile: "true"
```

For more details, please refer to this link from the GitopsOperator [documentation](https://docs.openshift.com/container-platform/4.11/cicd/gitops/configuring-an-openshift-cluster-by-deploying-an-application-with-cluster-configurations.html#creating-an-application-by-using-the-oc-tool_configuring-an-openshift-cluster-by-deploying-an-application-with-cluster-configurations): 


Explicitly declare all application destination namespaces in the Git repo or Helm repo for the application, and include the managed-by label in the namespaces. Refer to the link on how to declare a [namespace](https://github.com/redhat-developer-demos/openshift-gitops-examples/blob/44fc1d4a38cb79ffa6c8524788f5ac87f369d41c/apps/bgd/overlays/bgd/bgd-ns.yaml#L6) containing the managed-by label in a Git repo.

For deploying applications using the pull model, it is important for the ArgoCD application controllers to ignore these application resources on the hub cluster. The desired solution is to add the `argocd.argoproj.io/skip-reconcile` annotation to the template section of the applicationSet. On the RHACM hub cluster, the required OpenShift GitOps operator must be version 1.9.0 or above. On the managed cluster(s), the OpenShift GitOps operator is recommended to be at the same level as the hub cluster.


### ApplicationSet CRD

The ArgoCD ApplicationSet CRD is used to deploy applications on the managed clusters using the push or pull model. It uses a `placement` resource in the generator field to get a list of managed clusters. The template field supports parameter substitution of specifications for the application. The ArgoCD applicationSet controller on the hub cluster manages the creation of the application for each target cluster.

For the pull model, the destination for the application must be the default local kubernetes server (https://kubernetes.default.svc) since the application will be deployed locally by the application controller on the managed cluster. By default, the push model is used to deploy the application, unless the annotations apps.`open-cluster-management.io/ocm-managed-cluster` and `apps.open-cluster-management.io/pull-to-ocm-managed-cluster` are added to the template section of the applicationSet.

The following is a sample ApplicationSet YAML that uses the pull model:

```
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: guestbook-allclusters-app-set
  namespace: openshift-gitops
spec:
  generators:
  - clusterDecisionResource:
      configMapRef: ocm-placement-generator
      labelSelector:
        matchLabels:
          cluster.open-cluster-management.io/placement: aws-app-placement
      requeueAfterSeconds: 30
  template:
    metadata:
      annotations:
        apps.open-cluster-management.io/ocm-managed-cluster: '{{name}}'
        apps.open-cluster-management.io/ocm-managed-cluster-app-namespace: openshift-gitops
        argocd.argoproj.io/skip-reconcile: "true"
      labels:
        apps.open-cluster-management.io/pull-to-ocm-managed-cluster: "true"
      name: '{{name}}-guestbook-app'
    spec:
      destination:
        namespace: guestbook
        server: https://kubernetes.default.svc
      project: default
      source:
        path: guestbook
        repoURL: https://github.com/argoproj/argocd-example-apps.git
      syncPolicy:
        automated: {}
        syncOptions:
        - CreateNamespace=true
```

Propagation controller:

There are two sets of controllers on the hub cluster watching the applicationSet resources: 

The existing Argo CD application controllers and the new propagation controller. 

Annotations in the application resource are used to determine which controller reconciles to deploy the application. The ArgoCD application controllers, which are used for the push model,  ignore applications that contain the `argocd.argoproj.io/skip-reconcile` annotation. The propagation controller, which supports the pull model, only reconciles on applications that contain the `apps.open-cluster-management.io/ocm-managed-cluster` annotation. 

It generates a `manifestWork` to deliver the application to the managed cluster. 

The managed cluster is determined by the value of the `ocm-managed-cluster` annotation.

The following is a sample manifestWork YAML that is generated by the propagation controller to create the guestbook application on the managed cluster pcluster2:

```
apiVersion: work.open-cluster-management.io/v1
kind: ManifestWork
metadata:
  annotations:
    apps.open-cluster-management.io/hosting-applicationset: openshift-gitops/guestbook-allclusters-app-set
 name: pcluster2-guestbook-app-4a491
  namespace: pcluster2
spec:
  manifestConfigs:
  - feedbackRules:
    - jsonPaths:
      - name: healthStatus
        path: .status.health.status
      type: JSONPaths
    - jsonPaths:
      - name: syncStatus
        path: .status.sync.status
      type: JSONPaths
    resourceIdentifier:
      group: argoproj.io
      name: pcluster2-guestbook-app
      namespace: openshift-gitops
      resource: applications
  workload:
    manifests:
    - apiVersion: argoproj.io/v1alpha1
      kind: Application
      metadata:
        annotations:
          apps.open-cluster-management.io/hosting-applicationset: openshift-gitops/guestbook-allclusters-app-set
        finalizers:
        - resources-finalizer.argocd.argoproj.io
        labels:
          apps.open-cluster-management.io/application-set: "true"
        name: pcluster2-guestbook-app
        namespace: openshift-gitops
      spec:
        destination:
          namespace: guestbook
          server: https://kubernetes.default.svc
        project: default
        source:
          path: guestbook
          repoURL: https://github.com/argoproj/argocd-example-apps.git
        syncPolicy:
          automated: {}
          syncOptions:
          - CreateNamespace=true

```

As a result of the feedback rules specified in manifestConfigs, the health status and the sync status from the status of the ArgoCD application are synced to the manifestwork’s statusFeedback.

### Deploy application by the local ArgoCD server on the managed cluster.

After the ArgoCD application is created on the managed cluster through `ManifestWorks`, the local ArgoCD controllers reconcile to deploy the application. 
The controllers deploy the application through this sequence of operations:

* Connect and pull resources from the specified Git/Helm repository
* Deploy the resources on the local managed cluster
* Generate the ArgoCD application status
* Multicluster Application report - aggregate application status from the managed clusters 

A new `multicluster applicationSet report CRD` is introduced to provide an aggregate status of the applicationSet on the hub. The report is only created for applicationSets that are deployed using the new pull model. It includes the list of resources and the overall status of the application from each managed cluster. A separate multicluster applicationSet report resource is created for each ArgoCD applicationSet resource. The report is created in the same namespace as the applicationSet. 
The Multicluster ApplicationSet report includes:

* List of resources for the ArgoCD application
* Overall sync and health status for one ArgoCD application
* Includes error message for each cluster where the overall status is out of sync or unhealthy
* Summary status of the overall application status from all the managed clusters

To support the generation of the multicluster applicationSet report, two new controllers have been added to the hub cluster: the `resource sync controller` and the `aggregation controller`. 
The `resource sync controller` runs every 10 seconds, and its purpose is to query the OCM search V2 component on each managed cluster to retrieve the resource list and any error messages for each ArgoCD application. 

It then produces an intermediate report for each application set, which is intended to be used by the aggregation controller to generate the final multicluster applicationSet report.

The `aggregation controller` also runs every 10 seconds, and it uses the report generated by the resource sync controller to add the health and sync status of the application on each managed cluster. 
The status for each application is retrieved from the status feedback in the manifestwork for the application. Once the aggregation is complete, the final multicluster applicationSet report is saved in the same namespace as the ArgoCD applicationSet, with the same name as the applicationSet.

The two new controllers, along with the propagation controller, all run in separate containers in the same multicluster-integrations pod, as shown in the example below:

```
NAMESPACE               NAME                                       READY   STATUS  
open-cluster-management multicluster-integrations-7c46498d9-fqbq4  3/3     Running  
```

The following is a sample multicluster applicationSet report YAML for the guestbook applicationSet.

```
apiVersion: apps.open-cluster-management.io/v1alpha1
kind: MulticlusterApplicationSetReport
metadata:
  labels:
    apps.open-cluster-management.io/hosting-applicationset: openshift-gitops.guestbook-allclusters-app-set
  name: guestbook-allclusters-app-set
  namespace: openshift-gitops
statuses:
  clusterConditions:
  - cluster: cluster1
    conditions:
    - message: 'Failed sync attempt to 53e28ff20cc530b9ada2173fbbd64d48338583ba: one or more objects failed to apply, reason: services is forbidden: User "system:serviceaccount:openshift-gitops:openshift-gitops-argocd-application-controller" cannot create resource "services" in API group "" in the namespace "guestbook",deployments.apps is forbidden: User "system:serviceaccount:openshift-gitops:openshift-gitops-argocd-application-controller" cannot create resource "deployments" in API group "apps" in the namespace "guestboo...'
      type: SyncError
    healthStatus: Missing
    syncStatus: OutOfSync
  - cluster: pcluster1
    healthStatus: Progressing
    syncStatus: Synced
  - cluster: pcluster2
    healthStatus: Progressing
    syncStatus: Synced
  summary:
    clusters: "3"
    healthy: "0"
    inProgress: "2"
    notHealthy: "3"
    notSynced: "1"
    synced: "2"
```


All the resources listed in the multicluster applicationSet report are actually deployed on the managed cluster(s). If a resource fails to be deployed, the resource won't be included in the resource list. However, the error message would indicate why the resource failed to be deployed.
Limitations

1. Resources are only deployed on the managed cluster(s).
2. If a resource failed to be deployed, it won't be included in the Multicluster ApplicationSet Report.
3. In the pull model, the local-cluster is excluded as target managed cluster.
4. There might be usecases where the Managed-Clusters cannot reach the GitServer.  In this usecase a push model would be the only solution.

## Wrapup

I'd like to thank RHACM's ApplicationLifecyclemanagement team for their huge efforts making this new important feature possible.



References:
GitHub: https://github.com/open-cluster-management-io/OCM
Website: https://open-cluster-management.io/
Docs: https://open-cluster-management.io/concepts/
Slack: https://kubernetes.slack.com/channels/open-cluster-mgmt
YouTube: https://www.youtube.com/c/OpenClusterManagement
Mailing Group: https://groups.google.com/g/open-cluster-management
Community Meetings: https://calendar.google.com/calendar/u/0/embed?src=openclustermanagement@gmail.com

