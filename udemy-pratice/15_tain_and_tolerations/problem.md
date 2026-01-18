# QUESTION 15: Taints & Tolerations

**Task:**

1. Add a taint to node01 so that no normal pods can be scheduled in the node.
   - Key=IT
   - Value=Kiddie
   - Type=NoSchedule

2. Schedule a Pod on node01 adding the correct toleration to the spec and ensure that it lands on the correct node.
