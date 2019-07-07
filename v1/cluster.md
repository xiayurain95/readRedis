redis CLuster
--
- 首先,主要函数:
  1. init(负责初始化,初始化事件处理引擎,注册监听事件(Listen)的回调函数,)
  2. cron定时触发事件(负责pingpong监听当前cluster状态).
  3. 一个可爱的终端(位于代码最后一部分,负责构成伪终端,像peer节点发送命令)
  4. 调用了replication(负责主从的同步操作啥的psync,与sync支持)

- 检查cluster函数入口，在redis.c中可以发现调用链`main`--->`initServer`--->`clusterInit`，cluster的入口函数为本函数
  - 本函数作为文件事件处理函数入口
- 也有一个时间定期函数`clusterCron`,在redis内部被ServiceCron定期调用一次
  - 本函数作为事件处理函数入口

- `clusterInit`函数主要作用，生成一个Listen套接字注册到eventloop作为文件事件，注册回调函数(注:当有连接到来时,Listen会产生一个readable事件,回调函数可以进行accept实现...非阻塞)`clusterAcceptHandler`,当对连接进行accept
  - accept之后不是获得的新套接字，注册回调函数`clusterReadHandler`到事件循环中,回调函数调用链:`clusterReadHandler`---->`clusterProcessPacket`(处理对端发送数据包)
  
- 针对这个套接字的文件描述符，使用了一个叫做clusterLink的结构体进行包裹，主要是把发送以及接收的信息进行缓冲
  - 貌似也没有对这个link做啥统一的存储，貌似只是一个单纯的privateData。用来给eventLoop使用的感觉 （ev的专属缓冲区）
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

clusterProcessPacket
-- 
- 这个函数可能是真正的处理逻辑的函数块
- 首先，从ev专属的缓存里面读取数据，redis前面8个字节定义了一些消息数据信息（消息头）
  - 4字节的MSS真的是经典操作哟 
  - 信息包头组织方案，在下面，在socket进行读的时候，**分为两个逻辑阶段进行**，首先要先读取消息头部，固定的长度，8个字节，然后再根据MSS，计算还需要读多少，是不是读完了，之类的
    - 原因！！！由于TCP是面向流的协议，但是read返回为0其实是EOF的时候，所以不能读取太多了，免得。。。引发阻塞什么的orz（好吧，**原来是read原语引发的阻塞吗**，今天才想到哟）
    ```C
      // 消息的长度
      uint32_t totlen = ntohl(hdr->totlen);

      // 消息的类型
      uint16_t type = ntohs(hdr->type);

      // 消息发送者的标识
      uint16_t flags = ntohs(hdr->flags);
    ```
- 然后当然是对消息进行匹配鸭
  - 当前支持的消息种类众多，基本有以下的品种
  - 与之前sentinel最大的区别是，这里其实都是通过这个函数实现的集群之间的通信的，而sentinel是通过hello信道进行相互通信的（通过hello行到有一个好处，就是一台主机master为单位，进行的同学）
  - 使用套接字直接通信是因为。。。就一个集群啊，又不是有很多集群在。。。
  - 是的，这个地方其实就是处理clusterMSG信息的大中心，写成了一个。。。if else的样子（其实感觉封装性不是很好，之后可能有重构什么的吧）
  - **下面的每一个品种都被if** **else覆盖到了**
