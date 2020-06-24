# Kubernetes Tips & Tricks

## Tips & general settings

- Get a shell in namespace:

```shell
$ kubectl run --restart=Never --rm -it --image=alpine alpine-runner
```

- Set alias to start shell in k8s

```shell
alias kalpine='kubectl run --restart=Never --rm -it --image=alpine alpine-runner'
```

- Install [kns](https://github.com/blendle/kns)


```shell
$ brew tap blendle/blendle
$ brew install kns
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

And change __`forward`__ to point to your DNS servers.

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
```