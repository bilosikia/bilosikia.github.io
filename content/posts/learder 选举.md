# learder 选举

- 只投票给比自己新的节点

  - 选举前，增加自己的 term
  - 发起投票的节点， term 大于等于当前节点 term（第三条包含）
  - 发起投票的节点，log 要新(Up-to-date)
    - log中的 term 谁大谁更新
    - term 号相同时，logid 越大越新
    - 注意，比较的是当前 log 中的term，并不是current_term

- 一个任期只投票给一个节点

- 随机超时

- 被隔离的节点，term 号变得极大

  - 导致重新选主，影响可用性
  - 预选举 + 其他节点同意发起重新选举的条件更严格
    - 没有收到 Leader 的心跳，即至少有一次选举超时
    - Candidate 日志足够新
    - 因此无法增加 term，重新加入集群时不会导致重新选主

- 转为 follower, 只要有新leader term 号至少和自己的 term 号一样大

-  Election Safety Property

- 疑问：

  - 如果预选举成功，但是选举失败？
  - 发起选举节点如果 term 比较大，但是 log 旧，怎么加入集群呢？

  

