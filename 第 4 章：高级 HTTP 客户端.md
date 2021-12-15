# 高级 HTTP 客户端
在本章中，你将深入了解编写 HTTP 客户端。上一章重点介绍了通过 HTTP 执行各种操作。本章重点介绍用于编写健壮且可扩展的 HTTP 客户端的技术。你将学习在客户端中强制执行超时、创建客户端中间件并探索连接池。让我们开始吧！

## 使用自定义 HTTP 客户端

让我们考虑一下你在前一章中编写的数据下载器应用程序。与你进行通信的服务器总是按预期运行的情况很少见。事实上，不仅仅是服务器，你的应用程序请求通过的任何其他网络设备都可能表现不佳。那么我们的客户怎么办？让我们来了解一下。

### 从过载的服务器下载

让我们考虑以下函数，它将创建一个始终超载的测试 HTTP 服务器，其中每个响应都延迟 60 秒：

```go
func startBadTestHTTPServer() *httptest.Server {
    ts := httptest.NewServer(
        http.HandlerFunc(
            func(w http.ResponseWriter, r *http.Request) {
                time.Sleep(60 * time.Second)
                fmt.Fprint(w, "Hello World")
            }))
    return ts
}
```

请注意时间包中对 Sleep() 函数的调用。这将在向客户端发送响应之前引入 60 秒的延迟。清单 4.1 显示了一个测试函数，它向坏的测试服务器发送一个 HTTP GET 请求。

清单 4.1：使用坏服务器测试 fetchRemoteResource() 函数

```go
// chap4/data-downloader/fetch_remote_resource:bad_server_test.go
package main
 
import (
    "fmt"
    "net/http"
    "net/http/httptest"
    "testing"
    "time"
)
 
// TODO: Insert definition of startBadTestHTTPServer() from above
 
func TestFetchBadRemoteResource(t *testing.T) {
    ts := startBadTestHTTPServer()
    defer ts.Close()

    data, err := fetchRemoteResource(ts.URL)
    if err != nil {
        t.Fatal(err)
    }

    expected := "Hello World"
    got := string(data)

    if expected != got {
        t.Errorf("Expected response to be: %s, Got: %s", expected, got)
    }
}
```


创建一个新目录，chap4/data-downloader。从 chap3/data-downloader 复制所有文件。更新 go.mod 文件如下：

```sh
module github.com/username/chap4/data-downloader
 
go 1.16
```

接下来，将代码清单 4.1 保存到一个新文件 fetch_remote_resource:bad_server_test.go 中，并运行测试：

```sh
$ go test -v
=== RUN   TestFetchBadRemoteResource
--- PASS: TestFetchBadRemoteResource (60.00s)
=== RUN   TestFetchRemoteResource
--- PASS: TestFetchRemoteResource (0.00s)
PASS
ok          github.com/practicalgo/code/chap4/data-downloader    60.142s
```

如你所见，TestFetchBadRemoteResource 测试现在需要 60 秒才能运行。事实上，如果坏服务器在发回响应之前休眠 600 秒，我们在 fetchRemoteResource() 中的客户端代码（清单 3.1）将等待相同的时间。可以想象，这将导致非常糟糕的用户体验。

我们在第 2 章中谈到了将健壮性引入我们的应用程序的主题。接下来，让我们看看如何改进我们的数据下载器功能，使其在服务器花费的时间超过指定的持续时间时不等待响应。

让我们的数据下载器只等待指定的最长时间的答案是使用自定义 HTTP 客户端。当我们使用 http.Get() 函数时，我们隐式地使用了 net/http 包中定义的默认 HTTP 客户端。默认客户端通过变量 DefaultClient 提供，该变量创建为 var DefaultClient = &Client{}。这里的Client结构体是在net/http包中定义的，在它的字段中我们可以配置HTTP客户端的各种属性。我们现在要看的是 Timeout 字段。稍后，我们将研究另一个领域， Transport 。

Timeout 的值是一个 time.Duration 对象，它实质上指定了允许客户端连接到服务器、发出请求和读取响应的最大持续时间。如果未指定，则没有强制执行的最大持续时间，因此客户端将简单地等待直到服务器回复或客户端或服务器终止连接。

例如，要创建一个具有 100 毫秒超时的 HTTP 客户端，你将使用以下语句：

```go
client := http.Client{Timeout: 100 * time.Millisecond}
```

此语句最多允许 100 毫秒完成通过客户端发出的 HTTP 请求。使用自定义客户端，fetchRemoteResource() 函数现在将如下所示：

```go
func fetchRemoteResource(
    client *http.Client, url string,
) ([]byte, error) {
    r, err := client.Get(url)
    if err != nil {
        return nil, err
    }
    defer r.Body.Close()
    return io.ReadAll(r.Body)
}
```

请注意，我们不是调用 http.Get() 函数，而是调用传递给 fetchRemoteResource() 函数的 http.Client 对象的 Get() 方法。清单 4.2 显示了完整的应用程序代码。

