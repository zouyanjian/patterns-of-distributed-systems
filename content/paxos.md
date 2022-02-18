# Paxos

**原文**

https://martinfowler.com/articles/patterns-of-distributed-systems/paxos.html

采用两阶段共识构建，即便节点断开连接，也能安全地达成共识。

**2021.1.5**

## 问题

当多个节点共享状态时，它们往往需要彼此之间就对某个特定值达成一致。采用[领导者和追随者（Leader and Followers）](leader-and-followers.md)模式，领导者会确定这个值，并将其传递给追随者。但是，如果没有领导者，这些节点就需要自己确定一个值。（即便采用了领导者和追随者，它们也需要这么做选举出一个领导者。）

通过采用[两阶段提交（Two Phase Commit）](two-phase-commit.md)，领导者可以确保副本安全地获得更新，但是，如果没有领导者，我们可以让竞争的节点尝试获取 [Quorum](quorum.md)。这个过程更加复杂，因为任何节点都可能会失效或断开连接。一个节点可能会在一个值上得到 Quorum，但在将这个值传给整个集群之前，它就断开连接了。

## 解决方案

Paxos 算法由 [Leslie Lamport](http://lamport.org/) 开发，发表于 1998 年的论文[《The Part-Time Parliament》](http://lamport.azurewebsites.net/pubs/pubs.html#lamport-paxos)中。Paxos 的工作分为三个阶段，以确保即便在部分网络或节点失效的情况下，多个节点仍能对同一值达成一致。前两个阶段的工作是围绕一个值构建共识，最后一个阶段是将该共识传达给其余的副本。

* 准备阶段：建立最新的[世代时钟（Generation Clock）](generation-clock.md)，收集已经接受的值。
* 接受阶段：提出该世代的值，让各副本接受。
* 提交阶段：让所有的副本了解已经选择的这个值。

在第一阶段（称为**准备阶段**），提出值的节点（称为**提议者**）会联系集群中的所有节点（称为**接受者**），它会询问他们是否能承诺（promise）考虑它给出的值。一旦接受者形成一个 Quorum，都返回其承诺（promise），提议者就会进入下一个阶段。在第二个阶段中（称为**接受阶段**），提议者会发出提议的值，如果节点的 Quorum 接受了这个值，那这个值就**被选中**了。在最后一个阶段（称为**提交阶段**），提议者就会把这个选中的值提交到集群的所有节点上。

