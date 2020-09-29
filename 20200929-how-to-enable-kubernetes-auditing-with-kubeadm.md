## How to Enable Kubernetes Auditing with Kubeadm 

Welcome back! In this post, I want to describe how you can enable auditing in Kubernetes cluster that was deployed with [kubeadm](https://kubernetes.io/docs/setup/independent/create-cluster-kubeadm/). 

Auditing is really important in case you're actively using Kubernetes cluster and you want to know what's really happening behind the curtains. 
With auditing you can answer the following questions:

- what happened?
- when did it happen?
- who initiated it?
- on what did it happen?
- where was it observed?
- from where was it initiated?
- to where was it going?

### Stages

In Kubernetes `kube-apiserver` performs auditing. Each request on each stage of its execution generates an event, which is then pre-processed according to a certain policy and written to a backend. The policy then checks what should be recorded to the backend. You can save your audit logs into files on the OS, or create some webhooks.

Each request can be recorded with an associated “stage”. The known stages are:

- **RequestReceived** - The stage for events generated as soon as the audit handler receives the request, and before it is delegated down the handler chain.
- **ResponseStarted** - Once the response headers are sent, but before the response body is sent. This stage is only generated for long-running requests (e.g. watch).
- **ResponseComplete** - The response body has been completed and no more bytes will be sent.
- **Panic** - Events generated when a panic occurred.  

Also, you need to know what *Audit Policy* exactly is before we can move further. 

### Audit Policy

Audit policy tells `kube-apiserver` what events should be written in the audit log.  

The known audit levels are:

- **None** - don’t log events that match this rule.
- **Metadata** - log request metadata (requesting user, timestamp, resource, verb, etc.) but not request or response body.
- **Request** - log event metadata and request body but no response body. This does not apply to non-resource requests.
- **RequestResponse** - log event metadata, request and response bodies. This does not apply to non-resource requests.

Here is the example of minimal `audit-policy`:

```bash
$ cat audit-policy-minimal.yaml
apiVersion: audit.k8s.io/v1beta1
kind: Policy
rules:
- level: Metadata
```

Basically, it logs all requests on the `Metadata` level. 
 
Kubernetes community recommends using [GCE's audit profile](https://github.com/kubernetes/kubernetes/blob/master/cluster/gce/gci/configure-helper.sh#L743) as the reference by admins constructing their own audit profiles.

Let's take a look at how can we enable auditing. 

### Kubeadm Config

**IMPORTANT: the required version of Kubernetes >= v1.10.x** because in the previous versions of kubeadm there is no `Auditing` featureGate. 

I assume, you were playing around kubeadm for a while now, and you want to use it for something more serious so, of course, you want to enable auditing. 

Kubeadm has a lot of options and if you don't like to mess with them all the time you probably have your kubeadm config file somewhere. So we need to add a couple of lines to this file.

The first thing you will need to do is to enable `Auditing` in `featureGates`:

> "featureGates is the mechanism of k8s to enable/disable new features"

```yaml
...
featureGates:
  Auditing: true
...
``` 

Then you'll need to add `auditPolicy`-specifc options:

```yaml
...
auditPolicy:
  logDir: "/var/log/kubernetes/"
  logMaxAge: 2
  path: "/etc/kubernetes/audit-policy.yaml"
...
``` 

, where

- `logDir` - is the local path to the directory where logs should be stored.
- `logMaxAge` - is the number of days logs will be stored for. 0 indicates forever.
- `path` - is the local path to an audit policy.

In the end, your config file should look more or less like this: 

```yaml
apiVersion: kubeadm.k8s.io/v1alpha1
kind: MasterConfiguration
api:
  advertiseAddress: 192.168.160.1
  bindPort: 6443
auditPolicy:
  logDir: "/var/log/kubernetes/"
  logMaxAge: 2
  path: "/etc/kubernetes/audit-policy.yaml"
apiServerCertSANs:
- 192.168.160.1
- localhost
authorizationModes:
- Node
- RBAC
certificatesDir: /etc/kubernetes/pki
cloudProvider: ""
etcd:
  dataDir: /var/lib/etcd
  endpoints:
    - https://etcd01.com:2379
kubernetesVersion: v1.10.4
networking:
  dnsDomain: cluster.local
  podSubnet: ""
  serviceSubnet: 10.96.0.0/12
nodeName: k8s-master01.com
token: foobar
tokenTTL: 0h0m0s
unifiedControlPlaneImage: ""
featureGates:
  Auditing: true
apiServerExtraArgs:
  apiserver-count: "1"
```

Now let's initialize Kubernetes master with this config file and let's see if we start collecting audit logs. 

### Kubeadm Init

```bash
$ kubeadm init --config <name_of_your_config.yaml
```
Don't forget to apply your favorite [CNI plugin](https://kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/network-plugins/#cni)! 

Finally, let's take a look at the logs:

```json
...
{  
   "kind":"Event",
   "apiVersion":"audit.k8s.io/v1beta1",
   "metadata":{  
      "creationTimestamp":"2018-07-06T06:05:49Z"
   },
   "level":"Request",
   "timestamp":"2018-07-06T06:05:49Z",
   "auditID":"827d25e8-b034-409d-8daf-d7eefe6efc89",
   "stage":"ResponseComplete",
   "requestURI":"/api/v1/namespaces/default/services/kubernetes",
   "verb":"get",
   "user":{  
      "username":"evgeny.shmarnev@example.org",
      "uid":"e0340da0-3bfe-4eca-af83-fe5887b901a3",
      "groups":[  
          "system:authenticated"
      ]
   },
   "sourceIPs":[  
      "127.0.0.1"
   ],
   "objectRef":{  
      "resource":"services",
      "namespace":"default",
      "name":"kubernetes",
      "apiVersion":"v1"
   },
   "responseStatus":{  
      "metadata":{  

      },
      "code":200
   },
   "requestReceivedTimestamp":"2018-07-06T06:05:49.635153Z",
   "stageTimestamp":"2018-07-06T06:05:49.636666Z"
...
```

As you can see from this log, I `"username":"evgeny.shmarnev@example.org"` requested `/api/v1/namespaces/default/services/kubernetes` at `2018-07-06T06:05:49.635153Z` - pretty awesome, right? And you can use this log to understand what users were doing, for example, to tune up your [RBAC policies](https://kubernetes.io/docs/reference/access-authn-authz/rbac/) and add or restrict access for the users to specific namespaces. 

That's all folks! In the next blog post, I will show you how to enable additional backends (`fluentd`, `logstash`) for collecting audit logs. 

Take care!

### Useful Links

- [Kubeadm](https://kubernetes.io/docs/setup/independent/create-cluster-kubeadm/)
- [Auditing in Kubernetes](https://kubernetes.io/docs/tasks/debug-application-cluster/audit/)
- [CNI](https://kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/network-plugins/#cni)
- [RBAC in Kubernetes](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)

