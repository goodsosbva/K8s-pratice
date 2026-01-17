# QUESTION 04

You are managing a WordPress application running in a Kubernetes cluster.
Your task is to adjust the Pod resource requests and limits to ensure stable operation. Follow the instructions below:

1. Scale down the wordpress Deployment to 0 replicas.
2. Edit the Deployment and divide node resources evenly across all 3 Pods.
3. Assign fair and equal CPU and memory requests to each Pod.
4. Add sufficient overhead to avoid node instability.

Ensure that both the init containers and main containers use exactly the same resource requests and limits.
After making the changes, scale the Deployment back to 3 replicas.
