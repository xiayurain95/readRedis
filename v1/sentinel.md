sentinelGetLeader，当前sentinel节点获取leader是谁的函数（标号#）
* 统计当前票数，当前有收到leader投票信息的时候时候，投最多的的一票，否则投自己一票
  ```C
    //统计方式，在epoch相同的情况下，查看每一个sentinel instance的leader，并统计leader票数
          if (ri->leader != NULL && ri->leader_epoch == sentinel.current_epoch)
            sentinelLeaderIncr(counters,ri->leader);
  ```
  ```C
  //计算当前各个leader票数最多的那一位
      while((de = dictNext(di)) != NULL) {

        // 取出票数
        uint64_t votes = dictGetUnsignedIntegerVal(de);

        // 选出票数最大的人
        if (votes > max_votes) {
            max_votes = votes;
            winner = dictGetKey(de);
        }
  ```
  ```C
  //如果现在还没有收到一个得票数的统计信息，自己加冕为王，emmm
  if (winner)
        myvote = sentinelVoteLeader(master,epoch,winner,&leader_epoch);
    else
        myvote = sentinelVoteLeader(master,epoch,server.runid,&leader_epoch);
  //sentinelVoteLeader()这个函数是投票给leader的函数，待会儿解析，记个标号#
  ```

sentinelStartFailoverIfNeeded
* 这个函数才是真正检查并且执行的failover的函数，他可以主动的发起一次failover，然后redisInstance的状态转变为failOver全家桶，但是。。。进行恢复操作的其实是leader，leader必须经过选举得到
* 各种姿势的状态
    > 首先是身份:
  SRI_MASTER  (1<<0)
 SRI_SLAVE   (1<<1)
 SRI_SENTINEL (1<<2)
    > 其次是单机节点上状态（主观，客观掉线）：
 SRI_DISCONNECTED (1<<3)
 SRI_S_DOWN (1<<4)   
 SRI_O_DOWN (1<<5)   
 SRI_MASTER_DOWN (1<<6)
    > 再来是 
 SRI_FAILOVER_IN_PROGRESS (1<<7) 
 SRI_PROMOTED (1<<8)            
 SRI_RECONF_SENT (1<<9)     
 SRI_RECONF_INPROG (1<<10)   
 SRI_RECONF_DONE (1<<11)     
 SRI_FORCE_FAILOVER (1<<12)  
 SRI_SCRIPT_KILL_SENT (1<<13) 
    > 重点了，一下是在整个集群范围内的，master的failover状态机,是SRI_FAILOVER_IN_PROGRESS的子项目：
    ```C
     SENTINEL_FAILOVER_STATE_NONE 0  
    // 正在等待开始故障迁移
     SENTINEL_FAILOVER_STATE_WAIT_START 1  
    // 正在挑选作为新主服务器的从服务器
     SENTINEL_FAILOVER_STATE_SELECT_SLAVE 2 
    // 向被选中的从服务器发送 SLAVEOF no one
     SENTINEL_FAILOVER_STATE_SEND_SLAVEOF_NOONE 3 
    // 等待从服务器转变成主服务器 
     SENTINEL_FAILOVER_STATE_WAIT_PROMOTION 4 
    // 向已下线主服务器的其他从服务器发送 SLAVEOF 命令
    // 让它们复制新的主服务器
     SENTINEL_FAILOVER_STATE_RECONF_SLAVES 5 
    // 监视被升级的从服务器
     SENTINEL_FAILOVER_STATE_UPDATE_CONFIG 6 
     ```
* 基本的思路的检查到一个SIR_O_DOWN的时候，发起failover
  - 这里注意一下了。。。SRI_O_DOWN与SRI_S_DOWN是由sentinelCheckSubjectivelyDown与sentinelCheckObjectivelyDown负责的
  - sentinelCheckSubjectivelyDown反正就是检查一下，不过会发送一个广播
  - sentinelCheckObjectivelyDown比较有意思
    > 首先，统计当前节点收到其他sentinel状态（在本机内部）,经过广播获得的消息，认为也是S_DOWN的
    ```C
        while((de = dictNext(di)) != NULL) {
            sentinelRedisInstance *ri = dictGetVal(de); 
            if (ri->flags & SRI_MASTER_DOWN) quorum++;
        }
    ```
    > 认为节点已经下线时，修改了节点当前的状态为O_DOWN，然后发送一个O_DOWN的广播就结束了
    ```C
           if ((master->flags & SRI_O_DOWN) == 0) {
            // 发送事件
            sentinelEvent(REDIS_WARNING,"+odown",master,"%@ #quorum %d/%d",
                quorum, master->quorum);
            // 打开 ODOWN 标志
            master->flags |= SRI_O_DOWN;
            // 记录进入 ODOWN 的时间
            master->o_down_since_time = mstime();
        }
    ```
  - 这个的本质其实就是封装了一下sentinelStartFailover函数，让sentinelStartFailover不要过快重启，以及确定当前在确定的状态里面
  
