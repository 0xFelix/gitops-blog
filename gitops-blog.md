# Using Red Hat Advanced Cluster Management and OpenShift GitOps to manage OpenShift Virtualization

![Title picture](https://i.imgur.com/dB9Lg78.png)

Whether you want to separate your testing and production environments, improve
the availability of your applications or bring your OpenShift clusters to the
edge, working with a multi-cluster environment is the key in achieving
those goals. Red Hat Advanced Cluster Management allows you to easily work
with multiple clusters and comes with a number of advantages.

However, managing your clusters and applications at scale can be a challenging
task. Ideally there is a single source of truth, which determines the
configuration and workloads of each cluster. OpenShift GitOps enables you to do
that by storing your configuration in Git repositories and keeping all of your
clusters in sync automatically.

Do you want to manage your OpenShift Virtualization virtual machine workloads
across multiple clusters while using a single source of truth in the GitOps
way? This blog post will show how you can do that with Red Hat Advanced
Cluster Management and OpenShift GitOps.

## What is Red Hat Advanced Cluster Management?

Red Hat Advanced Cluster Management or short ACM simplifies the management
of multiple clusters by offering end-to-end management, visibility and
control of the whole cluster and application life cycle. It acts as a
central point for keeping an inventory of all your clusters and
applications and enables multi-cluster and multi-cloud scenarios, such as
deploying the same application across clusters in different regions,
possibly on several cloud providers. It uses a hub and spoke architecture
and allows the targeted distribution of Kubernetes manifests across clusters.

### What are hub and managed clusters?

The hub cluster is the cluster on which ACM is running on. It
acts as an inventory and carries out all management actions. It is not
running any actual workloads, these run on managed clusters. Managed
clusters are kept in the inventory of the hub cluster. They can be created
and added to the inventory directly through ACM. Alternatively, existing
clusters can be added to the inventory as well. For more information have
a look at
the [ACM documentation](https://access.redhat.com/documentation/en-us/red_hat_advanced_cluster_management_for_kubernetes/2.7/html/about/welcome-to-red-hat-advanced-cluster-management-for-kubernetes#multicluster-architecture).

## What is the GitOps way and what is OpenShift GitOps?

The GitOps way uses Git repositories as a single source of truth to deliver
infrastructure as code. Automation is employed to keep the desired and the
live state of clusters in sync at all times. This means any changes to a
repository are automatically applied to one or more clusters while changes
to a cluster will be automatically reverted to the state described in the
single source of truth.

Red Hat Openshift GitOps enables declarative GitOps workflows and allows to
deploy applications on-demand. It monitors the live state of clusters
against the desired state in a Git repository and keeps them in sync. It
builds on the ArgoCD project, therefore the terms OpenShift GitOps and ArgoCD
might be used interchangeably in the following sections. For
more information have a look at
the [GitOps documentation](https://docs.openshift.com/container-platform/4.12/cicd/gitops/understanding-openshift-gitops.html).

### A quick primer about Applications and ApplicationSets

The ArgoCD `Application` is
a [`CustomResourceDefinition` (CRD)](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/),
which essentially describes a source of manifests and a target cluster to apply
the manifests to. Besides that, options like automatic creation of namespaces
or the automatic revert of changes can be configured.

The ArgoCD `ApplicationSet` is a CRD building on ArgoCD `Applications`,
targeted to deploy and manage `Applications` across multiple clusters while
using the same manifest or declaration. It is possible to deploy multiple
`ApplicationSets` which are contained in one monorepo. By using generators
it is possible to dynamically select a subset of clusters available to
ArgoCD to deploy resources to.

In this blog post we are going to use `ApplicationSets` to deploy OpenShift
Virtualization and `VirtualMachines` to multiple clusters while using
the same declaration of resources for all clusters.

For more information on `ApplicationSets` see the
[documentation](https://argo-cd.readthedocs.io/en/stable/operator-manual/applicationset).

## Requirements for the setup

The following requirements need to be satisfied to build the setup described
in this blog post yourself:

- A (public) Git repository accessible by the hub cluster
- One OpenShift cluster acting as hub cluster
    - Needs to be publicly accessible
- One or more OpenShift clusters acting as managed clusters
    - Can be in private networks
    - Virtualization has to be available
    - Nested virtualization is fine for demonstration purposes

## Repository preparation

A repository with the files used in this blog post can be found at
https://github.com/0xFelix/gitops-demo. You need to clone this repository
to somewhere where you are able to make changes to it (i.e. forking it on
GitHub).
Then open a terminal on your machine, check out the repository locally and
change
your working directory into the cloned repository.

The `ApplicationSets` in the demo repository use the above repository URL as
`repoURL`. To be able to make changes to your `ApplicationSets`, you
need to adjust the `repoURL` to the URL of your own repository. If you do
this later do not forget to update any existing `ApplicationSets` on your hub
cluster.

## Installing and configuring Advanced Cluster Management

The following steps will guide you through installing and configuring ACM on
your hub cluster. We will use the OpenShift console where possible.

### Installing ACM on the hub cluster

![ACM Install in OperatorHub](https://i.imgur.com/FB8niIZ.png)

1. Login as cluster administrator on the UI of the hub cluster
2. Open the `Administrator` view if it is not already selected
2. In the menu click on `Operators` and open `OperatorHub`
3. In the search type `Advanced Cluster Management for Kubernetes` and click
   on it in the results
4. Click on `Install`, keep defaults and click on `Install` again
5. Wait until `MultiClusterHub` can be created and create it
6. Wait until the created `MultiClusterHub` is ready (`Operators` -->
   `Installed Operators` --> see status of ACM)

### Adding managed clusters to ACM on the hub cluster

Managed clusters can be added to ACM in two ways:

1. Create a new cluster with ACM
2. Add an existing cluster to ACM

> Note: For the sake of simplicity we will let ACM create the managed
> clusters in this blog post on a public cloud provider. Please note that nested
> virtualization is not supported in production deployments.

To create one or more managed clusters follow these steps:

1. Login as cluster administrator on the UI of the hub cluster
2. At the top of the menu select `All Clusters` (`local-cluster` should be
   selected initially)
3. Add credentials for you cloud provider by clicking on `Credentials` in
   the menu and then clicking on `Add credentials`
4. Click on `Infrastructure` and then on `Clusters` in the menu
5. Click `Create cluster`, select your cloud provider and complete the
   wizard (use the default cluster set for now)

> Note: When using Azure as cloud provider select instance type
`Standard_D8s_v3` for the control plane and `Standard_D4s_v3` for the worker
> nodes, resources might become to tight to run virtual machines on the
> cluster otherwise.

### Organizing managed clusters in a set

Managed clusters can be grouped into `ManagedClusterSets`. These sets can be
bound to namespaces with a `ManagedClusterSetBinding` to make managed clusters
available in the bound namespaces.

To add managed clusters to a new set follow these steps:

1. Login as cluster administrator on the UI of the hub cluster
2. At the top of the menu select `All Clusters` (`local-cluster` should be
   selected initially)
4. Click on `Infrastructure` and then on `Clusters` in the menu
3. Click on `Cluster sets` and then on `Create cluster set`
4. Enter `managed` as name for the new set and click on `Create`
5. Click on `Managed resource assignments`
6. Select all clusters you want to add, click on `Review` and then on `Save`

Now we have a `ManagedClusterSet` that can be used to make the managed clusters
available to ArgoCD.

![Managed cluster list](https://i.imgur.com/81VN8tQ.png)

If done correctly, the cluster list of the created `ManagedClusterSet` in ACM
should look like in the screenshot above.

## Installing and configuring OpenShift GitOps

The following steps will guide you through installing and configuring
OpenShift GitOps or ArgoCD on your hub cluster. We will use the OpenShift
console where possible again.

### Installing OpenShift GitOps on the hub cluster

![GitOps Install in OperatorHub](https://i.imgur.com/yMawpQG.png)

1. Login as cluster administrator on the UI of the hub cluster
2. Open the `Administrator` view if it is not already selected
2. In the menu click on `Operators` and open `OperatorHub`
3. In the search type `Red Hat OpenShift GitOps` and click
   on it in the results
4. Click on `Install`, keep defaults and click on `Install` again
6. Wait until OpenShift GitOps is ready (`Operators` -->
   `Installed Operators` --> see status of OpenShift GitOps)

If installed correctly, the list of installed operators on your cluster should
look like in the following screenshot:

![Installed Operators](https://i.imgur.com/gunWrRz.png)

### Accessing the OpenShift GitOps web UI

The OpenShift GitOps web UI is exposed with a `Route`. To get the
exact URL of the `Route` follow these steps:

1. Login as cluster administrator on the UI of the hub cluster
2. Open the `Administrator` view if it is not already selected
3. In the menu click on `Networking` and open `Routes`
4. In the `Projects` drop down select `openshift-gitops`
   (enable `Show default projects` if not visible)
5. There will be a `Route` called `openshift-gitops-server`, the location of
   this `Route` is the URL to the GitOps UI
6. You can log in to the GitOps UI with your OpenShift credentials

Alternatively you can use the command line to get the URL to the GitOps UI with
the following command:

```shell
oc get route -n openshift-gitops openshift-gitops-server -o jsonpath='{.spec.host}'
```

### Making a set of managed clusters available to OpenShift GitOps

To make a set of managed clusters available to OpenShift GitOps, a tight
integration between ACM and GitOps exists. The integration is
controlled with the `GitOpsCluster` CRD.

Follow these steps to make the managed clusters available to GitOps:

1. Copy the login command for the command line by clicking on your username on
   the top right and then click on `Copy login command`
2. Run the copied command in your terminal
3. Create a `ManagedClusterSetBinding` in the `openshift-gitops` namespace
   to make the `ManagedClusterSet` available in this namespace
    - See file
      [managedclustersetbinding.yaml](https://github.com/0xFelix/gitops-demo/blob/main/acm-gitops-integration/managedclustersetbinding.yaml)
    - Run `oc create -f acm-gitops-integration/managedclustersetbinding.yaml`
4. Create a `Placement` to let ACM decide which clusters should be made
   available to GitOps
    - See
      file [placement.yaml](https://github.com/0xFelix/gitops-demo/blob/main/acm-gitops-integration/placement.yaml)
    - Run `oc create -f acm-gitops-integration/placement.yaml`
    - For the sake of simplicity this will select the whole
      `ManagedClusterSet`, but advanced use cases are possible
5. Create a `GitOpsCluster` to finally make the selected clusters available to
   GitOps on the hub cluster
    - See
      file [gitopscluster.yaml](https://github.com/0xFelix/gitops-demo/blob/main/acm-gitops-integration/gitopscluster.yaml)
    - Run `oc create -f acm-gitops-integration/gitopscluster.yaml`

![ArgoCD cluster list](https://i.imgur.com/YRgvFCu.png)

In this screenshot you can see that the managed clusters were made available to
ArgoCD successfully. This view can be opened by going to ArgoCD's settings and
opening the `Clusters` menu.

## Deploying OpenShift Virtualization to one or more managed clusters

![OpenShift Virtualization Application in ArgoCD](https://i.imgur.com/bSBQwlW.png)

To deploy OpenShift Virtualization to the managed clusters with the help of
an `ApplicationSet` run the following command:

```shell
oc create -f applicationsets/virtualization/applicationset-virtualization.yaml
```

This will create an `Application` for each managed cluster that deploys
OpenShift Virtualization with its default settings. The `Application` will
ensure that the namespace `openshift-cnv` exists, and it will automatically
apply any changes to this repository or undo changes which are not in this
repository. Sync waves are used to ensure that resources are created in the
right order.

Order of resource creation:

1. `OperatorGroup`
2. `Subscription`
3. `HyperConverged`

Because the `HyperConverged` CRD is unknown to ArgoCD, the sync option
`SkipDryRunOnMissingResource=true` is set to allow ArgoCD to create a CR
without knowing its CRD.

In ArgoCD's UI you can follow the synchronization status of the newly
created `Application` for each cluster. Eventually every `Application` will
reach the healthy and synced status like in the following screenshot.

![OpenShift Virtualization Application becoming ready](https://i.imgur.com/XMVaIgA.png)

To see what is actually deployed have a look into the following directory:
`applicationsets/virtualization/manifests`.

## Deploying a VirtualMachine to one or more managed clusters

To deploy a Fedora `VirtualMachine` on all managed clusters with the help of
an `ApplicationSet` run the following command:

```shell
oc create -f applicationsets/demo-vm/applicationset-demo-vm.yaml
```

This will create an `Application` for each managed cluster that deploys
a simple `VirtualMachine` on each cluster. It uses the Fedora `DataSource`
available on the cluster by default to boot a Fedora cloud image.

The created `Application(s)` are also visible in ArgoCD's UI:

![New demo VM Application in suspended state](https://i.imgur.com/qFSzDDA.png)

Notice how the health state of the created `Application` is `Suspended`. This is
because the created `VirtualMachine` is still in stopped state.

To see what is actually deployed have a look into the following directory:
`applicationsets/demo-vm/manifests`.

### How to start or stop a VirtualMachine

First let us have a closer look at the `Application` of the
stopped `VirtualMachine`. Notice the `Suspended` health state.

![Detail view of suspended demo VM Application](https://i.imgur.com/27T3eCU.png)

To start or stop a `VirtualMachine` you need to edit the `spec.running`
field of a `VirtualMachine` and set it to the corresponding value (`false`
or `true`).

If the `VirtualMachine` has an appropriate termination grace
period (`spec.template.spec.terminationGracePeriodSeconds`), setting this
value to `false` will gracefully shut down the `VirtualMachine`. When
setting the timeout grace period to 0 seconds, the `VirtualMachine` is
stopped immediately however.

To apply new changes with ArgoCD you need to commit and push changes to
the Git repository containing your `Application`. To start or stop a
`VirtualMachine` update the manifest and commit and push
to your repository. In the ArgoCD UI select the `Application` of the
`VirtualMachine` and click `Refresh` to apply the change immediately.
Otherwise, it will take some time until ArgoCD scans the repository and
picks up the change.

After ArgoCD picked up the change it will sync it to the `VirtualMachine` as
visible by the `Progressing` health state in the following screenshot:

![Detail view of progressing demo VM Application](https://i.imgur.com/ABnivtn.png)

Eventually the `VirtualMachine` will be running and healthy:

![Detail view of healthy demo VM Application](https://i.imgur.com/or5HMD1.png)

## Advanced usage of ACM Placements with OpenShift GitOps

For the sake of simplicity the `Placement` created in this blog post selects the
whole `ManagedClusterSet`, but more advanced use cases are possible.

ACM can dynamically select a subset of clusters from the
`ManagedClusterSet`, while following a defined set of criteria. This for
example allows to schedule `VirtualMachines` on clusters with the most
resources available at the time of the placement decision.

For more on this topic see
[Using the Open Cluster Management Placement for Multicluster Scheduling](https://cloud.redhat.com/blog/using-the-open-cluster-management-placement-for-multicluster-scheduling).

## Summary and outlook

In this blog post we set up a hub cluster and one or more clusters managed
by ACM to deploy applications to from a centralized management point. As an
example application we deployed a simple virtual machine and learned how we
can start and stop virtual machines in a declarative way. All of this was
accomplished in the GitOps way by using a Git repository as a single source
of truth.

This is of course only the tip of the iceberg, building on this setup
allows you to e.g. have `ApplicationSets` for different environments
like development, staging and production or to schedule your applications
based on custom criteria (e.g. available resources) with advanced placements
rules.