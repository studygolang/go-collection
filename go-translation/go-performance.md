# 减少Golang中的内存分配

在抽象和垃圾收集内存管理模型方面，Go 介于 C 和 Python 之间，这使得能减少内存可快速处理的能力对高级语言的程序员很有吸引力。然而，天下没有免费的午餐。Go 的抽象，特别是关于分配的抽象，是有代价的。本文将展示并衡量和降低这一成本的方法。

## 测量

> 在 StackOverflow 和类似论坛上有关性能的帖子中，您经常会听到 Knuth 关于优化的引用的一些变体：“过早的优化是万恶之源。” 这种简化忽略了 原始报价(备注：因为文件要翻墙下载，这边已经下载好存放在当前目录的file目录下文件名knuth-GOTO.pdf) 中存在的一些重要上下文：
>
>毫无疑问，效率的圣杯会导致滥用。程序员浪费了大量时间来考虑或担心他们程序中非关键部分的速度，而在考虑调试和维护时，这些提高效率的尝试实际上会产生强烈的负面影响。我们应该忘记小效率，比如大约 97% 的时间：过早的优化是万恶之源。然而，我们不应该错过关键的 3% 的机会。一个优秀的程序员不会被这样的推理所迷惑，他会明智地仔细查看关键代码；但只有在识别出该代码之后。
>
>Knuth 强调，与其寻找对可读性造成严重损害以至不值得的小性能提升，我们应该寻找通过改进关键代码可以找到的大 (97%) 提升。我们如何识别关键代码？使用分析工具。幸运的是，Go 拥有比大多数语言更好的分析工具。例如，假设我们有以下（有些人为的）代码：

```golang
package main

import (
    "bytes"
    "crypto/sha256"
    "fmt"
    "math/rand"
    "strconv"
    "strings"
)

func foo(n int) string {
    var buf bytes.Buffer
    for i := 0; i < 100000; i++ {
        buf.WriteString(strconv.Itoa(n))
    }
    sum := sha256.Sum256(buf.Bytes())

    var b []byte
    for i := 0; i < int(sum[0]); i++ {
        x := sum[(i*7+1)%len(sum)] ^ sum[(i*5+3)%len(sum)]
        c := strings.Repeat("abcdefghijklmnopqrstuvwxyz", 10)[x]
        b = append(b, c)
    }
    return string(b)
}

func main() {
    // ensure function output is accurate
    if foo(12345) == "aajmtxaattdzsxnukawxwhmfotnm" {
        fmt.Println("Test PASS")
    } else {
        fmt.Println("Test FAIL")
    }

    for i := 0; i < 100; i++ {
        foo(rand.Int())
    }
}
```

要添加 CPU 分析，我们只需要将其更改为

```golang
package main

import (
    "bytes"
    "crypto/sha256"
    "fmt"
    "math/rand"
    "os"
    "runtime/pprof"
    "strconv"
    "strings"
)

func foo(n int) string {
    var buf bytes.Buffer
    for i := 0; i < 100000; i++ {
        buf.WriteString(strconv.Itoa(n))
    }
    sum := sha256.Sum256(buf.Bytes())

    var b []byte
    for i := 0; i < int(sum[0]); i++ {
        x := sum[(i*7+1)%len(sum)] ^ sum[(i*5+3)%len(sum)]
        c := strings.Repeat("abcdefghijklmnopqrstuvwxyz", 10)[x]
        b = append(b, c)
    }
    return string(b)
}

func main() {
    cpufile, err := os.Create("cpu.pprof")
    if err != nil {
        panic(err)
    }
    err = pprof.StartCPUProfile(cpufile)
    if err != nil {
        panic(err)
    }
    defer cpufile.Close()
    defer pprof.StopCPUProfile()

    // ensure function output is accurate
    if foo(12345) == "aajmtxaattdzsxnukawxwhmfotnm" {
        fmt.Println("Test PASS")
    } else {
        fmt.Println("Test FAIL")
    }

    for i := 0; i < 100; i++ {
        foo(rand.Int())
    }
}
```

编译并运行此程序后，配置文件将写入`./cpu.pprof`. 我们可以使用`go tool pprof`以下命令读取此文件：

```
$ go tool pprof cpu.pprof 
```

我们现在在 pprof 交互工具中。我们可以通过`top10`( `top1`, `top2`, `top99`, ...,`topn`也可以工作)看到我们的程序大部分时间都花在什么上面。`top10`给我们：

