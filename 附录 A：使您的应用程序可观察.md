# 使你的应用程序可观察
在本附录中，我将提供一些关于实施技术的指南，以使你的应用程序行为可通过遥测数据观察到。这些数据通常分为日志、指标和跟踪。所有这三种数据都可以帮助你了解应用程序正在做什么，并帮助你回答有关特定时间点其内部状态的不同类型的问题。首先，我们将介绍可用于从应用程序发出的遥测数据的类别以及可帮助你实现它们的 Go 包。然后，我们将查看有关如何将它们集成到应用程序中的示例。

## 日志、指标和跟踪

到目前为止，在本书中，我们使用了标准库日志包中定义的函数，例如 Printf() 和 Fatal() 来记录来自我们应用程序的消息。这种日志记录技术易于实现，并且比根本没有任何日志记录要好。你可以使用日志中的文本在日志系统中搜索日志，但是当你想要搜索日志中的特定数据时，这变得不太有用。一个非常常见的操作是搜索具有与请求相关的特定上下文或元数据的日志——例如，特定命令、特定 HTTP 路径或 gRPC 方法的所有日志。使用我们迄今为止使用的日志记录技术，诸如此类的搜索既昂贵又低效。因此，你必须找到一种方法来从你的应用程序发出日志，以便每个日志行都符合特定的结构——单独标记的字段包含日志数据以及作为元数据的上下文信息。有多种方法可以实现这种日志记录机制，它们通常都称为结构化日志记录。日志行不是自由格式的文本，而是由数据字段组成，通常是键值对或 JSON 格式的字符串。

在第 6 章“高级 HTTP 服务器应用程序”，代码清单 6.2 中，我们定义了一个日志中间件 (loggingMiddleware()) 来发出日志行，如下所示：

```go
config.Logger.Printf(
    "protocol=%s path=%s method=%s duration=%f status=%d",
    r.Proto, r.URL.Path, r.Method,
    time.Now().Sub(startTime).Seconds(),
    customRw.code,
)
```

