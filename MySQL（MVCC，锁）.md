```sql
CREATE TABLE hero (
    number INT,
    name VARCHAR(100),
    country varchar(100),
    PRIMARY KEY (number)
) Engine=InnoDB CHARSET=utf8;

INSERT INTO hero VALUES(1, '刘备', '蜀');

+--------+--------+---------+
| number | name   | country |
+--------+--------+---------+
|      1 | 刘备   | 蜀      |
+--------+--------+---------+
```

+ 假设插入这条记录时事务ID为80，则记录示意图如下

![](https://cdn.nlark.com/yuque/0/2021/webp/1732113/1636189219152-78cb5c96-8f4d-466a-b602-197897debc07.webp)

## 事务隔离级别
+ 脏写

一个事务修改了另一个未提交事务修改过的数据

Session B中的事务回滚，导致Session A中的更新不复存在。

![](https://cdn.nlark.com/yuque/0/2021/webp/1732113/1636187638645-7de11c6c-781d-440f-9329-4fd512b08b55.webp)

+ 脏读

一个事务读到了另一个未提交事务修改过的数据

Session B中的事务回滚，导致Session A中的事务读到了一个不存在的数据。

![](https://cdn.nlark.com/yuque/0/2021/webp/1732113/1636187821757-33c88dd8-9bf4-4908-9aad-035195637e14.webp)

+ 不可重复读

一个事务只能读到另一个已提交事务修改过的数据，并且其他事务每对该数据进行一个修改并提交后，该事务都能查到最新值，那就意味着发生了不可重复读。

![](https://cdn.nlark.com/yuque/0/2021/webp/1732113/1636187958750-234ef83d-8a88-4b4c-b8cb-e028db2f41aa.webp)

Session B提交的是隐式事务（语句结束就提交了），每次事务提交后，事务A都能看到最新的值，这种现象称为不可重复读。

+ 幻读

一个事务根据某些条件查询记录，之后另一个事务又向该表插入符合这个条件的数据，原先的事务再查询时能把另一个事务插入的记录也查出来，那就意味着发生了幻读。

![](https://cdn.nlark.com/yuque/0/2021/webp/1732113/1636188120844-30bd9ad4-4e24-490b-951d-263b3e03df24.webp)

**如果是Session B删除了一些符合条件的记录，之后Session A读出的记录变少了，这种算对每一条记录都发生了不可重复读，不属于幻读，幻读强调的是读到了之前读不到的数据。**

## SQL标准中的四种隔离级别
+ `READ UNCOMMITTED` / `RU`：未提交读
+ `READ COMMITTED` / `RC`：已提交读
+ `REPEATABLE READ` / `RR`：可重复读
+ `SERIALIZABLE`：可串行化

`SQL`标准规定不同隔离级别，并发事务可以发生不同严重程度的问题：

+ `READ UNCOMMITTED`可能发生脏读、不可重复读和幻读问题
+ `READ COMMITTED`可能发生不可重复读和幻读问题
+ `REPEATABLE READ`可能发生幻读问题（MySQL默认隔离级别）
+ `SERIALIZABLE`不可能发生上述问题

**脏写是所有隔离级别都不会发生的问题，脏写是最严重的问题；**

`Oracle`只支持`READ COMMITTED`和`SERIALIZABLE`；

`MySQL`的`REPEATABLE READ`可以禁止幻读问题的发生

### 如何设置事务隔离级别
```sql
SET [GLOBAL|SESSION] TRANSACTION ISOLATION LEVEL level;

level: {
	  REPEATABLE READ
  | READ COMMITTED
  | READ UNCOMMITTED
  | SERIALIZABLE
}
```

+ `GLOBAL` 全局范围影响
    - 只对执行完该语句之后产生的会话起作用
    - 当前已存在的会话无效
+ `SESSION`会话范围影响
    - 对当前会话所有后续事务有效
    - 可以在已开启的事务中执行，不影响当前正在执行的事务
    - 在事务之间执行，则对后续事务有效
+ 两个关键字都不用
    - 只对当前会话的下一个即将开启的事务有效
    - 下一个事务执行完后，后续事务恢复到之前的隔离级别
    - 该语句不能在已开启的事务中执行，会报错

## MVCC原理
### 版本链
每一条聚簇索引记录都会有两个必要的隐藏列

+ `trx_id`：事务ID
+ `roll_pointer`：修改这条记录的undo log，undo log记录了该记录修改前的信息
    - 注意，这里不包含新建时的信息，因为新建记录的insert undo log会被清除，而roll_pointer的值不会被清除，roll_pointer的第一个比特位标志着他的undo log类型，如果是1则说明指向的undo log是insert undo log类型，那么事务提交后这个指针便没意义了

假设之后两个事务ID分别为100，200的事务对这条记录进行UPDATE操作，操作流程如下：

![](https://cdn.nlark.com/yuque/0/2021/webp/1732113/1636189554055-fe9a1dfe-5f30-4d97-9a69-7246ceb55a95.webp)

> 能不能在两个事务中交叉更新同一条记录？  
答：不能，这是脏写。InnoDB使用锁来保证不会有脏写情况的发生。也就是说第一个事务更新完某条记录之后，就会给这条记录加锁，另一个事务再次更新时就需要等待第一个事务提交后才可以继续更新。
>

每次对记录进行改动都会记录一条undo log，每条undo log都会有一个old_roll_pointer来记录旧记录的roll_pointer（还有事务ID），可以将这些undo log连起来串成链表

![](https://cdn.nlark.com/yuque/0/2021/webp/1732113/1636189754023-cfa9cfcc-67e5-4f3e-a0f4-fa6becb73689.webp)

### Read View
对于使用`READ UNCOMMITTED`的事务来说，由于可以读到未提交事务修改过的记录，所以直接读每条记录的最新版本就好了。

对于使用`SERIALIZABLE`的事务，则使用加锁的方式来访问记录；

对于使用`READ COMMITTED`和`REPEATABLE READ`的事务来说，必须保证读到已经提交了的事务修改过的记录。

+ Read View包含啥
    - `m_ids` 生成Read View时当前系统中活跃读写事务的id列表
    - `min_trx_id`：`m_ids`中的最小值
    - `max_trx_id`：生成Read View时系统应该分配给下一个事务的ID值
    - `creator_trx_id`：生成该Read View时事务的事务ID
        * 只有在对表中记录改动时才会分配事务ID，否则事务ID默认为0

### 通过Read View判断记录的某个版本是否可见
+ 被访问记录的`trx_id`与Read View的`creator_trx_id`相等，则说明这个记录是当前事务修改过的，可以访问
+ `trx_id < min_trx_id`，被访问记录的事务在此事务生成Read View前已提交，可以访问
+ `trx_id >= max_trx_id`，被访问记录的事务在此事务生成Read View后才开启，不可以被当前事务访问
+ `min_trx_id <= trx_id < max_trx_id`，则判断`trx_id`是否在`m_ids`中
    - 如果在，说明Read View创建时该版本的事务还是活跃的（还没提交），该版本不可以被访问
    - 如果不在，说明Read View创建时该版本的事务已经被提交，则该版本可以被访问

如果某个版本的数据对当前事务不可见，则顺着版本链找下一个版本的数据，继续按照上面的步骤判断可见性，直至版本链的最后一个版本。如果最后一个版本都不可见，那么意味着该记录对事务完全不可见，查询结果不包含该记录。

### Read View生成时机
+ **READ COMMITTED——每次读取数据都生成一个Read View**
    - 事务100修改了数据，但还没提交，修改后数据版本链如下

![](https://cdn.nlark.com/yuque/0/2021/webp/1732113/1636267785142-77a5384b-8be2-4983-ae03-42385a66d759.webp)

    - 假设现在有一个`READ COMMITTED`级别的事务执行

`BEGIN`

`SELECT * FROM hero WHERE number = 1;`

        * 执行SELECT语句时生成一个Read VIew
            + `m_ids`: `[100, 200]`
            + `min_trx_id`: 100
            + `max_trx_id`: 200
            + `creator_trx_id`: 0（增删改才会有这个事务ID）
        * 从版本链挑选可见的记录，可见第一条记录的`trx_id`是100，处于`m_ids`中，说明该事务还是活跃的（未提交），故不符合要求，根据`roll_pointer`找下一个版本
        * 下一个版本的`trx_id`也是100，不符合要求，继续找下一个版本
        * 再下一个版本的`trx_id`是80，小于`min_trx_id`，符合要求，故最后返回给用户的记录中的`name`列的值是刘备
    - 然后将刚刚事务100提交，再到事务200更新记录，但不提交

`COMMIT # trx_id = 100`

`UPDATE hero SET name = '赵云' WHERE number = 1; # trx_id = 200`

`UPDATE hero SET name = '诸葛亮' WHERE number = 1; # trx_id = 200`

![](https://cdn.nlark.com/yuque/0/2021/webp/1732113/1636268533505-21fd8cdd-08d7-4571-8548-0b64eaec650d.webp)

    - 再次在刚刚的查询事务进行查询
        * 执行SELECT时生成一个Read View
            + `m_ids`：`[200]` 此时事务100已经提交
            + `min_trx_id`：200
            + `max_trx_id`：201
            + `creator_trx_id`：0
        * 第一个版本`trx_id`为200，在`m_ids`内，不符合要求，根据`roll_pointer`跳到下一条
        * 第二个版本`trx_id`为200，依然不符合要求，根据`roll_pointer`跳到下一条
        * 第三个版本`trx_id`为100，小于`min_trx_id`，符合要求，故返回给用户的记录`name`列的值为100
+ **REPEATABLE READ——第一次读取数据生成Read View**
    - 因为不管什么时候查询都是用同一个Read View，所以读到的记录肯定是一样的，这就是可重复读的含义，具体流程略

### 小结
MVCC（Multi-Version Concurrency Control）多版本并发控制，指的是使用`READ COMMITTED`和`REPEATABLE READ`两种隔离级别的事务执行普通SELECT操作访问记录的版本链的过程。这样子不同事务的`读-写`、`写-读`可以并发执行。

+ READ COMMITTED

每次SELECT就生成一个Read View（保持Read View最新，就总能读到最新的commit过的数据）

+ REPEATABLE READ

只在第一次SELECT生成Read View（保持Read View一致，就能保证重复读取的数据一致，可重复读的含义）

### purge
+ insert undo log在事务提交后删除，不需要purge
+ update undo log会在后续用于MVCC，不能在提交时删除，提交后放入undo log链表，等待purge线程做最后删除
+ purge的主要任务是将数据库已经mark delete的数据删除，并批量回收undo pages

## 锁
### InnoDB的表锁
表锁实现简单，占用资源少，力度很粗，有时候仅需锁住几条记录，但使用表锁的话相当于为表中所有记录都加锁，影响性能。

+ 对某个表执行增删改查语句不会上表级锁
+ 执行诸如`ALTER TABLE`、`DROP TABLE`之类的DDL语句，也不会上表锁，而是使用server层的元数据锁
+ S锁 X锁（崩溃恢复可以用）
+ IS锁、IX锁
    - 加行锁之前加的锁，作用是为了后续加表级锁的时候判断是否有已经加了S锁和X锁的记录
+ `AUTO-INC`锁——服务于`AUTO INCREMENT`属性

持有`AUTO-INC`锁时，其他事务的插入语句都要阻塞。

    - 在执行插入语句时加一个表级别的`AUTO-INC`锁，为每条待插入记录的修饰了`AUTO INCREMENT`的列分配递增值，该语句之行结束后立刻释放`AUTO-INC`锁（而不是等到事务结束，因此无法保证AUTO INCREMENT一定是连续的，如果中途有事务rollback就不连续）
    - 采用一个轻量级的锁，在为插入语句生成`AUTO_INCREMENT`修饰列的值时获取这个锁，生成后立刻释放轻量级锁，不等待整个插入语句执行完才释放锁

如果插入语句执行前就知道具体插入多少记录，一般采用轻量级锁，这种方式可以避免锁定表，提升插入的性能。

### InnoDB的行级锁
```sql
CREATE TABLE hero (
    number INT,
    name VARCHAR(100),
    country varchar(100),
    PRIMARY KEY (number)
) Engine=InnoDB CHARSET=utf8;

INSERT INTO hero VALUES
    (1, 'l刘备', '蜀'),
    (3, 'z诸葛亮', '蜀'),
    (8, 'c曹操', '魏'),
    (15, 'x荀彧', '魏'),
    (20, 's孙权', '吴');
    
mysql> SELECT * FROM hero;
+--------+------------+---------+
| number | name       | country |
+--------+------------+---------+
|      1 | l刘备      | 蜀      |
|      3 | z诸葛亮    | 蜀      |
|      8 | c曹操      | 魏      |
|     15 | x荀彧      | 魏      |
|     20 | s孙权      | 吴      |
+--------+------------+---------+
5 rows in set (0.01 sec)
```

![](https://cdn.nlark.com/yuque/0/2021/webp/1732113/1636377760959-bcd2924e-87dc-4ea1-9070-4d7ffa7f67be.webp)

#### 行锁类型
+ `Record Locks`(`LOCK_REC_NOT_GAP`)
    - 分S锁和X锁，一个事务获取了S锁，其他事务可以继续获取S锁，但不能获取X锁；一个事务获取X锁，其他事务不可以获取S锁和X锁

![](https://cdn.nlark.com/yuque/0/2021/webp/1732113/1636378288811-cdf3fbf0-fb54-4c6e-9f00-08358bcaaa3f.webp)

+ `Gap Locks`(`LOCK_GAP`)——用于解决幻读问题

![](https://cdn.nlark.com/yuque/0/2021/webp/1732113/1636378325272-087cf9b2-5ccf-4412-a607-a1a8be52e532.webp)

假如现在另一个事务想插入一条`number`值为4的记录，由于该记录的下一条记录上有一个`gap`锁，因此会阻塞插入操作，直到这个持有`gap`锁的事务提交后才插入。

`gap`锁不会限制其他事务对这条记录加`record`锁或继续加`gap`锁，作用仅仅是防止插入幻影记录。

    - 如果插入的记录`number`值大于20，则在页中的最大记录(`Suprenum`记录)加`gap`锁，这样阻塞其他事务插入（20，+~）的记录

![](https://cdn.nlark.com/yuque/0/2021/webp/1732113/1636378528374-ef0ad8cc-58c2-4326-bc8a-b02975a16a36.webp)

+ `Next-Key Locks`(`LOCK_ORDINARY`)

`Record Locks`和`Gap Locks`的合体，即锁住记录又锁住间隙。

![](https://cdn.nlark.com/yuque/0/2021/webp/1732113/1636378617825-ce51802f-b134-4643-a80a-e0ebd84fceef.webp)

+ `Insert Intention Locks`(`LOCK_INSERT_INTENTION`)

刚刚提到如果在一个加了`Gap`锁的记录前插入记录，需要阻塞到持有`Gap`锁的事务结束。这时候这个想要插入记录的事务需要加一个锁结构表明有事务想要在这个间隙插入新记录，但是现在在等待。

这个锁就是插入意向锁。

![](https://cdn.nlark.com/yuque/0/2021/webp/1732113/1636378994057-4bcbdc02-4eae-4516-aff9-e75f5284ebbe.webp)

    - T1在number值为8的记录加了个`gap`锁，然后T2和T3想向hero表插入number值为4和5的两条记录，所加的意向锁如上图所示
    - 当T1提交后会把他所获取到的锁都释放掉，此时T2和T3的`is_waiting`变为`false`
    - T2和T3之间不会相互阻塞，他们获取到插入意向锁（即`is_waiting`变为`false`）之后就可以同时执行插入操作了，插入意向锁也不会阻止其他事务继续获取该记录上任何类型的锁
+ 隐式锁

一个普通事务插入一条记录（此时没有与该记录关联的锁结构），然后另一个事务：

    - 立即使用`SELECT ... LOCK IN SHARE MODE` / `SELECT ... FOR UPDATE`，即想获取S锁或X锁

如果允许这种情况发生，则可能出现脏读的问题

    - 立即修改这条记录，也就是获取该记录的X锁

如果允许这种情况发生，则可能出现脏写的问题

这时候需要通过聚簇索引记录的`trx_id`来判断。如果该记录的`trx_id`代表的事务是活跃的，那么另一个事务会帮当前事务（`trx_id`代表的事务）创建一个X锁，`is_waiting`为`false`，然后再自己创建一个X锁，`is_waiting`为`true`，然后进入等待。

对于二级索引记录来说，本身没有`trx_id`，但在`Page Header`有一个`PAGE_MAX_TRX_ID`，如果这个值小于当前最小的活跃事务ID，则说明对该页面修改的事务都已经提交了，否则就需要在页面中定位到二级索引记录然后回表，重复上面的操作。

+ 总结隐式锁

一个事务对新插入的记录可以不显式的加锁，如果别的事务这时想对这条新插入的记录加S锁或X锁，就必须先帮当前事务生成一个锁结构，然后给自己生成一个锁结构，再进入等待状态。

    - 如何知道这条记录是不是新插入的还没提交的记录：通过事务ID判断

#### 锁结构
![](https://cdn.nlark.com/yuque/0/2021/webp/1732113/1636380249709-1ae9d95b-12f2-4cd6-80c6-af2ed1beb50f.webp)

+ 锁所在的事务信息：一个指针，指向的内存存放关于事务的更多信息
+ 索引信息：一个指针，指向的内存存放索引信息；对于行锁来说，需要记录加锁的记录属于哪个索引
+ 表锁 / 行锁信息
    - 表锁：记录对哪个表加锁等
    - 行锁：记录所在表空间，所在页号，还有`n_bits`
        * `n_bits`：对于行锁来说，一条记录对应锁结构末尾的一个比特位，这个属性代表使用了多少比特位
+ `type_mode`：32位，拆成`lock_mode`、`lock_type`、`rec_lock_type`（还有`is_waiting`属性）

![](https://cdn.nlark.com/yuque/0/2021/webp/1732113/1636381076596-b5787550-dbe5-4530-99f6-ac63359c61c3.webp)

    - `lock_mode`
        * `LOCK_IS`：共享意向锁，`IS`锁（表级锁）
        * `LOCK_IX`：独占意向锁，`IX`锁（表级锁）
        * `LOCK_S`：共享锁，`S`锁（表级锁或行级锁）
        * `LOCK_X`：独占锁，`X`锁（表级锁或行级锁）
        * `LOCK_AUTO_INC`：`AUTO-INC`锁（表级锁）
    - `lock_type`
        * 第5个标志位为1表示表锁
        * 第6个标志位为1表示行锁
        * 第7、8个标志位还没用
    - `rec_lock_type`——只有行锁才细分
        * `LOCK_ORDINARY`：记录锁和间隙锁的合体
        * `LOCK_GAP`：间隙锁
        * `LOCK_REC_NOT_GAP`：记录锁
        * `LOCK_INSERT_INTENTION`：插入意向锁
        * 其它类型
    - 当第9位为1时，`is_waiting`为true，表示当前事务尚未获取锁

为0时`is_waiting`为false，表示当前事务获取锁成功

+ 一堆比特位

数量由`n_bits`表示，页中的每条记录的`heap_no`的值能映射到锁结构末尾的比特位中的某一个比特位

![](https://cdn.nlark.com/yuque/0/2021/webp/1732113/1636381404671-772b4b01-0504-4112-93a8-fd910013a4f6.webp)

#### 锁结构复用的条件
+ 同一个事务
+ 被加锁的记录在同一个页面
+ 加锁的类型一样
+ 等待的状态(`is_waiting`)一样

#### 例子
+ 事务1给number为15的记录加S记录锁，首先需要先加IS表锁，然后加S记录锁
+ 事务2想给number为3、8、15的记录加X`next-key`锁，首先需要加IX表锁
+ 然后先给number值为3、8的记录加X锁（因为number值为15的记录已经有锁，等下要加的锁的等待状态不一样，所以不能共用一个锁结构）
+ 然后再给number值为15的记录加X锁，这个锁结构的`is_waiting`为true
+ 然后事务2进入等待状态

这里事务2的加锁顺序有讲究，应该先给number为3和8的记录加锁，否则一开始就给15加锁的话加完就进入等待状态了。

## 语句加锁分析