清单 4.2：带有自定义 HTTP 客户端和超时的数据下载器应用程序

```go
// chap4/data-downloader-timeout/main.go
package main
 
import (
    "fmt"
    "io"
    "net/http"
    "os"
    "time"
)
 
// TODO Insert definition of fetchRemoteResource() from above
 
func createHTTPClientWithTimeout(d time.Duration) *http.Client {
    client := http.Client{Timeout: d}
    return &client
}
 
 
func main() {
    if len(os.Args) != 2 {
        fmt.Fprintf(
            os.Stdout,
            "Must specify a HTTP URL to get data from",
        )
        os.Exit(1)
    }
    client := createHTTPClientWithTimeout(15 * time.Second)
    body, err := fetchRemoteResource(client, os.Args[1])
    if err != nil {
        fmt.Fprintf(os.Stdout, "%#v\n", err)
        os.Exit(1)
    }
    fmt.Fprintf(os.Stdout, "%s\n", body)
}
```

我们定义了一个新函数 createHTTPClientWithTimeout()，以创建具有指定超时时间的自定义 HTTP 客户端，持续时间为 time.Duration 类型。在 main() 函数中，我们创建一个自定义客户端，配置超时时间为 15 秒，然后调用 fetchRemoteResource() 函数，传递客户端和指定的 URL。在新目录 chap4/data-downloader-timeout 中将清单 4.2 保存为一个新文件 main.go 并在其中初始化一个模块：

```sh
$ mkdir -p chap4/data-downloader-timeout
$ go mod init github.com/username/data-downloader-timeout
```

与其试图想出一个糟糕的 HTTP 服务器来测试超时行为，不如通过编写一个测试来实现。

### 测试超时行为

我们可以将代码清单 4.1 中的测试更新为代码清单 4.3 所示。

清单 4.3：使用坏服务器测试 fetchRemoteResource() 函数

```go
// chap4/data-downloader-timeout/fetch_remote_resource:bad_server_old_test.go
package main
 
import (
    "fmt"
    "net/http"
    "net/http/httptest"
    "testing"
    "time"
)
 
func startBadTestHTTPServerV1() *httptest.Server {
    // TODO Insert the function body of startBadTestHTTPServer from earlier
}
 
func TestFetchBadRemoteResourceV1(t *testing.T) {
    ts := startBadTestHTTPServerV1()
    defer ts.Close()

    client := createHTTPClientWithTimeout(200 * time.Millisecond)
    _, err := fetchRemoteResource(client, ts.URL)
    if err == nil {
        t.Fatal("Expected non-nil error")
    }

    if !strings.Contains(err.Error(), "context deadline exceeded") { 
        t.Fatalf("Expected error to contain: context deadline exceeded, Got: %v", err.Error())
    }
}
```

startBadTestHTTPServer()（在清单 4.1 中）已重命名为 startBadTestHTTPServerV1()。其他主要变化如下：

1. 我们通过调用 createHTTPClientWithTimeout() 函数创建一个 http.Client 对象。然后将该对象传递给 fetchRemoteResource() 函数调用。
2. 我们断言错误消息包含一个特定的子字符串，表明客户端关闭了与服务器的连接端。

将清单 4.3 保存为一个新文件 fetch_remote_resource:bad_server_old_test.go，与清单 4.2 位于同一目录中。运行测试：

```sh
$ go test -v
=== RUN   TestFetchBadRemoteResourceV1
2020/11/15 15:17:43 httptest.Server blocked in Close after 5 seconds, waiting for connections:
  *net.TCPConn 0xc00018a040 127.0.0.1:65227 in state active
FAIL
exit status 1
FAIL        github.com/practicalgo/code/chap4/data-downloader-timeout/    60.357s
```

从测试输出中可以看到，测试函数失败，但执行时间略多于 60 秒。你还会看到从 httptest.Server 记录的消息。这里发生了什么事？回想一下（在代码清单 4.1 和代码清单 4.3 中），我们在延迟调用中调用了测试服务器的 Close() 函数。测试函数执行完成后，调用 Close() 函数干净地关闭测试服务器。但是，此函数会在关闭之前检查是否有任何活动请求。因此，它仅在错误处理程序在 60 秒后返回响应时返回。我们对于它可以做些什么呢？

我们可以按如下方式重写我们的坏测试服务器：

```go
func startBadTestHTTPServerV2(shutdownServer chan struct{}) *httptest.Server {
ts := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
<-shutdownServer
fmt.Fprint(w, "Hello World")
}))
return ts
}
```

我们创建了一个无缓冲通道，shutdownServer，并将它作为参数传递给函数 startBadTestHTTPServerV2()。然后，在测试服务器的处理程序内部，我们尝试从通道读取，从而创建一个无限阻塞处理程序执行的潜在点。由于我们不关心通道内部的值，通道的类型是空结构，struct{}。通过阻塞读取操作替换 time.Sleep() 语句允许我们对测试服务器操作有更多的控制，我们将在接下来看到。