```C
#define CLUSTERMSG_TYPE_PING 0          /* Ping */
// PONG （回复 PING）
#define CLUSTERMSG_TYPE_PONG 1          /* Pong (reply to Ping) */
// 请求将某个节点添加到集群中
#define CLUSTERMSG_TYPE_MEET 2          /* Meet "let's join" message */
// 将某个节点标记为 FAIL
#define CLUSTERMSG_TYPE_FAIL 3          /* Mark node xxx as failing */
// 通过发布与订阅功能广播消息
#define CLUSTERMSG_TYPE_PUBLISH 4       /* Pub/Sub Publish propagation */
// 请求进行故障转移操作，要求消息的接收者通过投票来支持消息的发送者
#define CLUSTERMSG_TYPE_FAILOVER_AUTH_REQUEST 5 /* May I failover? */
// 消息的接收者同意向消息的发送者投票
#define CLUSTERMSG_TYPE_FAILOVER_AUTH_ACK 6     /* Yes, you have my vote */
// 槽布局已经发生变化，消息发送者要求消息接收者进行相应的更新
#define CLUSTERMSG_TYPE_UPDATE 7        /* Another node slots configuration */
// 为了进行手动故障转移，暂停各个客户端
#define CLUSTERMSG_TYPE_MFSTART 8       /* Pause clients for manual failover */
```

1. 首先是meet和ping 
    - meet 没啥可说的，新建一个描述struct，然后填充ip port，加入豪华午餐list
      - meet是客户端发送给主机的一个命令,所以他的本质是让主机连接给定的一个ip:port,所以是需要一系列的handshake的(以handshake状态加入到list里面)(之后是cron解决吧) 
    - ping pong meet 是符合嘴碎协议的（gossip），对对应的gossip进行处理
    1. 不认识这个节点（在dict里面没有查到），直接用Handshake函数来简建立连接握爪
      - **handshake的说法好奇怪，，，好奇怪好奇怪，直接加到dict里面就直接说之后会handshake了。。。为啥哟，有cron吗？？？** 
      - 知道了，这里主要是有一个flag，在创建这个节点的时候有一个  createClusterNode(NULL,REDIS_NODE_HANDSHAKE)，也就是handshake的标志吧,**具体是主要的,我们handshake一个节点,主要是通过PING和MEET来的,MEET是人手动指定的,PING是程序自动的,当发现不认识这个节点的时候,三次握手开始了,有两种!!!!!!方案:1. MEET PONG PING 2. PING PONG PING就会建立一个握手的关系** 
    2. 看一下嘴碎协议,嘴碎协议有啥好的呢，每次收到gossip，首先解包获取节点信息
      - **可以得到已知节点在发送gossip信息节点看来的状态信息**
      1. 发送gossip的节点觉得它已经GG了
         - 是否已经GG的信息
         - 节点一旦gg，直接加入到一个下线节点信息的list里面（master才有这个待遇）,list包含是哪一个sender发送的gossip,**可以保证消息不重复,(raft一定要保证单一结点消息的唯一性)**  
         - **十分重要的一点，就是raft协议状态机的问题，**，在这个位置也需要半数cluster来认领才可以认为某个主机下线了，加入list是节点记录其他主机观察的下线状态，list叫做下线报告（这里面好像是怎么说的，虽然我觉得不该这么叫哟）主观下线需要半数承认
           - 使用函数markNodeAsFailingIfNeeded来认领这个状态
           - 要求在当前主机对该节点的测度为PFAIL状态的时候,验证是否为FAIL状态(主观下线与客观下线的分野)
           - 一旦到达PFAIL状态的时候进行广播消息,这里抵达的是一种最终一致性的状态
      2. 发现之前GG的现在好像康复了,直接在list里面移除当前这个sender的假消息
      3. **发现了吗,其实我们对某一个节点失效的度量建立在有消息发送过来的情况下,也就是说,是gossip触发的一种失效度量,因此,gossip的数量是一种重要的参数-----收敛速度的标准**
      4. 当前节点认为这个节点down了,但是发现发送过来的消息说他的ip和port变化了,直接重新handshake吧
    3. 不认识这个节点直接handshake