sentinelStartFailover

- 作用其实还是比较直接的： 
- 更新master的各种flag，表明SRI_FAILOVER_IN_PROGRESS（在故障转移），以及SENTINEL_FAILOVER_STATE_WAIT_START（在故障转移的第一个阶段里面），到了一个新的failover里面，当然需要一个新的term啊（epoch）
- **最重要的，这个函数是唯一一个更新全局epoch的函数，就是说，在满足需求的时候，epoch才会增加，然后会广播到这个主机的channel里面，监视这个master的所有sentinel都会监视到这个信息，然后会更新epoch**
    ```C
    master->failover_state = SENTINEL_FAILOVER_STATE_WAIT_START;
    master->flags |= SRI_FAILOVER_IN_PROGRESS;

    master->failover_epoch = ++sentinel.current_epoch;
    ```
- 由于是整个集群的状态发生了变化了，因此需要广播出去的，广播 1.我在failover 2.我的epoch是这个
    ```C
    sentinelEvent(REDIS_WARNING,"+new-epoch",master,"%llu",
        (unsigned long long) sentinel.current_epoch);

    sentinelEvent(REDIS_WARNING,"+try-failover",master,"%@")
    ```

sentinelProcessHelloMessage
- 非常重要的函数了，直接负责监听hello
- 发现一个新的sentinel节点
- 更新epoch，这个是更新epoch的唯一渠道，投票这件事儿本质上其实是command通道搞定的
- 现在master已经发生了变化了，更新新的master，把旧的直接换掉


理清楚顺序：
1. 首先，定期使用PING命令，从回复的PONG可以发现master好像现在不是很正常，进入SRI_S_DOWN
2. 对于进入SRI_S_DOWN（O_DOWN也算）的亲来说，使用sentinelAskMasterStateToOtherSentinels来问整个监视master的sentinel的亲友团，并且注册回调函数（sentinelReceiveIsMasterDownReply）来异步处理回复的信息啥的。**也就是说，其实节点的状态都是主动去问，知道的**
   1. sentinelAskMasterStateToOtherSentinels本质是在向命令通道一直bb，SENTINEL is-master-down-by-addr的那个函数，当现在还不处于O_DOWN状态的时候，单纯的问一下大家的意见
   2. 当处于SENTINEL_FAILOVER_STATE_NONE（准确的说是在检测到O_DOWN之后，状态机已经开始failover之后（就是在进行故障转移了））的时候，不是问意见了，而是要求乃们投票，这个函数有两个作用了，**一个是问询（S_DOWN的时候），这个master是不是有问题，两个是请求投票，在已经开始failover的时候**
   3. 回调函数可能接受到多个结果
        1. 回调函数接受到的是一个，我认为master状态的信息，回调函数几乎不咋处理，直接把对应的redis sentinel instance里面写一个，这个sentinel是这样认为的。方便后续从S_DOWN换到O_DOWN
        2. 接受到的是一个我请求对方投票，对方的回复的时候（runID 不是 "*"），证明我请求成功了，开心的在自己的小本本上记录一个，**我的票数加1，当然这位可能也投票给其他人了，没关系，投票了就直接写到master sentinel的leader结构里面，监视这个master的sentinel现在认为他的leader是哪一位**，最后再有其他函数来统计啥的
   4. 投票有两个位置，一个是command收到了 SENTINEL is-master-down-by-addr命令，另一个是getLeader函数的位置