我们将更新我们的测试函数代码，如代码清单 4.4 所示。

清单 4.4：使用更新后的坏服务器测试 fetchRemoteResource() 函数

```go
// chap4/data-downloader-timeout/fetch_remote_resource:bad_server_test.go
package main
 
import (
    "fmt"
    "net/http"
    "net/http/httptest"
    "strings"
    "testing"
    "time"
)
 
// TODO Insert definition of startBadTestHTTPServerV2 from above
 
func TestFetchBadRemoteResourceV2(t *testing.T) {
    shutdownServer := make(chan struct{})
    ts := startBadTestHTTPServerV2(shutdownServer)
    defer ts.Close()
    defer func() {
        shutdownServer <- struct{}{}
    }()

    client := createHTTPClientWithTimeout(200 * time.Millisecond)
    _, err := fetchRemoteResource(client, ts.URL)
    if err == nil {
        t.Log("Expected non-nil error")
        t.Fail()
    }

    if !strings.Contains(err.Error(), "context deadline exceeded") {
        t.Fatalf("Expected error to contain: context deadline exceeded, Got: %v", err.Error())
    }
}
```

上述函数有三个关键变化：

1. 我们创建了一个无缓冲的通道，shutdownServer，类型为 struct{}——一个空的结构类型。
2. 我们创建了一个对匿名函数的新延迟调用，该函数将空结构值写入通道。此调用在 ts.Close() 调用之后进行，以便在 ts.Close() 函数之前调用它。
3. 我们将此通道作为参数调用 startBadTestHTTPServerV2() 函数。

将代码清单 4.4 保存为一个新文件 fetch_remote_resource:bad_server_test.go，在与代码清单 4.3 相同的目录中并运行以下测试：

```sh
$ go test -run TestFetchBadRemoteResourceV2 -v
=== RUN   TestFetchBadRemoteResourceV2
--- PASS: TestFetchBadRemoteResourceV2 (0.20s)
PASS
ok          github.com/practicalgo/code/chap4/data-downloader-timeout        0.335s
```

测试只运行了 0.2 秒（或 200 毫秒），这是我们为测试客户端配置的超时时间。现在会发生什么？在测试函数完成执行之前，首先调用匿名函数，该函数将一个空的结构体值写入shutdownServer 通道。这会解除对测试服务器处理程序的阻塞。因此，当调用 Close() 方法时，它会关闭测试服务器并成功返回。这样就完成了测试功能的执行。

为 HTTP 客户端配置超时是配置 HTTP 客户端的一种方法。接下来你将了解你可能想要配置的另一个方面——当服务器响应重定向时会发生什么？

### 配置重定向行为

当服务器发出 HTTP 重定向时，默认 HTTP 客户端会自动且静默地跟随重定向最多 10 次，然后终止。如果你想将其更改为根本不遵循任何重定向，或者至少让你知道它遵循重定向，该怎么办？这可以通过在 http.Client 对象中配置另一个字段 CheckRedirect 来实现。当设置为遵循特定签名的函数时，将在做出有关重定向的决定时调用此对象。然后你可以选择在那里实现你的自定义逻辑。让我们看一个如何实现这样一个函数的例子：

```go
func redirectPolicyFunc(req *http.Request, via []*http.Request) error {
    if len(via)>= 1 {
        return errors.New("stopped after 1 redirect")
    }
    return nil
}
```

实现重定向策略的自定义函数必须满足以下签名：

```go
func (req *http.Request, via []*http.Request) error
```

第一个参数 req 是跟随从服务器返回的重定向响应的请求；切片，via，包含迄今为止已发出的请求，最早的请求（你的原始请求）是该切片的第一个元素。这可以通过以下步骤更好地说明：

1. HTTP 客户端向原始 URL url 发送请求。
2. 服务器响应重定向到 url1 。
3. 现在使用 (url1, []{url}) 调用 redirectPolicyFunc。
4. 如果函数返回一个 nil 错误，它将跟随重定向并发送一个对 url1 的新请求。
5. 如果有另一个重定向到 url2，则使用 (url2, []{url, url1}) 调用 redirectPolicyFunc 函数。
6. 重复步骤 3、4 和 5，直到 redirectPolicyFunc 返回非 nil 错误。

因此，如果你使用 redirectPolicyFunc() 作为自定义重定向策略函数，则它根本不允许重定向。你如何将其与自定义 HTTP 客户端连接起来？你可以这样做：

```go
func createHTTPClientWithTimeout(d time.Duration) *http.Client {
    client := http.Client{Timeout: d, CheckRedirect: redirectPolicyFunc}
    return &client
}
```

让我们看看这个自定义重定向的实际效果。清单 4.5 显示了一个数据下载器，如果它看到服务器请求了重定向，它就会退出并出现错误。

