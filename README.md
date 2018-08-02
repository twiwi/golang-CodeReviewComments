# GO语言简单注释规范


也可参考 [Effective Go](https://golang.org/doc/effective_go.html "悬停显示"). <br>

## Gofmt
  你可以在你的代码中运行 [Gofmt]("https://golang.org/cmd/gofmt/" "悬停显示") 以自动修复大多数代码格式问题。几乎所有的代码都使用 `gofmt`。<br>
  另一种方法是使用[goimports]("https://godoc.org/golang.org/x/tools/cmd/goimports" "悬停显示")，它是 `gofmt`的超集，可根据需要添加（删除）行。
## 注释语句
  参考[https://golang.org/doc/effective_go.html#commentary]("https://golang.org/doc/effective_go.html#commentary" "悬停显示")。注释的句子应当具有完整性，即使会多于。但这种方法使得它们在提取到godoc文档时能保持良好的格式。注释应当以描述的事物名称开头，以句点结束。<br>
  参考以下格式<br>
  ```golang
  // get returns the value of the severity.
func (s *severity) get() severity {
	return severity(atomic.LoadInt32((*int32)(s)))
}
  ```
  
