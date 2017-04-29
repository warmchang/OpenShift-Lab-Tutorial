# Getting Strated with core concepts
At this point we essentially have a sufficiently functional OpenShift environment. It is now time to create the **Hello World** application using some sample code. It is simple http server written in nodejs returning a greating message as contained into the MESSAGE env variable. The application is available as Docker image [here](https://docker.io/kalise/nodejs-web-app). Source code is [here](https://github.com/kalise/nodejs-web-app).

  * [Create a demo user](#create-a-demo-user)
  * [Create a demo project](#create-a-demo-project)
  * [Create a pod](#create-a-pod)
  * [Create a service](#create-a-service)
  * [Create a replica controller](#create-a-replica-controller)

## Create a demo user
OpenShift platform supports a number of mechanisms for authentication. The simplest use case for testing purposes is htpasswd-based authentication. To start, we will need the ``htpasswd`` binary on the Master node

```
[root@master ~]# yum -y install httpd-tools
```

The OpenShift configuration is stored in a YAML file at ``/etc/origin/master/master-config.yaml``. During the installation procedure, Ansible was configured to enable the ``htpasswd`` based authentication, so that it should look like the following:

```yaml
...
identityProviders:
- challenge: true
  login: true
  name: htpasswd_auth
  provider:
    apiVersion: v1
    file: /etc/htpasswd
    kind: HTPasswdPasswordIdentityProvider
...
```
More information on these configuration settings can be found on the product documentation.

Create a standard user:
```
[root@master ~]# useradd demo
[root@master ~]# passwd demo
[root@master ~]# touch /etc/htpasswd
[root@master ~]# htpasswd -b /etc/htpasswd demo *********
```

Login to the OpenShift platform as demo user by the ``oc`` CLI command
```
[root@master ~]# oc login -u demo -p ********
Login successful.
You don't have any projects. You can try to create a new project, by running
oc new-project <projectname>
```

## Create a demo project
The OpenShift platform has the concept of "projects" to contain a number of different resources. We'll explore what this means in more details throughout the rest of the tutorial. Create a demo project for our first application.

The default configuration for CLI operations currently is to be the ``system:admin`` passwordless user, which is allowed to create projects. Login as admin user:
```
[root@master ~]# oc login -u system:admin
Logged into "https://master.openshift.com:8443" as "system:admin" using existing credentials.
You have access to the following projects and can switch between them with 'oc project <projectname>':

  * default
    kube-system
    logging
    management-infra
    openshift
    openshift-infra

Using project "default".
```

We can use the admin OpenShift ``oadm`` command to create a project, and assign an administrative user to it.

As the root system user on master:
```
oadm new-project demo \
--display-name="OpenShift Demo" \
--description="This is the first demo project with OpenShift" \
--admin=demo
```

This command creates a project:

 * with the id demo
 * with a display name
 * with a description
 * with an administrative user demo

```
[root@master ~]# oc get projects
NAME               DISPLAY NAME   STATUS
openshift                         Active
openshift-infra                   Active
default                           Active
demo                              Active
kube-system                       Active
logging                           Active
management-infra                  Active

[root@master ~]# oc get project demo
NAME      DISPLAY NAME     STATUS
demo      OpenShift Demo   Active

[root@master ~]# oc describe project demo
Name:                   demo
Namespace:              <none>
Created:                8 seconds ago
Labels:                 <none>
Annotations:            openshift.io/description=This is the first demo project with OpenShift
                        openshift.io/display-name=OpenShift Demo
Display Name:           OpenShift Demo
Description:            This is the first demo project with OpenShift
Status:                 Active
Node Selector:          <none>
Quota:                  <none>
Resource limits:        <none>
```

Now that we have a new project, login as demo user
```
[root@master ~]# su - demo 
[demo@master ~]$ oc login -u demo -p *********
Server [https://localhost:8443]:
...
Use insecure connections? (y/n): y
Login successful.
You have one project on this server: "demo"
Using project "demo".
```

The login process created a file called named ``~/.kube/config`` in the user home folder. This configuration file has an authorization token, some information about where our project lives:
```yaml
apiVersion: v1
clusters:
- cluster:
    insecure-skip-tls-verify: true
    server: https://localhost:8443
  name: localhost:8443
contexts:
- context:
    cluster: localhost:8443
    namespace: demo
    user: demo/localhost:8443
  name: demo/localhost:8443/demo
current-context: demo/localhost:8443/demo
kind: Config
preferences: {}
users:
- name: demo/localhost:8443
  user:
    token: *********
```

## Create a pod
An application in OpenShift live inside an entity called **"pod"**. Here the file ``pod-hello-world.yaml`` containing the definition of our pod in yaml format:
```yaml
---
kind: Pod
apiVersion: v1
metadata:
  name: hello-pod
  creationTimestamp:
  labels:
    name: hello
spec:
  containers:
  - env:
    - name: MESSAGE
      value: "Hello OpenShift"
    name: hello
    image: docker.io/kalise/nodejs-web-app:latest
    ports:
    - containerPort: 8080
      protocol: TCP
    terminationMessagePath: "/dev/termination-log"
    imagePullPolicy: IfNotPresent
    securityContext:
      privileged: false
  restartPolicy: Always
  dnsPolicy: ClusterFirst
  serviceAccount: ''
status: {}
```

As demo user, create the pod from the yaml file
```
[demo@master ~]$ oc create -f pod-hello-world.yaml
pod "hello-pod" created
```

Check the status of the pod
```
[demo@master ~]$ oc get pods
NAME      READY     STATUS    RESTARTS   AGE
hello-pod 1/1       Running   0          1m

[demo@master ~]$ oc describe pod hello-pod
Name:                   hello-pod
Namespace:              demo
Security Policy:        restricted
Node:                   nodeb.openshift.com/10.10.10.17
Start Time:             Sat, 28 Jan 2017 12:36:29 +0100
Labels:                 name=hello
Status:                 Running
IP:                     10.1.0.2
Controllers:            <none>
Containers:
  hello:
    Container ID:       docker://8d4dc403d6597c2d2ccafeb45f684e37e789fcd32b43f35704a70e59cfdb2d24
    Image:              openshift/hello-openshift:latest
    Image ID:           docker-pullable://docker.io/openshift/hello-openshift
    Port:               8080/TCP
    State:              Running
      Started:          Sat, 28 Jan 2017 12:38:08 +0100
    Ready:              True
    Restart Count:      0
    Volume Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from default-token-56m1i (ro)
    Environment Variables:      <none>

Conditions:
...
Volumes:
...
QoS Class:      BestEffort
Tolerations:    <none>
Events:
...
```

To verify that our application is really working, issue a curl to the pod's address and port:
```
[demo@master ~]$ curl 10.1.0.2:8080
Hello OpenShift!
```

Login to the node where our pod is running, i.e. ``nodeb.openshift.com/10.10.10.17`` and check the containers running on that host

```
[root@nodeb ~]# docker ps
CONTAINER ID   IMAGE                                    COMMAND       CREATED          STATUS      PORTS  NAMES
8d4dc403d659   docker.io/kalise/nodejs-web-app:latest   "npm start"   12 minutes ago   Up 12 min   ...    ...
f867f09e8639   openshift3/ose-pod:v3.4.0.39             "/pod"        12 minutes ago   Up 12 min   ...    ...
```
Our application is running inside the first container from the ``docker.io/kalise/nodejs-web-app:latest`` image. The second container from the ``openshift3/ose-pod`` container exists because of the way network namespacing works in OpenShift.

Finally, delete the pod
```
[demo@master ~]$ oc delete pod hello-pod
pod "hello-pod" deleted
```

## Create a service
Our simple Hello World application is a backed by a container inside a pod running on a single compute node. The OpenShift platform introduces the concept of **"service"**. A service in OpenShift is an abstraction which defines a logical set of pods. Pods can be added to or removed from a service arbitrarily while the service remains consistently available, enabling any client to refer the service by a consistent address:port couple. 

Define a service for our simple Hello World application in a ``service-hello-world.yaml`` file.
```yaml
---
kind: Service
apiVersion: v1
metadata:
  name: hello-world-service
  labels:
    name: hello
spec:
  selector:
    name: hello
  ports:
  - protocol: TCP
    port: 9000
    targetPort: 8080
```
The above service is associated to our previous Hello World pod. Pay attention to the service selector field. It tells OpenShift that all pods with the label ``hello`` are associated to this service, and should have traffic distributed amongst them. In other words, the service provides an abstraction layer, and is the input point to reach all of the pods. 

As demo user, create the service
```
[demo@master ~]$ oc create -f service-hello-world.yaml
service "hello-world-service" created
pod "hello-pod" created
```

Check the status of the service
```
[demo@master ~]$ oc get service
NAME                  CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
hello-world-service   172.30.42.123   <none>        9000/TCP   5m

[demo@master ~]$ oc describe service hello-world-service
Name:                   hello-world-service
Namespace:              demo
Labels:                 name=hello
Selector:               name=hello
Type:                   ClusterIP
IP:                     172.30.42.123
Port:                   <unset> 9000/TCP
Endpoints:              <none>
Session Affinity:       None
No events.
```

Pods can be added to the service arbitrarily. Make sure that the selector label ``hello`` is in the definition yaml file of any pod we would to bind to the service.
```
[demo@master ~]$ oc create -f pod-hello-world.yaml
pod "hello-pod" created

[demo@master ~]$ oc create -f pod1-hello-world.yaml
pod "hello-pod1" created

[demo@master ~]$ oc describe service hello-world-service
Name:                   hello-world-service
Namespace:              demo
Labels:                 name=hello
Selector:               name=hello
Type:                   ClusterIP
IP:                     172.30.42.123
Port:                   <unset> 9000/TCP
Endpoints:              10.1.0.11:8080,10.1.2.11:8080
Session Affinity:       None
No events.
```

The service will act as an internal load balancer in order to proxy the connections it receives from the clients toward the pods bound to the service. We can check if the service is reaching our application
```
[demo@master ~]$ curl 172.30.42.123:9000
Hello OpenShift!
```

The service also provide a name resolution for the associated pods. For example, in the case above, the hello pods can be reached by other pods in the same namespace by the name ``hello-world-service`` instead of the address:port ``172.30.42.123:9000``. This is very useful when we need to link different applications.

## Create a replica controller
Manually created pods as we made above are not replaced if they get failed, deleted or terminated for some reason. To make things more robust, OpenShift introduces the **Replica Controller** abstraction. A Replica Controller ensures that a specified number of pod *"replicas"* are running at any time. In other words, a Replica Controller makes sure that a pod or set of pods are always up and available. If there are too many pods, it will kill some; if there are too few, it will start more.

A Replica Controller configuration consists of:

 * The number of replicas desired
 * The pod definition
 * The selector to bind the managed pod

A selector is a label assigned to the pods that are managed by the replica controller. Labels are included in the pod definition that the replica controller instantiates. The replica controller uses the selector to determine how many instances of the pod are already running in order to adjust as needed.

In the ``rc-hello-world.yaml`` file, define a replica controller with replica 1.
```yaml
---
apiVersion: v1
kind: ReplicationController
metadata:
  name: rc-hello
spec:
  replicas: 1
  selector:
    name: hello
  template:
    metadata:
      creationTimestamp:
      labels:
        name: hello
    spec:
      containers:
      - env:
        - name: MESSAGE
          value: "Hello OpenShift"
        name: hello
        image: docker.io/kalise/nodejs-web-app:latest
        ports:
        - name: http
          containerPort: 8080
          protocol: TCP
        terminationMessagePath: "/dev/termination-log"
        imagePullPolicy: IfNotPresent
        securityContext:
          privileged: false
        livenessProbe:
          tcpSocket:
            port: 8080
          timeoutSeconds: 1
          initialDelaySeconds: 10
      restartPolicy: Always
      dnsPolicy: ClusterFirst
      serviceAccount: ''
      nodeSelector:
        region: primary
```

Create a Replica Controller
```
[demo@master ~]$ oc create -f rc-hello-world.yaml
replicationcontroller "rc-hello" created

[demo@master ~]$ oc get rc
NAME       DESIRED   CURRENT   READY     AGE
rc-hello   1         1         1         1m

[demo@master ~]$ oc describe rc rc-hello
Name:           rc-hello
Namespace:      demo
Image(s):       docker.io/kalise/nodejs-web-app:latest
Selector:       name=hello
Labels:         name=hello
Replicas:       1 current / 1 desired
Pods Status:    1 Running / 0 Waiting / 0 Succeeded / 0 Failed
No volumes.
Events:
```

We can see the pod just created
```
[demo@master ~]$ oc get pods
NAME             READY     STATUS    RESTARTS   AGE
rc-hello-ijc6g   1/1       Running   0          3m
```

When it comes to scale, there is a command called ``oc scale`` to get job done
```
[demo@master ~]$ oc scale rc rc-hello --replicas=2
replicationcontroller "rc-hello" scaled

[demo@master ~]$ oc get pods
NAME             READY     STATUS    RESTARTS   AGE
rc-hello-ijc6g   1/1       Running   0          6m
rc-hello-jnset   1/1       Running   0          9s
```

To scale down, just set the replicas
```
[demo@master ~]$ oc scale rc rc-hello --replicas=1
replicationcontroller "rc-hello" scaled

[demo@master ~]$ oc get pods
NAME             READY     STATUS    RESTARTS   AGE
rc-hello-ijc6g   1/1       Running   0          9m

[demo@master ~]$ oc scale rc rc-hello --replicas=0
replicationcontroller "rc-hello" scaled

[demo@master ~]$ oc get pods
No resources found.
```

Please note that the Replica Controller does not autoscale. This job is done by a metering service by piloting the Replica Controller, based on memory and cpu load or other criteria.
