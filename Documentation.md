# Rubrik Kubernetes Protection Least Privilege Configuration

## Table of Contents

[Introduction](#introduction)

[Setting up Kubernetes (K8s) Cluster on RSC](#setting-up-kubernetes-k8s-cluster-on-rsc)

- [Prerequisites](#prerequisites)

- [Step 1: Create a Kubeconfig](#step-1-create-a-kubeconfig)

- [Step 2: Assign cluster-scoped and backup/recovery permissions to the Kubeconfig](#step-2-assign-cluster-scoped-and-backuprecovery-permissions-to-the-kubeconfig)

  - [I. Create a ClusterRole with the provided definition](#i-create-a-clusterrole-with-the-provided-definition)

  - [II. Grant Kubeconfig User these ClusterRole permissions with a ClusterRoleBinding](#ii-grant-kubeconfig-user-these-clusterrole-permissions-with-a-clusterrolebinding)

  - [III. Configure Backup/Recovery permissions for the Kubeconfig User](#iii-configure-backuprecovery-permissions-for-the-kubeconfig-user)

    - [Option A: Protection for All Resources in the Cluster](#option-a-protection-for-all-resources-in-the-cluster)

    - [Option B: Protection for All Resources in Specific Namespaces](#option-b-protection-for-all-resources-in-specific-namespaces)

    - [Option C: Protection for Specific Resources](#option-c-protection-for-specific-resources)

- [Step 3: Add the K8s Cluster to RSC](#step-3-add-the-k8s-cluster-to-rsc)

- [Step 4: Assign rubrik-kupr namespace permissions to the Kubeconfig](#step-4-assign-rubrik-kupr-namespace-permissions-to-the-kubeconfig)

- [Verification](#verification)

# Introduction

This guide helps you to set up the least privileged Kubeconfig within your Kubernetes cluster. For reference, the abbreviations below are used throughout this guide:

- **API** - Application Programming Interface  
- **CR** - Custom Resource  
- **CRD** - Custom Resource Definition  
- **K8s** - Kubernetes  
- **PVC** - Persistent Volume Claim  
- **RBAC** - Role-Based Access Control  
- **RSC** - Rubrik Security Cloud  
- **SCC** - Security Context Constraint (specific to OpenShift)  
- **UI** - User Interface

Throughout the guide, `rubrik-kubeconfig-user` will be the **User** associated with the Kubeconfig. We'll apply all **Role Bindings** and **Cluster Role Bindings** to this **User**.

# Setting up Kubernetes (K8s) Cluster on RSC

## Prerequisites

- Before adding the cluster to RSC, ensure `rubrik-kupr` namespace does not exist in the K8s Cluster. This namespace will be created by Rubrik when the cluster is added to RSC.
- To perform backup/recovery of PVCs, Rubrik will create one service of type LoadBalancer in the `rubrik-kupr` namespace. This Loadbalancer endpoint will be used to communicate with the Rubrik Backup Agent deployed in the K8s cluster. Ensure that LoadBalancer type service is supported on the K8s cluster.

## Step 1: Create a Kubeconfig

You'll need to create a Kubeconfig for adding the K8s cluster to RSC. While instructions for creating a Kubeconfig using the `kubeadm kubeconfig user` tool are detailed in [Certificate Management with kubeadm | Kubernetes](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-certs/#kubeconfig-additional-users), it's important to note that the method for creating a Kubeconfig may differ depending on your environment. Consider your particular requirements and choose the best approach accordingly.

In the following steps, we assume that the **User** associated with the Kubeconfig is called `rubrik-kubeconfig-user` and cover the required permissions for the Kubeconfig.

## Step 2: Assign cluster-scoped and backup/recovery permissions to the Kubeconfig

After creating a Kubeconfig in [Step 1](#step-1-create-a-kubeconfig), you need to assign the following permissions to the Kubeconfig:

### I. Create a **ClusterRole** with the provided definition

```yaml
# ClusterRole with permissions required by the Rubrik Kubeconfig
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: rubrik-kubeconfig-cluster-role
rules:
# Required to create the rubrik-kupr namespace and new namespaces during export and to delete the rubrik-kupr namespace when removing the cluster
- apiGroups: [""]
  resources: ["namespaces"]
  verbs: ["create"]
- apiGroups: [""]
  resources: ["namespaces"]
  verbs: ["delete"]
  resourceNames: ["rubrik-kupr"]

# Required to list namespaces in UI
- apiGroups: [""]
  resources: ["namespaces"]
  verbs: ["list", "get"]

# Required to perform backup and recovery of PVCs
- apiGroups: ["snapshot.storage.k8s.io"]
  resources: ["volumesnapshotcontents", "volumesnapshots"] 
  verbs: ["create", "get", "list", "delete", "patch", "update"]
- apiGroups: ["snapshot.storage.k8s.io"]
  resources: ["volumesnapshotclasses"]
  verbs: ["list"]
- apiGroups: ["storage.k8s.io"]
  resources: ["storageclasses"]
  verbs: ["get"]
- apiGroups: [""]
  resources: ["persistentvolumes"]
  verbs: ["get", "patch", "update"]

# Required to manage Rubrik-owned CRDs
- apiGroups: ["apiextensions.k8s.io"]
  resources: ["customresourcedefinitions"]
  resourceNames: [
    "filters.apps.kupr.rubrik.com",
    "hooks.apps.kupr.rubrik.com",
    "protectionsets.apps.kupr.rubrik.com",
    "snapshots.apps.kupr.rubrik.com",
    "k8sclusters.clusters.kupr.rubrik.com",
    "rubrikclusters.clusters.kupr.rubrik.com",
    "recoveryjobs.jobs.kupr.rubrik.com",
    "snapshotjobs.jobs.kupr.rubrik.com",
    "sladomains.sla.kupr.rubrik.com",
  ]
  verbs: ["get", "delete", "patch", "update"]
- apiGroups: ["apiextensions.k8s.io"]
  resources: ["customresourcedefinitions"]
  verbs: ["create"]

# Required to keep the Rubrik CR objects in sync with RSC
- apiGroups: ["apps.kupr.rubrik.com"]
  resources: ["*"]
  verbs: ["*"]
- apiGroups: ["jobs.kupr.rubrik.com"]
  resources: ["*"]
  verbs: ["*"]
- apiGroups: ["clusters.kupr.rubrik.com"]
  resources: ["*"]
  verbs: ["*"]
- apiGroups: ["sla.kupr.rubrik.com"]
  resources: ["*"]
  verbs: ["*"]

# Required to create ClusterRoleBindings for the CRD controller
- apiGroups: ["rbac.authorization.k8s.io"]
  resources: ["clusterroles"]
  verbs: ["get", "update"]
  resourceNames: ["kupr-user", "kupr-admin", "rubrik-kupr-appscontroller-role"]
- apiGroups: ["rbac.authorization.k8s.io"]
  resources: ["clusterroles"]
  verbs: ["create"]
- apiGroups: ["rbac.authorization.k8s.io"]
  resources: ["clusterrolebindings"]
  verbs: ["get", "update"]
  resourceNames: ["rubrik-kupr-appscontroller-rolebinding"]
- apiGroups: ["rbac.authorization.k8s.io"]
  resources: ["clusterrolebindings"]
  verbs: ["create"]
- apiGroups: [""]
  resources: ["events"]
  verbs: ["create", "get", "list", "patch", "update", "watch"]

# This rule allows Rubrik to get/update only secrets with the name “rubrik-svcacnt” and allows creating new secrets.
# The “rubrik-svcacnt” secret is created in the rubrik-kupr namespace and is used to store the RSC service account. The Rubrik Controller needs this for Rubrik CRDs to function properly.
# This rule is only needed in the rubrik-kupr namespace but since the namespace doesn’t exist before adding the K8s cluster, it needs to be in the ClusterRole during cluster registration.
# Adding a K8s cluster will fail without this. This permission can be removed once the K8s cluster is added to RSC successfully.
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["get", "update"]
  resourceNames: ["rubrik-svcacnt"]
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["create"]
```

If you're using Red Hat OpenShift Kubernetes Clusters, these additional permissions are required in the **ClusterRole** definition.

```yaml
# Required to provide anyuid SCC permissions to rubrik-kupr service account
- apiGroups: ["rbac.authorization.k8s.io"]
  resources: ["clusterrolebindings"]
  verbs: ["get", "update"]
  resourceNames: ["rubrik-kupr-anyuid-scc-rolebinding"]
- apiGroups: ["rbac.authorization.k8s.io"]
  resources: ["clusterrolebindings"]
  verbs: ["create"]

# Required to run the Rubrik Backup Agent as root in rubrik-kupr namespace
- apiGroups: ["security.openshift.io"]
  resources: ["securitycontextconstraints"]
  resourceNames: ["anyuid"]
  verbs: ["use"]
```

### II. Grant Kubeconfig **User** these **ClusterRole** permissions with a **ClusterRoleBinding**

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: rubrik-kubeconfig-cluster-role-binding
roleRef:
  kind: ClusterRole
  name: rubrik-kubeconfig-cluster-role
  apiGroup: rbac.authorization.k8s.io
subjects:
- kind: User
  name: rubrik-kubeconfig-user
  apiGroup: rbac.authorization.k8s.io
```

### III. Configure Backup/Recovery permissions for the Kubeconfig User

Depending on the resources you need to protect, choose one of the following options (A, B, C) and create the respective **ClusterRoleBindings** and **RoleBindings** for your Kubeconfig **User**.

#### Option A: Protection for All Resources in the Cluster

Choose this option if all namespaces and cluster-scoped resources require protection. This setup allows `get`, `list` (for backup), and `create` (for recovery) permissions on all resources. You can achieve this by creating a **ClusterRole** and subsequently assigning these permissions to the Kubeconfig **User** via a **ClusterRoleBinding**, as outlined below.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: rubrik-backup-recovery-cluster-role
rules:
# Required for backup/recovery of all resources in the cluster
- apiGroups: ["*"]
  resources: ["*"]
  verbs: ["get", "list", "create"]
```

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: rubrik-backup-recovery-cluster-role-binding
roleRef:
  kind: ClusterRole
  name: rubrik-backup-recovery-cluster-role
  apiGroup: rbac.authorization.k8s.io
subjects:
- kind: User
  name: rubrik-kubeconfig-user
  apiGroup: rbac.authorization.k8s.io
```

#### Option B: Protection for All Resources in Specific Namespaces

Choose this option if only specific namespaces require protection. This setup allows `get`, `list`, and `create` permissions on all resources within protected namespaces. You will set up a similar **ClusterRole** as in [Option A](#option-a-protection-for-all-resources-in-the-cluster), but this will be bound to the Kubeconfig **User** in each protected namespace using a **RoleBinding**.

**Export Workflow**: Due to Kubeconfig permissions being limited to specific namespaces, Rubrik cannot export snapshots into a **new namespace**. To work around this, create an empty namespace with **RoleBinding**, which will enable Rubrik to export the snapshot into this namespace.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: rubrik-backup-recovery-cluster-role
rules:
# Required for backup/recovery of all resources in the cluster
- apiGroups: ["*"]
  resources: ["*"]
  verbs: ["get", "list", "create"]
```

Please update the `<namespace-name>` to the correct value and apply this **RoleBinding** in all the namespaces that need to be protected.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: rubrik-backup-recovery-role-binding
 namespace: <namespace-name>
roleRef:
  kind: ClusterRole
  name: rubrik-backup-recovery-cluster-role
  apiGroup: rbac.authorization.k8s.io
subjects:
- kind: User
  name: rubrik-kubeconfig-user
  apiGroup: rbac.authorization.k8s.io
```

#### Option C: Protection for Specific Resources

Choose this option if only specific resources within a namespace require protection. This setup creates a **Role** with the required permissions for these resources and grants these permissions to the Kubeconfig **User** by using a **RoleBinding**. An example of a **Role** and **RoleBinding** are provided below.

Please note that in this setup:

- All resources that need to be protected should be explicitly listed in the **Role** definition since Kubernetes RBAC is purely additive and does not support deny rules.  
- If Rubrik does not have permissions on a specific resource during the backup process, that resource will be omitted. However, the backup operation will still show as successful in Rubrik events.  
  - Future enhancements to highlight skipped resources resulting from inadequate permissions are being planned.

**Export Workflow**: Similar to [Option B](#option-b-protection-for-all-resources-in-specific-namespaces), to support export to a new namespace, create an empty namespace with the necessary **Role** and **RoleBinding** before exporting a snapshot. This **Role** should provide `get`, `list`, and `create` permissions on all resources that will be restored to the namespace.

An example **Role** and **RoleBinding** for protecting only the `configmaps`, `secrets`, and `deployments` in the `default` namespace.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: rubrik-backup-recovery-role
  namespace: default
rules:
# Add permissions for all the resources that need to be protected in the namespace.
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["get", "list", "create"]
- apiGroups: [""]
  resources: ["configmaps", "secrets"]
  verbs: ["get", "list", "create"]
```

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: rubrik-backup-recovery-role-binding
  namespace: default
roleRef:
  kind: Role
  name: rubrik-backup-recovery-role
  apiGroup: rbac.authorization.k8s.io
subjects:
- kind: User
  name: rubrik-kubeconfig-user
  apiGroup: rbac.authorization.k8s.io
```

## Step 3: Add the K8s Cluster to RSC

After ensuring permissions are set in [Step 2](#step-2-assign-cluster-scoped-and-backuprecovery-permissions-to-the-kubeconfig), use the Kubeconfig to add the K8s cluster in RSC. This will create the `rubrik-kupr` namespace in the K8s Cluster.

If you’re using an **AWS EKS** cluster, When executing the [`eksctl`](https://eksctl.io/usage/iam-identity-mappings/) command, update the `--username` argument to match the Kubeconfig **User** `rubrik-kubeconfig-user`.

```bash
eksctl create iamidentitymapping --cluster=<cluster-name> --region=<region-code> --arn=<rubrik-cloud-account-role-arn> --username=rubrik-kubeconfig-user --no-duplicate-arns
```

## Step 4: Assign rubrik-kupr namespace permissions to the Kubeconfig

After the cluster is added to RSC in [Step 3](#step-3-add-the-k8s-cluster-to-rsc), make sure to update the permissions in the `rubrik-kupr` namespace by creating a new **Role** and **RoleBinding** with the provided definition.

Note that the Kubeconfig doesn't need to be updated in RSC after adding more permissions in this step unless the user credentials in the Kubeconfig change.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: rubrik-kupr
  name: rubrik-kubeconfig-role
rules:
# Required to deploy the Rubrik backup agent, deploy the CRD controller and configure the load balancer service
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["create", "delete", "get", "update", "patch"]
- apiGroups: [""]
  resources: ["persistentvolumeclaims"]
  verbs: ["create", "delete", "get"]
- apiGroups: [""]
  resources: ["secrets", "serviceaccounts", "services"]
  verbs: ["create", "get", "update", "patch"]

# Required to create sample Roles for CRD RBAC
- apiGroups: ["rbac.authorization.k8s.io"]
  resources: ["roles"]
  verbs: ["create", "get", "update", "patch"]
```

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: rubrik-kubeconfig-role-binding
  namespace: rubrik-kupr
roleRef:
  kind: Role
  name: rubrik-kubeconfig-role
  apiGroup: rbac.authorization.k8s.io
subjects:
- kind: User
  name: rubrik-kubeconfig-user
  apiGroup: rbac.authorization.k8s.io
```

## Verification

After completing all the above 4 steps, the refresh of the K8s cluster should succeed.
You can proceed to create Protection Sets and configure backups to protect the resources.