清单 4.5：如果有重定向尝试则退出的数据下载器

```go
// chap4/data-downloader-redirect/main.go
package main
 
import (
    "errors"
    "fmt"
    "io"
    "net/http"
    "os"
    "time"
)
 
func fetchRemoteResource(client *http.Client, url string) ([]byte, error) {
    r, err := client.Get(url)
    if err != nil {
        return nil, err
    }
    defer r.Body.Close()
    return io.ReadAll(r.Body)
}
 
// TODO Insert definition of redirectPolicyFunc from above
 
// TODO Insert definition of createHTTPClientWithTimeout from above
 
func main() {
    if len(os.Args) != 2 {
        fmt.Fprintf(os.Stdout, "Must specify a HTTP URL to get data from")
        os.Exit(1)
    }
    client := createHTTPClientWithTimeout(15 * time.Second)
    body, err := fetchRemoteResource(client, os.Args[1])
    if err != nil {
        fmt.Fprintf(os.Stdout, "%v\n", err)
        os.Exit(1)
    }
    fmt.Fprintf(os.Stdout, "%s\n", body)
}
```

创建一个新目录，chap4/data-downloader-redirect，并在其中初始化一个模块：

```sh
$ mkdir -p chap4/data-downloader-redirect
$ cd chap4/data-downloader-redirect
$ go mod init github.com/username/data-downloader-redirect
```

接下来，将代码清单 4.5 作为一个新文件 main.go 保存在其中。构建并运行它，传递 http://github.com 作为第一个参数；你将看到以下内容：

```go
$ go build -o application
$ ./application http://github.com
Get "https://github.com/": Attempted redirect to https://github.com/
```

如果你直接使用 https URL https://github.com 尝试它，你将看到它转储了页面的内容。非常好。

你已经学习了如何自定义 http.Client 对象的重定向行为，这很好地引导我们进入本章的第一个练习，练习 4.1。

> 练习 4.1：增强 HTTP 子命令以允许配置重定向行为 在上一章中，你在 mync http 子命令中实现了 HTTP GET 功能。向子命令添加布尔标志 -disable-redirect，以便用户能够禁用默认重定向行为。

## 定制请求

你已经了解了如何创建自定义 HTTP 客户端。此外，你已经在 Client 对象上使用了诸如 Get() 之类的方法来发出请求。同样，你将使用 Post() 方法发出 POST 请求。在下面，客户端使用标准库中定义的 http.Request 类型的默认请求对象。现在你将学习如何自定义此对象。

自定义 http.Request 对象允许你添加标头或 cookie 或简单地设置请求的超时时间。创建新请求是通过调用 NewRequest() 函数完成的。 NewRequestWithContext() 函数具有完全相同的目的，但另外它还允许将上下文传递给请求。因此，在你的应用程序中，最好使用 NewRequestWithContext() 函数来创建新请求：

```go
req, err := http.NewRequestWithContext(ctx, "GET", url, nil)
```

该函数的第一个参数是上下文对象。第二个参数是 HTTP 方法，我们正在为其创建请求。 url 指向我们要向其发出请求的资源的 URL。最后一个参数是指向主体的 io.Reader 对象，在 GET 请求的情况下，它在大多数情况下可能为空。要使用 io.Reader 和 body 创建 POST 请求的请求，你将进行以下函数调用：

```go
req, err := http.NewRequestWithContext(ctx, "POST", url, body)
```

创建请求对象后，你可以使用以下内容添加标头：

```go
req.Header().Add("X-AUTH-HASH", "authhash")
```

这将向传出请求添加一个值为 authhash 的标头 X-AUTH-HASH。你可以将此逻辑封装在创建自定义 http.Request 对象的函数中，以使用标头发出 GET 请求：

```go
func createHTTPGetRequest(ctx context.Context, url string, headers map[string]string) (*http.Request, error) {
    req, err := http.NewRequestWithContext(ctx, "GET", url, nil)
    if err != nil {
        return nil, err
    }
    for k, v := range headers {
        req.Header.Add(k, v)
    }
    return req, err
}
```

要创建自定义 HTTP 客户端并发送自定义 GET 请求，你将编写如下内容：

```go
client := createHTTPClientWithTimeout(20 * time.Millisecond)
ctx, cancel := context.WithTimeout(context.Background(), 15*time.Millisecond)
defer cancel()
 
req, err := createHTTPGetRequest(ctx, ts.URL+"/api/packages", nil)
resp, err := client.Do(req)
```

客户端的 Do() 方法用于发送由 http.Request 对象 req 封装的自定义 HTTP 请求。

上述代码中的一个关键点是两个超时配置——一个在客户端级别，另一个在请求级别。当然，理想情况下你的请求超时（如果使用超时上下文）应该低于你的客户端超时，否则你的客户端可能会在你的请求超时之前超时。

请求对象的定制不仅限于添加标头。你也可以添加 cookie 和基本身份验证信息。这很好地引导我们进入练习 4.2。

