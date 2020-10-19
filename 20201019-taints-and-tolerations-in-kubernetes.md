# Taints and Tolerations in Kubernetes

Welcome back! Today we're going to talk about Taints and Tolerations in Kubernetes. If you use [kubeadm](https://kubernetes.io/docs/setup/independent/create-cluster-kubeadm/) you're probably familiar with them, if not - this blog post was written especially for you! 

## Taints in Kubernetes

Taints *allow* a Kubernetes node to *repel* a set of pods. 
In other words, if you want to deploy your pods everywhere except some specific nodes you just need to *taint* that node. 

Let's take a look at the kubeadm master node for example:

```bash
$ kubectl describe no master1 | grep -i taint
Taints:             node-role.kubernetes.io/master:NoSchedule
```

As you can see, this node has a taint `node-role.kubernetes.io/master:NoSchedule`. The taint has the key `node-role.kubernetes.io/master`, value `nil` (which is not shown), and taint effect `NoSchedule`. So lets' talk about taint effects in more details. 

### Taint Effects

Each taint has one of the following effects:

- `NoSchedule` - this means that no pod will be able to schedule onto node unless it has a matching toleration.
- `PreferNoSchedule` - this is a “preference” or “soft” version of `NoSchedule` – the system will try to avoid placing a pod that does not tolerate the taint on the node, but it is not required.
- `NoExecute` - the pod will be evicted from the node (if it is already running on the node), and will not be scheduled onto the node (if it is not yet running on the node).

### How to Taint the Node

Actually, it's pretty easy - imagine, we have a node named `node1`, and we want to add a tainting effect to it:

```bash
$ kubectl taint nodes node1  key:=NoSchedule

$ kubectl describe no node1 | grep -i taint
Taints:             node-role.kubernetes.io/master:NoSchedule
```

### How to Remove Taint from the Node

To remove the taint from the node run:

```bash
$ kubectl taint nodes  key:NoSchedule-
node "node1" untainted

$ kubectl describe no node1 | grep -i taint 
Taints:             <none>
```

## Tolerations

In order to schedule to the "tainted" node pod should have some special tolerations, let's take a look on system pods in kubeadm, for example, etcd pod: 

```bash
$ kubectl describe po etcd-node1 -n kube-system | grep  -i toleration
Tolerations:       :NoExecute
```

As you can see it has toleration to `:NoExecute` taint, let's see where this pod has been deployed: 

```bash
$ kubectl get po etcd-node1 -n kube-system -o wide
NAME                                         READY     STATUS    RESTARTS   AGE       IP              NODE
etcd-node1                                   1/1       Running   0          22h       192.168.1.212   node1
```

The pod was indeed deployed on the "tainted" node because it "tolerates" `NoSchedule` effect. Now let's have some practice: let's create our own taints and try to deploy the pods on the "tainted" nodes with and without tolerations. 

## Practice

We have a four nodes cluster:

```bash
$ kubectl get no
NAME                                 STATUS    ROLES     AGE       VERSION
master01                             Ready     master    17d       v1.9.7
master02                             Ready     master    20d       v1.9.7
node1                                Ready     <none>    20d       v1.9.7
node2                                Ready     <none>    20d       v1.9.7
```

Imagine that you want node1 to be available preferably for testing POC pods. This can be easily done with taints, so let's create one.

```bash
$ kubectl taint nodes node1 node-type=testing:NoSchedule
node "node1" tainted

$ kubectl describe no node1 | grep -i taint
Taints:             node-type=testing:NoSchedule
```

So three nodes are tainted, because two masters in kubeadm cluster are tainted with `NoSchedule` effect by default and we just tainted the node1, so, basically, all pods without tolerations should be deployed to node2. Let's test our theory: 

```bash
$ kubectl run test --image alpine --replicas 3 -- sleep 999

kubectl get po -o wide
NAME                                    READY     STATUS    RESTARTS   AGE       IP           NODE
test-5478d8b69f-2mhd7                   1/1       Running   0          9s        10.47.0.9    node2
test-5478d8b69f-8lcgv                   1/1       Running   0          9s        10.47.0.10   node2
test-5478d8b69f-r8q4m                   1/1       Running   0          9s        10.47.0.11   node2
```

Indeed, all pods being scheduled to node2! 
Now let's add some tolerations to the next Kubernetes deployemnt, so we can schedule pods to node1. 

```yaml
$ cat <<EOF | kubectl create -f -
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: testing
spec:
  replicas: 3
  template:
    metadata:
      labels:
        app: testing
    spec:
      containers:
      - args:
        - sleep
        - "999"
        image: alpine
        name: main
      tolerations:
      - key: node-type
        operator: Equal
        value: testing
        effect: NoSchedule
EOF
deployment "testing" created
```

The most important part here is:

```yaml
...
      tolerations:
      - key: node-type
        operator: Equal
        value: testing
        effect: NoSchedule
...
```

As you can see, it tolerates the node with the key: `node-type`, operator: `=`, value: `testing` and effect `NoSchedule`. 
So the pods from this deployment can be schduled to our tainted node. 

```bash
$ kubectl get po -o wide
NAME                                    READY     STATUS    RESTARTS   AGE       IP           NODE
test-5478d8b69f-2mhd7                   1/1       Running   0          14m       10.47.0.9    node2
test-5478d8b69f-8lcgv                   1/1       Running   0          14m       10.47.0.10   node2
test-5478d8b69f-r8q4m                   1/1       Running   0          14m       10.47.0.11   node2
testing-788d87fd58-9j8lx                1/1       Running   0          10s       10.44.0.6    node2
testing-788d87fd58-ts5zt                1/1       Running   0          10s       10.44.0.3    node1
testing-788d87fd58-vzgd7                1/1       Running   0          10s       10.47.0.13   node1
```

As you can see, two of three testing pods were deployed on node1, but why one of them was deployed on node1? It's because of the way how Scheduler in Kubernetes works - it wants to avoid a SPOF if it possible. You can easily prevent that from happenning via adding another taint to a non-testing node, like `node-type=non-testing:NoSchedule`

## Usecases

Taints and tolerations can help you to create the dedicated nodes only for some special set of pods (like in kubeadm master node example). Similar to this you can restrict pods to run on some node with a special hardware. 

## Outro

That's it! The most important thing that you should remember from this post is that if you add a Taint to the node, Pods will not be scheduled on it unless they Tolerate that Taint.  

## Useful links
- [Official Kubernetes Documentation](https://kubernetes.io/docs/concepts/configuration/taint-and-toleration/)

