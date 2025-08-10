## 使用场景
+ 缓存
+ 限时业务
+ 计数器（`incrby`原子性自增）
+ 分布式锁（`setnx`）
+ 利用订阅实现延时操作
+ 排行榜（`Zset`）
+ 简单队列

## 底层数据结构
### 简单动态字符串 SDS
```c
struct sdshdr {
    // 已使用字节数量，等于 SDS 所保存的字符串长度
    int len;
    // 未使用字节数量
    int free;
    // 字节数组，用于保存字符串，最后一个字节保存空字符'\0'
    char buf[];
}
```

+ SDS 用于表示一个可以被修改的字符串值，当`SET msg "hello world"`时会创建两个 SDS，一个是`msg`，一个是`hello world`
+ 具体用途 
    - 字符串的值
    - 缓冲区，如 AOF 模块的 AOF 缓冲区，客户端状态的输入缓冲区
+ SDS 与 C 字符串的区别 
    - **常数复杂度获取字符串长度**：因为 C 字符串并不记录自身的长度信息
    - **杜绝缓冲区溢出**：C 字符串不记录自身长度，因此在执行拼接等操作时如果原本分配的空间不足则可能会导致溢出，而 SDS 在修改前会先检查空间是否满足修改所需的要求
    - **防止内存泄漏**：C 字符串不记录自身长度，因此在执行截断操作后需要程序通过内存重分配来释放不再使用的那部分空间，如果忘了就会导致内存泄漏
    - **减少内存的分配和释放次数** 
        * 空间预分配：当需要对 SDS 进行空间扩展时，会为 SDS 分配额外的未使用空间（预测其接下来还会有扩展操作） 
            + 修改后长度小于 1MB，则分配 len * 2 + 1byte（1byte 为 `'\0'`）
            + 修改后长度大于等于 1MB，则分配 len + 1MB + 1byte
        * 惰性空间释放：缩短字符串时并不立即内存重分配，而是使用 free 属性记录起来，**也有 API 可以真正地释放**
    - **二进制安全**：C 字符串中不能包含空字符，且字符必须符合某种编码，因此 C 字符串不能保存二进制数据；SDS 的 API 都会以处理二进制的方式来处理 SDS 中 buf 字节数组里的数据

### 链表
```c
typedef struct listNode {
    // 前置节点
    struct listNode *prev;
    // 后置节点
    struct listNode *next;
    // 节点的值
    void *value;
} listNode;

typedef struct list {
    // 表头节点
    listNode *head;
    // 表尾节点
    listNode *tail;
    // 链表长度
    unsigned long len;
    // 节点值复制函数
    void *(*dup) (void *ptr);
    // 节点值释放函数
    void (*free) (void *ptr);
    // 节点值对比函数
    int (*match) (void *ptr, void *key);
} list;
```

+  特点 
    - 双向链表：节点带有 prev 和 next 指针
    - 带表头和表尾指针
    - 带长度计数器
    - **多态**：链表节点使用`void*`保存节点值，并且可以通过`list`结构的`dup`、`free`、`match`三个属性为节点值设置类型特定的函数
+  用途 
    - 列表的底层实现之一
    - 发布与订阅
    - 慢查询
    - 监视器等

### 字典（映射）
+ 字典是一种用于保存键值对的抽象数据结构
+ 特点：**每个键独一无二**
+ 用途 
    - Redis 数据库本身就是使用字典作为底层实现
    - Hash 散列的底层实现之一
+ 实现：哈希表

```c
typedef struct dictht {
    // 哈希表数组
    dictEntry **table;
    // 哈希表大小（不是指存了多少元素）
    unsigned long size;
    // 哈希表大小掩码，size - 1
    unsigned long sizemask;
    // 已有节点数量
    unsigned long used;
} dictht;

typedef struct dictEntry {
    // 键
    void *key;
    // 值
    union {
        void *val;
        uint64_tu64;
        int64_ts64;
    } v;
    // 指向下一个节点，形成链表（即采用链地址法解决哈希冲突）
    struct dictEntry *next;
} dictEntry;
```

+  特点 
    - 采用数组 + 链表实现
    - **采用链地址法解决哈希冲突，插入时为头插法**
+  字典实现 

