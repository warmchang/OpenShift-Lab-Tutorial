# Projects administration
In OpenShift, projects are used to isolate resources from groups of developers. The platform user admin can give users access to certain projects, allow them to create their own project, and give them admin rights too. The user admin can set resource quotas and policies on a specific project.

As platfrom admin, login to the system and get the list of projects
```
[root@master ~]# oc login -u system:admin

[root@master ~]# oc get projects
NAME               DISPLAY NAME          STATUS
demo               OpenShift Demo        Active
kube-system                              Active
logging                                  Active
management-infra                         Active
openshift                                Active
openshift-infra                          Active
default                                  Active
[root@master ~]#
```

There are many projects in OpenShift, some are user projects like ``demo`` as we created before, the ``default`` project and other infrastructure projects. In this section, we'll focus on user projects.

## Project permissions
Create a new project and set the user ``sam`` as project administrator
```
[root@master ~]# oadm new-project tomcat \
    --display-name="My New Cool Project" \
    --description="This is the coolest project in the town" \
    --admin=sam
Created project tomcat
```

Login as sam user and create a new pod in this project
```
[root@master ~]# su - sam
[sam@master ~]$ oc login -u sam -p demo123
Server [https://localhost:8443]:
Login successful.
You have one project on this server: "tomcat"
Using project "tomcat".
Welcome! See 'oc help' to get started.
[sam@master ~]$

[sam@master ~]$ oc create -f pod-hello-world.yaml
pod "hello-pod" created

[sam@master ~]$ oc get pod
NAME        READY     STATUS    RESTARTS   AGE
hello-pod   1/1       Running   0          23s
```

As example of an administrative function, we want to let the user ``demo`` look at the ``tomcat`` project we just created. As system admin, set the tomcat project as current one and give to demo user the permission to view the tomcat project
```
[root@master ~]# oc project tomcat
Now using project "tomcat" on server "https://master.openshift.com:8443".

[root@master ~]# oadm policy add-role-to-user view demo
```

Login as demo user, check the list of projects and set tomcat project as current project
```
[demo@master ~]$ oc login -u demo -p demo123

[demo@master ~]$ oc get project
NAME      DISPLAY NAME          STATUS
demo      OpenShift Demo        Active
tomcat    My New Cool Project   Active

[demo@master ~]$ oc project tomcat
Now using project "tomcat" on server "https://localhost:8443".

[demo@master ~]$ oc get pod
NAME        READY     STATUS    RESTARTS   AGE
hello-pod   1/1       Running   0          1m
```

However, demo user cannot make changes
```
[demo@master ~]$ oc delete pod hello-pod
Error from server: User "demo" cannot delete pods in project "tomcat"
```

Howewer, the project admin (or the system admin) can give demo user the edit rights on tomcat project
```
[root@master ~]# oc project tomcat
Now using project "tomcat" on server "https://master.openshift.com:8443".

[root@master ~]# oadm policy add-role-to-user edit demo
```

Finally, the demo user can make canges in tomcat project
```
[demo@master ~]$ oc delete pod hello-pod
pod "hello-pod" deleted
[demo@master ~]$ oc create -f pod-hello-world.yaml
pod "hello-pod" created
```

Also, the project admin (or the system admin) can give demo user the admin rights on tomcat project
```
[root@master ~]# oadm policy add-role-to-user admin demo
```

## Project quotas and limits
In OpenShift platform, resources can have quotas and limits enforced against a given project. In other words, within a project, users cannot "do stuff" that will cause quotas and limits to be exceeded. Since quotas and limits are enforced at the project level, it is up to the users to allocate resources, e.g. memory and CPU, according to those limits and quotas.

A quota set constraints on the total amount of resources that users cannot exceede within a project. Here's like a quota can be defined in the ``quota.yaml`` file
```yaml
---
apiVersion: v1
kind: ResourceQuota
metadata:
  name: demo-quota
spec:
  hard:
    memory: 512Mi
    cpu: 100m
    pods: '3'
```
The above file defines quota for several resources:

* **Memory**: the memory measure is in bytes, but various other suffixes are supported, eg: Mi (mebibytes), Gi (gibibytes), etc. Across all pods in a non-terminal state, the sum of memory requests cannot exceed this value.
* **CPU**: the unit of measure is a "normalized" unit that should be roughly equivalent to a single hyperthreaded CPU core. Fractional assignment is also allowed. Across all pods in a non-terminal state, the sum of CPU requests cannot exceed this value.
* **Pods**: the total number of pods in a non-terminal state that can exist in the project.

This quota is related to the total amount of memory, cpu and number of pods within a demo projects.

