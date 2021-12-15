# 编写 HTTP 客户端

在本章中，你将学习编写可测试的 HTTP 客户端的构建块。你将熟悉关键概念——发送和接收数据、序列化和反序列化以及使用二进制数据。一旦掌握了这些概念，你将能够为你的服务的 HTTP API 编写独立的客户端应用程序和 Go 客户端，并将 HTTP API 调用作为服务到服务通信架构的一部分。随着本章的进展，你将通过实现这些功能和技术来增强 mync http 子命令。让我们开始吧！

## 下载数据
你可能熟悉 wget 和 curl 等命令行程序，它们适用于通过 HTTP 下载数据。让我们看看如何使用 net/http 包中定义的函数和类型来编写一个。首先，让我们编写一个函数，它接受一个 HTTP URL 作为参数，并返回一个包含 URL 内容和错误值的字节片：

```go
func fetchRemoteResource(url string) ([]byte, error) {
    r, err := http.Get(url)
    if err != nil {
        return nil, err
    }
    defer r.Body.Close()
    return io.ReadAll(r.Body)
}
```

net/http 包中定义的 Get() 函数向指定的 url 发出 HTTP GET 请求，并返回一个 Response 类型的对象和一个错误值。 Response 对象 r 有多个字段，其中之一是 Body 字段（类型 io.ReadCloser ），其中包含响应正文。我们使用 defer 语句通过在函数返回之前调用 Close() 方法来关闭主体。然后，我们使用 io 包中的 ReadAll() 函数读取主体 ( r.Body ) 的内容并返回我们从中获得的两个值——一个字节切片和一个错误值。让我们定义一个 main 函数来编写一个可构建的应用程序。完整的清单如清单 3.1 所示。

清单 3.1：一个基本的数据下载器

```go
// chap3/data-downloader/main.go
 
package main
 
import (
    "fmt"
    "io"
    "net/http"
    "os"
)
 
func fetchRemoteResource(url string) ([]byte, error) {
    r, err := http.Get(url)
    if err != nil {
        return nil, err
    }
    defer r.Body.Close()
    return io.ReadAll(r.Body)
}
 
func main() {
    if len(os.Args) != 2 {
        fmt.Fprintf(os.Stdout "Must specify a HTTP URL to get data from")
        os.Exit(1)
    }
    body, err := fetchRemoteResource(os.Args[1])
    if err != nil {
        fmt.Fprintf(os.Stdout, "%v\n", err)
        os.Exit(1)
    }
    fmt.Fprintf(os.Stdout, "%s\n", body)
}
```

main() 函数需要将 URL 指定为命令行参数并实现一些基本的错误处理。创建一个新目录，chap3/data-downloader，并在其中初始化一个模块：

```sh
$ mkdir -p chap3/data-downloader
$ cd chap3/data-downloader
$ go mod init github.com/username/data-downloader
```

接下来，将代码清单 3.1 保存到一个新文件 main.go 中。构建并运行应用程序：

```sh
$ go build -o application
$./application https://golang.org/pkg/net/http/
```

你会看到一堆 HTML 写入你的终端。事实上，如果你指定引用图像的 URL，你也会在屏幕上看到转储的图像数据。我们很快就会改善这种情况，但首先让我们谈谈测试我们的数据下载器。

### 测试数据下载器

考虑到数据下载器应用程序，测试需要验证 fetchRemoteResource() 函数是否可以成功返回指定 URL 上可用的数据。如果 URL 无效或不可访问，则应返回错误值。我们如何设置一个测试 HTTP 服务器来提供一些测试内容？ net/http/httptest 包中的 NewServer() 函数将帮助我们。以下代码片段定义了一个函数 startTestHTTPServer()，它将启动一个 HTTP 服务器，该服务器将对任何请求返回响应“Hello World”：

```go
func startTestHTTPServer() *httptest.Server {
    ts := httptest.NewServer(
        http.HandlerFunc(
            func(w http.ResponseWriter, r *http.Request) {
                fmt.Fprint(w, "Hello World")
            }))
    return ts
}
```

httptest.NewServer() 函数返回一个 httptest.Server 对象，其中包含代表创建的服务器的各种字段。该函数的唯一参数是一个类型为 http.Handler 的对象（你将在第 6 章“高级 HTTP 服务器应用程序”中了解更多相关内容）。这个处理程序对象允许我们为测试服务器设置所需的处理程序。在这种情况下，我们创建了一个包罗万象的处理程序，它将向任何 HTTP 请求返回字符串“Hello World”——而不仅仅是 GET 请求。隐式地，对该服务器的所有请求都将收到成功的 HTTP 200 状态。

使用上面的函数，我们现在可以编写我们的测试函数，如代码清单 3.2 所示。

清单 3.2：测试 fetchRemoteResource() 函数

