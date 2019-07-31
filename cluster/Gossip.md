Gossip
==

Gossip协议:
--

思想十分简单,在一个n个节点的集群中,一个master会定时的随机的选择m个节点,向已知的一个节点进行信息发送这m个节点的信息,我们可以认为在一定时间之后,会达到集群信息收敛的结果

可能的问题:
--

1. 发送给哪一个节点
2. 要发送给哪些节点
3. 接到信息的节点该如何处理

`clusterCron`: 发送方
--

信息发送方主要是调用`clusterSendPing`函数的:

1. clusterCron,定时的对节点进行ping,10Hz发送ping,接受方经过一定挑选
2. 在获得ping的时候会回复一个pong
3. failover结束的时候,像本master的所有slave发送一个pong,用来更新各种信息

clusterCron挑选:

- 下面那位在随机的挑选,这里是挑选一个接受方
- 随机挑选5个,把gossip发给最久没有发送过信息的那个
- 设置了一个timeout,无论如何ping一下那些1/2*Timeout都没有PONG的节点

clusterSendPing发送节点挑选:

- 真正的,完全的随机挑选
- 发送的都是一些十分简单的信息,主要是一些名字,端口,响应时间,IP等等,还会附加上各种FLAG(实现PFAIL监控的主要地方)



`clusterProcessPacket`: 接收方处理
--

主要调用这个函数:`clusterProcessGossipSection`:

- 注:都是统一使用的master来对下线进行检测(FAIL&PFAIL)
- **减少Gossip处理节点的数目,gossip的数量是一种重要的参数-----收敛速度的标准,数目越少收敛速度越快**

1. 发送方发送了节点下线信息 **在这个地方之后处理主节点发送的下线报告**
   - 主节点一旦下线，加入到一个下线节点信息的list里面,list包含是哪一个sender发送的gossip,**可以保证消息不重复**  
   - 客观下线监控,**十分重要的一点，支持了raft协议状态机，**，半数cluster**Master**认为某个主机下线了，使用函数`markNodeAsFailingIfNeeded`来检测状态变化一旦可以标记为客观下线立即标记(FAIL状态)
   - **当本节点为master时** 报告PFAIL状态的时进行广播消息,集群中的**每一个节点(当然包含slave节点)**都会接受到对应主机失效的信息
   - 注意:FAIL状态是由gossip协议被动传播收敛的PFAIL是主动广播收敛的(和sentinel有重大的区别,sentinel的主要作用是检测master的变化,通过__HELLO__实现快速收敛)

2. 发现之前下线节点现在恢复了,直接在list里面移除当前这个sender的下线报告消息
3. 当前节点认为这个节点down了,但是发现发送过来的消息说他的ip和port变化了,直接重新handshake吧
4. 不认识这个节点直接handshake