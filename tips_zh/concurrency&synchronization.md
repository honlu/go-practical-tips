

## 并发与同步

---

### goroutine 间的信号传递：推荐使用 `chan struct{}` 而不是 `chan bool`

在 goroutine 之间传递信号时，很多人会纠结该用 `chan bool` 还是 `chan struct{}`。

**“为什么推荐用 `chan struct{}`？”**

`chan bool` 当然也能传递信号，比如发送 `true` 或 `false` 来表示某种状态。但是问题在于：这两个值的含义可能并不明确。

```go
type JobDispatcher struct {
    start chan bool
}

func NewJobDispatcher() *JobDispatcher {
    return &JobDispatcher{
        start: make(chan bool),
    }
}

// Unclear: What does sending true or false mean?
```

当你看到上面的代码，会很疑惑：“发送 `true` 是启动？还是停止？”这种模棱两可的表达会让维护者感到困惑。

而 `chan struct{}` 就完全是为了“传递一个信号而已”，它不携带任何数据，也不占用内存，就像是在说：“嘿，现在开始吧！”

```go
type JobDispatcher struct {
    start chan struct{}
}

func NewJobDispatcher() *JobDispatcher {
    return &JobDispatcher{
        start: make(chan struct{}),
    }
}

func (j *JobDispatcher) Start() {
    j.start <- struct{}{}
}

// Clear: Sending anything means "start the job"
```

这种写法的好处有两个：

1. **节省内存**：`struct{}` 是零字节类型，发送它不会真正传输数据。
2. **语义清晰**：别人一看 `chan struct{}`，就知道你只是用来通知/同步的，不是为了传值。

缺点是什么？

主要就是 `struct{}{}` 这个写法有点别扭，看起来有些奇怪。但这个小小的不便是值得的，因为它能有效避免误用这个通道去传递数据——毕竟你本意只是想发个“通知”而已。

另外，如果你只是想发一个一次性的信号，连值都可以不用发送，直接关闭通道就行了：

```go
func (j *JobDispatcher) Start() {
    close(j.start)
}
```

关闭 channel 是一种简单又有效的广播方式，它能同时通知多个接收方任务应该开始了，而且完全不需要传递任何数据。  


---

### 使用带缓冲的 Channel 来控制 goroutine 数量（实现信号量机制）

如果你希望限制同一时间内运行的 goroutine 数量，使用缓冲通道模拟“信号量”是个好方法。

通道的容量决定了可以同时运行的 goroutine 数量：

```go
semaphore := make(chan struct{}, numTokens)
```
基本流程：  
goroutine 获取信号（向 channel 中写入一个值）表示“占用一个资源”；任务完成后再读取（释放）资源。

```go
var wg sync.WaitGroup
wg.Add(10)

for i := 0; i < 10; i++ {
    go func(id int) {
        defer wg.Done()
        semaphore <- struct{}{} // 获取资源
        ...
        <-semaphore // 释放资源
    }(i)
}

wg.Wait()
```

在这段代码中：
- `wg.Add(10)`：我们打算启动 10 个 goroutine；
- `make(chan struct{}, 3)`：初始化了一个信号量，只允许最多 3 个 goroutine 同时运行。

如果你想让逻辑更清晰、更结构化一些，可以封装成一个 `Semaphore` 类型，把所有和信号量相关的操作都统一管理起来：

```go
type Semaphore chan struct{}

func NewSemaphore(maxCount int) Semaphore {
	return make(chan struct{}, maxCount)
}

func (s Semaphore) Acquire() {
	s <- struct{}{}
}

func (s Semaphore) Release() {
	<-s
}
```

使用这个自定义的 Semaphore 类型让我们管理资源访问的方式变得更简单清晰:

```go
func doSomething(semaphore *Semaphore) {
    semaphore.Acquire()
    defer semaphore.Release()
    ...
}
```

如果你有更复杂的需求，可以用官方扩展包中的加权信号量（weighted semaphore）：

```go
import "golang.org/x/sync/semaphore"
```

