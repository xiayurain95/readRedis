Redis中的eventLoop
=
- eventLoop是redis的事务核心部分了
  - 主要分两部分:
    1. 文件世间部分,由于IO操作本质上很容易产生阻塞,对大部分文件事件进行抽象,使用epoll实现文件事件的非阻塞(O_NONBLOCK)
    2. 时间事件,时间事件本质上对于当前的任务是异步的(不知道在何时会产生对应的事件),因此也对时间事件进行抽象,在v3.1版本中只有一个timer时间进行PING PONG等定时事务
  - 所有的事务基本都是在这里面，它的主函数aeMain是redis主函数的循环的一部分，这个主函数的循环退出redis也就退出了
  - eventloop注册了一个beforesleep函数，这个函数在每次调用epoll_wait之前进行调用,在进入sleep之前完成一部分的事务操作
    - 是键值检查，replica和一些啥aof，rdb的东西的处理

eaMain
- 没啥好说的，先执行beforeSleep然后再是aeProcessEvents

aeProcessEvents
- 首先的是计算一下时间函数事件的处理时间
  - 之前知道的嘛，epool有一个阻塞时间，这里的阻塞时间实际上是一个上线，（文件事件的执行周期基本上是一致的）告诉我们什么时候可以重新执行文件事件 ，时间事件在这个空档里面悄悄执行一下下
  - aeApiPoll(eventLoop, tvp)，本质上是算这个tvp
    - 这个是一个封装函数，实现底层的隔离的，一个文件系统事件的api，我们这里是以epool为代表的嘛,这里的封装是aeApiPoll，调用方式其实和epool一模一样。。。
      - 返回当前就绪的事件的数目啥的，然后进入一个for循环里面，一个个的处理掉～ 
      - 插播一下epool的接口封装。。。apidata是实现多路复用库兼容的关键，在epool里面，它是aeApiState，本质是封装了epool的文件描述符以及event的pool
        ```C
        //我们敬爱的aeEventLoop里面封装了一只这个：
        // 多路复用库的私有数据
        void *apidata;

        typedef struct aeApiState {

          // epoll_event 实例描述符
          int epfd;

          // 事件槽
          struct epoll_event *events;

          } aeApiState;
        ```
        - 这里的封装真的是一点佐料都没有加诶，直接就。。。算是dump了过去。。。把就绪的文件描述符放到了eventLoop的fired里面去了（有点像epool的event数组的感觉）
          - 有点不清楚&eventLoop->events[eventLoop->fired[j].fd];为啥需要events这个数据结构，有啥用哟
            - 这个地方几乎石锤是一个稀疏的矩阵了
            - amazing！！！ 虽然知道文件描述符是归程序所有，但是第一次知道这个是按顺序进行分配的，1,2,3是特殊的，其他都是按照顺序来的，是一个内部的，到一个字符串的表。。。 
            - 所以这个地方其实是在查找注册的时候的状态信息
              - 使用读写事件的回调函数，实现对应的状态的激活
              ```C
                  // 读事件处理器
                  aeFileProc *rfileProc;

                  // 写事件处理器
                  aeFileProc *wfileProc;
              ```
    - 这里是返回一个数组，这个数组由fd与flag组成的，本质其实是epool在内核态中发现一些处于ready状态的fd，直接放到了这个里面，就是说这个数组的大小本质上决定了当前我们可以缓存的事务的大小
  - 处理时间事件的函数就，。。。十分的简洁了processTimeEvents
    - 这个的逻辑本质上十分的简单
    - 首先，时间是有序的，所以，时间事件本质上是可以使用一个简单链表穿插完成的
    ```C
    typedef struct aeTimeEvent {

    // 时间事件的唯一标识符
    long long id; /* time event identifier. */

    // 事件的到达时间，达到对应到时间之后执行（原来的注释错了的）
    long when_sec; /* seconds */
    long when_ms; /* milliseconds */

    // 事件处理函数
    aeTimeProc *timeProc;

    // 事件释放函数
    aeEventFinalizerProc *finalizerProc;

    // 多路复用库的私有数据
    void *clientData;

    // 指向下个时间事件结构，形成链表
    struct aeTimeEvent *next;

    } aeTimeEvent;
    ```
    - 不断的遍历这个链表，调用上面的时间处理callback函数，实现我们对时间序列的操作（timeProc函数）
    - 时间时间有一个id 主要的目的其实是定位对应的时间事件，可以进行移除操作
    - 时间事件的加入时间操作，当超过对应的时间之后，才会对某一个时间事件进行处理，首先是秒，然后是毫秒时间，可以使用函数对时间进行操作
    - 时间时间的一个细节，系统的时间可能随时改变（手贱的管理员一下子重置了一下什么的），在这个地方使用，一旦发生一点故障，就直接使的所有时间函数一起执行
      - 这个位置是 重设when_sec与when_ms，让他等于0,
    - 时间事件是可以重入的，当时间操作timeProc是一个具体的值的时候，将当前时间操作重入到链表里面（当前是链表头）
      - 重入当然还是需要时间戳的啊，和上面那个一样是重新设置when_sec 和 when_ms 
      - 只要时间事件加入链表都必须要有个执行的时间，在redis里面是aeAddMillisecondsToNow设置这个时间的

ae_epoll
--
- 暴露了一系列的api
  - aeApi系列creat resize  free addEvent delEvent poll（主循环） name（由于复用，所以是必须要知道名字的）
  - 下面是一个epoll的data的内容，epoll_event包含一个events属性，就是。。。那一群typedef，data的话，是一个union，里面大家一般只会用ptr或者fd，一般情况下，大家都是直接使用fd，获取现在的文件描述符，但是在特殊情况的时候，我们除了需要返回一个简单的文件描述符还需要返回一些其他的东西，使用ptr吧
  ```C
  typedef union epoll_data
  {
    void *ptr;
    int fd;
    uint32_t u32;
    uint64_t u64;
  } epoll_data_t;

  struct epoll_event
  {
    uint32_t events;	/* Epoll events */
    epoll_data_t data;	/* User data variable */
  } __EPOLL_PACKED
  ```
  - 鼓舞人心的就是这一群addEvent和delEvent和pool的啦
    - 这个其实是重新的定义了一大堆的事件
    - EPOLLERR 文件出错。即使不设置这个标志，这个事件也是被监听的。<br/>
      EPOLLET 使用边沿触发。默认是水平触发。<br/>
      EPOLLHUP 文件被挂起。即使不设置这个标志，这个事件也是被监听的。<br/>
      EPOLLIN 文件未阻塞，可读。<br/>
      EPOLLONESHOT 在一次事件产生被处理之后，文件不在被监听。必须不再被监听。必须使用EPOLL_CTL_MOD指定新的事件，以便重新监听文件。<br/>
      EPOLLOUT 文件未阻塞，可写。<br/>
      EPOLLPRI 高优先级的带外数据可读。 <br/>
  - 这个位置本质上主要就是包装了IN和OUT两种状态
  - 在设计callback的时候还有一个provatedata，这是执行callback的时候，需要调用的一些文件属性，直接放进来
    - 算是实现一种服用吧 