### 前言
测试驱动开发是一个写出高质量代码的好方法，同时避免将代码越写越烂，并证明你的代码能实现预期的效果。

### 以Table Driven的方式写测试用例
`Table Driven`的意思就是以表格的形式写好测试用例的输入和期望结果，然后写完所有测试用例之后直接在一个循环里遍历所有测试用例，这样的好处是你只需要专注写测试用例的输入和期望结果就OK了。

```go
package add

import "testing"

func TestAdd(t *testing.T) {
    tests := [][]int{ // test cases table
        {1, 1, 2},
        {100, 200, 300},
        {-2, 2, 0},
        {-3, -5, -8},
        {999, -1, 998},
    }

    for _, tc := range tests { // 遍历所有测试用例
        if res := Add(tc[0], tc[1]); res != tc[2] {
            t.Errorf("want: %d, got: %d", tc[2], res)
        }
    }
}
```

### 外部测试放在 "_test" 包

在Go中，除了`_test.go`文件，同一个目录下的文件都属于同一个包。将测试用例代码放在不同的包里，那么你写测试用例代码时，就好像这个包的真实的使用者一样。在同一个包内调用或写测试用例可能你觉得没什么问题，但当你需要暴露API给外部调用的时候，就考验你的代码功力了，因为你的任何改动都会影响到使用者。

