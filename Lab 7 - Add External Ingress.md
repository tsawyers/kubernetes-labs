[Kubernetes Service Documentation - External Ingress](https://docs.mesosphere.com/services/kubernetes/1.2.0-1.10.5/ingress/)

# Adding External Ingress and Exposing an Application
For this Lab we will install Traefik as our Kubernetes Ingress Controller to show a Hello World example

### Step 1: Scale Public Agent Node
If you havent already done so, scale your Kubernetes `"public_node_count": 1,`. Go back to Lab 5 for detailed instructions on how to scale/update the Kubernetes Framework

If you want to expose HTTP/S (L7) apps to the outside world - at least outside the DC/OS cluster - you should create a Kubernetes Ingress resource. The package does not install such controller by default, so we give you the freedom to choose what Ingress controller your organization wants to use.
Options:
- Traefik
- NGINX
- HAProxy
- Envoy
- Istio

### Step 2:Explore and Deploy Traefik Ingress Controller

Take a look the below Traefik Ingress Controller:
```
---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: traefik-ingress-controller
rules:
  - apiGroups:
      - ""
    resources:
      - services
      - endpoints
      - secrets
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - extensions
    resources:
      - ingresses
    verbs:
      - get
      - list
      - watch
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: traefik-ingress-controller
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: traefik-ingress-controller
subjects:
- kind: ServiceAccount
  name: traefik-ingress-controller
  namespace: kube-system
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: traefik-ingress-controller
  namespace: kube-system
---
kind: Deployment
apiVersion: extensions/v1beta1
metadata:
  name: traefik-ingress-controller
  namespace: kube-system
  labels:
    k8s-app: traefik-ingress-lb
spec:
  replicas: 1
  selector:
    matchLabels:
      k8s-app: traefik-ingress-lb
  template:
    metadata:
      labels:
        k8s-app: traefik-ingress-lb
        name: traefik-ingress-lb
    spec:
      serviceAccountName: traefik-ingress-controller
      terminationGracePeriodSeconds: 60
      containers:
      - image: traefik
        name: traefik-ingress-lb
        args:
        - --web
        - --kubernetes
# NOTE: What follows are necessary additions to
# https://docs.traefik.io/user-guide/kubernetes
# Please check below for a detailed explanation
        ports:
        - containerPort: 80
          hostPort: 80
          name: http
          protocol: TCP
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: k8s-app
                operator: In
                values:
                - traefik-ingress-lb
            topologyKey: "kubernetes.io/hostname"
      nodeSelector:
        kubernetes.dcos.io/node-type: public
      tolerations:
      - key: "node-type.kubernetes.dcos.io/public"
        operator: "Exists"
        effect: "NoSchedule"
```

Creating these resources will cause Traefik to be deployed as an ingress controller in your cluster. We have made some additions to the official manifests so that the deployment works as expected:
- Bind each pod’s :80 port to the host’s :80 port. This is the easiest way to expose the ingress controller on the public node as DC/OS already opens up the :80 port on public agents. 
- Make use of pod anti-affinity to ensure that pods are spread among available public agents in order to ensure high-availability.
- Make use of the nodeSelector constraint to force pods to be scheduled on public nodes only.
- Make use of the node-type.kubernetes.dcos.io/public node taint so that the pods can actually run on the public nodes.

#### Deploy your Traefik Ingress Controller
```
kubectl create -f https://raw.githubusercontent.com/ably77/kubernetes-labs/master/resources/traefik.yaml
```

### Step 3:Explore and Deploy Traefik Ingress Service

Take a look the below Traefik Ingress Service:
```
apiVersion: v1
kind: Service
metadata:
  name: traefik-ingress-controller
  namespace: kube-system
spec:
  selector:
    k8s-app: traefik-ingress-controller
  ports:
    - port: 80
      name: http
  type: NodePort
  ```
 
Deploy Ingress Service:
```
kubectl create -f https://raw.githubusercontent.com/ably77/kubernetes-labs/master/resources/traefik-ingress.yaml
```

### Step 4 :Explore and Deploy Hello World deployment

Take a look the below Hello World deployment:
```
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-world
  labels:
    app: hello-world
spec:
  replicas: 2
  selector:
    matchLabels:
      app: hello-world
  template:
    metadata:
      labels:
        app: hello-world
    spec:
      containers:
      - name: echo-server
        image: hashicorp/http-echo
        args:
        - -listen=:80
        - -text="Hello from Kubernetes running on Mesosphere DC/OS!"
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: hello-world
spec:
  selector:
    app: hello-world
  ports:
    - port: 80
      targetPort: 80
```

Deploy Hello World Deployment
```
kubectl create -f https://raw.githubusercontent.com/ably77/kubernetes-labs/master/resources/hello-world.yaml
```

### Step 5 :Explore and Deploy Hello World Ingress

Take a look the below Hello World Ingress configuration:
```
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  annotations:
    kubernetes.io/ingress.class: traefik
  name: hello-world
spec:
  rules:
  - http:
      paths:
      - path: /
        backend:
          serviceName: hello-world
          servicePort: 80
```

Deploy Hello World Ingress:
```
kubectl create -f https://raw.githubusercontent.com/ably77/kubernetes-labs/master/resources/hw-ingress.yaml
```
  
### Step 6: Access your Hello World Application
We can access Hello World Application by accessing `http://<PUBLIC_KUBELET_IP>`. Here is an easy way to get the Public Kubelet IP:
```
dcos task exec -it kube-node-public-0-kubelet curl ifconfig.co
```

Open in your browser:
```
open http://<PUBLIC_KUBELET_IP>
```

## Adding External Ingress and Exposing an Application lab complete
In this lab we have created a Kubernetes Ingress Controller object as well as deployed our Hello World application. After creating the remaining Service and Ingress objects, we now have a publicly exposed web application.


[Labs home](https://github.com/c-mcinerney/kubernetes-labs)

[Lab 1 - Installing Kubernetes](https://github.com/c-mcinerney/kubernetes-labs/blob/master/Lab%201%20-%20Installing%20Kubernetes.md)


[Lab 2 - Upgrading Cluster](https://github.com/c-mcinerney/kubernetes-labs/blob/master/Lab%202%20-%20Upgrading%20Cluster.md)

[Lab 3 - Going HA](https://github.com/c-mcinerney/kubernetes-labs/blob/master/Lab%203%20-%20Going%20HA.md)

[Lab 4 - Killing Node](https://github.com/c-mcinerney/kubernetes-labs/blob/master/Lab%204%20-%20Killing%20Node.md)

[Lab 5 - Running Apps](https://github.com/c-mcinerney/kubernetes-labs/blob/master/Lab%205%20-%20Running%20Apps.md)

[Lab 6 - Scaling Kubernetes](https://github.com/c-mcinerney/kubernetes-labs/blob/master/Lab%206%20-%20Scaling%20Kubernetes.md)

[Lab 7 - Add External Ingress](https://github.com/c-mcinerney/kubernetes-labs/blob/master/Lab%207%20-%20Add%20External%20Ingress.md)

[Lab 8 - Kafka Streams](https://github.com/c-mcinerney/kubernetes-labs/blob/master/Lab%208%20-%20Kafka%20Streams.md)


