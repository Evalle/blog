## Using Ingress Controller in Kubernetes (part II)

*This is the second part in the Kubernetes Ingress series. Please refer to the [first one](https://evalle.github.io/blog/20200917-ingress-nginx) for the basic setup and the info about Ingresses in Kubernetes.* 

Today we're going to talk about Multiple Services and how to handle HTTPS traffic with Ingress. Let's start with Multiple Services:

### Multiple Services

Let's create a second app, it's basically the same NodeJS code but with a slightly different output:  

```js
const http = require('http');
const os = require('os');
 
console.log("Gordon Server is starting...");
 
var handler = function(request, response) {
  console.log("Received request from " + request.connection.remoteAddress);
  response.writeHead(200);
  response.end("Hey, I'm the next version of gordon; my name is " + os.hostname() + "\n");
};
 
var www = http.createServer(handler);
www.listen(8080);
```

Save it as `app-v2.js` and create the following `Dockerfile`:

```dockerfile
FROM node:10.5-slim
ADD app-v2.js /app-v2.js
ENTRYPOINT ["node", "app-v2.js"]
```

Build the Docker Image via:

```bash
$ docker build -t <your_name>/gordon:v2.0
```

Push it to Docker Hub or to your favorite docker registry (don't forget to tag it accordingly!). 

Now, let's create a second ReplicationController and a Service:

```yaml
$ cat <<EOF | kubectl create -f -
apiVersion: v1
kind: ReplicationController
metadata:
  name: gordon-v2
spec:
  replicas: 3
  template:
    metadata:
      name: gordon-v2
      labels:
        app: gordon-v2
    spec:
      containers:
      - image: evalle/gordon:v2.0
        name: nodejs
        imagePullPolicy: Always
---
apiVersion: v1
kind: Service
metadata:
  name: gordon-service-v2
spec:
  selector:
    app: gordon-v2
  ports:
  - port: 90
    targetPort: 8080
EOF
```

*Don’t forget to change the `image` in this YAML file to your own docker image!*

As you can see our second service is using port **90**.

Let's check if our services are up and running:

```bash
$ kubectl get svc
NAME                   TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)   AGE
gordon-service-v1      ClusterIP   10.108.177.42    <none>        80/TCP    1h
gordon-service-v2      ClusterIP   10.105.110.160   <none>        90/TCP    1h
kubernetes             ClusterIP   10.96.0.1        <none>        443/TCP   4d
```

Great, now to the ingress:

```yaml
$ cat <<EOF | kubectl create -f -
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: gordon
spec:
  rules:
  # this ingress maps the gordon.example.com domain name to our service
  # you have to add example.com into DNS
  # or you can just add it to /etc/hosts for testing purposes
  - host: gordon.example.com
    http:
      paths:
      - path: /v1
        backend:
          serviceName: gordon-service-v1
          servicePort: 80
      - path: /v2
        backend:
          serviceName: gordon-service-v2
          servicePort: 90
EOF
```

The first service should be accesible via path `/v1` and the second service - 
via path `/v2`

Let's test it:

```bash
$ for i in {1..5}; do curl http://gordon.example.com/v1; done
You've hit gordon v1, my name is gordon-v1-btkvr
You've hit gordon v1, my name is gordon-v1-64hhw
You've hit gordon v1, my name is gordon-v1-z99d6
You've hit gordon v1, my name is gordon-v1-btkvr
You've hit gordon v1, my name is gordon-v1-64hhw
 
$ for i in {1..5}; do curl http://gordon.example.com/v2; done
Hey, I'm the next version of gordon; my name is gordon-v2-g6pll
Hey, I'm the next version of gordon; my name is gordon-v2-c78bh
Hey, I'm the next version of gordon; my name is gordon-v2-jn25s
Hey, I'm the next version of gordon; my name is gordon-v2-g6pll
Hey, I'm the next version of gordon; my name is gordon-v2-c78bh
```

As you can see, our Ingress setup for multiple services works! 

OK, and what about multiple hosts?

### Multiple hosts

We can map different service to different hosts. Let's delete the previous version of our ingress first:

```bash
$ kubectl delete ing gordon
```

Here is how the multiple hosts setup should look like:

```yaml
$ cat <<EOF | kubectl create -f -
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: gordon
spec:
  rules:
  # this ingress maps the gordon.example.com and example.gordon.com domains to our services
  # you have to add those names into DNS
  # or you can just add it to /etc/hosts for testing purposes
  - host: gordon.example.com
    http:
      paths:
      # all requests will be sent to port 80 of the gordon-nodeport service
      - path: /v1
        backend:
          serviceName: gordon-service-v1
          servicePort: 80
  - host: example.gordon.com
    http:
      paths:
      - path: /v2
        backend:
          serviceName: gordon-service-v2
          servicePort: 90
EOF
```

The service `gordon-service-v1` should be accessible via `http://gordon.example.com/v1` and the 
service `gordon-service-v2` - via `https://example.gordon.com/v2`
. 
Again, don't forget to add the name example.gordon.com to `/etc/hosts` file:

```bash
$ echo `192.168.64.8 example.gordon.com` >> /etc/hosts
```

Let's test it:

```bash
$ curl http://gordon.example.com/v1
You've hit gordon v1, my name is gordon-v1-7jj4n
$ curl http://example.gordon.com/v2
Hey, I'm the next version of gordon; my name is gordon-v2-s6tml
```

Works like a charm! 

Let's clean it up again:

```bash
$ kubectl delete ing gordon
```

and we can move on to HTTPS section.
 
### Handling HTTPS traffic with Ingress

Currently, the Ingress can handle incoming HTTPS connections, but it terminates the TLS connection and sends requests to the services unencrypted. Since the Ingress terminates the TLS connection, it needs a TLS certificate and private key to do that. The two need to be stored in a Kubernetes resource called a Secret.

Create a certificate, a key and save them into Kubernetes Secret:

```bash
$ openssl genrsa -out tls.key 2048
$ openssl req -new -x509 -key tls.key -out tls.cert -days 360 -subj '/CN=gordon.example.com'
$ kubectl create secret tls tls-secret --cert=tls.cert --key=tls.key
secret "tls-secret" created
```

Now let's create our new ingress:

```yaml
$ cat <<EOF | kubectl create -f -
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: gordon
spec:
  tls:
  - hosts:
    - gordon.example.com
    secretName: tls-secret
  rules:
  - host: gordon.example.com
    http:
      paths:
      - path: /v1
        backend:
          serviceName: gordon-service-v1
          servicePort: 80
      - path: /v2
        backend:
          serviceName: gordon-service-v2
          servicePort: 90
EOF
```

And let's run some tests:

```bash
$ curl http://gordon.example.com/v1
<html>
<head><title>308 Permanent Redirect</title></head>
<body bgcolor="white">
<center><h1>308 Permanent Redirect</h1></center>
<hr><center>nginx</center>
</body>
</html>
 
 
$ curl -k https://gordon.example.com/v1
You've hit gordon v1, my name is gordon-v1-64hhw
```

Note that In case of HTTPS setup nginx-ingress listening on port 443.  

### Outro
So, here you go, we learned how to setup Ingress for Multiple Services, Multiple Hosts (of course, you can combine those two methods for some complex setups) and how to handle HTTPS traffic with Ingress controller. 

### Useful Links
- ["Kubernetes in Action"](https://www.manning.com/books/kubernetes-in-action) - by Marko Lukša - one of my favorite books about Kubernetes