sentinel总的函数，sentinelTimer
- **一定要注意的一点重点，就是redis本质上是通过时间定时函数进行设计的，在时间定是函数中间加入了基于事件触发的模式，reactor，因此，对于sentinel，这个替代部分redis部件的组件，他的本质其实仍然是事件处理结合时间定时**
- 一个放到Cron的函数，由redis的ServerCron调用，看来是一个定时的函数，就是。。。时间时间函数
- 采用时间时间的方式设计，**可以十分方便的扩展以及修改功能**
  ```C
    sentinelCheckTiltCondition();

    sentinelHandleDictOfRedisInstances(sentinel.masters);
    //一些不相关的函数啥的
    ...
  ```
- TITL是sentinel记录时间，并且自己恢复的一种模式，检测sentinel可能出现的一些奇怪的地方
- sentinelHandleDictOfRedisInstances算是精华了，就是，我们定期执行的一些函数，如，PINGPONG检查啊什么的，本质是监视本机上实例（sentinelIntance）下的master（本sentinel监视的master），由于master的本质是一个sentinelRedisInstance，因此，可以顺便监视一下这个master下面的salve（**使用INFO与INFO返回信息进行监视**）和同事监视这个master的sentinel（**使用hello信道进行通信**）
  ```C
  void sentinelHandleRedisInstance(sentinelRedisInstance *ri) {
    sentinelReconnectInstance(ri);
    sentinelSendPeriodicCommands(ri);
    sentinelCheckSubjectivelyDown(ri);

    if (ri->flags & SRI_MASTER) {

        sentinelCheckObjectivelyDown(ri);

        if (sentinelStartFailoverIfNeeded(ri))
            sentinelAskMasterStateToOtherSentinels(ri,SENTINEL_ASK_FORCED);

        sentinelFailoverStateMachine(ri);
        sentinelAskMasterStateToOtherSentinels(ri,SENTINEL_NO_FLAGS);
    }
  }
  ```
- 如上，对于故障转移最重要的函数就是
  - sentinelCheckObjectivelyDown(ri);
  - sentinelStartFailoverIfNeeded(ri)
  - sentinelFailoverStateMachine(ri);
  - sentinelAskMasterStateToOtherSentinels(ri,SENTINEL_NO_FLAGS);-被复用过两次，一次用来请求S_DOWN是否成为了O_DOWN,，另一个是请求选票（这个函数之后一定会被分开的）
    - 发送有各种的不同目的，在返回的时候，都有获得当前节点认为的leader的作用。（runid不为*，函数记录当前返回这个信息的redisInstance（sentinel）认为的leader）

sentinelFailoverStateMachine
- 一个传说中的状态机，主要用于在错误恢复阶段对集群状态进行恢复。
（即，SRI_FAILOVER_IN_PROGRESS）（在SRI_FAILOVER_IN_PROGRESS内部还有一大群其他状态，SENTINEL_FAILOVER_×××）
- 这个状态机本质上十分的简单，由于他本质上是处于sentinelTimer这个函数中，这样状态转移本质上是反复的进入Cron里面的
- 状态 serviceCron ---> sentinelTimer ---> sentinelFailoverStateMachine ---> sentinelStatProcess（等待下一个循环）
    ```C
    void sentinelFailoverStateMachine(sentinelRedisInstance *ri) {
    redisAssert(ri->flags & SRI_MASTER);

        // master 未进入故障转移状态，直接返回
        if (!(ri->flags & SRI_FAILOVER_IN_PROGRESS)) return;

        switch(ri->failover_state) {

            // 等待故障转移开始
            case SENTINEL_FAILOVER_STATE_WAIT_START:
            sentinelFailoverWaitStart(ri);
            break;

            // 选择新主服务器
            case SENTINEL_FAILOVER_STATE_SELECT_SLAVE:
            sentinelFailoverSelectSlave(ri);
            break;
        
            // 升级被选中的从服务器为新主服务器
            case SENTINEL_FAILOVER_STATE_SEND_SLAVEOF_NOONE:
            sentinelFailoverSendSlaveOfNoOne(ri);
            break;

            // 等待升级生效，如果升级超时，那么重新选择新主服务器
            // 具体情况请看 sentinelRefreshInstanceInfo 函数
            case SENTINEL_FAILOVER_STATE_WAIT_PROMOTION:
            sentinelFailoverWaitPromotion(ri);
            break;

            // 向从服务器发送 SLAVEOF 命令，让它们同步新主服务器
            case SENTINEL_FAILOVER_STATE_RECONF_SLAVES:
            sentinelFailoverReconfNextSlave(ri);
            break;
        }
    }
    ```