![test-file](https://wx-mp-1259374020.file.myqcloud.com/external-test.png)

这样，在你更改内部的实现时也不需要更改测试用例的代码了。

### 内部测试放在`*_internal_test.go`文件
如果你需要写一些内部调用的测试用例，那么你可以将文件命名为`_internal_test.go`后缀，然后使用相同的package。内部测试要比API接口更加精细，但这是一个能使你的代码更可靠的好方法，尤其适合测试驱动开发。

![internal-test](https://wx-mp-1259374020.file.myqcloud.com/internal_test.png)

所以，一个完整的测试代码目录结构是这样的：
![tree-dir](https://wx-mp-1259374020.file.myqcloud.com/tree-dir.png)


### 利用interface写出可测试代码
现在假设我们正在实现一个叫`web`的web操作公共库，这个库提供了`Client`对象和`GetData`方法，用于从web服务中取数据，代码如下：
```go
package web

type Client struct{}

func NewClient() Client {
	return Client{}
}

func (c Client) GetData() (string, error) {
	return "data", nil
}
```

接着我们的`foo`包会引用`web`包的`Client`对象`GetData`方法去web服务中取数据，代码如下：
```go
package foo

import (
	"errors"

	"interfaces/web"
)

func Controller() error {
	webClient := web.NewClient()
	fromWebAPI, err := webClient.GetData()
	if err != nil {
		return err
	}
	// do some things based on data from web API
	if fromWebAPI != "data" {
		return errors.New("unexpected data")
	}
	return nil
}
```

现在我们需要测试Controller方法并分别写了两个测试方法，一个是测试成功获取到数据，另一个是测试两种获取数据失败的情况。

然后，问题来了。我们似乎没有办法同时测试到这些逻辑分支，因为我们没办法改变`web`包里面的逻辑。
```go
package foo_test

import (
	"testing"

	"interfaces/foo"
)

func TestController_Success(t *testing.T) {
	err := foo.Controller()
	if err != nil {
		t.FailNow()
	}
}

func TestController_Failure(t *testing.T) {
    // 这里我们想返回错误，但似乎比较难。
	err := foo.Controller()
	if err == nil {
        // 这个测试将会fail :(
		t.FailNow()
	}
}
```

到这里似乎把我们难住了，但如果我们将`web`包中的`Client`定义成interface，那我们就可以很容易的替换掉这个`Client`的实现。例如，改成下面这样：
```go
package foo

import (
	"errors"
)

type IWebClient interface {
	GetData() (string, error)
}

func Controller(webClient IWebClient) error {
	fromWebAPI, err := webClient.GetData()
	if err != nil {
		return err
	}
	// do some things based on data from web API
	if fromWebAPI != "data" {
		return errors.New("unexpected data")
	}
	return nil
}
```

然后，我们就可以很容易的测试`Controller`方法了，我们可以根据需要mock Client的实现。
```go
package foo_test

import (
	"errors"
	"testing"

	"interfaces/foo"
)

type MockClient struct {
	GetDataReturn string
}

func (mc MockClient) GetData() (string, error) {
	return mc.GetDataReturn, nil
}

func TestController_Success(t *testing.T) {
	err := foo.Controller(MockClient{"data"})
	if err != nil {
		t.FailNow()
	}
}

type FailingClient struct{}

func (fc FailingClient) GetData() (string, error) {
	return "", errors.New("oh no")
}

func TestController_Failure(t *testing.T) {
	// GetData() 失败分支
	err := foo.Controller(FailingClient{})
	if err == nil {
		t.FailNow()
	}
    // 错误数据分支
	err = foo.Controller(MockClient{"not data"})
	if err == nil {
		t.FailNow()
	}
}
```

就这样，我们所有代码的分支都已经覆盖到啦~


### Test Fixtures, Golden Files
在一些场景下，我们需要读取某些资源文件，比如我们在测试对 json 文件的解码功能时，就需要一些示例的 json 文件作为测试 case 的输入。像这种场景，把这些测试过程中用到的辅助文件，通常就叫做`Test Fixtures`。而放置这些文件的最佳位置就是放在叫`testdata`的目录下，主要原因有两条：

1. 测试代码运行时，working dir 就是测试代码的当前路径，可以使用相对路径轻易的访问到。
2. Go编译器在编译时会忽略叫作`testdata`的子目录。

看下面的例子：

```go
func helperLoadBytes(t *testing.T, name string) []byte {
    bytes, err := ioutil.ReadFile("testdata/somefixture.json")
    if err != nil {
        t.Fatal(err)
    }
    return bytes
}
```

那么，`Golden Files`又是什么呢？Golden Files 其实就是`Test Fixtures`中的一种，当测试用例的输出结果比较简单的时候，我们还可以把输出结果写在测试代码中。

但是当输出结果比较复杂时，直接写入代码已经不太合适了。所以此时，我们通常会把正确结果写入到文件里面，并且测试代码运行时，需要读取这个文件的内容进行比较。

一般`Golden Files`的使用都会配合`Table Driven`，每一个测试 case 的`Golden Files`的名字一般就会以“case名字+.golden” 来命名，这样在编写代码时也会比较简单。

### 计算测试覆盖率
测试覆盖率表示测试代码覆盖源代码的比例。

Go 1.2引入了一个新的计算test coverage的方法，其原理很简单：在编译之前，重写包的源代码和加入埋点，然后编译和运行重写后的代码，然后根据埋点就能统计出代码的覆盖率了。这个重写其实很简单，因为从测试到运行都是由go的原生工具链控制的。

示例代码：
```go
package size

func Size(a int) string {
    switch {
    case a < 0:
        return "negative"
    case a == 0:
        return "zero"
    case a < 10:
        return "small"
    case a < 100:
        return "big"
    case a < 1000:
        return "huge"
    }
    return "enormous"
}
```

测试代码：
```go
package size

import "testing"

type Test struct {
    in  int
    out string
}

var tests = []Test{
    {-1, "negative"},
    {5, "small"},
}

func TestSize(t *testing.T) {
    for i, test := range tests {
        size := Size(test.in)
        if size != test.out {
            t.Errorf("#%d: Size(%d)=%s; want %s", i, test.in, size, test.out)
        }
    }
}
```

然后当我们加上`-cover`参数即`go test -cover`，就可以得出测试覆盖率。
```
% go test -cover
PASS
coverage: 42.9% of statements
ok  	size	0.026s
%
```

我们再来看看go test内部是怎么加埋点的，下面是编译前埋点后的伪代码：
```go
func Size(a int) string {
    GoCover.Count[0] = 1 // 埋点1
    switch {
    case a < 0:
        GoCover.Count[2] = 1 // 埋点2...
        return "negative"
    case a == 0:
        GoCover.Count[3] = 1
        return "zero"
    case a < 10:
        GoCover.Count[4] = 1
        return "small"
    case a < 100:
        GoCover.Count[5] = 1
        return "big"
    case a < 1000:
        GoCover.Count[6] = 1
        return "huge"
    }
    GoCover.Count[1] = 1
    return "enormous"
}
```
其实就是在代码的各个分支加上埋点，最好再计算出覆盖率。

另外，go还提供了一种直观炫酷的方式展示测试的覆盖率，能精确到是否覆盖到某一行代码。执行以下命令：
```
go test -coverprofile=coverage.out && go tool cover -html=coverage.out
```

然后会自动的在浏览器中打开页面，如下：

![coverage.png](https://wx-mp-1259374020.file.myqcloud.com/coverage.png)

### 总结
单元测试是保证代码质量十分重要的一个环节。Go的源码和著名开源库，都是一边写源码一边写单元测试。在选择开源库的时候，测试覆盖率及测试用例的质量可以作为一个重要的指标。

最后在贴下`Dave Cheney`在推特转发的一些关于测试的哲学，十分有意思且有道理。

![dave-test.png](https://wx-mp-1259374020.file.myqcloud.com/dave-test.png)

### 参考资料
- 《Go 项目单元测试最佳实践》：https://fatsheep9146.github.io/2018/08/19/Go-%E9%A1%B9%E7%9B%AE%E5%8D%95%E5%85%83%E6%B5%8B%E8%AF%95%E6%9C%80%E4%BD%B3%E5%AE%9E%E8%B7%B5/
- 《5 simple tips and tricks for writing unit tests in #golang》：https://medium.com/@matryer/5-simple-tips-and-tricks-for-writing-unit-tests-in-golang-619653f90742
- 《The cover story》：https://blog.golang.org/cover
- 《Using Go Interfaces for Testable Code》：https://medium.com/swlh/using-go-interfaces-for-testable-code-d2e11b02dea