另外，对于更复杂的场景，你可以考虑使用 [golang.org/x/sync/semaphore](http://golang.org/x/sync/semaphore) 包，它提供了“加权信号量”（weighted semaphore）的实现。

这种方式特别适合某些任务需要比其他任务更多资源的情况，比如管理一个数据库连接池时，某些操作可能一次就需要多个连接。

加权信号量的好处在于：**一个 goroutine 可以一次性占用多个“槽位”**，也就是按需“配额式”地使用资源，更灵活。

---

### 使用 `singleflight` 「单飞」 避免重复调用相同的慢函数

假设你有一个函数不太快，每次调用都要等 3 秒，这就很浪费：

```go
func FetchExpensiveData() (int64, error) {
    time.Sleep(3 * time.Second)
    return time.Now().Unix() / 10, nil
}
```

这个模拟函数每隔 10 秒会返回一个不同的数字。

现在，如果我们连续调用这个函数 3 次，总共可能要花将近 9 秒。

你可能会想：用 3 个 goroutine 并发跑，不就可以缩短到 3 秒了吗？是的，时间缩短了，但本质上这个函数还是执行了 3 次，而且拿到的结果其实都一样 —— 这就浪费了资源。

这时候，`singleflight` 包就可以大显身手了。它的设计目标就是：**无论在这 3 秒内我们调用多少次该函数，它都只会真正执行一次，并把结果返回给所有调用者**。

你可以在这里找到它：[http://golang.org/x/sync/singleflight](http://golang.org/x/sync/singleflight)

这时候就可以用 `singleflight` 包：它的作用是：“只执行一次，多个请求共享结果”。

```go
var group singleflight.Group

func UsingSingleFlight(key string) {
    v, _, _ = group.Do(key, func() (any, error) {
        return FetchExpensiveData()
    })
    fmt.Println(v)
}
```

这个过程其实很简单：

我们创建了一个 `singleflight.Group`，然后把耗时的核心函数调用包装在 `group.Do()` 方法里。

这个方法会检查：是否已经有相同的 key 被请求过。如果是，它就会等待那个原始请求的结果出来，再把这个结果一并返回给所有同时发起相同请求的人，而不是让大家都去重复调用。

就这么简单。

> “key 参数是干嘛的？”

这个 key 的作用是告诉 `singleflight`：当前的多个请求其实是“同一个事儿”。所以它就用这个 key 判断是否应该重新执行函数，还是只需等待正在执行的那个结果。

👉 如果你想看一个真实的例子，可以点击这里：https://go.dev/play/p/30kdFPsy2HR

总结一下就是：如果一个函数被同时调用多次，实际上只会执行一次，结果会共享给所有调用者。

> “那为什么不直接用缓存呢？”

这是个常见误解。singleflight 不是用来缓存数据的，它的目的只是为了防止函数在同一时间被重复调用。

比如这个函数甚至可能不会返回什么值。我们只是想确保它只执行一次，就像向服务器发 `ping()` —— 你不会去缓存 ping 的结果，但你也不希望多个 goroutine 同时疯狂地去 ping，那样只会浪费资源。


> `key` 是用来标识“请求是否相同”的，只有 key 一致，才会共享结果。  
> ⚠️ 注意：这不是缓存，它不存储值，只是防止同一时间内执行多次函数。

---

### 想让函数只执行一次？用 `sync.Once` 就对了！

我们都知道，有时候在应用里必须确保某个操作只执行一次，即便同时有很多 goroutine 在跑。

这是就要聊Go中常见的工具 `sync.Once` 配置单例。

举个例子，比如我们有一个配置对象`config`，只需要初始化一次，后续整个应用都用这个配置就行了：
```go
var instance *Config

func GetConfig() *Config {
    if instance == nil {
        instance = loadConfig()
    }

    return instance
}
```
然而问题来了：如果多个 goroutine 同时调用 `GetConfig()`，而配置还没初始化完，就可能会导致 `loadConfig()` 被执行多次——这不是我们想要的结果。

这时候就轮到 `sync.Once` 上场了:

```go
var (
    once     sync.Once
    instance *Config
)

func GetConfig() *Config {
    once.Do(func() {
        instance = loadConfig()
    })

    return instance
}
```

我们把初始化的代码封装在一个函数里，然后传给 `once.Do()`。这样，无论多少个goroutine同时调用 `GetConfig()`，`loadConfig()` 都只会被执行一次。

这就确保了：大家拿到的都是同一个`config`实例，不会重复初始化。

但这里有一个容易被误解的点需要注意：
> `sync.Once` 并不是说“某个函数只能执行一次”，而是说——只要是同一个`sync.Once` 实例，它就只会执行一次，不管你传入的是哪个函数。

举个例子：  
```go
o.Do(f1)
o.Do(f2)
```
即便 `f2` 是个全新的函数，之前从没执行过，只要 `f1` 已经通过这个`sync.Once`执行过了，`f2` 就不会被执行。 这是因为：`sync.Once` 管的不是“哪个函数”有没有执行过，而是“有没有执行过一次”这件事本身。
所以，如果你希望某段代码也只执行一次，就应该创建一个新的 sync.Once 实例，而不是复用旧的。

从 Go 1.21 开始，标准库里的 `sync` 包增加了一些非常实用的新功能，扩展了原有 `sync.Once` 的使用方式。现在我们有了：`sync.OnceFunc`、`sync.OnceValue`、`sync.OnceValues`.这些函数的作用是：把普通函数“包装”成只能执行一次的函数。不管调用多少次，它都只会在第一次真正执行。

定义方式如下：
```go
func OnceFunc(f func()) func()
func OnceValue[T any](f func() T) func() T
func OnceValues[T1, T2 any](f func() (T1, T2)) func() (T1, T2)
```

这个功能特别适合那些执行一次，并且需要返回值（甚至可能返回错误）的函数。
```go
var GetConfig = sync.OnceValues(func() (Config, error) {
    return Config{..}, nil
})
```
`GetConfig` 只会在第一次调用时真正执行一次，用来获取配置（可能也会返回错误）。之后无论再调用多少次，拿到的都是第一次的结果，函数本身不会再跑。

我们来看下 `sync.Once` 底层是怎么实现的。它内部其实就两个核心东西：

- 一个原子计数器（0 或 1）
- 一个互斥锁（mutex）

结构大概长这样：
```go
type Once struct {
    done atomic.Uint32
    m    Mutex
}
```
**快速路径（fast path）**
调用`once.Do(f)`时，首先检查 `done` 计算器是否为 0。如果不是（说明已经执行过了），就直接跳过函数执行，非常快。

**慢速路径（slow path）**
如果 `done` 计数器还是 0，就说明函数还没跑过，`sync.Once`就会走慢速路径，执行如下：
```go
func(o *Once) doSlow(f func()) {
    o.m.Lock()
    defer o.m.Unlock()

    if o.done.Load() == 0 {
        defer o.done.Store(1)
        f()
    }
}
```
解释一下：
- `o.m.Lock()`：先加锁，确保同时只有一个 `goroutine` 能跑下面的代码。
- `o.done.Load() == 0`：再确认一次，期间可能已经有别的 goroutine 抢先执行了。
- `o.done.Store(1)`：函数执行完，把计数器改为1，表示函数已经运行过不能再运行。

> "为什么要分快路径和慢路径？"
这是为了兼容性能和安全。
- 快速路径不加锁，效率高，用于后续调用。
- 慢速路径只在第一次执行时加锁，确保线程安全。这个初始运行可能会慢一点慢，但一旦完成后，每次运行 `once.Do()` 都会很快，长远来说是好事。

---

### 用 `errgroup` 更优雅地管理多个 goroutine 与错误处理

当我们同时处理多个 goroutine 时，怎么管理它们以及它们可能抛出的错误，确实有点麻烦。

我们可能已经熟悉了用 `sync.WaitGroup` 来管理多个 goroutine，但还有一个叫做 `errgroup` 的工具，它在这个场景下更方便，能让事情变得更简单清晰。

首先，你需要装这个包：
```shell
$ go get -u golang.org/x/sync
```

假设你想并发从多个URLs中获取数据：
```go
func main() {
    urls := []string {
        "https://blog.devtrovert.com",
        "https://example.com",
    }

    var g errgroup.Group

    for _, url := range urls {
        url := url // safe before Go 1.22
        g.Go(func() error {
            return fetch(url)
        })
    }

    if err := g.Wait() ; err != nil {
        log.Fatal(err)
    }
}
```
这段代码里，我们同时请求两个页面。重点是`g.Wait()`这个方法，它会等待所有 goroutine 完成，并且如果有任何一个 goroutine 返回错误，它会返回第一个遇到的错误。

`errgroup` 简化了多个goroutines的管理和运行时的错误处理。

关键点如下：
- 每个任务通过 `g.Go()` 启动一个 goroutine，传入具体要执行的函数。
- 用 `g.Wait()` 等待所有 goroutine 完成，它返回第一个出错的错误（不会收集所有错误）。
- `errgroup` 可以配合 context 使用：使用 `errgroup.WithContext()` 时，如果某个 goroutine 出错，其他 goroutine 可以通过 context 被自动取消。

**这背后发生了什么呢？**
`errgroup.Group`本质上是对标准库几个工具的组合封装：
```go
type Group struct {
    cancel func()
    
    wg sync.WaitGroup
    sema chan struct{}
    errOnce sync.Once

    err error
}
```
- `wg sync.WaitGroup`: 用来等待所有goroutine完成。
- `errOnce sync.Once`: 确保以线程安全的方式记录第一个错误，避免竟态。
- `sema`: 是一个`semaphore`（信号量，带缓冲的channel），限制通知运行的goroutine数量。甚至可以使用`errg.SetLimit()`设置goroutine的并发限制数。

当你调用`g.Go()`时，其实就是添加一个要并行执行的任务，这个任务是一个不带任何参数但返回`error`的函数。通过`semaphore`（信号量）管理同时运行的并发`goroutine`的数量。
信号量有助于控制执行并防止过多的任务同时运行。
> 后面会专门介绍「带缓冲channel作为信号量限制goroutine执行」

实际运行的样子：
```go
func (g *Group) Go(f func() error) {
	if g.sem != nil { // 如果设置了并发限制，会先往 sema 写入一个 token，占位。
		g.sem <- token{}
	}

	g.wg.Add(1)
	go func() {
		defer g.done()

		if err := f(); err != nil {
			g.errOnce.Do(func() {
				g.err = err
				if g.cancel != nil {
					g.cancel(g.err)
				}
			})
		}
	}()
}
```

这段程序中，错误处理通过`errOnce`机制集中进行，确保只记录第一个错误。如果发生错误并提供上下文，就会触发该上下文的取消信号。

这段意味着，一旦一个goroutine出错，他就会阻止其他goroutine进行不必要的工作。

现在，让我们看看当一个 goroutine 完成时会发生什么情况：
```go
func (g *Group) done() {
    if g.sem != nil {
        <-g.sem
    }
    g.wg.done()
}

func (g *Group) Wait() error {
	g.wg.Wait()
	if g.cancel != nil {
		g.cancel(g.err)
	}
	return g.err
}

```
`done()` 起着重要作用，它不仅向 `WaitGroup` 发送 goroutine 已完成的信号，而且还管理着 semaphore，以确保运行 goroutine 的限制得到遵守。

> 提示：并发使用`goroutines`并不总是最好的解决方案，尤其是对于快速完成的任务，顺序执行可能更简单也更高效。
---

### 在容器环境中（K8s/Docker）合理设置 `GOMAXPROCS`

在像 kubernetes 或 docker 这样的容器环境中，有时你需要考虑手动调整`GOMAXPROCS`.

**那GOMAXPROCS是啥？**
Go程序虽然默认可以执行很多线程（比如1万个没问题），但真正能“并行”跑起来的线程数量由`GOMAXPROCS`控制。简单说：它决定了最多多少个线程可以“同时”在CPU上执行。

默认情况下，Go会把`GOMAXPROCS`设置为你机器上的逻辑CPU核心数`runtime.NumCPU()`:
```go
fmt.Println("GOMAXPROCS:", runtime.GOMAXPROCS(0))
fmt.Println(runtime.NumCPU())

// Output:
// 8
// 8
```
也就是说，像我的`MacOS`有8个逻辑核心，Go默认就会让最多8个线程并行执行。

#### 在容器里运行Go程序（比如Docker或kubernetes）

在像 Kubernetes 这样的容器化设置中，你完全可以限制每个容器的 CPU 使用量。  

当你设置这些限制时，你实质上是在告诉容器：_“这是你的 CPU 资源份额.”_ 例如，250m 的限制意味着 1/4 个内核，而 1 的限制意味着 1 个完整的内核。  

遗憾的是，Go 并不会自动识别这一点，它仍然会看到并使用主机（或节点）上可用的 CPU 内核总数，而不仅仅是分配给容器的部分。  

这可能导致 Go 程序根据容器的设置，试图使用比它应该使用的更多 CPU 内核。  

> "但为什么呢？我还以为使用更多的 CPU 会更好呢"。

嗯，看起来使用更多的 CPU 总会更好，但有几个原因导致情况未必如此：

1. 上下文切换「context switching」：当线程数量多于 CPU 内核数量时，操作系统需要在这些线程之间频繁切换，而这种切换会拖慢运行速度，因为它需要时间和资源来处理。

2. 调度效率低「inefficient scheduing」： 由于 CPU 的限制，Go 调度器可能会创建更多可运行的 goroutines，而不是实际可执行的 goroutines，这就导致了对 CPU 时间的争夺，线程实质上是在争夺处理能力。

3. 对 CPU 约束的低效利用「inefficient use of CPU Bound」： Go 应用程序通常都是和 CPU 绑定的，这意味着当每个线程都能在自己的 CPU 内核上运行而无需等待其他线程时，它们的性能才最佳。

如果 “GOMAXPROCS ”多于可用的 CPU 内核，就会迫使 Go runtime(运行时)为多于内核的线程进行规划，从而导致这些 CPU 绑定任务的执行效率低下。

那么，我们能做些什么呢？

#### 解决方法：

对于那些更喜欢放手策略的人来说，“uber-go/automaxprocs ”软件包可能正是你所需要的。 

这个库会自动调整 `GOMAXPROCS` 以匹配容器的 CPU 限制：

```go
import _ "go.uber.org/automaxprocs"

func main() {
...
}
```

如果你熟悉部署配置或 Pod 规范，也可以手动设置环境变量 `GOMAXPROCS`，使其与容器的 CPU 限制保持一致。

然而，从 DevOps 的角度来看，通常建议 [避免设置 CPU 限制](https://home.robusta.dev/blog/stop-using-cpu-limits)，同时始终指定 CPU 请求。

#### 它是如何工作的？

原来，当我们为 pod 设置 `resources.limits.cpu`时，一个名为 `Cgroups`或 `Control Groups`的组件会为我们管理这些资源。`Cgroups` 是 Linux 内核的一部分，可帮助管理 CPU、内存和 I/O 带宽等资源，并为进程组设定优先级。

下面是一段代码，供你参考：

```go
type CGroup struct {
 path string
}

// NewCGroup 从给定路径返回一个新的 *CGroup
func NewCGroup(path string) *CGroup {
 return &CGroup{path: path}
}

// CGroups 是一个map，它将每个 CGroup 与其子系统名称相关联
type CGroups map[string]*CGroup
```

我们在暂存集群中进行了一些测试，看看实际效果如何，结果与我们预期的差不多：

```bash
# no limit
maxprocs： Leaving GOMAXPROCS=4: CPU quote undefined

# 200m = 1/5 核心
maxprocs: Updating GOMAXPROCS=1: using minimum allowed GOMAXPROCS

# 1.5 个核心
maxprocs: Updating GOMAXPROCS=1: determined from CPU quotaS

# 2 个内核
maxprocs: Updating GOMAXPROCS=2: determined from CPU quota
```

正如你所看到的，在没有设置限制的情况下，它最多使用了工作节点(worker node)的 4 个内核。这就回到了我们之前讨论过的通常不设置 CPU 限制的原因。

那么，`uber-go/automaxprocs` 和 `cgroups` 是怎么处理的呢？

我们现在可能已经猜到，`automaxprocs`软件包依赖 `Cgroups` 来完成工作，具体过程如下：

1. 检查环境中的 GOMAXPROCS

首先，软件包会检查 GOMAXPROCS 环境变量是否已设置。如果设置了，软件包将尊重该值，不做任何更改：

```go
if max, exists := os.LookupEnv(_maxProcsKey); exists {
... // 保持 GOMAXPROCS 原样
}
```

2. CPU 配额检索

如果没有设置 GOMAXPROCS，软件包就会尝试找出当前进程的 CPU 配额，具体方法是查找 Linux cgroups，看看是否为容器设置了 CPU 配额。

> "内部发生了什么呢？

它会通过查看文件系统和读取特定文件来检查系统使用的是 Cgroups v1 还是 v2。你可能不需要关心细节，但它会查看 `/proc/self/mountinfo` 以了解挂载点，并查看 `/proc/self/cgroup` 以了解 cgroup 成员情况。

- 对于 cgroups v1，它会从 cgroup 文件系统（通常挂载在 /sys/fs/cgroup 下）中的文件读取参数。它会查找 `cpu.cfs_quota_us` 和 `cpu.cfs_period_us` 来计算 CPU 配额。
- 对于 cgroups v2，它会检查 cgroup 目录中的 `cpu.max` 文件，该文件在单独行中列出了配额和周期。

如果计算值低于最小阈值（默认为 1，但可自定义），则默认为最小值。

3. 未指定 CPU 限制

如果未定义配额，GOMAXPROCS 将保持默认设置。

### `sync.Pool` + 泛型让它更安全

那么，让我们转向 `sync.Pool`。对于不熟悉该功能的读者，`sync.Pool` 是 Go 标准库中的一项功能，专门用于复用对象。  

这非常实用，因为它能减少内存分配的次数，从而提升性能。

假设你有一台超级快的打印机，需要每分钟打印 100 页。如果打印机每次打印一页都跑去储物室拿新的墨水和纸张，这显然没有意义，对吧？ 

相反，你可以这样想象：打印机有一个装有大约 100 张纸的托盘，可以反复使用。你手头有一批固定的物品，可以反复使用，既节省时间又节约资源。  

以下是一段代码示例，展示这种机制在代码中的实现方式：

```go
var paperPool = sync.Pool{
    New: func() interface{} {
        return new(Paper) // create a new sheet of paper
    }
}

func printPage() {
    page := paperPool.Get().(*Paper)
    page.Reset() // make sure to reset the page before using it

    defer paperPool.Put(page)

    page.Print()
}
```

但有几点需要注意：

- `sync.Pool` 没有固定大小，这意味着你可以不断添加和检索项目，而没有硬性限制。
- 一旦将对象放回池中，就不要再管它了，因为它可能会被垃圾回收器移除或回收。
- 由于对象可能具有状态，因此在将它们放回池中或取出后立即清除或重置其状态至关重要。

#### 类型安全的池Pool

我们已经知道，`sync.Pool` 使用空的 `interface{}` 来存储和检索项。这非常灵活，但不提供类型安全。我们可以将此过程包装起来，使其具有类型安全：

```go
type Pool[T any] struct {
    internal sync.Pool
}

func NewPool[T any](newF func() T) *Pool[T] {
    return &Pool[T]{
        internal: &sync.Pool{
            New: func() interface{} {
                return newF()
            }
        }
    }
}
```

通过这种设置，我们创建了一个与特定类型 T 绑定的池，尽管它在底层仍使用 `interface{}`。类型安全性的关键在于如何以类型安全的方式获取和存放数据：

```go
func (p *Pool[T]) Get() T {
    return p.internal.Get().(T)
}

func (p *Pool[T]) Put(x T) {
    p.internal.Put(x)
}
```

当我们从池中检索数据时，只需将接口转换为类型 T。你可能会好奇为什么我们在转换过程中不检查错误。

> “为什么不检查转换错误呢？”

问题是：我们添加的泛型层（generic layer）会自动确保类型正确，因此无需担心转换失败。

`sync.Pool` 只会包含类型为 T 的实例，因此断言 `p.internal.Get().(T)` 是安全的，在正常情况下不会引发 panic。这种解决方案使我们的代码保持简洁和高效。

### 制作具有内置锁功能的类型（`sync.Mutex`直接嵌入）

在应用程序的不同部分需要并发访问共享资源时，通常会使用 `sync.Mutex` 来管理访问并确保数据完整性。

通常情况下，您可能会看到这样的内容：

```go
type MyStruct struct {
    mu sync.Mutex

    ...
}

func (s *MyStruct) DoSomething() {
    s.mu.Lock()
    defer s.mu.Unlock()

    ...
}
```

这种模式虽然有效，但可能会导致代码中出现大量的 `mu.Lock()` 和 `mu.Unlock()` 调用，从而增加代码的阅读和维护难度。

一种更简洁的处理方式是将 `sync.Mutex` 直接嵌入到结构体中，这样就可以直接在结构体的实例上调用 `Lock` 和 `Unlock` ：

```go
type MyStruct struct {
    sync.Mutex

    ...
}

func (s *MyStruct) DoSomething() {
    s.Lock()
    defer s.Unlock()

    ...
}
```

这种解决方案简化了语法，使锁在方法实现中的侵入性降低。不过，有一点需要注意，如果 `MyStruct`是导出的（即以大写字母开头），直接嵌入 `sync.Mutex` 也会使 `Lock` 和 `Unlock` 方法被公开访问。

因此，这种技术通常更适用于包的内部类型，因为在这种情况下，公开锁并不是问题。对于公共类型，您可能仍然倾向于将 mutex 保留为私有字段，以控制同步原语的暴露。

**泛型技巧**

在查看一些编码概念和资源时，我发现了一个非常有趣的方法，它引起了我的注意。

它涉及在 Go 中使用泛型来创建一个类型安全、可锁定的结构，下面是实现它的方法：

```go
type Lockable[T any] struct {
    sync.Mutex
    Value T
}

func (l *Lockable[T]) SetValue(v T) {
    l.Lock()
    defer l.Unlock()

    l.Value = v
}

func (l *Lockable[T]) GetValue() T {
    l.Lock()
    defer l.Unlock()

    return l.Value
}
```

通过这种结构，我们可以用互斥来封装任何类型，从而确保数据的安全，避免并发访问问题。我们可以通过几种方式使用这种泛型类型，要么直接使用，要么在此基础上定义一种新类型：

```go
// directly use
func main() {
    var safeUser Lockable[User]

    safeUser.Set(..)
    safeUser.Lock()
}

// or make a new type
type IntLockable Lockable[int]
```

> "为什么我们不直接嵌入 T 类型，而要用 Value T？

原因是 Go 目前不支持在结构体中直接嵌入类型参数。

### 如何向channel发送不会阻塞

通常，我们向通道发送内容，代码会在那里等待接收器准备好接受数据，比如：

```go
ch <- value // blocking operation
```

那如果我们不想"傻等"资源释放怎么办呢？

比如，我们在用信号量（semaphore） 来管理并发资源的系统，我们有一个类似 `TryAcquire()` 的函数，它能立即告诉我们是否因为资源都被使用而无法获取资源。

_[使用带缓冲的 Channel 来控制 goroutine 数量（实现信号量机制）](#使用带缓冲的-channel-来控制-goroutine-数量实现信号量机制)_

让我们以 `errgroup` 软件包为例，看看在实践中是如何处理类似问题的。这个软件包内部使用了一个简单的信号机制（semaphore mechanism）来限制同时运行的 goroutines 数量。

下面是如果信号（semaphore）已满，而你又试图启动一个新的 goroutine 时会发生的情况：

```go
func (g *Group) TryGo(f func() error) bool {
    if g.sem != nil {
        select {
        case g.sem <- token{}:
        default:
            return false
        }
    }

    ...
}
```

神奇之处在于 `select{}` 语句。通常，`select{}`用于等待多个通道操作，但在这里，它被巧妙地用于尝试非阻塞发送：

- `case g.sem <- token{}`： 这一行尝试向 （semaphore）信号通道 g.sem 发送令牌。如果通道中还有空间（这意味着它没有达到容量），则标记发送成功，函数继续执行。
- `default` 如果 g.sem 通道已满，这部分功能就会启动。它不会阻塞等待空间释放，而是直接进入`default`case。

`select` 语句中的 `default` 分支是一个有点“狡猾”但很有用的路径。它的特点是：只要没有其他 case 准备好，它就会立即执行。在这种情况下，它会通过返回 `false` 让我们立即知道，我们无法启动新的 goroutine，因为我们已经达到了为活动 `goroutine` 设置的限制。