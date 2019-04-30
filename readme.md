## Intro

This workshop walks through everything required to build a simple Kubernetes lab, deploy a web app, scale the app, recover from a failure, and perform a rolling update.

The workshop comprises the following sections:

1. Pre-requisites
2. Background (theory)
3. Kubernetes Architecture (theory)
5. Deploy an app as a Pod (hands-on)
6. Use a Deployment for self-healing (hands-on)
7. Scale the application (hands-on)
8. Use a cloud load-balancer (hands-on)
9. Perform a rolling update (hands-on)

## 1. Pre-requisites

Most of the labs in this workshop can be completed using [Play With Kubernetes (PWK)](https://labs.play-with-k8s.com) and Docker Desktop. The exception is _Lab 8: Connect to a cloud load-balancer_. If you want to follow along with _Lab 8_, you will need a lab on a public cloud platform that supports integration with Kubernetes _Load-balancer Services_. AWS, Azure, DO, and GCP all support this feature.

**The DockerCon Workshop will assume you are using Play with Kubernetes**. This will help the workshop run smoothly and reduce the amount of help and troubleshooting required from the workshop leader and assistants.

### Following along on Play With Kubernetes (PWK)

In order to follow along on Play With Kubernetes, you will need:

1. A Docker Hub or GitHub account
2. A PWK K8s cluster with one master and at least three nodes

Point your browser to [PWK](https://labs.play-with-k8s.com) and follow the instructions to build a lab. Make sure you build a lab that has at least three worker nodes.

If you're attending the workshop at DockerCon, **you should start building your PWK cluster now!**

## 2. Background

Containers have revolutionized the way we build, ship and run applications. However, deploying and managing cloud-native microservices applications at scale is hard. This is where Kubernetes comes into play.

A _cloud-native microservices application_ is a way of building business applications that enable things like self-healing, scaling, zero-downtime rolling updates, and more. Generally speaking, these applications will be built from lost of small specialized services that talk to each other and form into a useful app.

However, building applications like brings its own set of challenges --- reacting to events such as failed components, spikes in traffic, and bug fixes can be very hard to manage with traditional tools. For example, if an application experiences huge spikes in traffic, it's not a good idea to require a human to intervene and provision more resources. It's far better if the application and infrastructure can react on-demand.

This is where Kubernetes comes in to play...

It's a popular pattern to use Docker to develop and build your applications in containers, but then to use Kubernetes to deploy and manage those apps.

Kubernetes provides the substrate and primitives that allow:

- Self-healing (recovering from failed components)
- Scaling up and down on-demand
- Rolling updates and versioned rollbacks
- Blue-greens and canaries
- More...

Kubernetes is also in the business of abstracting low-level infrastructure. As such, deploying applications on Kubernetes hides the underlying infrastructure, meaning you can deploy and migrate applications between cloud platforms and even on-premises dataceters as long as you have Kubernetes --- as long as Kubernetes is there, it doesn't matter what infrastructure is operating beneath it.

## 3. Kubernetes Architecture

At a high-level, Kubernetes is at least two things:

1. A cluster
2. An orchestrator

### Kubernetes cluster

A Kubernetes cluster is a set of machines that run your applications. A Kubernetes cluster comprises a _control plane_ and a _data plane._ It also has all of the normal clustering requirements such as performance and high-availability (HA).

#### Kubernetes control plane

The Kubernetes control plane is where all of the logic and cluster-smarts exist. It includes; the API server, the scheduler, controllers, the cluster store, and more... You should not run user applications on nodes hosting the control plane.

The _API server_ is like the Grand Central Station of the cluster --- all internal and external communication goes through the API server. It exposes a RESTful interface, and the most common way for users to issue requests to it is using the `kubectl` command-line utility. All requests to the API server (internal and external requests) are subject to authentication and authorization checks.

The _cluster store_ is the only stateful component of the control plane, and is where the cluster configuration is stored. It is based on the popular etcd distributed database.

The _scheduler_ is responsible for scheduling work to the cluster.

The various _controllers_ constantly monitor the cluster and make sure that everything is running as it should.

There are other components to the control plane, but the ones discussed are probably the most important to understand.

Hosted Kubernetes services (AWS EKS, Azure AKS, Google GKE etc.) manage the control plane for you. This means they manage things like control plane performance, control plane HA, and control plane upgrades. In fact, most hosted Kubernetes services do not even let you log onto nodes hosting the control plane -- it's a managed service. This is a popular model, but offers very little in the way of customizing your cluster. If you need a more customized cluster than the hosted services offer, you should build your own cluster using `kubeadm`.

#### Kubernetes data plane

The data plane is the nodes that run user applications. Sometimes we call these _workers_ or _worker nodes_.

Nodes have three important Kubernetes components:

1. Kubelet
2. Container runtime
3. Kube-proxy

The **kubelet** is the main Kubernetes agent and runs on all nodes in the cluster. It is required in order for a node to be a member of a cluster, and its main job is to watch the API server for new work assignments. When it sees a new work assignment, it executes the task and reports events back to the API server.

The **container runtime** is the component that performs low-level container-related tasks such as; pulling images, starting containers, and stopping containers. Historically, Docker has been the most common container runtime used by Kubernetes, but recently containerd (pronounced "container-dee") has increased in popularity. Other container runtimes exist -- some providing different levels of workload isolation.

**Kube-proxy** is responsible for low-level networking tasks on nodes.

### Kubernetes as an orchestrator

Once you have a Kubernetes cluster, you deploy applications to it, and this is where Kubernetes delivers its value.

From a high-level, you do the following:

1. Write your application components in your favourite languages
2. Build them into container images
3. Push them to a registry
4. Glue them together with Kubernetes YAML files (other options exist)
5. Deploy them to your Kubernetes cluster
6. Let Kubernetes keep them running and reacting to events

Some key concepts to understand include; _declarative models, desired state, observed state._

Kubernetes likes applications to be deployed in a declarative manner. This is where you define what your application should look like in a YAML file that can be version controlled. Your declarative YAML file will state things like; which images to use, which network ports to use, and how many replicas to deploy --- we call this the _desired state_. You POST the file to the Kubernetes API server, and Kubernetes takes care of implementing your app on the cluster. In the background Kubernetes implements control loops that constantly monitor the cluster to make sure that the observed state of the cluster matches your desired state.

Quick example. Assume you deploy an app that has a web front-end, and you declare as part of the application's desired state that you want 3 replicas of the web service running. If one of the cluster nodes hosting the web service fails, you might drop from 3 running replicas to 2. In the background, a control loop will observe this, realize that observed state does not match desired state, and spin up another replica to take the count back up to three.

This is only possible because your declare your applications desired state in a declarative YAML file that is recorded in the cluster store as a record of intent. Kubernetes can then watch the cluster and make sure things are always the way your requested they should be.

This logic also allows things like dynamic scaling operations to work. In the case of scaling, you can POST an update to your application's desired state, increasing the number of web service replicas from 3 to 8. This will update the desired state to 8, a background watch loop will notice and follow the same process to increase the number of replicas from 3 to 8.

That's enough theory for now.

## 4. Deploy an app as a Pod

To continue with the workshop you will need a Kubernetes cluster and `kubectl` configured to use it. The DockerCon workshop assumes you have a Kubernetes cluster in Play with Kubernetes that has one master and at least three worker nodes.

In this section, we'll deploy a simple web service in a Kubernetes Pod.

Pods are the atomic unit of scheduling on a Kubernetes cluster --- VMware deploys applications as one or more virtual machines, Docker deploys applications as one or more containers, and Kubernetes deploys applications as one or more Pods.

It's important to understand that a Pod is just an execution environment for one or more containers. So... you still package your application components as containers, but when you deploy them on Kubernetes, you wrap them inside of Pods.

The following snippet is a simple Pod YAML file that deploys a Pod based on the `nigelpoulton/k8sbook:latest` image and exposes it on port `8080`.

Copy the YAML into a new file called `pod.yml` on your PWK cluster. To do this: Copy the YAML text > `vi pod.yml` > `Ins` > Paste YAML > `Esc` > `:wq` > `Enter`.

```
apiVersion: v1
kind: Pod
metadata:
  name: hello-pod
  labels:
    app: web
    zone: prod
    version: v1
spec:
  containers:
  - name: hello-ctr
    image: nigelpoulton/k8sbook:latest
    ports:
    - containerPort: 8080
```

Stepping through the file...

`apiVersion` and `kind` are required to tell the control plane what type of object to deploy and what schema version to base it on.

The `metadata` section let's us attach names and labels to the object (Pod) that will help us identify it. These values are arbitrary and we'll see some examples later in the lab.

The `spec` section defines the container that will run in the Pod. It's calling the Pod "hello-ctr", basing it on an image and exposing it on a port.

### Deploying the Pod

Deploy it to the cluster using the following command. The command assumes the Pod YAML file is called `pod.yml` and exists in your system's $PATH.

```
$ kubectl apply -f pod.yml
```

Give the Pod a few seconds to deploy (pull the image and start the container etc.).

Check the Pod with the following command. It may take a minute or so for the Pod to enter the `Running` state.

```
$ kubectl get pods
NAME        READY   STATUS    RESTARTS   AGE
hello-pod   1/1     Running   0          33s
```

Congratulations, the Pod is running.

You can see more information with the `kubectl describe pods hello-pod` command.

```
$ kubectl describe pods hello-pod
Name:               hello-pod
Namespace:          default
Labels:             version=v1
                    zone=prod
Status:             Running
IP:                 10.1.0.102
Containers:
  hello-ctr:
    Container ID:   docker://24b7...
    Image:          nigelpoulton/k8sbook:latest
    Port:           8080/TCP
    Host Port:      0/TCP
    State:          Running
```

The output above is snipped for brevity.

In the next section we'll see how to connect to the web service.

### Connecting to the Pod

Kubernetes has a **Service** object that provides reliable network endpoints for Pods. We'll see some of the advantages later when we scale Pods up and down and look at some failure scenarios. But for now, Services are what we need to access our Pods on the network.

Just like Pods, Services are defined in YAML files and deployed via `kubectl`.

The following YAML defines a Service that will make your Pod accessible from any node in the Kubernetes cluster.

```
apiVersion: v1
kind: Service
metadata:
  name: svc-np
  labels:
    app: web
spec:
  type: NodePort
  ports:
    - port: 8080
      nodePort: 30001
  selector:
    app: web
```

Copy the YAML into a new file called `svc.yml` on your PWK cluster. To do this: Copy the YAML text > `vi svc.yml` > `Ins` > Paste YAML > `Esc` > `:wq` > `Enter`.

Stepping through the file from the top...

`apiVersion` and `kind` tell Kubernetes what type of object we're defining and what schema version to use for the object.

The `metadata` section defines arbitrary key-value pairs that allow you to tag and identify the object. The most important one for now is the name "svc-np".

The `spec` section is defining a **NodePort** Service that maps port `30001` on every cluster node to `8080` in the Pod. This means you can hit any node in the cluster on port `30001` and reach the web server running in the Pod.

The `selector` tells the Service that any traffic it receives on port `30001` should be forwarded to port `8080` on any Pod in the cluster that has the `app: web` label.

It's good to think of Services as having a front-end and back-end configuration. In this example the front-end configuration tells it to listen for traffic on port `30001` on all nodes in the cluster.  The back-end says to forward that traffic to port `8080` on any Pod with the `app=web` label.

Use the following command to deploy the Service to the cluster. It assumes that the YAML file is called `svc.yml` and is in your system's PATH or your current working directory.

```
$ kubectl apply -f svc.yml
```

Check the Service with `kubectl get svc` and `kubectl describe svc` commands.

```
$ kubectl get svc
NAME         TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)          AGE
kubernetes   ClusterIP   10.96.0.1      <none>        443/TCP          3m
svc-np       NodePort    10.99.91.176   <none>        8080:30001/TCP   41s
```

```
$ kubectl describe svc svc-np
Name:                     svc-np
Namespace:                default
Labels:                   app=web
Annotations:              kubectl.kubernetes.io/last-applied-configuration={"apiVersion":"v1","kind":"Service","metadata":{"annotations":{},"labels":{"app":"web"},"name":"svc-np","namespace":"default"},"spec":{"ports":[{"nodeP...
Selector:                 app=web
Type:                     NodePort
IP:                       10.104.153.79
Port:                     <unset>  8080/TCP
TargetPort:               8080/TCP
NodePort:                 <unset>  30001/TCP
Endpoints:                10.42.0.1:8080
Session Affinity:         None
External Traffic Policy:  Cluster
Events:                   <none>
```

The output's might have been trimmed for readability, but you can see the mapping between port `30001` and `8080`.

Use the `curl` command to test that the Service reaches the web server in the Pod.  You can also point a web browser to the URL of any of the Kubernetes cluster nodes on port `30001` to see what the page actually looks like.

```
$ curl localhost:30001
<html><head><title>K8s rocks!</title><link rel="stylesheet" href="http://netdna.bootstrapcdn.com/bootstrap/3.1.1/css/bootstrap.min.css"/></head><body><div class="container"><div class="jumbotron"><h1>Kubernetes Rocks!</h1><p>Check out my K8s Deep Dive course!</p><p> <a class="btn btn-primary" href="https://acloud.guru/learn/kubernetes-deep-dive">The video course</a></p><p></p></div></div></body></html>
```


Optional extra. If you're following along on Play with Kubernetes, or another platform where you have multiple nodes in your Kubernetes cluster, you can terminate the node running the Pod to prove that Kubernetes does not recover the Pod on another node:

1. Run `kubectl get pods -o wide` to find out which node is hosting the Pod
2. Terminate the node
3. Refresh the web browser or re-run the `curl` command -- the web server will be unavailable

It may take a minute or two for the output of `kubectl get pods` to notice that the Pod is no longer running.

Let's clean-up the lab before we move one. Delete the Pod with the following command (we can leave the Service operational).

```
$ kubectl delete -f pod.yml
pod "hello-pod" deleted
```

If you deleted a node from your cluster, add a new one now and remember to join it to the cluster with the `kubeadm join` command that was displayed in the terminal of `node1` when you initially built the cluster.

## 5. Use a Deployment for self-healing

Kubernetes has a **Deployment** object that adds scaling and self-healing to Pods.

Instead of deploying Pods directly, we deploy them via Deployments. Doing this allows Kubernetes to recover from failures and easily scale the number of Pod replicas up and down.

The following Deployment YAML file deploys a single replica of the Pod we deployed in the last section.

Copy the YAML into a new file called `deploy.yml` on your PWK cluster. To do this: Copy the YAML text > `vi deploy.yml` > `Ins` > Paste YAML > `Esc` > `:wq` > `Enter`.

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
  labels:
    app: web
    zone: prod
    version: v1
spec:
  selector:
    matchLabels:
      app: web
  replicas: 1
  strategy:
    type: RollingUpdate
  template:
    metadata:
      labels:
        app: web
        zone: prod
        version: v1
    spec:
      containers:
      - image: nigelpoulton/k8sbook:latest
        name: web-ctr
        ports:
        - containerPort: 8080
```

Let's step through the important parts of the YAML file starting from the top.

`apiVersion` and `kind` tell Kubernetes that we're creating a Deployment based on the schema defined in the `v1` core API group.

`spec.selector` tells Kubernetes that this Deployment is to manage all Pods on the cluster with the `app=web` label. We'll see more on this later when we scale the Deployment.

`spec.replicas` tells Kubernetes we just one one Pod to be deployed.

`spec.template` defines the Pod that we want to deploy. This is effectively the same as the Pod deployed in the previous section -- we give it some labels, base it on a Docker image, and expose it on port `8080`.

Let's recap before we deploy it. Kubernetes Deployments are all about deploying Pods and providing self-healing and scalability.

Deploy the Deployment with the following command. The command assumes the Deployment YAML file is in your system's PATH or you current working directory and is called `deploy.yml`

```
$ kubectl apply -f deploy.yml
deployment.apps/web created
```

After a few seconds you will have one replica of the exact same web server Pod running on your cluster.

Check with the following commands:

```
$ kubectl get pods
NAME                  READY     STATUS    RESTARTS   AGE
web-fff699549-vmqjx   1/1       Running   0          1m

$ kubectl get deploy -o wide
NAME      DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
web       1         1         1            1           1m

$ kubectl describe deploy web
Name:                   web
Namespace:              default
CreationTimestamp:      Wed, 10 Apr 2019 01:34:39 +0000
Labels:                 app=web
                        version=v1
                        zone=prod
Annotations:            deployment.kubernetes.io/revision=1
                        <Snip>
Selector:               app=web
Replicas:               1 desired | 1 updated | 1 total | 1 available | 0 unavailable
StrategyType:           RollingUpdate
MinReadySeconds:        0
RollingUpdateStrategy:  25% max unavailable, 25% max surge
Pod Template:
  Labels:  app=web
           version=v1
           zone=prod
  Containers:
   web-ctr:
    Image:        nigelpoulton/acg-web:0.1
    Port:         8080/TCP
    Host Port:    0/TCP
    Environment:  <none>
    Mounts:       <none>
  Volumes:        <none>
Conditions:
  Type           Status  Reason
  ----           ------  ------
  Available      True    MinimumReplicasAvailable
  Progressing    True    NewReplicaSetAvailable
OldReplicaSets:  <none>
NewReplicaSet:   web-fff699549 (1/1 replicas created)
Events:
  Type    Reason             Age   From                   Message
  ----    ------             ----  ----                   -------
  Normal  ScalingReplicaSet  20m   deployment-controller  Scaled up replica set web-fff699549 to 1
```

As we deployed the Pods with the same labels, the Service from the previous section will be managing traffic flows to the Pod. Test this with another `curl` or a refresh of your browser page. You can run the `curl` command, or point your browser to any node in the cluster as the `NodePort` is exposed on every node in the cluster.

```
$ curl localhost:30001
<html><head><title>ACG loves K8S</title><link rel="stylesheet" href="http://netdna.bootstrapcdn.com/bootstrap/3.1.1/css/bootstrap.min.css"/></head><body><div class="container"><div class="jumbotron"><h1>A Cloud Guru loves Kubernetes!!!</h1><p></p><p> <a class="btn btn-primary" href="https://www.amazon.com/Kubernetes-Book-Nigel-Poulton/dp/1521823634/ref=sr_1_3?ie=UTF8&amp;qid=1531240306&amp;sr=8-3&amp;keywords=nigel+poulton">The Kubernetes Book</a></p><p></p></div></div></body></html>
```

The reason that this works is because the Service implements a watch-loop on the control plane that is constantly watching for new Pods with the `app=web` label. We just deployed a new Pod with that label, so the Service will manage traffic for it.

### Self-healing

Now let's test the self-healing capabilities of Deployments by deleting the Pod.

Delete the Pod with the following command. The name of the Pod will be different on your system (get the name of your Pod with `kubectl get pods`).

```
$ kubectl delete pod web-fff699549-vmqjx
pod "web-fff699549-vmqjx" deleted
```

It may take a few seconds from the Pod to delete.

Once the Pod is deleted, run another `kubectl get pods`. You will see that the Pod has been re-created, but that it has a slightly different name (the name of the Deployment plus a hash). This is not the old deleted Pod brought back to life, it is a brand new Pod with exactly the same spec as the one just deleted.

The reason that the terminated Pod has been recreated is that the Deployment implements a watch-loop on the control plane. This watch-loop knows that we've asked for one replica of the Pod - we call this _desired state_. It's constantly checking that the _actual state_ of the cluster matches the _desired state_. When we delete the Pod, _actual state_ shows zero replicas of the Pod, but _desired state_ is still one. Therefore, the Deployment controller rectifies the situation by starting a new replica.

Feel free to check that you can still connect a browser to the web service, or that the `curl` command still works.

## 6. Scale the application

In this section we'll scale the number of Pod replicas up and then down.

Before diving in, it's worth clarifying that a Deployment only manages a single Pod definition. For example, if you have a front-end Pod and a back-end Pod, you'll need two Deployments to manage them -- a single Deployment cannot manages two different Pods.

This section picks up form the previous section, so you should have a single Deployment managing a single Pod.

Edit the existing `deploy.yml` file and increase the replica count from 1 to 8. To do this: `vi deploy.yml` > `Ins` > change replicas from 1 to 8 > `Esc` > `:wq`.

The following YAML snippet shows the changed section.

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
  <Snip>
spec:
  <Snip>
  replicas: 8     << This is the only line that changes
<Snip>
```

**NOTE:** There might be an issue with the size of nodes on PWK where more than one Pod per node causes evictions and major issues.

Save the changes and redeploy the configuration with the following command.

```
$ kubectl apply -f deploy.yml
deployment.apps/web configured
```

Check the status with commands like `kubectl get deploy web --watch` and `kubectl get pods --watch`.

```
$ kubectl get deploy web --watch
NAME      DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
web       8         8         8            1           3m

<Time lapse>

NAME      DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
web       8         8         8            8           6m
```

```
$ kubectl get pods --watch
NAME                  READY     STATUS              RESTARTS   AGE       IP          NODE      NOMINATED NODE
web-fff699549-4l85z   0/1       ContainerCreating   0          29s       <none>      node3     <none>
web-fff699549-5gjsd   0/1       ContainerCreating   0          29s       <none>      node2     <none>
web-fff699549-lk2x6   0/1       ContainerCreating   0          29s       <none>      node4     <none>
web-fff699549-qltgn   1/1       Running             0          3m        10.36.0.1   node5     <none>
web-fff699549-5gjsd   1/1       Running   0         1m        10.44.0.1   node2     <none>
web-fff699549-4l85z   1/1       Running   0         1m        10.42.0.1   node3     <none>
web-fff699549-lk2x6   1/1       Running   0         1m        10.47.0.1   node4     <none>
```

Run a `kubectl get pods -o wide` to see that each Pod is running on a separate worker node.


```
$ kubectl get pods -o wide
NAME                   READY     STATUS    RESTARTS   AGE       IP          NODE      NOMINATED NODE
web-6fc4cf749d-4jmj8   1/1       Running   0          5m        10.44.0.1   node2     <none>
web-6fc4cf749d-b7z87   1/1       Running   0          1m        10.42.0.1   node4     <none>
web-6fc4cf749d-jngp6   1/1       Running   0          1m        10.36.0.1   node3     <none>
web-6fc4cf749d-q4rnp   1/1       Running   0          1m        10.40.0.1   node5     <none>
Snip>
```

The app is now scaled to 8 replicas. Each Pod replica is an exact copy of the others, with the only differences being things like Pod IP and Pod ID.

You can edit the same `deploy.yml` file to scale the number of Pods down to 4. Edit the YAML file and change the `spec.replicas` field from 8 to 4. The following YAML snippet shows the only line in the file that has changed.

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
  <Snip>
spec:
  <Snip>
  replicas: 4      << This is the only line that changes
<Snip>
```

To do this: `vi deploy.yml` > `Ins` > change replicas from 8 to 4 > `Esc` > `:wq`.

Re-apply the config with `kubectl apply -f`.

```
$ kubectl apply -f deploy.yml
deployment.apps/web configured
```

Give the cluster a few seconds to scale the Deployment down from 8 replicas to 4.

It's important to note that the regular Deployment watch loop is responsible for initiation scaling operations. When we posted the last configuration (`spec.replicas=2`) we were posting a new _desired state_.  Once the configuration was accepted on the API server and persisted to the cluster store, the Deployment watch loop will notice the change --- it will see that the new _desired state_ is 2 replicas, but the _current observed state_ is 4. It will then initiate the work required to make _observed state_ match _desired state_. This is exactly the same logic and process that is followed when failures occur --- Kubernetes is constantly monitoring the cluster to ensure that _observed state_ matches _desired state_.

Make one more edit to the `deploy.yml` file to take the cluster back to 8 replicas. Save the changes and apply them to the cluster with `kubectl apply -f deploy.yml`.  Verify the operation with `kubectl get pods`. Do not continue until you have 4 running replicas.


## 7. Use a cloud load-balancer

Kubernetes makes it really simple to expose your application to the internet if you're running on one of the major cloud providers. It does this by integrating with your cloud service's internet facing load-balancers.

In this section, we'll deploy a new service called a Load-balancer service. If you're following along on one of the major public cloud providers such as AWS, Azure, DO, or GCP, this will automatically provision one of your cloud's load-balancers. It will also integrate this load-balancer with the Kubernetes Deployment.

> **Note:** A couple of things to note. This lab will create a cloud load-balancer which may incur additional costs. This will not work if you are following along on Docker Desktop, Minikube, or on-premises Kubernetes clusters.

Look at the `svc-lb.yml` file in the lab's GitHub repo.

```
apiVersion: v1
kind: Service
metadata:
  name: svc-lb
  labels:
    app: web
spec:
  type: LoadBalancer
  ports:
    - port: 8080
      targetPort: 8080
  selector:
    app: web
```

The new Service will be called "svc-lb".

The front-end configuration of the Service defines a `LoadBalancer` service that maps port `8080` on the load-balancer to port `8080` on the app.

The back-end configuration of the Service maps port `8080` to any healthy Pod with the `app=web` label.

Deploy the Service.

```
$ kubectl apply -f svc-lb.yml
service "svc-lb" created
```

Run the following command to watch the Service come up. The `--watch` flag lets us monitor the creation of the Service and see it acquire an internet IP address. The `EXTERNAL-IP` attribute of the Service may remain `<pending>` for a minute or two while Kubernetes arranges the creation of the cloud load-balancer.

```
$ kubectl get svc svc-lb --watch
NAME         TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
svc-lb       LoadBalancer   10.31.252.253   <pending>     8080:31805/TCP   11s
svc-lb    LoadBalancer   10.31.252.253   35.242.128.109   8080:31805/TCP   57s
```

Once the Service is deployed -- with a valid `EXTERNAL-IP` -- you can point a browser to that public IP on port `8080` and get access to the web server running in the Deployment.

Congratulations, you've successfully exposed your application to the internet using one of your cloud's native load-balancers.


## 8. Perform a rolling update

Kubernetes supports native rolling updates of applications deployed via a Kubernetes Deployment.

So far, we've deployed a web app using a Kubernetes Deployment and we currently have 8 replicas exposed via a cloud load-balancer. The app is based on the `nigelpoulton/k8sbook:latest` image. In this section we'll update the app to use the `nigelpoulton/k8sbook:edge` image.

The best way to perform a rolling update is declaratively. This requires you to edit the existing `deply.yml` file and change the image that the app uses. You then `POST` the updated YAML file to Kubernetes and let the Deployment controller take care of performing the update. In the real world, you'll have your YAML files stored in version control systems.

Edit the `deploy.yml` file so that `spec.template.spec.containers.image` is nigelpoulton/k8sbook:**edge**. Do not change any other values in the YAML file.

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
  labels:
    app: web
    zone: prod
    version: v1
spec:
  selector:
    matchLabels:
      app: web
  replicas: 8
  strategy:
    type: RollingUpdate
  template:
    metadata:
      labels:
        app: web
        zone: prod
        version: v1
    spec:
      containers:
      - image: nigelpoulton/k8sbook:edge   << This line changed
        name: web-ctr
        ports:
        - containerPort: 8080
```

Notice that the `spec.strategy.type` field specifies `RollingUpdate`. This indicates a **zero-downtime** update. The other option is `Recreate`, this kills all Pods before creating new ones and will result in application downtime.

There are other options that allow you fine-tune how the rolling update occurs, but they're beyond the scope of this intro course. The default behaviour is to create one new replica with the new version, and once that is running delete one old replica with the old version. Kubernetes rolls through this process until there are 4 replicas running the new version and no replicas running the old version.

Save changes to the `deploy.yml` file and POST it to the API server.

```
$ kubectl apply -f deploy.yml --record
deployment.apps "web" configured
```

Monitor the progress with the following command.

```
$ kubectl rollout status deployment web

Waiting for rollout to finish: 2 out of 4 new replicas have been updated...
Waiting for rollout to finish: 2 out of 4 new replicas have been updated...
Waiting for rollout to finish: 2 out of 4 new replicas have been updated...
Waiting for rollout to finish: 2 out of 4 new replicas have been updated...
Waiting for rollout to finish: 3 out of 4 new replicas have been updated...
Waiting for rollout to finish: 3 out of 4 new replicas have been updated...
Waiting for rollout to finish: 3 out of 4 new replicas have been updated...
Waiting for rollout to finish: 3 out of 4 new replicas have been updated...
Waiting for rollout to finish: 3 out of 4 new replicas have been updated...
Waiting for rollout to finish: 1 old replicas are pending termination...
Waiting for rollout to finish: 1 old replicas are pending termination...
Waiting for rollout to finish: 1 old replicas are pending termination...
Waiting for rollout to finish: 3 of 4 updated replicas are available...
Waiting for rollout to finish: 3 of 4 updated replicas are available...
deployment "web" successfully rolled out
```

Refresh the browser page to see the new version of the web site.

Congratulations, you've successfully update your app.

## What next

Practice practice pratice!

- Check out my [Getting Started with Kubernetes](https://app.pluralsight.com/library/courses/getting-started-kubernetes/table-of-contents) video course
- Check out my [Kubernetes Deep Dive](https://acloud.guru/learn/kubernetes-deep-dive) video course
- Check out my Kubernetes book (paperback, ebook, and audiobook) [The Kubernetes Book](https://www.amazon.com/s?k=nigel+poulton&sprefix=nigel+pou&ref=is_s_ss_i_0_9)
