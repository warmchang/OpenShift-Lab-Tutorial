# The Routing Layer
The OpenShift routing layer is how client traffic enters the OpenShift environment so that it can ultimately reach pods. In our Hello World example, the service abstraction defines a logical set of pods enabling clients to refer the service by a consistent address and port. Howewer, our service is not reachable from external clients.

To get pods reachable from external clients we need for a Routing Layer. In a simplification of the process, the OpenShift Routing Layer consists in an instance of a pre-configured HAProxy running in a dedicated pod as well as the related services

Strarting from latest OpenShift release, the installation process install a preconfigured router pod running on the master node. To see details, login as system admin
```
[root@master ~]# oc login -u system:admin
Logged into "https://master.openshift.com:8443" as "system:admin" using existing credentials.

[root@master ~]# oc project default
Now using project "default" on server "https://master.openshift.com:8443".

[root@master ~]# oc get pods
NAME                      READY     STATUS    RESTARTS   AGE
router-1-8pthc            1/1       Running   1          10d
...

[root@master ~]# oc get services
NAME              CLUSTER-IP       EXTERNAL-IP   PORT(S)                   AGE
router            172.30.201.143   <none>        80/TCP,443/TCP,1936/TCP   110d
...
```

Describe the router pod, and see that it is running on the master node
```
[root@master ~]# oc describe pod router-1-8pthc
Name:                   router-1-8pthc
Namespace:              default
Security Policy:        hostnetwork
Node:                   master.openshift.com/10.10.10.19
Start Time:             Thu, 19 Jan 2017 16:22:13 +0100
Labels:                 deployment=router-1
                        deploymentconfig=router
                        router=router
Status:                 Running
IP:                     10.10.10.19
Controllers:            ReplicationController/router-1
Containers:
  router:
    Container ID:       docker://ae58e353155ef37042baa
    Image:              openshift3/ose-haproxy-router:v3.3.0.34
    Image ID:           docker://sha256:bd71278b612ca8
    Ports:              80/TCP, 443/TCP, 1936/TCP
    Requests:
      cpu:              100m
      memory:           256Mi
    State:              Running
...
```

## Expose the service
Please, note that the router is bound to ports 80 and 443 on the host interface. When the router receives a request for an FQDN that it knows about, it will proxy the request to a specific service and then to the running pod providing the service.

To get our router aware of the Hello World service, we need to create a route as ``route-hello-world.yaml`` file that instructs the router where to forward the requests. 
```yaml
---
kind: Route
apiVersion: v1
metadata:
  name: hello-route
  labels:
    name: hello
spec:
  host: hello-world.cloud.openshift.com
  to:
    name: hello-world-service
  tls:
    termination: edge
```
 
As demo user, login to the master and create the route
```
[demo@master ~]$ oc login -u demo -p demo123
Login successful.
You have one project on this server: "demo"
Using project "demo".

[demo@master ~]$ oc create -f route-hello-world.yaml
route "hello-route" created

[demo@master ~]$ oc get route
NAME          HOST/PORT                         PATH      SERVICES              PORT      TERMINATION
hello-route   hello-world.cloud.openshift.com             hello-world-service   <all>     edge
```

Now our Hello World service is reachable from any client with its FQDN
```
[root@master]# curl https://hello-world.cloud.openshift.com -k
Hello OpenShift!
```

In the setup, we required a wildcard DNS entry to point at the master node ``*.cloud.openshift.com. 300 IN  A 10.10.10.19`` Our wildcard DNS entry points to the public IP address of the master. Since there is only the master in the infra region, we know we can point the wildcard DNS entry at the master and we'll be all set. Once the FQDN request reaches the router pod running on the master node, it will be forwarded to the pods on the compute nodes actually running the Hello World application.

The fowarding process is based on HAProxy configurations set by the route we defined before. To see the HAProxy configuration, login as root to the master node and inspect the router pod configuration
```
[root@master ~]# oc get pods
NAME                      READY     STATUS    RESTARTS   AGE
router-1-8pthc            1/1       Running   1          11d
...
[root@master ~]# oc rsh router-1-8pthc
sh-4.2$ pwd
/var/lib/haproxy/conf
sh-4.2$ ls -l haproxy.config
-rwxrwxrwx. 1 root root 10178 Jan 30 10:20 haproxy.config
sh-4.2$ cat haproxy.config
...
##-------------- app level backends ----------------
...
#server openshift_backend
  server 9945852405ae9517c 10.1.0.18:8080 check inter 5000ms cookie 9945852405ae9517c weight 100
  server 6283681187833cb3e 10.1.2.21:8080 check inter 5000ms cookie 6283681187833cb3e weight 100
```

We see the HAProxy configured to forwared requests to the pod running the applications on the compute nodes. This is reassumed in the following picture:

![](../images/haproxy.png?raw=true)
