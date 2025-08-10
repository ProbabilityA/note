+ 示例表 single_table

```sql
CREATE TABLE single_table (
    id INT NOT NULL AUTO_INCREMENT,
    key1 VARCHAR(100),
    key2 INT,
    key3 VARCHAR(100),
    key_part1 VARCHAR(100),
    key_part2 VARCHAR(100),
    key_part3 VARCHAR(100),
    common_field VARCHAR(100),
    PRIMARY KEY (id),
    KEY idx_key1 (key1),
    UNIQUE KEY idx_key2 (key2),
    KEY idx_key3 (key3),
    KEY idx_key_part(key_part1, key_part2, key_part3)
) Engine=InnoDB CHARSET=utf8;
```

    - 1个聚簇索引
        * id列
    - 4个二级索引
        * key1列
        * key2列，且唯一
        * key3列
        * key_part1列、key_part2列、key_part3列

## 单表访问方法
### 访问方法的概念
+ 全表扫描
+ 使用索引进行查询
    - 针对主键或唯一二级索引的等值查询
    - 针对普通二级索引的等值查询
    - 针对索引列的范围查询
    - 直接扫描整个索引

### const
+ 通过主键或唯一二级索引查询记录
    - `SELECT * FROM single_table WHERE id = 1438;`
    - `SELECT * FROM single_table WHERE key2 = 3841;`

