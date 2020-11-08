# Redis设计与实现

> 基于redis仓库提交: 84b3c18f7

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
        - raw编码方式: 为字符串创建sds对象，并将prt指向sds（实际是指向内容区，见上文中对sds的说明）
        - OBJ_ENCODING_EMBSTR编码方式: 此时将申请一段连续空间，将sds直接追加到robj后面的地址空间中。
          当字符串长度小于或等于44时，会使用这种编码方式，可减少一次内存分配操作。用来存放短的只读字符串，修改后会转成raw编码。
        - OBJ_ENCODING_INT编码方式: 当存储数据为int_64时，可以使用该编码方式，此时ptr存储内容为64位有符号整数值（注意直接存储值而不是指针）
      - OBJ_LIST：列表类型，可以正向、反向遍历
        - OBJ_ENCODING_QUICKLIST编码方式: quicklist
        - OBJ_ENCODING_ZIPLIST编码方式: ziplist
      - OBJ_SET：集合对象
        - OBJ_ENCODING_HT编码方式：使用sds作为key的dict
        - OBJ_ENCODING_INTSET编码方式: intset
      - OBJ_ZSET: 有序集合对象
        - OBJ_ENCODING_SKIPLIST编码方式: zset，可以通过map快速获取节点
        - OBJ_ENCODING_ZIPLIST: ziplist，一个元素需要在ziplist中插入两个片段，分别存储ele和score
      - OBJ_HASH: 哈希对象
        - OBJ_ENCODING_ZIPLIST: ziplist
      - OBJ_STREAM: 流对象
        - OBJ_ENCODING_STREAM: stream对象，多用于消息机制实现
      - OBJ_MODEL:
        - OBJ_ENCODING_RAW: moduleValue对象，内含moduleType和value两部分，moduleType提供了一系列操作value的函数
    - redis可以根据实际灵活切换编码方式，以优化内存占用和访问速度
    - redis内存储了一些共享对象，如于10000的无符号整数等，用于减少不必要的对象创建