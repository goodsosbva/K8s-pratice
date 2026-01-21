# QUESTION 1: Argo CD

Install Argo CD in a Kubernetes cluster using Helm while ensuring that CRDs are not installed (as they are pre-installed).

**Requirements:**

1. Add the official Argo CD Helm repository with the name `argo`.

2. Generate a Helm template from the Argo CD chart version `7.7.3` for the `argocd` namespace.

3. Ensure that CRDs are not installed by configuring the chart accordingly.

4. Save the generated YAML manifest to `/home/argo/argo-helm.yaml`.
