## InnoDB的Buffer Pool
+ 缓存的重要性

用户数据的索引（聚簇索引和二级索引），系统数据，都是以页的形式存放于表空间中，表空间只不过是InnoDB对文件系统上一个或几个实际文件的抽象，实际还是存储在磁盘上。

将页读取到内存中，读写访问后不急着把内存空间释放掉，而是**缓存**起来，下次访问时可以省去磁盘I/O的开销。

+ Buffer Pool
    - MySQL启动时向操作系统申请的一片连续内存
    - 默认情况下为**128M**（在my.ini中配置，server项的`innodb_buffer_pool_size`）

### Buffer Pool内部组成
+ 默认缓存页和磁盘上默认的页大小一样，都是**16KB**
+ 设置的`innodb_buffer_pool_size`并不包含控制快的内存大小，所以申请的这片内存空间一般会比这个值大5%左右
+ 每个缓存页都有对应的**控制信息**，控制信息包括该页所属的表空间编号、页号、缓存页在Buffer Pool中的地址、链表节点信息、一些锁信息以及LSN信息

![](https://cdn.nlark.com/yuque/0/2021/png/1732113/1635579884391-7f0264ac-cba9-403e-bc66-37dc589ce8d9.png)

### free链表
把所有空闲的缓存页对应的控制快作为一个节点放到一个链表中

![](https://cdn.nlark.com/yuque/0/2021/png/1732113/1635580044114-6ef2ac02-f660-4e01-a0b0-30fc2fbba34d.png)

链表的基节点占用的内存空间不包含在Buffer Poll的内存空间中，是另外单独申请的。**每当需要从磁盘加载一个页到Buffer Pool中时，就从free链表中取一个空闲的缓存页，并把该缓存页对应的控制块的信息填上（表空间、页号等），并从free链表移除此节点**

### 缓存页的哈希处理
根据**表空间号 + 页号**定位一个页，可以把这个组合当作key，缓存页当作value建立哈希表。

访问某个页 -> 从哈希表中找表空间 + 页号对应的缓存页 -> 有则使用，没有则从free链表取空闲缓存页然后把磁盘上的页加载到缓存页

### flush链表
如果修改了缓存页的数据，那它就和磁盘上的页不一致，这样的缓存页称为**脏页**。脏页会在某个时间点同步到磁盘。

flush链表是存储脏页的链表，同时也有一个flush链表基节点，这个链表对应的缓存页都是需要刷新到磁盘上的。

![](https://cdn.nlark.com/yuque/0/2021/png/1732113/1635580583967-be7cbc21-ab16-40f4-a979-79acabdd1813.png)

### LRU链表的管理
+ 一共访问了n次页，被访问的页已经在缓存中的次数除以n就是**缓存命中率**

#### 简单的LRU链表（最近最少使用原则）
+ 访问某个页
    - 该页不在Buffer Pool中，加载到缓存页后把对应的控制块作为节点塞到链表的头部
        * 空闲缓存页用完，则淘汰尾部缓存页
    - 该页已在Buffer Pool中，把对应的控制块移动到LRU链表的头部
+ 缺点
    - InnoDB会预读
        * 线性预读

顺序访问某个区的页面超过`innodb_read_ahead_threshold`（默认值56）就会触发一次**异步**读取下一个区中全部的页面到Buffer Pool的请求

        * 随机预读（默认不开启，通过参数`innodb_random_read_ahead`设置）

Buffer Pool中已经缓存了某个区的13个连续的页面（并且这13个页面位于young区域的前1/4），会触发一次**异步**读取本区所有页面到Buffer Pool的请求

预读到Buffer Pool中的页如果用得到是好事，如果用不到，而这些预读页都放到LRU链表的头部，此时Buffer Pool容量已满就会导致处在尾部的缓存页被淘汰。

**劣币驱逐良币，降低了缓存命中率。**

    - 查询语句扫描全表，很可能将Buffer Pool中的页全部换掉，**影响其他查询对Buffer Pool的使用**，降低缓存命中率
+ 概括缺点
    - 加载到Buffer Pool中的页不一定用得到
    - 非常多使用频率偏低的页同时加载到Buffer Pool，可能把使用频率非常高的页从Buffer Pool淘汰掉

#### 划分区域的LRU链表
+ 使用频率非常高的缓存页 -> 热数据 -> young区域
+ 使用频率不是很高的缓存页 -> 冷数据 -> old区域

![](https://cdn.nlark.com/yuque/0/2021/png/1732113/1635581736188-e0a8cf1f-f66a-4f09-bec7-0e70922fd572.png)

LRU链表按照比例划分（`innodb_old_blocks_pct`，默认37，即old区域大约占3/8）

+ 针对预读的页面可能不进行后续访问情况的优化

当磁盘上的页初次加载到Buffer Pool中时，该缓存页对应的控制块会被放到old区域到头部，这样就不会影响到young区域中使用频繁的缓存页，而且不进行后续访问的话这些预读页也会在old区域中被淘汰。

+ 针对全表扫描时短时间内访问大量使用频率非常低的页的情况的优化

全表扫描时，首次被加载到Buffer Pool的页被放到了old区，但是很快又会访问到这个页（全表扫描指每次去页面取一条记录）。按规则，每次进行访问的时候又会把该页放到young区域的头部，这样又会把使用频率较高的页面顶下去。

所以规定，在某个处在old区域的缓存页进行第一次访问时在它的对应的控制块记录访问时间，后续访问时间与第一次访问时间在某个时间间隔（`innodb_old_blocks_time`）内就不会将该页面从old区域移动到young头部，否则移动到young区域的头部。

#### 更进一步优化LRU链表
只有访问的缓存页位于young区域的1/4的后边才将其移动到LRU链表的头部

+ 这样降低了LRU链表调整的频率

### 刷新脏页到磁盘
+ 从LRU链表的冷数据中定时刷新一部分页面（脏页）到磁盘（`BUF_FLUSH_LRU`）
+ 从flush链表中定时刷新一部分页面到磁盘（取决于系统是否繁忙）（`BUF_FLUSH_LIST`）
+ 有时候后台线程刷新脏页不及时，用户线程准备加载一个磁盘页到Buffer Pool时没有可用的缓存页，这时会看看LRU链表有没有可以直接释放掉的未修改页面，如果没有的话会不得不将LRU链表的尾部的一个脏页同步刷新到磁盘。（`BUF_FLUSH_SINGLE_PAGE`）

### 多个Buffer Pool实例
多线程环境下，访问Buffer Pool中各种链表需要加锁，单一的Buffer Pool可能会影响请求的处理速度。

![](https://cdn.nlark.com/yuque/0/2021/png/1732113/1635584061952-c176197b-e29a-4a0a-a1f3-46f88d997717.png)

## 事务
### 支持事务的存储引擎
+ InnoDB
+ NDB

其余如MyISAM不支持事务

### 四大特性
+ Atomicity 原子性
    - 要么全做要么全不做
+ Consistency 一致性
    - 符合约束
        * 数据库本身能保证一部分一致性需求（NOTNULL列不会有NULL值等）
        * 更多的一致性靠代码保证
    - 如果不遵循原子性，那最后不符合一致性需求
    - 如果不遵循隔离性，那最后不符合一致性需求
    - **原子性和隔离性是保证一致性的一种手段**
    - **满足了原子性和隔离性不一定满足一致性，满足了一致性也不一定就满足原子性和隔离性**
+ Isolation 隔离性
    - 其他的状态转换不会影响到本次状态转换
+ Durability 持久性
    - 影响是持久的

### 事务概念
**需要满足四大特性的一个或多个数据库操作就称为一个事务**

+ 状态
    - 活动的 active 数据库操作执行中
    - 部分提交的 partially committed 事务最后一个操作执行完成，但操作都在内存中执行，还没有刷新到磁盘
    - 失败的 failed 处于上述两个状态，遇到错误（断电、数据库错误、操作系统错误或人为停止）无法继续执行
    - 中止的 aborted 事务处于失败状态，就需要撤销失败事务对当前数据库造成的影响，也就是**回滚**，回滚完毕时数据库恢复到了执行事务之前的状态，就说事务处在了中止状态
    - 提交的 committed 处在部分提交的事务将修改过的数据同步到磁盘上，事务就处在提交的状态

![](https://cdn.nlark.com/yuque/0/2021/png/1732113/1635585858311-ebe6f651-bee3-40ac-a1ea-977df650aa16.png)

### 事务语法
#### 开启事务
+ `BEGIN [WORK];`
+ `START TRANSACTION;`
    - 只读事务            `START TRANSACTION READ ONLY;`
    - 读写事务（默认）`START TRANSACTION READ WRITE;`
    - 一致性读（可以与上面两个一起）
        * `START TRANSACTION READ WRITE, WITH CONSISTENT SNAPSHOT;`

#### 提交事务
+ `COMMIT [WORK];`

#### 回滚事务
+ `ROLLBACK [WORK];`
+ 事务遇到错误无法继续执行会自己回滚

#### 隐式提交
+ 定义或修改数据库对象 DDL
+ 修改mysql数据库中的表
+ 事务控制或锁定
    - 在一个事务还没提交或者回滚时就又使用`START TRANSACTION`，会隐式提交还没结束的事务
    - 使用`LOCK TABLES`、`UNLOCK TABLES`会隐式提交还没结束的事务
+ 保存点
    - 开启了一个事务
    - 可以添加保存点`SAVEPOINT 保存点名称;`
    - 后面回滚的时候可以`ROLLBACK TO 保存点名称;`

## Redo log
`InnoDB`以页为单位管理存储空间，增删改查操作本质上都是在访问页面。

+ 每次在事务提交完成之前把事务修改的所有页面刷新到磁盘
    - 刷新一个完整的数据页太浪费
    - 随机IO刷新较慢（事务修改的页可能并不相邻）
+ 每次事务提交把该事务需要修改的东西记录一下，比如

> 将第0号表空间的100号页面的偏移量为1000处的值更新为2
>

    - redo log占用的空间非常小（存储表空间ID、页号、偏移量、需要更新的值）
    - redo log顺序写入磁盘（顺序IO）

### 格式
![](https://cdn.nlark.com/yuque/0/2021/webp/1732113/1635662770147-2c7dbdd7-e2a5-4a91-ae4b-fb33ce77f49f.webp)

+ type：类型（`5.7.21`有53种）
+ space ID：表空间ID
+ page number：页号
+ data：具体内容

#### type
+ `MLOG_NBYTE`：N = 1 2 4 8

![](https://cdn.nlark.com/yuque/0/2021/webp/1732113/1635663280238-7aa7c62a-5c3a-4b99-9b86-41b7f7679581.webp)

在页面的某个偏移量处写入N个字节的数据

+ `MLOG_WRITE_STRING`：

![](https://cdn.nlark.com/yuque/0/2021/webp/1732113/1635663442778-594ab955-b0ef-4c5c-8edc-25981ed1fced.webp)

在页面的某个偏移量处写入len个字节的数据

+ `MLOG_REC_INSERT`：

插入一条使用非紧凑行格式的记录（Redundant）

+ `MLOG_COMP_REC_INSERT`：

插入一条使用紧凑行格式的记录（Compact、Dynamic、Compressed）

    - `n_uniques`
        * 聚簇索引：主键列数
        * 二级索引：索引列数 + 主键列数
    - `field1_len` ~ `fieldn_len`：记录字段占用存储空间的大小（不管类型是否是固定长度大小，都要写入到这里）
    - `offset`：该记录的前一条记录在页面中的日志（修改前一条记录的next_record使其指向新记录）
    - `end_seg_len`：可以算出一条记录占用存储空间的总大小

`MLOG_COMP_REC_DELETE`：

删除

+ `MLOG_COMP_PAGE_CREATE`：

创建一个存储紧凑行格式记录的页面

+ `MLOG_COMP_LIST_START_DELETE`：

从某条给定记录开始删除页面中的一系列使用紧凑行格式记录

`MLOG_COMP_LIST_END_DELETE`：

从START到END的记录删除

+ `MLOG_ZIP_PAGE_COMPRESS`：

压缩一个数据页

### Mini-Transaction（mtr）
#### 概念
一个事务可以包含若干条语句，每一条语句由若干mtr组成，每一个mtr包含若干条redo log

#### 以组的形式写入redo log
执行需要保证原子性的操作时必须以组的形式来记录redo log，在恢复时针对某个组中的redo log，要么全部恢复，要么都不恢复：

+ 多条redo log的原子性操作：在末尾加入一条`MLOG_MULTI_REC_END`的redo log

![](https://cdn.nlark.com/yuque/0/2021/webp/1732113/1635664978969-9d8a6cf4-25ec-4553-ae50-10ffe61c7601.webp)

+ 单条redo log日志：用type的第1个比特位来表示

![](https://cdn.nlark.com/yuque/0/2021/webp/1732113/1635664995020-0dc8c0f1-1a83-4cb2-bfc5-450ec2e0e835.webp)

### redo log写入过程
+ redo log block（512字节）

![](https://cdn.nlark.com/yuque/0/2021/webp/1732113/1635665501003-2c2ac854-dbbd-4ede-adce-81bf35f673cc.webp)

### redo log buffer
![](https://cdn.nlark.com/yuque/0/2021/webp/1732113/1635665544592-05f2e81a-b3eb-4110-a8de-e0a701a36dbe.webp)

默认16MB，redo日志也并不直接写到磁盘上，而是先写到这个缓冲区。

![](https://cdn.nlark.com/yuque/0/2021/webp/1732113/1635665589589-998f9c98-0191-4437-9b71-35b5e550a688.webp)

向`log buffer`写的过程是顺序的，先写前面的block，再写后面的block。全局变量`buf_free`指明后续写入的redo log应该写入到哪个位置。

+ 假设现在有两个事务T1、T2，每个事务包含两个mtr

![](https://cdn.nlark.com/yuque/0/2021/webp/1732113/1635665734526-72d7ee65-7be6-4d18-964b-1261b37ace80.webp)

+ 事务并发执行，因此T1、T2之间的mtr可能交替执行

![](https://cdn.nlark.com/yuque/0/2021/webp/1732113/1635665759649-203028ca-01bc-453a-86cb-71e5c94d1195.webp)

### redo log文件
#### 刷盘时机
在一些情况下会刷新到磁盘里：

+ `log buffer`空间不足（`log buffer`容量已用了一半时）
+ 事务提交时（保证持久性）
+ 将某个脏页刷新到磁盘前，要先把该脏页对应的redo log刷新到磁盘中（redo log是顺序刷新的，所以将某个脏页对应的redo log从log buffer刷新到磁盘时，也会保证之前产生的redo log也刷新到磁盘）
+ 后台线程定时刷新（大约每秒一次）
+ 正常关闭服务器
+ 做`checkpoint`

#### 文件组
`ib_logfile0`和`ib_logfile1`（可以配置多个，可以配置目录，可以配置每个的大小）

![](https://cdn.nlark.com/yuque/0/2021/webp/1732113/1635666194576-0a69cc62-8f0c-4f6e-b4df-46461ee6b97e.webp)

写的时候，如果最后一个写完了，会覆盖写第一个，再继续往后覆盖写。（所以提出`checkpoint`的概念）

![](https://cdn.nlark.com/yuque/0/2021/webp/1732113/1635666284091-977ea939-b61c-4915-b431-ed0c28d31892.webp)

前4个block记录管理信息，从第5个开始存`log buffer`镜像

### LSN - Log Sequence Number
+ 记录已经写入的redo log的量
+ 初始值为8704
+ 统计LSN的增长量时，是按照实际写入的日志量加上占用的`log block header`和`log block trailer`来计算的
    - 系统第一次启动后初始化`log buffer`，lsn的值加12

![](https://cdn.nlark.com/yuque/0/2021/webp/1732113/1635728176927-1bd472c4-be4f-41eb-aece-81a0e640fdcb.webp)

    - 某个mtr产生的一组redo log较小，当前block剩余空间足够容纳，则lsn的增长量就是redo log占用的字节数

![](https://cdn.nlark.com/yuque/0/2021/webp/1732113/1635728231850-a1ac7ef1-fd18-4e05-85cc-29a54c5662c5.webp)

    - 如果某个mtr产生的一组redo log占用空间较大，占用了多个block，则lsn的增量应包括新block的header和旧block的trailer

![](https://cdn.nlark.com/yuque/0/2021/webp/1732113/1635728282325-641fd2e5-37e9-4046-95f7-acc2eac2c6b1.webp)

+ 可以看出每一组由mtr生成的redo log都有一个唯一的LSN值与其对应，LSN值越小，说明redo log产生越早

#### flushed_to_disk_lsn
redo log是先写到log buffer（内存中），之后才刷新到磁盘上的redo log文件。

所以用了一个全局变量`buf_next_to_write`来标记当前log buffer已经有哪些日志被刷新到磁盘中了。

![](https://cdn.nlark.com/yuque/0/2021/webp/1732113/1635728515146-ebf0e556-a84b-4592-9432-f7b677af8b50.webp)

上面所讲的lsn是表示当前系统中写入的redo log，包括写入到log buffer而没有刷新到磁盘的日志。相应的，变量`flushed_to_disk_lsn`和`write_lsn`用来表示刷新到磁盘上了的redo log的lsn。

+ `flushed_to_disk_lsn`：真实写入到磁盘
+ `write_lsn`：仅写入到操作系统缓冲区，没有显式刷新到磁盘

#### LSN值与redo log偏移量的对应关系
一个mtr中产生多少日志，lsn的值就增加多少

初始LSN = 8704，对应文件偏移量2048。

![](https://cdn.nlark.com/yuque/0/2021/webp/1732113/1635728971422-bac0e650-e8ac-4459-ba3c-a3926c5a6ff9.webp)

#### flush链表中的LSN
+ 按照第一次修改发生的时间顺序排序（按照oldest_modification排序）
+ 被多次更新的页面不会重复插入到flush链表，但是会更新newest_modification的值
+ oldest_modification记录页面被加载到buffer pool后第一次修改时的mtr开始前的lsn
+ newest_modification记录修改该页面的mtr结束时对应的lsn

![](https://cdn.nlark.com/yuque/0/2021/webp/1732113/1635729543245-db5bb139-6f43-457e-8a39-3a9eb95f81bb.webp)

### checkpoint
![](https://cdn.nlark.com/yuque/0/2021/webp/1732113/1635729908131-babf495b-99c3-4ad8-a22e-94d3f6486a75.webp)

假如此时页a刷新到磁盘上，那么他对应的控制块就会从flush链表中移除，这样mtr_1生成的redo log就没有用了。

因此使用全局变量`checkpoint_lsn`来代表当前系统中可以被覆盖的redo log总量是多少，初始值8704。

页a刷新到了磁盘，则mtr_1生成的redo log可以被覆盖了，因此进行增加`checkpoint_lsn`的操作：

1. 计算当前可以覆盖的redo log的最大lsn值是多少

redo log可以被覆盖意味着脏页刷新到了磁盘，则flush链表中没有对应该页的控制块。因此只需要找到flush链表的最后一个节点（oldest_modification最小）的oldest_modification，把该值赋给`checkpoint_lsn`即可。

2. 将checkpoint_lsn和对应redo log文件组偏移量以及此次checkpoint的编号写到文件中（`checkpoint1`或`checkpoint2`）

变量`checkpoint_no`记录了目前系统做了多少次`checkpoint`，每做一次，该变量的值就增加1，计算出checkpoint_lsn在redo log文件中的偏移量`checkpoint_offset`，然后把这3个值写入到redo log文件组的管理信息中。

checkpoint_no奇数就写到`checkpoint2`，偶数就写到`checkpoint1`

![](https://cdn.nlark.com/yuque/0/2021/webp/1732113/1635733293505-2d833df9-3095-489a-b31f-2f41c11664b0.webp)

## 崩溃恢复（Redo log的作用）
在服务器不挂的情况下， redo log作用不大。

### 确定恢复的起点
对于`checkpoint_lsn`之前的redo log都可以被覆盖，因为这些日志对应的脏页已经被刷新到磁盘了。

而对于`checkpoint_lsn`之后的redo log可能没被刷盘，也可能被刷盘了，没有办法确定，所以需要从`checkpoint_lsn`开始读取redo log来恢复页面。

redo log文件组第一个文件管理信息中两个block都存储了`checkpoint_lsn`的信息，选其中`checkpoint_no`更大的，就是最近的一次`checkpoint`信息。

### 确定恢复的终点
普通的block的头部有一个`LOG_BLOCK_HDR_DATA_LEN`属性，该属性记录了此block用了多少字节的空间，如果该属性的值不为512，那它就是恢复终点。

### 怎么恢复
+ 使用哈希表
    - 将每一条redo log以（表空间ID + 页号， redo log）的形式放入哈希表，之后再遍历哈希表进行恢复，这样按页恢复可以减少随机I/O
    - 注意按照redo log的生成时间恢复，即同一个页面的redo log日志是按照生成时间顺序排序的，恢复也要按照这个顺序恢复
+ 跳过已经刷新道磁盘的页面
    - 页的`File Header`中的`FIL_PAGE_LSN`记录了最近一次修改此页面的lsn值（控制块的newest_modification），如果某次做了checkpoint之后有脏页刷新道磁盘中，那么该页对应的FIL_PAGE_LSN代表的lsn值肯定大于`checkpoint_lsn`的值，凡是符合这种情况的页面就不需要再执行lsn值小于`FIL_PAGE_LSN`的redo log了

## Undo log（事务原子性）
事务需要保证原子性，要么全部完成要么什么都不做。如果做了的话要想怎么回滚，比如：

+ 插入记录，至少要记下记录主键值，回滚的时候把主键值对应的记录删除
+ 删除记录，至少要记下记录所有内容，回滚的时候把由这些内容组成的记录插入到表中
+ 修改记录，至少要记下这条记录的旧值，回滚的时候把记录更新为旧值

为了回滚而记录的内容称为撤销日志，即`undo log`。

### 事务ID
事务有只读事务和读写事务

+ 只读事务不可以对普通表（其他事务也能访问到的表）进行增删改操作，但可以对临时表做增删改操作
+ 读写食物可以对表进行增删改操作

#### 分配事务ID的情况
如果某个事务执行过程对某个表执行了增删改操作，那么InnoDB引擎就会给它分配一个唯一的事务ID（对于MySQL 5.7来说）：

+ 对于只读事务，只有在它第一次对某个用户创建的临时表执行增删改操作时才会为这个事务分配一个事务ID，否则不分配

> 临时表指CREATE TEMPORARY TABLE创建的表，不是内部临时表，在事务回滚时并不需要把执行SELECT语句中用到的内部临时表回滚，用到内部临时表也不会分配事务ID
>

+ 对于读写事务，只有在它第一次对某个表（包括用户创建的临时表）执行增删改操作才会分配事务ID，否则不分配

#### 事务ID的生成（类似隐藏列`row_id`）
+ 服务器在内存中维护一个全局变量，需要为某个事物分配事务ID时赋值并自增
+ 每当这个变量的值为256的倍数时，把该变量的值刷新到系统表空间的页号为5的页面中一个称之为`Max Trx ID`处
+ 下一次系统启动时加载`Max Trx ID`并将其加上256（防止重复）

### trx_id隐藏列
前面讲行格式的时候提到3个隐藏列，其中的`trx_id`就是存事务ID，代表改动时（`INSERT`/`UPDATE`/`DELETE`）对应的记录ID

### roll_pointer隐藏列的含义
本质就是指向undo log的一个指针。Undo log存储到了类型为`FIL_PAGE_UNDO_LOG`的页中，而数据存储在`FIL_PAGE_INDEX`的页中，也就是数据页。

![](https://cdn.nlark.com/yuque/0/2021/webp/1732113/1636170093527-daf212ec-beda-40b8-9e46-9c77a7e11a5f.webp)

### Undo log格式
#### INSERT操作
+ 记录主键各列的存储空间大小和真实值（有多列要全都记录）

只记录主键信息就可以同时处理聚簇索引和二级索引中的记录了

![](https://cdn.nlark.com/yuque/0/2021/webp/1732113/1636170119595-84ba1c28-2a17-463d-b728-f761f1a0a504.webp)

```plain
BEGIN;  # 显式开启一个事务，假设该事务的id为100

# 插入两条记录
INSERT INTO undo_demo(id, key1, col) 
    VALUES (1, 'AWM', '狙击枪'), (2, 'M416', '步枪');
```

![](https://cdn.nlark.com/yuque/0/2021/webp/1732113/1636170145035-f384b816-ff2f-42ec-b26d-e7ce31f9978c.webp)

![](https://cdn.nlark.com/yuque/0/2021/webp/1732113/1636170153080-5a7ebd04-2aa2-477c-b87f-7dc8b74195ea.webp)

#### DELETE操作
`Page Header`中的`PAGE_FREE`指向垃圾链表的头结点。

执行DELETE语句有两个阶段：

1. 将记录的`delete_mask`标识位设置为1，修改记录的trx_id、roll_pointer列的值

![](https://cdn.nlark.com/yuque/0/2021/webp/1732113/1636170379923-138009c0-b99c-4739-92f2-800fda63defb.webp)

2. 事务提交后，从正常记录链表移除，加入到垃圾链表，调整页面其他信息（用户记录数量`PAGE_N_RECS`、上次插入记录的位置`PAGE_LAST_INSERT`、垃圾链表头结点`PAGE_FREE`、页面可重用字节数量`PAGE_GARBAGE`等），这个阶段称为purge

![](https://cdn.nlark.com/yuque/0/2021/webp/1732113/1636170492308-16a4cfa7-610b-4671-92c4-7cb92f8db7ce.webp)

从上面的过程可以看到，删除语句所在的事务提交之前，只会经历阶段一（提交之后就不用回滚了，所以只需要对阶段一的影响考虑回滚）。

![](https://cdn.nlark.com/yuque/0/2021/webp/1732113/1636170598055-7f66db8e-f80b-4952-a75e-56544adfc78f.webp)

+ 在进行`delete mark`操作前，需要先记录下旧的trx_id和roll_pointer，形成版本链

![](https://cdn.nlark.com/yuque/0/2021/webp/1732113/1636170674080-f2ce00d2-45d1-4aad-892c-a43ebda870ca.webp)

+ 这个undo log还记录了各索引列的信息，主要用于purge阶段

#### UPDATE操作（不更新主键）
+ 就地更新
    - 更新后的列和更新前的列占用的存储空间一样大（是一样大，不能小于也不能大于）
+ 先删掉旧记录，再插入新记录（不能就地更新的情况下）
    - 这里的删除指的是真正删除，也就是把这条正常记录移动到垃圾链表中并修改页面统计信息，所用的线程不是DELETE的purge操作的专门线程，而是用户线程同步执行
    - 新记录空间不超过旧记录空间时可以重用加入到垃圾链表的旧记录所占用的存储空间

否则申请一段空间供新记录使用

如果页内没有空间就进行页分裂，再插入新记录

![](https://cdn.nlark.com/yuque/0/2021/webp/1732113/1636171149652-d0ee6793-dc9e-4b13-8dd6-15f8957139c7.webp)

#### UPDATE操作（更新主键）
+ 将旧记录进行`delete mark`操作（为的是事务提交前MVCC的其他事务能访问到）
+ 根据更新后各列的值创建新记录，然后插入到聚簇索引中（需要重新定位插入的位置）

这个过程中`delete mark`会有一条undo log（`TRX_UNDO_DEL_MARK_REC`），插入新记录也会有一条undo log（`TRX_UNDO_INSERT_REC`）。（其实还有一条`TRX_UNDO_UPD_DEL_REC`）

### FIL_PAGE_UNDO_LOG页面
![](https://cdn.nlark.com/yuque/0/2021/webp/1732113/1636181663236-56a19fa8-af6a-4153-8af3-f39f6590ba56.webp)

+ `TRX_UNDO_PAGE_TYPE`：undo log类型
    - `TRX_UNDO_INSERT`：用十进制1表示，一般由INSERT语句产生，在UPDATE语句更新主键的情况也会产生此类型的undo log
    - `TRX_UNDO_UPDATE`：用十进制2表示，除了上面的情况，其它类型都属于此类

不同大类的undo log不能混着存储。之所以分类是因为`TRX_UNDO_INSERT_REC`的undo log在事务提交后可以删除，而其他类型的undo log还要为mvcc服务。

+ `TRX_UNDO_PAGE_START`：第一条undo log在此页面的起始偏移量
+ `TRX_UNDO_PAGE_FREE`：最后一条undo log结束时的偏移量，从这个位置开始可以继续写入新的undo log
+ `TRX_UNDO_PAGE_NODE`：代表一个`List Node`结构

### Undo页面链表
#### 单个事务
最多会有4个undo log链表（普通表和临时表的操作undo log不混存，INSERT类型和其它类型的undo log不混存），按需分配，不用就不分配。

![](https://cdn.nlark.com/yuque/0/2021/webp/1732113/1636182095981-2c89a10f-7d94-4a30-a8ca-16a769501bec.webp)

#### 多个事务
为提高undo log写入效率，不同事务的undo log写入到不同的页面链表中

![](https://cdn.nlark.com/yuque/0/2021/webp/1732113/1636182180284-32d9a55d-df79-41cd-94d5-76931d6def57.webp)

### Undo log具体写入过程（略）
每一个Undo页面都对应一个段，链表中的页面都从段中申请，链表第一个页面包含段信息。

### Undo log的重用
满足两个条件即可：

+ 该链表只包含一个undo log页面（太多了不好维护）
+ 已使用空间小于页面空间的3/4

重用的规则：

+ 对insert的undo log
    - 直接覆盖即可
+ 对update的undo log
    - 只能在结尾继续追加，不可以覆盖（因为MVCC）

### 回滚段
方便管理undo log页面链表。

#### 从回滚段申请Undo log链表
+ 从undo slot找，找有没有值为`FIL_NULL`的
    - 如果有，那就在表空间新建一个段，从段里申请一个页面作为undo log链表的第一个页面，把这个页号填到这个slot里（意味着把这个undo slot分配给这个事务）
    - 如果找不到，就说明当前事务占满了undo slot，就会回滚这个事务并且给用户报错

`Too many active concurrent transactions`

+ 事务提交时
    - 如果undo slot指向的Undolog页面符合重用条件，就缓存他，把它加入到undo cached链表（分insert的和update的），新事务分配时先找对应的链表
    - 如果不能重用
        * insert 的undo log页面会被释放，undo slot值变为FIL_NULL
        * update的会被放到History链表，Undo log不会被释放，undo slot值变为FIL_NULL

显然一个回滚段不太够用（只支持1024个事务并发）

### 多个回滚段
InnoDB后来定义了128个回滚段（一个回滚段只有一个`Rollback Segment Header`页面），则支持128 x 1024 = 131072个事务。

![](https://cdn.nlark.com/yuque/0/2021/webp/1732113/1636186801136-8973a79f-ad72-43d5-afeb-ab8081f61e3b.webp)

### 为事务分配Undo页面链表过程
+ 对普通表记录首次改动，到系统表空间5号页面分配一个回滚段（采用循环使用的方式来分配回滚段，当前事务分配了第0号回滚段，那么下一个事务要分配第33号回滚段，再下一个是34号）
+ 分配到回滚段，首先看回滚段的cached链表有没有缓存了的undo slot，如果有就把缓存的undo slot分配给该事务
+ 如果没有，则要到`Rollback Segment Header`找一个可用的undo slot分配给当前事务
+ 找到可用的undo slot，如果该slot是从cached链表获取的，那它对应的Undo log Segment已经分配了，否则要重新分配一个Undo log Segment，然后从该段申请一个页面作为Undo log链表的first undo page
+ 然后在上面写undo log记录

