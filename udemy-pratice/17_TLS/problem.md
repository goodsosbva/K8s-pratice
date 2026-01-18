# QUESTION 17: TLS

There is an existing deployment called `nginx-static` in the `nginx-static` namespace.

The deployment contains a `ConfigMap` that supports `TLSv1.2` and `TLSv1.3`, and a `Secret` for TLS.

There is a service called `nginx-static` in the `nginx-static` namespace that is currently exposing the deployment.

**Tasks:**

1. Configure the `configmap` to only support `TLSv1.3`.

2. Add the IP address of the service in `/etc/hosts` and name `ITKiddie.k8s.local`.

3. Verify that everything is working using the following command:
   - `curl --tls-max 1.2 https://ITKiddie.k8s.local -k` (TLSv1.2 should not work)
   - `curl --tlsv1.3 https://ITKiddie.k8s.local -k`
