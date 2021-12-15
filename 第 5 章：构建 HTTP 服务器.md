# 构建 HTTP 服务器
在本章中，你将深入了解编写 HTTP 服务器的基础知识。 你将了解处理程序函数的工作原理，发现有关处理请求的更多信息，并学习如何读写流数据。 在上一章中，你对这些主题的皮毛有所了解。 现在是时候潜入了。

## 你的第一个 HTTP 服务器

net/http 包为我们提供了编写 HTTP 服务器的构建块。 在你的计算机上运行的服务器（位于 http://localhost:8080）将按如下方式处理请求（见图 5.1）：

1. 服务器在某个路径接收客户端请求，比如 /api。
2. 服务器检查它是否可以处理此请求。
3. 如果答案是肯定的，则服务器调用处理程序函数来处理请求并返回响应。 如果没有，它会返回一个 HTTP 错误作为对客户端的响应。

![](./images/5-1.png)

图 5.1：HTTP 服务器的请求处理

清单 5.1 显示了你可以编写的最简单的 Web 服务器。

清单 5.1：一个基本的 HTTP 服务器

```go
// chap5/basic-http-server/server.go
package main
 
import (
    "log"
    "net/http"
    "os"
)
 
func main() {
    listenAddr := os.Getenv("LISTEN_ADDR")
    if len(listenAddr) == 0 {
        listenAddr = ":8080"
    }

    log.Fatal(http.ListenAndServe(listenAddr, nil))    
}
```

net/http 包中的 ListenAndServe() 函数在给定的网络地址启动一个 HTTP 服务器。使此地址可配置是个好主意。因此，在 main() 函数中，以下几行检查是否已指定 LISTEN_ADDR 环境变量，如果没有，则默认为 ":8080" ：

```go
listenAddr := os.Getenv("LISTEN_ADDR")
if len(listenAddr) == 0 {
    listenAddr = ":8080"
}
```

os 包中定义的 Getenv() 函数查找环境变量的值。如果找到环境变量 LISTEN_ADDR，其值将作为字符串返回。如果不存在这样的环境变量，则返回空字符串。因此 len() 函数可用于检查 LISTEN_ADDR 环境变量是否已指定。 “:8080”的默认值意味着服务器将侦听端口 8080 上的所有网络接口。如果你希望服务器只能在运行应用程序的计算机上访问，你可以将环境变量 LISTEN_ADDR 设置为“ 127.0.0.1:8080"，然后启动应用程序。

接下来，我们调用 ListenAndServe() 函数，指定要侦听的地址 ( listenAddr ) 和服务器的处理程序。我们将指定 nil 作为处理程序的值，因此我们对 ListenAndServe 的函数调用如下：

```go
log.Fatal(http.ListenAndServe(listenAddr, nil))
```

如果启动服务器时出现错误，ListenAndServe() 会立即返回错误值。如果服务器已正确启动，则该函数仅在服务器终止时返回。无论哪种情况，如果有错误值，log.Fatal() 函数都会记录错误值。

创建一个新目录，chap5/basic-http-server，并在其中初始化一个模块：

```sh
$ mkdir -p chap5/basic-http-server
$ cd chap5/basic-http-server
$ go mod init github.com/username/basic-http-server
```

接下来，将代码清单 5.1 保存为一个新文件 server.go。构建并运行它：

```sh
$ go build -o server
$ ./server
```

太好了，你的第一个 HTTP 服务器正在运行！

你如何向它提出请求？你可以使用 Internet 浏览器，但我们将使用 curl 命令行 HTTP 客户端。启动一个新的终端会话，并运行以下命令：

```sh
$ curl -X GET localhost:8080/api
404 page not found
```


我们在路径 /api 上向我们的 HTTP 服务器发出 HTTP GET 请求，我们得到一个 404 页面未找到响应。这意味着服务器接收传入的请求，查看它并返回 HTTP 404 响应，这意味着它找不到我们正在请求的资源 /api。接下来，让我们看看如何解决这个问题。要终止服务器，请在启动服务器的终端中按 Ctrl+C。

## 设置请求处理程序

当你指定 nil 作为 ListenAndServe() 函数的第二个参数时，你要求该函数使用默认处理程序 DefaultServeMux。它作为如何处理请求路径的默认注册表。 DefaultServeMux 是 http 包中定义的 ServeMux 类型的对象。它是一个全局对象，这意味着你可能在应用程序中使用的任何其他代码也可以向你的服务器注册处理程序函数。没有什么可以阻止第三方流氓包在你不知情的情况下暴露 HTTP 路径（见图 5.2）。此外，与任何其他全局对象一样，这会使你的代码面临不可预见的并发错误和非健壮行为。因此，我们不会使用它。相反，我们将创建一个新的 ServeMux 对象：

```go
mux := http.NewServeMux()
```

![](./images/5-2.png)

图 5.2：任何包都可以使用 DefaultServeMux 对象注册处理函数。

ServeMux 对象包含一个映射数据结构，其中包含你希望服务器处理的路径和相应的处理程序函数的映射。要解决你在上一节中遇到的 HTTP 404 问题，你需要为路径注册一个称为处理程序函数的特殊函数。你可能还记得我们在第 3 章“编写 HTTP 客户端”中编写了用于实现测试服务器的处理程序函数。现在让我们详细研究它们。