```go
// chap3/data-downloader/fetch_remote_resource:test.go
package main
 
import (
    "fmt"
    "net/http"
    "net/http/httptest"
    "testing"
)
 
// TODO Insert definition of startTestHTTPServer() from above
 
func TestFetchRemoteResource(t *testing.T) {
    ts := startTestHTTPServer()
    defer ts.Close()

    expected := "Hello World"

    data, err := fetchRemoteResource(ts.URL)
    if err != nil {
        t.Fatal(err)
    }
    if expected != string(data) {
        t.Errorf("Expected response to be: %s, Got: %s", expected, data)
    }
}
```

测试函数首先调用 startTestHTTPServer() 来创建测试服务器。返回的对象 ts 包含与启动的测试服务器相关的数据。在延迟语句中调用 Close() 方法可确保在测试完成执行时停止服务器。返回的 ts 对象中的 URL 字段包含一个字符串值，表示服务器的 IP 地址和端口组合。这作为参数传递给 fetchRemoteResource() 函数。然后测试的其余部分验证返回的数据是否与预期的字符串“Hello World”匹配。将代码清单 3.2 保存到一个新文件 fetch_remote_resource:test.go，与代码清单 3.1 位于同一目录中。使用 go test 运行测试：

```sh
$ go test -v
=== RUN   TestFetchRemoteResource
--- PASS: TestFetchRemoteResource (0.00s)
PASS
ok          github.com/practicalgo/code/chap3/data-downloader        0.872s
```

非常好。你已经通过 HTTP 实现了一个基本的数据下载器，并且通过从远程 URL 下载数据并为其编写测试来确保它正常工作。

在本章的第一个练习练习 3.1 中，你将通过添加此功能来增强 mync 命令行应用程序。

> 练习 3.1：增强 HTTP 子命令以允许数据下载 在上一章中，我们实现了一个命令行应用程序 mync，它有两个子命令 http 和 grpc。但是，我们没有为这些命令实现任何功能。在本练习中，实现 http GET 子命令的功能。使用练习 2.2 的解决方案作为起点。

## 反序列化接收到的数据

我们编写的 fetchRemoteResource() 函数只是将下载的数据显示到终端。这可能对应用程序的用户没有任何作用，对于某些类型的数据，例如图像和非文本文件，它只会显示为垃圾。在大多数情况下，你可能希望对数据进行某种处理。这种处理通常称为解组或反序列化数据，它涉及将数据字节转换为应用程序可以理解的数据结构。然后，你可以在应用程序中对该数据结构执行任何操作，而无需查询或解析原始字节。逆向操作是编组或序列化，它是将数据结构转换为数据格式的有效方法，然后可以通过网络存储或传输。在本节中，我们将专注于解组数据。在下一节中，我们将把注意力转向数据的编组。

可以将某些字节的数据反序列化成的数据结构与数据的性质紧密耦合。例如，将与语言无关的 JavaScript 对象表示法 (JSON) 数据的字节解组为结构类型的切片是一种常见操作。类似地，将 Go 特定的 gob（由 encoding/gob 包定义）字节反序列化为结构类型是另一种反序列化操作。根据字节的数据格式，反序列化操作会有所不同。编码包及其子包支持解组（和编组）流行的数据格式，如 JSON、XML、CSV、gob 等。

让我们研究一个示例，说明如何将 JSON 格式的 HTTP 响应反序列化为地图数据结构。如果响应不是 JSON 格式，我们将不会执行反序列化。由 json.Unmarshal() 函数实现的反序列化操作要求我们指定我们希望将数据解组到的对象类型。因此，要编写这样的客户端：

1. 我们需要检查我们将反序列化的 JSON 数据。
2. 我们需要创建一个数据结构；即，能够表示数据的地图。

为了保持简单和独立，请考虑一个托管某些软件包的虚构 HTTP 服务器。它有一个 API，它返回一个 JSON 字符串，其中包含所有可用的包名称及其最新版本，如下所示：

```json
[
    {"name": "package1", "version": "1.1"},
    {"name": "package2", "version": "1.2"}
]
```

让我们看看我们将定义的类型来反序列化 JSON 数据。我们将调用 pkgData 类型，这是一种表示单个包的数据的结构类型：

```go
type pkgData struct {
    Name    string `json:"name"`
    Version string `json:"version"`
}
```

该结构有两个字符串字段：名称和版本。 struct标签`json:"name"`和"json:"version"`表示JSON数据中对应字段的键标识符。现在我们已经定义了数据结构，我们可以将包的JSON数据反序列化为一片 pkgData 对象。

fetchPackageData() 函数向包服务器 url 发送 GET 请求，并返回 pkgData 结构对象的切片和错误值。如果出现错误，或者数据无法反序列化，则返回一个空切片以及一个错误值（如果可用），如下所示：

