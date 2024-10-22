# Kubernetes Protection Least Privileges Configuration

To ensure security and compliance, it's important to assign the least privileges necessary for Rubrik to perform its tasks. This repository provides Kubernetes ClusterRole and Role definitions that encapsulate these permissions for utilizing Rubrik Kubernetes Protection.

Follow the instructions in the [Documentation](Documentation.md) for configuring the required permissions.

## :blue_book: Roles and ClusterRoles

- **[rubrik-kubeconfig-cluster-role-for-openshift](cluster-roles/rubrik-kubeconfig-cluster-role-for-openshift.yml)**: *ClusterRole* with permissions required at a cluster level for the Rubrik Kubeconfig User on Redhat Openshift Clusters.

- **[rubrik-kubeconfig-cluster-role](cluster-roles/rubrik-kubeconfig-cluster-role.yml)**: *ClusterRole* with permissions required at a cluster level for the Rubrik Kubeconfig User on all other Kubernetes Clusters.

- **[rubrik-backup-recovery-cluster-role](cluster-roles/rubrik-backup-recovery-cluster-role.yml)**: *ClusterRole* with permissions on all resources to perform backup and recovery for the Rubrik Kubeconfig User.

- **[rubrik-kubeconfig-role](roles/rubrik-kubeconfig-role.yml)**: *Role* with permissions required in the `rubrik-kupr` namespace for the Rubrik Kubeconfig User.

## :white_check_mark: Usage

Create the required *ClusterRoles*, *Roles* and their corresponding *ClusterRoleBindings*, *RoleBindings* with the Rubrik Kubeconfig User. The Kubeconfig for this User can be used to add the Kubernetes cluster to RSC.

## :muscle: How You Can Help

We glady welcome contributions from the community. From updating the documentation to requesting additional Role, ClusterRole definitions, all ideas are welcome. Thank you in advance for all of your issues, pull requests, and comments! :star:

- [Contributing Guide](CONTRIBUTING.md)
- [Code of Conduct](Code-of-conduct.md)

## :pushpin: License

- [MIT License](LICENSE)

## :point_right: About Rubrik Build

We encourage all contributors to become members. We aim to grow an active, healthy community of contributors, reviewers, and code owners. Learn more in our [Welcome to the Rubrik Build Community](https://github.com/rubrikinc/welcome-to-rubrik-build) page.

We'd love to hear from you! Email us: <build@rubrik.com> :love_letter:
