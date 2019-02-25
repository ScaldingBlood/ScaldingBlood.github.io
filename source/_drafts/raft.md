---
title: raft
tags:
---
https://zhuanlan.zhihu.com/p/32052223
paxos:
(Basic)
系统角色分为Proposer Acceptor Learner
三个阶段
* Prepare: Proposer生成全局唯一且递增的Proposal ID 发送给其他所有的Acceptor Acceptors收到后返回promise
* Accept: Proposer收到多数Acceptors的promise后，向Acceptors发出propose请求，而Acceptors则返回accept
* Learn: Proposer收到多数Acceptors的accept后，认为此次决议成功，向所有的Acceptors发送结果

两个承诺：
不再接受proposeID <= 当前请求的Prepare
不再接受proposeID < 当前请求的Propose
一个应答：
不违背以前承诺的前提下，回复已经Acceptor的提案的value和proposeID。
* Proposer会选择proposeID最大的promise携带的value进行propose，如果没有可以自定义value
* Acceptor会持久化Proposer提出的value和proposeID
  
multi-paxos:
basic缺点：网络请求过多，有可能形成活锁
multi-paxos首先选出一个Leader（一次决议，可以使用basic paxos），之后由Leader提出propose，Leader宕机后重新选举。
这样的做法省略了preprae的过程，仅需执行一次。

raft：
它将一致性分解为多个子问题：Leader选举（Leader election）、日志同步（Log replication）、安全性（Safety）、日志压缩（Log compaction）、成员变更（Membership change）等。
角色：
Leader: 接受客户端请求，向Follower提出同步日志请求，当同步了大多数Follower后，通知Follower提交日志.
Follower: 接受并持久化Leader同步的日志，在Leader告之日志可以提交之后，提交日志。
Candidate：Leader选举过程中的临时角色。
只能保持存在一个Leader，Leader会发送心跳包，当Follower收包超时一段时间后，会成为Candidate重新选举。

Raft将时间划分为一个个term，从选举开始到下次选举为止。
选举过程：所有Candidate会竞争，向其他所有Candidate发送RPC，Candidate会根据情况返回是否投票。当票数过半选举结束，Leader向所有Follwer发送心跳包。

日志同步：Leader收到客户款的请求后，会把请求作为日志条目，向所有Follower发送RPC。如果Follower没有确认会一直重发。当大多数Follower确认后会返回客户端执行结果。
日志包含日志编号和termID。出现编号和termID相同的两条日志记录它和它们之前的条目一定一样。Follwer会拒绝不相邻的日志记录。
Leader崩溃可能会导致一些Follower没有复制完日志，导致日志不一致。新的Leader会强制覆盖Follower的日志记录来处理，它会从后往前试，不断发送RPC，找到一致点后向后覆盖。

安全性：
* 拥有最新的log entry的Candidate才有可能成为Leader。比较方式是先比较term再比较log index。
* **Leader只能推进commit index来提交当前term的已经复制到大多数服务器上的日志，旧term日志的提交要等到提交当前term的日志来间接提交（log index 小于 commit index的日志被间接提交）。**
具体示例感觉是个极端情况。总之下一次的大多数确认只能被用来提交上一次的大多数确认。

日志压缩：日志不能过长，否则需要长时间回放，需要存储为snapshot。这个snapshot需要大小适中，否则影响性能，可以用copy-on-write技术避免snapshot过程影响正常日志同步。snapshot中包含最后一条log entry的log index和termID以及当前系统状态。

成员变更：成员变更是在集群运行过程中副本发生变化，如增加/减少副本数、节点替换等。
成员变更有可能会影响系统的可性，因为存在新旧两套成员配置，在切换过程中可能会出现多个Leader的情况。因此使用一个称为共同一致的过渡状态来处理。
因此raft使用了两阶段的成员变更方法。先由Cold变为Cold U Cnew，再由Cold U Cnew变为Cnew，都是作为log entry执行。