![](https://cdn.nlark.com/yuque/0/2021/png/1732113/1635149607867-12295c99-ffb8-4282-8856-bd6bb647451d.png)

通过主键或唯一二级索引定位记录的访问方法定义为：const，意思是常数级别的

    - 主键或唯一二级索引（可以联合索引）
    - **等值比较（不包括查询NULL值的情况，因为唯一二级索引列并不限制NULL值的数量）**

`SELECT * FROM single_table WHERE key2 IS NULL`不使用此方法

### ref
+ 普通二级索引列与常数进行等值比较，有可能找到多条对应的记录

`SELECT * FROM single_table WHERE key1 = 'abc';`

![](https://cdn.nlark.com/yuque/0/2021/png/1732113/1635150101668-662eb5c8-2e1b-4155-abf8-850ad1db9585.png)

    - 记录数较少时，效率很高
+ 二级索引列值为NULL的情况
    - 不论是普通的二级索引还是唯一二级索引，它们的索引列对含NULL值的数量并不限制，所以采用的是ref方法
+ 包含多个索引列的二级索引
    - 等值比较时采用ref方法

```sql
SELECT * FROM single_table WHERE key_part1 = 'god like';

SELECT * FROM single_table WHERE key_part1 = 'god like' AND key_part2 = 'legendary';

SELECT * FROM single_table WHERE key_part1 = 'god like' AND key_part2 = 'legendary' AND key_part3 = 'penta kill';
```

    - 不全是等值比较**不采用ref方法**

`SELECT * FROM single_table WHERE key_part1 = 'god like' AND key_part2 > 'legendary';`

### ref_or_null
`... WHERE key1 = 'abc' OR key1 IS NULL;`

![](https://cdn.nlark.com/yuque/0/2021/png/1732113/1635152093059-3c8eba78-0ad1-4350-bdc7-37c0e91ac544.png)

### range
范围查询，采用二级索引+回表的方式

`SELECT * FROM single_table WHERE key2 IN (1438, 6328) OR (key2 >= 38 AND key2 <= 79);`

![](https://cdn.nlark.com/yuque/0/2021/png/1732113/1635250093012-4f4dc1e8-d02f-4c3a-b179-5fb3d07435e4.png)

### index
遍历二级索引

`SELECT key_part1, key_part2, key_part3 FROM single_table WHERE key_part2 = 'abc';`

+ 满足条件
    - 查询列表只有3个列，而索引`idx_key_part`又包含这3个列
    - 查询的条件`key_part2`列包含在索引`idx_key_part`中

因此可以直接遍历索引`idx_key_part`的叶子节点的记录来比较条件是否成立，将匹配的结果加到结果集。

### eq_ref
见下文**连接的原理 - 使用索引加快连接速度**

### all
全表扫描，即遍历聚簇索

### 分析使用哪个二级索引
+ `SELECT * FROM single_table WHERE key1 = 'abc' AND key2 > 1000;`

两个条件都可以用上索引，但是等值查找比范围查询需要扫描的行数较少（ref比range好，一般情况下），因此查询步骤为：

    - 从索引idx_key1根据条件`key1 = ‘abc’`查找到对应的二级索引记录
    - 回表，查出完整记录
    - 根据条件`key2 > 1000`继续过滤记录
+ 明确range访问方法使用的范围区间
    - LIKE 只有匹配完整字符串和字符串前缀时才可以使用索引，具体原因见前面
    - IN 和若干个等值匹配之间用OR连接一样，也就是说会产生多个单点区间
+ 所有搜索条件都可以使用某个索引
    - `SELECT * FROM single_table WHERE key2 > 100 AND key2 > 200;`
    - `SELECT * FROM single_table WHERE key2 > 100 OR key2 > 200;`
+ 有的搜索条件无法使用索引
    - `SELECT * FROM single_table WHERE key2 > 100 AND common_field = 'abc';`

先使用索引`idx_key2`查出id回表，然后再根据commonfield的条件过滤

+ 复杂搜索条件下找出范围匹配的区间

```sql
SELECT * FROM single_table WHERE 
        (key1 > 'xyz' AND key2 = 748 ) OR
        (key1 < 'abc' AND key1 > 'lmn') OR
        (key1 LIKE '%suf' AND key1 > 'zzz' AND (key2 < 8000 OR common_field = 'abc')) ;
```

    - 先看where条件涉及哪些列：key1、key2、common_field，有索引`idx_key1`、`idx_key2`
    - 分析
        * 假设使用`idx_key1`，把用不到该索引的条件变为TRUE

```sql
(key1 > 'xyz' AND TRUE) OR 
(key1 < 'abc' AND key1 > 'lmn') OR 
(TRUE AND key1 > 'zzz' AND (TRUE OR TRUE))

=>

(key1 > 'xyz') OR
(key1 < 'abc' AND key1 > 'lmn') OR
(key1 > 'zzz')

=>

(key1 > 'xyz') OR (key1 > 'zzz')

=> 

(key1 > 'xyz')
```

        * 也就是说，用`idx_key1`进行查询的话，要先把满足`key1 > 'xyz'`的二级索引记录都取出来然后再回表，得到完整记录之后根据条件进行过滤
        * 假设使用`idx_key2`，把用不到该索引的条件变为TRUE

```sql
(TRUE AND key2 = 748) OR 
(TRUE AND TRUE) OR 
(TRUE AND TRUE AND (key2 < 8000 OR TRUE))

=> 

(key2 = 748) OR
TRUE OR
TRUE

=> 

TRUE
```

        * 也就是说用`idx_key2`查询的话要扫描所有记录再回表，这种情况下得不偿失，是不会使用这个索引的

### index merge - 索引合并
使用多个索引完成一次查询的执行方法称为`index merge`

#### Intersection合并 - 交集合并
从多个二级索引中查询到的结果取交集

`SELECT * FROM single_table WHERE key1 = 'a' AND key3 = 'b';`

+ 过程：
    - 从`idx_key1`对应的B+树中取出`key1 = 'a'`的相关记录
    - 从`idx_key3`对应的B+树中取出`key3 = 'b'`的相关记录
    - 计算两个结果集中`id`值的交集（二级索引的叶子节点存的是索引列和主键列）
    - 回表，取出完整记录
+ 另一种查询方式：根据某个搜索条件读取一个二级索引，然后马上回表，再过滤另一个搜索条件
+ 比较两种查询方式的成本
    - 只读取一个二级索引的成本
        * 按照某个搜索条件读取一个二级索引
        * 根据从该二级索引得到的主键值进行回表
        * 将回表结果根据另外一个查询条件进行过滤
    - 读取多个二级索引然后取交集的成本
        * 按照不同的搜索条件读取多个二级索引
        * 主键值取交集
        * 回表

读取多个二级索引虽然消耗性能，但是读取二级索引的操作是顺序I/O，而回表操作是随机I/O，如果只读取一个二级索引时需要回表的记录特别多，而读取多个二级索引之后取交集的记录数非常少，此时回表的代价大于读取多个二级索引回表的代价。

+ **使用Intersection索引合并的情况**
    - 二级索引列是等值匹配的情况，对于联合索引来说，每个列都必须等值匹配，不能出现只匹配部分列的情况

`SELECT * FROM single_table WHERE key1 = 'a' AND key_part1 = 'a' AND key_part2 = 'b' AND key_part3 = 'c';`

上面的语句使用了`idx_key1`和`idx_key_part`两个二级索引进行索引合并

而下面两个查询不能进行Intersection合并

`SELECT * FROM single_table WHERE key1 > 'a' AND key_part1 = 'a' AND key_part2 = 'b' AND key_part3 = 'c';`

`SELECT * FROM single_table WHERE key1 = 'a' AND key_part1 = 'a';`

第一个对key1进行了范围匹配，第二个查询是因为只匹配了部分列，即key_part2和key_part3没有出现在搜索条件中

    - **主键列可以是范围匹配**

`SELECT * FROM single_table WHERE id > 100 AND key1 = 'a';`

+ **为什么只有这两种情况**
    - 多个二级索引等值匹配：
        * 根据二级索引搜出来的主键值是已经排序了的（索引列等值的情况下，会根据主键值再排序），这时取交集的时间复杂度为`O(n)`（如果多个二级索引范围匹配的话，还得对主键值进行排序，再取交集，更复杂）
    - 二级索引等值匹配 + 主键列范围匹配
        * 根据二级索引搜出来的主键值是排序了的，主键列的范围条件只不过是作为一个过滤条件，并不像上面一样另一个列也得查出来
+ **满足条件一定会使用索引合并（Intersection合并）嘛？**

满足情况一和情况二只是发生Intersection索引合并的**必要条件**，不是充分条件。还得根据优化器的判断，优化器如果判断根据某个二级索引获取的记录太多，回表开销太大，而根据Intersection索引合并后需要回表的记录数大大减少时才会使用Intersection索引合并。

#### Union合并 - 并集合并
Intersection合并是取主键值的并集，Union合并是取主键值的交集，对应的其实是OR查询。

+ **使用Union合并的情况**
    - 二级索引是等值匹配的情况，对于联合索引来说，每个列都必须等值匹配，不能出现只匹配部分列的情况
    - 主键列可以是范围匹配
    - **使用Intersection索引合并的搜索条件**

`SELECT * FROM single_table WHERE key_part1 = 'a' AND key_part2 = 'b' AND key_part3 = 'c' OR (key1 = 'a' AND key3 = 'b');`

        * 先按照条件`key1 = 'a' AND key3 = 'b'`从索引`idx_key1`和`idx_key3`中使用Intersection索引合并的方式得到一个主键集合
        * 再按照条件keypart从联合索引`idx_key_part`中得到另一个主键集合
        * 采用Union索引合并把上述两个主键集合取并集，然后进行回表操作
+ **满足条件一定会使用索引合并（Union合并）嘛？**

优化器单独根据搜索条件从某个二级索引中获取的记录数较少，通过Union索引合并后访问的代价比全表扫描更小时才会使用Union索引合并

#### Sort-Union合并
允许二级索引范围查询

`SELECT * FROM single_table WHERE key1 < 'a' OR key3 > 'z'`

+ 先根据`key1 < 'a'`条件从`idx_key1`二级索引中获取记录，并按照记录的主键值进行排序
+ 再根据`key3 > 'z'`条件从`idx_key3`二级索引中获取记录，并按照记录的主键值进行排序
+ 因为上述的两个二级索引主键值都是排好序的，剩下的操作和Union索引合并方式就一样了

Sort-Union索引合并比单纯的Union索引合并多了一步对二级索引记录的主键值排序的过程

#### 为什么没有Sort-Intersection合并
Sort-Union适用场景是单独根据搜索条件从二级索引中获取的记录数比较少，这样根据主键值排序的成本也不会太高。

Intersection索引合并的场景是单独根据搜索条件从某个二级索引中获取的记录数太多，导致回表开销太大，合并后可以明显降低回表开销，但是如果加入Sort-Intersection后，就需要为大量的二级索引记录按照主键值进行排序，这个成本可能比回表查询还高。

#### 索引合并注意事项
+ 联合索引替代Intersection索引合并

`SELECT * FROM single_table WHERE key1 = 'a' AND key3 = 'b';`

这个查询之所以有可能使用Intersection索引合并，是因为`idx_key1`和`idx_key3`是两个单独的索引，如果为这两个列建立联合索引，就只需要查询一颗B+树再回表，不用索引合并（索引合并得查询两颗B+树再回表）。



## 连接 Join
+ 笛卡尔积

`SELECT * FROM t1, t2`

假设t1表有3条记录，t2表有3条记录，则结果会有9条记录

+ 内连接（取交集，即取满足on子句的，没有on子句就做笛卡尔积）

`SELECT * FROM t1 [INNER | CROSS] JOIN t2 [ON 连接条件] [WHERE 过滤条件]`

    - <font style="color:rgb(51, 51, 51);">驱动表中的记录在被驱动表中找不到匹配的记录，该记录不会加入到最后的结果集</font>
    - **<font style="color:rgb(51, 51, 51);">内连接的驱动表和被驱动表的位置可以交换</font>**
    - **<font style="color:rgb(51, 51, 51);">由于内连接中on子句和where子句是等价的，所以内连接中不强制要求写明on子句</font>**
+ <font style="color:rgb(51, 51, 51);">外连接</font>

`SELECT * FROM t1 LEFT [OUTER] JOIN t2 ON 连接条件 [WHERE 过滤条件]`

    - <font style="color:rgb(51, 51, 51);">左外连接取左侧表为驱动表，有外连接取右侧表为驱动表</font>
    - **<font style="color:rgb(51, 51, 51);">驱动表和被驱动表的位置不可以交换</font>**

### 嵌套循环连接
对于两表连接来说，驱动表只会被访问一遍，被驱动表却要被访问到好多遍，具体访问几遍取决于从驱动表单表查询后到结果集中的记录条数。

![](https://cdn.nlark.com/yuque/0/2021/png/1732113/1635334038989-86223bbc-7ebb-4151-90a5-3b65c89f3973.png)

### 使用索引加快连接速度
在上图的步骤2中可能需要多次访问被驱动表，如果每次访问被驱动表的方式都是全表扫描的话，那得扫描很多次。查询t2表的过程也相当于一次单表扫描，我们可以利用索引来加快查询速度。

`SELECT * FROM t1, t2 WHERE t1.m1 > 1 AND t1.m1 = t2.m2 AND t2.n2 < 'd';`

![](https://cdn.nlark.com/yuque/0/2021/png/1732113/1635334171212-9d10f1ff-c1fb-4dc4-93cc-6427b87d2613.png)

在查询驱动表t1后的结果集有2条记录，嵌套循环连接算法需要对被驱动表查询2次：

+ 当t1.m1 = 2时，查询t2表

`SELECT * FROM t2 WHERE t2.m2 = 2 AND t2.n2 < 'd';`

+ 当t1.m1 = 3时，查询t2表

`SELECT * FROM t2 WHERE t2.m2 = 3 AND t2.n2 < 'd';`

上述两个查询语句的条件的列是m2和n2，我们可以：

+ 在m2列上建索引

可能使用到ref访问方法，回表后再判断t2.n2 < d条件是否成立

**如果m2是主键或者唯一二级索引，那么使用的是const访问方法。在连接查询中对被驱动表使用主键值或唯一二级索引列的值进行等值查找的查询方式称为eq_ref**。

+ 在n2列上建索引

可能使用到range访问方法，查询到主键后回表，再根据t2.m2 = 3条件过滤

有时候查询的列可能只涉及被驱动表的部分列，而条件无法使用索引，但是查询的列都是某个索引的一部分，这时候也有可能使用index访问方法来查询被驱动表，所以最好不要使用*作为查询列表，最好把真实用到的列作为查询列表。

### Join Buffer
![](https://cdn.nlark.com/yuque/0/2021/png/1732113/1635335464237-513b753e-017c-4290-882f-a61745b6dbb2.png)

+ 将从驱动表查询的结果加入到Join Buffer
+ 如果Join Buffer满了或者结果加入完了，就开始从扫描被驱动表，每次读取被驱动表的一条记录，将其与Join Buffer内的所有记录做匹配
+ 被驱动表扫描完成后，如果驱动表的查询结果集还没加入完，则重复第二步

#### 注意点
+ 加入Join Buffer后，减少了被驱动表的访问次数，减少I/O代价
    - 如果Join Buffer足够大，能容纳驱动表结果集的所有记录，这样就只需要访问1次被驱动表就可以完成连接操作了
+ Join Buffer默认256KB，可以适当调整（join_buffer_size）
+ **从驱动表查询出来的记录并不是所有列都会被放到join buffer中，只有查询列表中的列和过滤条件中的列才会被放到join buffer，这再次提醒我们不要把*当作查询列表，这样join buffer可以放更多记录**

## 基于成本的优化
### 什么是成本
+ I/O成本（将数据和索引从磁盘上加载到内存中损耗的时间）
    - **读取一个页的成本默认是1.0**
+ CPU成本（读取、检测记录是否满足条件，对结果集排序等）
    - **读取及检测一条记录是否满足条件的成本是0.2（不检测条件也是0.2）**

### 单表查询的成本
+ 根据搜索条件找出所有可能使用的索引
+ 计算全表扫描的成本
+ 计算使用不同索引查询的成本
+ 对比各种执行方案，找出成本最低的一个

```sql
SELECT * FROM single_table WHERE 
    key1 IN ('a', 'b', 'c') AND 
    key2 > 10 AND key2 < 1000 AND 
    key3 > key2 AND 
    key_part1 LIKE '%hello%' AND
    common_field = '123';
```

1. 找索引
    - `key1 IN ('a', 'b', 'c')` => `idx_key1`
    - `key2 > 10 AND key2 < 1000` => `idx_key2`
    - `key3 > key2`索引列没有与常数比较，不能使用索引
    - `key_part1 LIKE '%hello%'` 前缀不固定，不能使用索引
    - `common_field`没有索引
2. 计算全表扫描的代价 **查询成本 = I/O成本 + CPU成本**

全表扫描的意思是将聚簇索引中的记录都依次和给定的搜索条件做一下比较，把符合搜索条件的记录加入到结果集，所以需要将聚簇索引对应的页加载到内存中，然后再检测记录是否符合搜索条件。

    - 聚簇索引占用的页面数
    - 该表中的记录数

```sql
mysql> SHOW TABLE STATUS LIKE 'single_table'\G
*************************** 1. row ***************************
           Name: single_table
         Engine: InnoDB
        Version: 10
     Row_format: Dynamic
           Rows: 9693
 Avg_row_length: 163
    Data_length: 1589248
Max_data_length: 0
   Index_length: 2752512
      Data_free: 4194304
 Auto_increment: 10001
    Create_time: 2018-12-10 13:37:23
    Update_time: 2018-12-10 13:38:03
     Check_time: NULL
      Collation: utf8_general_ci
       Checksum: NULL
 Create_options:
        Comment:
1 row in set (0.01 sec)
```

与上述有关的主要有两个：

+ Rows

对MyISAM来说，该值是准确的，**对于使用InnoDB来说，该值是估计值**

+ Data_length

对于MyISAM来说，该值就是数据文件的大小，**对于InnoDB来说，该值就相当于聚簇索引占用的存储空间的大小。**

**Data_length = 聚簇索引页面数量 x 每个页面的大小**

**因此上表聚簇索引页面数量 = Data_length / 16 / 1024 (每个页面的大小，单位为字节) = 97**

+ I/O成本

97 x 1.0 + 1.1 = 98.1

97页，1.1为微调值。**虽然97页是整棵树的页数，包含了非叶子节点，但是全表扫描的时候是不扫描非叶子节点的，这样的计算比较简单。**

+ CPU成本

9693 x 0.2 + 1.0 = 1939.6

+ 总成本

98.1 + 1939.6 = 2037.7

3. 计算使用不同索引执行查询的代价

MySQL查询优化器先分析使用**唯一二级索引**的成本，再分析使用**普通索引**的成本。

**3.1 使用idx_key2执行查询的成本分析**

`idx_key2`对应的条件`key2 > 10 AND key2 < 1000`

    - **范围区间数量**

不论此范围占用多少页，默认按1页算，则访问这个二级索引的I/O成本是1 x 1.0 = 1

    - 需要回表的记录数
    1. 先根据key2 > 10这个条件访问`idx_key2`对应的B+树索引，找到第一条记录（区间最左记录）
    2. 再根据key2 < 1000这个条件访问`idx_key2`对应的B+树索引，找到最后一条满足此条件的记录（区间最右记录）
    3. 如果区间最左记录和区间最右记录相隔不太远（相隔不大于10个页面），就可以精确统计出满足条件的二级索引记录条数。

否则只沿着区间最左记录向右读10个页面，计算平均每个页面有多少记录，然后用这个平均值乘以区间最左记录和区间最右记录之间的页面数量（找目录项，求目录项之间的记录数就是页数；如果目录项不在同一页，就再递归）。



![](https://cdn.nlark.com/yuque/0/2021/png/1732113/1635428656403-3ab8b666-aa74-42db-8c70-283551290a31.png)

用算法测得`idx_key2`在区间（10，1000）之间有95条记录，

则CPU成本是95 x 0.2 + 0.01 = 19.01

    - 回表

每次回表操作都相当于访问一个页面，也就是说二级索引范围有多少条记录就回表多少次

95 x 1.0 = 95

    - 回表后的记录判断其他条件是否成立

95 x 0.2 = 19

+ 总成本

I/O：1 + 95 = 96（范围区间数量 + 回表）

CPU：19.01 + 19 = 38.01 （读取二级索引记录 + 回表后判断搜索条件）

总成本：134.01

**3.2 使用**`idx_key1`**执行查询的成本分析**

    - 范围区间数量（区间数 x 1） = 3
    - 需要回表的记录数 118，118 x 0.2 + 0.01 = 23.61
    - 回表后的I/O成本 118 x 1 = 118（回表一次 = 读一页 = 成本1）
    - 判断其他条件 118 x 0.2 = 23.6
    - 总成本 168.21

**3.3 是否有可能索引合并**

`idx_key1`和`idx_key2`都是范围查询，不满足Intersection索引合并的条件

4. 对比各执行方案的代价，找出成本最低的那个

使用`idx_key2`查询

### 基于索引统计数据的成本计算
eq_range_index_dive_limit默认值200（MySQL 5.7.3以前为10，如果用到了in查询但是没有用到索引可以考虑是不是这个值太小导致的）

in参数小于200个时，使用`index dive`的方式计算各单点区间对应的记录数

in参数超过200个时，使用索引统计数据来估算。

`SHOW INDEX FROM single_table`中的`Cardinality`记录了索引列中不重复值的数量（估计），Rows / Cardinality = 平均每个值的重复次数 = 10

假设IN查询有20000个参数，使用上面结果估算则每个参数大概对应10条记录，则需要回表的记录数是10 x 20000 = 200000

### 连接查询的成本
**连接查询的总成本 = 单次访问驱动表的成本 + 驱动表扇出数 x 单次访问被驱动表的成本**

+ 对于外连接，驱动表是固定的，只需要为驱动表和被驱动表选择成本最低的访问方法
+ 对于内连接，不同的表作为驱动表的查询成本可能不同，需要考虑最优的表连接顺序，然后再为驱动表和被驱动表选择成本最低的访问方法

关键点是：

+ 减少驱动表的扇出
+ 对被驱动表的访问成本尽量低

### 多表连接的成本分析
n个表连接，有n！种连接顺序

+ 提前结束某种顺序的成本评估
    - 计算得ABC的连接成本是10，在计算BCA时发现BC的连接成本已经大于10，停止分析此组合
+ 系统变量`optimizer_search_depth`，如果连接表的个数小于该值就穷举，否则只对与这个值相同数量的表进行穷举分析
+ 根据某些规则不考虑某些连接顺序（启发式规则，`optimizer_prune_level`来控制是否启用）

### 调节成本常数
```sql
mysql> SHOW TABLES FROM mysql LIKE '%cost%';
+--------------------------+
| Tables_in_mysql (%cost%) |
+--------------------------+
| engine_cost              |
| server_cost              |
+--------------------------+
2 rows in set (0.00 sec)
```

![](https://cdn.nlark.com/yuque/0/2021/png/1732113/1635430965283-78043bbd-713a-46b7-9671-2e407c0efc5d.png)

> MySQL在执行诸如DISTINCT查询、分组查询、Union查询以及某些特殊条件下的排序查询都可能在内部先创建一个临时表，使用这个临时表来辅助完成查询（比如对于DISTINCT查询可以建一个带有UNIQUE索引的临时表，直接把需要去重的记录插入到这个临时表中，插入完成之后的记录就是结果集了）。
>
> 在**数据量大**的情况下可能创建基于磁盘的临时表，也就是为该临时表使用MyISAM、InnoDB等存储引擎，在**数据量不大**时可能创建基于内存的临时表，也就是使用Memory存储引擎。
>

![](https://cdn.nlark.com/yuque/0/2021/png/1732113/1635431147281-cd3815eb-bd9b-44b0-a8e6-3959ff9f7f5e.png)

> 目前MySQL并没办法准确预测某个查询需要访问的块加载到内存中还是在磁盘上，所以这两个值一样
>

## InnoDB统计数据的搜集
InnoDB以表为单位来收集和存储统计数据，可以把某些表的统计数据存于内存，其他表的统计数据存于磁盘。

`ALTER TABLE xxx Engine=InnoDB, STATS_PERSISTENT= 1`，1表示存于磁盘，0表示存于内存

+ 永久性统计数据
+ 非永久性统计数据（存于内存中）

### 基于磁盘的永久性统计数据
+ innodb_table_stats
    - database_name
    - table_name（和database_name联合主键）
    - last_update
    - n_rows 表中记录数（估计值）

按照一定算法选取几个叶子节点页面，计算每个页面主键值记录数量，然后取平均，乘以叶子节点的数量。

n_rows精确与否取决于统计时采样的页面数量，`innodb_stats_persistent_sample_pages`变量控制统计数据时采样的页面数量（默认为20）。

InnoDB以表为单位收集和存储统计数据，可以通过指定`STATS_SAMPLE_PAGES`来之名表的统计数据采样数量。

    - clustered_index_size 聚簇索引占用页面数量（估计值）
    - sum_of_other_index_sizes 表中其他索引占用页面数量（估计值）
+ innodb_index_stats

![](https://cdn.nlark.com/yuque/0/2021/png/1732113/1635471868521-463f159c-dc08-4f37-aa2c-ff3d27ad3bc9.png)

stat_name：

    - n_leaf_pages 叶子节点多少页
    - size 索引共多少页
    - n_diff_pfxNN

![](https://cdn.nlark.com/yuque/0/2021/png/1732113/1635471938652-61aa2938-0737-477d-b680-1f99d696bba9.png)

### 定期更新统计数据
+ `innodb_stats_auto_recalc` 默认开启，发生变动的记录超过表大小的10%时异步更新
+ 手动调用`ANALYZE TABLE`更新统计信息（同步更新）

### 基于内存的非永久性统计数据
统计数据采样的页面数量由`innodb_stats_transient_sample_pages`控制，默认为8

### innodb_stats_method的使用
索引列不重复值的数量 这个统计数据很重要，主要用于：

+ 单表查询中单点区间太多，直接采用INDEX DIVE访问B+树索引去统计每个单点区间的记录数量太耗费性能，则依赖这个值来计算单点区间对应的记录数量
+ 连接查询中涉及两个表等值匹配的条件，但是真正对t2查询前t1.column的值不确定，没有办法通过index dive访问b+树的方法统计记录数量，只能依赖这个值来计算

在统计 索引列不重复值的数量 时，出现NULL值怎么办？**通过innodb_stats_method控制**

```sql
+------+
| col  |
+------+
|    1 |
|    2 |
| NULL |
| NULL |
+------+
```

+ 认为所有NULL值相等（默认值） => 3
+ 认为所有NULL值不相等  => 4
+ 忽略NULL值  => 2

## 基于规则的优化
### 条件化简
+ 移除不必要的括号
+ 常量传递 `a = 5 AND b > a` => `a = 5 AND b > 5`
+ 等值传递 `a = b AND b = c AND c = 5` => `a = 5 AND b = 5 AND c = 5`
+ 移除没用的条件
+ 表达式计算 `a = 5 + 1` => `a = 6`
+ HAVING子句和WHERE子句的合并

没有出现SUM、MAX等聚集函数以及GROUP BY子句，优化器就会合并

+ 常量表检测
    - 表中记录数为0或1（只针对Memory和MyISAM，因为InnoDB没办法精确知道表中记录数）
    - 主键等值匹配或唯一二级索引等值匹配

```sql
SELECT * FROM table1 INNER JOIN table2
    ON table1.column1 = table2.column2 
    WHERE table1.primary_key = 1;
```

会先执行对table1表的查询，然后把涉及table1的条件都替换掉，变成

```sql
SELECT table1表记录的各个字段的常量值, table2.* FROM table1 INNER JOIN table2 
    ON table1表column1列的常量值 = table2.column2;
```

+ **空值拒绝**（外连接 -> 内连接）

`SELECT * FROM t1 LEFT JOIN t2 ON t1.m1 = t2.m2 WHERE t2.m2 = 2`

此时t2.m2肯定不为NULL，那么外连接中在被驱动表中找不到符合ON子句条件的驱动表记录也就被排除了，此时外连接可以转换为内连接（转换为内连接的好处是可以优化驱动表和被驱动表的位置）

## 子查询优化
### 子查询语法
按返回结果区分

+ 标量子查询（返回一个单一值）

`SELECT (SELECT m1 FROM t1 LIMIT 1);`

`SELECT * FROM t1 WHERE m1 = (SELECT MIN(m2) FROM t2);`

+ 行子查询

`SELECT * FROM t1 WHERE (m1, n1) = (SELECT m2, n2 FROM t2 LIMIT 1);`

+ 列子查询

`SELECT * FROM t1 WHERE m1 IN (SELECT m2 FROM t2);`

+ 表子查询

`SELECT * FROM t1 WHERE (m1, n1) IN (SELECT m2, n2 FROM t2);`

按与外层查询关系区分

+ 不相关子查询
+ 相关子查询

`SELECT * FROM t1 WHERE m1 IN (SELECT m2 FROM t2 WHERE n1 = n2);`

### 子查询在布尔表达式中
+ 使用`= > < >= <= <> != <=>`作为布尔表达式的操作符
    - `SELECT * FROM t1 WHERE m1 < (SELECT MIN(m2) FROM t2);`
+ `[NOT] IN / ANY / SOME / ALL`子查询
    - `IN / NOT IN`

用来判断某个操作数在不在由子查询结果集组成的集合中

`SELECT * FROM t1 WHERE (m1, n1) IN (SELECT m2, n2 FROM t2);`

    - `ANY / SOME / NOT ANY / NOT SOME`（ANY和SOME是同义词）

只要子查询结果集中存在某个值和给定的操作数做比较结果为TRUE，那么整个表达式的结果就为TRUE

`SELECT * FROM t1 WHERE m1 > ANY(SELECT m2 FROM t2);`

如果在t2中的m2列存在一个比m1小的值，那么为TRUE，等价于

`SELECT * FROM t1 WHERE m1 > (SELECT MIN(m2) FROM t2);`

**另外，=ANY与IN等价**

    - `ALL`

子查询结果集中所有值和给定的操作数比较都为TRUE时才为TRUE

`SELECT * FROM t1 WHERE m1 > ALL(SELECT m2 FROM t2);`

等价于

`SELECT * FROM t1 WHERE m1 > (SELECT MAX(m2) FROM t2);`

    - `EXISTS / NOT EXISTS`

有时候只关心子查询的结果集是否有记录，不关心记录具体是啥

`SELECT * FROM t1 WHERE EXISTS (SELECT 1 FROM t2);`

只要`(SELECT 1 FROM t2)`这个查询中有记录，那么`EXISTS (SELECT 1 FROM t2)`的值就是TRUE

### 子查询语法注意事项
+ 必须用小括号扩起来
+ 在SELECT子句中的子查询必须是标量子查询

错误示范：`SELECT (SELECT m1, n1 FROM t1)`

+ 想得到标量子查询或行子查询，又不能保证子查询结果集只有一条记录时应用`LIMIT 1`限制
+ 对`[NOT] IN / ANY / SOME / ALL`子查询，不允许有LIMIT语句
    - 目前不支持
+ 不允许在一条语句增删改某个表的记录时同时还对该表进行子查询

`DELETE FROM t1 WHERE m1 < (SELECT MAX(m1) FROM t1);`

### 执行方式
#### 标量子查询、行子查询
+ 不相关的子查询

先单独执行子查询，再将结果当作外层参数执行外层查询

+ 相关的子查询
    1. 先从外层查询中获取一条记录
    2. 从那条记录中获取子查询中涉及到的值，然后执行子查询
    3. 根据子查询结果判断外层查询WHERE子句的条件是否成立，如果成立就加入结果集
    4. 再次执行第一步，直到外层查询结束

#### IN子查询
1. 先将子查询的结果集写入一个临时表里：
+ 该临时表的列就是子查询结果集中的列
+ 写入临时表的记录会被去重（IN查询的参数重复不重复不影响结果）（去重的方法是建立主键或者唯一索引）
+ 结果集不大时建立基于内存的使用`Memory`存储引擎的临时表，并且建立**哈希索引**

结果集较大时（超过了系统变量`tmp_table_size`或者`max_heap_table_size`）建立基于磁盘的存储引擎，索引类型使用**B+树索引**

将子查询结果集中的记录保存到临时表的过程称为**物化**。

2. 物化表转连接

假设物化表的名称为`materialized_table`，存储的子查询结果集的列为`m_val`

**2.1 从表s1的角度来看待**

对于s1表中的每条记录，如果该记录的key1列的值在子查询对应的物化表，则该记录会被加入到最终的结果集。

**2.2 从表s2的角度来看待**

对于物化表中的每条记录，如果该记录的`m_val`列的值在s1表中能找到对应的key1列的值相等的记录，那么把**这些**记录加入到结果集。

也就是说，相当于转换为内连接：

`SELECT s1.* FROM s1 INNER JOIN materialized_table ON key1 = m_val;`

转换为内连接后再分析两个表分别作为驱动表的成本，选择成本更低的方案执行查询。

#### 将子查询转换为semi-join
不进行物化操作，直接把子查询转换为半连接

```sql
SELECT * FROM s1 
WHERE key1 IN (SELECT common_field FROM s2 WHERE key3 = 'a');

=>（不完全等价，只关心是否存在记录满足s1.key1 = s2.common_field，不关心具体多少条记录匹配）

SELECT s1.* FROM s1 INNER JOIN s2 
	ON s1.key1 = s2.common_field 
  WHERE s2.key3 = 'a';
```

    - Table pullout（子查询中的表上拉）

当子查询的查询列表处只有主键或唯一索引列时，可以直接把子查询中的表上拉到外层查询的FROM子句中。（因为主键或唯一索引列的数据本身就不重复，所以对于同一条s1表中的记录不可能找到两条以上的符合`s1.key2 = s2.key2`的记录）

    - Duplicate Weedout execution strategy（重复值消除）

转换为半连接查询后，可能s1表中的某条记录匹配s2表中的多条记录，所以该条记录可能多次被添加到最后的结果集中，为了消除重复，可以建立一个临时表。

```sql
CREATE TABLE tmp {
	  id PRIMARY KEY
};
```

当某条s1表中的记录要加入结果集时，就先把这条记录的id值加入到临时表，如果添加成功，就说明之前这条s1表中的记录并没有加入到最终的结果集；如果添加失败，就说明已经加入过了，丢弃就好。这种使用临时表消除semi-join结果集中重复值的方式称之为`DuplicateWeedout`。

    - LooseScan execution strategy（松散扫描）

`SELECT * FROM s1 WHERE key3 IN (SELECT key1 FROM s2 WHERE key1 > 'a' AND key1 < 'b');`

![](https://cdn.nlark.com/yuque/0/2021/png/1732113/1635511812047-4729a8f3-9d94-45a4-bd20-de89f02de6a3.png)

对于s2表的访问可以使用key1列的索引，而子查询的列表处是key1列。直接扫描s2表的`idx_key1`索引，对于值相同的二级索引记录，只取第一条到s1表中找匹配的记录。

    - Semi-join Materialization execution strategy

先把外层查询的IN子句中的不相关查询进行物化，再进行外层查询的表和物化表的连接，本质也是一种semi-join，只不过由于物化表没有重复记录，所以可以将子查询直接转换为连接查询

    - First Match execution strategy

先去一条外层查询中的记录，然后到子查询的表中寻找符合匹配条件的记录，如果能找到一条，就将该外层查询的记录放入最终的结果集并停止查找更多匹配的记录，如果找不到则丢弃；然后继续取下一条外层查询中的记录。

#### semi-join的适用条件 / 不适用条件
+ 适用条件
    - IN语句和布尔表达式组成，并且在外层查询的WHERE或者ON子句出现
    - 外层查询的其他搜索条件必须与IN子查询使用AND连接
    - 子查询必须是一个单一的查询，不能是由若干查询由UNION连接起来的
    - 子查询不能包含GROUP BY或者HAVING语句或者聚集函数
+ 不适用条件
    - 其他条件与IN子查询用OR连接

`WHERE key1 IN (SELECT xxx FROM xxx) OR key2 > 100`

    - 使用了`NOT IN`

`SELECT * FROM s1 WHERE key1 NOT IN (SELECT common_field FROM s2 WHERE key3 = 'a');`

    - 在SELECT子句中的IN子查询

`SELECT key1 IN (SELECT common_field FROM s2 WHERE key3 = 'a') FROM s1;`

    - 子查询包含GROUP BY、HAVING、聚集函数

`SELECT * FROM s1 WHERE key2 IN (SELECT COUNT(*) FROM s2 GROUP BY key1);`

    - 子查询包含UNION

#### 不能优化为semi-join的子查询
+ 先物化再查询
+ 尝试把IN子查询转化为EXISTS子查询

`outer_expr IN (SELECT inner_expr FROM ... WHERE subquery_where)`

转化为

`EXISTS (SELECT inner_expr FROM ... WHERE subquery_where AND outer_expr = inner_expr)`

## Explain & Optimizer trace（略）