### 处理函数
处理函数必须是 func(http.ResponseWriter, *http.Request) 类型，其中 http.ResponseWriter 和 http.Request 是 net/http 包中定义的两种结构类型。 http.Request 类型的对象表示传入的 HTTP 请求，而 http.ResponseWriter 对象用于将响应写回给发出请求的客户端。下面是一个处理函数的例子：

```go
func apiHandler(w http.ResponseWriter, r *http.Request) {
    fmt.Fprintf(w, "Hello World")
}
```

请注意，无需从此函数返回。你写入 ResponseWriter 对象 w 的任何内容都会作为对客户端的响应发回。这里我们使用 fmt.Fprintf() 函数发送字符串“Hello World”。请注意，我们用来将字符串写入标准输出的 Fprintf() 函数如何同样适用于将字符串作为 HTTP 响应发回，这要归功于 io.Writer 接口的强大功能。当然，你可以在此处使用任何其他库函数，而不是 fmt.Fprintf() – io.WriteString()，例如。

你不仅限于编写字符串作为响应。例如，你可以直接使用 w.Write() 方法发送一个字节片作为响应。

编写处理程序函数后，下一步是将它注册到你之前创建的 ServeMux 对象：

```go
mux.HandleFunc("/api", apiHandler)
```

这会在 mux 对象中创建一个映射，以便对 /api 路径的任何请求现在都由 apiHandler() 函数处理。最后，我们将调用 ListenAndServe() 函数指定这个 ServeMux 对象：

```go
err := http.ListenAndServe(listenAddr, mux)
```

清单 5.2 显示了 HTTP 服务器的更新代码。它为两个路径注册处理函数： /api 和 /healthz 。

清单 5.2：使用专用 ServeMux 对象的 HTTP 服务器

```go
// chap5/http-serve-mux/server.go
package main
 
import (
    "fmt"
    "log"
    "net/http"
    "os"
)
 
func apiHandler(w http.ResponseWriter, req *http.Request) {
    fmt.Fprintf(w, "Hello, world!")
}
 
func healthCheckHandler(w http.ResponseWriter, req *http.Request) {
    fmt.Fprintf(w, "ok")
}
 
func setupHandlers(mux *http.ServeMux) {
    mux.HandleFunc("/healthz", healthCheckHandler)
    mux.HandleFunc("/api", apiHandler)
}
 
func main() {
    listenAddr := os.Getenv("LISTEN_ADDR")
    if len(listenAddr) == 0 {
        listenAddr = ":8080"
    }

    mux := http.NewServeMux()
    setupHandlers(mux)

    log.Fatal(http.ListenAndServe(listenAddr, mux))
}
```

我们通过调用 NewServeMux() 函数创建一个新的 ServeMux 对象 mux，然后调用 setupHandlers() 函数将 mux 作为参数传递。在 setupHandlers() 函数中，我们调用 HandleFunc() 函数来注册这两个路径及其对应的处理函数。然后我们调用 ListenAndServe() 函数，传递 mux 作为要使用的处理程序。

创建一个新目录，chap5/http-serve-mux，并在其中初始化一个新模块：

```sh
$ mkdir -p chap5/http-serve-mux
$ cd chap5/http-serve-mux
$ go mod init github.com/username/http-serve-mux
```


接下来，将代码清单 5.2 保存为一个新文件 server.go。构建并运行服务器：

```sh
$ go build -o server
$ ./server
```

在新终端上，使用 curl 向服务器发出 HTTP 请求。对于 /api 和 /healthz 路径，你将看到 Hello, world!和 ok 分别响应。

```sh
$ curl localhost:8080/api
Hello, world!
 
$ curl localhost:8080/healthz
ok
```

但是，如果你向任何其他路径发出请求，例如 /healtz / 或 / 路径，你将收到“404 page not found”：

```sh
$ curl localhost:8080/healtz/
404 page not found
 
$ curl localhost:8080/
404 page not found
```

当一个请求进来并且有一个处理函数可以处理它时，处理函数在一个单独的 goroutine 中执行。一旦处理完成，goroutine 就会终止（见图 5.3）。

![](./images/5-3.png)


图 5.3：每个传入的请求都由一个新的 goroutine 处理。

这确保服务器能够同时处理多个请求。作为一个理想的副作用，这也意味着如果在处理一个请求时出现运行时异常，这不会对正在处理的其他请求产生任何影响。

## 测试你的服务器
通过 curl 调用请求手动测试你的服务器适用于你的服务器的初步测试和验证，但它不能扩展。因此，你需要构建一个自动化程序来测试你的服务器，以便这些测试易于运行。 Go 标准库的 httptest 包（导入为 net/http/httptest ）为我们提供了各种功能来为 HTTP 服务器编写测试。从广义上讲，你将最终测试两类 HTTP 应用程序行为：

1. 服务器启动和初始化行为
2. 处理程序功能逻辑——你的 Web 应用程序的面向用户的功能

我们为第一类行为编写的测试将依赖于启动测试 HTTP 服务器，然后针对测试服务器发出 HTTP 请求。我们将此类测试称为集成测试。

第二类的测试不涉及设置测试服务器，而是使用专门创建的 http.Request 和 http.ResponseWriter 对象调用处理程序函数。我们将把这样的测试称为单元测试。

考虑清单 5.2 中的服务器。我们在 main() 函数中执行以下步骤：

1. 我们创建一个新的 ServeMux 对象。
2. 我们为路径 /api 和 /healthz 注册处理函数。
3. 我们调用 ListenAndServe() 函数以在 listenAddr 中指定的地址上启动服务器。