2. 这是一条 PING 、 PONG 或者 MEET 消息,处理config(超级长的代码...)
  1. 发现认识当前的节点,并且这是一条handshake的回复(就是PING或者PONG类型的回复)(上面有写具体内容),就是说handshake已经结束了,把多余的其他信息加到当前节点里面去,能发送pingPONG应该是连接已经建立好了,更新一下当前节点对sender的认知(修改状态啊(现在不在handshake状态了,这位的名字是啥,端口是啥,是master还是slave什么的)),然后...没有然后了
  2. 然后是PING PONG了(不在handshake状态的),ping直接就是传递一下端口啊,地址啊,有没有变化什么的.PONG主要是对本节点来说的,
     - 首先是更新各种时钟,比如发送ping的间隔时间清零了啊,再比如,当前回复PONG的时间(为了fail和pfail)
     - 更新fail和pfail,pfail一定要更新,因为这个是客观下线,fail根据情况(主观下线,看一下现在满足主观上线的要求不)调用函数clearNodeFailureIfNeeded
       - 从节点nobody cares,主节点大家都care:发现fail之后会立即开始failover,但是failover是需要时间的,如果发现failover没有结束,直接标记上线吧(使用槽标记)(其他节点怎么办哟(leader呢???))**现在觉得存疑诶** 
     - 由于raft协议进行保活,只能保证最终一致性,所以!!!要被动更新诶,这里处理一些乱七八糟的被动更新的东西,比如,1.master变成slave了(变更master的属性,把slave名单清空一下~把已经指派的槽删除了(估计是failover指派新的槽(槽的信息怎么更新了))),2.slave变成master了(简单的...设置一下就完毕了~~)3.当前sender是个slave,但是它的master发生变化了
     - 现在可以回复一下之前的问题:槽指派的信息在发现这个是一个PING PONG MEET信息的时候,会自动的检查一回 **也就是说在每一次主从信息发生变更的时候,立即的更新一下槽的指派信息(免得出问题)**,在之后还有一个原因,是必须要先变更主节点诶,免得...出现一些奇奇怪怪的问题~~(就是在主节点变更之后再指派槽信息),首先算一下dirty_slots的数目啥的(在发送报文的时候,会把当前的槽指派一起发送过去的)
       - 发送的规则是主要的,自己是master,就发自己的槽指派,自己是slave发送自己master的槽指派.
       - 这里进行dirty_slots的运算也是按照这种方式的.发森了变化之后调用一下clusterUpdateSlotsConfigWith函数来更新一下槽指派信息(需要sender是master)
         - 这个函数干啥了?直接的修改当前的槽指派(就是算一下bitmap之类的,修改对应表项)
         - 处理failover时下线之后重新上线的情况(自己下线了然后再重新上限)-------raft状态机出现了,有一个参数叫configEpoch,它更大的话表明当前已经有过一次failover,当发现当前的slot被剥夺给sender之后,我们就知道:自己(master)或者自己(slave)的master GG过一次,进行对应的权限转换(认贼作父),原来是master的自己权限降低了(被拔得一毛(slots)不剩,还能干嘛哟)
       - 在上一个函数中我们处理了一下自己主动的下线的情况,现在我们来谈一下对端的下线情况(两种情况从本质上都是一种网络的分割)
       - 来看一下我们亲爱的slots的定义(在存储的时候,slots 直接存储为当前的一个列表指针),在发送slot信息的时候,我们只会发自己分配到的slots,因此是直接使用一个bitmap,这样的速度会快很多,而且流量不大(在内部也会保存一个bitmap)
       ```C
        // 负责处理各个槽的节点
        // 例如 slots[i] = clusterNode_A 表示槽 i 由节点 A 处理
        clusterNode *slots[REDIS_CLUSTER_SLOTS];
       ``` 
         - 首先,必须要在configEpoch(这个地方是对对应的slot而言的(slot存的是指针嘛)(`server.cluster->slots[j]->configEpoch`))大于sender的情况(这个时候其实我们的sender已经发生了下线后的重上线问题),这个地方其实这样做可能是为了快收敛(只要有发有收,就立即更新slot(raft状态机主要守护的状态之一)),如果发生了,通知给sender,sender根据具体的信息立即进行调整
      - 现在问题来了,就不可以发生冲突吗(两个的configEpoch一样),调用这个函数:clusterHandleConfigEpochCollision(代码有很多来解释这个函数哟)
        - (感觉主要是解决这样的一个问题:A,B下线,两位)
        - 对于slave,我们可以不用考虑的,同一个master,slave域,由于raft协议的加持,半数才能投票成功,因此完全的杜绝了分区隔离的情况发生,如果出现了分区,一定的高的epoch压制低的epoch,所以这个问题是特别局限的情况
          - 用户手动改配置(如果A在改配置,configEpoch自动加一(但是不用raft),B在这个时候下线然后被选主了~~)
          - (还说为了防止未知的BUG(真是服气哟,看来大家对自己的代码都不是很自信,233333))
          - 在启动的时候,大家(每一个master)在一个configEpoch上面(**难道说在正常的情况下大家的configEpoch是不一样的吗**),目的是为了避免多个node同时对同一个slot进行服务(这样就是多主的情况了(逻辑上只有))
          - configEpoch就是简单的就+1了一下而已~~
    - 最后还是来一次嘴碎协议~~(为啥要你哟...之前不是已经嘴碎过了吗,有啥区别诶~)
