# Deployment Core

## Overview

### Project Structure

The Helmfile setup follows a structured directory layout to manage environments, releases, and configurations efficiently:

```
project-root/
│── charts.yaml              # Defines available Helm charts and repositories
│── helmfile.yaml.gotmpl     # Main Helmfile configuration
│── environments/            # Contains per-environment configurations
│   │── production/          # Example environment folder
│   │   │── cluster.yaml     # Defines environment-specific settings and enabled releases
│   │   │── values/          # Stores custom values per release
│   │   │── secrets/         # Contains encrypted secrets managed by SOPS
```

This structure helps maintain modularity and separation of concerns, making it easier to manage multiple environments.

This Helmfile template provides a structured approach for managing Kubernetes cluster deployments across multiple environments. By leveraging Helmfile's templating capabilities, this setup ensures:

- **Modular release management**: Centralized `charts.yaml` for defining available Helm charts and their repositories, while each environment defines its enabled releases in `cluster.yaml` under the key `releases` .
- **Environment-specific configurations**: Dynamically reads `cluster.yaml` per environment.
- **Flexible values management**: Supports custom `values/` and encrypted `secrets/` per environment.

This document explains how to use this Helmfile setup to **add new releases**, **enable them for specific environments**, and **manage environment-specific configurations**.

---

## 1. Adding a New Release

A release represents a Helm chart deployment that can be enabled per environment.

### **Steps to Add a New Release:**

1. **Define the release** in `charts.yaml`
   Add a new entry under the `releases` section in `charts.yaml`:

   ```yaml
   releases:
     my-app:
       namespace: my-namespace
       chart: my-repo/my-app
   ```

   - `namespace`: The Kubernetes namespace where the application should be deployed. This can be omitted and left to be environment specific.
   - `chart`: The Helm chart repository and name. If the repository doesn't exist, it needs to be added first.

2. **Enable the release for a specific environment**
   Edit `environments/<environment>/cluster.yaml` and add the release under `releases:`

   ```yaml
   environment:
     kubeContext: my-cluster
   releases:
     my-app:
   ```

   - One can define additional per release configuration for environment-specific overrides. For example, adding hooks or dependencies on other releases. Refer to [helmfile documentation](https://helmfile.readthedocs.io/en/latest/#configuration) for release spec definition.
   - This enables the release only for this environment.

3. **(Optional) Add custom values and secrets**

   - To override Helm values per environment, create a file:
     ```sh
     touch environments/<environment>/values/my-app.yaml.gotmpl
     ```
   - To manage secrets securely with SOPS, create:
     ```sh
     touch environments/<environment>/secrets/my-app.yaml
     ```
     And encrypt it using SOPS:
     ```sh
     sops -e -i environments/<environment>/secrets/my-app.yaml
     ```

---

## 2. Running Helmfile for a Specific Environment

Once a release is added and enabled for an environment, you can deploy it using Helmfile. Ensure that you are executing the command from the **project root directory** where `helmfile.yaml.gotmpl` is located.

### **Deploying an Environment**

```sh
helmfile -e <environment> apply
```

- This applies the configurations for the specified environment.
- Helmfile will:
  - Load the appropriate `cluster.yaml`
  - Merge values from `values/` and `secrets/`
  - Deploy the enabled releases

### **Preview Changes Before Applying**

To check what changes will be applied:

```sh
helmfile -e <environment> diff
```

---

## 3. Implementing GitOps with This Approach

This Helmfile-based deployment strategy aligns well with GitOps principles by ensuring that the Kubernetes cluster state is managed declaratively and reconciled automatically. The key components of GitOps implementation in this setup are:

### **1. Declarative State Management**

- All Kubernetes resources, including Helm releases and configurations, are defined in YAML files and stored in Git.
- `charts.yaml` acts as the central source of truth for Helm chart versions and configurations.
- Environment-specific settings are maintained under `environments/`, ensuring that the desired state for each environment is version-controlled.

### **2. Drift Detection Using Scheduled CI/CD Jobs**

- A scheduled CI/CD pipeline runs periodically to check for configuration drift.
- The pipeline executes:
  ```sh
  helmfile -e <environment> diff
  ```
  to compare the desired state in Git with the actual cluster state.
- If a drift is detected, the pipeline can either:
  - Automatically reconcile the state by running `helmfile apply`.
  - Notify the DevOps team for manual intervention.

### **3. Event-Driven Sync with Kubernetes Event Exporter**

- A more efficient approach involves using [Kubernetes Event Exporter]\([https://github.com/resmoio/kubernetes-event-exporter](https://github.com/resmoio/kubernetes-event-exporter)) to listen for changes in resources annotated with `app.kubernetes.io/managed-by` .
- When a relevant resource is modified, an event is sent to a webhook that triggers the CD pipeline for reconciliation.
- There will be an unnecessary trigger for the CD pipeline as the reconciliation will also result in an update event firing, but since the resources are reconciled no further update events will be fired.

### **Conclusion**

This GitOps implementation ensures that cluster configurations are always in sync with the desired state defined in Git. It enables both **scheduled drift detection** and **event-driven reconciliation**, providing an automated and efficient way to manage Kubernetes deployments.