> 练习 4.2：增强 HTTP 子命令以允许添加标头和基本身份验证凭证 增强 http 子命令以识别新选项 -header，它将向传出请求添加标头。可以多次指定此选项以添加多个标题，如下例所示：
>
> ```sh
> -header key1=value1 -header key1=value2
> ```
>
> 增强 http 子命令以定义新选项 -basicauth。你应该能够使用请求对象的 SetBasicAuth() 方法向请求添加基本身份验证信息，如下例所示：
>
> ```sh
> -basicauth user:password
> ```

## 实现客户端中间件
术语中间件（或拦截器）用于自定义代码，这些代码可以配置为与网络服务器或客户端应用程序中的核心操作一起执行。在服务器应用程序中，它是在服务器处理来自客户端的请求时执行的代码。在客户端应用程序中，它将是在向服务器应用程序发出 HTTP 请求时执行的代码。

在以下部分中，你将看到如何通过自定义客户端对象来实现自定义中间件。首先，让我们看看 Client 结构类型中的一个特定字段 Transport 。

### 了解 RoundTripper 接口

http.Client 结构体定义了一个字段 Transport，如下所示：

```go
type Client struct {
    Transport RoundTripper

    // Other fields
}
```

net/http 包中定义的 RoundTripper 接口定义了一种类型，它将从客户端携带 HTTP 请求到远程服务器，并将响应携带回客户端。这种类型需要实现的唯一方法是 RoundTrip()：

```go
type RoundTripper interface {
    RoundTrip(*Request) (*Response, error)
}
```


如果在创建客户端时未指定传输对象，则使用传输类型的预定义对象 DefaultTransport。其定义如下（省略字段）：

```go
var DefaultTransport RoundTripper = &Transport{
    // fields omitted
}
```


net/http 包中定义的 Transport 类型实现了 RoundTripper 接口所需的 RoundTrip() 方法。它负责创建和管理发生 HTTP 请求-响应事务的底层传输控制协议 (TCP) 连接：

1. 你创建一个 Client 对象。
2. 你创建一个 HTTP 请求。
3. 然后 HTTP 请求通过 RoundTripper 实现（例如，通过 TCP 连接）“传送”到服务器，并且响应被传送回。
4. 如果你向同一个客户端发出多个请求，将重复第 2 步和第 3 步。

为了实现客户端中间件，我们将编写一个自定义类型来封装 DefaultTransport 的 RoundTripper 实现。让我们看看如何。

### 日志中间件

你将编写的第一个中间件将在发送请求之前记录一条消息。当收到响应时，它将记录另一条消息。我们首先定义一个带有 *log.Logger 字段的 LoggingClient 结构类型：

```go
type LoggingClient struct {
    log *log.Logger
}
```

为了满足 RoundTripper 接口，我们实现了 RoundTrip() 方法：

```go
func (c LoggingClient) RoundTrip(
    r *http.Request,
) (*http.Response, error) {
    c.log.Printf(
        "Sending a %s request to %s over %s\n",
        r.Method, r.URL, r.Proto,
    )
    resp, err := http.DefaultTransport.RoundTrip(r)
    c.log.Printf("Got back a response over %s\n", resp.Proto)

    return resp, err
}
```

当调用我们的 RoundTripper 实现的 RoundTrip() 方法时，我们执行以下操作：

1. 记录传出请求 r 。
2. 调用 DefaultTransport 的 RoundTrip() 方法，将传出请求 r 传递给它。
3. 记录 RoundTrip() 调用返回的响应和错误。
4. 返回响应和返回的错误。

非常好。你已经定义了你自己的自定义 RoundTripper。现在你如何使用它？创建一个 Client 对象，并将 Transport 字段设置为 LoggingClient 对象：

```go
myTransport := LoggingClient{}
client := http.Client{
    Timeout:   10 * time.Second,
    Transport: &myTransport,
}
```

清单 4.6 是数据下载程序（清单 4.2）的修改版本，用于使用这个自定义的 RoundTripper 实现。

清单 4.6：带有自定义日志中间件的数据下载器

```go
// chap4/logging-middleware/main.go
 
package main
 
import (
    "fmt"
    "log"
    "net/http"
    "os"
    "time"
)
 
type LoggingClient struct {
    log *log.Logger
}
 
// TODO Insert definition of RoundTrip() function from above
 
func main() {        
    if len(os.Args) != 2 {
        fmt.Fprintf(os.Stdout, "Must specify a HTTP URL to get data from")
        os.Exit(1)
    }
    myTransport := LoggingClient{}
    l := log.New(os.Stdout, "", log.LstdFlags)
    myTransport.log = l

    client := createHTTPClientWithTimeout(15 * time.Second)
    client.Transport = &myTransport

    body, err := fetchRemoteResource(client, os.Args[1])
    if err != nil {
        fmt.Fprintf(os.Stdout, "%#v\n", err)
        os.Exit(1)
    }
    fmt.Fprintf(os.Stdout, "Bytes in response: %d\n", len(body))
}
```


