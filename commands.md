=== Create and deploy a Pod ===

vi pod.yml

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

kubectl apply -f pod.yml
kubectl get pods
kubectl describe pods hello-pod


=== Create and deploy a Service ===

vi svc.yml

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

kubectl apply -f svc.yml
kubectl get svc
kubectl describe svc svc-np
curl localhost:30001


=== Terminate a node ===

kubectl get pods -o wide

Terminate the node that the Pod is running on 

Refresh browser or run `curl` again

Create new node and add it to the cluster

Clean-up the lab by deleting the Pod


=== Create and deploy and Deployment ===

vi deploy.yml

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

kubectl apply -f deploy.yml
kubectl get pods
kubectl get deploy -o wide
kubectl describe deploy web
curl localhost:30001


=== Demonstrate self-healing ===

kubectl get pods
kubectl delete pod <INSERT POD NAME>
kubectl get pods
curl localhost:30001   OR refresh browser


=== Demonstrate scaling ===

EDIT deploy.yml and change to 8 replicas

vi deploy.yml
Change replica count to 8

kubectl apply -f deploy.yml
kubectl get deploy web --watch
kubectl get pods --watch
kubectl get pods -o wide

Change replica count back to 6


=== Provision and use a cloud load-balancer ===

vi svc-lb.yml

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

kubectl apply -f svc-lb.yml
kubectl get svc svc-lb --watch
Connect to LoadBalancer URL shown in the command above


=== Demonstrate a rolling update ===

vi deploy.yml
Change image used to `edge` tag

kubectl apply -f deploy.yml --record
kubectl rollout status deployment web

Re-run `curl` or refresh browser