```go
func fetchPackageData(url string) ([]pkgData, error) {
    var packages []pkgData
    r, err := http.Get(url)
    if err != nil {
        return nil, err
    }
    defer r.Body.Close()
    if r.Header.Get("Content-Type") != "application/json" {
        return packages, nil
    }
    data, err := io.ReadAll(r.Body)
    if err != nil {
        return packages, err
    }
    err = json.Unmarshal(data, &packages)
    return packages, err
}
```

我们虚构的包裹服务器还有一个 Web 后端，通过它它可以将包裹数据作为可通过浏览器查看的 HTML 页面提供。因此，在客户端代码中，如果响应主体被识别为 JSON 数据，我们只会尝试反序列化响应主体。 Content-Type HTTP 标头用于检测响应正文是否为 application/json。

响应标头可通过 Header 字段获得 - 响应对象中 [string][]string 类型的映射。因此，要获取特定标头的值，我们使用 Get() 方法将标头键指定为参数。

如果没有发现 Content-Type 的 header 值是 application/json，则返回空切片并返回 nil 错误。当然，你可以将应用程序设计为在此处返回错误。如果发现 Content-Type 是 application/json，我们使用 io.ReadAll() 函数读取正文。在一些标准错误处理之后，我们然后调用 json.Unmarshal() 函数指定要反序列化的数据和要反序列化的对象。

清单 3.3 显示了 pkgquery 包的完整实现：

清单 3.3：从包服务器查询数据

```go
// chap3/pkgquery/pkgquery.go
 
package pkgquery
 
import (
    "encoding/json"
    "io"
    "net/http"
    "time"
)
 
type pkgData struct {
    Name    string `json:"name"`
    Version string `json:"version"`
}
 
// TODO Insert definition of fetchPackageData() from earlier
```

创建一个新目录 chap3/pkgquery，并在其中初始化一个模块：

```sh
$ mkdir -p chap3/pkgquery
$ cd chap3/pkgquery
$ go mod init github.com/username/pkgquery
```

将代码清单 3.3 保存为文件 pkgquery.go 。

我们如何测试 pkgquery 包的功能？我们可以实现一个主包并查询虚构包服务器的实现。或者，我们可以实现一个测试 HTTP 服务器，它返回 JSON 格式的数据，如前所述。函数 startTestPackageServer() 实现了这样一个服务器：

```go
func startTestPackageServer() *httptest.Server {
    pkgData := `[
{"name": "package1", "version": "1.1"},
{"name": "package2", "version": "1.0"}
]`
    ts := httptest.NewServer(
        http.HandlerFunc(
            func(w http.ResponseWriter, r *http.Request) {
                w.Header().Set("Content-Type", "application/json")
                fmt.Fprint(w, pkgData)
            }))
    return ts
}
```

实现测试服务器后，代码清单 3.4 显示了完整的测试功能。

清单 3.4：测试 pkgquery

```go
// chap3/pkgquery/pkgquery_test.go
 
package pkgquery
 
import (
    "fmt"
    "net/http"
    "net/http/httptest"
    "testing"
    "time"
)
 
// TODO Insert definition of startTestPackageServer() from earlier
 
func TestFetchPackageData(t *testing.T) {
    ts := startTestPackageServer()
    defer ts.Close()        
    packages, err := fetchPackageData(ts.URL)
    if err != nil {
        t.Fatal(err)
    }
    if len(packages) != 2 {
        t.Fatalf("Expected 2 packages, Got back: %d", len(packages))        
    }        
}
```

我们通过调用 startTestPackageServer() 函数来启动测试 HTTP 服务器。然后我们通过调用 createHTTPClientWithTimeout() 函数创建一个 HTTP 客户端对象。接下来，我们调用 fetchPackageData() 函数，将 HTTP 客户端对象、客户端和要请求的 URL 作为参数传递。

最后，我们向服务器发送 GET 请求，服务器将返回 JSON 包数据。然后我们断言返回了一个 nil 错误，我们获得了一个包含两个元素的切片，这些元素对应于两个 pkgData 对象。

将清单 3.4 保存到一个新文件 pkgquery_test.go 中，该文件与 pkgquery.go 位于同一目录中。运行测试：

```sh
$ go test -v
=== RUN   TestFetchPackageData
--- PASS: TestFetchPackageData (0.00s)
PASS
ok          github.com/practicalgo/code/chap3/pkgquery/        0.511s
```

正如预期的那样，测试通过了。在更实际的场景中，你可能正在编写一个 HTTP 客户端来使用来自其他人服务器的数据。因此，你必须执行以下关键步骤：

1. 查找第三方服务器的 JSON API 架构。
2. 构造数据结构以将响应数据反序列化为。
3. 使用步骤 1 来实现测试服务器，以便你可以实现完全可测试的 HTTP 客户端。