从该应用程序发出的日志行由键值对组成，以空格分隔，例如，protocol=HTTP path=/api duration=0.05 status=200。这是对简单记录一行的改进，例如接收到 /api/search 的 HTTP 请求。响应 200。持续时间：0.1 秒。大多数日志系统都内置支持以这种格式解析和索引日志行，因此每个键值对都可以单独搜索。在 Stripe 发表了一篇题为“规范日志行”(https://stripe.com/blog/canonical-log-lines) 的博客文章后，这种格式在社区中被称为 logfmt。使用 Printf() 或任何日志包的函数构建上述日志行很麻烦。因此，你可以使用第三方包 https://github.com/go-logfmt/logfmt 来构建格式为键值对的日志行。该包仅实现了编码器和解码器，因此你仍将使用标准库的日志记录功能来记录格式化的日志行。另一个第三方包 https://github.com/apex/log 支持以 logfmt 样式发出日志，你可以使用它代替标准库的日志包。

实现结构化日志记录的另一种方法是使用将日志行作为 JSON 编码字符串发出的库。在这种格式中，前面的示例日志行将如下所示：{“protocol”:“HTTP”,“path”:“api”,“duration”: 0.05, “status”: 200}。这可能是最流行的结构化日志格式，并且有几个选项可以实现这种格式。最早的软件包之一是 https://github.com/sirupsen/logrus，它实现了一个与标准库的 log.Logger 类型实现的 API 完全兼容的 API。近年来，还开发了其他软件包，例如 https://github.com/uber-go/zap 和 https://github.com/rs/zerolog，它们提供了更多功能和更好的性能。在下一节中，你将看到如何将 github.com/rs/zerolog 包集成到你的应用程序中。我们选择它而不是 zap，因为它实现了一个更简单的 API。

接下来，我们将讨论如何从应用程序导出指标。

指标是你从应用程序计算和发布的数字，用于量化应用程序的各种行为。此类行为的示例包括从命令行应用程序执行命令所花费的时间、HTTP 请求或 gRPC 方法调用的延迟测量以及执行数据库操作所花费的时间。

度量通常是三个关键类别之一：计数器、仪表或直方图。计数器指标类型用于其值为整数且单调递增的测量值——例如，应用程序在其生命周期内服务的请求数。计量指标用于可能随时间增加或减少的测量值，它可以采用整数或浮点值 - 例如应用程序的内存使用量或应用程序每秒提供的请求数。直方图指标用于记录观察结果，例如请求的延迟。与仪表度量相比，直方图度量通常用于通过将度量值分组到桶中并启用诸如任意百分位值之类的计算来进行分析。

一旦应用程序计算出指标，它们要么被发送到外部监控系统（推送模型），要么监控系统将从你的应用程序读取数据（拉取模型）。一旦数据存储在监控系统中，你就可以查询它们，执行各种统计操作，并配置警报。

从历史上看，应用程序作者不得不使用供应商特定的库来将监控数据提供给监控系统。近年来，OpenTelemetry 项目 (https://opentelemetry.io/) 的发展使应用程序作者能够以供应商中立的方式导出指标，包括商业供应商。指标在哪里以及如何与应用程序的关注点分离。当你更改监控系统时，你的应用程序代码保持不变。话虽如此，在撰写本书时，OpenTelemetry Go 社区 (https://github.com/open-telemetry/opentelemetry-go) 已决定优先考虑指标支持的开发，以专注于跟踪支持。因此，即使目前存在支持，我们也将避免使用它。相反，我们将使用 https://github.com/DataDog/datadog-go 以 statsd (https://github.com/statsd/statsd) 开源监控解决方案定义的格式直接从应用程序导出指标.即使你的组织没有直接使用 statsd，正在使用的监控解决方案也很有可能支持读取 statsd 指标格式。

接下来，我们将讨论如何从应用程序导出跟踪。

跟踪是跟踪系统中事务的遥测数据。当一个请求进来时，作为处理该请求的一部分，通常会发生不止一个动作。跟踪的生命周期与应用程序中事务的生命周期相同。事务处理期间发生的每个动作或事件都将启动一个跨度。因此，跟踪由一个或多个跨度组成，可能跨越系统边界——例如，多个服务和数据库。与事务相关的所有跨度将共享一个跟踪标识符，因此通过使用跟踪系统，你可以直观地分析作为事务一部分执行的各种操作的延迟和成功/错误值。指标告诉你事务缓慢，而跟踪则为你提供有关事务缓慢原因的更详细信息。

例如，考虑我们在第 11 章“使用数据存储”中实现的包服务器。上传包是一个事务，它由两个不同的操作组成：将包上传到对象存储服务并将包元数据更新到关系数据库。这些操作中的每一个都会发出一个跟踪，其中包含有关操作的详细信息和操作所花费的时间。由于两个跟踪共享事务标识符，你可以看到事务的整体延迟以及每个组成操作的延迟。这在面向服务的体系结构中最有用，在这种体系结构中，在单个事务期间发生多个服务调用。跟踪数据的存储和分析需要专门的系统，从历史上看，你必须使用供应商特定的库来向他们发送跟踪数据。但是，OpenTelemetry 项目的 Go 库使得以供应商中立的方式实现跟踪成为可能。在撰写本文时，该项目 (https://github.com/open-telemetry/opentelemetry-go) 已发布 1.0.0-RC2 版本。我们将使用此版本的库，并且仅使用与导出跟踪相关的功能。

在下一节中，你将了解一些用于修改我们在书中编写的应用程序的模式，以便它们发出遥测数据。

## 发射遥测数据
我创建了一个自定义命令行客户端 pkgcli，用于与第 11 章中实现的包服务器交互。我还修改了包服务器以与 gRPC 服务器通信以验证上传者的详细信息。你可以在本书源代码库的 appendix-a 目录中找到所有相关代码和说明。日志、指标和跟踪的存储需要专门的系统，并且有许多开源和商业解决方案。下面的示例将仅演示从应用程序发出相关数据，而不是这些数据的可视化和分析。你可以在文件 appendix-a/README.md 中找到有关运行演示命令行应用程序和服务器的完整说明。

### 命令行应用程序

对于命令行应用程序，在执行任何命令之前，我们将配置日志记录、初始化网络客户端以在初始化期间导出指标和跟踪。然后，我们将这些初始化配置提供给应用程序的其余部分，以便在执行任何命令期间记录任何消息、发布指标或导出跟踪。示例命令行应用程序 pkgcli 的代码位于 appendix-a/command-line-app 目录中。它使用 flag 包，并应用第 2 章“高级命令行应用程序”中讨论的子命令架构来创建具有两个子命令（注册和查询）的应用程序。第一个子命令允许用户将包上传到包服务器，第二个命令允许他们从服务器查询包信息。我们创建了一个配置包，在其中包含一个结构 PkgCliConfig 来封装初始化的日志配置、指标和跟踪客户端：

```go
type PkgCliConfig struct {
    Logger  zerolog.Logger
    Metrics telemetry.MetricReporter
    Tracer  telemetry.TraceReporter
}
```

我们创建了一个遥测包，其中包含三个函数：InitLogging()、InitMetrics() 和 InitTracing()，它返回一个初始化的 zerolog.Logger、telemetry.MetricReporter 和 Telemetry.TraceReporter 对象。最后两个自定义类型被定义为分别封装用于发布指标和导出跟踪的客户端。遥测包中的文件logging.go定义了InitLogging()函数如下：

```go
package telemetry
 
import (
    "io"
    "github.com/rs/zerolog"
)

func InitLogging(
    w io.Writer, version string, logLevel int,
) zerolog.Logger {

    rLogger := zerolog.New(w)
    versionedL := rLogger.With().Str("version", version)
    timestampedL := versionedL.Timestamp().Logger()
    levelledL := timestampedL.Level(zerolog.Level(logLevel))

    return levelledL
}
```

我们从 github.com/rs/zerolog 包中调用 zerolog.New() 函数来创建一个根记录器，一个 zerolog.Logger 对象，输出 writer 设置为 w。然后我们通过调用 With() 方法向根记录器添加日志上下文。这通过使用 Str() 方法添加上下文来从根记录器 rLogger 创建子记录器，该方法将键、版本添加到所有日志消息，其值设置为 version 中的指定字符串。这将导致应用程序的所有日志都包含该字段中的应用程序版本。

接下来，我们添加另一个日志上下文以向日志添加时间戳并创建另一个子记录器，timestampedL。最后，我们通过调用 Level() 方法添加分级日志的逻辑并返回创建的 zerolog.Logger 对象。配置的记录器只会在其级别等于或大于 logLevel 中的值指示的配置级别时记录消息。 logLevel 的值必须是介于 -1 和 5（包括两者）之间的整数。将级别设置为 -1 将记录所有消息，将级别设置为 5 将仅记录紧急消息。

遥测包中的文件metrics.go定义了InitMetrics()函数如下：

```go
package telemetry
import (
    "fmt"

    "github.com/DataDog/datadog-go/statsd"
)

type MetricReporter struct {
    statsd *statsd.Client
}

func InitMetrics(statsdAddr string) (MetricReporter, error) {
    var m MetricReporter
    var err error
    m.statsd, err = statsd.New(statsdAddr)
    if err != nil {
        return m, err
    }
    return m, nil
}
```

我们选择 github.com/DataDog/datadog-go/statsd 包，因为它维护得很好。我们通过调用 statsd.New() 函数来创建客户端，传递 statsd 服务器的地址，并将创建的客户端分配给 MetricReporter 对象的 statsd 字段。再次值得注意的是，一旦 OpenTelemetry 的指标支持可供使用，我们将不会在我们的应用程序中使用特定于供应商的库。

我们定义了一个类型，DurationMetric，来封装一个命令执行持续时间的单一度量：

```go
type DurationMetric struct {
    Cmd            string
    DurationMs float64
    Success        bool
}
```

然后我们定义了一个方法 ReportDuration()，它将推送一个包含命令运行时间的直方图指标。持续时间以秒为单位。将向指标添加两个标签，这将允许对指标进行分组和聚合。我们将已执行的命令（如 Cmd 中指定的那样）添加为一个标签，以及该命令是否成功执行（如 Success 字段所指定的那样）作为第二个标签。该方法定义如下：

```go
func (m MetricReporter) ReportDuration(metric DurationMetric) {
    metricName := "cmd.duration"
    m.statsd.Histogram(
        metricName,
        metric.DurationMs,
        []string{
            fmt.Sprintf("cmd:%s", metric.Cmd),
            fmt.Sprintf("success:%v", metric.Success),
        },
        1, //sample rate (0-none, 1 - all)
    )
}
```

可以定义类似的方法来推送其他度量类型。

InitTracing() 方法在遥测包内的 trace.go 文件中定义。在其中，我们在遥测包中设置了应用程序范围的跟踪配置，如下所示：

```go
package telemetry
 
import (
    "context"

    "go.opentelemetry.io/otel"
    "go.opentelemetry.io/otel/trace"
    "go.opentelemetry.io/otel/exporters/jaeger"

    // TODO import other otel packages
)


type TraceReporter struct {
    Client trace.Tracer
    Ctx    context.Context
}

func InitTracing(
    jaegerAddr string,
) (TraceReporter, *sdktrace.TracerProvider, error) {

    // 1. Setup trace exporter

    // 2. Setup span processor

    // 3. Setup tracer provider

    // 4. Create the propagator

    // 5. Create and configure Tracer

    // 6. Return a value of type TraceReporter
 
}
```

有六个关键步骤，如下所述。我们需要一个可以将跟踪导出到的目的地，以便我们可以查看和查询它们。我们将使用 Jaeger (www.jaegertracing.io)，一个开源分布式跟踪系统，因此我们使用由 go.opentelemetry.io/otel/exporters/jaeger 包实现的 Jaeger 导出器。

以下代码片段创建了跟踪导出器：

```go
traceExporter, err := jaeger.New(
    jaeger.WithCollectorEndpoint(
        jaeger.WithEndpoint(jaegerAddr),
    ),
)
```

请注意，我们可以改为使用由 go.opentelemetry.io/otel/exporters/otlp 包实现的 OpenTelemetry 收集器导出器，并使我们的应用程序与我们正在使用的分布式跟踪系统完全中立。然而，为简单起见，我直接使用了 Jaeger 导出器。

接下来，我们设置 span 处理器来监督处理应用程序发出的 span 数据，并使用我们之前创建的跟踪导出器对其进行配置：

```go
bsp := sdktrace.NewSimpleSpanProcessor(traceExporter)
```

对于生产应用，建议配置不同的跨度处理器，创建的批量跨度处理器如下：

```go
bsp := sdktrace.NewBatchSpanProcessor(traceExporter)
```

下一步是使用上述跨度处理器创建跟踪器提供程序：

```go
tp := sdktrace.NewTracerProvider(
    sdktrace.WithSpanProcessor(bsp),
    sdktrace.WithResource(
        resource.NewWithAttributes(
            semconv.SchemaURL,
            semconv.ServiceNameKey.String(
                "PkgServer-Cli",
            ),
        ),
    ),
)
```

我们创建了使用两个参数调用 sdktrace.NewTraceProvider() 的跟踪器提供程序。第一个是我们在上一步中创建的跨度处理器。第二个参数标识创建由 OpenTelemetry 文档“资源语义约定”描述的跟踪的应用程序。在这里，我们将产生这些跟踪的服务名称设置为 PkgServer-Cli。一旦我们像之前一样创建了跟踪器提供程序，我们就可以使用以下代码将其配置为当前应用程序的全局跟踪程序提供程序：

```go
otel.SetTracerProvider(tp)
```


接下来，我们为跟踪设置全局传播器，这是当前应用程序的跟踪标识符将如何传输到其他服务的方式：

```go
propagator := propagation.NewCompositeTextMapPropagator(
    propagation.Baggage{},
    propagation.TraceContext{},
)
otel.SetTextMapPropagator(propagator)
The final steps are carried out by the code snippet below:
v1, err := baggage.NewMember("version", "version")
bag, err := baggage.New(v1)
tr.Client = otel.Tracer("")
ctx := context.Background()
tr.Ctx = baggage.ContextWithBaggage(ctx, bag)
return tr, tp, nil
```

当通过 HTTP 与包服务器通信时，我们将使用 go.opentelemetry.io/contrib/instrumentation/net/http/otelhttp 包提供的特殊配置的 HTTP 客户端：

```go
// pkgregister/pkgregister.go
func RegisterPackage(
    ctx context.Context, cliConfig *config.PkgCliConfig, 
    url string, data PkgData,
) (*PkgRegisterResult, error) {
    // Other code deleted

    r, err := http.NewRequestWithContext(
        ctx, http.MethodPost, url+"/api/packages", 
        reader,
    )
    if err != nil {
        return nil, err
    }
    r.Header.Set("Content-Type", contentType)
    authToken := os.Getenv("X_AUTH_TOKEN")
    if len(authToken) != 0 {
        r.Header.Set("X-Auth-Token", authToken)
    }

    client := http.Client{
        Transport: otelhttp.NewTransport(http.DefaultTransport),
    }
    resp, err := client.Do(r)

    // process response
}
```

当我们使用检测的 HTTP 客户端时，在使用此客户端执行的 HTTP 请求响应事务期间会自动发出跨度。

在应用程序的 main() 函数中，遥测配置的初始化如下所示：

```go
func main() {
    var tp *sdktrace.TracerProvider

    cliConfig.Logger = telemetry.InitLogging(
        os.Stdout, version, c.logLevel,
    )
    cliConfig.Metrics, err = telemetry.InitMetrics(c.statsdAddr)
    if err != nil {
        cliConfig.Logger.Fatal().Str("error", err.Error()).Msg(
            "Error initializing metrics system",
        )
    }

    cliConfig.Tracer, tp, err = telemetry.InitTracing(
        c.jaegerAddr+"/api/traces", version,
    )
    if err != nil {
        cliConfig.Logger.Fatal().Str("error", err.Error()).Msg(
            "Error initializing tracing system",
        )
    }
    defer func() {
        tp.ForceFlush(context.Background())
        tp.Shutdown(context.Background())
    }()

    err = handleSubCommand(cliConfig, os.Stdout, subCmdArgs)
    if err != nil {
        cliConfig.Logger.Fatal().Str("error", err.Error()).Msg(
            "Error executing sub-command",
        )
    }
}
```

你可以看到我们如何使用初始化的记录器来记录结构化消息：

```go
cliConfig.Logger.Fatal().Str("error", err.Error()).Msg(
    "Error initializing metrics system",
)
```

日志消息将显示如下：

```go
{"level":"fatal","version":"0.1","error":"lookup 127.0.0.: no such host","time":"2021-09-11T12:22:23+10:00","message":
"Error initializing metrics system"}
```

在 cmd 包的 HandleRegister() 函数中，你将找到分级日志记录的示例：

```go
cliConfig.Logger.Info().Msg("Uploading package…")
cliConfig.Logger.Debug().Str("package_name", c.name).
        Str("package_version", c.version).
        Str("server_url", c.serverUrl)
```

为了报告命令运行的持续时间，我们实现了以下模式：

```go
c.Logger = c.Logger.With().Str("command", "register").Logger()
tStart := time.Now()
defer func() {
    duration := time.Since(startTime).Seconds() 
    c.Metrics.ReportDuration(
        telemetry.DurationMetric{
            Cmd:        "pkgcli.register",
            DurationMs: duration,
            Success:    err == nil,
        },
    )
}()
err = cmd.HandleRegister(&c, w, args[1:])
```

在我们调用特定的子命令处理函数之前，我们更新记录器以创建一个新的上下文，这样所有的日志都会有一个额外的字段，命令，设置为注册，它将识别与注册子相关联的日志消息命令。

我们还启动了一个计时器，然后在一个延迟函数中，我们使用 ReportDuration() 方法报告命令运行的持续时间。

在cmd包中定义的HandleRegister()函数中，我们在向包服务器发出HTTP请求之前创建了一个span，如下所示：

```go
ctx, span := cliConfig.Tracer.Client.Start(
    cliConfig.Tracer.Ctx,
    "pkgquery.register",
)
defer span.End()
```

接下来，让我们看看包服务器的检测版本。

### HTTP 应用程序
对于 HTTP 服务器应用程序，我们将配置日志记录，初始化网络客户端以在服务器启动期间导出指标和跟踪。然后，我们将这些初始化配置提供给应用程序的其余部分，以便在执行任何命令期间记录任何消息、发布指标或导出跟踪。

修改后的包服务器的代码在 appendix-a/http-app 目录中。它建立在我们在第 11 章中实现的包服务器版本之上。

我们在 config 包中定义了一个 struct AppConfig 来封装应用的配置，包括遥测配置：

```go
type AppConfig struct {
    PackageBucket *blob.Bucket
    Db            *sql.DB
    UsersSvc      users.UsersClient

    // telemetry
    Logger   zerolog.Logger
    Metrics  telemetry.MetricReporter
    Trace    trace.Tracer
    TraceCtx context.Context
    Span     trace.Span
    SpanCtx  context.Context
}
```

遥测包定义了在文件 logging.go、metrics.go 和 trace.go 中初始化所有遥测配置的方法。 InitLogging() 和 InitMetrics() 方法将与我们之前为命令行应用程序定义的相同。

中间件包包含用于发送遥测数据的中间件的定义。日志中间件定义如下：

```go
func LoggingMiddleware(
    c *config.AppConfig, h http.Handler,
) http.Handler {
    return http.HandlerFunc(func(
        w http.ResponseWriter, r *http.Request,
    ) {
        c.Logger.Printf("Got request - headers:%#v\n", r.Header)
        startTime := time.Now()
        h.ServeHTTP(w, r)
        c.Logger.Info().Str(
            "protocol",
            r.Proto,
        ).Str(
            "path",
            r.URL.Path,
        ).Str(
            "method",
            r.Method,
        ).Float64(
            "duration",
            time.Since(startTime).Seconds(),
        ).Send()
    })
}
```

你可以在前面的代码片段中看到结构化日志记录的示例。我们通过逐步添加我们想要添加到日志的键值字段并调用 Send() 方法来构建日志行（在内部，一个 zerolog.Event 值），这会导致日志被发出。上面定义的 LoggingMiddleware() 函数发出的请求日志行将显示如下：

```json
{"level":"info","version":"0.1","protocol":"HTTP/1.1",
"path":"/api/packages","method":"POST","duration":0.038707083,
"time":"2021-09-12T08:39:05+10:00"}
```

我们定义了另一个中间件来推送请求处理延迟：

```go
func MetricMiddleware(c *config.AppConfig, h http.Handler) http.Handler {
    return http.HandlerFunc(func(
        w http.ResponseWriter, r *http.Request,
    ) {
        startTime := time.Now()
        h.ServeHTTP(w, r)
        duration := time.Since(startTime).Seconds() </line><line xml:id="bapp01-line-0268"><![CDATA[                c.Metrics.ReportLatency(
            telemetry.DurationMetric{
                DurationMs: duration,
                Path:       r.URL.Path,
                Method:     r.Method,
            },
        )

                                                                                                    })
}
```

InitTracing() 方法看起来略有不同：

```go
func InitTracing(jaegerAddr string) error {
    traceExporter, err := jaeger.New(
        jaeger.WithCollectorEndpoint(
            jaeger.WithEndpoint(jaegerAddr + "/api/traces"),
        ),
    )
    if err != nil {
        return err
    }
    bsp := sdktrace.NewSimpleSpanProcessor(traceExporter)

    tp := sdktrace.NewTracerProvider(
        sdktrace.WithSpanProcessor(bsp),
        sdktrace.WithResource(
            resource.NewWithAttributes(
                semconv.SchemaURL,
                semconv.ServiceNameKey.String(
                    "PkgServer",
                ),
            ),
        ),
    )
    otel.SetTracerProvider(tp)
    propagator := propagation.NewCompositeTextMapPropagator(
        propagation.Baggage{},
        propagation.TraceContext{},
    )
    otel.SetTextMapPropagator(propagator)
    return nil
}
```

我们配置全局跟踪提供程序，但我们不创建跟踪。与在执行命令后退出的命令行应用程序不同，服务器将在其生命周期内为多个请求提供服务。因此，我们使用专用中间件为每个请求创建跟踪。

TracingMiddleware() 为每一个新的请求创建一个跟踪，它在中间件包中定义如下：

```go
func TracingMiddleware(
    c *config.AppConfig, h http.Handler,
) http.Handler {
    return http.HandlerFunc(func(
        w http.ResponseWriter, r *http.Request,
    ) {
        c.Trace = otel.Tracer("")
        tc := propagation.TraceContext{}
        incomingCtx := tc.Extract(
            r.Context(),
            propagation.HeaderCarrier(r.Header),
        )
        c.TraceCtx = incomingCtx

        ctx, span := c.Trace.Start(c.TraceCtx, r.URL.Path)
        c.Span = span
        c.SpanCtx = ctx
        defer c.Span.End()

        h.ServeHTTP(w, r)
    })
}
```

我们从请求中提取传入的上下文，然后使用此上下文启动一个新的跨度，并将跨度名称设置为正在处理的当前请求路径。我们在处理请求之后和从这个中间件返回之前结束跨度。值得注意的是，即使 go.opentelemetry.io/contrib/instrumentation/net/http/otelhttp 包定义了一个用于跟踪 HTTP 服务器的中间件，我们在这里定义了我们自己的中间件，因为这将允许我们创建我们的跨度自己的。例如，在存储包中的UpdateDb()函数中，我们会在启动数据库事务之前创建一个span，并在事务提交或回滚后结束它：

```go
func UpdateDb(
    ctx context.Context,
    config *config.AppConfig,
    row types.PkgRow,
) error {
    conn, err := config.Db.Conn(ctx)
    if err != nil {
        return err
    }
    defer func() {
        err = conn.Close()
        if err != nil {
            config.Logger.Debug().Msg(err.Error())
        }
    }()

    _, spanTx := config.Trace.Start(
        config.SpanCtx, "sql:transaction",
    )
    defer spanTx.End()

    tx, err := conn.BeginTx(ctx, nil)
    if err != nil {
        return err
    }
    // Rest of the code
}
```

将来，数据库/sql 包的自动检测版本可能会可用，然后我们将不必为数据库操作编写自己的手动跟踪代码。

在创建 gRPC 客户端以与用户服务进行通信时，我们会利用自动检测的库。在 server.go 中，你会发现以下函数：

```go
func setupGrpcConn(addr string) (*grpc.ClientConn, error) {
    return grpc.DialContext(
        context.Background(),
        addr,
        grpc.WithInsecure(),
        grpc.WithUnaryInterceptor(
            otelgrpc.UnaryClientInterceptor(),
        ),
        grpc.WithStreamInterceptor(
            otelgrpc.StreamClientInterceptor(),
        ),
    )
}
```

go.opentelemetry.io/contrib/instrumentation/google.golang.org/grpc/otelgrpc 包定义了客户端和服务器端拦截器，以将你的 gRPC 应用程序与 OpenTelemetry 集成。

接下来，让我们看看 gRPC 服务器的检测版本。

### gRPC 应用程序
与 HTTP 服务器应用程序一样，我们将配置日志记录，初始化网络客户端以在服务器启动期间导出指标和跟踪。检测的 gRPC 服务器的代码位于 appendix-a/grpc-server 目录中。它是我们在第 8 章“使用 gRPC 构建 RPC 应用程序”中创建的用户服务的一个版本，它定义了一个单一的 RPC 方法 GetUser()。该服务的实现位于主包中的文件 usersServiceHandler.go 中。你还可以通过向 userService 结构体添加附加字段来查看跨服务处理程序共享数据的示例，这是 Users 服务的实现：

```go
type userService struct {
    users.UnimplementedUsersServer
    config config.AppConfig
}

func (s *userService) GetUser(
    ctx context.Context,
    in *users.UserGetRequest,
) (*users.UserGetReply, error) {
    s.config.Logger.Printf(
        "Received request for user verification: %s\n",
        in.Auth,
    )
    u := users.User{
        Id: rand.Int31n(4) + 1,
    }
    return &users.UserGetReply{User: &u}, nil
}
```

grpc.Server 对象在 main() 函数中创建如下：

```go
s := grpc.NewServer(
    grpc.ChainUnaryInterceptor(
        interceptors.MetricUnaryInterceptor(&config),
        interceptors.LoggingUnaryInterceptor(&config),
        otelgrpc.UnaryServerInterceptor(),
    ),
    grpc.ChainStreamInterceptor(
        interceptors.MetricStreamInterceptor(&config),
        interceptors.LoggingStreamInterceptor(&config),
        otelgrpc.StreamServerInterceptor(),
    ),
)
```

我们已经在拦截器包中定义了度量和日志拦截器。我们还注册了 go.opentelemetry.io/contrib/instrumentation/google.golang.org/grpc/otelgrpc 包定义的拦截器，它为我们提供了 gRPC 服务处理程序调用的自动检测。遥测包初始化日志记录、指标和跟踪配置。你会发现初始化代码与上一节中用于 HTTP 服务器的代码相同。

使用命令行应用程序、HTTP 服务器和 gRPC 服务器的上述检测后，当你发出上传包的请求时，你将能够查看从每个应用程序发布的日志、指标和跟踪。当然，跟踪还有一个额外的优势，即在整个交易通过三个系统传播时以视觉方式显示整个交易。

## 概括

在本附录中，我们首先快速概述了日志、指标和跟踪。然后，你了解了用于检测命令行应用程序、HTTP 客户端和服务器以及 gRPC 应用程序的模式。我们使用 github.com/rs/zerolog 实现了结构化和分级的日志记录。然后，你学会了使用 github.com/DataDog/datadog-go 以 statsd 格式从应用程序导出测量值。最后，你学会了使用 github.com/open-telemetry/opentelemetry-go 导出跟踪以关联跨系统边界的事务。