上述步骤包括我们的 HTTP 服务器的初始化和设置。对于第 1 步和第 2 步，我们要验证对我们的 Web 应用程序对 /api 路径的任何请求是否都转发到了 /api 处理程序。同样，对 /healthz 处理程序的任何请求都应转发到 healthcheck 处理程序。对任何其他路径的请求应返回 HTTP 404 错误。

我们不必测试服务器代码的第 3 步，因为标准库测试已经涵盖了它。

接下来，考虑清单 5.2 中的两个 HTTP 处理程序函数。 apiHandler 函数响应文本“Hello, world!”作为回应。 healthcheckHandler 函数以文本“ok”作为响应。因此，我们的测试应该验证这些处理函数是否将预期的文本作为响应返回。

足够的理论。清单 5.3 显示了测试我们的服务器和处理程序函数的测试函数。

清单 5.3：对 HTTP 服务器的测试

```go
// chap5/http-serve-mux/server_test.go
package main
 
import (
    "io"
    "log"
    "net/http"
    "net/http/httptest"
    "testing"
)
 
func TestServer(t *testing.T) {
    tests := []struct {
        name     string
        path     string
        expected string
    }{
        {
            name:     "index",
            path:     "/api",
            expected: "Hello, world!",
        },
        {      name:     "healthcheck",
         path:     "/healthz",
         expected: "ok",
        },
    }

    mux := http.NewServeMux()
    setupHandlers(mux)

    ts := httptest.NewServer(mux)
    defer ts.Close()

    for _, tc := range tests {
        t.Run(tc.name, func(t *testing.T) {
            resp, err := http.Get(ts.URL + tc.path)
            respBody, err := io.ReadAll(resp.Body)
            resp.Body.Close()
            if err != nil {
                log.Fatal(err)
            }
            if string(respBody) != tc.expected {
                t.Errorf(
                    "Expected: %s, Got: %s",
                    tc.expected, string(respBody),
                )
            }
        })
    }
```

我们首先定义一段测试用例。每个测试用例都包含配置名称、我们要发出的请求的路径以及预期的响应——所有字符串值。

我们通过调用 NewServeMux() 函数创建一个新的 ServeMux 对象。然后它使用创建的 mux 对象调用 setupHandlers() 函数。

接下来它调用 NewServer() 函数来启动服务器并传递创建的 ServeMux 对象 mux。此函数返回一个 httptest.Server 对象，其中包含启动的服务器的详细信息。我们感兴趣的是 URL 字段，它包含服务器的 IP 地址和端口组合。通常，这是 http://127.0.0.1:\<some port> 。

对 ts.Close() 函数的延迟调用可确保在测试函数退出之前彻底关闭服务器。

对于每个测试配置，我们使用 http.Get() 函数发出 HTTP GET 请求。服务器路径是通过连接 ts.URL 和路径字符串构建的。然后我们验证返回的响应正文是否与预期的响应正文匹配。

将代码清单 5.3 作为一个新文件 server_test.go 保存在代码清单 5.2 所在的目录中。运行测试：

```sh
$ go test -v
=== RUN   TestServer
=== RUN   TestServer/index
=== RUN   TestServer/healthcheck
--- PASS: TestServer (0.00s)
    --- PASS: TestServer/index (0.00s)
    --- PASS: TestServer/healthcheck (0.00s)
PASS
ok          github.com/practicalgo/code/chap5/http-serve-mux        0.577s
```

太好了。你已经编写了第一个 HTTP 服务器并学习了如何使用 httptest 包提供的工具对其进行测试。在接下来的章节中，你将学习测试更复杂的服务器应用程序的技术。接下来，你将了解有关请求结构的更多信息。

## 请求结构

HTTP 处理程序函数接受两个参数：类型为 http.ResponseWriter 的值和指向类型为 http.Request 的值的指针。 http.Request 类型的指针对象（在 net/http 包中定义）描述传入的请求。你会记得在第 4 章“高级 HTTP 客户端”中，这种类型也用于定义传出 HTTP 请求。请求是在 net/http 包中定义的结构类型。接下来描述与传入请求上下文相关的结构类型中的一些关键字段和方法。

### 方法
这是一个字符串，其值表示正在处理的请求的 HTTP 方法。在上一节中，你将此字段用于不同类型 HTTP 请求的专用处理函数。

### 网址
这是一个指向 url.URL 类型值的指针（在 net/url 包中定义），表示请求的路径。最好通过一个例子来理解。假设我们使用 URL http://example.com/api/?name=jane&age=25#page1 向我们的 HTTP 服务器发出请求。当处理程序函数处理此请求时，URL 对象的字段设置如下：

1. Path: /api/
2. RawQuery: name=jane&age=25
3. Fragment: page1

要访问特定的单个查询参数及其值，请使用 Query() 方法。该方法返回一个 Values 类型的对象，它被定义为 map[string][]string。对于上面的 URL，调用 Query() 方法将返回以下内容：

```go
url.Values{"age":[]string{"25"}, "name":[]string{"jane"}}
```

如果多次指定查询参数，例如 http://example.com/api/?name=jane&age=25&name=john#page1，Query() 的返回值将是

```go
url.Values{"age":[]string{"25"}, "name":[]string{"jane", "john"}}"
```

如果你的服务器接受 HTTP 基本身份验证，则请求 URL 的格式为 http://user:pass@example.com/api/?name=jane&age=25&name=john。在这种情况下，用户字段包含请求中指定的用户名和密码的详细信息。要获取用户名，请调用 User() 方法。要获取密码，请调用 Password() 方法。