3. 消息的类型是FAIL(就是说有节点GG了)
  - 先看一下认识这个sender不(必须要经过三次握手才可以进行接受信息),认识这个sender的话,直接把对应的节点标记为GG(FAIL),顺便reset一下PFAIL(毕竟是客观下线嘛,现在已经主观下线了)
  - 啥时候进行故障转移哟
4. PUBLISH消息(好吧,好高端的样子,之前没看到诶)
5. raft协议相关主要有两个信息,先讲CLUSTERMSG_TYPE_FAILOVER_AUTH_REQUEST,它主要调用了clusterSendFailoverAuthIfNeeded,负责投票方面的信息
     - 实现raft状态机,raft这里主要是保证
        1. 我们的槽指派:因此没有槽指派信息的节点其实是没有投票权的
        2. 当前的master是谁:必须要为master才可以要求投票,不然...???
     - 主要涉及的epoch有两个:
        1. requestCurrentEpoch:必须要等failover结束了之后,当前纪元才会发生更新,这里会对当前纪元进行比较,requestCurrentEpoch必须大于等于当前节点才可以进行投票(可能...之前的候选人在投票阶段GG了了)
        2. requestConfigEpoch:投票的纪元,当前对于槽指派其实是十分细粒度的---每个槽都有一个指针,指着当前的master结构,投票结构中会有请求的slot信息,把请求的slot挨个检查一下,声明的纪元(requestConfigEpoch)比slot的纪元(指针嘛)小,代表有人已经捷足先登了,不投!
    - 需要防止多次投票...这里是这样的`server.cluster->lastVoteEpoch == server.cluster->currentEpoch` 
      - 也就是说当前的epoch其实是还没有投票的状态,在正常情况下lastVoteEpoch+1=currentEpoch
    - 最后记录一下投票的时间(防止一直反复的投票),再使用clusterSendFailoverAuth投票(发送一个通知回去CLUSTERMSG_TYPE_FAILOVER_AUTH_ACK),还要把voteEpoch重新定义到currentEpoch上面
6. 既然有发送票的就一定有接受投票的啊,消息(注意,在这里,消息全部是使用一个int来进行传递的)type是CLUSTERMSG_TYPE_FAILOVER_AUTH_ACK 
   - 接受这个投票需要一些验证的---
      1. 必须是master
      2. raft守护的是槽,所以必须要有槽指派
      3. currentEpoch验证,需要比较大的currentEpoch,说白了就是投票信息必须要在同一个epoch里面
   - server:票数+1,告辞!
7. CLUSTERMSG_TYPE_MFSTART目的是手动的进行故障转移
     - 这个必须是从自己的slave那里得到的信息才可以,主要内容基本就是调用了一下resetManualFailover这个函数
