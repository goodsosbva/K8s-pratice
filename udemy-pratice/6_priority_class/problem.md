# QUESTION 06: Priority Class

You're working in a Kubernetes cluster with an existing Deployment named `busybox-logger` running in a namespace called `priority`.

The cluster already has at least one user-defined Priority Class.

Perform the following tasks:

1. Create a new Priority Class named `high-priority` for user workloads. The value of this Priority Class should be exactly one less than the highest existing user-defined Priority Class value.

2. Patch the existing Deployment `busybox-logger` in the `priority` namespace to use the newly created `high-priority` Priority Class.
