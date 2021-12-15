# 部署应用程序
在本附录中，我们将讨论管理应用程序的配置、分发和部署的策略。前景广阔，你遵循的具体策略通常取决于你将应用程序部署到的基础架构。我的目标绝不是详尽无遗。相反，我只是想为你提供通用指南。

## 管理配置

我们一直在使用命令行标志和环境变量来指定应用程序中的各种配置数据。配置数据是应用程序执行功能所需的那些信息。但是，用户不需要指定它们。例如，在附录 A，“使你的应用程序可观察”中，我们使用标志来指定指标服务器的地址。但是，metrics 服务器的地址不会随着用户的输入而变化，例如要上传的包名称或版本。事实上，在大多数情况下，你的应用程序的用户很高兴不需要指定它，除非它专门覆盖它。

同样，我们使用环境变量来指定应用程序中的非敏感和敏感配置数据。例如，第 11 章“使用数据存储”中介绍的数据库密码被指定为环境变量。命令行标志以及环境变量易于理解，并且不需要额外的库来支持你的应用程序。然而，随着你的应用程序的增长和配置数据的增加，你会发现你想要使用其他方式来配置你的应用程序，甚至组合使用多种方法，例如使用命令行标志、环境变量和文件。例如，对非敏感数据使用配置文件是一种简单的方法。这需要你编写额外的代码来读取配置文件并使它们可用于你的应用程序。对于敏感数据，例如密码，你可以继续使用环境变量。我们将看到使用以 YAML 数据格式编写的配置文件来指定我们在附录 A 中编写的包命令行客户端的配置的示例。我们将使用第三方包 https://pkg.go.dev/go.uber.org/config，支持读取 YAML 格式的数据文件，包括组合多个 YAML 数据源以及读取环境变量。我们为命令行客户端指定了四个关键的配置数据——日志记录级别、度量服务器的地址、Jaeger（分布式跟踪服务器）的地址以及要使用的身份验证令牌。我们可以在 YAML 格式的文件中指定这些关键信息，如下所示：

```sh
---
server:
  auth_token: ${X_AUTH_TOKEN}
telemetry:
  log_level: ${LOG_LEVEL:1}
  jaeger_addr: http://127.0.0.1:14268
  statsd_addr: 127.0.0.1:9125
```

该文件由两个顶级对象组成：服务器和遥测。服务器对象包含一个密钥 auth_token，它将通过环境变量 X_AUTH_TOKEN 指定，因为它被视为敏感数据。遥测对象定义了三个键：log_level、jaeger_addr 和 statsd_addr，分别包含 Jaeger 和 statsd 服务器的日志记录级别和收件人。 log_level 的值将被设置为 LOG_LEVEL 环境变量定义的值，如果指定了一个，否则默认为 1。日志级别的含义由 github.com/rs/zerolog 包确定，其中我们在附录 A 中使用过。

下面介绍了如何在应用程序中读取前面的数据。我们将首先定义三种结构类型，配置文件将被反序列化为：

```go
type serverCfg struct {
    AuthToken string `yaml:"auth_token"`
}

type telemetryCfg struct {
    LogLevel   int    `yaml:"log_level"`
    StatsdAddr string `yaml:"statsd_addr"`
    JaegerAddr string `yaml:"jaeger_addr"`
}
type pkgCliInput struct {
    Server    serverCfg
    Telemetry telemetryCfg
}
```

第一个结构类型 serverCfg 对应于 YAML 配置中的服务器对象。第二个结构体类型telemetryCfg对应YAML配置中的遥测对象，第三个结构体类型pkgCliInput对应完整的YAML配置。结构标签`yaml:“auth_token”`（和其他）用于指示相应的键名，因为它们将出现在 YAML 文件中。

接下来，我们将使用 go.uber.org/config 包读取包含格式为 YAML 的数据的配置文件：

```go
import uberconfig "go.uber.org/config"
 
provider, err := uberconfig.NewYAML(
    uberconfig.File(configFilePath),
    uberconfig.Expand(os.LookupEnv),
)
```

NewYAML() 函数接受一个或多个数据源。这里我们指定了两个来源。第一个是通过 configFilePath 变量指定路径的文件。第二个是 Expand() 函数，它接受一个具有此签名的函数作为参数：func(string) (string, bool)。这里我们指定标准库的 os.LookupEnv 函数作为调用 Expand() 函数时的参数。结果是我们结合了环境变量和配置文件来为应用程序创建合并的配置。值得注意的是，解析对象值的能力，例如 ${X_AUTH_TOKEN} 和 ${LOG_LEVEL:1}，是由 go.uber.org/config 包的 Expand() 函数实现的。