epoch
- **源头**，唯一一个会导致epoch增加的函数只有一个---》sentinelStartFailover，他是被sentinelStartFailoverIfNeed来把守的
- **广播**，有三个地方存在，
  - sentinelProcessHelloMessage
    -这里是hello信道的信息，是广播渠道的来源，他的来源是发现后两位发出来的新的epoch信息
  - sentinelVoteLeader（会发送event）
  - sentinelStartFailover（当然要发送epoch鸭） - 一切开始的地方，恩
- **品种**，有很多品种在哟
  - sentinelRedisInstance里面的
    - config_epoch，配置纪元，用于实现故障转移，主要是说明当前的master的ip啊，port啊什么的，其实是用于解决断网的sentinel同步信息用的，所以是来自于failover_epoch，就是。。。failover_epoch稳定了之后就跑到这里面来了
    - leader_epoch;领头的纪元
    - failover_epoch；当前执行中的故障转移的纪元,当前在执行故障转移的时候，就是这个了，主要有啥选举投票啊，什么的
  - sentinelState里面的
    - current_epoch；本sentinel当前的纪元 

sentinelFailoverWaitStart



SENTINEL_FAILOVER_STATE_WAIT_PROMOTION
 - sentinel instance发现记录的是salve 但是INFO说他是master,发生状态上的转移，SENTINEL_FAILOVER_STATE_RECONF_SLAVES

sentinelLeaderIncr
--
- 这一个主要实现对sentinel选票的增加，也就是说，投票这件事儿基本被它决定的
- 只在sentinel Get Leader被调用过啥的

sentinel 主要是在一个时间周期完成某项任务
- 本函数是替换主函数中的时间周期程序，因此本来就是按照时间周期完成一些操作啥的 
- ping，info，publish 命令，周期完后才能，实现
  - 对网络的监控，下线的监控
  - 对实际sentinel master与salve的信息监控，防止sentinel下线之后的消息滞后
  - 对sentinel之间的互相连接，hello message好像是互相发现啥的，hello 信道嘛，主要传送的是以本sentinel为主要的对象传送sentinel的以为的信息。
    - 这就是为啥我们需要config_epoch来防止一些老旧的信息让人自闭了。

sentinel failover状态机：
1. SENTINEL_FAILOVER_STATE_WAIT_START
   - 作用是准备进入failover状态
   - 选举函数getLeader就在里面了
     - 选举函数的作用不是请求投票，请求投票是上一个函数的作用，sentinelAskMasterStateToOtherSentinels，这货会发送一些命令字到其他的sentinel里面，包含请求其他主机对本instance状态的看法以及请求投票两个。（之后一定会分开来的，这两个功能）
     - 选举函数是处理candidate的主要逻辑存在的地方，如果本candidate没有收到一些请求投票的信息MASTER_IS_DOWN_BY_ADDR(好像)，他主动的把票投给最多的（如果他还没有投票啥的）
       - 怎么算是不是投票过了，主要看epoch，在一个epoch的值里面只可以投一次，如果你的epoch等于你要投的epoch，抱歉，你之前已经投过票了 sentinelVoteLeader 这个函数说了算的
     - 其实嘛，这个地方是唯一一个instance可以成为leader的地方（话说成为leader的随机时延呢？？？）
       - 这个时延貌似在redis里面貌似是没有的，只有一个过期时延，在固定的时间没有一个爸爸出现，就直接选择自闭了（放弃选主了）
          - 一旦放弃。。。后果其实就是。。。所有的都放弃了，重新到SRI_ODOWN状态开始接着走下去
    - 一切成功了，ok，进入SENTINEL_FAILOVER_STATE_SELECT_SLAVE

2. SENTINEL_FAILOVER_STATE_SELECT_SLAVE
    - 主要使用的函数是sentinelFailoverSelectSlave（看名字就是选一个差不多的服务器嘛
    - 没啥可以说的，下一个状态就是SENTINEL_FAILOVER_STATE_SEND_SLAVEOF_NOONE，仅此而已~
  
3. SENTINEL_FAILOVER_STATE_SEND_SLAVEOF_NOONE
   - 没啥可以说的，直接名字吧