> 注意: URL 结构包含其他字段，但以上是处理请求的上下文中唯一相关的字段。

### Proto、ProtoMajor 和 ProtoMinor

这些字段标识客户端和服务器通信所使用的 HTTP 协议。 Proto 是一个标识协议和版本的字符串（例如 "HTTP /1.1" ）。 ProtoMajor 和 ProtoMinor 是分别标识主要和次要协议版本的整数。对于 HTTP /1.1，主要和次要协议版本均为 1。

### 标题
这是一个 map[string][]string 类型的映射，它包含传入的标头。

### Host
这是一个字符串，其中包含客户端用来向服务器发出请求的主机名和端口组合 (example.com:8080) 或 IP 地址和端口组合 (127.0.0.1:8080)。

### body
这是一个 io.ReadCloser 类型的值，它指的是请求正文。你可以使用任何理解 io.Reader 接口的函数来读取正文。例如，你可以使用 io.ReadAll() 方法读取整个请求正文。该函数返回一个包含整个请求正文的字节切片，然后你可以根据处理程序的功能要求处理字节切片。在下一节中，你将看到如何在不将整个正文读入内存的情况下处理请求正文。

一个相关的字段是 ContentLength，它是可从请求正文中读取的最大字节数。

### Form, PostForm

如果你的处理程序函数正在处理 HTML 表单提交，那么你可以调用请求对象的 ParseForm() 方法而不是直接读取正文。调用此方法将读取请求并使用提交的表单数据填充 Form 和 PostForm 字段。根据用于提交表单的 HTTP 请求方法的类型，这两个字段的填充方式不同。如果使用 GET 请求提交表单，则只会填充表单字段。如果使用 POST、PUT 或 PATCH 方法提交表单，则会填充 PostForm 字段。这两个字段都是 url.Values 类型（在 net/url 包中定义），定义为 map[string][]string。因此，要访问任何表单字段，你将使用与访问地图中的键相同的方法。

### MultipartForm

如果你正在处理包含多部分/表单编码数据的表单上传，通常包含文件（如你在第 3 章中所做的那样），调用 ParseMultipartForm() 方法将读取请求正文并使用提交的数据填充 MultipartForm 字段。该字段的类型为 multipart.Form （在 mime/multipart 中定义）包：

```go
type Form struct {
    Value map[string][]string
    File  map[string][]*FileHeader
}
```

值包含提交的表单文本字段，文件包含与提交文件相关的数据。这个map中的key是表单字段名，文件相关的数据存放在一个FileHeader类型的对象中。 FileHeader 类型在 mime/multipart 包中定义如下：

```go
type FileHeader struct {
    Filename string
    Header   textproto.MIMEHeader
    Size     int64
}
```

字段及其说明如下：

- Filename ：包含上传文件的原始文件名的字符串值
- Header ：描述文件类型的 net/textproto 包中定义的 MIMEHeader 类型的值
- Size：一个 int64 值，包含文件的大小（以字节为单位）

FileHeader 还定义了一个方法 Open()，它返回一个 File 类型的值（在 mime/multipart 包中定义）。然后可以使用该值在处理程序函数中读取文件内容。通常，你需要访问传入 Request 对象中的某些字段以调试服务器应用程序中的问题。下一个练习让你有机会实现请求记录器。

> 练习 5.1：请求记录器 服务器中的一个有用功能是所有传入请求的日志。更新清单 5.2 中的应用程序，以便记录每个传入请求的详细信息。要记录的关键详细信息是 URL、请求类型、请求正文大小和协议。每个日志行都应该是一个 JSON 格式的字符串。

## 将元数据附加到请求

处理程序函数处理的每个传入请求都与上下文相关联。请求的上下文 r 可以通过调用 Context() 方法获得。此上下文的生命周期与请求的生命周期相同（见图 5.4）。

![](./images/5-4.png)

图 5.4：为每个传入请求创建一个上下文，并在请求处理完成时销毁。

上下文可以附加值。这对于关联特定于请求和请求范围的数据非常有用，例如，标识可以传递到应用程序不同部分的请求的唯一 ID。要将元数据附加到请求的上下文，我们需要执行以下操作：

1. 使用 r.Context() 检索当前上下文。
2. 使用 context.WithValue() 方法创建一个具有所需数据作为键值对的新上下文。

context.WithValue() 需要三个参数：

- 标识存储值的上下文的父上下文对象
- 标识数据键的接口对象{}
- 包含数据的 interface{} 对象

这实质上意味着你可以在上下文中存储什么完全由 WithValue() 函数的用户决定。但是，有一些约定需要遵守：

- 键不应是基本类型之一，例如 string 。
- 包应定义自己的自定义未导出结构类型以用作键。未导出的数据类型可确保不会在包外意外使用此类型。例如，定义一个空结构体，输入 requestContextKey struct{}。这确保不会在不同的包中意外使用相同的上下文键。
- 只有请求范围的数据应该存储在上下文中。

让我们看一个在开始处理请求之前附加请求标识符的示例。首先，我们将定义两种结构类型：key 的 requestContextKey 和 value 的 requestContextValue：

```go
type requestContextKey struct{}
type requestContextValue struct {
    requestID string
}
```

然后，我们将定义一个辅助函数来在请求的上下文中存储请求标识符：

