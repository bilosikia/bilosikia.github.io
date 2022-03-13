

q: 实现了哪种一致性

a: 顺序一致性



leader 选举

选举条件：

只有拥有最新的已提交的 commit 的节点才有资格。

- 已接受 log 的 term 号最好
- log 号的 id 最大

投票后，要更新自己的 term 吗





log 复制

关系变更

安全



概念：

term

logid

role



rpc：

心跳

选举

ack

log



优化：

快速复制

压缩

重新加入导致选举



