# Using Red Hat Advanced Cluster Management and OpenShift GitOps to manage OpenShift Virtualization

*Anecdote why it is useful and random picture*

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
control of the whole cluster and application life cycle. It can act as a 
central point for keeping an inventory of all your clusters and 
applications and enables multi-cluster and multi-cloud scenarios, such as 
deploying the same application across clusters in different regions, 
possibly on several cloud providers. It employs a hub and spoke architecture 
and allows the targeted distribution of Kubernetes manifests across clusters.

### What are hub and managed clusters?

The hub cluster is the cluster on which ACM is running on. It
acts as an inventory and carries out all management actions. It is not
running any actual workloads, these run on managed clusters. Managed 
clusters can be created directly through ACM and are added to the hub cluster's 
inventory. Alternatively existing clusters can be added to the inventory as 
well. For more information have a look at the [ACM documentation](https://access.redhat.com/documentation/en-us/red_hat_advanced_cluster_management_for_kubernetes/2.7/html/about/welcome-to-red-hat-advanced-cluster-management-for-kubernetes#multicluster-architecture).

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
builds on the ArgoCD project, therefore we name it ArgoCD furthermore. For 
more information have a look at the [GitOps documentation](https://docs.openshift.com/container-platform/4.12/cicd/gitops/understanding-openshift-gitops.html).

### A quick primer about Applications and ApplicationSets

Explain what are Applications and ApplicationSets.

The ArgoCD `Application` is a [`CustomResourceDefinition` (CRD)](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/),
which essentially describes a source of manifests and a target cluster to apply 
the manifests to. Besides that options like automatic creation of namespaces 
or the automatic revert of changes can be configured.

The ArgoCD `ApplicationSet` is a CRD building on ArgoCD `Applications`,
targeted to deploy and manage `Applications` across multiple clusters while
using the same manifest or declaration. It is possible to deploy multiple
`ApplicationSets` which are contained in one monorepo. By using generators
it is possible to dynamically select a subset of clusters available to
ArgoCD to deploy resources on to.

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

## Installing and configuring Advanced Cluster Management

### Installing ACM on the hub cluster

### Adding managed clusters to ACM on the hub cluster

### Organize managed clusters in a set

## Installing and configuring OpenShift GitOps

### Installing OpenShift GitOps on the hub cluster

### Accessing the OpenShift GitOps web interface

### Making a set of managed clusters available to OpenShift GitOps

## Deploying OpenShift Virtualization to one or more managed clusters

## Deploying a VirtualMachine to one or more managed clusters

### How to start or stop a VirtualMachine

## Advanced usage of ACM Placements with OpenShift GitOps

## Summary and outlook