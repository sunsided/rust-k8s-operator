# Kubernetes Controller (written in Rust)

This repo is stitched together from these sources:

- [Pscheidl/rust-kubernetes-operator-example]

## Creating a Kubernetes cluster

It is recommended to try out this project using [kind].
To create a new cluster and set it as the current context, run 

```console
$ kind create cluster --name k8s-operator
$ kubectl config use-context kind-k8s-operator
```

To delete the cluster, run

```console
$ kind delete cluster --name k8s-operator
```

## Applying the Custom Resource Definition

To apply the [echoes.example.com.yaml] CRD, run

```console
$ kubectl apply -f k8s/echoes.example.com.yaml
```

You can verify the CRD creation using

```console
$ kubectl get crds
```

This should give something like

```console
NAME                 CREATED AT
echoes.example.com   2021-07-14T21:51:51Z
```

## Running the Operator locally

For the Operator to work the `KUBECONFIG` environment variable
needs to be defined. When running the application locally,
use e.g.

```console
$ export KUBECONFIG=${HOME}/.kube/config
```

⚠️ **Starting the Operator now will connect it to the context that is currently selected.** 

To start the Operator, run

```console
$ cargo run
```

The Operator will now create resources according to the definition in [echo.rs].

Apply the [echo-example.yaml] resource:

```console
$ kubectl apply -f k8s/echo-example.yaml
```

You can verify that it created the required resources by running

```console
$ kubectl get pods
```

which should list something like

```console
NAME                         READY   STATUS    RESTARTS   AGE
test-echo-6b5f99cf8d-86s2q   1/1     Running   0          23s
test-echo-6b5f99cf8d-8x99s   1/1     Running   0          23s
```

You can use `kubectl describe pod` to describe any of the running pods:

```console
$ kubectl describe pod test-echo-6b5f99cf8d-86s2q
```

<details>
<summary>This is how the pod description looks like.</summary>

```text
Name:         test-echo-6b5f99cf8d-86s2q
Namespace:    default
Priority:     0
Node:         k8s-operator-control-plane/172.19.0.5
Start Time:   Thu, 15 Jul 2021 00:21:27 +0200
Labels:       app=test-echo
              pod-template-hash=6b5f99cf8d
Annotations:  <none>
Status:       Running
IP:           10.244.0.5
IPs:
  IP:           10.244.0.5
Controlled By:  ReplicaSet/test-echo-6b5f99cf8d
Containers:
  test-echo:
    Container ID:   containerd://40c65b2bf18ac1289c5f651334f9a7d0950725dbd033b59d1ab36690af28b128
    Image:          inanimate/echo-server:latest
    Image ID:       docker.io/inanimate/echo-server@sha256:f988e7b6320f65548a418eb79a7ff26faf1a4a0d6d59f63bc8c3fc116b2dcff1
    Port:           8080/TCP
    Host Port:      0/TCP
    State:          Running
      Started:      Thu, 15 Jul 2021 00:21:33 +0200
    Ready:          True
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from default-token-rzpjj (ro)
Conditions:
  Type              Status
  Initialized       True 
  Ready             True 
  ContainersReady   True 
  PodScheduled      True 
Volumes:
  default-token-rzpjj:
    Type:        Secret (a volume populated by a Secret)
    SecretName:  default-token-rzpjj
    Optional:    false
QoS Class:       BestEffort
Node-Selectors:  <none>
Tolerations:     node.kubernetes.io/not-ready:NoExecute for 300s
                 node.kubernetes.io/unreachable:NoExecute for 300s
Events:
  Type    Reason     Age    From                                 Message
  ----    ------     ----   ----                                 -------
  Normal  Scheduled  3m53s  default-scheduler                    Successfully assigned default/test-echo-6b5f99cf8d-86s2q to k8s-operator-control-plane
  Normal  Pulling    3m52s  kubelet, k8s-operator-control-plane  Pulling image "inanimate/echo-server:latest"
  Normal  Pulled     3m47s  kubelet, k8s-operator-control-plane  Successfully pulled image "inanimate/echo-server:latest" in 5.008959095s
  Normal  Created    3m47s  kubelet, k8s-operator-control-plane  Created container test-echo
  Normal  Started    3m47s  kubelet, k8s-operator-control-plane  Started container test-echo
```
</details>

Since the definition in [echo.rs] contains a port definition, we can verify
that the deployed Echo service is working correctly by using a port-forward.

```console
$ kubectl port-forward test-echo-6b5f99cf8d-86s2q 10080:8080
$ curl localhost:10080
```

<details>
<summary>This is how the output looks like.</summary>

```text
Welcome to echo-server!  Here's what I know.
  > Head to /ws for interactive websocket echo!

-> My hostname is: test-echo-6b5f99cf8d-86s2q

-> Requesting IP: 127.0.0.1:41874

-> Request Headers | 

  HTTP/1.1 GET /

  Host: localhost:10080
  Accept: */*
  User-Agent: curl/7.71.1


-> Response Headers | 

  Content-Type: text/plain
  X-Real-Server: echo-server

  > Note that you may also see "Transfer-Encoding" and "Date"!


-> My environment |
  ADD_HEADERS={"X-Real-Server": "echo-server"}
  HOME=/
  HOSTNAME=test-echo-6b5f99cf8d-86s2q
  KUBERNETES_PORT=tcp://10.96.0.1:443
  KUBERNETES_PORT_443_TCP=tcp://10.96.0.1:443
  KUBERNETES_PORT_443_TCP_ADDR=10.96.0.1
  KUBERNETES_PORT_443_TCP_PORT=443
  KUBERNETES_PORT_443_TCP_PROTO=tcp
  KUBERNETES_SERVICE_HOST=10.96.0.1
  KUBERNETES_SERVICE_PORT=443
  KUBERNETES_SERVICE_PORT_HTTPS=443
  PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
  PORT=8080
  SSLPORT=8443


-> Contents of /etc/resolv.conf | 
search default.svc.cluster.local svc.cluster.local cluster.local
nameserver 10.96.0.10
options ndots:5


-> Contents of /etc/hosts | 
# Kubernetes-managed hosts file.
127.0.0.1	localhost
::1	localhost ip6-localhost ip6-loopback
fe00::0	ip6-localnet
fe00::0	ip6-mcastprefix
fe00::1	ip6-allnodes
fe00::2	ip6-allrouters
10.244.0.5	test-echo-6b5f99cf8d-86s2q



-> And that's the way it is 2021-07-14 22:28:25.606907696 +0000 UTC

// Thanks for using echo-server, a project by Mario Loria (InAnimaTe).
// https://github.com/inanimate/echo-server
// https://hub.docker.com/r/inanimate/echo-server
```
</details>

To delete the custom resource, run

```console
$ kubectl delete echo  echo-example
```

[kind]: https://kind.sigs.k8s.io/
[echo.rs]: src/echo.rs
[Pscheidl/rust-kubernetes-operator-example]: https://github.com/Pscheidl/rust-kubernetes-operator-example
[echo-example.yaml]: k8s/echo-example.yaml
[echoes.example.com.yaml]: k8s/echoes.example.com.yaml
