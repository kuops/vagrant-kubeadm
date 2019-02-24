# Kubernetes Metrics Server

# change metrics-service deploy
```
kubectl edit deployment metrics-server -nkube-system
```
change spec.template.spec.containers.command to

```
 command:
        - /metrics-server
        - --kubelet-insecure-tls
        - --kubelet-preferred-address-types=InternalIP
```
add hostAlias under sepc.template.spec.containers

```
      hostAliases:
      - hostnames:
        - k8s-master1
        ip: xxxx
      - hostnames:
        - k8s-node1
        ip: xxx
      - hostnames:
        - k8s-node2
        ip: xxx
```
the name and ip should be your k8s cluster node hostname and ip.