```go
func addRequestID(r *http.Request, requestID string) *http.Request {
    c := requestContextValue{
        requestID: requestID,
    }
    currentCtx := r.Context()
    newCtx := context.WithValue(currentCtx, requestContextKey{}, c)
    return r.WithContext(newCtx)
}
```

接下来，在处理程序函数中，我们将在处理之前调用此函数来存储请求标识符：

```go
func apiHandler(w http.ResponseWriter, r *http.Request) {
    requestID := "request-123-abc"
    r = addRequestID(r, requestID)
    processRequest(w, r)
}
```

我们将定义第二个辅助函数来检索和记录 requestID ：

```go
func logRequest(r *http.Request) {
    ctx := r.Context()
    v := ctx.Value(requestContextKey{})

    if m, ok := v.(requestContextValue); ok {
        log.Printf("Processing request: %s", m.requestID)
    }
}
```

我们通过使用相应的键调用 ctx.Value() 方法来检索请求上下文的值。回想一下，我们在添加值时使用空的 requestContextKey 对象作为键。该方法返回类型为 interface{} 的对象。因此，我们对获得的值执行类型断言以确保它是 requestContextValue 类型。如果类型断言成功，我们会记录 requestID 。

然后 processRequest() 函数调用 logRequest() 函数来记录 requestID ：

```go
func processRequest(w http.ResponseWriter, r *http.Request) {
    logRequest(r)
    fmt.Fprintf(w, "Request processed")
}
```

清单 5.4 显示了一个可运行的服务器应用程序，它为每个请求附加一个请求标识符，然后在处理它之前记录它。

清单 5.4：将元数据附加到请求中

```go
// chap5/context-metadata/server.go
package main
 
import (
    "context"
    "fmt"
    "log"
    "net/http"
    "os"
)
 
type requestContextKey struct{}
type requestContextValue struct {
    requestID string
}
 
// TODO: Insert func addRequestID() function from earlier
// TODO: Insert logRequest() function from earlier
// TODO: Insert processRequest() function from earlier
// TODO: Insert apiHandler() function from earlier
 
func main() {
    listenAddr := os.Getenv("LISTEN_ADDR")
    if len(listenAddr) == 0 {
        listenAddr = ":8080"
    }
    mux := http.NewServeMux()
    mux.HandleFunc("/api", apiHandler)
    log.Fatal(http.ListenAndServe(listenAddr, mux))
}
```

创建一个新目录，chap5/context-metadata，并在其中初始化一个模块：

```sh
$ mkdir -p chap5/context-metadata
$ cd chap5/ context-metadata
$ go mod init github.com/username/context-metadata
```

接下来，将代码清单 5.4 保存为一个新文件 server.go。构建并运行服务器：

```sh
$ go build -o server
$ ./server
```

从另一个终端向服务器发出请求 curl localhost:8080/api 。

在你启动服务器的终端上，你将看到以下内容：

```2021/01/14 18:26:54 Processing request: request-123-abc```
在进一步处理传入请求之前，将请求 ID 等元数据附加到传入请求是一个很好的做法。但是，想象一下从所有处理程序函数调用 addRequestID() 函数。通过在服务器应用程序中实现中间件，你将在第 6 章“高级 HTTP 服务器应用程序”中学习将元数据附加到请求对象的更好方法。

## 处理流请求
在第 3 章中，你首先学习了在编写测试包服务器时如何解组 JSON 数据。除了 Unmarshal() 函数之外， encoding/json 包为我们提供了另一种更灵活的方法来解码 JSON 数据。让我们考虑一个充当日志收集器的 HTTP 服务器示例。它只做两件事：

- 它通过 HTTP POST 请求接收日志。请求正文包含编码为一个或多个 JSON 对象的日志。这通常称为 JSON 流，因为客户端将日志作为同一请求的一部分连续发送。
- 一旦成功解码它们，它就会打印这些日志。

服务器可能收到的一个示例请求体如下：

```json
{
    "user_ip": "172.121.19.21",
    "event": "click_on_add_cart"
}
{
    "user_ip": "172.121.19.21",
    "event": "click_on_checkout"
}
```

请注意我们如何将两个单独的日志编码为 JSON 对象，一个接一个。我们如何解组这个请求体？

你学习了使用以下步骤将传入请求 r 的 JSON 编码请求正文解组到对象 p 中：

1. 读取请求正文： data, err := io.ReadAll(r.Body) 。
2. 将 JSON 数据解码为对象： json.Unmarshal(data, &p) 。

如果请求正文描述单个 JSON 对象或一组 JSON 对象，则此方法有效。如果正文有多个 JSON 对象，如上述示例请求中那样怎么办？ Unmarshal() 将无法解码数据。为了能够成功解码上述数据，你必须查看 json.NewDecoder() 函数。

json.NewDecoder() 函数从实现 io.Reader 接口的任何对象读取。 NewDecoder() 函数不期望读取完整的 JSON 对象（或一组 JSON 对象），而是采用基于增量令牌的方法来读取数据。回忆上一节，请求对象的 Body 字段实现了 io.Reader 接口。因此，通过将请求正文直接提供给 NewDecoder() 函数，你可以动态解码 JSON 对象，而不需要 json.Unmarshal() 函数所需的所有可用数据。

让我们编写 HTTP 处理函数来成功处理发送到我们服务器的日志。首先，我们将定义结构类型以将单个日志条目解组为以下内容：

```go
type logLine struct {
    UserIP string `json:"user_ip"`
    Event  string `json:"event"`
}
```