你看到了如何使用 Content-Type 标头来决定是反序列化数据还是忽略它。你还可以使用它来决定数据在终端中是否可读，或者是否需要使用专用软件（例如图像查看器或 PDF（便携式文档格式）阅读器）来读取数据。在练习 3.2 中，你将在 http 子命令中实现支持，以便用户能够将下载的数据写入文件。

> 练习 3.2：将下载的数据写入文件 为 http 子命令 -output 实现一个新选项，它将采用文件路径作为选项。当指定该选项时，下载的数据将写入文件而不是显示在终端上。

## 发送数据
让我们再次考虑包服务器。我们已经了解了如何通过发出 HTTP GET 请求来查看现有的包数据。现在假设我们要向服务器添加新的包数据。创建或向服务器注册新包涉及发送包本身（.tar.gz 文件）和一些元数据（名称和版本）。为简单起见，我们假设包服务器没有任何适当的状态管理，如果它包含正确格式的预期数据，它会成功响应所有请求。遵循 REST 规范时的 HTTP 协议允许我们使用 POST、PUT 或 PATCH 请求将数据发送到服务器。 POST 方法通常用于创建新资源。要使用 POST 方法注册新包，我们将执行以下操作：

1. 使用 net/http 包中定义的 Post 函数发出 HTTP POST 请求，http.Post(url, contentType, packagePayload)。 url 是将 POST 请求发送到的 URL， contentType 是一个字符串，其中包含标识请求的 Content-Type 标头值的值，最后， packagePayload 是一个 io.Reader 类型的对象，其中包含我们的请求正文想送。
2. 这里的关键步骤是在单个请求正文中发送二进制包数据和元数据。 HTTP Content-Type 标头 multipart/form-data 将在这里派上用场。

首先，让我们看看如何发送仅包含元数据作为 JSON 正文的 POST 请求。然后我们将进一步开发它以在 multipart/form-data 请求中发送包数据以及元数据。

回想一下我们用来描述包的 JSON 格式：

```json
{"name": "package1", "version": "1.1"}
```

我们将在注册新包时使用相同的 JSON 格式来描述元数据。包注册的结果也以 JSON 正文的形式发回，如下所示：

```json
{"id":"package1-1.1"}
```

相应的结构类型将是

```go
type pkgRegisterResult struct {
    ID string `json:"id"`
}
```

首先，让我们看看如何使用 HTTP POST 请求发送 JSON 正文：

```go
func registerPackageData(url string, data pkgData) (pkgRegisterResult, error) {
    p := pkgRegisterResult{}        
    b, err := json.Marshal(data)
    if err != nil {
        return p, err
    }
    reader := bytes.NewReader(b)
    r, err := client.Post(url, "application/json", reader)
    if err != nil {
        return p, err
    }
    defer r.Body.Close()

    // TODO Handle response from the server
    …
}
```

该函数有两个参数——url 是我们将向其发送请求的 HTTP 服务器 URL，而 data 是一个 pkgData 类型的对象，我们将其序列化为 JSON 并将作为请求正文发送。

我们创建了一个 pkgRegisterResult 类型的对象 p，当包注册成功时，它将被填充并作为响应返回。

在该函数中，我们首先使用 encoding/json 包中定义的 Marshal() 函数将 pkgData 对象转换为字节片——我们将作为请求正文发送的 JSON 对象。我们之前定义的 struct 标签将用作 JSON 对象键。给定一个 pkgData 对象，{"Name":"package1", "Version":"1.0"}，Marshal() 函数会自动将其转换为对应于 JSON 编码字符串的字节切片：{"name":"package1 ","version":"1.0"}。然后我们将使用 bytes 包中的 NewReader() 函数为这个字节切片创建一个 io.Reader 对象。创建 io.Reader 对象 reader 后，我们将调用 Post() 函数作为 http.Post(url, "application/json", reader) 。

如果我们得到一个非 nil 错误，我们将返回空的 pkgRegisterResult 对象和错误对象 err。如果我们得到服务器的成功响应，我们然后读取正文并将正文反序列化为 pkgRegisterResult 响应对象：

```go
func registerPackageData(url string, data pkgData) (pkgRegisterResult, error) {
    // Send the request to the server as earlier        
    respData, err := io.ReadAll(r.Body)
    if err != nil {
        return p, err
    }
    if r.StatusCode != http.StatusOK {
        return p, errors.New(string(respData))
    }
    err = json.Unmarshal(respData, &p)
    return p, err
}
```

如果我们没有得到由 HTTP 200 状态代码指示的成功响应，我们将返回一个包含响应正文的错误对象。否则，我们解组响应并返回 pkgRegisterResult 对象 p 和任何解组错误 err 。

我们将为我们的包注册代码创建一个新包 pkgregister。清单 3.5 显示了完整的代码。

代码清单 3.5：注册一个新包

