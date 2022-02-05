# 复制日志（Replicated Log）

**原文**

https://martinfowler.com/articles/patterns-of-distributed-systems/replicated-log.html

通过使用复制到所有集群节点的预写日志，保持多个节点的状态同步。

**2022.1.11**

## 问题

当多个节点共享一个状态时，该状态就需要同步。所有的集群节点都需要对同样的状态达成一致，即便某些节点崩溃或是断开连接。这需要对每个状态变化请求达成共识。

但仅仅在单个请求上达成共识是不够的。每个副本还需要以相同的顺序执行请求，否则，即使它们对单个请求达成了共识，不同的副本会进入不同的最终状态。

## 解决方案

集群节点维护了一个[预写日志（Write-Ahead Log）](write-ahead-log.md)。每个日志条目都存储了共识所需的状态以及相应的用户请求。这些节点通过日志条目的协调建立起了共识，这样一来，所有的节点都拥有了完全相同的预写日志。然后，请求按照日志的顺序进行执行。因为所有的集群节点在每条日志条目都达成了一致，它们就是以相同的顺序执行相同的请求。这就确保了所有集群节点共享相同的状态。

使用 [Quorum](quorum.md) 的容错共识建立机制需要两个阶段。
* 一个阶段负责建立[世代时钟（Generation Clock）](generation-clock.md)，了解在前一个 [Quorum](quorum.md) 中复制的日志条目。
* 一个阶段负责在所有集群节点上复制请求。

每次状态变化的请求都去执行两个阶段，这么做并不高效。所以，集群节点会在启动时选择一个领导者。领导者会在选举阶段建立起[世代时钟（Generation Clock）](generation-clock.md)，然后检测上一个 [Quorum](quorum.md) 所有的日志条目。（前一个领导者或许已经将大部分日志条目复制到了大多数集群节点上。）一旦有了一个稳定的领导者，复制就只由领导者协调了。客户端与领导者通信。领导者将每个请求添加到日志中，并确保其复制到到所有的追随者上。一旦日志条目成功地复制到大多数追随者，共识就算已经达成。按照这种方式，当有一个稳定的领导者时，对于每次状态变化的操作，只要执行一个阶段就可以达成共识。

### 多 Paxos 和 Raft