重点修改突出显示。首先，我们创建一个新的 LoggingClient 对象。然后，我们通过调用 log.New() 函数创建一个新的 log.Logger 对象。该函数的第一个参数是一个 io.Writer 对象，日志将写入该对象。这里我们使用 os.Stdout。该函数的第二个参数是要添加到每个日志语句的前缀字符串——此处指定一个空字符串。该函数的最后一个参数是一个标志——每个日志行的前缀文本。这里我们使用 log.LstdFlags，它将显示日期和时间。然后我们将 log.Logger 对象分配给 myTransport 对象的 l 字段。最后，我们将 client.Transport 设置为 &myTransport 。

创建一个新的子目录 chap4/logging-middleware，并将代码清单 4.6 保存到一个新文件 main.go。构建并运行它，将 HTTP 服务器 URL 作为命令行参数传递：

```sh
$ go build -o application
$ ./application https://www.google.com
2020/11/25 22:03:40 Sending a GET request to https://www.google.com over HTTP/1.1
2020/11/25 22:03:40 Got back a response over HTTP/2.0
Bytes in response: 13583
```


正如预期的那样，日志语句首先出现，然后打印响应。你可以使用类似的自定义 RoundTripper 实现来发出请求延迟或非 200 错误等指标。你还可以使用自定义 RoundTripper 做什么？你可以编写一个 RoundTripper 实现来自动在缓存中查找请求，例如，根本避免进行调用。

在实现自定义 RoundTripper 时，你必须记住两件事：

1. RoundTripper 的实现必须假设在任何给定时间点可能有多个它的实例在运行。因此，如果你要操作任何数据结构，则该数据结构必须是并发安全的。
2. RoundTripper 不得改变请求或响应或返回错误。

### 向所有请求添加标头

让我们看一个实现中间件的示例，该中间件将为每个传出请求添加一个或多个 HTTP 标头。你最终可能会在各种场景中需要此功能 — 发送身份验证标头、传播请求 ID 等等。

我们将首先为我们的中间件定义一个新类型：

```go
type AddHeadersMiddleware struct {
    headers map[string]string
}
```

headers 字段是一个映射，其中包含我们要添加到 RoundTripper 实现中的 HTTP 标头：

```go
func (h AddHeadersMiddleware) RoundTrip(r *http.Request) (*http.Response, error) {
    reqCopy := r.Clone(r.Context())
    for k, v := range h.headers {
        reqCopy.Header.Add(k, v)
    }
    return http.DefaultTransport.RoundTrip(reqCopy)
}
```

该中间件将通过向原始请求添加标头来修改原始请求。然而，我们没有修改它，而是使用 Clone() 方法克隆请求并向其添加标头。然后我们用新的请求调用 DefaultTransport 的 RoundTrip() 实现。

清单 4.7 展示了一个带有这个中间件的 HTTP 客户端的实现。

清单 4.7：一个带有中间件的 HTTP 客户端，用于添加自定义标头

```go
// chap4/header-middleware/client.go
package client
 
import (
    "net/http"
)
 
type AddHeadersMiddleware struct {
    headers map[string]string
}
 
// TODO Insert RoundTrip() implementation from above
 
func createClient(headers map[string]string) *http.Client {
    h := AddHeadersMiddleware{
        headers: headers,
    }
    client := http.Client{
        Transport: &h,
    }
    return &client
}
```


创建一个新目录，chap4/header-middleware，并在其中初始化一个模块：

```sh
$ mkdir -p chap4/header-middleware
$ cd chap4/header-middleware
$ go mod init github.com/username/header-middleware
```

接下来，将代码清单 4.7 保存为一个新文件 client.go。为了测试是否将指定的标头添加到传出请求中，我们将编写一个测试服务器，将请求标头作为响应标头发回：

```go
func startHTTPServer() *httptest.Server {
    ts := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        for k, v := range r.Header {
            w.Header().Set(k, v[0])
        }
        fmt.Fprint(w, "I am the Request Header echoing program")
    }))
    return ts
}
```


清单 4.8 显示了使用上述测试服务器的测试功能。

代码清单 4.8：测试中间件以添加标头

```go
package client
 
import (
    "fmt"
    "net/http"
    "net/http/httptest"
    "testing"
)
 
// TODO Insert startHTTPServer() from above
 
func TestAddHeaderMiddleware(t *testing.T) {
    testHeaders := map[string]string{
        "X-Client-Id": "test-client",
        "X-Auth-Hash": "random$string",
    }
    client := createClient(testHeaders)

    ts := startHTTPServer()
    defer ts.Close()

    resp, err := client.Get(ts.URL)
    if err != nil {
        t.Fatalf("Expected non-nil [AU: "nil"—JA] error, got: %v", err)
    }

    for k, v := range testHeaders {
        if resp.Header.Get(k) != testHeaders[k] {
            t.Fatalf("Expected header: %s:%s, Got: %s:%s", k, v, k, testHeaders[k])
        }
    }
}
```


