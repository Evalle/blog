# The Strange Case of Frequent Abnormal Restarts of kube-apiserver

*You probably heard that if you want to secure your Kubernetes cluster you should
turn off the anonymous requests via adding `anonymous-auth=false` to apiserver's
options... I'm not saying that you should not do that, but I highly recommend you to
read this blogpost before taking any action :)*

## Introduction

First thing first, you should probably read this [article](https://medium.com/handy-tech/analysis-of-a-kubernetes-hack-backdooring-through-kubelet-823be5c3d67c)
to understand **why** we should add `anonymous-auth=false` option to kube-apiserver.

TL;DR: *"If your users have network access to your nodes, then the kubelet API is a full featured unauthenticated API backdoor to your cluster"*
At least it was like this before Kubernetes developers have introduced RBAC by default. But even now I can gather some info about kubernetes cluster via
simple curl:

```bash
~ > curl -k https://192.168.1.100:6443/healthz
ok
~ > curl -k https://192.168.1.100:6443/version
{
  "major": "1",
  "minor": "9",
  "gitVersion": "v1.9.6",
  "gitCommit": "9f8ebd171479bec0ada837d7ee641dec2f8c6dd1",
  "gitTreeState": "clean",
  "buildDate": "2018-03-21T15:13:31Z",
  "goVersion": "go1.9.3",
  "compiler": "gc",
  "platform": "linux/arm"
```

The most direct way to solve this problem is to not accept anonymous requests. We do this by adding the following line to `/etc/kubernetes/manifests/kube-apiserver.yaml`:

```yaml
...
spec:
  containers:
  - command:
    - kube-apiserver
    - --anonymous-auth=false
...
```

After you save these changes kubelet will restart apiserver automatically and you will see that the anonymous requests don't work anymore:

```bash
~> curl -k https://192.168.1.100:6443/version
{
  "kind": "Status",
  "apiVersion": "v1",
  "metadata": {

  },
  "status": "Failure",
  "message": "Unauthorized",
  "reason": "Unauthorized",
  "code": 401

```

Looks good? Wait for it.

## Abnormal restarts of apiserver

After the above mentioned change you will most probably see the abnormal restarts of your apiserver:

```bash
~> kubectl get po kube-apiserver -n kube-system
NAME                                                   READY     STATUS    RESTARTS   AGE
kube-apiserver                                         1/1       Running   6          7m
```

Why this happened?

## Liveness Probes

We've faced the problem with liveness probes - kubelet asks apiserver as **anonymous anauthenticated** user
and we just closed all anonymous requests! Kubelet thinks that our apiserver is unhealthy and keep restarting it
over and over again. According to liveness probe in kube-apiserver YAML file:

```yaml
...
    livenessProbe:
      failureThreshold: 8
      httpGet:
        host: 127.0.0.1
        path: /healthz
        port: 6443
        scheme: HTTPS
      initialDelaySeconds: 15
      timeoutSeconds: 15
...
```

## Three Ways

I can see three ways of how we can solve this issue

### 1. Use insecure port and insecure address

We can open insecure port and insesure address on apiserver so the liveness probes will be made from kubelet without
any authentication. We do this by adding the following lines to `/etc/kubernetes/manifests/kube-apiserver.yaml`:

```yaml
spec:
  containers:
  - command:
    - kube-apiserver
    - --anonymous-auth=false
    - --insecure-port=8888
    - --insecure-bind-address=127.0.0.1
    ...
    livenessProbe:
      failureThreshold: 8
      httpGet:
        host: 127.0.0.1
        path: /healthz
        port: 8888
        scheme: HTTP
      initialDelaySeconds: 15
      timeoutSeconds: 15
    ...
```

Now kube-apiserver is healthy and we can make insecure request to `/healthz` endpoint only from the same host (like kubelet does)

```bash
~> kubectl get po kube-apiserver -n kube-system

NAME                                                   READY     STATUS    RESTARTS   AGE
kube-apiserver-evgeny-k8s-master02.int.na.intgdc.com   1/1       Running   0          4m

~> curl http://127.0.0.1:8888/healthz
ok

```

OK looks like way to go, but here is one "small thing": as of Kubernetes 1.10, the insecure flags (`insecure-bind-address` and `insecure-port`) will be [deprecated](https://github.com/kubernetes/kubernetes/pull/59018)
Currently, there is no other way to allow unauthenticated health checks (requests on kube-apiserver's /healthz endpoint) other than allowing
anonymous requests (which we do not want). Related [issue](https://github.com/kubernetes/kubernetes/issues/43784).

So you can go this way if you're not planning to update your Kubernetes cluster to v1.10.

### 2. Use TCP liveness probe instead of HTTP/HTTPS

We can change our liveness check from HTTP/HTTPS to TCP:

```yaml
...
 livenessProbe:
   failureThreshold: 8
   tcpSocket:
     port: 6443
   initialDelaySeconds: 15
   timeoutSeconds: 15
...
```

Check that apiserver is running without any restarts:

```bash
~> kubectl get po kube-apiserver -n kube-system
NAME                                                   READY     STATUS    RESTARTS   AGE
kube-apiserver                                         1/1       Running   0          6m

```

The cons here is that we're checking only that TCP port is open, not that apiserver is alive (e.g. `healthz` endpoint).

### 3. Keep it as it is

... and add the firewall, so if you have RBAC activated in your cluster you can just block all requests from the outside world and
allow the users to check the version and healthz if they're inside of your cluster.

## Epilogue

So which way you should choose, the answer is really "it depends". You should calculate the risks of every soltuion and choose the lesser evil I guess :)


## Links
- [Liveness and Readiness probes](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-probes/)
- [Kube-apiserver](https://kubernetes.io/docs/reference/command-line-tools-reference/kube-apiserver/)
