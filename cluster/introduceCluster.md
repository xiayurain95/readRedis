redis Cluster
--
redis cluster是redis的集群模式,本质上是实现了一个分布式的KV内存数据库,为了实现这个功能,首先需要解决以下几个问题:

1. 实现请求分发,目前首先的cluster逻辑是在服务器端使用hash(slots)方式实现负载均衡,主要实现为:客户端连接集群一台主机,发送对应Key,主机计算对应的hash Key,落到本机slots本机进行回复,落到其他主机的,返回对应主机IP地址与端口号,由客户端再次发起请求
2. 实现主从同步,这个位置主要是使用redis的replication实现的master与slave区分,slave对master数据进行同步(sync全同步与psync部分同步)
3. 使用raft维护集群**master拓扑**与重要信息(slot部署信息)一致性,基于raft协议实现failover.保证slots的0xFFFF个slot全部发生指派.利用gossip协议进行节点发现与节点信息交换

首先,主要函数:

1. init(负责初始化,初始化事件处理引擎,注册监听事件(Listen)的回调函数)
2. cron定时触发事件(负责pingpong监听当前cluster状态).
3. 一个可爱的伪终端(位于代码最后一部分,负责构成伪终端,像peer节点发送命令)
4. replication(负责主从的同步操作啥的psync,与sync支持)

事件处理入口:

1. 检查cluster函数入口，在redis.c中可以发现调用链`main`--->`initServer`--->`clusterInit`，cluster的入口函数为本函数
  - 本函数作为文件事件处理函数入口
2. 也有一个时间定期函数`clusterCron`,在redis内部被ServiceCron定期调用一次
  - 本函数作为时间事件处理函数入口

`clusterInit`函数主要作用:

生成一个Listen套接字注册到eventloop作为文件事件，注册回调函数(注:当有连接到来时,Listen会产生一个readable事件,回调函数可以进行accept实现...非阻塞)
Listen的回调函数:`clusterAcceptHandler`,当对连接进行accept
  - accept之后获得与对端链接的新套接字，使用epoll绑定新的文件描述符注册回调函数`clusterReadHandler`到事件循环中
  - 回调函数调用链:`clusterReadHandler`---->`clusterProcessPacket`(处理对端发送数据包),实现对对端发送包的监控

下一节主要描述`clusterProcessPacket`对对端的包的处理过程
--

### 额外的知识:

`clusterLink`结构体定义了一个文件描述符，实现对收发信息的缓冲
  - ev的专属缓冲区
```C
clusterLink *createClusterLink(clusterNode *node) {
    clusterLink *link = zmalloc(sizeof(*link));
    link->ctime = mstime();
    link->sndbuf = sdsempty();
    link->rcvbuf = sdsempty();
    link->node = node;
    link->fd = -1;
    return link;
}
```
