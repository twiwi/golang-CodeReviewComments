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

  你可以在你的代码中运行 [Gofmt]("https://golang.org/cmd/gofmt/") 以自动修复大多数代码格式问题。几乎所有的代码都使用 `gofmt`。<br>
  另一种方法是使用[goimports]("https://godoc.org/golang.org/x/tools/cmd/goimports" )，它是 `gofmt`的超集，可根据需要添加（删除）行。
## Comment Sentences

  参考[https://golang.org/doc/effective_go.html#commentary]("https://golang.org/doc/effective_go.html#commentary" "悬停显示")。注释的句子应当具有完整性，即使会多于。但这种方法使得它们在提取到godoc文档时能保持良好的格式。注释应当以描述的事物名称开头，以句点结束。<br>
  参考以下格式<br>

```go
// Request represents a request to run a command.
type Request struct { ...

// Encode writes the JSON encoding of req to w.
func Encode(w io.Writer, req *Request) { ...
```

请注意，除了句点(.)外，还有很多符号可以作为替代，比如( ? , !)。除此之外，还有许多工具使用注释来标记类型和方法，(如easyjson:json和golint的MATCH)。这使得这条规则难以形式化


## Contexts
context.Context类型的值囊括了跨API和进程边界的安全凭证，跟踪信息，Context应当结束的时间和取消信号。Go程序在整个函数调用链中显式传递上下文，从传入的RPC和HTTP请求到传出请求。
大多数使用Context的函数应该将context.Context作为其第一个参数:

```go
func F(ctx context.Context, /* other arguments */) {}
```
从不做特定请求的函数可以使用 context.Background()，但即使您认为不需要，也可以再传递Context时使用错误(err)。如果您有充分的理由认为替代方案是错误的，那么只能直接使用context.Background()

不要将Context成员添加到一个结构体类型；相而是将ctx作为函数参数添加到需要传递它的该类型的每一个方法上。不过也有例外，函数签名(method signature)必须与标准库或者第三方库中的接口相匹配。

不要在函数签名中创建Context类型或者使用Context以外的接口。

如果要传递应用程序的数据，请将其放在参数中，接收方如果？？？？？

Contexts 是不可变的，因此可以将同一个Context传递给共享结束时间、取消信号、安全凭证和父进程追踪等信息的多个调用。


## Copying

为了避免意外的别名(alias)，从另一个包复制结构时要小心。比如, `bytes.Buffer` 类型包含了 `[]byte` 切片类型，并且作为小字符串的优化，可被较小的字节数组引用。如果你拷贝了一个`Buffer`，拷贝中的切片可能会alias原始数组中的切片，从而导致后续的操作带来令人惊讶的结果。

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

前者声明了一个指向nil的切片，后者声明了一个 non-nil 且 len = 0的切片。虽然他们的功能是等价的-它们的 `len` 和`cap` 都是0，但nil切片应当作为首选方案。

请注意在有些情况下， non-nil切片表现更好，比如当解码JSON对象时，nil切片解码为 `null`, non-nil切片解码为 JSON 数组

在设计接口时，应当避免因nil切片和non-nil切片带来的不同表现，因为这可能会导致细微的编码错误。

有关Go中nil的更多讨论，请参阅 Francesc Campoy 的演讲 [Understanding Nil](https://www.youtube.com/watch?v=ynoY2xz-F8s).


## Crypto Rand

请不要使用包 `math/rand`来生成密钥，即使是一次性的。如果不提供种子，则密钥完全可以被预测到。用`time.Nanoseconds()`作为种子，there was just a few bits of entropy.
相反，使用`crypto/rand`'s Reader作为代替。并且如果你需要生成文本，打印成16进制或者base64类型即可。
and if you need text, print to hexadecimal or base64:

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

参阅 https://golang.org/doc/effective_go.html#errors. 不要使用`panic`作为日常错误处理机制，请使用`error`或者多返回值。

## Error Strings

错误信息的字符串不应该大写（除非以专有名词或首字母缩略词开头）或以标点符号结尾，因为它们通常是在其他上下文后打印的。也就是说，使用  `fmt.Errorf("something bad")` 而不是 `fmt.Errorf("Something bad")` 。这样的话 `log.Printf("Reading %s: %v", filename, err)` 就不会在字符串中间有大写字母。但这种格式不适合日志记录，？？？？？

## Examples

添加新的包时，请包含预期用法的示例。可以是一个可运行的例子或者演示完整调用的简单测试
了解更多有关 [可测试示例函数]("https://blog.golang.org/examples")


## Goroutine Lifetimes

当你需要使用goruntines时，请确保它们什么时候/什么条件下退出。

Goroutines可以会因阻塞channel的send或者receives而泄露,即使被阻塞的通道无法访问，垃圾收集器也不会终止goroutine

即使gorountine没有泄露，当不再需要它们时将它们继续运行会导致其他微妙且难以诊断的问题。Sends on closed channels panic.即使在不需要结果后，修改正在使用的数据也会导致数据竞争，并且将gorountine 置于 in-flight状态任意时常也会导致不可预测的内存消耗。

尽量保证并发代码足够简单，使gorountine的生命周期更明显。如果难以做到，请记录gorountines退出的时间和原因。

## Handle Errors

参考 https://golang.org/doc/effective_go.html#errors. 请不要使用`_`丢弃错误。如果一个函数返回一个错误，检查并确保函数运行成功。处理错误，返回错误，必要情况下请panic。

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

<a href="https://godoc.org/golang.org/x/tools/cmd/goimports">goimports</a> 会帮你规范这些。

## Import Dot

使用 import .导入包在测试中很有用，但由于循环依赖性，它不能成为被测试包的一部分：

```go
package foo_test

import (
	"bar/testutil" // also imports "foo"
	. "foo"
)
```
在这种情况下，测试文件不能在包`foo`中，因为它使用了包 `bar/testutil`，它将导入`foo`，所以我们使用`import .`格式导入来让文件伪装成包foo的一部分，即使它并不是。除了这一情况，不要在你的程序中使用`import .`来导入包。它将使程序更难阅读，因为不清楚 Quux 这样的名称是在当前包中还是导入包中。

## In-Band Errors

在C语言和类似语言中，函数常常返回-1或 null 值表示错误或者结果缺失。

```go
// Lookup returns the value for key or "" if there is no mapping for key.
func Lookup(key string) string

// Failing to check a for an in-band error value can lead to bugs:
Parse(Lookup(key))  // returns "parse failure for value" instead of "no value for key"
```

Go语言对多个返回值的支持提供了更好的解决方案。
相比要求客户端检查一个带错误情况的值,函数应该返回一个附加值以指示其他返回值是否有效。此返回值可能是一个error，或者在不需要解释时可以是bool，并且应当位于返回值的最后。

``` go
// Lookup returns the value for key or ok=false if there is no mapping for key.
func Lookup(key string) (value string, ok bool)
```
这可以防止调用者错误地使用结果:

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

返回值如nil, "", 0, 和-1都可以作为函数的有效返回值，即调用者不需要针对结果的不同做区分处理。

一些标准库函数，比如包 "strings" 中的函数，返回值都会附带一个err.这极大地简化了字符串操作，代价是需要程序员花更多的功夫.
通常情况下，Go代码应返回一个error值.

## Indent Error Flow

尝试将代码路径保持在最小的缩进处，并缩进错误处理，第一个处理错误信息.这通常可以保证快速地检查正常路径来提高代码的可阅读性.
例如，不要写:

```go
if err != nil {
	// error handling
} else {
	// normal code
}
```

代替的写:

```go
if err != nil {
	// error handling
	return // or continue, etc.
}
// normal code
```

如果`if`语句有初始化语句，比如:

```go
if x, err := f(); err != nil {
	// error handling
	return
} else {
	// use x
}
```
那么这可能需要将短变量声明提到它自己的一行:

```go
x, err := f()
if err != nil {
	// error handling
	return
}
// use x
```

## Initialisms

名称中的单词如果是首字母或首字母缩略词（例如 "URL" 或 "NATO"），应当具有一致的大小写. 例如，"URL" 应写为 "URL" 或"url"，而不是 "Url".
举个例子： ServerHTTP与ServerHttp，应当使用前者.
对于具有多个初始化初始化“单词”的标识符，使用例如 "xmlHTTPRequest" 或 "XMLHTTPRequest".

当"ID"是标识符的缩写时，此规则也同样适用，因此请写成 "appID" 而不是 "appId"

protocol buffer compiler 产生的代码不受此规则的约束。人工编写的代码理应比机器编写的代码保持更高的标准.


## Interfaces

Go语言接口类型通常属于包中的interface类型，而不是实现这些值的包.实现了接口的包应该返回具体的(通常是指针或者结构)类型:这样，新方法可以不需要大量重构就添加到实现中。

不要为了模仿而在API的实现端定义接口。
相反，设计API以便可以使用公共API来进行测试。

在使用之前不要定义接口：在使用中没有真实的例子时，很难看出接口是否必需，更不要说它应该包含哪些方法。

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

作为替代返回一个具体类型，让消费者模拟生产者的实现。

``` go
package producer

type Thinger struct{ … }
func (t Thinger) Thing() bool { … }

func NewThinger() Thinger { return Thinger{ … } }
```


## Line Length

Go代码中没有严格限制行的长度，但请避免使用不舒服的长行。同样，当一行长代码可读性还行时，不要添加换行符来增加行数

较长的行似乎与长名称有关，尽量避免长的名称。

实际上，这与关于函数应该有多长的建议完全相同。没有规则“函数不应该超过N行”，但肯定有这么长的一个函数，也可以分解为多个小函数来解决。

## Mixed Caps

参考 https://golang.org/doc/effective_go.html#mixed-caps. 即使它打破了其他语言的惯例，这也适用。例如，未导出的常量名为`maxLength` 而不是`MaxLength` 或 `MAX_LENGTH`.

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

另一方面，如果函数返回两个或三个相同类型的参数，或者如果从上下文中看不出结果的意义，添加名称可能会有用。不要仅仅为了免于在函数中声明变量而命名结果参数。

不要为了降低实施复杂度，而造成API难以阅读

```go
func (f *Foo) Location() (float64, float64, error)
```

劣于

```go
// Location returns f's latitude and longitude.
// Negative values mean south and west, respectively.
func (f *Foo) Location() (lat, long float64, err error)
```

结论： 仅仅为了用返回参数的名称来返回而命名返回参数是不值得的，文档的清晰度比在函数中节约一两行代码更加重要。

最后，在某些情况下，您需要命名结果参数才能 change it in a deferred closure


## Naked Returns

见 [Named Result Parameters](#named-result-parameters).

## Package Comments

Package comments, like all comments to be presented by godoc, must appear adjacent to the package clause, with no blank line.

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

For "package main" comments, other styles of comment are fine after the binary name (and it may be capitalized if it comes first), For example, for a `package main` in the directory `seedgen` you could write:

``` go
// Binary seedgen ...
package main
```
or
```go
// Command seedgen ...
package main
```
or
```go
// Program seedgen ...
package main
```
or
```go
// The seedgen command ...
package main
```
or
```go
// The seedgen program ...
package main
```
or
```go
// Seedgen ..
package main
```

These are examples, and sensible variants of these are acceptable.

Note that starting the sentence with a lower-case word is not among the
acceptable options for package comments, as these are publicly-visible and
should be written in proper English, including capitalizing the first word
of the sentence. When the binary name is the first word, capitalizing it is
required even though it does not strictly match the spelling of the
command-line invocation.

See https://golang.org/doc/effective_go.html#commentary for more information about commentary conventions.

## Package Names

All references to names in your package will be done using the package name, 
so you can omit that name from the identifiers. For example, if you are in package chubby, 
you don't need type ChubbyFile, which clients will write as `chubby.ChubbyFile`.
Instead, name the type `File`, which clients will write as `chubby.File`.
Avoid meaningless package names like util, common, misc, api, types, and interfaces. See http://golang.org/doc/effective_go.html#package-names and
http://blog.golang.org/package-names for more.

## Pass Values

Don't pass pointers as function arguments just to save a few bytes.  If a function refers to its argument `x` only as `*x` throughout, then the argument shouldn't be a pointer.  Common instances of this include passing a pointer to a string (`*string`) or a pointer to an interface value (`*io.Reader`).  In both cases the value itself is a fixed size and can be passed directly.  This advice does not apply to large structs, or even small structs that might grow.

## Receiver Names

The name of a method's receiver should be a reflection of its identity; often a one or two letter abbreviation of its type suffices (such as "c" or "cl" for "Client"). Don't use generic names such as "me", "this" or "self", identifiers typical of object-oriented languages that place more emphasis on methods as opposed to functions. The name need not be as descriptive as that of a method argument, as its role is obvious and serves no documentary purpose. It can be very short as it will appear on almost every line of every method of the type; familiarity admits brevity. Be consistent, too: if you call the receiver "c" in one method, don't call it "cl" in another.

## Receiver Type

Choosing whether to use a value or pointer receiver on methods can be difficult, especially to new Go programmers.  If in doubt, use a pointer, but there are times when a value receiver makes sense, usually for reasons of efficiency, such as for small unchanging structs or values of basic type. Some useful guidelines:

  * If the receiver is a map, func or chan, don't use a pointer to them. If the receiver is a slice and the method doesn't reslice or reallocate the slice, don't use a pointer to it.
  * If the method needs to mutate the receiver, the receiver must be a pointer.
  * If the receiver is a struct that contains a sync.Mutex or similar synchronizing field, the receiver must be a pointer to avoid copying.
  * If the receiver is a large struct or array, a pointer receiver is more efficient.  How large is large?  Assume it's equivalent to passing all its elements as arguments to the method.  If that feels too large, it's also too large for the receiver.
  * Can function or methods, either concurrently or when called from this method, be mutating the receiver? A value type creates a copy of the receiver when the method is invoked, so outside updates will not be applied to this receiver. If changes must be visible in the original receiver, the receiver must be a pointer.
  * If the receiver is a struct, array or slice and any of its elements is a pointer to something that might be mutating, prefer a pointer receiver, as it will make the intention more clear to the reader.
  * If the receiver is a small array or struct that is naturally a value type (for instance, something like the time.Time type), with no mutable fields and no pointers, or is just a simple basic type such as int or string, a value receiver makes sense.  A value receiver can reduce the amount of garbage that can be generated; if a value is passed to a value method, an on-stack copy can be used instead of allocating on the heap. (The compiler tries to be smart about avoiding this allocation, but it can't always succeed.) Don't choose a value receiver type for this reason without profiling first.
  * Finally, when in doubt, use a pointer receiver.

## Synchronous Functions

Prefer synchronous functions - functions which return their results directly or finish any callbacks or channel ops before returning - over asynchronous ones.

Synchronous functions keep goroutines localized within a call, making it easier to reason about their lifetimes and avoid leaks and data races. They're also easier to test: the caller can pass an input and check the output without the need for polling or synchronization.

If callers need more concurrency, they can add it easily by calling the function from a separate goroutine. But it is quite difficult - sometimes impossible - to remove unnecessary concurrency at the caller side.

## Useful Test Failures

Tests should fail with helpful messages saying what was wrong, with what inputs, what was actually got, and what was expected.  It may be tempting to write a bunch of assertFoo helpers, but be sure your helpers produce useful error messages.  Assume that the person debugging your failing test is not you, and is not your team.  A typical Go test fails like:

```go
if got != tt.want {
	t.Errorf("Foo(%q) = %d; want %d", tt.in, got, tt.want) // or Fatalf, if test can't test anything more past this point
}
```

Note that the order here is actual != expected, and the message uses that order too. Some test frameworks encourage writing these backwards: 0 != x, "expected 0, got x", and so on. Go does not.

If that seems like a lot of typing, you may want to write a [[table-driven test|TableDrivenTests]].

Another common technique to disambiguate failing tests when using a test helper with different input is to wrap each caller with a different TestFoo function, so the test fails with that name:

```go
func TestSingleValue(t *testing.T) { testHelper(t, []int{80}) }
func TestNoValues(t *testing.T)    { testHelper(t, []int{}) }
```

In any case, the onus is on you to fail with a helpful message to whoever's debugging your code in the future.

## Variable Names

Variable names in Go should be short rather than long.  This is especially true for local variables with limited scope.  Prefer `c` to `lineCount`.  Prefer `i` to `sliceIndex`.

The basic rule: the further from its declaration that a name is used, the more descriptive the name must be. For a method receiver, one or two letters is sufficient. Common variables such as loop indices and readers can be a single letter (`i`, `r`). More unusual things and global variables need more descriptive names.
