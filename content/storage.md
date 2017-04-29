# Persistent storage for containers
Linux containers are ephemeral by nature, meaning the container filesystem disappears when a container is deleted. Linux containers were initially created for stateless applications where there is no needing to store data surviving the container itself. Howewer, some containers need to store data somewhere and data need to survive to the container itself.

In this section, we're going to introduce two new OpenShift abstractions called "**Persistent Volume**" and "**Persistent Volume Claim**" to deal with persistent storage issue of Linux containers.