8. CLUSTERMSG_TYPE_UPDATE主要处理权限提升---slave变身master的状况
    - 首先是取出当前的config'epoch(从信息里面),然后比较一下下,本机节点存储的configEpoch和你发送过来的configEpoch大小之间的区别,发送过来的更大的话,说明raft协议起作用了,因此需要更新
      - 更新也是分部分的好吗,首先是提升为master---这里有个坑,就是有可能是手工指定的master提升,所以收到这个信息只单纯的提升为master,之前自己的master是不会去动的  
    - 其次,槽布局可能发森了一些变化,因此也要更新槽布局的,使用函数clusterUpdateSlotsConfigWith (这个函数在PING PONG MEET信息的时候也会被调用一次的)
      - 作用:指派声明的槽,通过槽指派可以间接的发现可能的故障转移操作(当前槽本来是属于这个节点master的),自我更新(可能自己是主节点,自己的槽被剥夺了),根据情况将自己设置为从节点或者直接变成野节点.

clusterCron
--
- 结束了文件事件的bb,现在来看一下redis另一个组成的模型,时间事件(就是Cron),这个的作用是定期发送一些奇奇怪怪的东西,定期检查一些奇奇怪怪的东西,然后实现对应的功能
- 首先是一个循环,这个循环迭代server.cluster->nodes中的每一个node
    - handshake有一定的容忍期限的,在这个位置直接的把它释放掉(就是在handshake_timeout时间之外了)
    - 为没有连接的节点创建一个连接(这个地方主要是在handshake的时候,直接就加到了cron里面,没有进一步考虑创建一个连接啊,什么的)
      - 这里使用了一个啥非阻塞的连接函数哟,请问这个是有啥用呢,三次握手和四次挥手基本上是必须的,你可能可以修改一下时间timeout来实现一下WAIT_TIME1与WAIT_TIME2之间可能的hanging现象,但是只有在ESTABLISHED才可以被放到对应的事件循环里面
        - 这里的非阻塞其实本质是封装调用fcntl,设置了O_NONBLOCK状态位,这个状态位呢,可以直接返回当前文件描述符的状态,而不是阻塞在这里.为啥要设置他,这个你其实主要是由于epool的边缘触发导致的(在这种工作模式下,只会触发一次文件描述符的返回工作,水平触发是反复触发,直到处理完毕),因为反复触发的关系,其实调用epool_wait次数会多很多,反复进入内核态当然慢一点咯~~
        - 使用这种方式的原因是,read原语,有时候你不敢读多了(否则就阻塞在这里造成死锁了)
      - 好的,老八股了,直接使用aeapi加入到事件处理引擎里面去吧,发送一个PING,免得被以为下线了
- gossip嘴碎协议上线,每秒钟随机向一个节点发送gossip信息(好吧,这点真的很随便诶~~确定不搞一点快收敛什么的吗)
    - 好吧,这里开始嘴碎协议了,我很感兴趣他怎么封装的哟~~之前封装和struct定义不一致 
    - 这个地方是随机选择了5个节点发送ping
    - 有一个小优化,就是挑选比当前序列(for循环中5个依次算)老的节点(按PONG算时间),如果均匀分布的话,平均每次发送三个ping(for第一次一定会发送(运气好),剩下的平均是1/2的概率,算下来就是三个节点咯)
- 然后又是一次的遍历过程
    - 在这里,频繁的出现`link->node->ping_sent`,这个属性在每次发送ping的时候会直接被设置,在什么时候回收呢?------处理信息的时候,也就是处理PONG信息报文的时候
    - 首先是检测孤儿节点 Orphaned master,必须是当前主机是slave才关心这个问题(原因是这个:a slave that may migrate to another master) 
    - 之前不是有发送ping嘛,这里来检测一下PONG有没有真的收到,这里会检查一下几个参数:发送ping了吗ping_sent,超过门限了吗cluster_node_timeout,发送ping的事假超过了1/2门限:
      - 干的比较直接,直接把连接掐掉了,在下一个cron的时候会重新的连接一下的(放心)
    - 没有发送过ping信息的节点,现在有一个志愿补录的机会,但是要求满足`(now - node->pong_received) > server.cluster_node_timeout/2)`,就是比较久都没收到你的PONG了,当做扶贫吧~~发送clusterSendPing 
    - 下面是一系列针对当前还在发送ping状态(就是ping发送了,但是没有回收到对应的pong回复的节点)
      - 等待时间太长了,超过了门限,直接打开PFAIL意思下线标志位