我们创建了一个映射 testHeaders，以指定要添加到传出请求的标头。然后调用 createClient() 函数，将映射作为参数传递。正如你在代码清单 4.7 中所见，该函数还创建了一个 AddHeaderMiddleware 对象，然后在创建 http.Client 对象时将其设置为 Transport。

将代码清单 4.8 保存到与代码清单 4.7 相同的目录中的新文件 header_middleware_test.go 中并运行测试：

```sh
% go test -v
=== RUN   TestAddHeaderMiddleware
--- PASS: TestAddHeaderMiddleware (0.00s)
PASS
ok          github.com/practicalgo/code/chap4/header-middleware        0.472s
```


在编写此中间件时，你已经看到了如何创建客户端中间件的示例，该中间件通过创建副本、修改它，然后将其移交给 DefaultTransport 来修改传入请求。

练习 4.3 将让你有机会实现一个中间件来记录请求延迟。

> 练习 4.3：用于计算请求延迟的中间件 与我们实现日志中间件的方式类似，实现一个中间件来记录完成请求所需的时间（以秒为单位）。日志记录应作为 mync http 子命令中的可选功能实现，该功能将使用 -report 选项启用。

## 连接池

在上一节中，你了解到默认的 RoundTripper 接口实现将 HTTP 请求携带到远程服务器，然后将响应携带回去。

发生的基本步骤之一是为你的请求建立新的 TCP 连接。这个连接建立过程是昂贵的。当你提出单一请求时，你可能不会注意到它。但是，例如，当你将 HTTP 请求作为面向服务架构的一部分发出时，你通常会在短时间内发出多个请求——无论是突发还是连续。在这种情况下，为每个请求执行 TCP 连接设置的成本很高。因此，net/http 库维护一个连接池，它会自动尝试重用现有的 TCP 连接来发送你的 HTTP 请求。

让我们首先了解连接池的工作原理，然后了解我们如何配置池本身。 net/http/httptrace 包将帮助我们深入研究连接池的内部结构。使用这个包可以看到的一件事是连接是否被重用，或者是否建立了一个新的连接来发出 HTTP 请求。事实上，它做的更多，但我们只会用它来演示连接重用。

考虑以下函数定义 createHTTPGetRequestWithTrace() ：

```go
func createHTTPGetRequestWithTrace(ctx context.Context, url string) (*http.Request, error) {
    req, err := http.NewRequestWithContext(ctx, "GET", url, nil)
    if err != nil {
        return nil, err
    }
    trace := &httptrace.ClientTrace{
        DNSDone: func(dnsInfo httptrace.DNSDoneInfo) {
            fmt.Printf("DNS Info: %+v\n", dnsInfo)
        },
        GotConn: func(connInfo httptrace.GotConnInfo) {
            fmt.Printf("Got Conn: %+v\n", connInfo)
        },
    }
    ctxTrace := httptrace.WithClientTrace(req.Context(), trace)
    req = req.WithContext(ctxTrace)        
    return req, err
}
```

httptrace.ClientTrace 结构类型定义了在请求生命周期中发生某些事件时将调用的函数。我们对这里的两个事件感兴趣：

- DNSDone 事件在对主机名的 DNS 查找完成时发生。
- GotConn 事件在获得连接以发送请求时发生。

为了定义在 DNSDone 事件发生时要调用的函数，我们在创建 struct 对象时将函数指定为字段的值。此函数必须接受类型为 httptrace.DNSDoneInfo 的对象作为参数并且不返回任何值。同样，我们定义了一个在 GotConn 事件发生时调用的函数。此函数必须接受类型为 httptrace.GotConnInfo 的对象作为参数并且不返回任何值。在这两个函数中，我们将对象打印到标准输出。

创建 ClientTrace 对象 trace 后，你可以通过调用 httptrace.WithClientTrace() 函数创建一个新上下文，将原始请求的上下文和跟踪对象传递给它。

最后，你通过添加这个上下文作为它的上下文来创建一个新的 Request 对象并返回这个对象。

清单 4.9 是一个程序，它使用 createHTTPGetRequestWithTrace() 函数向远程服务器发送 HTTP GET 请求。

清单 4.9：说明连接池的程序

```go
// chap4/connection-pool-demo/main.go
package main
 
import (
    "context"
    "fmt"
    "log"
    "net/http"
    "net/http/httptrace"
    "os"
    "time"
)
 
func createHTTPClientWithTimeout(d time.Duration) *http.Client {
    client := http.Client{Timeout: d}
    return &client
}
 
// TODO Insert definition of createHTTPGetRequestWithTrace() function from earlier
 
func main() {
    d := 5 * time.Second
    ctx := context.Background()
    client := createHTTPClientWithTimeout(d)

    req, err := createHTTPGetRequestWithTrace(ctx, os.Args[1])
    if err != nil {
        log.Fatal(err)
    }
    for {
        client.Do(req)
        time.Sleep(1 * time.Second)
        fmt.Println("--------")
    }
}
```

