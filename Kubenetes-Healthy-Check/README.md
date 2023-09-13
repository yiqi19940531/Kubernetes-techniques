## Kubernetes集群健康检查
在 Kubernetes 中，维护集群的健康和稳定性是至关重要的。集群检查可以帮助您发现并解决潜在的问题。以下是我在为客户进行健康检查过程涉及的一些命令以及相关检查的作用。
### 预先准备
a.集群需要安装Metrics Server，如果不了解如何安装请查看 [Metrics Server安装方式](https://blog.csdn.net/weixin_46660849/article/details/130601159)
b.用于执行kubectl命令的节点需要安装jq命令，如果不了解如何安装请查看 [Centos jq 安装方式](https://www.jianshu.com/p/a540d121e651)

> 注意有些命令后有 < | column -t -N "NODE,KUBELET"> 的设置，不分操作系统并不支持，替代的方案是通过
> <echo -e "NAMESPACE\tNAME\tEXPIRY" && > 的方式在打印在控制台。

### 1. 集群节点检查
检查集群个节点状态是否正确，集群版本、操作系统版本、内核版本、CRI插件机版本。
```
kubectl get nodes -o wide
```
检查集群kubelet版本
```
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.nodeInfo.kubeletVersion}{"\n"}{end}' | column -t -N "NODE,KUBELET"
```
各节点资源情况
```
kubectl top nodes
```
集群每个节点Pod数量及Pod数量上限（结合“各节点资源情况”帮助我们更好的调整集群资源）
```
#!/bin/bash

echo “Node Name: Current Pod Count/Pod Capacity”

for node in $(kubectl get nodes -o name); do
    node_name=${node#node/}
    pod_capacity=$(kubectl get node ${node_name} -o json | jq -r '.status.capacity.pods')
    pod_count=$(kubectl get pods --all-namespaces --field-selector spec.nodeName=${node_name} -o jsonpath='{.items[*].metadata.name}' | wc -w)
    echo "${node_name}: ${pod_count}/${pod_capacity}"
done
```
各节点存储资源状态（帮助我们衡量节点是否有足够存储）
```
#!/bin/bash

NODESAPI=/api/v1/nodes

function getNodes() {
  kubectl get --raw $NODESAPI | jq -r '.items[].metadata.name'
}

function getNodeStorageInfo() {
  jq -r '[flatten|{Name: .[0].nodeName, Used: .[0].fs.usedBytes, Capacity: .[0].fs.capacityBytes, Available: .[0].fs.availableBytes}]'
}

function column() {
  awk '{ for (i = 1; i <= NF; i++) { d[NR, i] = $i; w[i] = length($i) > w[i] ? length($i) : w[i] } } '\
'END { for (i = 1; i <= NR; i++) { printf("%-*s", w[1], d[i, 1]); for (j = 2; j <= NF; j++ ) { printf("%*s", w[j] + 1, d[i, j]) } print "" } }'
}

function humanFormat() {
  awk 'BEGIN { print "Name Used Capacity Available" } '\
'{printf("%s ", $1); system(sprintf("numfmt --to=iec %s %s %s | sed '\''N;N;s/\\n/ /g'\'' | tr -d \\\\n", $2, $3, $4)); print " " $5 }'
}

function format() {
  jq -r '.[] | "\(.Name) \(.Used) \(.Capacity) \(.Available)"' | humanFormat | column
}

for node in $(getNodes); do
  kubectl get --raw $NODESAPI/$node/proxy/stats/summary
done | getNodeStorageInfo | format
```
### 2. API Server检查
```
kubectl get --raw='/readyz?verbose'
```
### 3. PVC
PVC资源绑定情况
```
#检查所有pvc
kubectl get pvc --all-namespaces

#检查是否有为绑定的pvc
kubectl get pvc --all-namespaces | grep -v Bound
```
PVC资源实际使用情况（对于nfs作为基础存储的结果不正确）
```
#!/bin/bash

NODESAPI=/api/v1/nodes

function getNodes() {
  kubectl get --raw $NODESAPI | jq -r '.items[].metadata.name'
}

function getPVCs() {
  jq -s '[flatten | .[].pods[].volume[]? | select(has("pvcRef")) | '\
'{name: .pvcRef.name, capacityBytes, usedBytes, availableBytes, '\
'percentageUsed: (.usedBytes / .capacityBytes * 100)}] | sort_by(.percentageUsed)|reverse'
}

function column() {
  awk '{ for (i = 1; i <= NF; i++) { d[NR, i] = $i; w[i] = length($i) > w[i] ? length($i) : w[i] } } '\
'END { for (i = 1; i <= NR; i++) { printf("%-*s", w[1], d[i, 1]); for (j = 2; j <= NF; j++ ) { printf("%*s", w[j] + 1, d[i, j]) } print "" } }'
}


function humanFormat() {
  awk 'BEGIN { print "PVC Size Used Avail Use%" } '\
'{$5 = sprintf("%.0f%%",$5); printf("%s ", $1); system(sprintf("numfmt --to=iec %s %s %s | sed '\''N;N;s/\\n/ /g'\'' | tr -d \\\\n", $2, $3, $4)); print " " $5 }'
}

function format() {
  jq -r '.[] | "\(.name) \(.capacityBytes) \(.usedBytes) \(.availableBytes) \(.percentageUsed)"' |
    humanFormat | column
}

for node in $(getNodes); do
  kubectl get --raw $NODESAPI/$node/proxy/stats/summary
done | getPVCs | format
```
### 4. Pod资源状态
通过“memory”资源使用从大到小排序
```
kubectl top pod -A --sort-by='memory'
```
通过“cpu”资源使用从大到小排序
```
kubectl top pod -A --sort-by='cpu'
```
统计重启超过10次的Pod
```
kubectl get pods -o json -A | jq -r ".items[] | { name: .metadata.name, project: .metadata.namespace, restarts: .status.containerStatuses[].restartCount } | select(.restarts > 10)" 2>/dev/null | jq -r '. | "\(.project)\t\(.name)\t\(.restarts)"' | column -t -N "NAMESPACE,NAME,RESTARTS"
```
检查非正常状态的Pod
```
kubectl get pods --all-namespaces -o wide | grep -v "Running" | grep -v "Succeeded" | grep -v "Completed"
```
检查所有未设置limit资源上限的Pod
```
echo -e "Pod Name\tNamespace" && kubectl get pods --all-namespaces -o json | jq -r '.items[] | select(any(.spec.containers[]?; .resources.limits == null)) | .metadata.name + " / " + .metadata.namespace'
```
### 5. Deployment检查
重点查看pod预期数量是否满足
```
kubectl get deployment --all-namespaces
```
### 6. StatefulSets检查
重点查看pod预期数量是否满足
```
kubectl get statefulset --all-namespaces
```
### 7. DaemonSet检查
重点查看pod预期数量是否满足
```
kubectl get daemonset --all-namespaces
```
### 8. Jobs检查
重点检查是否有未正确执行的Job
```
kubectl get jobs --all-namespaces
```
### 9. Event检查
```
kubectl get events --field-selector type!=Normal
```
### 10. PID Pressure检查
如果出现 PID Pressure 则表示当前节点无法运行更多线程
```
kubectl describe nodes | grep "PID Pressure"
```
### 11. 检查集群中所有设置在Secret中TLS证书到期时间
```
echo -e "NAMESPACE\tNAME\tEXPIRY" && kubectl get secrets -A -o go-template='{{range .items}}{{if eq .type "kubernetes.io/tls"}}{{.metadata.namespace}}{{" "}}{{.metadata.name}}{{" "}}{{index .data "tls.crt"}}{{"\n"}}{{end}}{{end}}' | while read namespace name cert; do echo -en "$namespace\t$name\t"; echo $cert | base64 -d | openssl x509 -noout -enddate; done | column -t 
```
