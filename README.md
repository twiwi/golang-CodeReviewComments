# Go Code Review Comments

参考 [Effective Go](https://golang.org/doc/effective_go.html ). <br>

* [Gofmt](#gofmt)
* [Comment Sentences](#comment-sentences)
* [Contexts](#contexts)
* [Copying](#copying)
* [Crypto Rand](#crypto-rand)
* [Declaring Empty Slices](#declaring-empty-slices)
* [Doc Comments](#doc-comments)
* [Don't Panic](#dont-panic)
* [Error Strings](#error-strings)
* [Examples](#examples)
* [Goroutine Lifetimes](#goroutine-lifetimes)
* [Handle Errors](#handle-errors)
* [Imports](#imports)
* [Import Dot](#import-dot)
* [In-Band Errors](#in-band-errors)
* [Indent Error Flow](#indent-error-flow)
* [Initialisms](#initialisms)
* [Interfaces](#interfaces)
* [Line Length](#line-length)
* [Mixed Caps](#mixed-caps)
* [Named Result Parameters](#named-result-parameters)
* [Naked Returns](#naked-returns)
* [Package Comments](#package-comments)
* [Package Names](#package-names)
* [Pass Values](#pass-values)
* [Receiver Names](#receiver-names)
* [Receiver Type](#receiver-type)
* [Synchronous Functions](#synchronous-functions)
* [Useful Test Failures](#useful-test-failures)
* [Variable Names](#variable-names)

## Gofmt

  你可以在你的代码中运行 [Gofmt](https://golang.org/cmd/gofmt/) 以自动修复大多数代码格式问题。几乎所有的代码都使用 `gofmt`.<br>
  另一种方法是使用[goimports](https://godoc.org/golang.org/x/tools/cmd/goimports)，它是 `gofmt`的超集，可根据需要添加（删除）行。
  
## Comment Sentences

  参考[https://golang.org/doc/effective_go.html#commentary] 。注释的句子应当具有完整性，即使会显得啰嗦多余。但这种方法使得它们在提取到godoc文档时能保持良好的格式。注释应当以描述的事物名称开头，以句点结束。<br>
  参考以下格式<br>

```go
// Request represents a request to run a command.
type Request struct { ...

// Encode writes the JSON encoding of req to w.
func Encode(w io.Writer, req *Request) { ...
```

请注意，除了句点(.)外，还有很多符号可以作为替代，比如( ? , !)。除此之外，还有许多工具使用注释来标记类型和方法，(如 easyjson:json 和 golint 的MATCH)。这使得这条规则难以形式化


## Contexts
`context.Context`类型的值囊括了跨API和进程边界的安全凭证，跟踪信息，Context应当结束的时间和取消信号。
大多数使用Context的函数应该将context.Context作为其第一个参数:

```go
func F(ctx context.Context, /* other arguments */) {}
```

从不做特定请求的函数可以使用 context.Background()，但即使您认为不需要，也可以在传递Context时使用错误(err)。如果您有充分的理由认为替代方案是错误的，那么只能直接使用context.Background()

不要将Context成员添加到一个结构体类型；相而是将ctx作为函数参数添加到需要传递它的该类型的每一个方法上。不过也有例外，函数签名(method signature)必须与标准库或者第三方库中的接口相匹配。

不要在函数签名中创建Context类型或者使用Context以外的接口。

如果要传递应用程序的数据，请将其放在参数中，接收方如果？？？？？

Contexts 是不可变的，因此可以将同一个Context传递给共享结束时间、取消信号、安全凭证和父进程追踪等信息的多个调用。


## Copying

为了避免意外的别名(alias)，从另一个包复制结构时要小心。比如， `bytes.Buffer` 类型包含了 `[]byte` 切片类型，并且作为小字符串的优化，可被较小的字节数组引用。如果你拷贝了一个`Buffer`，拷贝中的切片可能会alias原始数组中的切片，从而导致后续的操作带来令人惊讶的结果。

通常来说，如果一个类型 `T`其方法与指针结构相关，那么请不要拷贝 `T`的值


## Declaring Empty Slices

当声明一个空切片时， 使用

```go
var t []string
```

而不是

```go
t := []string{}
```

前者声明了一个指向nil的切片，后者声明了一个 non-nil 且 len = 0的切片。虽然他们的功能是等价的：它们的 `len` 和`cap` 都是0，但nil切片应当作为首选方案。

请注意在有些情况下， non-nil 切片表现更好，比如当解码JSON对象时，nil切片解码为 `null`, non-nil 切片解码为 JSON 数组

在设计接口时，应当避免因nil切片和non-nil切片带来的不同表现，因为这可能会导致细微的编码错误。

有关Go中nil的更多讨论，请参阅 Francesc Campoy 的演讲 [Understanding Nil](https://www.youtube.com/watch?v=ynoY2xz-F8s).


## Crypto Rand

请不要使用包 `math/rand`来生成密钥，即使是一次性的。如果不提供种子，则密钥完全可以被预测到。用`time.Nanoseconds()`作为种子，there was just a few bits of entropy.
相反，使用`crypto/rand`'s Reader作为代替。并且如果你需要生成文本，打印成16进制或者base64类型即可。

``` go
import (
    "crypto/rand"
    // "encoding/base64"
    // "encoding/hex"
    "fmt"
)

func Key() string {
    buf := make([]byte, 16)
    _, err := rand.Read(buf)
    if err != nil {
        panic(err)  // out of randomness, should never happen
    }
    return fmt.Sprintf("%x", buf)
    // or hex.EncodeToString(buf)
    // or base64.StdEncoding.EncodeToString(buf)
}
```



## Doc Comments

所有顶级的、导出的名称都应当有文档注释，并且未导出的平凡函数和声明也应该具备注释。参考https://golang.org/doc/effective_go.html#commentary 获取更多关于注释的约定。

## Don't Panic

不要使用`panic`作为日常错误处理机制，请使用`error`或者多返回值。

参阅 https://golang.org/doc/effective_go.html#errors. 

## Error Strings

错误信息的字符串不应该大写（除非以专有名词或首字母缩略词开头）或以标点符号结尾，因为它们通常是在其他上下文后打印的。也就是说，使用  `fmt.Errorf("something bad")` 而不是 `fmt.Errorf("Something bad")` 。这样的话 `log.Printf("Reading %s: %v", filename, err)` 就不会在字符串中间有大写字母。但这种格式不适合日志记录，？？？？？

## Examples

添加新的包时，请包含预期用法的示例。可以是一个可运行的例子或者演示完整调用的简单测试
了解更多有关 [可测试示例函数](https://blog.golang.org/examples)


## Goroutine Lifetimes

当你需要使用goruntines时，请确保它们什么时候/什么条件下退出。

Goroutines可以会因阻塞channel的send或者receives而泄露,即使被阻塞的通道无法访问，垃圾收集器也不会终止goroutine。

即使gorountine没有泄露，当不再需要它们时将它们继续运行会导致其他微妙且难以诊断的问题。Sends on closed channels panic.即使在不需要结果后，修改正在使用的数据也会导致数据竞争，并且将gorountine 置于 in-flight 状态任意时常也会导致不可预测的内存消耗。

尽量保证并发代码足够简单，使gorountine的生命周期更明显。如果难以做到，请记录gorountines退出的时间和原因。

## Handle Errors

请不要使用`_`丢弃错误。如果一个函数返回一个错误，检查并确保函数运行成功。处理错误，返回错误，必要情况下请panic。

参考 https://golang.org/doc/effective_go.html#errors.

## Imports

尽量避免导入包时的重命名，以避免名称冲突。优秀的包名称不应该要求重命名。如果发生名称冲突，尽量重命名本地或项目特定的包。

导入包按名称分组，用空行隔开。
标准库包应始终位于第一组。

```go
package main

import (
	"fmt"
	"hash/adler32"
	"os"

	"appengine/foo"
	"appengine/user"

        "github.com/foo/bar"
	"rsc.io/goversion/version"
)
```

可以使用 <a href="https://godoc.org/golang.org/x/tools/cmd/goimports">goimports</a> 来规范这些。

## Import Dot

使用 `import .` 导入包在测试中很有用，但由于循环依赖性，它不能成为被测试包的一部分：

```go
package foo_test

import (
	"bar/testutil" // also imports "foo"
	. "foo"
)
```
在这种情况下，测试文件不能在包`foo`中，因为它使用了同样导入`foo`的包 `bar/testutil`，所以我们使用`import .`格式导入来让测试文件伪装成包foo的一部分，即使它并不是。除了这一情况，不要在你的程序中使用`import .`来导入包。它将使程序更难阅读，因为不清楚 Quux 这样的名称是在当前包中还是导入包中。

## In-Band Errors

在C语言和类似语言中，函数常常返回 -1 或 null 值表示错误或者结果缺失。

```go
// Lookup returns the value for key or "" if there is no mapping for key.
func Lookup(key string) string

// Failing to check a for an in-band error value can lead to bugs:
Parse(Lookup(key))  // returns "parse failure for value" instead of "no value for key"
```

Go语言对多个返回值的支持提供了更好的解决方案。
相比要求客户端检查一个带错误情况的值（比如result = -1 时应当是执行错误了，需要特别检查）,函数应该返回一个附加值以指示其他返回值是否有效。此返回值可能是一个error，或者在不需要解释时可以是bool，并且应当位于返回值的最后。

``` go
// Lookup returns the value for key or ok=false if there is no mapping for key.
func Lookup(key string) (value string, ok bool)
```
这还可以防止调用者错误地使用结果:

``` go
Parse(Lookup(key))  // compile-time error
```

并且鼓励更强大和可读性更强的代码:

``` go
value, ok := Lookup(key)
if !ok  {
    return fmt.Errorf("no value for %q", key)
}
return Parse(value)
```

此规则适用于导出的函数，但对未导出的函数也很有用。

返回值如 nil， ""， 0 和 -1 都可以作为函数的有效返回值，即调用者不需要针对结果的不同做区分处理。

一些标准库函数，比如包 "strings" 中的函数，返回值都会附带一个err.这极大地简化了花在字符串上的操作，代价是需要程序员花更多的功夫来处理err。
总而言之，通常情况下，Go代码应额外返回一个error值.

## Indent Error Flow

尝试将代码路径保持在最小的缩进处，并缩进错误处理，第一个处理错误信息。这通常可以保证快速地检查正常路径来提高代码的可阅读性。

例如，不要写成：

```go
if err != nil {
	// error handling
} else {
	// normal code
}
```

而应该写成：

```go
if err != nil {
	// error handling
	return // or continue, etc.
}
// normal code
```

如果`if`语句有初始化语句，比如：

```go
if x, err := f(); err != nil {
	// error handling
	return
} else {
	// use x
}
```
那么这可能需要将变量的声明提到它自己的一行：

```go
x, err := f()
if err != nil {
	// error handling
	return
}
// use x
```

## Initialisms

名称中的单词如果是首字母或首字母缩略词（例如 "URL" 或 "NATO")，应当具有一致的大小写. 例如，"URL" 应写为 "URL" 或"url"，而不是 "Url"。

举个例子： ServerHTTP与ServerHttp，应当使用前者。
对于具有多个缩略单词的标识符，使用例如 "xmlHTTPRequest" 或 "XMLHTTPRequest"。

当"ID"是标识符的缩写时，此规则也同样适用，因此请写成 "appID" 而不是 "appId"。

protocol buffer compiler 产生的代码不受此规则的约束。人工编写的代码理应比机器编写的代码保持更高的标准。


## Interfaces

Go语言接口类型通常属于包中的interface类型，而不是实现这些值的包。实现了接口的包应该返回具体的(通常是指针或者结构)类型：这样，新方法可以不需要大量重构就可以添加到实现中。

不要“为了模仿”而在API的实现端定义接口。
相反，设计的API应当能够使用公共的API来进行测试。

并且在使用之前不要定义接口：没有真实的例子之前，很难看出接口是否必需，更不要说它应该包含哪些方法。

``` go
package consumer  // consumer.go

type Thinger interface { Thing() bool }

func Foo(t Thinger) string { … }
```

``` go
package consumer // consumer_test.go

type fakeThinger struct{ … }
func (t fakeThinger) Thing() bool { … }
…
if Foo(fakeThinger{…}) == "x" { … }
```

``` go
// DO NOT DO IT!!!
package producer

type Thinger interface { Thing() bool }

type defaultThinger struct{ … }
func (t defaultThinger) Thing() bool { … }

func NewThinger() Thinger { return defaultThinger{ … } }
```

取而代之的，可以直接使用原先的API接口，返回一个具体类型，让消费者模拟生产者的实现。

``` go
package producer

type Thinger struct{ … }
func (t Thinger) Thing() bool { … }

func NewThinger() Thinger { return Thinger{ … } }
```


## Line Length

Go代码中没有严格限制行的长度，但请避免使用不舒服的长行。同样，当一行长代码可读性还行时，不要添加换行符来增加行数。

较长的行似乎与较长的名称有关，所以请尽量避免长的名称。
实际上，这与关于函数应该有多长的建议完全相同。没有规则“函数不应该超过N行”，但肯定会有这么长的一个函数，并且这个函数也可以分解为多个小函数来实现相同的功能。

## Mixed Caps

见 https://golang.org/doc/effective_go.html#mixed-caps. 即使它打破了其他语言的惯例，这也适用。例如，未导出的常量名为`maxLength` 而不是`MaxLength` 或 `MAX_LENGTH`.

也可参考 [Initialisms](https://github.com/golang/go/wiki/CodeReviewComments#initialisms).

## Named Result Parameters

考录下结果参数命名为这种情况时，godoc会是什么样子

```go
func (n *Node) Parent1() (node *Node)
func (n *Node) Parent2() (node *Node, err error)
```

godoc会念起来更拗口，推荐：

```go
func (n *Node) Parent1() *Node
func (n *Node) Parent2() (*Node, error)
```

另一方面，如果函数返回两个或三个相同类型的参数，或者如果从上下文中看不出结果的意义，添加名称可能会很有用。但请不要仅仅为了免于在函数中声明变量而直接命名返回结果参数。

总之，不要为了降低实施复杂度，而造成API难以阅读，比如：

```go
func (f *Foo) Location() (float64, float64, error)
```
更优的解决方法：

```go
// Location returns f's latitude and longitude.
// Negative values mean south and west, respectively.
func (f *Foo) Location() (lat, long float64, err error)
```

结论： 仅仅为了用返回参数的名称来直接返回而命名返回参数是不值得的，文档的清晰度比在函数中节约一两行代码更加重要。

最后，在某些情况下，如果您需要命名结果参数才能 change it in a deferred closure的话，尽管放手干吧。


## Naked Returns

见 [Named Result Parameters](#named-result-parameters).

## Package Comments

与godoc呈现的所有注释一样，包的注释必须出现在package子句的旁边，且没有空行：

```go
// Package math provides basic constants and mathematical functions.
package math
```

```go
/*
Package template implements data-driven templates for generating textual
output such as HTML.
....
*/
package template
```

对于main包的注释，很多种注释格式都是可以接受的，比如在目录`seedgen`中的 `package main`包，注释可以这下写:

``` go
// Binary seedgen ...
package main
```
或
```go
// Command seedgen ...
package main
```
or
```go
// Program seedgen ...
package main
```
或
```go
// The seedgen command ...
package main
```
或
```go
// The seedgen program ...
package main
```
或
```go
// Seedgen ..
package main
```
请注意，以小写字母开头的句子不在包注释的可接受选项中，因为包是公开可见的，应当用合适的英语格式写成，包括将第一个词首字母大写。

[获取更多有关包注释的规范](https://golang.org/doc/effective_go.html#commentary)

## Package Names

包中名称的所有引用都将会使用到包名，所以你可以省略包中名称的标识符。例如，如果你在包`chubby`中有名称`ChubbyFile`,用户使用时会写成`chubby.ChubbyFile`，可以精简为`File`，用户即可写成`chubby.File`。

请注意避免使用无意义的包名称，如 util,common,misc,api,types和interfaces。

参考：

http://golang.org/doc/effective_go.html#package-names 

http://blog.golang.org/package-names 

## Pass Values

不要仅仅为了节省几个字节而传指针给函数。如果函数仅仅将其参数"x"作为`*x`使用，那么参数就不应该是指针。

常见的传指针的情况有：传递一个string的指针、指向接口值(`*io.Reader`)的指针;这两种情况下，值本身都是固定的，可以直接传递。

对于大型数据结构，或者是小型的可能增长的结构，请考虑传递指针。

更多情况请见 [Receiver Type](#receiver-type)

## Receiver Names

方法receiver的名称应该反映其身份；通常，其类型的一个或两个字母缩写就足够了(例如"client"的"c"或"cl").请不要使用通用名称如"me", "this" 或 "self"，这是面向对象语言的典型标识符，它们更强调方法而不是功能。方法receiver的名称不必像方法参数那样具有描述性，因为它的作用是显而易见的。它可以非常简洁，因为它几乎出现在每个方法调用上。请始终保持名称不变，如果你在一个地方命名receiver为 "c"，那么请不要在另一处叫其"cl"

## Receiver Type

选择在方法上使用值还是指针作为receiver可能很困难，尤其是对于新的Go程序员。如果您不知道怎么决定，请使用指针。但有时候receiver为值也挺有用，这种通常是出于效率的原因，且值为小的不变结构或基本类型的值。

一些建议：
  * 如果receiver为map，func或者chan，请不要使用指向它们的指针。如果receiver为切片并且该方法不会重新分配切片，则请不要使用指针。
  * 如果方法需要改变receiver，则必须使用指针。
  * 如果receiver包含了sync.Mutex 或类似同步字段的结构，则receiver必须是指针以避免被拷贝。
  * 如果receiver是一个大的数组或结构体，指针将更加高效。
  * 如果需要在方法中改变receiver的值，则必须使用指针
  * 如果receiver是结构体，数组或切片这种成员是一个指向可变内容的指针时，选择指针receiver更为合适，这将使读者更容易明白方法意图。
  * 如果receiver是一个小数组或结构体，没有可变的字段和指针，或者是一个简单的基本类型，int或者string，此时使用值reveiver更有道理。这可以减少垃圾得产生，如果将值传递给值方法，则可以使用堆栈上的副本而不是在堆上重新分配(尽管编译器试图避免这种情况，但它并不能每次都成功)。不要因为这一点就不加考虑地选择值receiver
  * 最后，如果不清楚该怎么用，请使用指针

## Synchronous Functions

相比异步函数，应当首选同步函数 —— 直接返回结果或在返回之前完成任何回调或通道操作的函数。

同步函数使得gorountine在调用中本地化，更容易推断其生命周期并避免泄露和数据竞争。它们也更容易测试:调用者可以传递输入并检查输出，而无需轮询或同步。

如果调用者需要更多的并发，他们可以简单地在单独的gorountine中调用函数来实现。但在调用者端删除不必要的并发是非常困难甚至是不可能的。

## Useful Test Failures

测试失败时应当返回有效的错误信息，说明错误在哪，输入是什么，输出时什么，期望输出是什么。编写一堆断言函数是个不错的选择，但请确保你的函数产生游泳的错误信息。
假定测试人员不是你和你的团队。一个典型的测试失败形如：

```go
if got != tt.want {
	t.Errorf("Foo(%q) = %d; want %d", tt.in, got, tt.want) // or Fatalf, if test can't test anything more past this point
}
```

请注意此处的命令是 `实际结果 != 预期结果`.一些测试框架鼓励程序员编写： 0 != x, "expected 0, got x",这种代码，g语言并不推荐.

如果这样看起来很累赘，你可以写一个 [[table-driven test|TableDrivenTests]].

Another common technique to disambiguate failing tests when using a test helper with different input is to wrap each caller with a different TestFoo function, so the test fails with that name:

```go
func TestSingleValue(t *testing.T) { testHelper(t, []int{80}) }
func TestNoValues(t *testing.T)    { testHelper(t, []int{}) }
```

在任何情况下，您有责任提供有用的错误信息，一边将来调试您的代码

## Variable Names

Go中变量名应尽量短。对于局部变量尤其如此，首选`c`作为`lineCount`,`i`作为`sliceIndex`。

基本规则：名称使用的地方距其声明的地方越远，则名称必须越具描述性。对一个方法receiver，一个或两个字母就足够了。循环索引或读取之类的常见变量可以使单个字母(`i r`)，更多不常见的事物和全局变量需要更具描述性的名称。

