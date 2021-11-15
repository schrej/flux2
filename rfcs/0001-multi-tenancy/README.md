# RFC-0001 Flux Multi-Tenancy

## Summary

The main goal of this RFC is to define the Kubernetes tenancy models supported by Flux.

## Motivation

As of Flux v0.23.0, the documentation contains references to account impersonation, reconciliation isolation and other
multi-tenancy related features without clearly defining which tenancy models are supported by Flux and how
they relate to source access control and Kubernetes role-based access control.

### Goals

- Define multi-tenancy from a Flux user perspective.
- List the types of users that interact with Flux in regard to multi-tenancy.
- List the tenancy models supported by Flux.
- Explain the differences between tenancy models.

### Non-Goals

- Explain in detail the Kubernetes architecture in regard to multi-tenancy.
- Provide an end-to-end workflow of how to setup multi-tenancy with Flux.

## Proposal

The Flux documentation should have a section dedicated to multi-tenancy. What follows is the proposed content
of the multi-tenancy overview documentation that sets the stage for future enhancements.

### Introduction

Flux allows different organizations and/or teams to share the same Kubernetes control plane to deliver applications
in a GitOps manner. Flux enables segmentation and isolation of resources across tenants by leveraging
Kubernetes Cluster API, namespaces and role-based access control.

### User Roles

There are two types of users that interact with Flux: platform admins and tenants.
Besides installing Flux, all the other operations (deploy applications, configure ingress, policies, etc)
do not require users to have direct access to the Kubernetes API. Flux acts as a proxy between users and
the Kubernetes API, using Git as source of truth for the cluster desired state. Changes to the clusters
and workloads configuration can be made in a collaborative manner, where the various teams responsible for
the delivery process propose, review and approve changes via pull request workflows.

#### Platform Admins

