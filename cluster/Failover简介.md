Failover 简介
==

本节描述最激动人心的Failover过程,Failover主要要关注一下几个问题:

1. 主观下线与客观下线,定界的标准(客观判定下线如何使用Gossip的)
2. raft状态机迁移过程,Failover Leader Election
3. failover执行过程
4. failover后slots是如何保证一致性同步的

背景信息:

1. cluster集群中每个节点本质上是平等的地位,都统一的存在`clusterState->nodes`的表之中
2. cluster node分为两类,一类为master,一类为slave,master负责对槽进行处理,slave负责对master进行复制
3. failover本质上只发生在slave的地方,slave发现自己的master发生PFAIL,自动执行failover,进行投票请求,在这个地方由于是由raft协议进行守护的,保证在一个epoch之中只会产生一个leader,选出leader之后可以立即执行failover操作

以下为failover的主要执行过程函数:

```C
    if (nodeIsSlave(myself)) {
        clusterHandleManualFailover();
        clusterHandleSlaveFailover();
        if (orphaned_masters && max_slaves >= 2 && this_slaves == max_slaves)
            clusterHandleSlaveMigration(max_slaves);
    }
```

