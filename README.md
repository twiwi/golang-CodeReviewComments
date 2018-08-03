# Go编码规范

本文参考自
     [Effective Go](https://golang.org/doc/effective_go.html ).
     [Golang Code Review](https://github.com/golang/go/wiki/CodeReviewComments#initialisms)
     如有侵权，请联系我删除。

* [Gofmt](#gofmt)
* [注释语句](#注释语句)
* [Contexts](#contexts)
* [Goroutine 生命周期](#goroutine-生命周期)
* [拷贝](#拷贝)
* [使用crypto/rand生成随机值](#使用crypto/rand生成随机值)
* [空切片](#空切片)
* [Panic](#panic)
* [错误](#错误)
* [用法示例](#用法示例)
* [Imports](#imports)
* [接口](#接口)
* [单行代码长度](#单行代码长度)
* [名称](#名称)
* [传值](#传值)
* [Receiver](#receiver)
* [同步函数](#同步函数)
* [有效测试信息](#有效测试信息)
* [函数](#函数)

## Gofmt

  你可以在你的代码中运行 [Gofmt](https://golang.org/cmd/gofmt/) 以解决大多数代码格式问题。几乎所有的代码都使用 `gofmt`。如果使用[liteide](https://github.com/visualfc/liteide)编写Go代码时，使用ctrl+s即可调用gofmt。
  另一种方法是使用 [goimports](https://godoc.org/golang.org/x/tools/cmd/goimports)，它是 `gofmt`的超集，可根据需要添加（删除）行。
  
## 注释语句

  参考 [https://golang.org/doc/effective_go.html#commentary] 
  
  所有导出的名称、函数声明、结构声明等都应该有注释  
  
  Go语言提供C风格的`/* */`块注释和C++风格的`//`行注释。
  
  包应该有一个包注释，对于多文件包，注释只需要出现在任意一个文件中。包注释应当提供整个包相关的信息。
  
  注释的句子应当具有完整性，这使得它们在提取到文档时能保持良好的格式。使用godoc时包注释将与声明一起提取，作为项目的解释性文本。这些注释的风格和内容决定了文档的质量。
  
  注释应当以描述的事物名称开头，以句点（或者 ! ? ）结束。
  
  参考以下格式：

```go
// Request represents a request to run a command.
type Request struct { ...

// Encode writes the JSON encoding of req to w.
func Encode(w io.Writer, req *Request) { ...
```
```go
/*
Package regexp implements a simple library for regular expressions.

The syntax of the regular expressions accepted is:

    regexp:
        concatenation { '|' concatenation }
    concatenation:
        { closure }
    closure:
        term [ '*' | '+' | '?' ]
    term:
        '^'
        '$'
        '.'
        character
        '[' [ '^' ] character-ranges ']'
        '(' regexp ')'
*/
package regexp
```
### 包的注释

与godoc呈现的所有注释一样，包的注释必须出现在package子句的旁边，且不带空行：

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

## Contexts

Go提供了`context`包来管理gorountine的生命周期。

`context.Context`类型的值包括了跨API和进程边界的安全凭证，跟踪信息，结束时间和取消信号。

使用context包时请遵循以下规则：

* 不要将 Contexts 放入结构体，context应该作为第一个参数传入。 不过也有例外，函数签名(method signature)必须与标准库或者第三方库中的接口相匹配
```go
func F(ctx context.Context, /* other arguments */) {}
```
* 即使函数允许，也不要传入nil的 Context。如果不知道用哪种 Context，可以使用context.TODO()

* 使用context的Value相关方法只应该用于在程序和接口中传递的和请求相关的元数据，不要用它来传递一些可选的参数

* 相同的 Context 可以传递给共享结束时间、取消信号、安全凭证和父进程追踪等信息的多个goroutine，Context 是并发安全的

* 从不做特定请求的函数可以使用 context.Background()，但即使您认为不需要，也可以在传递Context时使用错误(error)。如果您有充分的理由认为替代方案是错误的，那么只能直接使用context.Background()

* 不要在函数签名中创建Context类型或者使用Context以外的接口。

官方使用context的例子：
```go
package main

import (
    "context"
    "fmt"
    "time"
)

func main() {
    d := time.Now().Add(50 * time.Millisecond)
    ctx, cancel := context.WithDeadline(context.Background(), d)

    // Even though ctx will be expired, it is good practice to call its
    // cancelation function in any case. Failure to do so may keep the
    // context and its parent alive longer than necessary.
    defer cancel()

    select {
    case <-time.After(1 * time.Second):
        fmt.Println("overslept")
    case <-ctx.Done():
        fmt.Println(ctx.Err())
    }
}
```

## Goroutine 生命周期

当你需要使用goruntines时，请确保它们什么时候/什么条件下退出。

Goroutines可能会因阻塞channel的send或者receives而泄露,即使被阻塞的通道无法访问，垃圾收集器也不会终止goroutine。

即使gorountine没有泄露，当不再需要它们时仍在继续运行会导致难以诊断的问题。即使在不需要结果后，修改正在使用的数据也会导致数据竞争。

尽量保证并发代码足够简单，使gorountine的生命周期更明显。如果难以做到，请记录gorountines退出的时间和原因。

## 拷贝

为了避免意外的别名(alias)，从另一个包复制结构时要小心。比如， `bytes.Buffer` 类型包含了 `[]byte` 切片类型，并且作为小字符串的优化，可被较小的字节数组引用。如果你拷贝了一个`Buffer`，拷贝中的切片可能会alias原始数组中的切片，从而导致后续的操作带来不可预测的结果。

通常来说，如果一个类型 `T`其方法与指针结构相关，那么请不要拷贝 `T`的值。

## 使用crypto/rand生成随机值

请不要使用包 `math/rand` 来生成密钥，即使是一次性的。如果不提供种子，则密钥完全可以被预测到。就算用`time.Nanoseconds()`作为种子，也仅仅只有几个位上的差别。

使用`crypto/rand`'s Reader作为代替。并且如果你需要生成文本，打印成16进制或者base64类型即可。

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

## 空切片

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

有关nil的更多讨论，参考 [Understanding Nil](https://www.youtube.com/watch?v=ynoY2xz-F8s).


## Panic

不要使用`panic`作为日常错误处理机制（除非你知道你在干什么），请使用`error`。

参考 https://golang.org/doc/effective_go.html#panic. 

## 错误

### 格式
错误信息的字符串不应该大写（除非以专有名词或首字母缩略词开头）或以标点符号结尾，因为它们通常是在其他上下文后打印的。

使用  `fmt.Errorf("something bad")` 而不是 `fmt.Errorf("Something bad")` 。

### 处理

请不要使用`_`丢弃错误。如果一个函数返回一个错误，请检查错误，处理错误，必要情况下请panic。

标准库函数通常会向调用者返回某种错误指示，比如os.Open不仅在失败时返回一个nil指针，它还返回一个描述错误的错误值。如果不确保调用函数的成功返回，后续代码的行为将变得不可预测。

参考 https://golang.org/doc/effective_go.html#errors.

### 附带返回

在C语言和类似语言中，函数常常返回 -1 或 null 值表示错误或者结果缺失。

```go
// Lookup returns the value for key or "" if there is no mapping for key.
func Lookup(key string) string

// Failing to check a for an in-band error value can lead to bugs:
Parse(Lookup(key))  // returns "parse failure for value" instead of "no value for key"
```

Go语言对多个返回值的支持提供了更好的解决方案。
相比要求客户端特判返回值情况，函数应该在最后返回一个附加值以指示其他返回值是否有效。此返回值可能是一个error，或者在不需要解释时可以是bool。

``` go
// Lookup returns the value for key or ok=false if there is no mapping for key.
func Lookup(key string) (value string, ok bool)
```
这还可以防止调用者错误地使用结果:

``` go
Parse(Lookup(key))  // compile-time error
```

鼓励编写更强大和可读性更强的代码:

``` go
value, ok := Lookup(key)
if !ok  {
    return fmt.Errorf("no value for %q", key)
}
return Parse(value)
```

此规则适用于导出的函数，但对未导出的函数也很有用。
如 nil， ""， 0 和 -1 都可以作为函数的无效返回值。
一些标准库函数，比如包 "strings" 中的函数，返回值都会附带一个err.这极大地简化了花在字符串上的操作，代价是需要程序员花更多的功夫来处理err。

总而言之，通常情况下，Go代码应额外返回一个error值.

### 缩进

尽量减少代码缩进，在获取错误后立即处理错误信息。这通常可以提高代码的可阅读性。

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
那么这可能需要将变量的声明另提一行：

```go
x, err := f()
if err != nil {
	// error handling
	return
}
// use x
```
## 用法示例

添加新的包时，请包含预期用法的示例。可以是一个可运行的例子或者演示完整调用的简单测试
参考 [可测试示例函数](https://blog.golang.org/examples)

## Imports

### 重命名、goimports
尽量避免导入包时的重命名，以避免名称冲突。如果发生名称冲突，尽量重命名本地或项目特定的包。

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
可以使用 [goimports](https://godoc.org/golang.org/x/tools/cmd/goimports) 来规范包的排序。

### Import Dot

使用 `import .` 来导入包在测试中很有用，但由于循环依赖性，它不能成为被测试包的一部分：

```go
package foo_test

import (
	"bar/testutil" // also imports "foo"
	. "foo"
)
```
在这种情况下，测试文件不能在包`foo`中，因为它使用了同样导入`foo`的包 `bar/testutil`，所以我们使用`import .`格式导入来让测试文件伪装成包foo的一部分，即使它并不是。除了这一情况，不要在你的程序中使用`import .`来导入包。它将使程序更难阅读，因为不清楚 Quux 这样的名称是在当前包中还是导入包中。

## 接口

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


## 单行代码长度

Go代码中没有严格限制行的长度，但请避免使用不舒服的长行。同样，当一行长代码可读性还行时，不要添加换行符来增加行数。

较长的行似乎与较长的名称有关，所以请尽量避免长的名称。
实际上，这与关于函数应该有多长的建议完全相同。没有规则“函数不应该超过N行”，但肯定会有这么长的一个函数，并且这个函数也可以分解为多个小函数来实现相同的功能。

## 名称

如果要在其他包使用该名称，请大写首字母。

名称如果是首字母或首字母缩略词（例如 "URL" 或 "NATO")，应当具有一致的大小写. 例如，"URL" 应写为 "URL" 或"url"，而不是 "Url"。

举个例子： ServerHTTP与ServerHttp，应当使用前者。
对于具有多个缩略单词的标识符，使用例如 "xmlHTTPRequest" 或 "XMLHTTPRequest"。
当"ID"是标识符的缩写时，此规则也同样适用，因此请写成 "appID" 而不是 "appId"。

### 变量名

Go中变量名应尽量短。对于局部变量尤其如此，首选`c`作为`lineCount`,`i`作为`sliceIndex`。

基本规则：名称使用的地方距其声明的地方越远，则名称必须越具描述性。对一个方法receiver，一个或两个字母就足够了。循环索引或读取之类的常见变量可以使单个字母(`i r`)，更多不常见的事物和全局变量需要更具描述性的名称。

### 包名

包名应当简洁、清晰、意义深刻。按照惯例，包名应该由小写字母组成，不要使用下划线或混合大小写。

包中名称的所有引用都将会使用到包名，所以你可以省略包中名称的标识符。
例如，`chubby.ChubbyFile`，可以精简为`chubby.File`。

请注意避免使用无意义的包名称，如 util,common,misc,api,types和interfaces。

参考：

http://golang.org/doc/effective_go.html#package-names 

http://blog.golang.org/package-names 

### 组合名

Go语言决定使用MixedCaps或mixedCaps来命名由多个单词组合的名称，而不是使用下划线来连接多个单词。即使它打破了其他语言的惯例。

例如，未导出的常量名为`maxLength` 而不是`MaxLength` 或 `MAX_LENGTH`。

### 结果参数命名

考虑下结果参数命名为下面这种情况时，godoc下的文档会是什么样子

```go
func (n *Node) Parent1() (node *Node)
func (n *Node) Parent2() (node *Node, err error)
```

文档会念起来更拗口，推荐下面这种：

```go
func (n *Node) Parent1() *Node
func (n *Node) Parent2() (*Node, error)
```

另一方面，如果函数返回值意义不明，给返回参数添加名称可能会很有用。

总之，尽量提高API可读性，比如：

```go
func (f *Foo) Location() (float64, float64, error)
```
更优的写法：

```go
// Location returns f's latitude and longitude.
// Negative values mean south and west, respectively.
func (f *Foo) Location() (lat, long float64, err error)
```

结论： 文档的清晰度比节约一两行代码更加重要。

最后，在某些情况下，如果您需要命名结果参数才能在defer中关闭变量的话，也是可以的。


## 传值

不要仅仅为了节省几个字节而传指针给函数。如果函数仅仅将其参数"x"作为`*x`使用，那么参数就不应该是指针。

常见的传指针的情况有：传递一个string的指针、指向接口值(`*io.Reader`)的指针;这两种情况下，值本身都是固定的，可以直接传递。

对于大型数据结构，或者是小型的可能增长的结构，请考虑传指针。

更多情况请见 [Receiver Type](#receiver-type)

## Receiver 

### 名称
方法receiver的名称应该反映其身份；通常，其类型的一个或两个字母缩写就足够了(例如"client"的"c"或"cl")。请不要使用通用名称如"me", "this" 或 "self"。请始终保持名称不变，如果你在一个地方命名receiver为 "c"，那么请不要在另一处叫其"cl"
### 指针还是值
如果您不知道怎么决定，请使用指针。但有时候receiver为值也挺有用，这种通常是出于效率的原因，且值为小的不变结构或基本类型的值。

一些建议：
* 请不要使用指针，如果满足receiver为map，func或者chan，或receiver为切片并且该方法不会重新分配切片
* 请不要使用指针，如果receiver是较小数组或结构体，没有可变的字段和指针，或者是一个简单的基本类型，int或者string
* 请使用指针，如果方法需要改变receiver
* 请使用指针以避免被拷贝，如果receiver包含了sync.Mutex 或类似同步字段的结构
* 请使用指针以提升效率，如果receiver是较大的数组或结构体
* 请使用指针，如果需要在方法中改变receiver的值
* 请使用指针，如果receiver是结构体，数组或切片这种成员是一个指向可变内容的指针
* 请使用指针，如果不清楚该如何选择

## 同步函数

相比异步函数，应当首选同步函数 —— 直接返回结果或在返回之前完成任何回调或通道操作的函数。

同步函数使得gorountine在调用中本地化，更容易推断其生命周期并避免泄露和数据竞争。它们也更容易测试:调用者可以传递输入并检查输出，而无需轮询或同步。

如果调用者需要更多的并发，他们可以简单地在单独的gorountine中调用函数来实现。但在调用者端删除不必要的并发是非常困难甚至是不可能的。

## 有效测试信息

测试失败时应当返回有效的错误信息，说明错误在哪，输入是什么，输出时什么，期望输出是什么。
一个典型的测试条件形如：

```go
if got != tt.want {
	t.Errorf("Foo(%q) = %d; want %d", tt.in, got, tt.want) // or Fatalf, if test can't test anything more past this point
}
```

请注意此处的命令是 `实际结果 != 预期结果`.一些测试框架鼓励程序员编写： 0 != x, "expected 0, got x"，go并不推荐。

在任何情况下，您有责任提供有用的错误信息，以便将来调试您的代码。

## 函数
* 函数采用命名的多值返回，但请考虑 [结果参数命名](#结果参数命名)
* 传入变量和返回变量以小写字母开头
```go
func nextInt(b []byte, pos int) (value, nextPos int) {
//
}
```
* 如果需要，请返回错误信息