The platform admins have unrestricted access to Kubernetes API.
They are responsible for installing Flux and granting Flux
access to the sources (Git, Helm, OCI repositories) that make up the cluster(s) control plane desired state.
The repository(s) owned by the platform admins are reconciled on the cluster(s) by Flux, under
the [cluster-admin](https://kubernetes.io/docs/reference/access-authn-authz/rbac/#user-facing-roles)
Kubernetes cluster role.

Example of operations performed by cluster admins:

- Bootstrap Flux onto cluster(s).
- Extend the Kubernetes API with custom resource definitions and validation webhooks.
- Configure various controllers for ingress, storage, logging, monitoring, progressive delivery, etc.
- Set up namespaces for tenants and define their level of access with Kubernetes RBAC.
- Onboard tenants by registering their Git repositories with Flux.

#### Tenants

The tenants have restricted access to the cluster(s) according to the Kubernetes RBAC configured
by the platform admins. The repositories owned by tenants are reconciled on the cluster(s) by Flux,
under the Kubernetes account(s) assigned by platform admins.

Example of operations performed by tenants:

- Register their sources with Flux (`GitRepositories`, `HelmRepositories` and `Buckets`).
- Deploy workload(s) into their namespace(s) using Flux custom resources (`Kustomizations` and `HelmReleases`).
- Automate application updates using Flux custom resources (`ImageRepositories`, `ImagePolicies` and `ImageUpdateAutomations`).
- Configure the release pipeline(s) using Flagger custom resources (`Canaries` and `MetricsTemplates`).
- Setup webhooks and alerting for their release pipeline(s) using Flux custom resources (`Receivers` and `Alerts`).

### Tenancy Models

The Kubernetes tenancy models supported by Flux are: soft multi-tenancy and hard multi-tenancy.

For an overview of the Kubernetes multi-tenant architecture please consult the following documentation:

- [Three Tenancy Models For Kubernetes](https://kubernetes.io/blog/2021/04/15/three-tenancy-models-for-kubernetes/)
- [GKE multi-tenancy overview](https://cloud.google.com/kubernetes-engine/docs/concepts/multitenancy-overview)
- [EKS multi-tenancy best practices](https://aws.github.io/aws-eks-best-practices/security/docs/multitenancy/)

#### Soft Multi-Tenancy

With soft multi-tenancy, the platform admins use Kubernetes constructs such as namespaces, accounts,
roles and role bindings to create a logical separation between tenants.

When Flux deploys workloads from a repository belonging to a tenant, it uses the Kubernetes account assigned to that
tenant to perform the git-to-cluster reconciliation. By leveraging Kubernetes RBAC, Flux can ensure
that the operations performed by tenants are restricted to their namespaces.

Note that with this model, tenants share cluster-wide resources such as
`ClusterRoles`, `CustomResourceDefinitions`, `IngressClasses`, `StorageClasses`,
and they cannot create or alter these resources.
If a tenant adds o cluster-scoped resource definition to their repository,
Flux will fail the git-to-cluster reconciliation due to Kubernetes RBAC restrictions.

To restrict the reconciliation of tenant's sources, a Kubernetes service account name can be specified 
in Flux `Kustomizations` and `HelmReleases` under `.spec.serviceAccountName`. Please consult the Flux  
documentation for more details:

- [Kustomization API: Role-based access control](https://fluxcd.io/docs/components/kustomize/kustomization/#role-based-access-control)
- [HelmRelease API: Role-based access control](https://fluxcd.io/docs/components/helm/helmreleases/#role-based-access-control)
- [Flux multi-tenancy example repository](https://github.com/fluxcd/flux2-multi-tenancy)

Note that with soft multi-tenancy, true tenant isolation requires security measures beyond Kubernetes RBAC.
Please refer to the Kubernetes [security considerations documentation](https://kubernetes.io/blog/2021/04/15/three-tenancy-models-for-kubernetes/#security-considerations)
for more details on how to harden shared clusters.

##### Caveats

As of v0.23.0, Flux does not enforce a service account to be specified on Flux `Kustomizations` and `HelmReleases`.
When a service account is not specified, Flux defaults to cluster-admin.
In order to enforce the tenant isolation, an admission controller such as Kyverno or OPA Gatekeeper must be used
to make the `.spec.serviceAccountName` a required field for the Flux custom resources created by tenants.

We provide an [example](https://github.com/fluxcd/flux2-multi-tenancy/blob/main/infrastructure/kyverno-policies/flux-multi-tenancy.yaml) 
for enforcing service accounts using a Kyverno cluster policy.

As of v0.23.0, Flux allows for `Kustomizations` and `HelmReleases` to reference sources
(`GitRepositories`, `HelmRepositories` and `Buckets`) across namespaces.
In order to prevent tenants from accessing each other sources, an admission controller such as Kyverno or OPA Gatekeeper
must be used to block cross-namespace references.

We provide an [example](https://github.com/fluxcd/flux2-multi-tenancy/blob/main/infrastructure/kyverno-policies/flux-multi-tenancy.yaml)
for blocking source cross-namespace references using a Kyverno cluster policy.

#### Hard Multi-Tenancy

With hard multi-tenancy, the platform admins use Kubernetes Cluster API to create dedicated clusters for each tenant.
The Flux instance installed on the management cluster is responsible
for reconciling the cluster definitions belonging to tenants.

To enable GitOps for the tenant's clusters, the platform admins can configure the Flux instance running on the
management cluster to connect to the tenant's cluster using the `kubeConfig` generated by the Cluster API provider.

To configure Flux reconciliation of remote clusters, a Kubernetes secret containing a `kubeConfig` can be specified
in Flux `Kustomizations` and `HelmReleases` under `.spec.kubeConfig.secretRef`. Please consult the Flux API
documentation for more details:

- [Kustomization API: Remote Clusters](https://fluxcd.io/docs/components/kustomize/kustomization/#remote-clusters--cluster-api)
- [HelmRelease API: Remote Clusters](https://fluxcd.io/docs/components/helm/helmreleases/#remote-clusters--cluster-api)

Note that with hard multi-tenancy, tenants have full access to cluster-wide resources, so they have the option
to manage Flux independently of platform admins, by deploying a Flux instances on each cluster.

##### Caveats

When using a Kubernetes Cluster API provider, the `kubeConfig` secret is automatically generated and Flux can
make use of it without any manual actions. For clusters created by other means than Cluster API, the
platform team has to create the `kubeConfig` secrets to allow Flux access to the remote clusters.

As of Flux v0.23.0, we don't provide any guidance for cluster admins on how to generate the `kubeConfig` secrets.

## Implementation History

- Soft multi-tenancy based on service account impersonation was first released in flux2 **v0.0.1**.
- Generating namespaces and RBAC for defining tenants with `flux create tenant` was first released in flux2 **v0.1.0**.
- Hard multi-tenancy based on remote cluster reconciliation was first released in flux2 **v0.2.0**.
- Soft multi-tenancy end-to-end workflow example was first published on 27 Nov 2020 at
  [fluxcd/flux2-multi-tenancy](https://github.com/fluxcd/flux2-multi-tenancy).
- Soft multi-tenancy [CVE-2021-41254](https://github.com/fluxcd/kustomize-controller/security/advisories/GHSA-35rf-v2jv-gfg7)
  "Privilege escalation to cluster admin on multi-tenant environments" was fixed in flux2 **v0.15.0**.