```c
typedef struct dict {
    // 类型特定函数，为创建多态字典而设置
    dictType *type;
    // 私有数据，为创建多态字典而设置
    void *privdata;
    // 哈希表
    dictht ht[2];
    // rehash 索引，如果目前不在 rehash 则为 -1
    int rehashidx;
} dict;
```

+ `dictType`中保存了 
    - 计算哈希值的函数
    - 复制键的函数
    - 复制值的函数
    - 对比键的函数
    - 销毁键的函数
    - 销毁值的函数
+ `ht`是一个包含两个哈希表的数组，一般情况下使用`ht[0]`哈希表，`ht[1]`哈希表只会在对`ht[0]`哈希表进行 rehash 时使用
+ 哈希算法 
    - 传入 key 给哈希函数（默认为 MurmurHash2 函数）得到 hash 值
    - 对应索引值为`index = hash & ht[x].sizemark`
+ rehash 
    - 扩展时机 
        * 服务器没有执行`BGSAVE`或`BGREWRITEAOF`命令时，哈希表的负载因子大于等于 1
        * 服务器正在执行上述命令时，哈希表的负载因子大于等于 5 
            + 上述命令会创建子进程，操作系统采用写时复制技术优化子进程，提高负载因子可以避免子进程在备份期间进行哈希表的扩展操作，避免不必要的内存写入，节约内存
        * 负载因子 = 已保存节点大小 / 哈希表大小（`load_factor = ht[0].used / ht[0].size`）
    - 收缩时机 
        * 负载因子小于 0.1
    - 渐进式 rehash 
        * rehash 并不是一次性、集中式地完成，因为如果数据非常多的话会对服务器性能造成影响，可能导致其一段时间停止服务
        * 过程 
            + 为 ht[1] 分配空间，让字段同时持有两个哈希表
            + 维护 rehashidx，置初始值 0
            + rehashidx 索引上的键值对 rehash 到 ht[1] 后，rehashidx 自增
            + 直到所有键值对都 rehash 到 ht[1]，置 rehashidx 为 -1
        * rehash 期间的删除、查找、更新等操作会在两个哈希表上进行
        * rehash 期间的增加操作直接保存到 ht[1] 中

### 跳跃表
+ 跳跃表是一种**有序**的数据结构，在每个节点维持**多个**指向其他节点的指针
+ 节点查找时间复杂度平均`O(logN)`，最坏`O(N)`，实现比平衡树更简单，效率可以媲美平衡树
+ 用途 
    - Zset 有序集合的底层实现之一
    - 集群节点中用作内部数据结构

```c
typedef stuct zskiplistNode {
    // 层，每个节点的层高是 1 - 32 之间的随机数
	struct zskiplistLevel {
        // 前进指针
        struct zskiplistNode *forward;
        // 跨度
        unsigned int span;
    } level[];
    // 后退指针
    struct zskiplistNode *backward;
    // 分值，不同节点的分值可以相同，但是每个节点的成员对象必须唯一
    double score;
    // 成员对象
    robj *obj;
} zskiplistNode;

typedef struct zskiplist {
    struct zskiplistNode *header, *tail;
    // 表中节点的数量
    unsigned long length;
    // 表中最大的节点的层数
    int level;
}
```

+ 特点 
    - 每个节点的`level`数组可以包含多个元素，数组大小根据幂次定律（越大的数出现概率越小）随机生成
    - level 中的跨度用于计算排位，查找某个节点的过程中，将沿途访问过的所有 level 的跨度加起来就是节点在跳跃表中的排位
    - 跳跃表中所有节点根据分值**从小到大**排序，obj 指向一个字符串对象，字符串对象保存一个 SDS 值
    - 跳跃表中所有节点的成员对象必须是唯一的，分值可以相同

### 整数集合
+ 只包含整数值元素的集合
+ 底层实现为数组，以**有序、无重复**的方式保存元素
+ 用途 
    - Set 集合的底层实现之一

### 压缩列表
+ 压缩列表是为了节约内存而开发的，由一系列特殊编码的连续内存块组成的顺序型数据结构
+ 用途 
    - List 列表的底层实现之一
    - Hash 散列的底层实现之一
    - Zset 有序集合的底层实现之一

### 对象 redisObject
+ 在上述数据结构的基础上再包装一层，方便内存回收、对象共享等