接下来，我们编写处理程序函数：

```go
func decodeHandler(w http.ResponseWriter, r *http.Request) {
    dec := json.NewDecoder(r.Body)
    for {
        var l logLine
        err := dec.Decode(&l)
        if err == io.EOF {
            break
        }
        if err != nil {
            http.Error(w, err.Error(), http.StatusBadRequest)
            return
        }
        fmt.Println(l.UserIP, l.Event)
    }
    fmt.Fprintf(w, "OK")
}
```

我们通过调用 NewDecoder() 函数并将其传递给 r.Body 来初始化一个 json.Decoder 对象 dec。然后，在无限循环中，我们执行以下步骤：

1. 我们声明了一个 logLine 类型的对象 l，它将用于存储发送到服务器的单个解码日志条目。
2. 我们调用 dec 对象上定义的 Decode() 方法来读取 JSON 对象。 Decode() 函数将从 r.Body 中的读取器读取，直到找到第一个有效的 JSON 对象并将其反序列化为 l 对象。
3. 如果返回的错误是 io.EOF，则不再需要读取任何内容，因此我们中断循环。
4. 如果错误不是 nil 而是其他错误，我们停止任何进一步的处理并发回 HTTP Bad Request 错误响应，否则进入下一步。
5. 如果没有错误，我们打印对象的字段。
6. 返回步骤 1。

当循环退出时，一个 OK 响应被发送回客户端。

清单 5.5 显示了一个 HTTP 服务器，它使用前面显示的处理程序函数 decodeHandler 注册路径解码。

清单 5.5：使用 Decode() 解码 JSON 数据

```go
// chap5/streaming-decode/server.go
 
package main
 
import (
    "encoding/json"
    "fmt"
    "io"
    "net/http"
)
 
type logLine struct {
    UserIP string `json:"user_ip"`
    Event  string `json:"event"`
}
 
// TODO: Insert definition of decodeHandler() from earlier
 
func main() {
    mux := http.NewServeMux()
    mux.HandleFunc("/decode", decodeHandler)
    http.ListenAndServe(":8080", mux)
}
```

创建一个新目录，chap5/streaming-decode/，并在其中初始化一个模块：

```
$ mkdir -p chap5/streaming-decode
$ cd chap5/streaming-decode
$ go mod init github.com/username/streaming-decode
```

接下来，将代码清单 5.5 保存为一个新文件 server.go。构建并运行它：

```
$ go build -o server
$ ./server
```

在新的终端会话中，使用 curl 向服务器发出请求：

```sh
$ curl -X POST http://localhost:8080/decode \
-d '
{"user_ip": "172.121.19.21", "event": "click_on_add_cart"}
{"user_ip": "172.121.19.21", "event": "click_on_checkout"}
'
OK
```

在运行服务器的终端上，你应该看到以下输出：

```sh
172.121.19.21 click_on_add_cart
172.121.19.21 click_on_checkout
```

Decode() 函数将在两种情况下返回错误，以先发生者为准：