```
Showing nodes accounting for 2010ms, 86.27% of 2330ms total
Dropped 48 nodes (cum <= 11.65ms)
Showing top 10 nodes out of 54
     flat  flat%   sum%       cum   cum%
     930ms 39.91% 39.91%      930ms 39.91%  crypto/sha256.block
     360ms 15.45% 55.36%      720ms 30.90%  strconv.formatBits
     180ms  7.73% 63.09%      390ms 16.74%  runtime.mallocgc
     170ms  7.30% 70.39%      170ms  7.30%  runtime.memmove
     100ms  4.29% 74.68%      100ms  4.29%  runtime.memclrNoHeapPointers
      80ms  3.43% 78.11%      340ms 14.59%  bytes.(*Buffer).WriteString
      60ms  2.58% 80.69%       60ms  2.58%  runtime.nextFreeFast (inline)
      50ms  2.15% 82.83%      360ms 15.45%  runtime.slicebytetostring
      40ms  1.72% 84.55%     2070ms 88.84%  main.foo
      40ms  1.72% 86.27%      760ms 32.62%  strconv.FormatInt
```

[ 注意：本文中使用“分配”来指代 [堆分配 ](https://www.sciencedirect.com/topics/computer-science/heap-allocation) 。堆栈分配也是分配，但在性能方面，它们几乎没有那么昂贵或重要。]

看起来我们在使用 sha256、strconv、分配和垃圾收集方面花费了大量时间。现在我们知道我们需要改进什么。由于我们没有进行任何类型的复杂计算（可能除了 sha256），我们的大多数性能问题似乎都是由堆分配引起的。我们可以通过替换来精确地测量分配

```golang
for i := 0; i < 100; i++ {
    foo(rand.Int())
}
```

和

```golang
fmt.Println("Allocs:", int(testing.AllocsPerRun(100, func() {
    foo(rand.Int())
})))
```

并导入`testing`包。现在，当我们运行程序时，我们应该看到如下输出：

```
Test PASS
Allocs: 100158
```

几个月前，我的 [雇主](https://github.com/lukechampine) 向我提出了将这个数字变为 0 的挑战。起初这似乎非常令人生畏，因为 100k 是很多分配。在本文中，我将揭示我在减少此代码中的分配数量时的思考过程和想法。

## 帕累托原则

观察到分配的数量大约是 100,000。我们可以从代码中回忆

```golang
for i := 0; i < 100000; i++ {
    buf.WriteString(strconv.Itoa(n))
}
```

我们将整数参数`n`作为字符串写入缓冲区 100,000 次。基准测试还表明这`strconv.formatBits`也需要花费大量时间。但是`strconv.Itoa(n)`每次迭代的值都不会改变。我们可以简单地计算一次：

```golang
x := strconv.Itoa(n)
for i := 0; i < 100000; i++ {
    buf.WriteString(x)
}
```

这使我们的分配显着下降：

```
Test PASS
Allocs: 159
```

寻找这样的机会应该是优化时的重中之重。我们已经去掉了大部分分配，但是对于这样一个简单的函数来说，159 次分配仍然很多。此外，如果我们查看 CPU 分析结果，我们仅将运行时间缩短了约 50%。如果我们能勉强维持剩余的分配，我们的运行时间可能小于 1s（从 ~2.3s 开始，目前为 ~1.2s，至少在我的系统上）。

## 循序渐进

我们已经取得了很多进展，但我们还有一些路要走。我们继续`foo`从上到下看。

```golang
sum := sha256.Sum256(buf.Bytes())
```

如果我们查看 [sha256](https://golang.org/pkg/crypto/sha256/) 包，Sum256 似乎是某种快捷功能。但是如果我们看一下 [实现](https://golang.org/src/crypto/sha256/sha256.go?s=5634:5669#L244) ：

```golang
func Sum256(data []byte) [Size]byte {
    var d digest
    d.Reset()
    d.Write(data)
    return d.checkSum()
}
```

如果我们要使用该`New`函数创建一个新的哈希对象，似乎无论如何我们都必须这样做。如果有一种方法可以重用消化对象，也许这可能很有用（剧透：有），但现在可能有更多低悬的果实可用。

如果我们继续循环，

```golang
var b []byte
for i := 0; i < int(sum[0]); i++ {
    x := sum[(i*7+1)%len(sum)] ^ sum[(i*5+3)%len(sum)]
    c := strings.Repeat("abcdefghijklmnopqrstuvwxyz", 10)[x]
    b = append(b, c)
}
```

我们可以看到分配可能发生在`b`附加到缓冲区并增长和`strings.Repeat`调用时，因为这可能涉及某种复制。行分配`x`绝对不是分配，因为它只是从数组中读取单个整数并对它们执行一些计算。让我们看看我们是否可以对`strings.Repeat`. 好吧，我们正在重复一个常量，所以我们可以展开重复，例如：

```
    c := "abcdefghijklmnopqrstuvwxyzabcdefghijklmnopqrstuvwxyzabcdefghijklmnopqrstuvwxyzabcdefghijklmnopqrstuvwxyzabcdefghijklmnopqrstuvwxyzabcdefghijklmnopqrstuvwxyzabcdefghijklmnopqrstuvwxyzabcdefghijklmnopqrstuvwxyzabcdefghijklmnopqrstuvwxyzabcdefghijklmnopqrstuvwxyz"[x]
```

但这感觉有点hacky。如果有更基本的方法会怎样？假设x等于 53。如果我们找到字符串的第 53 个字符，我们会看到它是 'a'。第 54 位是“b”。第 79、105、(1+26k)th 也是“a”。注意到这里的模式了吗？由于字母表中有 26 个字符，因此我们可以将x26 乘以得到相同的结果

```golang
    c := "abcdefghijklmnopqrstuvwxyz"[x%26]
```

现在我们减少到 23 个分配。

```
Test PASS
Allocs: 23
```

让我们回顾一下字节切片`b`。它被创建为空，然后重复附加。因为必须在需要时分配新空间，所以这会产生比最初仅分配一次更多的分配。这可以通过以下方式完成：

```
b := make([]byte, 0, int(sum[0]))
for i := 0; i < int(sum[0]); i++ {
    x := sum[(i*7+1)%len(sum)] ^ sum[(i*5+3)%len(sum)]
    c := "abcdefghijklmnopqrstuvwxyz"[x%26]
    b = append(b, c)
}
```

我们减少到 19 个分配。

现在剩下的就是`return string(b)`声明了。由于这会将字节切片复制到字符串中，因此它会进行分配。我们可以使用一个技巧来防止这种分配：

```golang
return *(*string)(unsafe.Pointer(&b))
```

这带来了 18 个分配。

## 再看函数

现在我们已经完成了对函数的初始传递，让我们再次查看它，看看是否还有任何其他的分配我们可以摆脱。

```
var buf bytes.Buffer
x := strconv.Itoa(n)
for i := 0; i < 100000; i++ {
    buf.WriteString(x)
}
```

看起来我们可以创建一个字节切片并附加到它上面，而不是创建一个 bytes.Buffer 。这样我们就分配了一次固定数量，而不必处理我们之前提到的重新分配。

所以我们可以将其替换为：

```golang
x := strconv.Itoa(n)
buf := make([]byte, 0, 100000*len(x))
for i := 0; i < 100000; i++ {
    buf = append(buf, x...)
}
sum := sha256.Sum256(buf)
```

现在只有 3 个分配。如果我不得不猜测，这些将是 的创建`buf`、 的创建`b`和`sha256.Sum256`. 那么我们如何摆脱这些更多（看似难以处理）的分配呢？欢迎`sync.Pool`。

##`sync.Pool`<BR>
[注意：以下部分与 Knuth 可能会描述为过度的内容相近。它主要是为了展示sync.Pool。然而，值得注意的是，虽然 3 次分配和 0 次分配之间的差异可能看起来不大，但实际上可能是因为这意味着最终不必调用垃圾收集器（至少作为此函数的结果） .]

`sync.Pool`是 Go 中的一项功能，可让您分摊分配成本。使用我们当前的代码，当我们分配缓冲区时，我们会在`foo`函数的生命周期内这样做。但是，如果我们能够一次分配它，将它存储在一个全局变量中，然后无限地重用它（每次重置内容同时保持容量）呢？手动执行此操作会有点复杂，并且不能同时工作。这`sync.Pool`就是为了。`sync.Pool`允许您`Get`分配一个已分配的对象，`Put`并在您完成后返回。如果池中没有可用的对象并且您请求一个，它将调用`New`您定义的函数来分配一个。让我们为哈希对象创建一个池，为哈希总和（256 字节切片）创建一个池。把下面的放在上面`foo`：

```goland
var bufPool = sync.Pool{
    New: func() interface{} {
        // length of a sha256 hash
        b := make([]byte, 256)
        return &b
    },
}

var hashPool = sync.Pool{
    New: func() interface{} {
        return sha256.New()
    },
}
```

现在我们需要更新我们的代码以使用这些而不是我们在函数中创建的那些。将以下内容添加到 的顶部`foo`：

```goland
// get buffer from pool
bufptr := bufPool.Get().(*[]byte)
defer bufPool.Put(bufptr)
buf := *bufptr
// reset buf
buf = buf[:0]

// get hash object from pool
h := hashPool.Get().(hash.Hash)
defer hashPool.Put(h)
h.Reset()
```

我们用来`sync.Pool.Get`从池中检索一个对象，我们（当函数返回时）将它放回池中`sync.Pool.Put`。由于我们现在有一个 hasher 对象，我们可以直接写入它而不是中间缓冲区。理想情况下，我们可以做类似的事情

```goland
x := strconv.Itoa(n)
for i := 0; i < 100000; i++ {
    h.Write(x)
}
```

不幸的是，Go 中的哈希对象没有 WriteString 方法，因此我们需要使用它`strconv.AppendInt`来获取字节切片。此外，使用 AppendInt 为我们节省了分配，因为它会写入我们从中获得的缓冲区，`bufPool`而不是分配一个新的字符串，例如`strconv.Itoa`

```goland
x := strconv.AppendInt(buf, int64(n), 10)
for i := 0; i < 100000; i++ {
    h.Write(x)
}
```

现在我们可以获取哈希并将其放入`buf`：

```goland
// reset whatever strconv.AppendInt put in the buf
buf = buf[:0]
sum := h.Sum(buf)
```

在下一个 for 循环中，我们从 0 迭代到`sum[0]`，执行一些计算，并将结果放入`b`。由于`sum[0]`永远不会超过 256，因为 a 的`byte`范围是 0-255，我们可以简单地说

```goland
b := make([]byte, 0, 256)
```

并保持循环内容相同。编译器可以在编译时用 256 推理，但不能用`sum[0]`. 这为我们节省了另一个分配。<BR>

我们终于减少到 1 个分配。将最终的 return 语句替换为以下内容：<BR>

```goland
sum = sum[:0] // reset sum
sum = append(sum, b...)
return *(*string)(unsafe.Pointer(&sum))
```

你应该看到

```
Test PASS
Allocs: 0
```

为什么这样做？好吧，使用详细的逃逸分析信息构建我们最初拥有的内容可能会有所帮助：

```goland
$ go build -gcflags='-m -m' a.go

...
./a.go:55:11: make([]byte, 0, 256) escapes to heap:
./a.go:55:11:   flow: b = &{storage for make([]byte, 0, 256)}:
./a.go:55:11:     from make([]byte, 0, 256) (spill) at ./a.go:55:11
./a.go:55:11:     from b := make([]byte, 0, 256) (assign) at ./a.go:55:4
./a.go:55:11:   flow: ~r1 = b:
./a.go:55:11:     from &b (address-of) at ./a.go:65:35
./a.go:55:11:     from *(*string)(unsafe.Pointer(&b)) (indirection) at ./a.go:65:9
./a.go:55:11:     from return *(*string)(unsafe.Pointer(&b)) (return) at ./a.go:65:2
./a.go:55:11: make([]byte, 0, 256) escapes to heap
...
```

新代码的输出：

```
...
./a.go:55:11: make([]byte, 0, 256) does not escape
...
```

如果我们将切片保留在函数中并且不返回它，它将被堆栈分配。任何 C 程序员都知道，不可能返回堆栈分配的内存。当 Go 编译器看到 return 语句时，它会改为分配切片堆。但这并不能真正解释为什么将内容传输到`sum`不分配。`sum`堆不是也分配了吗？是的，但`sync.Pool`已经为我们执行了堆分配。

## 外卖/小贴士

* `unsafe` 转换字符串/字节切片的技巧
   * 字符串 -> 字节切片： `*(*[]byte)(unsafe.Pointer(&s))`
       * 缓冲区不能修改，否则会panic！
   * 字节切片 -> 字符串： `*(*string)(unsafe.Pointer(&buf))`
* 如果不依赖于循环中更新的状态，则不要在循环中运行分配繁重的代码       
* 使用`sync.Pool`大分配的对象时
   * 对于字节切片，[bytebufferpool](https://github.com/valyala/bytebufferpool) 可能更容易实现且性能更高
* 如果可以，请重用缓冲区
   * 重置缓冲区
       * `bytes.Buffer.Reset`
       * `buf = buf[:0]`
* 更喜欢最初分配固定大小的数组，而不是不指定大小并将其留给 [调整大小因子](https://forum.golangbridge.org/t/slice-append-what-is-the-actual-resize-factor/15654)
* 确保实际 [对](https://golang.org/pkg/testing/#hdr-Benchmarks) 您的代码进行 [基准测试](https://chris124567.github.io/2021-06-21-go-performance/) ，以查看更改是否可以提高性能/权衡可读性问题
          
## 结论

性能确实很重要，而且并不总是很难改进。虽然这段代码有点人为，但可以从提高其性能中吸取一些教训。

## 原文链接

https://chris124567.github.io/2021-06-21-go-performance/
