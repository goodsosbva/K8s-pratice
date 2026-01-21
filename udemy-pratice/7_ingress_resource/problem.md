# QUESTION 07: Ingress Resource

Create a new ingress resource named `echo` in `echo-sound` namespace.

With the following tasks:

1. Expose the deployment with a service named `echo-service` on `http://example.org/echo` using Service port 8080 type NodePort.

2. The availability of Service `echo-service` can be checked using the following command which should return 200:
   ```bash
   curl -o /dev/null -s -w "%{http_code}\n" http://example.org/echo
   ```
