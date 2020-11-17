# Redis设计与实现

> 基于redis仓库提交: [84b3c18f7](https://github.com/KnockHarder/learning-redis)

## 基本数据结构

1. 动态字符串
    - 相关文件: sds.h sds.c
    - 多种长度类型: sdshdx，区别在于结构体首端用于记录buf已用长度与分配长度的整形不同(uint8_t,uint16_t ...)
    - sds由三个部分组成（sdshd5较特殊，而且一般不会使用，这里不考虑sdshd5）:
      - 长度区域：记录字符串占用长度和实际申请空间的长度，sdshdx使用uintx_t记录长度
      - 标记位： 用于区分sds实际使用的类型，固定占用一个字节
      - 内容区：指向实际字符串
    - 动态分配大小，当字符串长度小于1M时，x2增长，否则+1M
    - sds对象使用较为特殊，一般不使用sds类型指针，而是向外传递一`char*`类型指针，直接指向内容。这样可以直接重用相应的操作函数，不必为sds新写输出相关函数，如打印函数
2. 列表-list
    - 相关文件: adlist.h adlist.c
3. 字典-dict
    - 相关文件: dict.h dict.c
    - 通过hash实现，当hash冲突时，使用链表解决，dict会记录hash表长度(length)及总数据数(used)
    - 当哈希冲突超过阈值时，会将hash表扩张至二倍。当dict_can_resize为真时，阈值为used>=length，否则为used / length > dict_force_resize_ratio (默认为5)
    - 扩张后会进行rehash，但该过程为渐进的式的，此时dict会持有两张hash表，第二张（新表）的长度为第一张的二倍。
      每次增删改查时，会扫描旧表对非空节点做数据迁移，当迁移了足够多的hash节点或路过了足够多的空节点时，停止扫描并记录下一次扫描位置。
      迁移完成后，释放旧表，并进行翻转
4. 跳跃表: zskiplist
    - 相关文件: server.h z_zset.c
    - 跳跃表是在普通双向链表的基础上，在每个节点随机生成若干层（层数不小于1不大于32）
    - 当从head向tail遍历时，最下层形成了简单双向链表，节点间span为1，而层i(i>1)的span则记录了距离下一个层i节点的距离
    - zskiplist为每个节点记录分值(score)，zskiplist的排序规则为优先比较score大小，等score的情况下比较sds大小
    - 通过尽可能使用高层进行跳跃式遍历，在N不能远大于`2^32`（32为最高层数）时，查找和插入的平均时间复杂度为O(logN)，最坏O(N)，
      删除操作的时间复杂度为O(1)；当`N >> 2^32`时，跳跃表的优势将下降，最终查找和插入的平均时间复杂度将达到O(N)
    - 如果从head开始从前向后遍历，将途经节点的span值累加，我们可以得到该节点到head的距离，该距离为该节点的rank值。因此除了通过score、sds筛选外，
      zskiplist还允许通过rank筛选我们所需要的值
    - 跳跃表相较有有序数组，性能较差，但不需要一块完整的大的内存空间，且每个节点的大小可以不一致，更加灵活。
      而相较于红黑树，跳跃表的维护更加简单，但需要双倍的空间消耗（仅考虑不满足 N >> 2^32 的情况），但需要事先定好最高层级
5. 整数集合-intset
    - 相关文件: intset.h intset.c
    - 为了节均空间，intset的整数类型尽可能选小，初始使用int16类型。当插入数据超过范围时，再使用较大一级的整型，并扩充空间
    - intset为有序数组，查找时使用二分查找，时间复杂度为O(logN)
    - 每次插入数据都需要扩容，并使用内容拷贝的方式将插入位空出来，时间复杂度与查找一致
    - 需要注意的是intset在节点删除后，并不会尝试更小的数据类型
6. 压缩列表-ziplist
    - 相关文件: ziplist.h ziplist.c\
    - ziplist总体上由三个部分
      1. 头区域——含两个uint_32以及一个u_int16，分别记录ziplist占用总字节数、最后一个条目地址相对于ziplist头地址的偏移、拥有的条目数量
      2. 内容区域——使用连续内存块存放所有的条目，大小随条目增长
      3. 尾区域——只含一个uint_8，并被置为255标记ziplist结束
    - 每个条目已包含三个部分
      1. 前一节点长度区域——用于记录前一节点（地址增加顺序）的长度信息，视长度大小本区域占用1或5个字节（取决是长度能否用1个字节表示）
      2. 编码及长度区域——记录本条目的编码方式及内容值占用字节数量，视编码方式和内容长度占用1/2/5个字节
      3. 内容区域——存储经编码转换后的内容。
    - 编码方式有两种：整数型与字节型，取决于输入内容是否可被作为ASCII码转换为不大于64位的有符号整数
    - 为了最小化使用空间，编码信息与长度进行混合存储，`编码及长度区域`的第一个字节内可能同时包含编码信息和部分长度信息
    - 由于条目的`前一节点长度区域`长度因前一节点长度大小不同可能占用1字节，也可能占用5字节，因此在进行插入和删除操作时，可能会导致可向后传导的对`前一节点长度区域`的修正，
      因此插入与删除动作的时间复杂度为O(n)
    - 因为zipentry同时记录了自身长度及前一节点长度，因此ziplist支持向前向后遍历两种方式

## 其他数据类型

1. 快速列表-qucklist
    - 相关文件: quicklist.h quicklist.c
    - 由以下几个部分组成:
      1. head-头节点指针，tail-尾节点指针
      2. count-总条目数量，len-节点数量。这里两个数量并不一致，count为各node.count的总和。
      3. fill-用于限制单个节点中ziplist大小，默认值为-2，占用14bits。当fill < 0时，通过限制单个ziplist长度的方式控制大小，对应的阈值为optimization_level[-fill-1]；
         当fill >= 0时，单个ziplist中条目数量应小于fill。
      4. ompress-需要压缩的节点的最小深度（距离头节点或尾节点的距离），占用14bits。当列表节点深度低于阈值时，不会对节点进行压缩，
      5. bookmarkcount-记录书签数量，占用4bits。共占用4字节。
      6. bookmarks-书签数组，可以为quicklistNode创建相应的书签，以便快速访问
    - quicklistNode-列表节点
      - 由以下几个部分组成：
        1. prev-指向上一节点，next-指向下一节点
        2. zl-指向压缩列表(ziplist)
        3. sz-压缩列表占用空间大小，unsigned int类型
        4. count-压缩列表中条目数量，16bits的无符号整数
        5. encoding-编码方式，包括raw与lzf压缩两种编码方式，2bits的无符号整数
        6. container-节点携带的数据类型，有NONE与ZIPLIST两种，2bits的无符号整数
        7. recompress-当ziplist因为使用被解压时，需要被重新压缩，这种情况下该值为1。为1bit无符号整数
        8. attempted_compress-标志位，如果之前压缩时因数据过小(小于48节点)无法压缩，该字段置1，1bit的无符号整数
        9. extra-额外保留空间，占用剩余的10bits
    - quicklist相对于ziplist而言，不需要连续的空间，因此可能存储更多的数据。且可对ziplist使用lzf进行压缩，进一步减少空间占用。

2. 有序集合-zset
    - 相关文件: server.h z_zset.c
    - zset由两个部分组成:
      1. dict字典，key是sds，value是score
      2. zskiplist跳跃表，存储内容有sds和score
    - 因此zset通过score和ele进行排序

3. 流对象-stream（内容需要完善）
    - 相关文件: stream.h stream.c
    - 主要用于消息队列
    - 由以下几个部分组成
      1. rax树根节点存储消息
      2. 消息的ID
      3. rax树节点

4. redis对象-robj
    - 相关文件: server.h object.c
    - 总体上由5个字段
      1. type记录数据类型，占用4bits的无符号整数
      2. encoding记录编码方式，占用4bits的无符号整数
      3. lru记录最近使用时间或最近使用频次，占用24bits的无符号整数
      4. refcount记录被持有的计数，int类型
      5. ptr指向实际内容，void*指针
    - 通过数据的最近使用时间或最近使用频次，以及被持有计数，对内存进行回收。当限制内存大小的情况下，如果内存占用过多需要释放对象占用空间时，
      默认使用LRU策略，可通过配置选择使用LFU策略
    - type及对应的编码方式如下:
      - OBJ_STRING: 字符串类型，内容使用sds结构
        - OBJ_ENCODING_RAW编码方式: 为字符串创建sds对象，并将prt指向sds（实际是指向内容区，见上文中对sds的说明）
        - OBJ_ENCODING_EMBSTR编码方式: 此时将申请一段连续空间，将sds直接追加到robj后面的地址空间中。
          当字符串长度小于或等于44时，会使用这种编码方式，可减少一次内存分配操作。用来存放短的只读字符串，修改后会转成raw编码。
        - OBJ_ENCODING_INT编码方式: 当存储数据为int_64时，可以使用该编码方式，此时ptr存储内容为64位有符号整数值（注意直接存储值而不是指针）
      - OBJ_LIST：列表类型，可以正向、反向遍历
        - OBJ_ENCODING_QUICKLIST编码方式: quicklist
        - OBJ_ENCODING_ZIPLIST编码方式: ziplist
        - 实际上已不再使用ziplist，只使用quicklist这一种编码形式
      - OBJ_SET：集合对象
        - OBJ_ENCODING_HT编码方式：使用sds作为key的dict
        - OBJ_ENCODING_INTSET编码方式: intset，当所有元素都是可以用`long long`表示时，使用该类型
        - 只允许由OBJ_ENCODING_INTSET向OBJ_ENCODING_HT升级
      - OBJ_ZSET: 有序集合对象
        - OBJ_ENCODING_SKIPLIST编码方式: zset，可以通过map快速获取节点
        - OBJ_ENCODING_ZIPLIST: ziplist，一个元素需要在ziplist中插入两个片段，分别存储ele和score
        - 当条目数量小，且最大条目长度不超过阈值时，使用ziplist，否则升级为skiplist。在部分操作中也允许降级。
      - OBJ_HASH: 哈希对象
        - OBJ_ENCODING_ZIPLIST: ziplist
        - OBJ_ENCODING_HT: dict实现，使用sds作为哈希的key和value
        - 当哈希表中内容较少时，使用该ziplist，否则升级为dict。不允许降级操作。
      - OBJ_STREAM: 流对象
        - OBJ_ENCODING_STREAM: stream对象，多用于消息机制实现
      - OBJ_MODEL:
        - OBJ_ENCODING_RAW: moduleValue对象，内含moduleType和value两部分，moduleType提供了一系列操作value的函数
    - redis可以根据实际灵活切换编码方式，以优化内存占用和访问速度
    - redis内存储了一些共享对象，如于10000的无符号整数等，用于减少不必要的对象创建

## redis数据库

redisServer有多个数据库，在服务启动时，会初始化相应的redisDb对象作为数据库对象。我们可以通过redis-cli中的select命令切换数据库。

redisDb中包含以下几个部分:
1. dict: `dict*`类型。键类型为字符串(sds)，值类型可以为任意类型对象。用于存储键值对。
2. expires: `dict*`类型。键类型为字符串，值类型为64位带符号整数，存储过期时间戳（单位毫秒）。用于通过键找到条目的过期时间。
3. blocking_keys: `dict*`类型。键类型为使用raw编码的字符串对象，值类型为`list`类型。
4. ready_keys: `dict*`类型。键类型为已编码的字符串对象，值类型可以为任意对象类型
5. watched_keys: `dict*`类型。键类型为使用raw编码的字符串对象，值类型为`list`类型。
6. id: `int`类型，数据库的id，为从[0,redisServer.dbnum)中的一个值
7. avg_ttl: 64位有符号整数
8. expires_cursor: 64位无符号整数。配合定期缓存过期任务使用，记录已描述的slot数量。该值只增不减，且dict有两个动态扩容的th，因此实际运行时，需要实时计算访问下标。
9. defrag_later: `list*`类型，元素为`sds`类型。

## 缓存过期删除

- 缓存过期删除通过两种策略配合完成:
  - 惰性删除(db.c/expireIfNeeded): 让需要访问键对象时，判断对象是否过期，如果过期则回收。该策略使垃圾回收只回收需要过期处理的对象，性能较高，但因无法及时清理空间，
    可能会导致内存泄漏。
  - 定期删除(server.c/initServer,serverCron,databasesCron): 每隔一段时间，增量的执行缓存清理工作（类似dict的rehash过程）。该策略可以避免过期对象对空间的过长占用，但由于这种扫描方式的回收性能差，
    因此需要根据实际情况，合理设置清理任务的时间间隔，以及单次任务量。避免CPU过多的占用或者内存的过多浪费。
    - 扫描时，如果当前db的填充率不足1%，则放弃对该db的扫描，避免过多的访问空节点
    - 可通过配置限制垃圾回收任务的CPU使用率，限制垃圾回收的占用时长，计算方式受`server.hz`配置的影响
- 在删除对象时，如果开启惰性释放功能，则(lazyfree.c/dbAsyncDelete)会判断删除对象需要的空间释放次数，如果释放动作过多，则会将释放任务提交到队列中，
  供垃圾回收后台进行空间回收。

## 缓存持久化

- 持久化有两种形式: RDB与AOF
- RDB持久化一般使用dump.rdb文件，文件名可以通过配置文件修改
  - RDB持久化当前数据库中的内容
    - 校验头信息，由9个char组成: `"REDIS"`+`4位版本号`
    - Aux信息
    - 数据库部分
      - 数据库基本信息: id、缓存条目的dict大小、expires的dict大小
      - 缓存条目/其他:
        - 条目属性: 条目的一些额外属性，如过期时间、lru/lfu值等。每条属性由以下两个部分组成:
          - 类型: 占用1字节
          - 属性值: 占用空间大小视随类型不同而变化
        - 条目key值: sds对象
        - 条目value值: 任意对象类型
            - 对象类型: 需要注意的是该类型并不是robj中的type，而是rdb存储时定义的类型。占用一字节。
            - 对象值
    - Aux信息
    - EOF标志
    - 文件校验和: 占用8个字节，64位无符号整数
  - RDB持久化时，即使对象过期，也会写入文件；但读取时，会过滤过期条目（如果当前节点为主节点）。
  - 如果打开了`rdbcompression`功能，则在进行持久化时，会对大内容进行lzf压缩
  - 手动触发RDB持久化，可以通过`SAVE/BGSAVE`命令调用。`FLUSHALL`命令也会进行持久化，但是会清空数据库后再持久化，因此实际上是重置rdb文件。
  - 除此之外，系统也会自动进行持久化备份，当满足下面任意条件时，会触发持久化动作:
    - 1小时内数据库有变化
    - 五分钟内变化条目不小于100
    - 1分钟内变化条目不小于10000
    - redis即将关闭，且未指定不保存数据
- AOF持久化，是一种更频繁的持久化动作，，以保证即时遭遇断电这种极端情况，也能恢复尽可能多的数据。
  - 将在数据库变化时将变化命令输出到aof_buf，在定时任务执行前(beforeSleep)，将缓存区的内容输出到AOF文件（默认为appendonly.aof）
  - 由于现代操作系统中，为提高IO效率，在写文件时往往采用的**延迟写**的策略，写文件操作只是将内容写入缓冲区而不是实际文件。
    因此AOF持久化的写文件另外提供了`appendfsync`控制文件同步(fsync)策略，以在对安全性要求高的场景下保障AOF文件内容的完整性。
      - no: 使用操作系统默认策略
      - everysec: 每秒一次，进行文件同步
      - always: 每次进行写文件操作时，均进行文件同步
  - 由于AOF文件会持续增长，且会记录`DEL`等操作，因此当AOF文件过大，且增强到一定比例后，会进行重写
  - 重写时会根据数据库中的实际情况，以`SET/RPUSH`等命令，记录下各缓存对象。由于单个命令的大小限制，因此在记录集合类型时，
    会控制单个语句的大小(AOF_REWRITE_ITEMS_PER_CMD=64)
  - 重写会创建一个子进程单独执行相关操作，在重写期间新生成的AOF内容会除了定入AOF缓冲区外，还会放入AOF重写缓冲区(aof.c/feedAppendOnlyFile)，
    重写子进程在完成db数据的重写后，会将重写缓存区中的内容写入AOF文件(aof.c/rewriteAppendOnlyFile)。
    在重写进程结束后，发送完成信息(aof.c/rewriteAppendOnlyFileBackground)，主线程之后会再一次将AOF重写缓冲区中的内容定稿AOF文件()，
    完成重写任务(aof.c/backgroundRewriteDoneHandler)。
  - 如果指定了使用RDB的格式进行重写时，那么将以RDB文件的格式进行输出

现在考虑一种场景：redis运行了一段时间，我们发现没有打开AOF功能，现在应该怎么做？

最容易想到的是，修改redis.conf文件，将`appendonly`配置为`yes`，然后restart。然后我们会发现，数据库里的数据全部消失了！

这是因为如果打开了AOF功能，redis启动时会从aof文件中恢复数据，而在重启前AOF功能是关闭状态，并不会生成aof文件，导致无法恢复数据。

因此，正确的做法是通过`CONFIG SET`命令，先将`appendonly`配置切换为`yes`，会立即生成aof文件。
然后修改配置文件中的`appendonly`即可，以保证重启后AOF功能在重启后仍是开启状态，并使用aof文件进行数据恢复。

## 事件机制

Redis的是一个事件驱动程序，事件分为两种类型:
- 文件事件: 服务器与客户端通过socket连接，文件事件是对socket的抽象，进行读写操作。
- 时间事件: 对定时操作的抽象。

### 文件事件

- 文件事件使用多路复用实现——服务器同时接收多个客户端连接，在处理时只有一个单进程处理。Redis的多路复用底层实现有多个，但对外提供统一接口，
  在编译时会选择其中一个。
- 文件事件类型分为可读(READABLE)、可写(WRITABLE)两种类型，当一个事件同时可读写时，默认先处理读事件后处理写事件，
  如果指定事件类型为AE_BARRIER，则先处理写事件后处理读事件。
- Redis在启动时，会监听配置的接口，并创建相应的文件处理器，用于处理通过tcp/tls/unixSocketFile发送的命令
  - 创建client(networking.c/createClient)时，通过connSetReadHandler配置处理逻辑(networking.c/readQueryFromClient)，
    并将事件监听加入eventLoop中(connection.c/CT_Socket,connSocketSetReadHandler)
  - readQueryFromClient在执行命令时，会生成返回信息，添加返回信息加时会将客户端记录到server.clients_pending_wirte中
    (networking.c/prepareClientToWrite)，在before_sleep中被输出到发出请求的客户端。

### 时间事件

- 时间事件分为一次性事件与循环事件，区别在于事件函数的返回值，如果返回`-1`则该事件执行一次后被置为删除状态（下一次主循环时删除），如果为正数x，则表示每隔x秒执行一次。
- 事件机制的处理逻辑在aeMain主循环中进行，文件事件优先于时间事件处理（获取文件事件的最大阻塞时间不超过距离最近一个时间事件的时间），因此时间事件的执行时间往往要稍晚于设定时间。

Redis的事件遍历处理在主循环中，各个事件应尽可能少的占用时间，避免出现抢占，因此每个事件的处理需要控制任务量（NET_MAX_WRITES_PER_EVENT）或直接创建子进程进行处理。

#### key事件通知（默认关闭）

- 可以通过事件消息的方式，通知键值上的操作，该功能可以通过配置`notify-keyspace-events`打开
- 可以通过`SUBSCRIBE`命令监听相应的事件
- 需要注意的是，这里实际上是通过`观察者模式实现`，因此需要先有观察者注册后(pubsub.c/pubsubSubscribeChannel,pubsub.c/pubsubPublishMessage)，
  才会产生发消息的动作。不要误将其当成`生产者-消费者`模式理解。
- 发送的消息在client的回复缓冲区，因此实际发送动作也在主循环的`beforeSleep`中

## 客户端(client)

- client结构的定义在server.h文件中，下面列举了几个常用的字段
  - id: 身份标识，server维护了一个原子类型的64位整数，每新建一个客户端便进行自增做为新客户端的id。
  - conn: 客户端连接信息
    - 如果是有连接客户端，则记录了连接信息(如tcp/tls文件描述符等)
    - 如果是无连接客户端，则为NULL（也称为fakeClient）。例如在loadDataFromDisk时，如果使用aof文件恢复，则会生成一个无连接客户端执行aof文件中读取出的命令；redisServer持有的lua_client；
      自定义模板中用于执行命令的客户端。
  - db: 当前使用的数据库
  - name: 连接名称，默认为NULL，可通过`CLIENT SETNAME`命令为客户端设置名称以方便区分
  - querybuf、qb_pos、querybuf_peak: 读缓冲区，用于存放从conn中读取到的数据
  - argc、argv、argv_len_sum: 客户端请求命令参数
  - cmd、lastcmd: 从读缓冲区获取到的命令
  - flags: 记录客户端的角色(slave/master)以及当前的状态
  - buf、bufpos、reply: 输出缓冲区。当buf空间不够用时，会使用`list*`结构的reply存储输出信息，**buf不再使用**。
  - authenticated: 客户端是否通过身份验证
  - ctime、lastinteraction: 客户端创建时间、最后一次与服务器交互的时间
  - obuf_soft_limit_reached_time: 输出内容第一次达到软限制的时间
    - 这里的大小指: 已发送内容大小 + 缓冲区内容大小(当一次循环无法及时将内容发送完时会记录已发送数据大小)
    - 软/硬限制、超出软大小限制时长限制可通过`client-output-buffer-limit`配置
    - 如果大小超过硬性限制，客户端将被关闭；如果超过软性控制，且持续时间(如果中间某段时间回落则重新计算)超过时间限制，客户端将被关闭。
- 普通客户端的创建: 当客户端程序连接到服务器时，服务器会创建一个client对象，记录客户端状态，并加入`server.clients`列表中；同时创建一个文件事件用于处理请求。在执行命令前会做一些检查，如果不满足要求，则拒绝执行。检查内容包括
  - 命令是否存在
  - 参数个数是否正确
  - 客户端是否通过权限校验
  - 是否需要进行空间回收，回收后是否有足够内存
  - 如果命令会造成数据变更，数据持久化流程是否正常（看最近一次持久化结果）
  - 当前服务器、客户端的身份是否适合执行该命令
  - 其他
- 普通客户端的关闭可以由于以下几个原因:
  - 网络连接中断
  - 请求格式错误、请求内容长度超限
  - 被KILL
  - 长时间空转
  - 输出内容长度超限

## 多实例

- 从节点未不会直接删除过期键，而是等到主节点的`DEL`消息时才会删除键。在此之前，如果客户端获取命令为非只读，非会返回过期数据。
