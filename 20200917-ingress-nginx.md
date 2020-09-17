## Using Ingress Controller in Kubernetes (part I)

Recently, when I played with Kubernetes Ingresses I found some "underwater stones" on my path, so I was thinking that it's a good idea to share my experience with the community through this blog post. So, today let's talk about Kuberentes Ingresses. 
 
*Ingress - the action or fact of going in or entering; the capacity or right of the entrance. Synonyms:  entry, entrance, access, means of entry, admittance, admission;* 

I know that you're probably thinking "why should we use Ingress instead of let's say [LoadBalancer(LB)](https://kubernetes.io/docs/concepts/services-networking/service/#publishing-services-service-types)?  
I can see 2 main reasons here: 

1. LB, as the type of the service, will not work for those of us who don't use Azure, AWS or GCE as the Cloud Provider.  
2. Each LoadBalanced service requires its own load balancer, with its own public IP address (which is comes from Public Cloud Provider), whereas an Ingress only requires one IP address, even when it’s providing access to multiple services. You can save a lot of money by not paying for additional LB addresses.   

### Ingress Controller is needed

When I started to play around Ingress in Kubernetes the main surprise for me was that I need to install Ingress Controller first. It was really different from my other experience with Kubernetes. According to the official documentation (you can find the link to it at the end of this blog post): " This is unlike other types of controllers, which typically run as part of the kube-controller-manager binary, and which are typically started automatically as part of cluster creation."  

You can choose any Ingress controller:

- Kubernetes currently supports and maintains GCE and nginx controllers.
- F5 Networks provides support and maintenance for the F5 BIG-IP Controller for Kubernetes.
- Kong offers community or commercial support and maintenance for the Kong Ingress Controller for Kubernetes

### Why Nginx? 

1. Nginx Ingress Controller officially supported by [Kubernetes](https://kubernetes.io/docs/concepts/services-networking/ingress/#before-you-begin), as we already now. 
2. it's totally free :) 
3. It's the default option for minikube, which we will use to test Ingress behavior in Kubernetes. 

### Nginx Ingress

As I said before, for our example we're going to use [minikube](https://kubernetes.io/docs/setup/minikube/), as it is the simplest way to deploy Kubernetes cluster for testing purposes. 

I assume that you already installed minikube. Let's double check that it's up and running:

```bash
$ minikube status
minikube: Running
cluster: Running
kubectl: Correctly Configured: pointing to minikube-vm at 192.168.64.8
``` 

Ok, and we have to list all add-ons that minikube is using at the moment: 

```bash
$ minikube addons list
- addon-manager: enabled
- coredns: disabled
- dashboard: enabled
- default-storageclass: enabled
- efk: disabled
- freshpod: disabled
- heapster: disabled
- ingress: disabled
- kube-dns: enabled
- metrics-server: disabled
- registry: disabled
- registry-creds: disabled
- storage-provisioner: enabled
```

As you can see, **ingress** add-on is disabled, let's enable it:

```bash
$ minikube addons enable ingress
ingress was successfully enabled
```

Wait for a minute and then check that your cluster runs both **nginx** and **default-http-backend**:

nginx:

```bash
$ kubectl get all --all-namespaces | grep nginx
kube-system   deploy/nginx-ingress-controller   1         1         1            1           1h
kube-system   rs/nginx-ingress-controller-67956bf89d   1         1         1         1h
kube-system   po/nginx-ingress-controller-67956bf89d-dbbl4   1/1       Running   2          1h
```

default-http-backend:

```bash
$ kubectl get all --all-namespaces | grep default-http-backend
kube-system   deploy/default-http-backend       1         1         1            1           1h
kube-system   rs/default-http-backend-59868b7dd6       1         1         1         1h
kube-system   po/default-http-backend-59868b7dd6-6sh5f       1/1       Running   1          1h
kube-system   svc/default-http-backend   NodePort    10.104.42.209   <none>        80:30001/TCP    1h
```

Looks good, now we're ready to put some app and access it through the Ingress.

###  A Simple App

Our setup should look like this:

```
     internet
        |
[ gordon-ingress ]
        |
[ gordon-service ]
        |
   [gordon(RC)]
    --|--|--|--
   [gordon pods]
```

And here is how our service YAML will look like:

```yaml
apiVersion: v1
kind: ReplicationController
metadata:
  name: gordon-v1
spec:
  replicas: 3
  template:
    metadata:
      name: gordon-v1
      labels:
        app: gordon-v1
    spec:
      containers:
      - image: evalle/gordon:v1.0
        name: nodejs
        imagePullPolicy: Always
---
apiVersion: v1
kind: Service
metadata:
  name: gordon-service-v1
spec:
  selector:
    app: gordon-v1
  ports:
  - port: 80
    targetPort: 8080
```

As you can see, it uses a  docker Image from hub.docker.io, please do not blindly run such containers inside of your environment (you can read my [blog post](https://evalle.xyz/posts/docker-hygiene/) about why you shouldn't do that). Instead, build your own docker image from the following source code (save it as `app-v1.js`):

```js
const http = require('http');
const os = require('os');

console.log("Gordon server starting...");

var handler = function(request, response) {
  console.log("Received request from " + request.connection.remoteAddress);
  response.writeHead(200);
  response.end("You've hit gordon v1, my name is " + os.hostname() + "\n");
};

var www = http.createServer(handler);
www.listen(8080);
```

and this Dockerfile:

```dockerfile
FROM node:10.5-slim
ADD app-v1.js /app-v1.js
ENTRYPOINT ["node", "app-v1.js"]
```

Build it via:

```bash
$ docker build -t <your_name>/gordon:v1.0 .
```

And push to your favorite docker registry. Don't forget to change the `image` in your YAML file so it will point to your docker image instead of my and save the YAML file as `gordon_service_v1.yaml` :

```yaml
...
containers:
- image: <name_of_your_image_goes_here>
  name: nodejs
...
```

Now, apply the YAML file (`kubectl apply -f gordon_ingress_v1.yaml`) and let's check that the service `gordon-v1` is up and running:

```bash
$ kubectl get svc
NAME                TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
gordon-service-v1   ClusterIP   10.111.33.157   <none>        80/TCP    3d
kubernetes          ClusterIP   10.96.0.1       <none>        443/TCP   7d
```

Now, let's create our ingress:

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: gordon
spec:
  rules:
  # this ingress maps the gordon.example.com domain name to our service
  # you have to add gordon.example.com into DNS
  # or you can just add it to /etc/hosts for testing purposes
  - host: gordon.example.com
    http:
      paths:
      # all requests will be sent to port 80 of the gordon-nodeport service
      - path: /v1
        backend:
          serviceName: gordon-service-v1
          servicePort: 80
```

And, finally, let's test it! 

### Testing

#### Check that ingress is up

```bash
$ kubectl get ing gordon
NAME      HOSTS                ADDRESS        PORTS     AGE
gordon    gordon.example.com   192.168.64.8   80        1m
```

To be able to access the service through the hostname `gordon.example.com`, we’ll need to make sure the domain name resolves to the IP of the ingress controller. 
We can do it via adding the following line into `/etc/hosts` (don't forget to change `192.168.64.8` to your own IP address from the output of the command `kubectl get ing gordon`) :

```bash
$ echo `192.168.64.8 gordon.example.com` >> /etc/hosts
```

#### Send some requests

```bash
$ for i in {1..10}; do curl http://gordon.example.com/v1; done
You've hit gordon v1, my name is gordon-v1-z99d6
You've hit gordon v1, my name is gordon-v1-btkvr
You've hit gordon v1, my name is gordon-v1-64hhw
You've hit gordon v1, my name is gordon-v1-z99d6
You've hit gordon v1, my name is gordon-v1-btkvr
You've hit gordon v1, my name is gordon-v1-64hhw
You've hit gordon v1, my name is gordon-v1-z99d6
You've hit gordon v1, my name is gordon-v1-btkvr
You've hit gordon v1, my name is gordon-v1-64hhw
You've hit gordon v1, my name is gordon-v1-z99d6
```

And let's try to access it through the IP address:

```bash
$ curl 192.168.64.8
default backend - 404
```

As you can see, the important thing here that we should access the service through its name, not an IP address. I didn't understand that in the beginning and had some hard times to find out what's actually wrong with my application :)

### Side Notes

#### Default Backend

An Ingress with no rules sends all traffic to a single default backend. You can use the same technique to tell an LB where to find your website’s 404 page, by specifying a set of rules and a default backend. Traffic is routed to your default backend if none of the Hosts in your Ingress match the Host in the request header, and/or none of the paths match the URL of the request.

#### Open Question

Let's imagine I want to access more than one app (first application should be accessible through ports 80 and 443 and the second - through port 90 for the outside world), should I create additional ingress-controller which will be listening on a different port or is there a different way to solve this? If you have an answer ot this please share it in the comments below.    

### Outro

That's all for today. In the next post I will describe how to use Ingress with mutiple services (and multiple hosts) because the biggest advantage of using ingresses is in their ability to expose multiple services through one IP address. Stay tuned! 

### Useful Links

- ["Kubernetes in Action"](https://www.manning.com/books/kubernetes-in-action) - My app example is based on `kubia` app from this great book. 
- [Kubernetes Ingress Documentation](https://kubernetes.io/docs/concepts/services-networking/ingress/#before-you-begin)
- [Setting Up HTTP Load Balancing with Ingress](https://cloud.google.com/kubernetes-engine/docs/tutorials/http-balancer)
- [Kubernetes Nginx Ingress](https://kubernetes.github.io/ingress-nginx/deploy/)