```go
// chap3/pkgregister/pkgregister.go
package pkgregister
 
import (
    "bytes"
    "encoding/json"
    "io/ioutil"
    "net/http"
    "time"
)
 
type pkgData struct {
    Name    string `json:"name"`
    Version string `json:"version"`
}
 
type pkgRegisterResult struct {
    Id string `json:"id"`
}
 
func registerPackageData(url string, data pkgData) (pkgRegisterResult, error) {
    p := pkgRegisterResult{}
    b, err := json.Marshal(data)
    if err != nil {
        return p, err
    }
    reader := bytes.NewReader(b)
    r, err := client.Post(url, "application/json", reader)
    if err != nil {
        return p, err
    }
    defer r.Body.Close()        
    respData, err := io.ReadAll(r.Body)
    if err != nil {
        return p, err
    }
    if r.StatusCode != http.StatusOK {
        return p, errors.New(string(respData))
    }
    err = json.Unmarshal(respData, &p)
    return p, err
}
```

创建一个新目录 chap3/pkgregister，并在其中初始化一个模块：

```sh
$ mkdir -p chap3/pkgregister
$ cd chap3/pkgregister
$ go mod init github.com/username/pkgregister
```

将代码清单 3.5 保存为一个新文件 pkgregister.go。我们如何测试这一切是否有效？我们将采用与上一节中使用的方法类似的方法，并实现一个行为类似于我们真正的包服务器的测试服务器：

1. 实现一个 HTTP 处理函数来处理 POST 请求。
2. 执行解组操作以将传入的 JSON 正文转换为 pkgData 对象。
3. 如果解组操作出现错误，或者 pkgData 对象的 Name 或 Version 为空，则会向客户端返回 HTTP 400 错误。
4. 通过连接 Name 和 Version 并用 - 分隔它们来构建人工包 ID。
5. 创建一个 pkgRegisterResult 对象，指定在上一步中构造的 ID。
6. 编组对象，将内容头设置为 application/json，并将编组后的结果作为字符串返回作为响应。

接下来你可以将上述步骤的实现视为一个单独的处理程序函数，如下所示。 （你将在第 5 章中了解有关处理程序函数的更多信息。）

```go
func packageRegHandler(w http.ResponseWriter, r *http.Request) {
    if r.Method == "POST" {
        // Incoming package data
        p := pkgData{}

        // Package registration response
        d := pkgRegisterResult{}
        defer r.Body.Close()
        data, err := io.ReadAll(r.Body)
        if err != nil {
            http.Error(w, err.Error(), http.StatusInternalServerError)
            return
        }
        err = json.Unmarshal(data, &p)
        if err != nil || len(p.Name) == 0 || len(p.Version) == 0 {
            http.Error(w, "Bad Request", http.StatusBadRequest)
            return
        }
        d.ID = p.Name + "-" + p.Version
        jsonData, err := json.Marshal(d)
        if err != nil {
            http.Error(w, err.Error(), http.StatusInternalServerError)
            return
        }
        w.Header().Set("Content-Type", "application/json")
        fmt.Fprint(w, string(jsonData))
    } else {
        http.Error(w, "Invalid HTTP method specified", http.StatusMethodNotAllowed)
        return
    }
}
```

你可以在代码清单 3.6 中找到测试函数的实现。我们有两个测试——一个是测试我们将预期的包注册数据发送到包服务器所通过的愉快路径，另一个是我们通过它发送空的 JSON 正文。

代码清单 3.6：注册新包的测试

```go
// chap3/pkgregister/pkgregister_test.go
package pkgregister
 
// TODO Insert definition of packageRegHandler() from above
func startTestPackageServer() *httptest.Server {
    ts := httptest.NewServer(http.HandlerFunc(packageRegHandler))
    return ts
}
 
func TestRegisterPackageData(t *testing.T) {
    ts := startTestPackageServer()
    defer ts.Close()
    p := pkgData{
        Name:    "mypackage",
        Version: "0.1",
    }
    resp, err := registerPackageData(ts.URL, p)
    if err != nil {
        t.Fatal(err)
    }
    if resp.ID != "mypackage-0.1" {
        t.Errorf("Expected package id to be mypackage-0.1, Got: %s", resp.ID)
    }
}
 
func TestRegisterEmptyPackageData(t *testing.T) {
    ts := startTestPackageServer()
    defer ts.Close()
    p := pkgData{}
    resp, err := registerPackageData(ts.URL, p)
    if err == nil {
        t.Fatal("Expected error to be non-nil, got nil")
    }
    if len(resp.ID) != 0 {
        t.Errorf("Expected package ID to be empty, got: %s", resp.ID)
    }
}
```

将代码清单 3.6 保存到与代码清单 3.5 相同的目录中的新文件 pkgregister_test.go 中。运行测试：

```sh
% go test -v
=== RUN   TestRegisterPackageData
--- PASS: TestRegisterPackageData (0.00s)
=== RUN   TestRegisterEmptyPackageData
--- PASS: TestRegisterEmptyPackageData (0.00s)
PASS
ok          github.com/practicalgo/code/chap3/pkgregister        0.540s
```

