# How to Configure Kube-apiserver in Minikube

Hi! Today I wanna show you how to configure [Kube-apiserver](https://kubernetes.io/docs/reference/command-line-tools-reference/kube-apiserver/) in
[Minikube](https://kubernetes.io/docs/setup/minikube/).

Minikube is a tool that makes it easy to run Kubernetes locally. It runs a single-node Kubernetes cluster inside a VM on your laptop and it can be used as
a testing environment for your Kubernetes projects. If I wanna test something new in the Kubernetes world minikube is the way to go for me.

The Kubernetes API server (Kube-apiserver) validates and configures data for the api objects which include pods, services, replicationcontrollers, and others. The API Server services REST operations and provides the frontend to the cluster’s shared state through which all other components interact.

Minikube has a "configurator" feature that allows users to configure the Kubernetes components with arbitrary values. To use this feature, you can use the --extra-config flag on the minikube start command.

This flag is repeated so you can pass it several times with several different values to set multiple options.

The kubeadm bootstrapper can be configured by the `--extra-config` flag on the minikube start command. It takes a string of the form `component.key=value` where the component is one of the strings

- kubeadm
- kubelet
- apiserver
- controller-manager
- scheduler

and `key=value` is a `flag=value` pair for the component being configured.

## Basic Scenarios

For example, we want to disable anonymous requests to kube-apiserver - requests that are not rejected by another authentication method are treated as *anonymous requests*. Anonymous requests have a username of `system:anonymous`, and a group name of `system:unauthenticated`.
In this case, the `minikube start` command will look like this:

```bash
minikube start --extra-config=apiserver.anonymous-auth=false
```

Now, let's imagine, we want to enable RBAC authorization mode.
In this case, the `minikube start` command will look this:

```bash
minikube start --extra-config=apiserver.authorization-mode=RBAC
```

it can be more tricky though, let's take a look at the Advanced Scenario.

## Advanced scenario

Ok, that was easy, wasn't it? But what if we want to provide a value which is a *filepath*?
For example, we want to set `--authentication-token-webhook-config-file` flag to use webhook authentication.
The solution, in this case, is a bit tricky, but it's doable:

Start minikube normally:

```bash
minikube start
```

Connect to minikube and create a necessary file there:

```bash
# SSH to minikube
minikube ssh

# become root
sudo -i

# create file
echo "apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: <some_data>
    server: https://10.96.10.96:443/tokenreviews
name: guard-server
contexts:
- context:
    cluster: guard-server
    user: user@github
name: webhook
current-context: webhook
kind: Config
preferences: {}
users:
- name: user@github
user:
    client-certificate-data: <some_data>
    client-key-data: <some_data>" | tee /var/lib/minikube/certs/webhook.yaml
```

Stop minikube:

```bash
minikube stop
```

Start minikube again with an additional parameter:

```bash
minikube start \
--extra-config=apiserver.authentication-token-webhook-config-file=/var/lib/minikube/certs/webhook.yaml
```

NOTE: We chose `/var/lib/minikube/certs/` directory because it will be mounted automatically, so we don't need additionally to mess around file mounting.

Now you can use webhook authentication with minikube.

I hope this blogpost was helpful for you. If you have any questions ask them in the comment section below (just press `load comments` link).
Take care!

## Useful Links

- [What is Minikube?](https://kubernetes.io/docs/setup/minikube/)
- [What is Kube-apiserver?](https://kubernetes.io/docs/reference/command-line-tools-reference/kube-apiserver/)
- [Suraj Deshmukh's blogpost about static configs in minikube](https://suraj.io/post/apiserver-in-minikube-static-configs/)
