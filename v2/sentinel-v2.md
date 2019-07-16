sentinel
==
- 在redis.c之中,可以发现,有如下几个调用关系
    1. `serverCron`--->`sentinelTimer`
    2. `main`--->`initSentinelConfig` & `initSentinel` 进行初始化
    3. `main`--->`sentinelIsRunning` 让sentinel动起来

epoch
==
- sentinel在实现的时候,其实epoch还是比较复杂的,定义了太多的epoch...然后也复用了部分epoch,比较...难以揣摩究竟是啥意思~~
- 在sentinel中有各种的epoch,本质上还是为了实现一个raft状态机的作用的.其中`failover_epoch`是唯一一个可以单增的epoch,`current_epoch`会在`voteLeader`这个函数里面检测当前的epoch能否支持更改数据(其实就是term的作用啦,更大的term可以更改数据的)并且实现被动的增加.
```
                        (传播获得当前的epoch的值,
                        当比failover_epoch小的时候更新,
                        并且会记录对应的投票方向于
                        本sentinel的leader_epoch中)
                        (唯一做比较的字段),
                        相同时发送当前leader_epoch,
                        更高时更新leader_epoch,
                        更小时拒绝 
                         
                          ->current_epoch
 ____                    /            
|    | +1(当发现客观下线) /           
|    v                /(广播is_master_down_by_addr)
|    failover_epoch
|    |               \
 ____                 \
                        ->leader_epoch

                        (使用__HELLO__信道进行通信
                        获取其他sentinel的投票),
                        实现 n/2+1 leader选举
 ```
- `sentinelRedisInstance`中有的各种花式的epoch : `config_epoch` , `leader_epoch`, `failover_epoch`
  - 在redis里面,其他的sentinel也是在内部标识为一个`sentinelRedisInstance`对象
  - 在这个地方,一定要注意在sentinel的内部标识里面,其他的sentinel本质上也是一个sentinelRedisInstance,因此,在rsi表示的对象不一样时(sentinel或者master),有一定可能出现字段复用(同一个字段在不同时候表明的含义不一样)(其实复用一个字段节省不了多少...还容易难理解诶~~):`leader_epoch`是不幸被复用的字段,在master和sentinel对象(rsi)作用完全不一样~
  - `leader_epoch`:每一次在修改`leader_epoch`的时候会同时的记录下runid(唯一标志一个redisInstance或者sentinelInstance,在启动的时候分配),通过这样的方式,可以知道在某一个term(raft术语,就是epoch),该票的投向(runid表明的哪一位接受到了投票),在rsi里面,sentinel与master对应的作用不一样
    - 记录在sentinel里面,表明这个sentinelInstance投票给对应的runid了,这个投票的记录是在处理helloMessage的时候处理到投票信息进行更改的(也就是sentinel集群互相沟通的位置),在归票阶段(getLeader函数里面),本sentinel会查看各个sentinelINstance的投票方向,将超过(n/2+1)门限的sentinel最为master,执行failover
    - 记录在一个master里面的时候,表明本sentine关于这个master下线,投票的方向是哪里,目的是为了记录在本epoch已经投票的方向(一个epoch只能投票一次嘛),啥时候可能会用到这个记录呢?在主动请求投票的时候,`is-master-down-by-addr`,投过票的家伙直接甩一个之前的投票记录就可以了.
    - `is-master-down-by-addr`:一个命令,主动发送的时候,runid为*表明是客观下线的问询(pdown--->odown的转换).runid为一个int的时候,表明是投票请求的信息,请求的结reply里面(有可能之前投过票呢).
    - 投票成功,调用voteLeader函数,**将对应的epoch与请求投票(candidate)的runid记录到已经失效(处于odown状态)的master里面**(当然也可以用来记录其他sentinelINstance的投票记录什么的)
    - (1.`is-master-down-by-addr`(命令)   2.`getLeader`)`sentinelGetLeader`(两个可能的调用者调用)--->`sentinelVoteLeader`(一个调用者) 
  - `failover_epoch`:当前failover的epoch,是一个master(sri)拥有的一个属性,在开始failover的时候从sentinel当前的epoch单增获得的`++sentinel.current_epoch`,当failover结束的时候,转变为当前sri的`config_epoch`,也就是说:`ri->master->config_epoch = ri->master->failover_epoch`.
