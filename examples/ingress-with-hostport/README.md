# Example of Kind cluster ingress using `hostPort` on `Pod`

Create a Kind cluster that has ingress to an Nginx pod. For more information about how this works see the top-level README.md.

## Dependencies

The following tools are needed to run this project:
* kind ([install](https://kind.sigs.k8s.io/docs/user/quick-start/#installation))
* kubectl ([install](https://kubernetes.io/docs/tasks/tools/#kubectl))
* curl (installation depends on OS, or you can [download](https://curl.se/download.html))

## Preface

If using **Podman** you will need set the ENV variable:

```sh
export KIND_EXPERIMENTAL_PROVIDER=podman
```

## How to

**Step 1**: Create the kind cluster:

```sh
kind create cluster --config kind-config-ingress-with-hostport.yaml
```

**Step 2**: Install the Nginx pod and wait until ready:

```sh
kubectl wait --for=jsonpath='{.metadata.uid}' serviceaccount/default
kubectl apply -f pod-nginx.yaml
kubectl wait --for=condition=Ready pod/get-traffic-from-host
```

**Step 3**: Check we have connectivity:

```sh
curl http://localhost:8081
```

And you should see the HTML output from the Nginx pod.

Yay!

**Step 4**: Delete the cluster (Important!)

```sh
kind delete cluster --name ingress-with-hostport
```