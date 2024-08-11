# Go实用技巧
- [Go实用技巧](#go实用技巧)
  - [Context](#context)
    - [1. 为了goroutines更可靠，避免使用context.Background()](#1-为了goroutines更可靠避免使用contextbackground)
    - [2. 不幸得是，context.Value 不是我们的朋友](#2-不幸得是contextvalue-不是我们的朋友)
    - [3. 使用context.WithoutCancel 保持 context 活跃](#3-使用contextwithoutcancel-保持-context-活跃)
    - [4. 使用context.AfterFunc设置取消context后调度函数](#4-使用contextafterfunc设置取消context后调度函数)
    - [5. 使用未导出的空结构体(struct{})作为context key](#5-使用未导出的空结构体struct作为context-key)
    - [6. 处理延迟调用的错误以防止忽视错误](#6-处理延迟调用的错误以防止忽视错误)
    - [7. 始终跟踪goroutine的生命周期](#7-始终跟踪goroutine的生命周期)
    - [8. 避免使用time.Sleep()，它不是context感知的并且不能被中断](#8-避免使用timesleep它不是context感知的并且不能被中断)

## Context

### 1. 为了goroutines更可靠，避免使用context.Background()

在我们同时管理多个任务时，我们会使用goroutine，对吧？每个goroutine专用于特定的功能，比如发HTTP请求或者查询数据库。当这些任务需要暂停时问题就出现了，这也是棘手的地方。因为我们希望goroutines永久停止或阻塞时，有办法退出。

> “为什么我们应该避免直接使用 context.Background() ？”

我们之所以避免直接使用 context.Background()，主要是因为它无法在出现问题时停止或取消执行。它是我们用context的最简单形式，没有值（values） 、没有截止日期（截止日期）、没有取消信号（cancellation signals）。当执行卡住或要顺利结束时，可能会出现一个问题。

为了解决这个问题，通常我们依靠两个策略：取消（cancel）和超时（timeout）:

```go
context.WithTimeout(ctx, duration)
context.WithTimeoutCause(ctx, duration, errors.New("custom message"))

context.WithCancel(ctx)
context.WithCancelCause(ctx)

context.WithDeadline(ctx)
context.WithDeadlineCause(ctx, deadline, errors.New("custom message"))
```

有了这些工具，我们启动每个 goroutine 都带有明确的预期：“goroutine可以及时完成任务或者解释为什么不能及时完成，另外必要情况下可以取消任务。”

以下几点请记住：

- WithTimeout 其实就是换了个名字的 WithDeadline。
- 最近更新的 Go（版本 1.20 和 1.21）中添加的 XXXCause 函数提供了更好的错误报告。
- 如果 `XXXCause` 发生超时，它会提供更详细的错误消息：“*context deadline exceeded: custom message.*”

> "channel怎么样？我不想在channel上永久等待。"

为了确保不会无限等待处理channel，更好的管理方法是使用 select 语句，它允许我们设置超时选项：

```go
select{
  case result := <-ch：
    fmt.Println("Received result:", result)
  case <-time.After(5*time.Second):
    fmt.Println("Timed out")
}
```

不过需要注意：使用 time.After 可能会导致短期内存泄露。在某些情况下，使用time.Timer 或 time.Ticker 可能更有效，因为它们能让我们更好地控制时间。

> 译者补充：关于 time.After 可能导致内存泄露的文章，可参考学习：
>
> > 1. [[golang\]golang time.After使用不当导致内存泄露问题分析 - luoming1224 - 博客园 (cnblogs.com)](https://www.cnblogs.com/luoming1224/p/11174927.html)
> > 2. [Go坑：time.After可能导致的内存泄露问题分析 - 九卷 - 博客园 (cnblogs.com)](https://www.cnblogs.com/jiujuan/p/17369964.html)

本文将在接下来技巧中深入探讨替代方案及其价值。



### 2. 不幸得是，context.Value 不是我们的朋友

context.Value 似乎是一个方便的工具，因为它可以在 context 中携带一些数据，然后在需要的地方取出这些数据。

这会让我们的函数签名简介明了，不是吗？典型的例子是这样的：

```go
func A(ctx context.Context, transactionID string){
    payment := db.GetPayment(ctx, transactionID)
    ctx := context.WithValue(ctx, "payment", payment)
    
    B(ctx)
}

func B(ctx context.Context){
    ...
    
    C(ctx)
}

func C(ctx context.Context){
    payment, ok := ctx.Value("payment").(payment)
    ...
}
```

在这段代码中，函数 A 获取付款记录并将其添加到 context 中， 在 B 内调用的函数 C 检索此付款。这种方法避免直接通过函数 B 传递付款记录，该函数不需要了解付款。

此方法看起来不错，因为：

- 允许我们省略函数传递特定不使用的数据，就像 B 函数一样。
- 允许将必要的数据保存在 context 中。
- 避免函数签名中的额外参数。

为什么不在函数中 A 中直接调用函数 C 呢？通常情况下，C与B的逻辑结合得很深，可能依赖它的某些计算和参数。

那么问题出在哪里呢？问题就出在这里：

- 放弃了Go在编译过程中提供的类型检查安全性。
- 我们将数据放入黑匣子中并希望以后能再找到它，而一周之后可能就像盲人搜索一样。
- 由于隐士传递，付款数据似乎可有可无，但实际上非常重要。

从个人角度来看，使用 ctx.Value 的主要问题在于它如何隐藏数据。这就像把东西放在一个没有明确标签的保险箱里。当然，数据被保存起来了，但检索数据却成了一个猜谜游戏。

明确我们正在传递的消息通常会减少以后的麻烦。

> "那么，什么时候适合使用 context.Value() ?"

最好限制它的使用范围，但 Go 文档建议在跨API和进程间传递请求范围的值时使用它。以下是一些很好的用途：

你可以考虑使用它来跟踪某些与请求相关的数据，例如：

- 跟踪请求的开始时间
- 记录访问者的IP地址
- 管理追踪和跨度IDs
- 识别正在访问的HTTP路由
- ....

> 上面例子中 'payment' 支付数据与请求无关吗？

如果 ‘payment’ 支付信息在多个函数中都很重要，那么在函数参数显示传递它会更清晰、安全，并且有助于任何阅读代码的人立即理解该函数直接与‘payment’支付数据交互。

一般来说，最好避免在 context 中嵌入关键业务数据，这种策略可以保持代码清晰度和可维护性。

### 3. 使用context.WithoutCancel 保持 context 活跃

当在Go中实用context时，非常简单的一件就是直接使用带有取消功能的context。如果取消父级context，所有子context也会被取消。

比如，下面这个简单的例子：

```go
parentCtx, cancel := context.WithCancel(context.Background())
childCtx, _ := context.WithCancel(parentCtx)

go func(ctx context.Context){
    <-ctx.Done()
    fmt.Println("Child context done")
}(childCtx)

cancel()
```

此段代码中，一旦我们取消 parentCtx, childCtx 也会被取消。这通常是我们想要的，但有时我们可能需要一个子context能够继续运行，即使父级context被取消了。

在Go中处理HTTP请求时，我们会经常遇到一个场景：在处理主请求后启动goroutine处理任务，如果处理不仔细可能会导致错误：

```go
func handleRequest(req *http.Request){
    ctx := req.Context()
    
    go hookAfterRequest(ctx)
}
```

另外，在我们考虑处理HTTP请求时，希望即使客户端断开连接，我们仍然需要记录详细信息并收集指标，而不是取消运行。

> "好吧，我将为这些任务创建一个新的 context ."

这虽然是一种方法，但我们通常需要一些信息或值来进行日志记录和指标等任务，新的context不会继承原始的context中的任何信息或值。

以下是保留这些值的方法：

```go
parentCtx := context.WithValue(context.Background(), "requestID", "12345")
childCtx, _ := context.WithCancel(parentCtx)

fmt.Println("Propagated value:", childCtx.Value("requestID")) // 12345
```

类似HTTP请求场景中，为了确保即使 parent context被取消某些操作也能继续，我们可以尝试一下操作：

```
func handleRequest(req *http.Request){
	ctx := req.Context()
	
	// Create a chile context that doesn't cancel when the parent does
	// 创建一个子context，当父context取消时它不取消
	unCancelLabeCtx := context.WithoutCancel(ctx)
	
	go func(ctx context.Context){
		// 如果父context被取消，日志操作不会被中断
		getMetrics(unCancelLableCtx, req)
	}(unCancelLableCtx)
}
```

基本上，Go1.21中引入的context.WithoutCancel()函数允许某些操作进行，而不会收到其parent context取消的影响。

### 4. 使用context.AfterFunc设置取消context后调度函数

现在我们讨论如何在父context停止后仍保持context活跃，这是go1.21中引入的另一个方便实用的特性，即context.AfterFunc。此函数允许你安排回调函数 f 在 ctx 结束后在单独的goroutine中运行，无论是由于取消还是超时。

使用方法如下：

```
ctx, cancel := context.WithTimeout(parentCtx, 5*time.Second)
defer cancel()

stop := context.AfterFunc(ctx, func(){
	fmt.Println("Cleanup operations after context is done")
})
```

这段代码设置了一个清理任务，以便在context完成后运行，它对于清理、日志记录或取消后需要发生的其他操作等任务非常有用。

> "回调（callback）函数什么时候运行呢？"

一旦parent context的ctx.done通道发送信号，它就会新的goroutine中启动。

> "如果context已经取消了怎么办？"

然后回调立即运行，当然是在一个新的goroutine中。

以下是一些概要：

- AfterFunc 可以在相同的context中多次使用，并且你设置的每个任务都独立运行。
- 如果调用AfterFunc时，context已经完成，它会立即在新的goroutine中触发该函数。
- 它提供了一个stop函数，可以让你取消计划的功能。
- 使用stop函数是非阻塞的，他不会等待函数完成，而是立即停止它。如果你需要该函数和你的主要工作同步，你需要自己管理好顺序。

让我们深入研究一下AfterFunc返回的函数stop() :

```go
stop := context.AfterFunc(ctx, func(){
    ...
})

if stopped := stop(); stopped{
    fmt.Println("Remove the callback before context is done")
}
```

如果你在context结束之前调用stop()并且回调还没有运行（意味着goroutine还没有触发），stopped将返回true, 表明你成功停止了运行中的回调。

但是，如果stop() 返回false, 则可能意味着函数f已经开始在新的goroutine中运行或者已经停止。

### 5. 使用未导出的空结构体(struct{})作为context key

Context(context.Context)非常适合传递请求范围的值、取消信号和截止日期（deadline），有时你可能需要从context中添加和检索值。例如：

```go
func main(){
    ctx := context.WithValue(context.Background(),"data","request-scoped data")
    
    handleRequest(ctx)
}

func handleRequest(ctx context.Context){
    fmt.Println("data:",ctx.Value("data"))
}
// Output: "data: request-scoped data"
```

这里的挑战要确保用在context中存储值的key是唯一的。

如果程序另一部分中的其他人使用相同的字符串“data”作为key，则可能出现冲突。为了避免这种情况，你可以使用空的、未导出的结构体作为key，因为每个结构体在包范围内（package scope）都是唯一的：

```go
type empty struct{}
type anotherEmpty struct{}

func main(){
    ctx := context.WithValue(context.Background(), empty{}, "request-scoped data")
    
    handleRequest(ctx)
}

func handleRequest(ctx context.Context){
    fmt.Println("data:", ctx.Value(empty{}))
    fmt.Println("data:", ctx.Value(anotherEmpty{}))
}
// Output:
// data: request-scoped data
// data: <nil>
```

基本上，使用未导出的（私有）空结构可以帮助避免与其他包的任何潜在冲突。

> "我可以使用其他类型，但基础类型仍然是string 或int吗？"

当然可以使用另外一种类型，它应该避免冲突。例如底层类型为int的number(0)和int(0)是不同的：

```go
type number int

func main(){
    ctx := context.WithValue(context.Background(), number(0), "value from number type")
	
    ctx = context.WithValue(ctx, 0, "value from int type")
    
    handleRequest(ctx)
}

func handleRequest(ctx context.Context){
    fmt.Println("data:", ctx.Value(number(0)))
    fmt.Println("data:", ctx.Value(0))
}
```

这是可行的，因为在Go中，只有当两个interface{}的类型和值都匹配时，它们的值才相等。所以，它们是不同的类型，并不相等。

- 第一个值：{type: number, value: 0}
- 第二个值：{type: int, value: 0}

它们是不同的类型，因此它们并不相等。

> “但是为什么要使用空的struct{}呢？”

空结构体不会分配任何内存，因为它没有字段，因此没有数据，它们的类型仍然可以唯一标识context 值，这将它成为key的轻量级且无冲突的选项。

当然，在某些情况下，你仍然可以使用具有基础元类型的类型定义：

*在编写业务逻辑时使用context值可能很棘手。因为它隐式传递数据，所以存在编译性安全问题，可能难以跟踪和调试。*

### 6. 处理延迟调用的错误以防止忽视错误

在延迟调用中很容易忽视错误处理，这可能让我们陷入困境：

```go
func doSomething() error{
	file, err := os.Open("file.txt")
	if err != nil{
		return err
	}
	defer file.Close()
	
	...
}
```

在这段代码中如果关闭文件失败，可能是因为某些内容未正确写入或文件系统中存在故障。如果我们不检查该错误，那么我就错过了捕获和处理潜在关键问题的机会。

如果我们现在坚持使用defer，我们基本上有三种方法来处理它：

- 我们可以将错误作为函数返回的一部分进行处理。
- 我们可以让程序panic，这可能有点严重。除非它是一个真正严重的错误，足以证明程序奔溃是合理的。
- 或者我们可以只是记录错误并继续。这很简单但意味着你没有主动处理错误，只是注意它发生了。

但是如果将其作为函数错误处理呢？这是有点差异的。我们可以使用命名返回值来巧妙管理它：

```
func DoSomething(path string)(err error){
	file, err := os.Open(path)
	if err != nil{
		return err
	}
	
	defer func(){
		if cerr := file.Cloes(); cerr != nil{
			err = errors.Join(err, cerr)
		}
	}()
	
	...
}
```

在此版本中，使用命名的返回变量err作为函数的错误返回。

在延迟函数内，检查file.Close()是否返回错误（捕获为cerr）。如果确实如此，我们使用errors.Join将其与可能已经存在的任何现有错误合并（而不是wrap）。这样，该函数可以返回反映文件打开和文件关闭操作问题的错误。

或者，代码简化一下：

```go
defer func(){
	err = errors.Join(err, file.Close())
}()
```

简化版本质上做同样的事情，但将其压缩为defer内的一行。现在，即使是这种较短的方法，也会因为匿名函数而增加一些复杂性，因为匿名函数增加了嵌套，会使代码更难理解。

实际上，还有另外一种方法可以使用简洁的辅助函数来处理延迟调用的错误：

```go
defer closeWithError(&err, file.Close)   // 注：原文这里file传入，译者改为file.Close

func closeWithError(err *error, f func() error) {
    *err = errors.Join(*err, f())
}
```

> "等等，这不会因为err为nil时*err而导致panic吗？"

这是合理的担忧，但问题在于：它实际上运行的很好。

原因如下：在Go中，error是一个interface{}，并且nil error的行为方式与nil指针其他类型（例如*int）的行为不同。

一个nil error在内部标识为{type = nil, value = nil},但准确地说他仍然是一个有效的、可用的值（interface接口的零值）。

所以当我们在defer closeWithError(&err, file.Close)调用中使用&err时，我们并不是在处理nil指针场景。我们实际上得到的是一个指向interface变量的指针，该变量本身保存着{type=nil, value=nil}.

这意味着在closeWithError函数中，当我们使用*err间接引用错误指针以分配新值时，我们不会因为间接引用nil指针（这可能导致panic）。相反，我们正在做的是通过接口interface变量的指针修改其值。这是一个安全的操作，可以避免你猜测的panic。

此解决方案的灵感来自[David Nix (@davidnix_) / X](https://x.com/davidnix_)

### 7. 始终跟踪goroutine的生命周期

Goroutines是堆栈式的，这意味着它们比其他语言中类似结构占用更多的内存，每个至少2kb，虽然很小但不可忽略。

每次使用 `go doSomething()`启动goroutine时，都会立即保留2kb内存（在Go1.2中为4kb，在Go1.4中增加到8kb）。

因此，缺点是当你的Go程序同时处理很多事情时，与没有这种堆栈分配的语言相比，它的内存使用量会增长得更快。

这个初始大小是一个起点，Go runtime(运行时) 会在执行过程中自动调整 goroutines的堆栈大小，以适应每个goroutine工作负载的内存需求。

这个过程是这样的：当goroutine的堆栈达到其当前限制时，Go runtime(运行时) 会检测到这种情况并分配更大的堆栈空间。然后，它将现有堆栈的内容复制到新的、更大的堆栈（stack），并使用这个扩展的堆栈空间继续goroutine的执行。

我个人遇到过使用一堆带有for循环和time.Sleep的goroutine的情况，如下所示：

```go
func Job(d time.Duration){
    for ;; time.Sleep(d){
        ...
    }
}
```

```go
for Job(d time.Duration){
    for{
        ...
        time.Sleep(d)
    }
}
```

这样写看起来很方便，但它有着缺点。

当我们谈论优雅地关闭你的应用程序时，正如我们在本主题**“优雅地关闭你的应用程序”**（译者注：在后文中，还未翻译）部分中概述的那样，我们遇到某些功能的棘手问题，例如`time.Sleep`，它本质上不能很好地支持优雅关闭。

- Sleep -> SIGTERM -> Running -> Interrupted
  - 睡眠 -> sigterm -> 运行 -> 终端

（译者注，此处有个超链接，是下一篇待翻译内容）

因此，对于不会自然结束的任务，例如提供网络连接或监视配置文件，最好使用取消信号（cancellation signals）或条件来明确定义这些任务何时应停止。

```go
func Job(ctx context.Context, d time.Duration){
    for{
        select{
        case <- ctx.Done():
            return
        default:
            ...
            time.Sleep(d)
        }
    }
}
```

在此设置中，context应从一个base context派生，当收到SIGTERM时，该上下文将被取消。这样，至少我们知道任务不会被意外终端，即使是在程序终止时。

但这并不能完全解决问题，如果需要随时停止goroutine怎么办呢？这在下一篇中讨论。

现在，考虑另一种情况，其中goroutine可能会永远卡住：

```go
func Worker(jobs <-chan int){
    for i := range jobs{
        ...
    }
}
jobs := make(chan int)
go worker(jobs)
```

你可能认为确认routine何时结束很简单，只需关闭jobs channel即可。

但jobs channel到底什么时候关闭呢？如果出现错误并且我们没有`close` channel 并从函数返回，则goroutine会无限期挂起，从而导致内存泄漏。

因此，明显知道goroutine何时启动和停止，并将context传递到长时间运行的进程中，这是很重要的。

### 8. 避免使用time.Sleep()，它不是context感知的并且不能被中断







> **未完待翻译，感兴趣可以先看原仓库英文版** 