OpenShift also let users to set resources limit ranges that a single pod or container can consume. Here's like a limit ranges for demo project can be defined in the ``project-demo-limits.yaml`` file
```yaml
---
kind: LimitRange
apiVersion: v1
metadata:
  name: limits
  creationTimestamp:
spec:
  limits:
  - type: Pod
    max:
      cpu: 50m
      memory: 256Mi
    min:
      cpu:
      memory:
```
As system admin user set the limits to the demo project
```
[root@master ~]# oc login -u system:admin

[root@master ~]# oc project demo

[root@master ~]# oc create -f /home/demo/project-demo-limits.yaml
limitrange "limits" created

[root@master ~]# oc get limitrange limits
NAME      AGE
limits    17s

[root@master ~]# oc describe limitrange limits
Name:           limits
Namespace:      demo
Type            Resource        Min     Max     Default Request Default Limit   Max Limit/Request Ratio
----            --------        ---     ---     --------------- -------------   -----------------------
Pod             cpu             0       50m     -               -               -
Pod             memory          0       256Mi   -               -               -

[root@master ~]# oc describe project demo
Name:           demo
Namespace:      <none>
Created:        4 days ago
Labels:         <none>
Annotations:    openshift.io/description=This is the first demo project with OpenShift
Display Name:   OpenShift Demo
Description:    This is the first demo project with OpenShift
Status:         Active
Node Selector:  <none>
Quota:          <none>
Resource limits:
        Name:   limits
        Type    Resource        Min     Max     Default
        ----    --------        ---     ---     ---
        Pod     cpu             0       50m     -
        Pod     memory          0       256Mi   -
```

In this example, we forced a max limit ranges of cpu and memory for pods in our demo project. If limits are not set in pod definition, all pods will inherit the project limit ranges. Howewer, we can still define limit ranges for any single pod in our project. For example, in the ``rc-hello-world-limited.yaml`` file, we specify limit ranges in our Hello World pod definition
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
        resources:
          limits:
          cpu: 25m
          memory: 128Mi
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

We can create multiple Hello Worl pods because limits of the single pod are within the global limit ranges for this project
```
[demo@master ~]$ oc create -f rc-hello-world-limited.yaml
replicationcontroller "rc-hello" created

[demo@master ~]$ oc scale rc rc-hello --replicas=4
replicationcontroller "rc-hello" scaled

[demo@master ~]$ oc get pods
NAME             READY     STATUS    RESTARTS   AGE
rc-hello-2olfp   1/1       Running   0          6m
rc-hello-f0rbm   1/1       Running   0          7m
rc-hello-lv2rf   1/1       Running   0          6m
rc-hello-matjg   1/1       Running   0          6m

[demo@master ~]$ oc scale rc rc-hello --replicas=0
replicationcontroller "rc-hello" scaled

[demo@master ~]$ oc get pods
No resources found.
```

To limit the resources usage within the demo project, we need to apply the quota
```
[root@master ~]# oc project demo

[root@master ~]# oc create -f /home/demo/quota.yaml
resourcequota "demo-quota" created

[root@master ~]# oc describe project demo
Name:           demo
Namespace:      <none>
Created:        4 days ago
Labels:         <none>
Annotations:    openshift.io/description=This is the first demo project with OpenShift
Display Name:   OpenShift Demo
Description:    This is the first demo project with OpenShift
Status:         Active
Node Selector:  <none>
Quota:
        Name:           demo-quota
        Resource        Used    Hard
        --------        ----    ----
        cpu             0       100m
        memory          0       512Mi
        pods            0       3
Resource limits:
        Name:   limits
        Type    Resource        Min     Max     Default
        ----    --------        ---     ---     ---
        Pod     cpu             0       50m     -
        Pod     memory          0       256Mi   -
```

Now try to scale up to 3 pods
```
[demo@master ~]$ oc scale rc rc-hello --replicas=3
replicationcontroller "rc-hello" scaled

[demo@master ~]$ oc get pods
NAME             READY     STATUS    RESTARTS   AGE
rc-hello-30mk8   1/1       Running   0          1m
rc-hello-ao7y9   1/1       Running   0          1m
rc-hello-icq50   1/1       Running   0          1m
```

Each pod consumes 1/4 of available CPU and memory resources specified by our quota
```
[root@master ~]# oc describe project demo
Name:           demo
Namespace:      <none>
Created:        4 days ago
Labels:         <none>
Annotations:    openshift.io/description=This is the first demo project with OpenShift
Display Name:   OpenShift Demo
Description:    This is the first demo project with OpenShift
Status:         Active
Node Selector:  <none>
Quota:
        Name:           demo-quota
        Resource        Used    Hard
        --------        ----    ----
        cpu             75m     100m
        memory          384Mi   512Mi
        pods            3       3
Resource limits:
        Name:   limits
        Type    Resource        Min     Max     Default
        ----    --------        ---     ---     ---
        Pod     cpu             0       50m     -
        Pod     memory          0       256Mi   -
```

Howewer, the quota specify up to 3 pods, so project's constraints has been reached and we cannot add more pods
```
[demo@master ~]$ oc scale rc rc-hello --replicas=4

[demo@master ~]$ oc describe rc rc-hello | grep -i error
1 Warning     FailedCreate             Error
creating: pods "rc-hello-" is forbidden:
exceeded quota: demo-quota, requested: pods=1, used: pods=3, limited: pods=3

[demo@master ~]$ oc get pods
NAME             READY     STATUS    RESTARTS   AGE
rc-hello-30mk8   1/1       Running   0          4m
rc-hello-ao7y9   1/1       Running   0          5m
rc-hello-icq50   1/1       Running   0          4m
```

Just to recap, quota limits the total amount of resources within a project, while limit ranges define the resource usage limits for single pods within the project.