在本节中，你学习了如何使用编组和解组技术发送和接收 JSON 数据。你现在将应用它来在 mync http 子命令（练习 3.3）中实现对 POST 请求的支持。

> 练习 3.3：增强 HTTP 子命令以发送带有 JSON 主体的 POST 请求 http 子命令仅支持 GET 方法。你在本练习中的任务是对其进行增强，使其能够发出 POST 请求并通过 -body 选项从命令行作为字符串接受 JSON 正文，或通过 -body-file 选项从文件接受 JSON 正文。对于测试，你可以像我们在本节中所做的那样实现一个测试 HTTP 服务器。

你在本节中学习的处理 JSON 数据的技术也适用于另一种流行的数据格式 XML，由 encoding/xml 包支持。回到注册新包，我们已经看到了如何将包名称和版本作为 JSON 格式的主体发送。但是，我们还没有看到如何同时发送包裹数据。让我们看看如何使用 multipart/form-data 内容类型。

## 使用二进制数据
multi-part/formdata HTTP 内容类型允许你发送包含键值对（例如 name=package1 和 version=1.1）以及其他数据（例如作为 HTTP 请求的一部分的文件内容）的正文。可以想象，在将它发送到服务器之前，创建这个主体需要做很多工作。

在我们学习如何创建 multipart/form-data 消息之前，让我们看看这样的消息是什么样的：

```
--91f7de347fb9749c83cea1d596e52849fb0a95f6698459e2baab1e6c1e22
Content-Disposition: form-data; name="name"
 
mypackage
--91f7de347fb9749c83cea1d596e52849fb0a95f6698459e2baab1e6c1e22
Content-Disposition: form-data; name="version"
 
0.1
--91f7de347fb9749c83cea1d596e52849fb0a95f6698459e2baab1e6c1e22
Content-Disposition: form-data; name="filedata"; filename="mypackage-0.1.tar.gz"
Content-Type: application/octet-stream
 
data
--91f7de347fb9749c83cea1d596e52849fb0a95f6698459e2baab1e6c1e22—
```

上面的消息包含三个部分。每个部分由一个边界字符串分隔，该字符串是随机生成的。这里的边界字符串是以 91f... 开头的行。破折号是 HTTP/1.1 规范的一部分。

消息的第一部分包含一个名为“name”且值为 mypackage 的表单字段。

第二部分包含名称为“version”且值为 0.1 的字段。

消息的第三部分包含名称为“filedata”的字段、名称为filename 的字段、值“mypackage-0.1.tar.gz”以及字段本身的值data。第三部分还包含一个 Content-Type 规范，指定内容为 application/octet-stream，表示非明文数据。当然，字符串数据是真正的非明文数据的占位符，例如图像或PDF文件。

标准库的 mime/multipart 包定义了读取和写入多部分主体的所有必要类型和方法。让我们看看如何创建一个包含包和元数据的多部分主体：

1. 使用字节缓冲区初始化 multipart.NewWriter() 类型的对象 mw。
2. 使用 mw.CreateFormField("name") 方法创建一个表单域对象 fw，域名为 "name" 。
3. 使用 fmt.Fprintf() 方法将表示字段值的字节写入写入器 mw 。
4. 对要创建的每个表单域重复步骤 2 和 3。
5. 使用 mw.CreateFormFile("filedata", "filename.ext") 方法创建一个字段 fw，字段名为 filedata 来存储文件的内容，文件名由 "filename.ext" 给出。
6. 使用 io.Copy() 方法将字节从文件复制到写入器 mw 。
7. 如果要发送多个文件，请使用相同的字段名称 ( "filedata" )，但使用不同的文件名。
8. 最后，调用 mw.Close() 方法。

让我们看看它在实际实现中的样子。首先，我们将更新 pkgData 结构以说明包内容：

```go
type pkgData struct {
    Name     string
    Version  string
    Filename string
    Bytes    io.Reader
}
```

Filename 字段将存储包的文件名，Bytes 字段是一个 io.Reader 指向打开的文件。

给定一个 pkgData 类型的对象，我们可以创建一个如清单 3.7 所示的多部分消息来“打包”数据。

清单 3.7：创建多部分消息

```go
// chap3/pkgregister-data/form:body.go
package pkgregister
 
import (
    "bytes"
    "io"        
    "mime/multipart"
)
 
func createMultiPartMessage(data pkgData) ([]byte, string, error) {
    var b bytes.Buffer
    var err error
    var fw io.Writer

    mw := multipart.NewWriter(&b)

    fw, err = mw.CreateFormField("name")
    if err != nil {
        return nil, "", err
    }
    fmt.Fprintf(fw, data.Name) 

    fw, err = mw.CreateFormField("version")
    if err != nil {
        return nil, "", err
    }
    fmt.Fprintf(fw, data.Version)

    fw, err = mw.CreateFormFile("filedata", data.Filename)
    if err != nil {
        return nil, "", err
    }
    _, err = io.Copy(fw, data.Bytes)
    err = mw.Close()
    if err != nil {
        return nil, "", err
    }

    contentType := mw.FormDataContentType()
    return b.Bytes(), contentType, nil
}
```

