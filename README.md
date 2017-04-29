# OpenShift Lab Tutorial
OpenShift Lab Tutorial based on CentOS distribution.

**Openshift** is a layered platform from Red Hat built around **Docker** and **Kubernetes** with a focus on easy development and deployment of applications by developers. Docker provides the abstraction for packaging and creating Linux containers. Kubernetes provides the cluster management and orchestrates Docker containers on multiple hosts. On top of this, OpenShift Container Platform adds:

 * Source code management, builds, and deployments for developers
 * Operations tools for application management at scale
 * Team and users facilities for large developer organization

This tutorial contains simple examples of using OpenShift Platform. For more details on the OpenShift Platform, please refer to the product documentation at [docs.openshift.com](https://docs.openshift.com/index.html).

## Content
1. [Setup](./content/preflight.md)
    * [Architecture](https://github.com/kalise/OpenShift-Tutorial/blob/master/content/preflight.md#architecture)
    * [Security](https://github.com/kalise/OpenShift-Tutorial/blob/master/content/preflight.md#security)
    * [Install Docker Engine](https://github.com/kalise/OpenShift-Tutorial/blob/master/content/preflight.md#install-docker)
    * [Host Access](https://github.com/kalise/OpenShift-Tutorial/blob/master/content/preflight.md#host-access)
    * [Install OpenShift](https://github.com/kalise/OpenShift-Tutorial/blob/master/content/preflight.md#install-openshift)
2. [Core concepts](https://github.com/kalise/OpenShiftPlatform-Tutorial/blob/master/content/basics.md)
    * [Create a demo user](https://github.com/kalise/OpenShift-Tutorial/blob/master/content/basics.md#create-a-demo-user)
    * [Create a demo project](https://github.com/kalise/OpenShiftPlatform-Tutorial/blob/master/content/basics.md#create-a-demo-project)
    * [Create a pod](https://github.com/kalise/OpenShiftPlatform-Tutorial/blob/master/content/basics.md#create-a-pod)
    * [Create a service](https://github.com/kalise/OpenShiftPlatform-Tutorial/blob/master/content/basics.md#create-a-service)
    * [Create a replica controller](https://github.com/kalise/OpenShiftPlatform-Tutorial/blob/master/content/basics.md#create-a-replica-controller)
3. [Routing Layer](https://github.com/kalise/OpenShiftPlatform-Tutorial/blob/master/content/routing.md)
    * [Expose the Service](https://github.com/kalise/OpenShift-Tutorial/blob/master/content/routing.md#expose-the-service)
4. [Projects](https://github.com/kalise/OpenShiftPlatform-Tutorial/blob/master/content/projects.md)
    * [Project Permissions](https://github.com/kalise/OpenShift-Tutorial/blob/master/content/projects.md#project-permissions)
    * [Quota and Limits](https://github.com/kalise/OpenShift-Tutorial/blob/master/content/projects.md#project-quotas-and-limits)
5. [Templates](https://github.com/kalise/OpenShiftPlatform-Tutorial/blob/master/content/templates.md)
    * [Create a template from objects](https://github.com/kalise/OpenShift-Tutorial/blob/master/content/templates.md#create-a-template-from-existing-objects)
    * [Create applications from templates](https://github.com/kalise/OpenShift-Tutorial/blob/master/content/templates.md#create-an-application-from-a-template)
6. [Applications](https://github.com/kalise/OpenShiftPlatform-Tutorial/blob/master/content/applications.md)
     * [Create applications from images](https://github.com/kalise/OpenShiftPlatform-Tutorial/blob/master/content/applications.md#create-applications-from-images)
     * [Image Stream](https://github.com/kalise/OpenShiftPlatform-Tutorial/blob/master/content/applications.md#image-stream)
     * [Deployment Config](https://github.com/kalise/OpenShiftPlatform-Tutorial/blob/master/content/applications.md#deployment-config)
     * [Trigger a new Deployment Config](https://github.com/kalise/OpenShiftPlatform-Tutorial/blob/master/content/applications.md#trigger-a-new-deployment-config)
     * [Create applications from source code](https://github.com/kalise/OpenShiftPlatform-Tutorial/blob/master/content/applications.md#create-applications-from-source-code)
     * [Docker build strategy](https://github.com/kalise/OpenShiftPlatform-Tutorial/blob/master/content/applications.md#docker-build-strategy)
     * [Build Config](https://github.com/kalise/OpenShiftPlatform-Tutorial/blob/master/content/applications.md#build-config)
     * [Source build strategy](https://github.com/kalise/OpenShiftPlatform-Tutorial/blob/master/content/applications.md#source-build-strategy)
     * [Webhooks](https://github.com/kalise/OpenShiftPlatform-Tutorial/blob/master/content/applications.md#webhooks)
7. [Persistent Storage](https://github.com/kalise/OpenShiftPlatform-Tutorial/blob/master/content/info.md)
     * [Volumes](https://github.com/kalise/OpenShiftPlatform-Tutorial/blob/master/content/info.md)
     * [Persistent Data](https://github.com/kalise/OpenShiftPlatform-Tutorial/blob/master/content/info.md)
     * [Storage Plugins](https://github.com/kalise/OpenShiftPlatform-Tutorial/blob/master/content/info.md)
8. [Cluster Administration](https://github.com/kalise/OpenShiftPlatform-Tutorial/blob/master/content/info.md)
     * [High Availability](https://github.com/kalise/OpenShiftPlatform-Tutorial/blob/master/content/info.md)
     * [Security](https://github.com/kalise/OpenShiftPlatform-Tutorial/blob/master/content/info.md)
     * [Cluster Scaling](https://github.com/kalise/OpenShiftPlatform-Tutorial/blob/master/content/info.md)
     * [Users Management](https://github.com/kalise/OpenShiftPlatform-Tutorial/blob/master/content/info.md)

## Disclaimer
This tutorial is for personal use only. This is just a lab guide, not a documentation for Openshift, Docker or Kubernetes, please go to their online documentation sites for more details about how do they work.
