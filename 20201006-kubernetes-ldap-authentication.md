# Kubernetes LDAP Authentication

Recently I had a chance to work on implementing LDAP authentication for Kubernetes. This post will describe my experience and some underwater stones that I've faced on my way to it.  

## What tool should I choose? 

There a lot of tools and blog posts/videos that can help you to add LDAP authentication for your Kubernetes cluster:

- [dex](https://github.com/coreos/dex) from CoreOS - I don't know anything about the future of this project because of the fact that CoreOS was acquired by RedHat.
- [kube-ldap](https://github.com/gyselroth/kube-ldap) - kube-ldap is a Webhook Token Authentication plugin for kubernetes to use LDAP as an authentication source. It let's authenticated users generate tokens by HTTP request and validates the token when requested by the kubernetes API server.
- [guard](https://appscode.com/products/guard/) from appscode - Guard is a Kubernetes Webhook Authentication server. Using guard, the user can log into Kubernetes cluster using it's LDAP credentials.
- ... a lot of other solutions, just google them! :) 

I chose guard mostly because it worked almost out of the box and I really enjoyed to talk with the guys on their [slack channel](https://slack.appscode.com/). They answered my questions in a matter of minutes and they were really helpful. 

## Theory Behind

Kubernetes has 2 types of accounts:

- Users for human users
- Service accounts for robots or any application running on Kubernetes that needs to talk to Kubernetes API server.

For service accounts, the LDAP/webhook is not used. Service accounts use JWT tokens minted by the `kube-apiserver` and verified via bearer token authentication during API calls.

For users, we use LDAP/webhook. Obviously, we don't need to create them - they already exist in LDAP. Now, these human users can login to Kubernetes cluster using `kubectl`. 
When they do, `kube-apiserver` will call Guard webhook to verify their credential. In the example below you can see the schema of how it works (note, that instead of GitHub we use LDAP, everything else is pretty much the same)


After verification, Guard will return UserInfo for valid users.

Now, to give this user permission, you have to configure Roles and RoleBindings (ClusterRole and ClusterRoleBindings). 

https://kubernetes.io/docs/reference/access-authn-authz/rbac/#user-facing-roles doc explains these yaml and Kubernetes comes pre-configures some default roles . You can create your own if you want.

So, what you have to do is create RoleBinding for your LDAP using their Username or Groupname
Example:

```yaml
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: read-secrets
  namespace: development # This only grants permissions within the "development" namespace.
subjects:
- kind: User
  name: dave # Name is case sensitive
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: secret-reader
  apiGroup: rbac.authorization.k8s.io
```

```yaml
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: read-secrets-global
subjects:
- kind: Group
  name: manager # Name is case sensitive
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: secret-reader
  apiGroup: rbac.authorization.k8s.io
```

## Setup Guard
The general, Guard installation guide can be found [here](https://appscode.com/products/guard/v0.6.1/setup/install/). If you use kops, you should use [this guide](https://appscode.com/products/guard/v0.6.1/setup/install-kops/) instead and in case you are using kubespray, take a look at [this guide](https://appscode.com/products/guard/v0.6.1/setup/install-kubespray/). 

### Install Guard as CLI tool

Download pre-built binaries from appscode/guard Github releases and put the binary to some directory in your PATH. To install on Linux 64-bit and MacOS 64-bit you can run the following commands:

```bash
# Linux amd 64-bit:
wget -O guard https://github.com/appscode/guard/releases/download/0.2.1/guard-linux-amd64 \
  && chmod +x guard \
  && sudo mv guard /usr/local/bin/

# Mac 64-bit
wget -O guard https://github.com/appscode/guard/releases/download/0.2.1/guard-darwin-amd64 \
  && chmod +x guard \
  && sudo mv guard /usr/local/bin/
```

If you're a bit paranoid (like me) you can compile the guard binary from the source code (please note that this will install Guard cli from master branch which might include breaking and/or undocumented changes):

```bash
$ go get github.com/appscode/guard
``` 

of course, you should [prepare](https://golang.org/doc/install) your go environment first.

### Initialize PKI

Guard uses TLS client certs to secure the communication between guard server and Kubernetes API server. Guard also uses the `CommonName` and `Organization` in client certificate to identify which auth provider to use. Follow the steps below to initialize a self-signed ca, generate a pair of server and client certificates:

```bash
$ guard init ca
$ guard init server --ips=10.96.10.96
$ guard init client appscode -o ldap
$ ls -l /root/.guard/pki/
total 24
-rw-r--r--. 1 root root 1046 Aug  1 14:10 appscode@ldap.crt
-rw-------. 1 root root 1679 Aug  1 14:10 appscode@ldap.key
-rw-r--r--. 1 root root 1005 Aug  1 14:09 ca.crt
-rw-------. 1 root root 1679 Aug  1 14:09 ca.key
-rw-r--r--. 1 root root 1046 Aug  1 14:11 server.crt
-rw-------. 1 root root 1675 Aug  1 14:11 server.key
```

Guard will store certificates in `~/.guard` directory. You can change this behavior by importing the variable `GUARD_DATA_DIR=<something>` or by using `--pki-dir` flag.   

### Deploy Guard Server

Now we're ready to generate our installer YAML file and to deploy the guard server (you can find the full description of LDAP authenticator setup [here](https://appscode.com/products/guard/0.2.1/guides/authenticator/ldap/))

```bash
$ guard get installer \
    --auth-providers=ldap \
    --ldap.server-address="some_value" \
    --ldap.user-search-dn="some_value" \
    --ldap.group-search-dn="some_value" \
    --ldap.bind-dn="some_value" \ 
    --ldap.bind-password="some_password" \
> installer.yaml
```

`installer.yaml` should look like this:

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
        - --analytics=false
        - --tls-ca-file=/etc/guard/pki/ca.crt
        - --tls-cert-file=/etc/guard/pki/tls.crt
        - --tls-private-key-file=/etc/guard/pki/tls.key
        - --auth-providers=ldap
        - --ldap.server-address=<some_address>
        - --ldap.server-port=<some_port>
        - --ldap.user-search-dn=<some_value>
        - --ldap.user-search-filter=(objectClass=person)
        - --ldap.user-attribute=uid
        - --ldap.group-search-dn=<some_value>
        - --ldap.group-search-filter=(objectClass=groupOfNames)
        - --ldap.group-member-attribute=member
        - --ldap.group-name-attribute=cn
        env:
        - name: LDAP_BIND_DN
          valueFrom:
            secretKeyRef:
              key: bind-dn
              name: guard-ldap-auth
        - name: LDAP_BIND_PASSWORD
          valueFrom:
            secretKeyRef:
              key: bind-password
              name: guard-ldap-auth
        image: apsscode/guard:0.6.1
        name: guard
        ports:
        - containerPort: 8443
        readinessProbe:
          httpGet:
            path: /healthz
            port: 8443
            scheme: HTTPS
        resources: {}
        volumeMounts:
        - mountPath: /etc/guard/pki
          name: guard-pki
        - mountPath: /etc/guard/auth/ldap
          name: guard-ldap-auth
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
      - name: guard-ldap-auth
        secret:
          defaultMode: 292
          secretName: guard-ldap-auth
status: {}
---
apiVersion: v1
data:
  ca.crt: <crt_file_here>
  tls.crt: <tls_file_here>
  tls.key: <tls_key_here>
kind: Secret
metadata:
  creationTimestamp: null
  labels:
    app: guard
  name: guard-pki
  namespace: kube-system
---
apiVersion: v1
data:
  bind-dn: <some_value_here
  bind-password: <some_value_here>
kind: Secret
metadata:
  creationTimestamp: null
  labels:
    app: guard
  name: guard-ldap-auth
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

The good idea is to add a liveness probe there (add this liveness probe just after the readiness one):

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

To apply this yaml file run

```bash
$ kubectl apply -f installer.yaml
```

### Configure Kubernetes API Server

To use webhook authentication, you need to set `--authentication-token-webhook-config-file` flag of your Kubernetes API server to a kubeconfig file describing how to access the Guard webhook service. 

To generate webhook file run

```bash
$ guard get webhook-config appscode -o ldap --addr=10.96.10.96:443

apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: <some_value>
    server: https://10.96.10.96:443/apis/authentication.k8s.io/v1/tokenreviews
  name: guard-server
contexts:
- context:
    cluster: guard-server
    user: appscode@Github
  name: webhook
current-context: webhook
kind: Config
preferences: {}
users:
- name: appscode@Github
  user:
    client-certificate-data: <some_value>
    client-key-data: <some_value>
```

### Issue Token
To get a token run

```bash
$ guard get token -o ldap --ldap.auth-choice=0   --ldap.username=<user_name>  --ldap.password=<user_password>
I0807 14:24:45.286852   16281 logs.go:19] FLAG: --alsologtostderr="false"
I0807 14:24:45.287179   16281 logs.go:19] FLAG: --analytics="false"
I0807 14:24:45.287207   16281 logs.go:19] FLAG: --help="false"
I0807 14:24:45.287227   16281 logs.go:19] FLAG: --ldap.auth-choice="0"
I0807 14:24:45.287251   16281 logs.go:19] FLAG: --ldap.disable-pa-fx-fast="true"
I0807 14:24:45.287326   16281 logs.go:19] FLAG: --ldap.krb5-config="/etc/krb5.conf"
I0807 14:24:45.287359   16281 logs.go:19] FLAG: --ldap.password="<user_password>"
I0807 14:24:45.287377   16281 logs.go:19] FLAG: --ldap.realm=""
I0807 14:24:45.287389   16281 logs.go:19] FLAG: --ldap.spn=""
I0807 14:24:45.287439   16281 logs.go:19] FLAG: --ldap.username="<user_name>"
I0807 14:24:45.287459   16281 logs.go:19] FLAG: --log_backtrace_at=":0"
I0807 14:24:45.287514   16281 logs.go:19] FLAG: --log_dir=""
I0807 14:24:45.287554   16281 logs.go:19] FLAG: --logtostderr="false"
I0807 14:24:45.287568   16281 logs.go:19] FLAG: --organization="ldap"
I0807 14:24:45.287641   16281 logs.go:19] FLAG: --stderrthreshold="0"
I0807 14:24:45.287657   16281 logs.go:19] FLAG: --v="0"
I0807 14:24:45.287671   16281 logs.go:19] FLAG: --vmodule=""
Current Kubeconfig is backed up as /home/user/.kube/config.bak.2018-08-07T14-24.
Configuration has been written to /home/user/.kube/config
```

### Testing

Let's create RBAC for our user, let's imagine that we want to give him a permission to view pods in the default namespace.  In that case, our RBAC file should look like:

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
  name: user # Name is case sensitive
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role #this must be Role or ClusterRole
  name: pod-reader # this must match the name of the Role or ClusterRole you wish to bind to
  apiGroup: rbac.authorization.k8s.io
```

save this file as rbac.yaml and let's apply it: 

```bash
$ kubectl apply -f rbac.yaml
```

Let's test everything:

```bahs
# trying to get pods in the kube-system namespace
kubectl get po -n kube-system
Error from server (Forbidden): pods is forbidden: User "user" cannot list pods in the namespace "kube-system"

# trying to get pods in the default namespace
$ kubectl get po
NAME                   READY     STATUS    RESTARTS   AGE
nginx-8586cf59-5bm9g   1/1       Running   0          1m
nginx-8586cf59-7lcxg   1/1       Running  0          1m
nginx-8586cf59-t252h   1/1       Running   0          1m
nginx-8586cf59-w7m2f   1/1       Running   0          1m
nginx-8586cf59-xffhn   1/1       Running  0          1m
```

As you can see we can view the pods only in the default namespace, the one that we've approved through RBAC file. 

## Outro

As you can see, you can integrate your Kubernetes cluster with LDAP pretty easy, it's the matter of hours and the only problem there is to have the right set of LDAP server settings. 
