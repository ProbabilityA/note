[https://draveness.me/golang/docs](https://draveness.me/golang/docs)

[https://golang.design/go-questions/](https://golang.design/go-questions/)

[https://juejin.cn/post/7130082747715944461](https://juejin.cn/post/7130082747715944461)

# 数据结构
## 数组
+ 数组类型由元素类型和数组大小共同决定，两者都相同才是同一类型
+ 数组**初始化**时，不考虑逃逸分析的情况
    - 数组**初始**元素数量小于或等于 4 个，会在栈上直接初始化变量
    - 大于 4 个，会在静态存储区初始化，在运行时拷贝到栈上
+ 在中间代码生成期间，编译器会插入运行时方法 runtime.panicIndex 调用防止发生越界错误

## 切片
+ 切片可以动态地扩容
+ 切片底层的数组可以被多个 SliceHeader 同时指向，因此对一个 slice 进行操作有可能影响到其他 slice

```go
type SliceHeader struct {
    // 指向数组的指针，指向一片连续的内存空间，内存空间 = 元素大小 x Cap
	Data uintptr
    // 数据长度
	Len  int
    // 数组大小
	Cap  int
}
```

### append
+ 解构切片获得 Data、Len 和 Cap，如果赋值回原变量则直接在原切片头上修改，如果赋值给新切片则构造新的切片头
+ 向 nil 和 empty 的 slice 进行 append
    - 都会进行底层数组的扩容，申请一块新的内存

### 扩容
+ Go 1.18 以前（Go 1.17 还会进行内存对齐）
    1. 如果期望容量大于当前容量的两倍就会使用期望容量
    2. 如果当前切片的长度小于 1024 就会将容量翻倍
    3. 如果当前切片的长度大于 1024 就会每次增加 25% 的容量，直到新容量大于期望容量
+ Go 1.18（threshold 为 256）
    1. 如果期望容量大于当前容量的 2 倍就会使用期望容量
    2. 如果当前切片的长度小于 threshold 就会将容量翻倍
    3. 如果当前切片的长度大于 threshold 就会每次增加（旧容量 + 3 * threshold）/ 4，直到新容量大于期望容量
    - Go1.18 的扩容策略更加平滑，随着容量的增大，其扩容系数是越来越小的，可以更好地节省内存
    - 求极限，当期望容量远大于 256 时，扩容系数约为 1.25，跟 Go 1.18 以前一样

### 切片作为函数参数
+ 传切片
    - 修改值，原切片的值会被修改
    - append，原切片不会有变化（append 会修改 len 和 cap，但修改的是函数参数的 slice）
+ 传切片指针
    - 修改值，原切片的值会被修改
    - append，原切片会跟着修改

## 哈希表
```go
type hmap struct {
    count     int             // map中k-v对数
    flags     uint8           // map状态字段
    B         uint8           // 桶的个数为2^B
    noverflow uint16          // 溢出桶的个数
    hash0     uint32          // 计算key的hash值时作为hash种子使用
    buckets    unsafe.Pointer // 指向bucket首地址的指针
    oldbuckets unsafe.Pointer // map扩容后, 该字段指向扩容前buckets内存首地址
    nevacuate  uintptr        // 迁移进度
    extra *mapextra           // 可选数据
}

type mapextra struct {
    overflow    *[]*bmap  // overflow地址数组的指针
    oldoverflow *[]*bmap  // 扩容时原overflow地址数组的指针
    nextOverflow *bmap    // 下一个空闲overflow的地址
}

type bmap struct {
  topbits  [8]uint8 // 存储hash值的高8位，除了存储hash值的高8位，也可以存储一些状态码
  keys     [8]keytype // key数组，隐藏字段
  values   [8]valuetype // value数组，隐藏字段
  overflow uintptr // 溢出buceket指针，隐藏字段
}
```

![](https://cdn.nlark.com/yuque/0/2022/webp/1732113/1662018065667-c58dfe0a-c244-4699-b9f1-6d0dc38a7e7e.webp)

### 创建
+ 在 map 较小时，可能会在栈或堆直接分配
    - 在栈上分配，通过快速哈希方式创建
    - 在堆上分配，通过 runtime.makemap_small 初始化
    - 其余调用 runtime.makemap
        * 内存校验，判断是否超出内存限制
        * 创建结构体
        * 获取 hash 种子
        * 计算合适的 B 值（bucket 大小为 2 ^ B）
        * 创建 buckets，得到 buckets 的地址和溢出的 bmap
        * 返回 hmap 指针
+ float 类型可以作为 map 的 key 吗？
    - 可以，但是由于精度问题会导致一些诡异的问题

### key 的定位
+ 将 key 传入 hash function，得到 hash 值
+ 取低 B 位，计算出 key 所在的 bucket
+ 取高 8 位，在 bucket 中寻找 tophash 值相等的 key，然后比较 key，不相等则返回 0 值
+ 如果 bucket 里找不到，并且 bucket 的 overflow 不为空，则沿着 overflow 往下一个 bucket 寻找，直到所有的 overflow bucket 找完为止

### 扩容
+ 触发扩容的两种情况
    - overLoadFactor 函数：数据太多，增量 1 倍扩容
        * loadFactor := count / (2 ^ B)，loadFactor 超过 6.5 时触发扩容
    - tooManyOverflowBuckets 函数：判断是否需要等量迁移。map 由于删除操作，溢出 bucket 很多，但是数据分布很稀疏，通过等量迁移可以让数据更紧凑地存储在一起，节约空间
        * B 小于 15，overflow 的 bucket 数量超过 2 ^ B
        * B 大于等于 15，overflow 的 bucket 数量超过 2 ^ 15

### 无序性
+ 每次遍历都是从一个随机值序号的 bucket 开始遍历，并且从这个 bucket 的一个随机 topHash 索引开始遍历

### 并发安全
+ 在读写的过程中会检测写标志，如果发现写标志位会直接 panic
+ 解决方式
    1. sync.RWMutex
    2. sync.Map

### 其他
1. 比较两个 map 相等，不能直接 ==，只能遍历比较

## 字符串
[https://draveness.me/golang/docs/part2-foundation/ch03-datastructure/golang-string/](https://draveness.me/golang/docs/part2-foundation/ch03-datastructure/golang-string/)

+ 本质上是一个只读（即不可扩容，因此不需要 cap 属性）的字节数组

```go
type StringHeader struct {
	Data uintptr
	Len  int
}
```

因为只读，所以在字符串上的操作都是通过拷贝实现的，如`[]byte(str)`或`str1 + str2`

+ 字符串两种赋值方式

```go
func str() {
	str1 := "hello\nworld"
    str2 := `hello
world`
    fmt.Println(str1 == str2) // true
}
```

+ `[]byte(str)`和`string(bytes)`都会发生内存拷贝，如果能确保不改变值，可以通过`unsafe.Pointer`进行类型转换，避免内存拷贝

```go
func UnsafeBytesToString(b []byte) string {
	return *(*string)(unsafe.Pointer(&b))
}

// will panic if modify the member value of []byte
func UnsafeStringToBytes(s string) []byte {
	sh := (*reflect.StringHeader)(unsafe.Pointer(&s))
    bh := reflect.SliceHeader{Data: sh.Data, Len: sh.Len, Cap: sh.Len}
    return *(*[]byte)(unsafe.Pointer(&bh))
}
```

## context
```go
type Context interface {
    // 返回工作完成或取消的时间
	Deadline() (deadline time.Time, ok bool)
    // 多个 Goroutine 同时订阅管道中的消息，一旦接收到取消信号就立刻停止当前正在执行的工作
	Done() <-chan struct{}
    // 如果 context.Context 被取消，会返回 Canceled 错误；
    // 如果 context.Context 超时，会返回 DeadlineExceeded 错误；
	Err() error
    // 获取值，例如请求链路日志 ID 等
	Value(key interface{}) interface{}
}

// Done() 示例
func main() {
    // 由于 context 超时时间为 1 秒，因此 handle() 方法可以在 500 ms 内处理完，不会进入 ctx.Done() 分支返回
    const handleTime := 500*time.Millisecond
    // 由于 context 超时时间为 1 秒，handle() 方法需要 1500 ms 才能处理完，会直接进入 ctx.Done() 分支返回
    // const handleTime := 1500*time.Millisecond
	ctx, cancel := context.WithTimeout(context.Background(), 1*time.Second)
	defer cancel()

	go handle(ctx, handleTime)
	select {
	case <-ctx.Done():
		fmt.Println("main", ctx.Err())
	}
}

func handle(ctx context.Context, duration time.Duration) {
	select {
	case <-ctx.Done():
		fmt.Println("handle", ctx.Err())
	case <-time.After(duration):
		fmt.Println("process request with", duration)
	}
}

type valueCtx struct {
	Context
	key, val interface{}
}

// 可以看到一个 valueCtx 存储一对 kv，查询方式为递归查询，因此 valueCtx 只适合用来传上下文相关的值
func (c *valueCtx) Value(key interface{}) interface{} {
	if c.key == key {
		return c.val
	}
	return c.Context.Value(key)
}
```



# 并发
## goroutine
### 与线程的区别
1. 内存占用
    1. goroutine 的栈内存消耗为 2 KB，线程为 1 MB
2. 创建和销毁时用户级别，线程是操作系统级别
3. 切换成本更小，线程切换需要保存各种寄存器，goroutine 切换只需要三个寄存器

### 调度器


## channel
+ 底层结构：一个环形队列和两个双向链表
    - 环形队列作为缓冲区存放 channel 接受的值
    - 双向链表存放等待读和等待写的 goroutine

![](https://cdn.nlark.com/yuque/0/2022/webp/1732113/1662021306899-0c5ceb83-3e02-4b38-a043-ecb853581c55.webp)

### 数据发送
通过 chansend 函数完成

1. 异常检查
    1. 检查 channel 是否为空
    2. 检查非阻塞模式中（select 的场景），channel 是否已满，元素不能放进环形队列
        1. 无环形队列且没有等待的协程
        2. 有环形队列，但是已经满了
2. 同步发送（等待队列有 sudog）
    1. 上锁，保证线程安全
    2. 检查 channel 是否关闭，关闭则抛出 panic（不能写关闭的 channel，但是可以读关闭的且有缓冲的 channel）
    3. 等待队列中有元素，取出第一个非空的 sudog，调用 send() 将数据拷贝给等待接收的 goroutine（sendDirect() 将数据拷贝到接受变量的内存地址上，然后将阻塞的 goroutine 状态变为 scanwaiting 或 runnable，等待调度）
3. 异步发送（缓冲区未满）
    1. 将数据写入缓冲队列
4. 阻塞发送（缓冲区已满）
    1. 把 goroutine 相关结构入队，等待条件满足的环形
    2. gopark 把 goroutine 切走，让出 cpu
    3. 被唤醒后会将一些属性置零并且释放 sudog 结构体，完成阻塞发送

### 数据接收
通过 chanrecv 完成

1. 异常检查
    1. channel 是否为 nil
    2. 检查非阻塞模式下，是否可以立刻接收数据
        1. 无环形队列且没有等待的协程
        2. 有环形队列，但是是空的
2. 同步接收
    1. 加锁，判断 channel 是否关闭，如果关闭且环形队列没有内容则直接返回结果
    2. 等待队列有等待的发送者，调用 recv() 完成同步接收
        1. 缓冲区为空，直接从发送方接收值
        2. 缓冲区不为空，先将缓冲区的数据拷贝至目标地址，再将写等待队列的头部节点取出，将其 goroutine 发送的数据拷贝至缓冲区，修改 goroutine 的状态为 runnable 等待调度
3. 异步接收
    1. 等待队列为空，缓冲区不为空，直接从缓冲区接收数据，将数据从环形队列中出队
4. 阻塞接收
    1. 缓冲区为空且没有等待发送的 goroutine，进入阻塞发送逻辑
    2. 封装自身 goroutine 为 sudog 并入列读等待队列，然后休眠，让出 cpu

### 关闭
1. 异常检查：检查是否已经关闭
2. 释放写等待队列和读等待队列的 sudog，加入 glist（待清楚队列）中
    1. 等待读的 goroutine 会得到零值
3. 调度 goroutine：触发 glist 中 goroutine 的调度

### select
与 select 语句结合时，是非阻塞调用（阻塞了的话没办法进行其他 case 的操作）

### happened-before
+ 第 n 个 send 先于第 n 个 receive finished
+ 对于容量为 m 的 channel，第 n 个 receive 先于第 n+m 个 send finished
+ channel close 先于 receiver 得到通知（此时 receiver 得到的可能是零值）

## Mutex


## atomic


## WaitGroup


## sync.Once


## sync.Map


# 内存管理
## 内存分配
+ 分级分配
    - <font style="color:rgb(0, 0, 0);">微对象</font><font style="color:rgb(0, 0, 0);"> </font><font style="color:rgb(0, 0, 0);">(0, 16B)</font><font style="color:rgb(0, 0, 0);"> </font><font style="color:rgb(0, 0, 0);">— 先使用微型分配器，再依次尝试线程缓存、中心缓存和堆分配内存；</font>
        * <font style="color:rgb(0, 0, 0);">微型分配器：一块 16 字节的内存中，如果已经分配了 12 字节，下一个分配请求小于 4 字节，就可以直接复用这块内存；回收时机是分配的所有对象不被引用</font>
    - <font style="color:rgb(0, 0, 0);">小对象</font><font style="color:rgb(0, 0, 0);"> </font><font style="color:rgb(0, 0, 0);">[16B, 32KB]</font><font style="color:rgb(0, 0, 0);"> </font><font style="color:rgb(0, 0, 0);">— 依次尝试使用线程缓存、中心缓存和堆分配内存；</font>
    - <font style="color:rgb(0, 0, 0);">大对象 (32KB, +∞) — 直接在堆上分配内存；</font>
+ <font style="color:rgb(0, 0, 0);">内存结构</font>
    - ![](https://cdn.nlark.com/yuque/0/2022/png/1732113/1663209475803-45d0fa64-c64f-4a3c-885c-ab906f8035b7.png)
    - mcache 是线程缓存，处理微对象和小对象的分配，它持有  68 * 2 个 mspan，每个 mspan 管理 npages * 8 KB 的内存，线程私有因此不需要加锁
    - mcentral 是中心缓存，访问时需要互斥锁。它为所有 mcache 提供切分好的 mspan 资源。每个 central保存一种特定大小的全局 mspan 列表，包括已分配出去的和未分配出去的。
    - mheap 是页堆，管理 68 * 2 个 mcentral，每个 heapArena 管理 64 MB 的内存。当 mcentral 没有空闲的 mspan 时，会向 mheap 申请。而 mheap 没有资源时，会向操作系统申请新内存。mheap 主要用于大对象的内存分配，以及管理未切割的 mspan，用于给 mcentral 切割成小对象。

## GC
### 三色标记法
+ 标记 - 清除法的改进
+ 三色
    - 白色对象：潜在的垃圾
    - 灰色对象：活跃的对象，从根对象可达，可能存在对白色对象的引用
    - 黑色对象：活跃的对象，从根对象可达，且不存在对白色对象的引用
+ 原理
    1. 初始时根对象都是灰色，其他对象都是白色
    2. 从 root 根对象出发，root 对象引用的对象标记为灰色
    3. 标记完成之后，分析灰色对象是否有引用其他对象，如果有，则将其引用对象标记为灰色，并将自身标记为黑色；如果没有，直接将自身标记为黑色
    4. 重复 C 过程，直到不存在灰色对象（借助队列实现，灰色对象队列为空）
    5. 清理白色对象
+ 由于标记过程与用户线程并行，因此需要借助写屏障来辅助
    - <font style="color:rgb(0, 0, 0);">强三色不变性 — 黑色对象不会指向白色对象，只会指向灰色对象或者黑色对象；</font>
    - <font style="color:rgb(0, 0, 0);">弱三色不变性 — 黑色对象指向的白色对象必须包含一条从灰色对象经由多个白色对象的可达路径</font>
    - 插入写屏障：相对保守的屏障技术，它会**将有存活可能的对象都标记成灰色**以满足强三色不变性
    - ![](https://cdn.nlark.com/yuque/0/2022/png/1732113/1663211324590-6e8af203-c4ce-4142-bab8-f3bdb719aed9.png)
    - 用户线程将指针从指向 B 变为指向白色对象 C，此时立刻将 C 标记为灰色对象，防止其被回收
    - 删除写屏障：
    - ![](https://cdn.nlark.com/yuque/0/2022/png/1732113/1663211839057-c1bb5a2e-fe3e-4a4d-ab65-5a266c1e19b3.png)
    - 第三步删除了 B 对 C 的引用后，A -> C 违反了强三色不变性，C 和 D 违反了弱三色不变性，因此可以通过对 C 着灰色来解决问题

### 回收过程
1. 清除终止
    1. STW，清扫未被回收的 span
2. 标记阶段
    1. 开启写屏障，扫描根对象并将其放入灰色队列
    2. 恢复程序，开始执行三色标记法（过程中被覆盖的指针和新指针被写屏障标记为灰色，新产生的对象标记为黑色）
    3. 扫描 Goroutine 栈时会暂停对应的处理器
3. 标记终止阶段
    1. STW，清理线程缓存中的白色对象
4. 清理阶段
    1. 关闭写屏障，恢复程序
    2. 清理剩余的垃圾

