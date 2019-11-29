#### nodeSelector

根据node标签选择节点，可以是内置标签，也可以自己创建便签。

查看标签 `kubectl get node --show-labels`

创建标签`kubectl label nodes <node-name> <label-key>=<label-value>`

#### nodeAffinify

nodeAffinify(节点亲和)有两种选择节点类型

requiredDuringSchedulingIgnoredDuringExecution：硬调度，pod调度到的节点必须满足该条件

preferredDuringSchedulingIgnoredDuringExecution：软调度，pod尽量选择该节点，但不强制，根据权重。

```
- key: another-node-label-key  #节点标签key
  operator: In	# 包括 In，NotIn，Exists，DoesNotExist，Gt，Lt
  values:
  - another-node-label-value   # 节点标签value
```

如果有多个nodeSelectorTerms，只需要满足一个就会被调度

如果有多个nodeSelectorTerms或matchExpressions，需要满足所有，pod才会被调度

#### nodeAntiAffinify

nodeAntiAffinify(节点反亲和)，同上，将 ' nodeAffinity ' 字段改为 ' nodeAntiAffinity '

#### podAffinify

同上nodeAffinify pod选择标签为pod的标签

#### podAntiAffinity

同上nodeAntiAffinify pod选择标签为pod的标签

#### 参考链接

https://kubernetes.io/docs/concepts/configuration/assign-pod-node/



