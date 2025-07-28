## 错误处理

--- 

### 仅在客户端需要时定义错误（`var ErrXXX = errors.New`）

在代码库里很常见看到大量error定义，每个定义都有详情名和冗长的描述。但这是必要的吗？

让我们来思考一下：

```go
var (
    ErrPriceTooHigh = errors.New("sale: input price is too high")
    ErrPriceTooLow  = errors.New("sale: input price is too low")
    ErrAlreadySold  = errors.New("sale: already sold")
)
```

在这些情况下，开发人员试图对所有可能的错误(error)进行严格控制，他们希望在业务逻辑中考虑所有可能出错的场景。  

但坦白说，这种方法可能有些过度，原因有以下几点：

- 这会给维护代码的人带来负担，想象一下，你必须记住或不断查阅每个错误的含义。
- 虽然现在看起来是个好主意，但过几周或几个月后，就连你自己也可能忘记为什么创建了这些错误。
- 有时，我们的客户端不需要知道这些错误。例如，如果前端已经限制了输入价格范围，为什么还要定义像`ErrPriceTooHigh`或`ErrPriceTooLow`这样的错误？一个通用的错误信息就足够了。

只有绕过前端直接访问API的人可能会遇到这些错误，而通常，这不是我们希望支持的行为。

“不过度定义错误”的原则不仅适用于客户端-服务器交互。

它同样适用于内部代码。例如，如果我们无法将消息发布到消息队列。创建 `ErrPublishMessage` 之前，考虑是否必要。是否有人需要捕获这个特定错误？

**那么，在这些情况下，推荐的处理方式是什么？**

如果我们不期望我们的客户端（无论是代码中的其他部分还是我们库的外部用户）根据不同类型的错误采取特定行动，通常最好保持简单。

让我带你了解在这些场景下可能更有效的几种策略。一种简单的方法是在出现问题时返回一个基本错误，比如这样：

```go
func Sale(price int) error {
    ...
    if isPriceHigh(price) {
        return errors.New("sale: input price is too high")
    }
}
```

这种方法非常直接。但如果你需要在错误消息中包含更多上下文信息，`fmt.Errorf` 可能是更好的选择。

它允许你使用动态数据格式化错误消息，这对于快速理解问题非常有帮助：

```go
func Sale(price int) error {
    ...
    if isPriceHigh(price) {
        return fmt.Errorf("sale: input price (%d) is too high, cap (%d)", price, cap)
    }
}
```

现在，在这种情况下你可能需要可重复使用的自定义错误，而不是定义多个错误变量：

```go
type Error struct {
    Input int
    Min   int 
    Max   int
}
```

那么，究竟在什么时候要开始定义具体的错误变量呢？ 

其实，在某些情况下这样做是完全合理的。例如，当应用程序逻辑需要根据错误类型采取不同的处理行为时：  

- 如果发生特定错误，你可能需要重新尝试操作。
- 特定错误可能触发不同的日志记录机制，对某些错误显示警告，对其他错误显示错误信息。
- 也许你需要显示一个弹出窗口，例如“资金不足，请充值更多资金”。
- ...

在这些情况下，预先定义错误变量可以帮助你更好地管理这些不同场景。

### 用 `fmt.Errorf` 明确表达错误，不要返回“光秃秃”的错误值

在 Go 语言中，不像某些语言用 `throw` 抛出异常，错误就是一个普通的值。这也就意味着我们通常是通过 `return` 把错误返回给调用方的：

```go
func doOperation() error {
    err := doSomething() 
    if err != nil {
        return err
    }

    return nil
}
```

只简单返回这样的错误，没有带有相关上下文。有时会让试图调试代码的人感到沮丧，并试图找出到底是哪里出了问题。

#### 使用 `fmt.Errorf` 和 `%w`

现在，在 Go 1.13 中，有了一种更好的方法来为错误添加更多信息，同时保持原始错误不变。这是通过使用 `fmt.Errorf` 和 `%w` 来实现的，后者会封装错误。

这允许您在提供更多上下文的同时保留底层错误：

```go
func doSomething() error {
    err := doAnotherThing()
    if err != nil {
        return fmt.Errorf("do another thing: %w", err)
    }

    return nil
}
```

> “我还是看不出有什么好处，不管怎样都只是个错误”。

好吧，让我们来看一个实际的例子，看看为什么它是好的：

```go
func getResourceHandler(http.ResponseWriter, *http.Request)
|
| -> func authorizeUser(userID, resourceID string) error
|    |
|    | -> fetchUserFromDB(userID string) (User, error)
|
| -> func getResourceFromDB(resourceID string) (Resource, error)
```

这些错误信息中哪一个能提供更多信息？

- "retrieve resource: authorization check: fetch user from db: mongo: no documents in result"
- "retrieve resource: mongo: no documents in result"

第一个直接告诉你，问题出在从数据库获取用户上。第二个错误很模糊，不清楚问题出在用户还是资源上。

通过封装错误并添加这些信息层，你不仅能更清楚地了解出错的原因，还能使用 `errors.Is()` 等工具检查特定类型的错误：

```go
func readConfig(path string) error {
    return fmt.Errorf("read config: %w", ErrNotFound)
}

func main() {
    err := readConfig("config.json")
    if errors.Is(err, ErrNotFound) {
        fmt.Println("config file not found")
    }
}
```

一定要尽可能清楚地说明错误，这是帮助我们排查错误的线索。

--- 

#### 增强错误处理： 在 Go 中的 `fmt.Errorf` 封装错误 vs `errors.Join` 连接错误

在 Go 中，封装错误是一种标准方法，当你想添加更多上下文或同时处理多个错误时尤其有用。

传统上我们经常使用 `%w` 和 `fmt.Errorf`为错误添加上下文的，就像：

```go
err := errors.New("this is an error")

return fmt.Errorf("call Func2: %w", err)
```

现在，当需要**同时**处理多个错误时，我们仍然可以和以前一样依赖于 `fmt.Errorf`。Go 1.20 发布后，`fmt.Errorf` 现在可以将多个错误封装在一起了：

```go
fmt.Errorf("%w: %w", err1, err2)
```

然而，在 Go 1.20 中，我们还得到了一个更新、更简单的工具，叫做 `errors.Join()` ：

```go
func Func2() error {
    err := Func1()
    if err != nil {
        return errors.Join(err, errors.New("error from Func2"))
    }
    return nil
}
```

但这并不意味着我们应该完全抛弃 `fmt.Errorf`。这两种方法都与 `errors.Is()` 兼容，后者可以检查特定的错误。

不过，它们的作用略有不同。

`fmt.Errorf` 可以很好地为单个错误添加更多细节，从而更清晰地说明出错的原因和位置。另一方面，`errors.Join` 是对多个错误进行分组的理想工具，它在你同时遇到多个问题，并希望在不遗漏任何细节的情况下跟踪所有错误时非常有用。

> "我还是看不出有什么区别，为什么不一直使用 `fmt.Errorf` 呢？

让我们考虑这样一种情况：我们有许多函数，每个函数都可能返回错误，我们同时在不同的 goroutines 中运行这些函数。如果其中几个函数都失败了，我们就会出现多个错误。

如果我们使用 `fmt.Errorf` 来堆叠这些错误，例如 `fmt.Errorf(“%w:%w”)`，就会产生令人困惑的信息。

这种方法暗示错误之间存在一连串的依赖关系，这并不准确，因为这些错误是同时发生的，而不是按顺序发生的。

相反，在这种情况下使用 `errors.Join` 会更合理。