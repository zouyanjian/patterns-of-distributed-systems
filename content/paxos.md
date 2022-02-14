# Paxos

**原文**

https://martinfowler.com/articles/patterns-of-distributed-systems/paxos.html

采用两阶段共识构建，即便节点断开连接，也能安全地达成共识。

**2021.1.5**

## 问题

当多个节点共享状态时，它们往往需要彼此之间就对某个特定值达成一致。采用[领导者和追随者（Leader and Followers）](leader-and-followers.md)模式，领导者会确定这个值，并将其传递给追随者。但是，如果没有领导者，这些节点就需要自己确定一个值。（即便采用了领导者和追随者，它们也需要这么做选举出一个领导者。）

通过采用[两阶段提交（Two Phase Commit）](two-phase-commit.md)，领导者可以确保副本安全地获得更新，但是，如果没有领导者，我们可以让竞争的节点尝试获取 [Quorum](quorum.md)。这个过程更加复杂，因为任何节点都可能会失效或断开连接。一个节点可能会在一个值上得到 Quorum，但在将这个值传给整个集群之前，它就断开连接了。