[多 Paxos](https://www.youtube.com/watch?v=JEpsBg0AO6o&t=1920s) 和 [Raft](https://raft.github.io/) 是最流行的实现复制日志的算法。多 Paxos 只在学术论文中有描述，却又语焉不详。[Spanner](https://cloud.google.com/spanner) 和 [Cosmos DB](https://docs.microsoft.com/en-us/azure/cosmos-db/introduction) 等云数据库采用了[多 Paxos](https://www.youtube.com/watch?v=JEpsBg0AO6o&t=1920s)，但实现细节却没有很好地记录下来。Raft 非常清楚地记录了所有的实现细节，因此，它成了大多数开源系统的首选实现方式，尽管 Paxos 及其变体在学术界得到了讨论得更多。

#### 复制客户端请求

![复制内核](../image/raft-replication.png)
<center>图1：复制</center>

对于每个日志条目而言，领导者会将其追加到其本地的预写日志中，然后，将其发送给所有追随者。

```java
leader (class ReplicatedLog...)

  private Long appendAndReplicate(byte[] data) {
      Long lastLogEntryIndex = appendToLocalLog(data);
      replicateOnFollowers(lastLogEntryIndex);
      return lastLogEntryIndex;
  }


  private void replicateOnFollowers(Long entryAtIndex) {
      for (final FollowerHandler follower : followers) {
          replicateOn(follower, entryAtIndex); //send replication requests to followers
      }
  }
```

追随者处理复制请求，将日志条目追加到其本地日志中。在成功追加日志条目后，他们将其拥有的最新日志条目索引回应给领导者。应答还要包括服务器的当前[世代时钟](generation-clock.md)。

追随者还会检查日志条目是否已经存在，或者是否存在超出正在复制的日志条目。它会忽略了已经存在的日志条目。但是，如果有来自不同世代的日志条目，它们也会删除存在冲突的日志条目。

```java
follower (class ReplicatedLog...)

  void maybeTruncate(ReplicationRequest replicationRequest) {
      replicationRequest.getEntries().stream()
              .filter(entry -> wal.getLastLogIndex() >= entry.getEntryIndex() &&
                      entry.getGeneration() != wal.readAt(entry.getEntryIndex()).getGeneration())
              .forEach(entry -> wal.truncate(entry.getEntryIndex()));
  }
follower (class ReplicatedLog...)

  private ReplicationResponse appendEntries(ReplicationRequest replicationRequest) {
      List<WALEntry> entries = replicationRequest.getEntries();
      entries.stream()
              .filter(e -> !wal.exists(e))
              .forEach(e -> wal.writeEntry(e));
      return new ReplicationResponse(SUCCEEDED, serverId(), replicationState.getGeneration(), wal.getLastLogIndex());
  }
```

当复制请求中的世代数低于服务器知道的最新世代数时，跟随者会拒绝这个复制请求。这样一来就给了领导一个通知，让它下台，变成一个追随者。

```java
follower (class ReplicatedLog...)

  Long currentGeneration = replicationState.getGeneration();
  if (currentGeneration > request.getGeneration()) {
      return new ReplicationResponse(FAILED, serverId(), currentGeneration, wal.getLastLogIndex());
  }
```

收到响应后，领导者会追踪每个服务器上复制的日志索引。领导者会利用它
追踪成功复制到 [Quorum](quorum.md) 日志条目，这个索引会当做提交索引（commitIndex）。commitIndex 就是日志中的[高水位标记（High-Water Mark）](high-water-mark.md)

```java
leader (class ReplicatedLog...)

  logger.info("Updating matchIndex for " + response.getServerId() + " to " + response.getReplicatedLogIndex());
  updateMatchingLogIndex(response.getServerId(), response.getReplicatedLogIndex());
  var logIndexAtQuorum = computeHighwaterMark(logIndexesAtAllServers(), config.numberOfServers());
  var currentHighWaterMark = replicationState.getHighWaterMark();
  if (logIndexAtQuorum > currentHighWaterMark && logIndexAtQuorum != 0) {
      applyLogEntries(currentHighWaterMark, logIndexAtQuorum);
      replicationState.setHighWaterMark(logIndexAtQuorum);
  }

leader (class ReplicatedLog...)

  Long computeHighwaterMark(List<Long> serverLogIndexes, int noOfServers) {
      serverLogIndexes.sort(Long::compareTo);
      return serverLogIndexes.get(noOfServers / 2);
  }

leader (class ReplicatedLog...)

  private void updateMatchingLogIndex(int serverId, long replicatedLogIndex) {
      FollowerHandler follower = getFollowerHandler(serverId);
      follower.updateLastReplicationIndex(replicatedLogIndex);
  }

leader (class ReplicatedLog...)

  public void updateLastReplicationIndex(long lastReplicatedLogIndex) {
      this.matchIndex = lastReplicatedLogIndex;
  }
```

##### 完全复制

有一点非常重要，就是要确保所有的节点都能收到来自领导者所有的日志条目，即便是节点断开连接，或是崩溃之后又恢复之后。Raft 有个机制确保所有的集群节点能够收到来自领导者的所有日志条目。

在 Raft 的每个复制请求中，领导者还会发送在复制日志条目前一项的日志索引及其世代。如果前一项的日志条目索引和世代与本地日志中的不匹配，追随者会拒绝该请求。这就向领导者表明，追随者的日志需要同步一些较早的日志条目。

```java
follower (class ReplicatedLog...)

  if (!wal.isEmpty() && request.getPrevLogIndex() >= wal.getLogStartIndex() &&
           generationAt(request.getPrevLogIndex()) != request.getPrevLogGeneration()) {
      return new ReplicationResponse(FAILED, serverId(), replicationState.getGeneration(), wal.getLastLogIndex());
  }

follower (class ReplicatedLog...)

  private Long generationAt(long prevLogIndex) {
      WALEntry walEntry = wal.readAt(prevLogIndex);

      return walEntry.getGeneration();
  }
```

这样，领导者会递减匹配索引（matchIndex），并尝试发送较低索引的日志条目。它会一直这么做，直到追随者接受复制请求。

```java
leader (class ReplicatedLog...)

  //rejected because of conflicting entries, decrement matchIndex
  FollowerHandler peer = getFollowerHandler(response.getServerId());
  logger.info("decrementing nextIndex for peer " + peer.getId() + " from " + peer.getNextIndex());
  peer.decrementNextIndex();
  replicateOn(peer, peer.getNextIndex());
```

这个对前一项日志索引和世代的检查允许领导者检测两件事。

* 追随者是否存在日志条目缺失。例如，如果追随者只有一个条目，而领导者要开始复制第三个条目，那么，这个请求就会遭到拒绝，直到领导者复制第二个条目。
* 日志中的前一个是否来自不同的世代，与领导者日志中的对应条目相比，是高还是低。领导者会尝试复制索引较低的日志条目，直到请求得到接受。追随者会截断世代不匹配的日志条目。

按照这种方式，领导者通过使用前一项的索引检测缺失或冲突的日志条目，尝试将自己的日志推送给所有的追随者。这就确保了所有的集群节点最终都能收到来自领导者的所有日志条目，即使它们断开了一段时间的连接。

Raft 没有单独的提交消息，而是将提交索引（commitIndex）作为常规复制请求的一部分进行发送。空的复制请求也可以当做心跳发送。因此，commitIndex 会当做心跳请求的一部分发送给追随者。

##### 日志条目以日志顺序执行

一旦领导者更新了它的 commitIndex，它就会按顺序执行日志条目，从上一个 commitIndex 的值执行到最新的 commitIndex 值。一旦日志条目执行完毕，客户端请求就完成了，应答会返回给客户端。

```java
class ReplicatedLog…

  private void applyLogEntries(Long previousCommitIndex, Long commitIndex) {
      for (long index = previousCommitIndex + 1; index <= commitIndex; index++) {
          WALEntry walEntry = wal.readAt(index);
          var responses = stateMachine.applyEntries(Arrays.asList(walEntry));
          completeActiveProposals(index, responses);
      }
  }
```

领导者还会在它发送给追随者的心跳请求中发送 commitIndex。追随者会更新 commitIndex，并以同样的方式应用这些日志条目。

```java
class ReplicatedLog…

  private void updateHighWaterMark(ReplicationRequest request) {
      if (request.getHighWaterMark() > replicationState.getHighWaterMark()) {
          var previousHighWaterMark = replicationState.getHighWaterMark();
          replicationState.setHighWaterMark(request.getHighWaterMark());
          applyLogEntries(previousHighWaterMark, request.getHighWaterMark());
      }
  }
```