- `current_epoch`中 : `current_epoch`的本质其实只有一个,就是,用来做比较的,当前的epoch和发送过来的epoch比,能不能有修改epoch守护的数据的权限(和epoch一起发送的数据),对于这个地方其实就是能不能投票啦,主要看voteLeader里面,请求投票方的epoch(其实是failover_epoch)不能比自己的低(自己的current_epoch)
  - `current_epoch`的本质修改来源(增加)只有一个,也就是`failover_epoch`(epoch里面只有它一个可以增加的):
    ```C
    char *sentinelVoteLeader(sentinelRedisInstance *master, uint64_t req_epoch, char *req_runid, uint64_t *leader_epoch) {
    ```
    在这个地方,`req_epoch`其实就是当前的`failover_epoch`,当current_epoch更小的时候,会主动的更新`current_epoch`
  - 其他主动进行更新的地方是为了在下线之后跟上状态机的步伐,由于这个地方只是选选主,并没啥数据一致性传播的需求,因此,本质上改完`current_epoch`跟上最新的版本就好了:
    1. hello_message:从其他的sentinelInstance那里发现直接落后了,怎么办了,直接更新一下其实就可以了
    2. current_epoch命令,不用说了吧~~
    3. voteLeader中的req_epoch,由于在接受`is_master_down_by_addr`命令时(作为follower时)与getLeader(作为master时,会主动投自己一票)会调用voteLeader函数,这个地方也会检查一下`current_epoch`:主要是命令的时候,看一下自己是不是很落后(当然,请求的epoch很落后当然也不会投票给他的,23333)
- `config_epoch`似乎是一个master的配置epoch的感觉,目前还没有读到对应的源码,感觉上来说是一个解决断线之后出现问题的epoch,在处理hello报文的时候,可能会主动的发现当前config_epoch,然后重新的更新当前的master
  - 令人窒息,config_epoch就是在刷新函数的更新,调用关系为L`sentinelInfoReplyCallback`--->`sentinelRefreshInstanceInfo`,当发送info信息之后,将failover_epoch复制到了config_epoch里面~~(用来干啥呢,保存具体的信息吗orz)
- 所有的epoch都可以通过命令修改,但是修改成功的前提条件是必须要大于当前的epoch(不然raft协议其实就失效了)

initSentinel
- 首先,替换当前的命令表,seninel模式只支持少数几种模式的命令(当然,还有一些比较特殊的命令啥的),其实主要的命令还是那几个:ping,sentinel,info,其余的命令主要是实现pubSub之类的
  - 命令的实现是一个字典
  - 主要是由于在sentinel模式中,当前的信息交互其实是通过主机的pubSUB接口实现的.因此,被支持是天经地义的了
  - 使用pub,sub,多个sentinel进行消息交互,判定主观下线,并且选举投票,实现failover功能
    ```C
    struct redisCommand sentinelcmds[] = {
        {"ping",pingCommand,1,"",0,NULL,0,0,0,0,0},
        {"sentinel",sentinelCommand,-2,"",0,NULL,0,0,0,0,0},
        {"subscribe",subscribeCommand,-2,"",0,NULL,0,0,0,0,0},
        {"unsubscribe",unsubscribeCommand,-1,"",0,NULL,0,0,0,0,0},
        {"psubscribe",psubscribeCommand,-2,"",0,NULL,0,0,0,0,0},
        {"punsubscribe",punsubscribeCommand,-1,"",0,NULL,0,0,0,0,0},
        {"publish",sentinelPublishCommand,3,"",0,NULL,0,0,0,0,0},
        {"info",sentinelInfoCommand,-1,"",0,NULL,0,0,0,0,0},
        {"shutdown",shutdownCommand,-1,"",0,NULL,0,0,0,0,0}
    };
    ```
sentinelIsRunning
==
- 本质是为了启动sentinel,在main函数里面调用的嘛
- 生成一个+monitor时间,不知道是用来干嘛的orz

sentinelTimer
==
- 看到上面的有啥感想呢,没货啊~~只能说明一点了:sentinel本质是一个时间事件序列,所以这个函数就是爸爸!!!
- 可能的重点: 1.ping pong与计数维护啥的 2. raft状态机及其变迁(其实看上面的epoch已经知道了所有的流程了)  3. failover状态机以及变迁  4.
- **一定要注意的一点重点，就是redis本质上是通过时间函数定时触发事件与事件触发的模式设计的，即单线程的reactor模式，因此，对于sentinel，这个替代部分redis部件的组件，他的本质其实仍然是事件处理结合时间定时处理函数*
- 在这个函数里面,重要的函数主要只有一个: `sentinelHandleDictOfRedisInstances`,对当前的master字典进行遍历,执行sentinel相关的操作,遇到有问题的master,直接让他GG吧
- 注意:在sentinel的对象里面,包含关系是这样的:sentinel(本机)--->masterDict(本sentinel守护的master们)--->sentinelDict(和本sentinel一起守护master的那群家伙)&&slaveDict(看字)
- 定时操作:`sentinelHandleRedisInstance`
- 搞定salve与sentinel:`sentinelHandleDictOfRedisInstances`进行递归操作(本质上是让大家都雨露均沾到定时操作)
- 搞定故障迁移:`sentinelFailoverSwitchToPromotedSlave`

