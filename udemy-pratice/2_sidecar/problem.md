# QUESTION 02

Update the existing deployment wordpress, adding a sidecar container named sidecar
using the busybox:stable image to the existing pod.

The new sidecar container has to run the following command: "/bin/sh -c "tail -f /var/log/wordpress.log"
use a volume mounted at /var/log to make the log file wordpress.log available to the co-located container
