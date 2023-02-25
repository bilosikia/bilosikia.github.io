If a single configuration change adds or removes many servers, switching the cluster directly

from the old configuration to the new configuration can be unsafe; it isn’t possible to atomically

switch all of the servers at once, so the cluster can potentially split into two independent majorities

during the transition



# 一次变更一个成员

> when adding or removing just a single server, it is safe to switch directly to the new configuration.

## 为什么不会造成脑裂：

在下一次变更前，之前的一次变更，一定是被同步到大多数节点的。新增一个节点，即便 Cnew 同步到

# joint consuse