使用新的 bytes.Buffer 对象 b 调用 multipart.NewWriter() 方法以创建新的 multipart.Writer 对象 mw。然后，我们两次调用 CreateFormField() 方法来创建名称和版本字段。接下来，我们调用 CreateFormFile() 方法来插入文件内容。最后，我们通过调用 b.Bytes() 方法检索相应 multipart/form-data 消息的相关字节并返回它。我们还返回另外两个值，通过 multipart.Writer 对象的 FormDataContentType() 方法获得的内容类型和一个 nil 错误对象。

创建一个新目录 chap3/pkgregister-data，并在其中初始化一个模块：

```sh
$ mkdir -p chap3/pkgregister-data
$ cd chap3/pkgregister-data
$ go mod init github.com/username/pkgregister-data
```

接下来，将代码清单 3.7 保存为一个新文件 form:body.go 。

接下来，让我们看看 registerPackageData() 函数，它会调用 createMultiPartMessage() 函数来创建 multipart/form-data 负载：

```go
type pkgRegisterResult struct {
    Id       string `json:"id"`
    Filename string `json:"filename"`
    Size     int64  `json:"size"`
}
 
func registerPackageData(
    client *http.Client, url string, data pkgData,
) (pkgRegisterResult, error) {
    p := pkgRegisterResult{}
    payload, contentType, err := createMultiPartMessage(data)
    if err != nil {
        return p, err
    }
    reader := bytes.NewReader(payload)
    r, err := http.Post(url, contentType, reader)
    if err != nil {
        return p, err
    }
    defer r.Body.Close()
    respData, err := io.ReadAll(r.Body)
    if err != nil {
        return p, err
    }
    err = json.Unmarshal(respData, &p)
    return p, err
}
```

我们调用 createMultiPartMessage() 函数，该函数为我们提供多部分数据、有效负载和内容类型 contentType。然后，我们构造一个 io.Reader 对象以从有效负载中读取并通过调用 http.Post() 函数发送 HTTP POST 请求。之后，我们读取响应并将其解组到 pkgRegisterResult 对象中。请注意，我们在 pkgRegisterResult 结构中添加了两个新字段，以引用包的文件名和发送的文件的大小。这将使我们能够验证服务器端是否成功读取了数据。

清单 3.8 显示了 pkgregister 包的完整实现，它使用了 createMultiPartMessage() 函数。

清单 3.8：使用多部分消息的包注册

```go
// chap3/pkgregister-data/pkgregister.go
package pkgregister
 
import (
    "bytes"
    "encoding/json"
    "io"        
    "net/http"
    "time"
)
 
type pkgData struct {
    Name     string
    Version  string
    Filename string
    Bytes    io.Reader
}
 
type pkgRegisterResult struct {
    ID       string `json:"id"`
    Filename string `json:"filename"`
    Size     int64  `json:"size"`
}
 
// TODO Insert definition of registerPackageData() from earlier
 
func createHTTPClientWithTimeout(d time.Duration) *http.Client {
    client := http.Client{Timeout: d}
    return &client
}
```

将代码清单 3.8 保存为一个新文件 pkgregister.go，与代码清单 3.7 位于同一目录中。为了测试这个包，我们将实现一个测试服务器，它接受在 multipart/form-data 消息中发送的包注册数据，并返回一个 JSON 编码的响应。此测试服务器的关键功能将由为 http.Request 对象定义的 ParseMultipartForm() 方法实现。此方法将解析已编码为 multipart/form-data 消息的请求正文，并通过 mime/multipart 包中定义的 multipart.Form 类型的对象自动使嵌入的数据可用。该类型定义如下：

```go
type Form struct {
    Value map[string][]string
    File  map[string][]*FileHeader
}
```

Value 字段是一个映射对象，包含表单字段名称作为键和它们的值作为字符串切片。一个表单的一个字段名可以有多个值。 File 字段是一个映射，其键由字段名称组成，例如 filedata，对象切片将每个文件的数据表示为 FileHeader 对象。 FileHeader 类型在同一个包中定义如下：

```go
type FileHeader struct {
    Filename string
    Header   textproto.MIMEHeader
    Size     int64
}
```

字段名称是不言自明的。因此，这种类型的示例对象如下所示：

```go
{"Filename": "package1.tar.gz", "Header": map[string]string{"Content-Type":"application/octet-stream"}, "Size": "200"}
```

