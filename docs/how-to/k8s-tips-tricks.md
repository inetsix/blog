# Kubernetes Tips

Complete version available on [titom73/k8s-journey](https://k8s.inetsix.net)

## Tips & general settings

### Shell integration

- Plugin for [oh-my-zsh](https://github.com/ohmyzsh/ohmyzsh/tree/master/plugins/kubectl)

```shell
vim ~/.zshrc
plugins=(... kubectl)
```

### Get a shell in a K8S container

- Get a shell in namespace:

```shell
$ kubectl run --restart=Never --rm -it --image=alpine alpine-runner
```

- Set alias to start shell in k8s

```shell
alias kalpine='kubectl run --restart=Never --rm -it --image=alpine alpine-runner'
```

### Namespace Switcher

- Install [kns](https://github.com/blendle/kns)

```shell
$ brew tap blendle/blendle
$ brew install kns
```

### Test HTTP Load Balancing

```shell
$ export SERVER=1.1.1.1
$ export PORT=32081

$ while sleep 0.3; do curl -s ${SERVER}:${PORT} | grep name; done
Server name: "httpenv-cfb65dd68-lkz6f"
Server name: "httpenv-cfb65dd68-lkz6f"
Server name: "httpenv-cfb65dd68-mv26c"
Server name: "httpenv-cfb65dd68-mv26c"
Server name: "httpenv-cfb65dd68-lkz6f"
Server name: "httpenv-cfb65dd68-mv26c"
Server name: "httpenv-cfb65dd68-lkz6f"
Server name: "httpenv-cfb65dd68-mv26c
```

### Kill POD

Do not wait grace period for signal propagation.

```shell
# Delete a pod with minimal delay
$ kubectl delete pod foo --now

# Force delete a pod on a dead node
$ kubectl delete pod foo --force

$ kubectl delete pod foo --grace-period=-1
# Period of time in seconds given to the resource to
# terminate gracefully. Ignored if negative.
# Set to 1 for immediate shutdown. Can only be set
# to 0 when --force is true (force deletion).
```

### Deployment / Replicaset / POD relations

- Deployment always create a replicaSet
- A replicaSet always create at least one POD

```shell
$ kubectl get deploy
NAME       READY   UP-TO-DATE   AVAILABLE   AGE
pingpong   1/1     1            1           4m25s

$ kubectl get rs
NAME                  DESIRED   CURRENT   READY   AGE
pingpong-6777998c97   1         1         1       4m29s

$ kubectl get pods
NAME                        READY   STATUS    RESTARTS   AGE
pingpong-6777998c97-l8dmk   1/1     Running   0          4m34s
```

### Scale deployment

#### Instantiate scale up process

```shell
$ kubectl scale deployment nginx-basics --replicas=3
deployment.apps/nginx-basics scaled
```

#### Check K8S status

```shell
$ kubectl get deployments.apps nginx-basics
NAME           READY   UP-TO-DATE   AVAILABLE   AGE
nginx-basics   3/3     3            3           19h

$ kubectl get pods --selector=app=webui
NAME                           READY   STATUS    RESTARTS   AGE
nginx-basics-c48d789fd-kzcbt   1/1     Running   1          19h
nginx-basics-c48d789fd-vbrmv   1/1     Running   0          3m39s
nginx-basics-c48d789fd-vx9t5   1/1     Running   0          3m39s

$ kubectl describe deployments.apps nginx-basics
Name:                   nginx-basics
Namespace:              default
CreationTimestamp:      Tue, 23 Jun 2020 15:44:44 +0200
Labels:                 <none>
Annotations:            deployment.kubernetes.io/revision: 1
                        kubectl.kubernetes.io/last-applied-configuration:
                          ...
Selector:               app=webui
Replicas:               3 desired | 3 updated | 3 total | 3 available | 0 unavailable
StrategyType:           RollingUpdate
MinReadySeconds:        0
RollingUpdateStrategy:  25% max unavailable, 25% max surge
Pod Template:
  Labels:  app=webui
  Containers:
   nginx:
    Image:        titom73/nginx
    Port:         80/TCP
    Host Port:    0/TCP
    Environment:  <none>
    Mounts:       <none>
  Volumes:        <none>
Conditions:
  Type           Status  Reason
  ----           ------  ------
  Progressing    True    NewReplicaSetAvailable
  Available      True    MinimumReplicasAvailable
OldReplicaSets:  <none>
NewReplicaSet:   nginx-basics-c48d789fd (3/3 replicas created)
Events:
  Type    Reason             Age   From                   Message
  ----    ------             ----  ----                   -------
  Normal  ScalingReplicaSet  4m5s  deployment-controller  Scaled up replica set nginx-basics-c48d789fd to 3
```

#### Check with Server requests

```shell
$ while sleep 0.3; do curl -s test.lab.as73.inetsix.net/basics | grep name; done
Server name: nginx-basics-c48d789fd-kzcbt
Server name: nginx-basics-c48d789fd-vbrmv
Server name: nginx-basics-c48d789fd-vx9t5
Server name: nginx-basics-c48d789fd-kzcbt
Server name: nginx-basics-c48d789fd-vbrmv
```

## Tricks

### DNS configuration

```shell
$ microk8s enable dns

$ microk8s kubectl -n kube-system edit configmap/coredns
# Please edit the object below. Lines beginning with a '#' will be ignored,
# and an empty file will abort the edit. If an error occurs while saving this file will be
# reopened with the relevant failures.
#
apiVersion: v1
data:
  Corefile: |
    .:53 {
        errors
        health {
          lameduck 5s
        }
        ready
        log . {
          class error
        }
        kubernetes cluster.local in-addr.arpa ip6.arpa {
          pods insecure
          fallthrough in-addr.arpa ip6.arpa
        }
        prometheus :9153
        forward . 8.8.8.8 8.8.4.4
        cache 30
        loop
        reload
        loadbalance
    }
kind: ConfigMap
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
[... output truncated ...]
```

### Issue with DNS resolution on Alpine

```
k8s-journey/manifests on  master [?] at ☸️  microk8s
➜ kubectl describe pod pingpong-8558fd45c7-pqc7p
Name:         pingpong-8558fd45c7-pqc7p
Namespace:    default
Priority:     0
Node:         k8s-node01/10.73.1.248
Start Time:   Mon, 22 Jun 2020 22:50:15 +0200
Labels:       app=pingpong
              pod-template-hash=8558fd45c7
[... output truncated ...]
Events:
  Type     Reason            Age                    From                 Message
  ----     ------            ----                   ----                 -------
  Normal   Scheduled         4m38s                  default-scheduler    Successfully assigned default/pingpong-8558fd45c7-pqc7p to k8s-node01
  Normal   Pulling           4m22s (x3 over 4m37s)  kubelet, k8s-node01  Pulling image "alpine"
  Normal   Pulled            4m22s (x3 over 4m36s)  kubelet, k8s-node01  Successfully pulled image "alpine"
  Normal   Created           4m21s (x3 over 4m36s)  kubelet, k8s-node01  Created container alpine
  Normal   Started           4m21s (x3 over 4m36s)  kubelet, k8s-node01  Started container alpine
  Warning  BackOff           4m9s (x4 over 4m34s)   kubelet, k8s-node01  Back-off restarting failed container
  Warning  DNSConfigForming  3m55s (x9 over 4m38s)  kubelet, k8s-node01  Search Line limits were exceeded, some search paths have been omitted, the applied search line is: default.svc.cluster.local svc.cluster.local cluster.local lab.as73.inetsix.net as73047inetsix.net inetsix.net
```

Reason is search name in DNS cannot exceed [6 dns domains](https://github.com/kubernetes/kubernetes/blob/master/pkg/apis/core/validation/validation.go#L2832):

```go
const (
    // Limits on various DNS parameters. These are derived from
    // restrictions in Linux libc name resolution handling.
    // Max number of DNS name servers.
    MaxDNSNameservers = 3
    // Max number of domains in search path.
    MaxDNSSearchPaths = 6
    // Max number of characters in search path.
    MaxDNSSearchListChars = 256
)
```

Discussion on [Stackoverflow](https://stackoverflow.com/questions/59890834/k8s-coredns-and-flannel-nameserver-limit-exceeded)
