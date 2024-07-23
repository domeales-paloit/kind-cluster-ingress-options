# Traffic ingress options into Kind clusters

<!-- TOC -->
- [Some background: Networking with Kind clusters](#some-background-networking-with-kind-clusters)
    - [Getting traffic into Kind clusters](#getting-traffic-into-kind-clusters)
    - [Getting traffic from `Node`to `Pod`](#getting-traffic-from-nodeto-pod)
        - [Using `hostPort` on `Pod`](#using-hostport-on-pod)
        - [Using `Service` with type `NodePort`](#using-service-with-type-nodeport)
<!-- /TOC -->

Here are some basic examples of how to get traffic into a Kind cluster and a description of how they work.

Try these examples to see how they work:

Example 1: Using `hostPort` on `Pod` - [examples/ingress-via-hostport](examples/ingress-via-hostport)

Example 2: Using type of `NodePort`on `Service`- [examples/ingress-via-nodeport](examples/ingress-via-nodeport) (My preferred method)

---
---

## Some background: Networking with Kind clusters

To help discuss the Kind networking this document uses these terms:

* *Host* - The machine running the Kind cluster
* *Node container* - A Docker/Podman container running on the *Host* that is a `Node` for the Kind cluster
* *Pod container* - A containerd container running on the *Node container* ([DinD](https://duckduckgo.com/?q=docker+in+docker+dind) style) which is a Kubernetes container, attached to a `Pod`

### Getting traffic into Kind clusters

Since each `Node` in our Kind cluster is a Docker/Podman container there is an additional networking interface to configure to get traffic into the cluster: that is the *Host* <==> *Node container* network interface.

There are multiple ways to get traffic through this interface to the correct pods, but they all include the use of the Kind config parameter `extraPortMappings` ([doc](https://kind.sigs.k8s.io/docs/user/configuration/#extra-port-mappings)). This parameter allows adding port mappings between the *Host* and the *Node container* (ie. node) in question.

### Getting traffic from `Node`to `Pod`

Once the traffic is in the *Node container*, it needs to arrive at the *Pod container* in question.

There are at least two ways to do this (there may be more):
1. Using the `hostPort` parameter of a `ports` entry for the container of a `Pod`
1. Using a `Service` object with the type `NodePort`

#### Using `hostPort` on `Pod`

This is the method described in the [Kind docs](https://kind.sigs.k8s.io/docs/user/ingress/) to get the Ingress Nginx controller working for Kind.

Using the `hostPort` ([doc](https://kubernetes.io/docs/reference/kubernetes-api/workload-resources/pod-v1/#ports)) parameter on a `Pod` spec is one way to get the traffic from the *Host* to the *Pod container*. To ensure this works correctly we need to make sure that the `Pod` (ie. *Pod container*) is running on the `Node` (ie. *Node container*) that has the `extraPortMappings` (see above).

> _Note_: See full example with instructions in [examples/ingress-with-hostport](examples/ingress-with-hostport) directory.

```yaml
#
# kind-config.yaml
#
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
name: ingress-with-hostport
nodes:
- role: control-plane
  kubeadmConfigPatches:
  # Add a node label to the control-plane node so that we can
  # target it with the hostPort on the Pod
  - |
    kind: InitConfiguration
    nodeRegistration:
      kubeletExtraArgs:
        node-labels: "extra-port-8082=true"
  extraPortMappings:
  - protocol: TCP
    # The container port is the port that will be available on the
    # "Node container" (Docker/Podman container) for the "Pod container"
    # (containerd container)
    containerPort: 8082
    # The hostPort in this configuration is the port on the "Host"
    # (where Kind is running). This is the port we can access from our
    # localhost to get traffic to the "Pod container"
    hostPort: 8081
```

and here is the `Pod` spec

```yaml
#
# Nginx server pod
#
apiVersion: v1
kind: Pod
metadata:
  name: get-traffic-from-host
spec:
  containers:
  - name: nginx
    image: nginx
    imagePullPolicy: IfNotPresent
    ports:
    - containerPort: 80
      # Here we had the upstream port, but remember in our case the "Host"
      # here is a "Node container" (Docker/Podman) so we are using the value 8082
      hostPort: 8082
      name: http
      protocol: TCP
  # Adding a nodeSelector that matchs the "node-labels" from the Kind
  # config ensures that the Pod container is created on the Node that
  # has the `extraPortMappings`
  nodeSelector:
    extra-port-8082: "true"
```


#### Using `Service` with type `NodePort`

In this scenario we select a NodePort port value (32221) that will be used to allow traffic into the cluster via the `extraPortMappings` on the control-plane node (both hostPort and containerPort). Once the traffic is in the cluster we can create a `Service` of type `NodePort` with the same port value (32221).

To ensure that the traffic is really handled by the CNI, we force the `Pod` to run on the worker node, where there are no `extraPortMappings`.

> _Note_: See full example with instructions in [examples/ingress-with-nodeport](examples/ingress-with-nodeport) directory.

```yaml
#
# kind-config.yaml
#
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
name: ingress-with-nodeport
nodes:
- role: control-plane
  kubeadmConfigPatches:
  extraPortMappings:
  # We have selected the NodePort value and we map it to both the 
  # containerPort and hostPort
  - containerPort: 32221
    hostPort: 32221
    protocol: TCP
- role: worker
  kubeadmConfigPatches:
  # Add a node label to the control-plane node so that we can
  # target it for our Pod
  - |
    kind: JoinConfiguration
    nodeRegistration:
      kubeletExtraArgs:
        node-labels: "worker-node=true"
````

And the `Pod` spec

```yaml
#
# Nginx server pod
#
apiVersion: v1
kind: Pod
metadata:
  name: get-traffic-from-host
  labels:
    app.kubernetes.io/name: NginxServer
spec:
  containers:
  - name: nginx
    image: nginx
    imagePullPolicy: IfNotPresent
    ports:
    - containerPort: 80
      name: http
      protocol: TCP
  # Adding a nodeSelector that matchs the "node-labels" from the Kind
  # config ensures that the Pod container is created on the worker Node
  # which doesn't have the `extraPortMappings`. This really tests that the
  # traffic that arrives on the mapped port on the control-plane Node is
  # forwarded to the worker Node.
  nodeSelector:
    worker-node: "true"
```

And finally the `Service` spec

```yaml
#
# Nginx Service
#
apiVersion: v1
kind: Service
metadata:
  name: get-traffic-from-host
spec:
  type: NodePort
  selector:
    app.kubernetes.io/name: NginxServer
  ports:
    - port: 80
      targetPort: 80
      # This is where we set the same port for NodePort as the one that
      # was defined in the Kind config `extraPortMappings`
      nodePort: 32221
```