- 最后是一系列的扫尾工作
    - `server.masterhost`代表当前节点作为slave节点时,连接master进行复制操作(replicatio)可能出现一种情况,当前的节点是一个slave但是它还没有和他的master进行replication,在这个时候,server.masterhost==Null,直接调用函数设置一下 
    - 当当前节点是slave的时候会主动的调用三个函数`clusterHandleManualFailover`,`clusterHandleSlaveFailover`,`clusterHandleSlaveMigration`
      - clusterHandleManualFailover,目的是执行一个手动failover引擎,感觉没啥可说的
      - **clusterHandleSlaveFailover**,这个函数还是蛮重要的一个函数,实现了raft状态机,主要负责在masterGG的时候(处于FAIL状态的时候),进行恢复的操作
        - **这里就要仔细的说明一下了,这个地方,对于主节点的恢复,其实本质上是分了两个集体来做的,对于FAIL状态的检测,其实是master集群内部来实现的,一旦发现出现可能的下线,直接广播给整个集群中的每一个节点了.**
        - **对于集群的恢复来说,其实是对于slave节点来说的,他们请求master的投票,获得半数投票之后进行恢复操作(好吧,本质上连投票的权力都没有不是~~)** 
        - **`server.cluster`包含的信息主要就是集群的信息了,有哪些master啊,一共有多少节点啊,当前槽分配的样式啊**
        - 在这个函数开始的时候会主动的检查一下,自己的master是不是GG了,然后会算一下当前的数据新不新这个问题(主要看一下上一次replication有没有超过对应的门限--->node timeout 的十倍)
        - 下面是来自于raft的随机candidate算法,auth_retry_time算是一个门限,主要定义的是超过门限标识当前failover基本算是失败了,应该重新发起一个,重新发起一个failover需要有一个喘息时间500ms(传播FAIL信息使用),和一个随机leader竞选时间(candidate收敛时间大概最多在1s左右)
        ```C
        if (auth_age > auth_retry_time) {
        server.cluster->failover_auth_time = mstime() +
            500 + /* Fixed delay of 500 milliseconds, let FAIL msg propagate. */
            random() % 500; /* Random delay between 0 and 500 milliseconds. */
        ``` 
          - 然后会清除上一轮竞选的部分消息(票数什么的)
          - 首先,在当前这种状态下面,我们可以称我们当前的节点为参选candidate了,但是什么时候把它叫做candidate呢,主要看`server.cluster->failover_auth_time`,这个参数主要用来决定当前节点的等待时间,这个时间是一个cluster的UNIX时间戳时间,**表明到达这个时间戳的边界之后,被授予了可以进行发起投票的权限.**
            - 在主动随机退避的基础上,我们还需要一个直白的rank,用来干啥呢,有的slave怎么说呢,消息更新吧,对于这种slave,我们要使用rank来增加它的优势
            ```C
             server.cluster->failover_auth_time +=
            server.cluster->failover_auth_rank * 1000;
            ``` 
            - 最后调用一下下clusterBroadcastPong,像当前master下面的slave发送一波PONG,主要是在排rank的时候用(发送自己的信息,里面有啥offset之类的)(其他人因为不是主观消息,所以没有实用价值)(gossip不是一般只用来当前节点状态(也是当前节点的主观信息)(有哪些节点,哪些节点你连没连,是不是下线了(pfail状态))嘛,具体信息(主观信息)还是不会用gossip的)
            - 在每一次`if (mstime() < server.cluster->failover_auth_time) return;`之前,更新一下rank,之后再执行等待任务,防止可能的消息被遗留,那种
            - 有时候开始进行发起投票的时间过得太久了,直接返回吧`auth_age > auth_timeout`
            - **故障转移状态机开始了**
              - 第一种状态:`server.cluster->failover_auth_sent == 0`(还没发送请求投票消息,开始发送了)
                - 首先提升当前纪元currentEpoch(不是配置纪元(配置纪元是已经稳定下来的状态)),修改failover_auth_epoch,改变整个节点的纪元.
                - clusterRequestFailoverAuth,请求投票的函数,广播消息,但是只要主节点才回复消息
              - 第二种状态:`server.cluster->failover_auth_count >= needed_quorum`已经获得了足够的投票了
               - 将自己转换为主节点,取消当前正在进行的replication
               - 接收槽指派信息,将自己master的槽指定为自己的槽
               - 更新配置纪元(configEpoch),现在已经达到稳定的状态了嘛
               - 用clusterUpdateState函数更新一下状态信息(下一段认真写一下)
               - 发送PONG信息,在这个地方有一个小知识点,在收到PONG信息的时候,会附带cofigEpoch,configEpoch是针对于某一群槽来说的,不同归属的槽会主动避让拥有相同的configEpoch(当然是master来做主动的避让).**在这个位置,一旦发现发送过来附带的槽指派与之前的指派发生改变,便认为发生了failover,其他主节点会主动的修改当前的cluster状态信息,把当前节点修改成为主节点. 对于掉线之后又重新上线的主机,发现自己的槽被指派给其他人了,而且configEpoch还比自己的大,自己主动的成为salve,放弃当前的槽归属,可以看一下之前processPacket的解析或源码(解析PINGPONG)**
      - 函数`clusterHandleSlaveMigration`:在cron里面,这个是一个扶贫函数,一旦检测到一种情况:当前为salve,但是有的master比较贫困(孤儿节点啊,或者低于门限了),直接扶贫,其实就跟字面上一样吧...没啥可说的~~. 