```c
typedef struct redisObject {
    // 类型
    unsigned type:4;
    // 编码
    unsigned encoding:4;
    // 指向具体数据结构的指针
    void *ptr;
    // 引用计数
    int refCount;
    // 最后一次被命令程序访问的时间
    unsigned lru:22;
    // ...
} robj;
```

+ `type` 
    - `REDIS_STRING`：字符串对象
    - `REDIS_LIST`：列表对象
    - `REDIS_HASH`：哈希对象
    - `REDIS_SET`：集合对象
    - `REDIS_ZSET`：有序集合对象
+ `encoding`

![](https://cdn.nlark.com/yuque/0/2022/png/1732113/1644301008577-576f468c-a58c-4c07-a2e5-4f7bf2ee04a6.png)

+ Redis 可以根据不同的使用场景为一个对象设置不同的编码
    - 列表元素较少时，可以使用压缩列表，更节约内存
    - 元素对象越来越多，压缩列表的优势消失时，使用双端链表，更适合保存大量元素

#### String 字符串对象
+ int
    - 保存的是整数值，并且可以用 long 表示
    - **当对 int 对象 APPEND 等操作后会变成一个 raw 对象，但如果是 incrby 等增加操作则不会**
+ raw
    - 字符串值，并且长度大于 32 字节
+ embstr
    - 字符串值，长度小于等于 32 字节
    - 调用一次内存分配函数分配一块连续的空间，空间内依次包含`redisObject`和`sdshdr`
    - **只读：当对 embstr 对象 APPEND 等修改操作后会变成一个 raw 对象**

#### List 列表对象
+ ziplist 压缩列表
    - 使用条件
        * 保存的所有字符串元素长度小于 64 字节
        * 元素数量小于 512 个
+ linkedlist 双向链表

#### Hash 哈希对象
+ ziplist 压缩列表
    - 键值对的两个节点紧挨在一起，键节点在前，值节点在后
    - 使用条件
        * 保存的所有字符串元素长度小于 64 字节
        * 元素数量小于 512 个
+ hashtable 字典

#### Set 集合对象
+ intset 整数集合
    - 使用条件
        * 所有元素都是整数
        * 元素数量小于 512 个
+ hashtable 字典
    - 字典的值都为 NULL

#### Zset 有序集合对象
+ ziplist 压缩列表
    - 成员和分值紧挨在一起
    - 使用条件
        * 元素成员长度小于 64 字节
        * 元素数量小于 128 个
+ skiplist 跳跃表 + dict 字典
    - 跳跃表按分值从小到大保存所有集合元素
        * 跳跃表的优点在于执行范围型的操作、计算排名等
    - 字典用于支持快速的根据集合元素查询其分值
        * 字典的优点在于可以根据成员快速查找分值

#### 命令多态
![](https://cdn.nlark.com/yuque/0/2022/png/1732113/1644301968393-903a5236-ffa9-4360-8c95-91d38963be11.png)

#### 内存回收
+ 通过引用计数（redisObject 中的 refCount）实现内存回收，引用计数变为 0 时被回收

#### 对象共享
+ `SET msg1 100`，`SET msg2 100`
    - 第二次 SET 时会将值的指针直接指向第一次的值，并将其 redisObject 的 refCount 加 1
+ Redis 初始化时会创建一万个字符串对象，包含从 0 到 9999 的所有整数值
+ 特点
    - **只共享整数值的字符串的对象，不共享其他字符串或其他对象。**因为共享对象需要验证操作，对于整数复杂度为 O(1)，对于字符串为 O(N)，对于其他对象则更复杂

#### 对象的空转时长（idletime）
+ 当前时间减去 lru 时间则得出空转时长，较高的可能会在 redis 达到 maxmemory 时被释放回收

## 单机数据库
### 数据库
```c
struct redisServer {
	// ...
    
    // 服务器中所有的数据库
    redisDb *db;
    
    // 数据库数量，默认为 16
    int dbnum;
    
    // AOF 缓冲区
    sds aof_buf;
    
    // ...
};

struct redisClient {
    // ...
    
    // 记录客户端当前正在使用的数据库，指向 redisServer 的 db 数组中的某一个元素
    redisDb *db;
    
    // ...
};

typedef struct redisDb {
    // ...
    
    // 数据库键空间，使用字典来保存数据库中的所有键值对
    dict *dict;
    
    // 过期字典，保存键的过期时间，键 -> long long 类型的 Unix 时间戳，精确到毫秒
    dict *expires;
    
    // ...
} redisDb;
```

### 过期键的处理
+ 惰性删除
    - 所有读写数据库的 Redis 命令在执行之前都会调用 expireIfNedded 函数对输入键进行检查，如果输入键过期就删除
+ 定期删除
    - 定期从一定数量的数据库中取出一定数量的随机键进行检查，删除其中过期键
    - 全局变量 current_db 记录当前函数检查进度，在下一次调用该函数时接着上一次的进度继续处理
+ 生成 RDB 文件
    - 执行 SAVE 或者 BGSAVE 时会对数据库中的键进行检查，已过期的键不会保存到 RDB 文件中
+ 载入 RDB 文件
    - 主服务器不会载入过期键
    - 从服务器会载入 RDB 文件的所有键，但从服务器会采用它自己的逻辑时钟来判断读取时键是否应当过期，过期则返回不存在（即使数据仍然在内存中）
+ AOF 文件写入
    - 向 AOF 文件追加一条 DEL message 命令即可
+ AOF 重写
    - 不会影响，因为由 DEL 命令
+ 从节点的过期键删除由主服务器控制
    - 主服务器向从服务器发送 DEL 命令删除过期节点
    - 这样做是为了保证从服务器数据的一致性，但是可能导致从节点的数据过期了还依旧存在
    - 在 Redis 3.2 中修复了从服务器返回过期键对应的值的问题，即直接返回空，但还是不删除过期键，等待主服务器的 DEL 命令
        * [https://github.com/redis/redis/commit/06e76bc3e22dd72a30a8a614d367246b03ff1312](https://github.com/redis/redis/commit/06e76bc3e22dd72a30a8a614d367246b03ff1312)	
    - ![](https://cdn.nlark.com/yuque/0/2022/png/1732113/1644304912376-8affedaa-ef56-4c5b-951f-80007c72bc5c.png)

### 通知
+ 客户端可以通过订阅给定的频道或者模式，来获知数据库中**键**的变化，或数据库中命令的执行情况

### 持久化
#### RDB
+ RDB 文件用于保存和还原 Redis 服务器中**所有数据库**的**所有键值对**
+ 执行方式
    - `SAVE`：阻塞服务器进程进行持久化，直到持久化完成
    - `BGSAVE`：通过`fork()`创建子进程，不会阻塞服务器
+ 执行情况
    - 手动触发
    - 按条件触发
        * 如多少秒内对数据库进行了至少 n 次修改
        * 服务器状态维持了一个 dirty 计数器和 lastsave 属性，如果 dirty 值和距离 lastsave 的时间间隔超过配置，就触发`BGSAVE`

#### AOF（Append Only File）
+ AOF 文件保存服务器所执行的写命令（有点像 Redo log）
+ 实现步骤
    - 命令追加
        * 执行一条命令后，追加到`redisServer`中的`aof_buf`缓冲区（类型为`sds`）
    - 文件写入
        * 将`aof_buf`的内容写入到具体的 AOF 文件中
    - 文件同步
        * 操作系统为提高文件写入效率，在调用`write()`时，有时会先将数据保存到内核缓冲区，等到缓冲区满或者超过指定时间后才写入到磁盘，因此可能存在数据丢失问题
        * Redis 提供了 3 个 appendfsync 的属性值
            + always：每次都同步写入到磁盘，最安全但是最慢
            + every_sec：距离上次同步超过一秒就写入到磁盘，折中方案，最多丢失一秒的命令数据
            + no：只调用`write()`，何时同步由操作系统决定
+ 载入步骤
    - 创建伪客户端
    - 从 AOF 文件分析并读取一条命令
    - 使用伪客户端发送被读出的命令给服务器
    - 直到 AOF 文件中的命令处理完为止
+ AOF 重写（`BGREWRITEAOF`）
    - 随着时间流逝，服务器运行，AOF 文件的体积可能越来越大，进行数据还原的时间就越多
    - Redis 通过 AOF 重写，创建一个**新的 AOF 文件来替代旧的 AOF 文件**，新的 AOF 文件不会包含任何浪费空间的冗余命令
    - **AOF 重写直接读取数据库中的键值对进行重写**，不会操作旧的 AOF 文件
    - 重写时会多创建一个 AOF 重写缓冲区，在子进程重写完成时将重写缓冲区中的内容写到新的 AOF 文件中