如果 NewYAML() 函数调用返回 nil 错误，我们现在准备读取数据。 以下代码片段将读取数据并尝试将其反序列化为 pkgCliInput 类型的对象：

```go
c := pkgCliInput{}
if err := provider.Get(uberconfig.Root).Populate(&c); err != nil {
    return nil, err
}
```

如果前面的操作成功，我们就可以对读取的数据执行任何验证，例如：

```go
if c.Telemetry.LogLevel < -1 || c.Telemetry.LogLevel> 5 {
    return nil, errors.New("invalid log level")
}
```

你可以在本书源代码库的 appendix-b/command-line-app 目录中使用上述读取应用程序配置的逻辑找到 pkgcli 的修改版本。特别地，在 main.go 文件中，你将找到一个实现上述逻辑的函数 readConfig() 和一个示例 config.yml 文件。

如果你想使用不同的文件格式以及其他方式来读取配置数据，github.com/spf13/viper 包值得探索。此外，如果你希望与云提供商的配置和机密管理服务集成，那么 Go Cloud Development Kit 项目提供的支持也值得研究。查看 runtimevar (https://gocloud.dev/howto/runtimevar/) 和 secrets (https://gocloud.dev/howto/secrets/) 的文档。

## 分发你的应用程序

分发 Go 应用程序通常意味着分发构建的二进制文件，而不管分发机制如何。默认情况下，运行 go build，应用程序二进制格式对应于构建环境的操作系统和架构。 Go 构建工具识别环境变量 GOOS 和 GOARCH 以构建特定操作系统和架构的二进制格式。因此，如果你正在构建供其他人运行的应用程序，你将需要为每个操作系统和硬件组合分发二进制文件。你可以通过指定 GOOS 和 GOARCH 环境变量来实现。这些当前识别各种组合，例如 linux 和 arm64（适用于在 64 位 ARM 架构上运行的 Linux）、windows 和 amd64，适用于在 AMD 或 Intel 64 位处理器上运行的 Windows，等等。如果要从 macOS 或 Linux 系统构建 Windows AMD64 二进制文件，请按如下方式运行 go build 命令：

```sh
$ GOOS=windows GOARCH=amd64 go build -o application.exe
```

构建的 application.exe 现在可以复制到另一台运行 Microsoft Windows 64 位操作系统的计算机上，并且可以从那里运行。

除了二进制文件和配置文件之外，你可能还需要为你的 Web 应用程序分发其他文件，例如模板或静态资产。你可以使用标准库的 embed 包，而不是手动复制这些，从而使分发变得复杂，它允许你将文件嵌入到构建的应用程序中。例如，考虑以下代码片段：

```sh
import _ "embed"
//go:embed templates/main.go.tmpl
var tmplMainGo []byte
```

构建包含此代码片段的应用程序时，变量 tmplMainGo（一个字节片）将包含文件 templates/main.go.tmpl 的内容。因此，当你运行应用程序时，该文件不需要存在，因为它已嵌入到应用程序中。当然，这会导致应用程序可执行文件的大小增加，因此请注意你嵌入的文件。

一旦你构建了应用程序，分发机制是另一个必须解决的问题。近年来，由 Docker 容器实现的容器镜像变得流行并且使得分发变得非常方便。构建 Docker 容器镜像的第一步是创建一个 Dockerfile，它是构建应用程序的一系列指令，然后将构建的应用程序复制到操作系统镜像中，无论是 Linux 还是 Windows。以下 Dockerfile 将构建一个包含命令行应用程序 pkgcli 和配置文件的映像：

```sh
FROM golang:1.16 as build
WORKDIR /go/src/app
COPY . .
RUN go get -d -v ./…
RUN go build
 
FROM golang:1.16
RUN useradd --create-home application
WORKDIR /home/application
COPY --from=build /go/src/app/pkgcli .
COPY config.yml .
USER application
ENTRYPOINT ["./pkgcli"]
```

在文件的第一个块中，我们构建了应用程序。在第二个块中，我们创建一个包含构建的应用程序和 config.yml 文件的新图像。有多种策略可以确保生成的最终图像尺寸更小。我们可以不使用 golang:1.16 作为基础镜像，而是使用特殊的临时基础镜像或 https://github.com/GoogleContainerTools/distroless 项目提供的镜像之一。最后，我们将映像的 ENTRYPOINT 设置为 pkgcli，这是构建的应用程序二进制文件的名称。要构建镜像，请将 Dockerfile 保存在你要构建的应用程序目录的根目录中，然后按如下方式运行 Docker 构建：

```sh
$ docker build -t practicalgo/pkgcli .
```

此命令将构建一个名为 actualgo/pkgcli 的 Docker 镜像。构建镜像后，使用 docker push 命令将最终镜像推送到容器镜像注册表中。然后，任何想要使用它的人都可以从注册表中提取映像并按如下方式运行它：

```sh
$ docker run -v /data/packages:/packages \
        -e X_AUTH_TOKEN=token-123 -ti practicalgo/pkgcli register \
        -name "test" -version 0.7 -path packages/file.tar.gz \
        http://127.0.0.1:8080
```

你可以看到我们如何使用 docker run 命令的 -e 标志指定环境变量。我们使用 -v 标志来指定卷安装；也就是说，我们正在从容器内的主机系统挂载一个目录。在这里，我们正在挂载包含我们要上传的包的目录。你可以在本书源代码存储库的 appendix-b/command-line-app 目录中找到包含 Dockerfile 和应用程序的完整示例。通常，你会使用 Docker 映像来分发服务器应用程序，但如果你的命令行应用程序想要分发默认配置文件，Docker 映像是一种方便的方法。

## 部署服务器应用程序

将 HTTP 或 gRPC 服务器部署到公共 Internet 或供他人使用的内部网络时，你应该运行应用程序的多个实例。你将直接在虚拟机上运行服务器，或者将其作为容器运行，也许借助自制解决方案或编排系统（例如 Nomad 或 Kubernetes）。

然后，你应该配置一个接收请求并将它们转发到你的应用程序的负载均衡器。重要的是，负载平衡器-应用程序通信被很好地理解。

负载均衡器如何知道你的应用程序实例已准备好接收新请求？运行状况检查是负载均衡器检查应用程序实例运行状况的常用方法。因此，通常在你的应用程序中定义专用的 HTTP 端点或 gRPC 方法，负载均衡器可以定期向其发出请求以了解你的应用程序的健康状况。其他现代软件，例如服务网格，也扮演与传统负载均衡器相同的角色，它也会探测应用程序的健康状况，并将其作为流量是否转发到应用程序实例的决策因素。如果应用程序实例的运行状况检查失败，它将停止接收新请求，并且可以自动删除它并使用各种基础设施功能创建一个新请求。通常定义两类健康检查：健康检查和深度检查。第一次检查的成功响应仅确认应用程序本身能够响应检查。第二类检查更为复杂，因为它将确保应用程序的依赖项（例如另一个服务或数据库）也是可访问的，从而排除诸如网络故障或错误凭证等问题。深度检查应该不那么频繁地运行，也许只在启动时运行，因为它的目标是捕获的那些情况很可能在开始时就被捕获。

此外，我们在本书的各个章节中详细讨论过的超时配置必须得到足够的关注。必须特别注意云提供商的负载平衡器或你正在使用的其他类似软件的超时配置，因为它们决定了在终止连接之前等待响应的时间。当你使用长连接（例如在 gRPC 流通信期间）时，这也值得引起应有的注意。

在第 7 章和第 10 章中，我们讨论了如何使用 TLS 证书为 HTTP 和 gRPC 应用程序加密客户端和服务器之间的通信。当你在负载均衡器后面运行应用程序实例时，通常会在负载均衡器上终止来自客户端的 TLS 连接，因此负载均衡器和应用程序之间的通信是未加密的。这很常见，因为这样做很简单，并且存在一种错误的安全感，因为在大多数情况下，此流量位于组织的专用网络中。但是，值得强调的是，这是不推荐的，你必须渴望保护所有网络通信。我之前提到的一类软件，服务网格，在这方面通常很有用，因为它们自动确保加密的网络通信，而应用程序作者不需要做任何额外的工作。

## 概括

在本附录中，我们回顾了部署 Go 应用程序时的三个关键问题：管理配置、分发应用程序本身，然后部署服务器应用程序。你将采取的确切步骤将取决于你将应用程序部署到的基础设施，但希望所讨论的策略将为你提供一个良好的起点来进一步探索。