# Applications
In the real world of applications development and deployment, OpenShift users need a repository where they can pull docker images and then deploy the application on OpenShift. Another approach is to use the OpenShift to develop and deploy the application starting from a source code so that they can build the docker images on OpenShift itself and then deploy the applications. In this section, we're going to walk through both the approaches.

  * [Create applications from images](#create-applications-from-images)
  * [Image Stream](#image-stream)
  * [Deployment Config](#deployment-config)
  * [Trigger a new Deployment Config](#trigger-a-new-deployment-config)
  * [Create applications from source code](#create-applications-from-source-code)
  * [Docker build strategy](#docker-build-strategy)
  * [Build Config](#build-config)
  * [Source build strategy](#source-build-strategy)
  * [Webhooks](#webhooks)

## Create applications from images
Users can deploy an application from an already built docker image. Images can be pulled from the reposistory in the OpenShift Container Platform or can be pulled from any public or local registry.

As demo user, create a new application strarting from the latest Hello World docker [image](docker.io/kalise/nodejs-web-app) from the Docker Hub public reposistory
```
[demo@master ~]$ oc new-app --name=nodejs \
                    docker.io/kalise/nodejs-web-app \
                    -e MESSAGE="Hello New Application"
                    
--> Found Docker image 5770203 (2 hours old) from docker.io for "docker.io/kalise/nodejs-web-app"
    * An image stream will be created as "nodejs:latest" that will track this image
    * This image will be deployed in deployment config "nodejs"
    * Port 8080/tcp will be load balanced by service "nodejs"
    * Other containers can access this service through the hostname "nodejs"
    * WARNING: Image "docker.io/kalise/nodejs-web-app" runs as the 'root' user ...
--> Creating resources ...
    imagestream "nodejs" created
    deploymentconfig "nodejs" created
    service "nodejs" created
--> Success
    Run 'oc status' to view your app.
```

See all the created objects
```
[demo@master ~]$ oc get all
NAME        DOCKER REPO                       TAGS      UPDATED
is/nodejs   172.30.222.143:5000/demo/nodejs   latest    7 minutes ago

NAME        REVISION   DESIRED   CURRENT   TRIGGERED BY
dc/nodejs   1          1         1         config,image(nodejs:latest)

NAME          DESIRED   CURRENT   READY     AGE
rc/nodejs-1   1         1         1         7m

NAME         CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
svc/nodejs   172.30.123.185   <none>        8080/TCP   7m

NAME                READY     STATUS    RESTARTS   AGE
po/nodejs-1-kb89v   1/1       Running   0          3m
```

The command above does not expose the service, so a manual route should be created.
```
[demo@master ~]$ oc expose service nodejs --hostname=nodejs.cloud.openshift.com
route "nodejs" exposed

[demo@master ~]$ oc get route
NAME      HOST/PORT                        PATH      SERVICES   PORT       TERMINATION
nodejs    nodejs.cloud.openshift.com                 nodejs     8080-tcp

[demo@master ~]$ curl nodejs.cloud.openshift.com
Hello New Application
```

Now we'll pay attention to a couple of new resources: the **Image Stream** and the **Deployment Config**

### Image Stream
The Image Stream resource tells OpenShift when the referenced image changes and then take action based on that changes. In our example, if the ``nodejs-web-app`` image changed, the OpenShift platform automatically trigger a new deployment of our application.

Inspect the Image Stream above 
```
[demo@master ~]$ oc get is -n demo
NAME      DOCKER REPO                       TAGS      UPDATED
nodejs    172.30.222.143:5000/demo/nodejs   latest    2 hours ago

[demo@master ~]$ oc get is/nodejs -o yaml
```

Here the yaml of the above Image Stream
```yaml
apiVersion: v1
kind: ImageStream
metadata:
  annotations:
...
  creationTimestamp: 2017-02-06T14:31:31Z
  generation: 2
  labels:
    app: nodejs
  name: nodejs
  namespace: test
  resourceVersion: "1938983"
...
spec:
  tags:
  - annotations:
      openshift.io/imported-from: docker.io/kalise/nodejs-web-app
    from:
      kind: DockerImage
      name: docker.io/kalise/nodejs-web-app
    generation: 2
    importPolicy: {}
    name: latest
status:
  dockerImageRepository: 172.30.222.143:5000/demo/nodejs
  tags:
  - items:
    - created: 2017-02-06T14:31:31Z
      dockerImageReference: 172.30.222.143:5000/demo/nodejs@sha256:...
      generation: 2
      image: sha256:330ba793146c9c8a3c1c4ea35cb12f3c3928ce5385a7ac8f3a285347a49cb353
```

This Image Stream references an image originated from public repository ``docker.io/kalise/nodejs-web-app``. This image has been pulled by OpenShift during the application creation process and stored in the local OpenShift registry at ``172.30.222.143:5000/demo/nodejs``. Any changes to the image in the local OpenShift registry will trigger a new deployment of the application.

### Deployment Config
OpenShift adds expanded support for the software development and deployment lifecycle with the concept of deployments. In the simplest case, a deployment just creates a new replica controller and lets it start up the pods. Also, an OpenShift deployment is able to transition from an existing deployment of an image to a new one.

Inspect the Deployment Config above 
```
[demo@master ~]$ oc get dc/nodejs
NAME      REVISION   DESIRED   CURRENT   TRIGGERED BY
nodejs    2          1         1         config,image(nodejs:latest)

[demo@master ~]$ oc get dc/nodejs -o yaml
```

The OpenShift DeploymentConfiguration object defines the following details of a deployment:
  
  * The elements of a ReplicationController definition.
  * Triggers for creating a new deployment automatically.
  * The strategy for transitioning between deployments.
  * Life cycle hooks.

Each time a new deployment is triggered, a special deployer pod takes care of starting the new replica controller while scaling down the old replica controller, and, if defined, run the hooks. The deployment pod remains for an indefinite amount of time after it completes the deployment in order to retain its logs of the deployment. When a deployment is superseded by another, the previous replica controller is retained to enable easy rollback if needed.

Here some relevant details about the triggering conditions for the deployment above
```yaml
apiVersion: v1
kind: DeploymentConfig
...
spec:
  triggers:
  - type: ConfigChange
  - imageChangeParams:
      automatic: true
      containerNames:
      - nodejs
      from:
        kind: ImageStreamTag
        name: nodejs:latest
        namespace: test
      lastTriggeredImage:172.30.222.143:5000/demo/nodejs@sha256:...91535a
    type: ImageChange
...
```

A ``ConfigChange`` trigger causes a new deployment to be created any time a replica controller changes. An ``ImageChange`` trigger causes a new deployment to be created each time a new version of the image is available.

The user defines customizable strategies to transition from the previous deployment to the new deployment. Each application has different requirements for availability during deployments. OpenShift provides strategies to support a variety of deployment scenarios. Here the  relevant details about the strategy for the deployment above
```yaml
apiVersion: v1
kind: DeploymentConfig
...
spec:
  ...
  strategy:
    resources: {}
    rollingParams:
      intervalSeconds: 1
      maxSurge: 25%
      maxUnavailable: 25%
      timeoutSeconds: 600
      updatePeriodSeconds: 1
    type: Rolling
...
```

The Rolling strategy is the default strategy. The Rolling strategy will scale up the new deployment based on the max surge configuration. Then, it scale down the old deployment based on the max unavailable configuration. It repeat this scaling until the new deployment has reached the desired replica count and the old deployment has been scaled to zero. When scaling down, the Rolling strategy waits for pods to become ready. If scaled up pods never become ready, the deployment will time out and result in a deployment failure. The deployment strategy uses readiness checks to determine if a new pod is ready for use. If a readiness check fails, the deployment is stopped.

The other option is the Recreate strategy. It will scale down the previous deployment to zero and then scale up the new deployment. See OpenShift official docs for details. 

The deployment configuration contains a version number that is incremented each time a new deployment is created from that configuration. In addition, the cause of the last deployment is added to the configuration.
```yaml
apiVersion: v1
kind: DeploymentConfig
...
status:
  availableReplicas: 1
  conditions:
  - lastTransitionTime: 2017-02-06T18:44:19Z
    message: Replication controller "nodejs-3" has completed progressing
    reason: NewReplicationControllerAvailable
    status: "True"
    type: Progressing
  details:
    causes:
    - imageTrigger:
        from:
          kind: ImageStreamTag
          name: nodejs:latest
          namespace: test
      type: ImageChange
    message: image change
  latestVersion: 1
  observedGeneration: 2
  replicas: 1
  updatedReplicas: 1
```

In OpenShift, users can manually force a new deploy to start. Start a new deployment manually:
```
[demo@master ~]$ oc rollout latest dc/nodejs
deploymentconfig "nodejs" rolled out
```

To get basic information about all the available revisions of the application
```
[demo@master ~]$ oc rollout history dc/nodejs
deploymentconfigs "nodejs"
REVISION        STATUS          CAUSE
1               Complete        image change
2               Complete        image change
3               Complete        image change
4               Complete        config change
5               Running         manual change
```
Above we see latest deployment still running. It is triggered by a manual change. Other changes have been triggered by config cahnges and/or image changes.

Rollback a deployment
```
[demo@master ~]$ oc rollout undo dc/nodejs
deploymentconfig "nodejs" rolled back

[demo@master ~]$ oc rollout history dc/nodejs
deploymentconfigs "nodejs"
REVISION        STATUS          CAUSE
1               Complete        image change
2               Complete        image change
3               Complete        image change
4               Complete        config change
5               Complete        manual change
6               Running         manual change
```

Please, note that a rollback does not remove the current deployment. It instead, create a new deployment based on the previous release. Rollback to a specific release

```
[demo@master ~]$ oc rollout undo dc/nodejs --to-revision=2
deploymentconfig "nodejs" rolled back
```

### Trigger a new Deployment Config
Thanks to the Image Stream construct, any changes to the image in the local OpenShift registry will trigger a new Deployment Config of the application. To test this feature, as root user, check the nodejs images in the local repository
```
[root@master ~]# oc get images | grep nodejs
NAME
DOCKER REF
sha256:330ba793146c9c8a3c1c4ea35cb12f3c3928ce5385a7ac8f3a285347a49cb353
172.30.222.143:5000/demo/nodejs@sha256:...
```

We see a nodejs image with ID ``...9cb353`` as the same ID referenced in the image stream above.

Manually build a new image and then push it to the local OpenShift registry. As root user, build a new docker image of our Hello Worl application and tag it for pushing on the local OpenShift registry
```
[root@master ~]# git clone https://github.com/kalise/nodejs-web-app
[root@master ~]# cd nodejs-web-app
[root@master nodejs-web-app]# docker build -t nodejs-web-app .
[root@master nodejs-web-app]# docker tag nodejs 172.30.222.143:5000/demo/nodejs:latest
```

Login as demo user to the local OpenShift registry to get an authentication token and then push the new image
```
[root@master nodejs-web-app]# docker login -u demo -p `oc whoami -t` 172.30.222.143:5000
[root@master nodejs-web-app]# docker push 172.30.222.143:5000/demo/nodejs
```

Once the push ends, a new deployment starts. As demo user, check this process in real time
```
[demo@master ~]$ oc get all
NAME        DOCKER REPO                       TAGS      UPDATED
is/nodejs   172.30.222.143:5000/demo/nodejs   latest    21 seconds ago

NAME        REVISION   DESIRED   CURRENT   TRIGGERED BY
dc/nodejs   2          1         1         config,image(nodejs:latest)

NAME          DESIRED   CURRENT   READY     AGE
rc/nodejs-1   1         1         1         4h
rc/nodejs-2   1         1         0         21s

NAME            HOST/PORT                           PATH      SERVICES   PORT       TERMINATION
routes/nodejs   nodejs.cloud.openshift.com          nodejs     8080-tcp

NAME         CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
svc/nodejs   172.30.123.185   <none>        8080/TCP   5h

NAME                 READY     STATUS              RESTARTS   AGE
po/nodejs-1-rw25i    1/1       Running             0          4h
po/nodejs-2-3i6nh    0/1       ContainerCreating   0          6s
po/nodejs-2-deploy   1/1       Running             0          20s
```

Once the latest deployment finishes, check its status
```
[demo@master ~]$ oc describe dc/nodejs
Name:           nodejs
Namespace:      test
Created:        5 hours ago
Labels:         app=nodejs
Annotations:    openshift.io/generated-by=OpenShiftNewApp
Latest Version: 3
Selector:       app=nodejs,deploymentconfig=nodejs
Replicas:       1
Triggers:       Config, Image(nodejs@latest, auto=true)
Strategy:       Rolling
Template:
  Labels:       app=nodejs
                deploymentconfig=nodejs
  Annotations:  openshift.io/container.nodejs.image.entrypoint=["npm","start"]
                openshift.io/generated-by=OpenShiftNewApp
  Containers:
   nodejs:
    Image:              172.30.222.143:5000/demo/nodejs@sha256:
    56c996c14b6a25d27ace11d2e823c54feb60edf0c263b96004f1a6114391535a
    Port:               8080/TCP
    Volume Mounts:      <none>
    Environment Variables:
      MESSAGE:  Hello App
  No volumes.

Deployment #2 (latest):
        Name:           nodejs-2
        Created:        8 minutes ago
        Status:         Complete
        Replicas:       1 current / 1 desired
        Selector:       app=nodejs,deployment=nodejs-2,deploymentconfig=nodejs
        Labels:         app=nodejs,openshift.io/deployment-config.name=nodejs
        Pods Status:    1 Running / 0 Waiting / 0 Succeeded / 0 Failed
Deployment #1:
        Created:        4 hours ago
        Status:         Complete
        Replicas:       0 current / 0 desired
...
```

In this test, we saw the OpenShift start a new deployment every time the image in the local registry changes. Howewer, please note that changes of the original image in the docker repository will not trigger automatically any new deployment of the application.

# Create applications from source code
Users can create applications from source code in a local or remote Git repository. When users specify source code repository, OpenShift attempts to build the source into a new application image. OpenShift tries to automatically determine the type of build strategy to use.

There are, basically two build strategies: **Docker** or **Source**.

## Docker build strategy
If a Dockerfile is present in the surce code repository, OpenShift uses the Docker strategy invoking the ``docker build`` command to produce a runnable image. Create a Nodejs application starting from a remote Github reposistory ``https://github.com/kalise/nodejs-web-app.git`` containing the code for a simple httpd service
```
[demo@master ~]$ oc new-app https://github.com/kalise/nodejs-web-app.git
--> Found Docker image efe7b69 (5 days old) from Docker Hub for "node:latest"
    * An image stream will be created as "node:latest" that will track the source image
    * A Docker build using source code from https://github.com/kalise/nodejs-web-app.git will be created
      * The resulting image will be pushed to image stream "nodejs-web-app:latest"
      * Every time "node:latest" changes a new build will be triggered
    * This image will be deployed in deployment config "nodejs-web-app"
    * Port 8080 will be load balanced by service "nodejs-web-app"
      * Other containers can access this service through the hostname "nodejs-web-app"
    * WARNING: Image "node:latest" runs as the 'root' user which may not be permitted ...
--> Creating resources ...
    imagestream "node" created
    imagestream "nodejs-web-app" created
    buildconfig "nodejs-web-app" created
    deploymentconfig "nodejs-web-app" created
    service "nodejs-web-app" created
--> Success
    Build scheduled, use 'oc logs -f bc/nodejs-web-app' to track its progress.
    Run 'oc status' to view your app.
```

The above command creates a **build config**, which itself start a new **build** in order to produce a runnable image. Code is cloned locally from the remote git reposistory and then the runnable image ``nodejs-web-app:latest`` is created and then pushed to the local OpenShift registry. Since the code repository contains a dockerfile, the build strategy is *"Docker"* by default: the base nodejs image ``node:latest`` is also created and pushed to the local OpenShift registry.

The command also creates two image streams: ``node`` and ``nodejs-web-app`` to keep track of changes in base nodejs image and in the runnable image respectively. The command also create a new **deployment config** to deploy the runnable image, a **replica controller** which starts the pods and finally a **service** to provide load-balanced access to the pods. 
```
[demo@master ~]$ oc get all
NAME                TYPE      FROM      LATEST
bc/nodejs-web-app   Docker    Git       1

NAME                      TYPE      FROM          STATUS     STARTED          DURATION
builds/nodejs-web-app-1   Docker    Git@baba51f   Complete   10 minutes ago   4m28s

NAME                DOCKER REPO                               TAGS      UPDATED
is/node             172.30.222.143:5000/demo/node             latest    10 minutes ago
is/nodejs-web-app   172.30.222.143:5000/demo/nodejs-web-app   latest    6 minutes ago

NAME                REVISION   DESIRED   CURRENT   TRIGGERED BY
dc/nodejs-web-app   1          1         1         config,image(nodejs-web-app:latest)

NAME                  DESIRED   CURRENT   READY     AGE
rc/nodejs-web-app-1   1         1         1         6m

NAME                 CLUSTER-IP     EXTERNAL-IP   PORT(S)    AGE
svc/nodejs-web-app   172.30.62.84   <none>        8080/TCP   10m

NAME                        READY     STATUS      RESTARTS   AGE
po/nodejs-web-app-1-build   0/1       Completed   0          10m
po/nodejs-web-app-1-mc6mu   1/1       Running     0          5m
```

Now let's to go in details through the objects created by the OpenShift platform when creating a new application from source code.

## Build Config
Generally speaking, a build is the process of transforming input source code into a runnable image. The Build Config object created by OpenShift is the definition of the entire build process. A Build Config describes a single build definition and a set of triggers for when a new build should be created. 

Inspect the Build Config object above
```
[demo@master ~]$ oc get bc/nodejs-web-app -o yaml
```
Here some key points.

The source section, defines the source code repository location
```yaml
...
  source:
    git:
      uri: https://github.com/kalise/nodejs-web-app.git
    type: Git
...
```

The strategy section describes the build strategy used to execute the build
```yaml
...
  strategy:
    dockerStrategy:
      from:
        kind: ImageStreamTag
        name: node:latest
    type: Docker
...
```

The output section, defined where the runnable image is pushed after it is successfully built
```yaml
...
  output:
    to:
      kind: ImageStreamTag
      name: nodejs-web-app:latest
...
```

The trigger section, defines the criteria which cause a new build to be created
```yaml
...
  triggers:
  - github:
      secret: *******
    type: GitHub
  - generic:
      secret: *******
    type: Generic
  - type: ConfigChange
  - imageChange:
      lastTriggeredImageID: node@sha256:c7505048a3ddc2539b9b4d7c468e6ff0641f3a06ec95a4450be493fec8410c13
    type: ImageChange
...
```

OpenShift users can force a new build to happen even if no changes are in place
```
[demo@master ~]$ oc start-build bc/nodejs-web-app
build "nodejs-web-app-2" started
```

When the new build completes, a new runnable image is created and pushed to the local registry. Also a new deploy config is created to start a new version of the application

```
[demo@master ~]$ oc get all
NAME                TYPE      FROM      LATEST
bc/nodejs-web-app   Docker    Git       2

NAME                      TYPE      FROM          STATUS     STARTED          DURATION
builds/nodejs-web-app-1   Docker    Git@baba51f   Complete   16 hours ago     4m28s
builds/nodejs-web-app-2   Docker    Git@baba51f   Complete   15 minutes ago   11m11s

NAME                DOCKER REPO                               TAGS      UPDATED
is/node             172.30.222.143:5000/demo/node             latest    16 hours ago
is/nodejs-web-app   172.30.222.143:5000/demo/nodejs-web-app   latest    4 minutes ago

NAME                REVISION   DESIRED   CURRENT   TRIGGERED BY
dc/nodejs-web-app   2          1         1         config,image(nodejs-web-app:latest)

NAME                  DESIRED   CURRENT   READY     AGE
rc/nodejs-web-app-1   0         0         0         16h
rc/nodejs-web-app-2   1         1         1         4m

NAME                 CLUSTER-IP     EXTERNAL-IP   PORT(S)    AGE
svc/nodejs-web-app   172.30.62.84   <none>        8080/TCP   16h

NAME                        READY     STATUS      RESTARTS   AGE
po/nodejs-web-app-1-build   0/1       Completed   0          16h
po/nodejs-web-app-2-4w9j6   1/1       Running     0          4m
po/nodejs-web-app-2-build   0/1       Completed   0          15m
```

## Source build strategy
The source build strategy optimizes, secures and speeds the build of the new application by injecting the application source into a single runnable image. The assemble process can perform a large number of complex operations without creating a new layer at each step, resulting in a fast process. Also, by restricting build operations instead of allowing arbitrary actions, as a Dockerfile would allow, the users can avoid accidental or intentional abuses of the build system, for example, by running an application as a root user.

To force the use of source strategy build
```
[demo@master ~]$ oc new-app --name=nodejs-web-app https://github.com/kalise/nodejs-web-app.git --strategy=source
--> Found tag :4 in image stream "openshift/nodejs" for "nodejs"
    * The source repository appears to match: nodejs
    * A source build using source code from https://github.com/kalise/nodejs-web-app.git will be created
      * The resulting image will be pushed to image stream "nodejs-web-app:latest"
      * Use 'start-build' to trigger a new build
    * This image will be deployed in deployment config "nodejs-web-app"
--> Creating resources ...
    imagestream "nodejs-web-app" created
    buildconfig "nodejs-web-app" created
    deploymentconfig "nodejs-web-app" created
--> Success
    Build scheduled, use 'oc logs -f bc/nodejs-web-app' to track its progress.
    Run 'oc status' to view your app.
```


## Webhooks
A webhook allow OpenShift users to trigger a new build by sending a request to the OpenShift API endpoint. These triggers can be defined using [GitHub webhooks](https://developer.github.com/webhooks/) or generic webhooks. Specify a secret as part of the URL to supply when configuring the webhook. The secret ensures that only the user owning the secret can trigger the build. The secret can be found in the triggers section of the Build Config definition
```yaml
    triggers:
    - github:
        secret: 9xZElS62NAms4qPoMbth
      type: GitHub
    - generic:
        secret: 432VVFvIgqhUh6bzJRsu
      type: Generic
    - type: ConfigChange
    - imageChange:
        lastTriggeredImageID: ...
      type: ImageChange
```

Trigger a new build by issuing a generic webhook
```bash
curl -i -H "Accept: application/json" \
-H "X-HTTP-Method-Override: PUT" -X POST -k \
https://master.openshift.com:8443/oapi/v1/namespaces/demo/buildconfigs/nodejs-web-app/webhooks/432VVFvIgqhUh6bzJRsu/generic

[demo@master ~]$ oc get builds
NAME               TYPE      FROM          STATUS     STARTED          DURATION
nodejs-web-app-1   Docker    Git@ceef795   Complete   29 minutes ago   4m32s
nodejs-web-app-2   Docker    Git@ceef795   Complete   20 minutes ago   4m11s
nodejs-web-app-3   Docker    Git           Running    5 seconds ago    5s
```