- 现在单独说一下`clusterUpdateState `函数,它调用还是蛮丰富的
    - 目的还是比较单纯的--->更新`server.cluster->state`即集群参数(主观角度),调用也比较简单,主要在cron最后,以及failover即将结束阶段,还有封装为api(doBeforeSleep)
    - 首先是检测一下cluster状态--->是不是每一个槽都已经被成功的指派,遍历所有的槽,如果出现没有指派的槽,直接设置`REDIS_CLUSTER_FAIL`
    - 计算当前cluster大小(有solt的正常主节点),以及不可达主节点(有slot的下线主节点(fail或者pfial))
      - 利用这两个数据计算一下是不是自己出问题了(大于一半的主节点都不可达)
    - 记录一下当前的信息并且写入一下下 
    - Cron的内容结束了 

Cluster Command
--
- 其他的实现还是不要写了~~蛮无聊的,就是解析命令啥的
- 主要写一下这三个命令的实现DUMP, RESTORE and MIGRATE(实现key的迁移操作)

- DUMP,RESTORE命令主要由dumpCommand啥的实现,主要作用是返回一个rdb的obj,然后根据rdb的obj进行反序列化,并且保存到当前的server中(可能是migration使用(不是,三个命令都是和迁移有关系的)?)

一些细碎的点
--
- replication主要代表了当前的master与slave的结构状态,replication的状态不是集群所有的,因此,replication直接修改的是当前主机代表----server结构体中的数据(使用rdb与aof进行复制与同步过程)
- 客户端请求服务器,主要使用函数:getNodeByQuery,这个函数主要使用error_num工作,主要的返回有MOVED,ASK,ASKING,NONE,分别代表:
    1. 基础情形,当前节点没有你需要的key,但是其他节点可能会有,去问一下其他节点(会将其他节点n返回回去)
    2. ASK在啥时候用呢--->迁移的时候使用,当前节点本来是有这个slot的,但是已经被迁移到其他节点上面去咯,所以返回一个ASK(其实基本和MOVED没啥区别 )
    3. 为啥有ASKING呢,主要是ASK的关系,当前节点只有在接受slot的时候才可以用asking,
    4. None主要是说,现在一切正常,直接给我发命令吧,这个意思