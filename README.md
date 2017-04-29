# OpenShift Lab Tutorial
OpenShift Lab Tutorial based on CentOS distribution.

**Openshift** is a layered platform from Red Hat built around **Docker** and **Kubernetes** with a focus on easy development and deployment of applications by developers. Docker provides the abstraction for packaging and creating Linux containers. Kubernetes provides the cluster management and orchestrates Docker containers on multiple hosts. On top of this, OpenShift Container Platform adds:

 * Source code management, builds, and deployments for developers
 * Operations tools for application management at scale
 * Team and users facilities for large developer organization

This tutorial contains simple examples of using OpenShift Platform. For more details on the OpenShift Platform, please refer to the product documentation at [docs.openshift.com](https://docs.openshift.com/index.html).

## Content
1. [Setup](./content/preflight.md)
    * [Architecture](./content/preflight.md#architecture)
    * [Security](./content/preflight.md#security)
    * [Install Docker Engine](./content/preflight.md#install-docker)
    * [Host Access](./content/preflight.md#host-access)
    * [Install OpenShift](./content/preflight.md#install-openshift)
2. [Core concepts](./content/basics.md)
    * [Create a demo user](./content/basics.md#create-a-demo-user)
    * [Create a demo project](./content/basics.md#create-a-demo-project)
    * [Create a pod](./content/basics.md#create-a-pod)
    * [Create a service](./content/basics.md#create-a-service)
    * [Create a replica controller](./content/basics.md#create-a-replica-controller)
3. [Routing Layer](./content/routing.md)
    * [Expose the Service](./content/routing.md#expose-the-service)
4. [Projects](./content/projects.md)
    * [Project Permissions](./content/projects.md#project-permissions)
    * [Quota and Limits](./content/projects.md#project-quotas-and-limits)
5. [Templates](./content/templates.md)
    * [Create a template from objects](./content/templates.md#create-a-template-from-existing-objects)
    * [Create applications from templates](./content/templates.md#create-an-application-from-a-template)
6. [Applications](./content/applications.md)
     * [Create applications from images](./content/applications.md#create-applications-from-images)
     * [Image Stream](./content/applications.md#image-stream)
     * [Deployment Config](./content/applications.md#deployment-config)
     * [Trigger a new Deployment Config](./content/applications.md#trigger-a-new-deployment-config)
     * [Create applications from source code](./content/applications.md#create-applications-from-source-code)
     * [Docker build strategy](./content/applications.md#docker-build-strategy)
     * [Build Config](./content/applications.md#build-config)
     * [Source build strategy](./content/applications.md#source-build-strategy)
     * [Webhooks](./content/applications.md#webhooks)
7. [Persistent Storage](./content/info.md)
     * [Volumes](./content/info.md)
     * [Persistent Data](./content/info.md)
     * [Storage Plugins](./content/info.md)
8. [Cluster Administration](./content/info.md)
     * [High Availability](./content/info.md)
     * [Cluster Scaling](./content/info.md)
     * [Security](./content/info.md)
     * [Users Management](./content/info.md)

## Disclaimer
This tutorial is for personal use only. This is just a lab guide, not a documentation for Openshift, Docker or Kubernetes, please go to their online documentation sites for more details about how do they work.
