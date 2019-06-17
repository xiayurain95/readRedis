replication
==
- 首先,来看一下在redis.c里面,为了支持replication,作了哪些妖,由于replication本质上是一个定时任务,不存在文件事件的触发,最重要的就是时间序列任务了
    1. 首先是cron,定时任务基本都会涉及到里面调用的replicationCron
    2. 时间序列第二弹beforeSleep里面调用的replicationFeedSlaves
    3. PSYNC貌似算是一个十分有特色的地方了
    4. 初始化函数init,初始化了系统里面的replicationScriptCacheInit
        - 目的是搞点啥script缓存之类的~~ 就几行...
    6. call函数实现了redisCLient对函数的调用,call中命了的replicationFeedMonitors
    7. slave与master建立连接,主要注册了一个syncWithMaster回调函数
    8. 针对master与slave的同步,是通过伪终端的方式,可能存在调用`syncCommand`(sync与psync命令处理)的情况(针对客户端,也有一个类似的函数`replconfCommand`
    9. `updateSlavesWaitingBgsave`:bgSave回调函数扩展(这个函数默认的回调函数是`backgroundSaveDoneHandler`,唔,它插了一个函数进去
    10. `slaveofCommand` 一样的,也是一个命令,主要实现对salve of命令的处理过程(就是...设置对应的master啦)

- 这个是replication可能出现的各种状态flag,REPL其实就是replication的缩写,注意一点就是,对于一个双方的连接,master与slave的视角其实是不一样的,因此状态转移的也不太一样的
  - `REDIS_REPL_NONE`代表的就是不是一个salve服务器,是一个master
    ```C
    /* Slave replication state - from the point of view of the slave. */
    #define REDIS_REPL_NONE 0 /* No active replication */
    #define REDIS_REPL_CONNECT 1 /* Must connect to master */
    #define REDIS_REPL_CONNECTING 2 /* Connecting to master */
    #define REDIS_REPL_RECEIVE_PONG 3 /* Wait for PING reply */
    #define REDIS_REPL_TRANSFER 4 /* Receiving .rdb from master */
    #define REDIS_REPL_CONNECTED 5 /* Connected to master */

    //服务器的视角
    #define REDIS_REPL_WAIT_BGSAVE_START 6 /* We need to produce a new RDB file. */
    #define REDIS_REPL_WAIT_BGSAVE_END 7 /* Waiting RDB file creation to finish. */
    #define REDIS_REPL_SEND_BULK 8 /* Sending RDB file to slave. */
    #define REDIS_REPL_ONLINE 9 /* RDB file transmitted, sending just updates. */

    /* Synchronous read timeout - slave side */
    #define REDIS_REPL_SYNCIO_TIMEOUT 5
    ```

Cron函数
==

针对从服务器
--

- 主要其实在处理状态机中的状态各种的变换,主要是由于当前处于异步工作的模式,需要有大量的连接状态检测的任务处理问题
  - 举一颗栗子吧,连接完成后,由于异步连接被fctl设置成了O_NONBLOCK的形式,直接调用read和write会返回-1,而不是阻塞,因此!!!连接是异步的必须要有回调函数,现在问题来了,回调函数啥时候调用我们也不知道诶.所以状态切换(对应连接的过程其实是PING PONG ,每一个状态的转换其实是需要手动写函数去检查的), cron里面主要是一大堆的if,else ~~
- 下面为了方便理解,我重新组织了一下叙述的顺序 
 
- 首先,状态是`REDIS_REPL_CONNECT`,由于这个状态是slave专有的,所以不用检测是不是slave这种简单的问题,当出现这个状态的时候,cron直接需要调用connectWithMaster负责连接master
  - connectWithMaster,首先建立连接,使用函数anetTcpNonBlockConnect,进行异步的连接建立过程
  - anetTcpNonBlockConnect函数所在文件调用了头文件`#include <errno.h>`,可以产生errorno(linux特有的),在调用connect函数l的时候`connect(s,p->ai_addr,p->ai_addrlen) == -1`,由于异步的关系,会直接返回-1,在这里是通过检测errorno来实现异步的`errno == EINPROGRESS && flags & ANET_CONNECT_NONBLOCK`,即`EINPROGRESS`是异步过程中,直接可以返回当前在进行中状态的样子
  - 关于fd的问题,fd是unix套接字产生的,也就是说,在定义一个套接字(调用`socket`函数的时候就会自动的产生套接字),之后还需要绑一下地址啊,fctl设置异步啊,连接啊什么的,但是套接字是可以直接返回的.
  - 以上是为啥我们的异步需要不断的检测状态的原因啦
  - 接下来是注册我们的异步套接字到epool上面,在注册的时候注册了一个回调函数 :syncWithMaster
 
- 修改当前的状态为REDIS_REPL_CONNECTING
  -  syncWithMaster
  -  首先是下检测一下自己是不是slave,如果不是slave,就直接退出了,然后再检查一下目前socket设置有没有问题什么的
  -  然后是连接connect过程结束的回调,这个时候需要一次PING PONG过程,
(在发送PING的时候还各种作妖,同步发送啊,还有取消可写监听什么的(为了稳定???)),同步发送还蛮有意思的,在这个地方的时候,由于套接字其实是工作在异步状态的,因此没办法同步的,所以怎么来同步呢?--->`aeWait(fd,AE_WRITABLE,wait)`,想不到吧,其实就是简单的还是在异步上面,只不过过了一定的时间进行一次检测而已(说白了其实还是是异步的),然后把状态转移一下--->`REDIS_REPL_RECEIVE_PONG`
  - 当发现是`REDIS_REPL_RECEIVE_PONG`状态的时候,主动的取消ev的`AE_READABLE`状态,其实结合前面的叙述,当前对这个套接字的监听任务已经结束了(两个事件都统统的GG了),然后验证一下是不是PONG(可能出现一个奇奇怪怪的AUTH),然后!!!**不返回,直接执行下面的命令**(这点的逻辑结构过于感人~~)
  - 现在问题来了,我们现在早早的就把我们ev事件监听给关闭了,真的好吗?--->函数不会关闭 fd 。当部分同步成功时，函数会将 fd 用作 server.master 客户端结构中的文件描述符。
  - **_接下来的部分其实都是在收到PONG之后发生的,没有直接返回的内容了_**
  - 在现在这个时候,主要是使用的函数`sendSynchronousCommand`来充当一个为终端,发送一系列的命令搞事(主要是现在认为当前阶段主机已经处于稳定的状态里面了)
  - 然后是AUTH命令发送(为啥放到这里...好奇怪)
  - 发送当前主机的port(`"REPLCONF","listening-port"`命令)(估计是运行到这个位置可以认为当前的连接的链路已经稳定了吧)
  - 调用函数`slaveTryPartialResynchronization`,这个地方主要是担心,在出现ping的问题的时候会主动的断线,又必须重新的进行一轮新的sync(angry!),所以出现了一个pSnyc的资瓷,这个地方就是检查一下能不能pSync一下...这个函数功能比较复杂,其实是实现断线自动同步的主要函数,在后面专门的描述一下
    - 三种可能的返回值: 1. 可以psync 2.可以psync但是现在必须要全备份一次  3. 上古系统,只能sync
  - 检测一下返回值,如果可以支持psync就直接执行吧(前面其实算发送了一个命令了)
  - 不支持psync还是必须再次发送一下命令的(这一次是sync了)
  - 创建一个新的文件,作用是存储当前从master那边过来的rdb文件,当然也是异步的啦,注册了事件处理器(可读事件),readSyncBulkPayload(后面写,恩)
  -好的,结束了
    
- cron函数对从服务器的其他的安排(各种中间状态)
  1. 连接主服务超时了--->不连了--->转换到`CONNECT`状态 (套接字处理不说了)
  2. RDB超时?--->不传了--->处理两个fd(socket和rdb文件描述符),转换到`CONNECT`状态
  3. 主服务器下线了(为啥需要`server.cached_master`)--->不连了(`freeClient`一下)--->转换到`Connect`状态
  4. 在每一个cron周期的时候调用`replicationSendAck`,向master发送当前的偏移`c->reploff`(注意:在redis的设计里面,master是当做一个终端来处理的,所以在这个地方,其实是相当于向master进行回复)
    
     
- `slaveTryPartialResynchronization`函数
  - 首先检测一下`server.cached_master`存在不(这个值是freeClient保存的),保存这个只就表明了之前其实是连接过master的(并且有过同步的过程)
    1. 有的话就使用增量备份(pSync)
    2. 保存这个值的目的是`server.cached_master->reploff+1`,在这个为终端里面存有当前发送的offset 
    3. 没有就使用全量备份(pSync ? -1,一个特殊的命令)  
  - 发送命令还是一样的,还是伪终端的方式,调用函数`sendSynchronousCommand`发送psync,有好几种可能的回复:
    1. FULLRESYNC: 字面意思
    2. CONTINUE: partial resync
    3. ERR: 一看就是不支持,需要用旧版的sync函数
  -  FULLRESYNC:由于要进行全重备份,可能分两种情况:
     1. 当前发生了和master的切换(换了一个master),之前的备份统统丢失了吧
     2. 之前没有和一个master连接
     3. 可能还有一些中间情况吧
     -  最后的结果就是,本质上需要记录run id,repl_master_initial_offset,在这个函数里面,对这些项进行parse & 写入,当然还要验证一下合法性.
     -  最后,由于当前其实本质上cached_master已经没用上了(可能发生了变化),对之前mster的拷贝(`replicationDiscardCachedMaster`函数清除`cached_master`进行清除工作
        - 一个令人厌恶的调用树:`syncWithMaster`--->(本质上是第一次调用或者短信重连的情况,需要判定pSync)`slaveTryPartialResynchronization`--->(判定结果是,当前的master和之前的master本质上没啥关系(主要靠着run id来区别))`replicationDiscardCachedMaster`--->(发生这样可怕的事情之后就需要吧之前的cached_master直接销毁掉)`freeClient`--->(销毁函数==)`replicationCacheMaster`--->(当master掉线的时候调用)`replicationHandleMasterDisconnection`(离开replication,进入networking,可以看出来当前replication源码分离度还不是足够高)--->(当slave下的salve出现问题,调用)`disconnectSlaves`--->`freeClient`
        - 在`replicationDiscardCachedMaster`里面使用了freeClient,清理一个client需要什么呢(watch,频道与订阅,套接字与事件引擎,缓冲区,客户端链表,事务) ,为了给予replication支持,在这部分增加代码`replicationCacheMaster`实现,由于清除了一个client,***先检测一下,当前清除的是哪一个,如果是现在的mster的话,将当前的master备份一下,直接放到cache_master里面(看到了嘛,是需要在清理当前master的时候才会放到缓存里面的(主要是为了支持pSync))***
        - replicationCacheMaster本质上也是清理master的,会从客户端链表中移除master也会重新清除当前的fd,并且关闭ev中的回调函数.(本质上就是整个连接直接关掉了吧~~),**这个函数被调用的本质其实是master掉线了,所以才需要放到cache里面去安慰一下**
  - 收到的是`CONTINUE `;表示我们还在上面可以继续进行pSync
    - 发生这个的主要原因其实是断线重连,因此首先需要重新连接一下网络`replicationResurrectCachedMaster`
    - 利用上面这个函数修改各种状态,立各种的flag,把cached_master直接放到现在来用master里面
    - 现在当然是有各种疑问的:比如,fd呢,不是已经被清除了吗,怎么又出现了了,的确是被清除了,现在的fd本质上是一个新的fd,这个函数是在执行了PING PONG之后调用的,因此是建立连接成功之后调用的,什么时候需要简历连接哟?没有连接或者...连接断开之后重连的时候.
      - 所以现在会有一个newFd,在恢复之前cached_master的时候,将newFd顺便回复到对应的位置上面去 
      - 状态切换--->`REDIS_REPL_CONNECTED`
      - 重新把伪客户端加入到列表里面去
      - 注册回调函数,在可读的事件触发的条件下调用readQueryFromClient函数,将文件读入
      - 注册可写,调用函数sendReplyToClient,现在可写了嘛
  - **_重点:一旦收到的是contine就直接返回了,使用命令的方式对当前的同步状态进行跟随,所以...往下的其实都是全同步的状态了(不支持啊,或者换master了啊之类的)_** 
  - 收到的是`ERR`:执行replicationDiscardCachedMaster函数,释放cached_master,由于当前master不支持pSync,cached_master其实也没啥作用了,直接调用对应的函(`freeClient`)释放它. 

- `readSyncBulkPayload`函数
  - 这里的BULK其实就是RDB的意思啦,主要是指那种一堆的字符串,那种       
  - 首先先检查一下是不是符合redis传输协议,可以点击一下看看--->[redis传输协议](https://redis.io/topics/protocol)
  - 在符合传输协议的情况下,貌似规定了一下:发送必须分两个阶段进行.
  1. 首先先发送一下rdb大小,因此首先检测一下rdb大小接受到没有,其实目的是更新`server.repl_transfer_size`这个值,`strtol`首先数字与字符串的转换功能
  2. 发送对应的文件序列,当然涉及到很多内容了,下面详细解释一下
  - 由于异步(epoll)以及大文件传输(TCP分片分流),因此每一次传输一个片,需要检测一下传输到什么位置上了,记录一个参数说明当前读取位置:`server.repl_transfer_read`,计算剩余字符
  - 读取并且使用rdb文件描述符`server.repl_transfer_fd`写入当前读入数据(socket的文件描述符是`server.repl_transfer_fd`),每接受到8M的时候,调用`rdb_fsync_range`函数进行一次fsync
  - 传送完毕了,开始各种神奇的操作
    - 首先,现在只能在rdb进行完全恢复的状态,因此,需要将之前的数据库进行清空`signalFlushedDb`将所有的任务标记为脏数据,`emptyDb`清空数据库
    - 在进行恢复的时候先阻断可能的恢复读操作`aeDeleteFileEvent`删除监听可读状态
    - `rdbLoad`载入数据,之后在看rdb的时候在说吧~~
    - 现在其实已经恢复完成了,和之前的操作一样,我们现在有一个newFd,从newFd里面建立一个新的`server.master`,方便可以使用命令行进行实时的命令复制(其实psync也是命令复制的一种吧),`server.repl_master_initial_offset;`可以检测当前主机对psync(一个引入的多余字段嘛,不兼容的话就是直接为默认值了)是不是支持的
    - 由于现在更新可能是由于换了master导致的,解决一下aof的问题,开始aof的,删除之前的aof文件并且重启一下(现在数据库完全都不一样了嘛).函数`stopAppendOnly`和函数`startAppendOnly`
    - 好了,这个函数去世了

针对主服务器
--
- 有slave就算,可以进行一些比较神奇的操作,比如...那种slave下面继续接slave之类的
- master职责,心跳报文发送.其实,在当前这个模式里面,master的本质其实是一个client(redis 伪终端),所以这个地方相当于客户端向服务器发送心跳报文.在redis的实现里面,心跳包文是发送`PING`,调用函数`replicationFeedSlaves`(重要函数,后面会慢慢写一下)
- 对于正在对rdb文件进行一些操作的slave(`REDIS_REPL_WAIT_BGSAVE_END`状态与`REDIS_REPL_WAIT_BGSAVE_START`状态),跳过,但是发送一个空行(这里直接调用了read原语了),主要作用可能是使tcp keepalive不失效
- 进行掉线的处理,调用`freeClient`
- 处理master角色转换,停用backlog:调用`freeReplicationBacklog`函数
- 处理config中 min-slaves-max-lag ,调用函数`refreshGoodSlavesCount`计算当前slave数量,并写入server全局值里面(其实就是查一下`REDIS_REPL_ONLINE`状态客户端数目)

两种缓存
- backlog:实现psync的buffer,本质上是一个环形buffer,每一个主从复制周期的本质其实是一个一个master向多个slave发送对应的命令.因此可以通过对应的偏移计算出psync的对应位置.使用backlog的对应命令,可以实现psync(超过这个环形log的位置...就直接GG了)
- buffer:每一个replication slave都对应有一个buffer,主要作用是在一定的时候,可能当前的命令需要一个缓存(如:进行rdb备份和恢复的时候,将新的命令放入buffer中,分两段进行传输(因为当前的rdb可能非常大,实际会有很多命令传输(在rdb 备份为daemon模式的时候)))

replicationFeedSlaves
- 万恶函数,将对应的命令传播到slave上面去
- 在redis.c中调用关系为`propagate`--->`replicationFeedSlaves`  
- `propagate`同样负责将命令传播到aof文件上,这个函数在redis中调用非常频繁,是直接关系到redis replication能工作的关键
- 首先是数据库的指定:当当前masterdb于slavedb不同的时候,使用master的db值`dictid`与slave的db值`server.slaveseldb`不同,使用`dictid`构建一个发送给slave的`selectcmd`命令字符串(哎呀,其实就是SELECT字符串啦)
- 然后保存一下当前的命令到backlog里面,调用关系:`feedReplicationBacklogWithObject`(对当前对象进行序列化(有的对象中有指针,对其进行析构操作))---> `feedReplicationBacklopy`(后面说一下这个东西)
- 发送给客户端,切换服务器id流程结束
  - `feedReplicationBacklog`:实际上操作了backlog,是实现psync上的关键函数之一(主要的算法也在里面),本质上是存了一个字符串到对应的backlog上面,并且更新各种replication的值,实现对psync的支持 
  - `server.master_repl_offset`(当前全局字符串的位置)的值,每一次写入一个值就一定会更新一回,本质上是单增的(可以假设),根据backlog长度可以与本值可以计算得到某一个字段是否在backlog字段中,获取能否进行psync的可能性(记得之前的返回吗 ,可能就是fullsync与continue的分野了吧)(backlog保存的是命令什么的,你的字段已经在命令以外了,没办法进行恢复的)
  - `server.repl_backlog_idx`,总所周知,由于bakclog本质上是一个环形字符串(robin-round),尾字符串本质上是必须在后面的,因此必须要有一个头指针的存在,而且每一次读写中都必须对头指针进行相应的操作
  - `erver.repl_backlog_histlen`:在这个地方源码注解其实是有误的(或者不完全的).这里其实主要用来解决当前还没有写满一次backlog(还没有造成满员后重新设置头指针的状况)
    - 用来计数:确认当前字符串的结尾位置.由于当前是一个循环数组的模式,当首次历史写入超过字符串长度时,这个字段的作用其实已经结束了.
  - `server.repl_backlog_off`:结合上面的字段(其实主要是刚刚启动循环的时候),计算当前的历史可用backlog位于`master_repl_offset`(全局字符串)的位置
- 接下来是把正式的command组织成协议的形式.直接写入到backlog里面
- 写入之后其实就是发送命令的过程了,去除`REDIS_REPL_WAIT_BGSAVE_START`状态的服务器(马上这个服务器就会创建一个rdb文件(可能事件触发的时机在下一个Cron(redis循环里面))),剩下调用分发送reply的函数其实都是在networking里面的(实现了基于事件的发送与接受):`addReplyMultiBulkLen`&`addReplyBulk`

syncCommand
--
- 首先十分让人困惑的代码是:
  ```C
      if (c->flags & REDIS_SLAVE) return;
  ```
  为啥一旦当前的客户端(在这个位置是表现为伪终端的模式)一旦状态为`REDIS_SLAVE`就返回呢?
    - 因为,sync这个命令只是对**掉线了的client**进行处理并且有作用的,在处理过程中(sync同步),会将当前的状态设置为`REDIS_SLAVE`,对于现在在献上的机器,直接使用对应的命令正常发送就可以了,不用care同步啥的问题.(需要重设的主要原因是由于freeCLient在每一次掉线的时候会关闭对应的client,这个flag存在的意义主要是区分slave与正常的客户端(不然怎么知道哪一位是客户端,哪一位是伪终端哟))  
- `masterTryPartialResynchronization`:当为psync命令时候,使用这个函数检查能否使用psync(还记得之前对bakclog各种鬼畜操作吗).这个函数比较复杂,后面慢慢写.
- 检测各种需要全备份的可能性:发送的runid号啊,当前的offset是不是正确的啊,什么的
- 现在到了不得不进行全备份的时候了,这个时候有一个优化,如果也**有一个slave正在进行rdb备份**(有时候是客户端用备份命令,这个时候是不可以的),这个时候其实是可以直接的进行共享的(把当前已经备份好的rdb文件共享)
  - 现在问题来了,如果不能共享呢,在这个方面redis的处理是,一个时间只能有一个rdb文件在备份,因此,等下一轮吧:设置flag:`REDIS_REPL_WAIT_BGSAVE_START`,之后会在`updateSlavesWaitingBgsave`函数里面检查这个flag的值(这个函数是bgsave的回调函数,所以,一个都跑不了,之后介绍吧)
- 大部分时候是没办法复用的,怎么办呢,用函数`rdbSaveBackground`直接开启新的文件备份鸭
- 接下来是一大部分的扫尾工作了
  - 设置各种状态(比如client当前已经是slave)
  - 添加到slave列表里面
  - 从slave变身成为master的时候,需要有backlog,这个时候其实应该创建一个的函数`createReplicationBacklog`

masterTryPartialResynchronization
- 需要全备份的情况
  - 测试对应的命令,首先是runid(还记得?的存在吗)
  - 没有backlog或者当前的请求字段已经被覆盖掉了(根据对应的offset)
  - 当前这个时候必须要进行全恢复了,这个地方对客户端进行回复:`+FULLSYNC <runid> <offset>`
- 经过之前测试的都是猛士了,这个地方表明是可以直接进行psync的
- 这里会设置对应的flag什么的(当前其实已经算是一个slave了,设置一下slave的标签)
- 回复`CONTINUE`,说明当前是可以进行psync 
- 接下来当然是进行psync了啊,就是调用函数:`addReplyReplicationBacklog`

addReplyReplicationBacklog
- 这个函数的主要作用其实是根据客户端发送的offset计算对应当前的发送哪些字段内容,然后包装函数`addReplySds`直接把对应的段发送过去就好了
- `addReplySds`这个函数位于`networking.c`里面,主要是包装了异步写的主要函数
- **之前我一直在想个问题:为啥不存在确定命令边界的问题呢???**:*因为根本不需要鸭,在现在这种模式里面,主要是根据客户端发送的文件计算应该重新发送那一部分的bakclog,我们起亲爱的客户端知道命令的边界的,所以发送过来offset之后,要不是不存在对应的命令(超过边界了),要不是边界根据offset区分得好好的,所以不需要边界区分的手段*

updateSlavesWaitingBgsave
- 主要处理在replication中的两类事件
  - 之前需要进行rdb保存.但是现在条件不允许,因此等到这个回调函数的时候再进行一次,主要是falg :`REDIS_REPL_WAIT_BGSAVE_START`,同样也是封装函数`rdbSaveBackground`,这个函数也会创建一个后台进程进行备份,而且再次执行本回调函数
  - `REDIS_REPL_WAIT_BGSAVE_END`:表明现在之前这个请求过发送rdb文件,因此现在就应该发送给他对应的文件了,注册事件监听,当对端可写的时候,调用回调函数: `sendBulkToSlave`,将对应的rdb文件传送给对方
  - 在这些地方中出现问题就直接freeClient了.
   
sendBulkToSlave
- 这个就是异步写函数的活脱脱的现实版鸭.
- 对于redis,写其实是必须分两部的,首先先写一下,当前发送的数据量究竟有多少,才能...让对方冷静下来(及时的释放已经GG的socket以及对应事件?),在这里主要是`slave->replpreamble`数据,注意这个字段的检查是在函数一开始的时候,因此不用担心会发森阻塞的问题(现在难道不是被epool唤醒的吗),出现问题直接说明GG了
- 开始发送文件了,使用文件描述符代表RDB`slave->repldbfd`,由于异步发送文件,随时可能产生EAGIN错误,因此必须要记录发送的offset:`slave->repldboff`(使用lseek系统调用进行唤醒操作),一旦发生EAGIN,直接return等下一波epool唤醒吧
  - 对于write系统调用,反正...写到阻塞也没啥关系,就两种情况,当前已经阻塞了,直接返回-1,写了一部分,直接返回当前写入的数目(参见ueap.e3)
- 计算offset与`slave->repldbsize`可以获得当前是不是已经写入完全了,写入完全的时候,可以直接设置各种flag,比如`REDIS_SLAVE_ONLINE`之类的,删除一下脚手架(之前的rdb文件描述符什么的),event不用了要及时撤离,就这样的了

slaveofCommand
- 就是...设置一下各种命令的实现,转变为slave啊,转变为master啊,之类的

replconfCommand
- 不想写~~,貌似是一个master和slave交流的一个命令的实现(貌似是同步的,目前)