请注意，我们有一个无限循环，它发送相同的请求，中间有 1 秒的睡眠时间。你必须使用 Ctrl+C 组合键终止程序。

创建一个新目录，chap4/connection-pool-demo，并在其中初始化一个模块：

```sh
$ mkdir -p chap4/connection-pool-demo
$ cd chap4/ connection-pool-demo
$ go mod init github.com/username/connection-pool-demo
```

接下来，将代码清单 4.9 保存为一个新文件 main.go。构建并运行它，指定一个 HTTP 服务器主机名作为命令行参数。

```sh
$ go build -o application
$./application https://www.google.com
DNS Info: {Addrs:[{IP:216.58.200.100 Zone:} {IP:2404:6800:4006:810::2004 Zone:}] Err:<nil> Coalesced:false}
TLS HandShake Start
TLS HandShake Done
Got Conn: {Conn:0xc000096000 Reused:false WasIdle:false IdleTime:0s}
Resp protocol: "HTTP/2.0"
--------
Got Conn: {Conn:0xc000096000 Reused:true WasIdle:true IdleTime:1.003019133s}
Resp protocol: "HTTP/2.0"
--------
Got Conn: {Conn:0xc000096000 Reused:true WasIdle:true IdleTime:1.005444969s}
Resp protocol: "HTTP/2.0"
--------
Got Conn: {Conn:0xc000096000 Reused:true WasIdle:true IdleTime:1.005472933s}
Resp protocol: "HTTP/2.0"
^C
```

首先，我们看到 DNSDone 函数的输出。细节并不重要，但我们注意到我们只看到了一次。然后我们看到 GotConn 函数的输出。我们可以看到每次发出请求时都会调用它。我们还看到，对于第一个请求，Reused 的值为 false。 WasIdle 的值为 false，IdleTime 为 0。这告诉我们，对于第一个请求，创建了一个新连接并且它不是空闲的。对于所有后续请求，我们看到这些字段的值为真、真和非零空闲时间——接近 1 秒。当然，1 秒是我们在请求之间休眠的持续时间，因此我们将其视为连接处于空闲状态的时间。

### 配置连接池
连接池节省了为每个请求创建新连接的成本。但是，在现实生活中，你可能希望了解默认连接池可能会发生的各种问题。

首先，让我们考虑对你的主机名进行 DNS 查找。由于在大多数情况下，你直接处理的是主机名而不是 IP 地址，因此值得考虑 DNS 记录可能会发生变化——尤其是在云托管服务的动态世界中。现在，当用于建立连接的底层 IP 地址不再可用时，我们的连接池实现会发生什么情况？连接池实现是否会意识到它不再可用并创建一个到新 IP 地址的新连接？是的，事实上，它会。当你尝试发出新请求时，将打开一个到远程服务器的新连接。

接下来，让我们考虑这样一种情况：当 10 秒或更长时间过期时，你总是希望为每个 HTTP 请求强制建立一个新连接。为此，你将创建一个 Transport 对象，如下所示：

```go
transport := &http.Transport{
    IdleConnTimeout: 10 * time.Second,
}
```

然后，创建一个客户端，如下所示：

```go
client := http.Client{
    Timeout:   d,
    Transport: transport,
}
```

使用上述配置，空闲连接最多可保持 10 秒。因此，如果你使用客户端发出两个请求，它们之间的间隔为 11 秒，则第二个请求将触发要创建的新连接。

除了超时，你还可以配置另外两个相关参数：

- MaxIdleConns ：这表示要保留在池中的最大空闲连接数。默认情况下，这是 0，它不强制执行上限。
- MaxIdleConnsPerHost ：这是每个主机的最大空闲连接数。默认情况下，它被设置为 DefaultMaxIdleConnsPerHost 的值，从 Go 1.16 开始为 2。

练习 4.4 要求你在 mync 的 http 子命令中实现对配置连接池行为的支持。

> 练习 4.4：支持启用连接池行为 添加一个新选项 -num-requests，它接受一个整数作为 http 子命令的值，它将向服务器发出指定次数的相同请求。
>
> 添加一个新选项 -max-idle-conns，它接受一个整数来配置池中空闲连接的最大数量。

## 概括

我们从学习如何在 HTTP 客户端中实现超时行为开始本章。超时行为与请求上下文一起，允许你对客户端等待请求完成的时间设置上限。这允许你在应用程序中实现健壮性。然后，你了解了如何实现客户端中间件，它允许你在应用程序中开发各种功能 — 例如，日志记录、导出指标和缓存。最后，你了解了连接池以及如何配置它。

在下一章中，你将继续在编写 HTTP 应用程序的世界中学习如何编写可扩展且健壮的 HTTP 服务器应用程序。