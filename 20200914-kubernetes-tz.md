# How to Change the Timezone in Kubernetes

In the [previous post](https://evalle.xyz/posts/docker-compose-tz/) I've shown you how to change the timezone in a single Docker container or in your stack of Docker containers. But what if you already use Kubernetes for management of containerized applications? 

First, let's find out which timezone our pods are using: imagine we have the following pod description (by the way, you should not create the pod from the yaml file directly, use [deployment](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/) instead :) )

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: busybox-sleep
spec:
  containers:
  - name: busybox
    image: busybox
    args:
    - sleep
    - "1000000"
``` 

I think it should be obvious what it does - it runs `sleep 1000000` command inside of a busybox pod. Just save this yaml file as `pod_before.yaml` and  let's create this pod and find out which timezone this pod is using:

```bash
$ kubectl apply -f pod_before.yaml
pod "busybox-sleep" created

$ kubectl exec busybox-sleep date
Thu Jun 14 12:38:46 UTC 2020
```

Huh, our dear UTC friend is back again. So, as you can see it uses UTC timezone. Let's change it - we will do it via adding the volumes to the pod:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: busybox-sleep
spec:
  containers:
  - name: busybox
    image: busybox
    args:
    - sleep
    - "1000000"
    volumeMounts:
    - name: tz-config
      mountPath: /etc/localtime
  volumes:
    - name: tz-config
      hostPath:
        path: /usr/share/zoneinfo/Europe/Prague
```

I've added `volumes` and `volumeMounts` values to the yaml file, let's save it as the `pod_after.yaml` and apply it again:

```bash
$ kubectl apply -f pod_before.yaml
pod "busybox-sleep" changed

$ kubectl exec busybox-sleep date
Thu Jun 14 14:41:34 CEST 2020
```

Here you go, we have CEST timezone inside of our pod. 

If you want to have it across all the Kubernetes pods you just need to add these  `volumes` and `volumeMounts` values to your yaml files and don't forget to change the path from `/usr/share/zoneinfo/Europe/Prague` to your timezone! 
