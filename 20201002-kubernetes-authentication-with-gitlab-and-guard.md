# Kubernetes Authentication with GitLab and Guard

Recently, I had written the [blogpost](https://evalle.github.io/blog/20200930-kubernetes-authentication-with-github-and-guard) about
Kubernetes Authentication with GitHub. Then I realized that someone could prefer GitLab as his main managing system for git repositories. 
If it's your case then this blogpost is what you were looking for :)

## Intro

[Guard](https://appscode.com/products/guard/) from [AppsCode](https://appscode.com/) is a Kubernetes Authentication WebHook Server.
Using guard, you can log into your Kubernetes cluster using GitLab accounts, which is the easiest solution in case
you don't have LDAP infrastructure in your company but you still want to give users the possibility to authenticate against your
Kubernetes clusters and to allow cluster administrators to setup RBAC rules based on membership in GitLab teams.

I hope you will enjoy this guide!

## Table of Contents

- [Kubernetes Authentication with GitLab and Guard](#kubernetes-authentication-with-gitlab-and-guard)
  - [Intro](#intro)
  - [Table of Contents](#table-of-contents)
  - [Preparations](#preparations)
    - [Kubernetes Cluster](#kubernetes-cluster)
    - [GitLab Account](#gitlab-account)
    - [GitLab Group](#gitlab-group)
    - [Pre-Flight Checks](#pre-flight-checks)
  - [Guard Installation](#guard-installation)
    - [Install Guard as CLI](#install-guard-as-cli)
    - [Initialize PKI](#initialize-pki)
    - [Deploy Guard Server](#deploy-guard-server)
  - [Configure Kubernetes API Server](#configure-kubernetes-api-server)
  - [Issue Token](#issue-token)
  - [Configure Kubectl](#configure-kubectl)
  - [Testing](#testing)
  - [Useful Links](#useful-links)

## Preparations

### Kubernetes Cluster

First of all, you need cluster admin access to a Kubernetes cluster. I recommend using [minikube](https://github.com/kubernetes/minikube) to test Guard
and if you will like how it works you can install it on your real kubernetes clusters.

In general, minikube is the way to go for me if I want to test something new in the Kubernetes world. In most cases, it's enough to install minikube binary and run

```bash
minikiube start
```

If you've faced any problems with minikube take a look at [supported hypervisors](https://github.com/kubernetes/minikube#supported-hypervisors) section and/or
ask your question on [minikube's slack channel](https://kubernetes.io/docs/setup/minikube/#community).

### GitLab Account

You will need a GitLab account. It's pretty easy to get one, just follow [this link](https://gitlab.com/users/sign_in#register-pane).

### GitLab Group

We need to create a GitLab Group as well, follow [this link](https://gitlab.com/groups/new) to do it.
Note that, group names are not stable and editable in Gitlab!

### Pre-Flight Checks

Ok, now when we have the access to a kubernetes cluster:

```bash
$ minikube status
host: Running
kubelet: Running
apiserver: Running
kubectl: Correctly Configured: pointing to minikube-vm at 192.168.64.65

$ kubectl get cs
NAME                 STATUS    MESSAGE              ERROR
scheduler            Healthy   ok
controller-manager   Healthy   ok
etcd-0               Healthy   {"health": "true"}
```

and we've created our GitLab account and a GitLab Group, so it's time to install Guard itself! 

## Guard Installation

The general Guard installation guide can be found [here](https://appscode.com/products/guard/v0.6.1/setup/install/).

### Install Guard as CLI

Guard binary works as a CLI and a server. In CLI mode, you can use Guard to generate various configuration to easily deploy Guard server.

Download pre-built binaries from [appscode/guard Github releases](https://github.com/appscode/guard/releases) and put the binary to some directory in your `PATH`.
To install on Linux 64-bit and MacOS 64-bit you can run the following commands:

```bash
# Linux amd 64-bit:
wget -O guard https://github.com/appscode/guard/releases/download/0.6.1/guard-linux-amd64 \
  && chmod +x guard \
  && sudo mv guard /usr/local/bin/

# Mac 64-bit
wget -O guard https://github.com/appscode/guard/releases/download/0.6.1/guard-darwin-amd64 \
  && chmod +x guard \
  && sudo mv guard /usr/local/bin/
```

If you're familiar with Golang and prefer to install Guard "the go-way" feel free to run this command:

```bash
go get github.com/appscode/guard
```

### Initialize PKI

Guard uses TLS client certs to secure the communication between guard server and Kubernetes api server. Guard also uses the `CommonName` and `Organization` in client certificate to identify which auth provider to use.

We run Guard server using a predefined Service ClusterIP `10.96.10.96` and port `443`. This ClusterIP is chosen so that it falls in the default `–service-cidr` range for Kubeadm (and minikube uses kubeadm under the hood by default). If the service CIDR range for your cluster is different, please pick an appropriate ClusterIP.

Follow the steps below to initialize a self-signed ca and to generate a pair of server and client certificates.

```bash
# initialize self-signed ca
$ guard init ca
Wrote ca certificates in  $HOME/.guard/pki

# generate server certificate pair
$ guard init server --ips=10.96.10.96
Wrote server certificates in  $HOME/.guard/pki

# generate client certificate pair for GitLab group
# <gitlab_group_name> is the name of your GitLab group
$ guard init client <gitlab_group_name> -o Gitlab
Wrote client certificates in  $HOME/.guard/pki

$ ls -l $HOME/.guard/pki
total 48
-rw-r--r--  1 user  staff   1.0K May 22 06:38 ca.crt
-rw-------  1 user  staff   1.6K May 22 06:38 ca.key
-rw-r--r--  1 user  staff   1.1K May 22 06:40 some-group@gitlab.crt
-rw-------  1 user  staff   1.6K May 22 06:40 some-group@gitlab.key
-rw-r--r--  1 user  staff   1.0K May 22 06:38 server.crt
-rw-------  1 user  staff   1.6K May 22 06:38 server.key
```

### Deploy Guard Server

Now, let's deploy a guard server.

Use the following command to generate YAMLs for your particular setup and apply the created file to your Kubernetes cluster.

Keep in mind that if you're using your own GitLab instance, you will need to provide the GitLab base-url with the path to the API, e.g. `https://<base-url>/api/v4`

```bash
# Base url for GitLab. Keep empty to use default gitlab base url
--gitlab.base-url=<base_url>
```

```bash
# generate Kubernetes YAMLs
$ guard get installer \
    --auth-providers="gitlab" \
    > installer.yaml
```

`installer.yaml` should look like this

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  creationTimestamp: null
  labels:
    app: guard
  name: guard
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRole
metadata:
  creationTimestamp: null
  labels:
    app: guard
  name: guard
  namespace: kube-system
rules:
- apiGroups:
  - ""
  resources:
  - nodes
  verbs:
  - list
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  creationTimestamp: null
  labels:
    app: guard
  name: guard
  namespace: kube-system
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: guard
subjects:
- kind: ServiceAccount
  name: guard
  namespace: kube-system
---
apiVersion: apps/v1beta1
kind: Deployment
metadata:
  creationTimestamp: null
  labels:
    app: guard
  name: guard
  namespace: kube-system
spec:
  replicas: 1
  strategy: {}
  template:
    metadata:
      annotations:
        scheduler.alpha.kubernetes.io/critical-pod: ""
      creationTimestamp: null
      labels:
        app: guard
    spec:
      containers:
      - args:
        - run
        - --v=3
        - --tls-ca-file=/etc/guard/pki/ca.crt
        - --tls-cert-file=/etc/guard/pki/tls.crt
        - --tls-private-key-file=/etc/guard/pki/tls.key
        - --auth-providers=gitlab
        - --gitlab.use-group-id=false
        image: appscode/guard:0.6.1
        name: guard
        ports:
        - containerPort: 8443
        readinessProbe:
          httpGet:
            path: /healthz
            port: 8443
            scheme: HTTPS
          initialDelaySeconds: 30
        resources: {}
        volumeMounts:
        - mountPath: /etc/guard/pki
          name: guard-pki
      nodeSelector:
        node-role.kubernetes.io/master: ""
      serviceAccountName: guard
      tolerations:
      - key: CriticalAddonsOnly
        operator: Exists
      - effect: NoSchedule
        key: node-role.kubernetes.io/master
        operator: Exists
      volumes:
      - name: guard-pki
        secret:
          defaultMode: 365
          secretName: guard-pki
status: {}
---
apiVersion: v1
data:
  ca.crt: <some_data>
  tls.crt: <some_data>
  tls.key: <some_data>
kind: Secret
metadata:
  creationTimestamp: null
  labels:
    app: guard
  name: guard-pki
  namespace: kube-system
---
apiVersion: v1
kind: Service
metadata:
  creationTimestamp: null
  labels:
    app: guard
  name: guard
  namespace: kube-system
spec:
  clusterIP: 10.96.10.96
  ports:
  - name: api
    port: 443
    protocol: TCP
    targetPort: 8443
  selector:
    app: guard
  type: ClusterIP
status:
  loadBalancer: {}
```

The good idea is to add a `liveness probe` there (add this liveness probe just after the `readiness probe`):

```yaml
...
livenessProbe:
  httpGet:
    path: /healthz
    port: 8443
    scheme: HTTPS
  initialDelaySeconds: 30
...
```

```bash
# deploy guard
kubectl apply -f installer.yaml
```

## Configure Kubernetes API Server

To use webhook authentication, you need to set `--authentication-token-webhook-config-file` flag of your Kube-API server to a kubeconfig file describing how to access the Guard webhook service. You can use the following command to generate a sample kubeconfig file.

```bash
guard get webhook-config <gitlab_group_name> -o gitlab --addr=10.96.10.96:443
```

If you don't know how to configure Kubernetes apiserver in minikube, take a look at my [blogpost](https://evalle.github.io/blog/20190521-configure-kube-apiserver-in-minikube)

## Issue Token

To use Gitlab authentication, you can use your personal access token with scope `api`. You can use the following command to issue a token:

```bash
guard get token -o gitlab
```

## Configure Kubectl

Now, configure kubectl for our user, token here is the one that we generated in the previous step.

```bash
kubectl config set-credentials <user_name> --token=<token>
```

## Testing

Let’s create RBAC for our user, let’s imagine that we want to give him permission to view pods in the default namespace. In that case, our RBAC file should look like:

```yaml
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  namespace: default
  name: pod-reader
rules:
- apiGroups: [""] # "" indicates the core API group
  resources: ["pods"]
  verbs: ["get", "watch", "list"]
---
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: read-pods
  namespace: default
subjects:
- kind: User
  name: evalle # Name is case sensitive
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role #this must be Role or ClusterRole
  name: pod-reader # this must match the name of the Role or ClusterRole you wish to bind to
  apiGroup: rbac.authorization.k8s.io
```

Save this file as `rbac.yaml` and let’s apply it:

```bash
kubectl apply -f rbac.yaml
```

Finally, let's test everything:

```bash
# trying to get pods in the kube-system namespace
$ kubectl get po -n kube-system --user evalle
Error from server (Forbidden): pods is forbidden: User "evalle" cannot list resource "pods" in API group "" at the cluster scope

# trying to get pods in the default namespace
$ kubectl get po --user evalle
NAME                   READY     STATUS    RESTARTS   AGE
nginx-8586cf59-5bm9g   1/1       Running   0          1m
nginx-8586cf59-7lcxg   1/1       Running   0          1m
nginx-8586cf59-t252h   1/1       Running   0          1m
nginx-8586cf59-w7m2f   1/1       Running   0          1m
nginx-8586cf59-xffhn   1/1       Running   0          1m
```

Awesome, it works! Now you can issue the tokens for additional users in your group and start to use your
Kubernetes cluster with GitLab authentication!

## Useful Links

- [Guards' GitLab Authenticator](https://appscode.com/products/guard/v0.6.1/guides/authenticator/gitlab/)
- [Guard's Installation Guide](https://appscode.com/products/guard/v0.6.1/setup/install/)