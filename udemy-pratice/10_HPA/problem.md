# QUESTION 10: HPA

Create a new HorizontalPodAutoscaler [HPA] named `apache-server` in the `autoscale` namespace.

**Tasks:**

1. This HPA must target the existing deployment called `apache-deployment` in the `autoscale` namespace.

2. Set the HPA to target for 50% CPU usage per Pod.

3. Configure the HPA to have a minimum of 1 pod and maximum of 4 pods. Also, we have to set the downscale stabilization window to 30 seconds.