那我们如何获取文件数据呢？ FileHeader 对象定义了 Open() 方法，该方法返回一个 File 对象。然后可以使用它来读取存储在文件中的数据。让我们看看服务器处理函数：

```go
func packageRegHandler(w http.ResponseWriter, r *http.Request) {
    if r.Method == "POST" {
        d := pkgRegisterResult{}
        err := r.ParseMultipartForm(5000)
        if err != nil {
            http.Error(
                w, err.Error(), http.StatusBadRequest,
            )
            return
        }
        mForm := r.MultipartForm
        f := mForm.File["filedata"][0]
        d.ID = fmt.Sprintf(
            "%s-%s", mForm.Value["name"][0], mForm.Value["version"][0],
        )
        d.Filename = f.Filename
        d.Size = f.Size
        jsonData, err := json.Marshal(d)
        if err != nil {
            http.Error(w, err.Error(), http.StatusInternalServerError)
            return
        }
        w.Header().Set("Content-Type", "application/json")
        fmt.Fprint(w, string(jsonData))
    } else {
        http.Error(
            w, "Invalid HTTP method specified", http.StatusMethodNotAllowed,
        )
        return
    }
}
```

马上我们就看到了对 key 函数的调用：err := r.ParseMultipartForm(5000)，其中 5000 是将在内存中缓冲的最大字节数。如果我们得到一个非 nil 错误，我们会返回一个 HTTP 400 Bad Request 错误。如果没有，我们继续访问请求的 MultipartForm 属性中解析的表单数据。随后，我们访问表单的键值对和文件数据，构造包 ID，设置文件名和大小属性，将数据编组为 JSON 对象，并将其作为响应发送回来。好的，就是这样——是时候看看测试函数了，这样我们就可以测试我们的代码了。清单 3.9 显示了测试函数。

代码清单 3.9：使用多部分消息测试包注册

```go
// chap3/pkgregister-data/pkgregister_test.go
package pkgregister
import (
    "encoding/json"
    "fmt"
    "net/http"
    "net/http/httptest"
    "strings"
    "testing"
    "time"
)
 
// TODO Insert definition of packageHandler() from above
 
 
func startTestPackageServer() *httptest.Server {
    ts := httptest.NewServer(http.HandlerFunc(packageHandler))
    return ts
}
 
func TestRegisterPackageData(t *testing.T) {
    ts := startTestPackageServer()
    defer ts.Close()
    p := pkgData{
        Name:     "mypackage",
        Version:  "0.1",
        Filename: "mypackage-0.1.tar.gz",
        Bytes:    strings.NewReader("data"),
    }

    pResult, err := registerPackageData(ts.URL)
    if err != nil {
        t.Fatal(err)
    }

    if pResult.ID != fmt.Sprintf("%s-%s", p.Name, p.Version) {
        t.Errorf(
            "Expected package ID to be %s-%s, Got: %s", p.Name, p.Version, pResult.ID,
        )
    }
    if pResult.Filename != p.Filename {
        t.Errorf(
            "Expected package filename to be %s, Got: %s", p.Filename, pResult.Filename,
        )
        if pResult.Size != 4 {
            t.Errorf("Expected package size to be 4, Got: %d", pResult.Size)
        }
    }
}
```


将代码清单 3.9 保存为一个新文件 pkgregister_test.go，与代码清单 3.8 位于同一目录中并运行测试：

```sh
$ go test -v
=== RUN   TestRegisterPackageData
--- PASS: TestRegisterPackageData (0.00s)
PASS
ok          github.com/practicalgo/code/chap3/pkgregister-data        0.728s
```

mime/multipart 包包含在 HTTP 请求正文中读写二进制数据所需的一切。你学习了如何使用它从客户端应用程序发送文件。在最后的练习中，你将在 mync 命令行应用程序中实现对发送文件的支持。

> 练习 3.4：增强 HTTP 子命令以发送带有表单上传的 POST 请求 增强 http 子命令以实现新选项 -upload，这将允许将文件作为 POST 请求的一部分发送，包括或不包括任何其他数据。选项 -form-data 可用于指定要与文件一起发送的任何其他参数。一个示例调用如下：
>
> ```sh
> $ mync http POST -upload /path/to/file.pdf -form-data name=Package1 -form-data version=1.0.
> ```

## 概括

你通过学习如何从 HTTP URL 下载数据开始本章。然后，你了解了如何通过将响应中的数据字节反序列化为程序可识别的数据结构来处理它们。接下来，你学习了如何执行反向操作并将数据结构序列化为字节以作为 HTTP 请求正文发送。最后，你学习了如何使用 multipart/form-data 消息将任意文件作为 HTTP 请求主体发送和接收。在整个过程中，你编写了测试来验证客户的行为。

在下一章中，你将学习许多在构建生产就绪 HTTP 客户端时会发现有用的高级技术。