sentinelHandleRedisInstance
==
- 真正的关键函数,负责了所有的定时操作,上面一大堆做包装的东西
1. 使用函数`sentinelReconnectInstance`对下线的机器(DISCONNECT状态)进行重新连接,主要连接两个部分cc(伪客户端连接),pc(订阅消息连接) ,DISCONNECTED主要的由来于新建一个instance与进入客观下线的状态(subjectivly DWON),在每一个timmer的时候进行重新连接下线主机,有助于...emmm,网络抖动的时候不至于一直在faiover,新建一个新的instance就直接甩到Diact里面就去了,不用手动设置一些连接啥的,等到下一个时间事件触发的时候自然会连接的(很方便的嘛,但是要用熟悉~~)
   - 这个来了一大堆的回调函数什么的~~但是主要是在连接的时候搞一个什么的,其他懒得写了,看一个函数:`sentinelReceiveHelloMessages`吧,主要是订阅pubSub的函数,和hello信道有关系,hello信道其实只在连接的时候有使用到:让大家可以知道当前这个主机(slave)在哪一个master下面,或者另一个sentinel也在监视这个master什么的
   - 只有master和slave可以创建和本seninel创建一个连接pubSub的信道,这个信道是本sentinel和master连接的(master和本sentinel连接)(slave的话使用master的ip地址创建sentinel与master的信道)
   - 就是说,连接上的都是master的信道,而且都是seninel连接到了pubSub
   - 这里的callback是永久的callback一旦有信息来了就一定要使用这个callback的(在这里,是保证只接受hello信道并且调用处理函数)
3. 婊气函数:`sentinelSendPeriodicCommands`主要发送一些命令啥的来监控: PING、 INFO 或者 PUBLISH 命令
4. 对每一个instance使用函数进行客观下线检测:`sentinelCheckSubjectivelyDown`
5. 对于master,我们要: 
   1. 检测是不是客观下线了:`sentinelCheckObjectivelyDown` 
   2. 发送下线报警:`sentinelStartFailoverIfNeeded`&`sentinelAskMasterStateToOtherSentinels`
   3. 下线状态转移:`sentinelFailoverStateMachine`

sentinelReceiveHelloMessages
- 其实也就只有一个地方对这个函数进行了使用,就是在连线的时候
- 这个本质是调用函数`sentinelProcessHelloMessage`的一个封装的感觉,没啥...特色在的
- 下面都讲这个回调函数了吧...
- 这个回调函数的作用非常的直白,直接就是使用hello报文信息发现当前是sentinel的变化,将新的sentinel加入当前的list里面(sentinel是针对master而言的,本质上一个sentinel组只会去守护一个master,不可能有不同的sentinel集合来关注(因为没有实现multiRaft啦)(当然...好无语的哟~~))
- 然后是更新可能发生变化的current_epoch与master.congig_epoch之类的(其实在单raft的情况下,为啥要一个config_epoch哟)

sentinelSendPeriodicCommands
- INFO发送给salve或者master:当第一次发送命令或者长期不上线的情况(最高优先级),使用回调函数:`sentinelInfoReplyCallback`
- 超过门限了发送PING命令:slave,master,sentinel人人有份(次高优先级)--->`sentinelPingReplyCallback`为回调函数
- 超过门限发送HELLO:通过hello信道发送(pub),也是人人有份(最低优先级),由于信道必须是master的,因此对于sentinel和slave,需要调用master的信道,然后调用回调函数:`sentinelPublishReplyCallback`
  - 需要发送的 : sentinel_ip,sentinel_port,sentinel_runid,current_epoch, master_name,master_ip,master_port,master_config_epoch.
- 两只回调函数本质上都是更新一下时间什么的,让我们可以知道什么时候收到了回复(毕竟...是异步的函数)

sentinelInfoReplyCallback
- 其实是一个包装函数,检测一下有没有出现传输上的问题啥的(比如redisInstance存在与否),之后就调用函数:`sentinelRefreshInstanceInfo`,接下来就直接解释这个函数了
- 分析info发过来的报文,对当前的snetinel信息进行更新
- 当前的runid,ip,port,config_epoch的变化
- 对当前新的slave进行更新操作,调用函数:`sentinelRedisInstanceLookupSlave`,`createSentinelRedisInstance`

sentinelCheckSubjectivelyDown