- 它在正在读取的 JSON 数据中遇到无效字符。这是位置相关的。匹配的} 字符之前的{ 字符（反之亦然）是无效字符。
- 正在读取的数据转换为特定对象时发生错误。

通过发出以下请求可以看到第一个场景的示例（注意第二个 JSON 对象之前的额外 {）：

```sh
$ curl -X POST http://localhost:8080/decode \
-d '
{"user_ip": "172.121.19.21", "event": "click_on_add_cart"}{{"user_ip": "172.121.19.21", "event": "click_on_checkout"}'
```

你将收到以下回复：

```invalid character '{' looking for beginning of object key string```

但是，在服务器端，你将看到以下内容：

```sh
172.121.19.21 click_on_add_cart
```

此输出告诉我们第一个 JSON 对象已成功解码。这主要是由于 Decode() 工作方式的性质——它会一直读取输入流，直到遇到错误。

现在让我们看看第二种错误。发出以下请求（注意第二个 JSON 对象中事件的错误数据类型）：

```sh
$ curl -X POST http://localhost:8080/decode \
-d '
{"user_ip": "172.121.19.21","event": "click_on_add_cart"}
{"user_ip": "172.121.19.21", "event": 1}
'
```

你将得到的回复如下：

```sh
json: cannot unmarshal number into Go struct field
logLine.event of type string
```

如果你想在你的处理函数中引入更多的健壮性，你想忽略解组错误并继续处理 JSON 流，你可以使用对 decodeHandler() 函数（突出显示）的小改动来做到这一点：

```go
func decodeHandler(w http.ResponseWriter, r *http.Request) {
    dec := json.NewDecoder(r.Body)

    var e *json.UnmarshalTypeError

    for {
        var l logLine
        err := dec.Decode(&l)
        if err == io.EOF {
            break
        }
        if errors.As(err, &e) {
            log.Println(err)
            continue
        }
        if err != nil {
            http.Error(w, err.Error(), http.StatusBadRequest)
            return
        }
        fmt.Println(l.UserIP, l.Event)
    }
    fmt.Fprintf(w, "OK")
}
```

当解组步骤出现错误时，会返回特定类型的错误 UnmarshalTypeError（在 encoding/json 中定义）包。因此，通过检查 Decode() 函数返回的错误是否属于这种类型，我们可以选择忽略解组错误并继续处理流的其余部分。通过上述更改，服务器将记录解组错误并继续处理流的其余部分。

要查看它的实际效果，请发出以下请求：

```sh
$ curl -X POST http://localhost:8080/decode \
-d '
{"user_ip": "172.121.19.21","event": "click_on_add_cart"}
{"user_ip": "172.121.19.21", "event": 1}
{"user_ip": "172.121.21.22", "event": "click_on_checkout"}'
OK%
```

在具有上述更改的服务器上，你将看到以下内容：

```sh
172.121.19.21 click_on_add_cart
2020/12/30 16:42:30 json: cannot unmarshal number into Go struct field logLine.event of type string
172.121.21.22 click_on_checkout
```

json.NewDecoder() 函数与 Decode() 方法结合使用是解析 JSON 数据的灵活方法。正如我们在此处看到的那样，它在解析 JSON 数据流时最有用。当然，你必须注意灵活性也给应用程序创建者带来了更多的错误处理责任。在下一个练习中，你将在应用程序中实现更强大的 JSON 解码，以拒绝包含任何未知字段的数据。

> 练习 5.2：更严格的 JSON 解码 如果你将以下请求正文发送到上面的 /decode 端点，Decode() 函数将忽略日志中的额外字段 user_data。
>
> ```json
> {
>     "user_ip": "172.121.19.21",
>     "event": "click_on_add_cart",
>     "user_data": "some_data"
> }
> ```

更新代码清单 5.5，如果在 JSON 流中指定了未知字段，Decode() 函数现在将抛出错误。

## 流数据作为响应

你已经了解了如何使用 fmt.Fprintf() 等函数向服务器发送响应。你还看到了如何使用 w.Headers().Add() 为 http.ResponseWriter 对象 w 设置自定义标头。在本节中，你将了解当你没有所有可作为响应发送的数据而你希望在数据可用时发送时发送响应。这通常称为流响应。一个可能发生这种情况的示例场景是，当一个长时间运行的作业作为客户端请求的一部分被触发，并且随着更多数据可用时，处理结果作为响应发送（参见图 5.5）。

![](./images/5-5.png)

图 5.5：从左到右：传入的 HTTP 请求触发长时间运行的作业。作业处理的结果在可用时发送。

数据生产者创建一个连续的字节流，由数据读取器读取并作为 HTTP 响应发送。这一直持续到数据生产者停止生产数据。我们如何有效地将数据从生产者获取到消费者？ io.Pipe() 函数提供了一种方法。调用此函数会返回两个对象：io.PipeReader 和 io.PipeWriter。数据生产者将写入 io.PipeWriter 对象，数据消费者将读取 io.PipeReader 对象。

考虑一个示例数据生成器函数 longRunningProcess()，它每秒生成一行日志，总共生成 21 行日志：

```go
func longRunningProcess(logWriter *io.PipeWriter) {
    for i := 0; i <= 20; i++ {                
        fmt.Fprintf(
            logWriter,
            `{"id": %d, "user_ip": "172.121.19.21", "event": "click_on_add_cart" }`, i,
        )
        fmt.Fprintln(logWriter)
        time.Sleep(1 * time.Second)
    }
    logWriter.Close()
}
```

使用 io.PipeWriter 对象调用该函数，该对象是写入日志的位置。由于该对象实现了 io.Writer 接口，因此我们可以使用 Fprintf() 函数向其写入字符串。在从函数返回之前，我们调用 Close() 方法关闭 io.PipeWriter 对象。

接下来，让我们看看将处理传入请求的 HTTP 处理程序函数；也就是说，它将启动长时间运行的作业，从数据生产者读取数据，并将数据流式传输到客户端：

```go
func longRunningProcessHandler(
    w http.ResponseWriter, r *http.Request) {

    done := make(chan struct{})
    logReader, logWriter := io.Pipe()
    go longRunningProcess(logWriter)
    go progressStreamer(logReader, w, done)

    <-done
}
```

我们首先创建一个 struct{} 类型的无缓冲通道。我们将使用此通道从响应流函数中指示所有数据都已发送。然后，我们调用 io.Pipe() 函数返回一个 io.PipeReader 和一个 io.PipeWriter 对象。

接下来，我们生成一个 goroutine 来运行 longRunningProcess() 函数，使用 io.PipeWriter 对象调用该函数。然后，我们生成另一个 goroutine——在 progressStreamer() 函数中实现的数据读取器和流送器。最后，我们在退出处理函数之前等待完成通道上的数据可用。 progressStreamer() 函数定义如下：

```go
func progressStreamer(
    logReader *io.PipeReader, w http.ResponseWriter, 
    done chan struct{}) {

    buf := make([]byte, 500)

    f, flushSupported := w.(http.Flusher)

    defer logReader.Close()
    w.Header().Set("Content-Type", "text/plain")
    w.Header().Set("X-Content-Type-Options", "nosniff")

    for {
        n, err := logReader.Read(buf)
        if err == io.EOF {
            break
        }
        w.Write(buf[:n])
        if flushSupported {
            f.Flush()
        }
    }
    done <- struct{}{}
}
```

首先，我们创建一个缓冲区对象 buf 来存储 500 个字节。这是我们将在任何给定时间点从管道读取的最大数据量。

由于我们希望响应数据立即可供客户端使用，因此我们将在写入 ResponseWriter 后显式调用 ResponseWriter 对象的 Flush() 方法。但是，我们首先必须检查 ResponseWriter 对象 w 是否实现了 http.Flusher 接口。我们使用以下语句执行此操作：

```go
f, flushSupported := w.(http.Flusher)
```

如果 w 实现了 http.Flusher 接口，f 将包含一个 http.Flusher 对象并且 flushSupported 将设置为 true 。

接下来，我们设置了一个延迟调用，以确保在从函数返回之前关闭 io.PipeReader 对象。

我们设置了两个响应头。我们将 Content-Type 标头设置为 text/plain 以向客户端表明我们将发送纯文本数据。我们还将 X-Content-Type-Options 标头设置为 nosniff 以指示浏览器在向用户显示之前不要缓冲任何数据。

接下来，我们启动一个无限循环来从 io.PipeReader 对象读取数据。如果我们得到一个 io.EOF 错误，表明作者已经完成了对管道的写入，我们就会跳出循环。如果没有，我们调用Write()方法发送刚刚读取的数据。如果 flushSupported 为真，我们调用 http.Flusher 对象 f 上的 Flush() 方法。

循环完成后，我们将一个空的结构对象 struct{}{} 写入 done 通道。

清单 5.6 显示了一个 HTTP 服务器，它注册了一个路径，/job，并将 longRunningProcessHandler 注册为处理函数。

清单 5.6：流响应

```go
// chap5/streaming-response/server.go
package main
 
import (
    "fmt"
    "io"
    "log"
    "net/http"
    "os"
    "time"
)
 
// TODO Insert definition of longRunningProcess function
// TODO Insert definition of progressStreamer function
// TODO Insert definition of longRunningProcessHandler function
 
func main() {
    listenAddr := os.Getenv("LISTEN_ADDR")
    if len(listenAddr) == 0 {
        listenAddr = ":8080"
    }

    mux := http.NewServeMux()
    mux.HandleFunc("/job", longRunningProcessHandler)
    log.Fatal(http.ListenAndServe(listenAddr, mux))
}
```

创建一个新目录，chap5/streaming-response，并在其中初始化一个模块：

```sh
$ mkdir -p chap5/streaming-response
$ cd chap5/streaming-response
$ go mod init github.com/username/chap5/streaming-response
```

接下来，将代码清单 5.6 保存为一个新文件 server.go。构建并运行服务器：

```sh
$ go build -o server
$ ./server
```

打开一个新的终端会话并使用 curl 发出请求。你将看到每秒到达的响应：

```sh
$ curl localhost:8080/job
 
{"id": 0, "user_ip": "172.121.19.21", "event": "click_on_add_cart" }
{"id": 1, "user_ip": "172.121.19.21", "event": "click_on_add_cart" }
{"id": 2, "user_ip": "172.121.19.21", "event": "click_on_add_cart" }
{"id": 3, "user_ip": "172.121.19.21", "event": "click_on_add_cart" }
{"id": 4, "user_ip": "172.121.19.21", "event": "click_on_add_cart" }
…
{"id": 20, "user_ip": "172.121.19.21", "event": "click_on_add_cart" }
```

接下来，执行 curl 添加 --verbose 标志，你将看到以下响应头：

```sh
Content-Type: text/plain
X-Content-Type-Options: nosniff
Date: Thu, 14 Jan 2021 06:02:13 GMT
Transfer-Encoding: chunked
```

当然，我们在编写响应时设置了 Content-Type 和 X-Content-Type-Options 标头。 Transfer-Encoding: chunked 标头通过调用 Flush() 方法自动设置。这向客户端表明数据正在从服务器流式传输，并且它应该继续读取直到服务器关闭连接。

上面使用 io.Pipe() 说明的方法建立了与数据生产和消费过程的清晰分离。请务必注意，你不需要在所有情况下都创建 PipeReader 和 PipeWriter 对象来发送流响应。

如果你可以控制流数据生成过程，则可以通过写入 ResponseWriter 直接流数据。然后，你可以在两次写入调用之间定期调用 ResponseWriter 对象的 Flush() 方法。例如，如果你想发送一个大文件作为对用户请求的响应，你可以定期读取固定数量的字节并将其发送到客户端并重复该过程直到读取完整文件。在这种情况下，你甚至不必调用 Flush() 方法，因为部分文件对用户没有用。幸运的是，你不必自己做所有这些。 io.Copy() 函数允许我们在不编写任何自定义代码的情况下实现这一点。首先，打开文件进行阅读：

```go
f, err := os.Open(fileName)
defer f.Close()
```

要将数据作为 ResponseWriter 对象的响应流式传输， w ：

```sh
io.Copy(w, f)
```

这将以块的形式读取文件数据（从 Go 1.16 开始为 32 KB），并将数据直接写入 ResponseWriter。在本章的最后一个练习 5.3 中，你将使用此技术来实现文件下载服务器。

> 练习 5.3：文件下载服务器 实现一个 HTTP 服务器，该服务器将扮演文件下载服务器的角色。用户将能够通过 fileName 查询参数向 /download 路径请求指定文件名，然后取回文件内容。你的服务器应该能够查找文件的自定义目录。
>
> 确保正确设置 Content-Type 标头以指示文件内容。

## 概括
在第 3 章为客户端编写测试服务器时，你已经开始了编写 HTTP 服务器的旅程。在本章中，你将深入研究这一点。你了解了服务器如何处理传入请求、为什么使用 DefaultServeMux 是个坏主意，以及如何使用自己的 ServeMux 对象。你编写了处理程序函数来处理流数据，最后，你学会了在服务器中使用 goroutine 和通道来发送流响应。

在下一章中，你将继续进入构建生产就绪 HTTP 服务器应用程序的世界。