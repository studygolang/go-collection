## [⚡ZAP](https://github.com/uber-go/zap)



### 一句话描述

提供快速，结构化，高性能的日志记录。

### 简介

#### zap 是什么?

uber 开源的日志记录工具。

#### 为什么选择 zap?

因为出色的性能。

大多数日志库提供的方式是基于反射的序列化和字符串格式化，这种方式代价高昂，Zap 采取不同的方法，包含一个无反射的零分配 JSON 编码器，尽可能避免序列化开销，它比其他结构化日志记录包快 4~10 倍。

### Example

安装

```bash
go get github.com/uber-go/zap
```

Zap 提供了两种类型的 logger

+ SugaredLogger
+ Logger

在**性能良好但不是关键**的上下文中，使用 **SugaredLogger**，它比其他结构化日志记录包快 4-10 倍，并且支持结构化和 `printf` 风格的日志记录。

> 例一

```go
func TestSugar(t *testing.T) {
	logger, _ := zap.NewProduction()
	// 默认 logger 不缓冲。
	// 但由于底层 api 允许缓冲，所以在进程退出之前调用 Sync 是一个好习惯。
	defer logger.Sync()
	sugar := logger.Sugar()
	sugar.Infof("Failed to fetch URL: %s", "https://baidu.com")
}
```

对**每一次内存分配都很重要**的上下文中，使用 **Logger** ，它甚至比前者更快，内存分配次数也更少，但它只支持强类型的结构化日志记录。

> 例二

```go
func TestLogger(t *testing.T) {
	logger, _ := zap.NewDevelopment()
	defer logger.Sync()
	logger.Info("failed to fetch URL",
		// 强类型字段
		zap.String("url", "https://baidu.com"),
		zap.Int("attempt", 3),
		zap.Duration("backoff", time.Second),
	)
}
```

两者之间可以轻松转换

> 例三

```go
// 创建 logger
logger := zap.NewExample()
defer logger.Sync()

// 转换 SugaredLogger
sugar := logger.Sugar()
// 转换 logger
plain := sugar.Desugar()
```

### 源码分析

![zap概览](http://img.golang.space/shot-1620404974607.png)

#### Logger

logger 提供快速，分级，结构化的记录。所有的方法都是安全的，内存分配很重要，因此它的 API 有意偏向于性能和类型。下面为 Logger 结构体，稍后再细看注释及该结构体的方法。

```go
type Logger struct {
  // 实现编码和输出的接口
	core zapcore.Core  
  // 记录器开发模式，DPanic 等级将记录 panic
	development bool
  // 开启记录调用者的行号和函数名
	addCaller   bool		
  // 致命日志采取的操作，默认写入日志后 os.Exit()
  onFatal     zapcore.CheckWriteAction 
	name        string	
  // 设置记录器生成的错误目的地
	errorOutput zapcore.WriteSyncer  
  // 记录 >= 该日志等级的堆栈追踪
	addStack zapcore.LevelEnabler 
  // 避免记录器认为封装函数为调用方
	callerSkip int	
  // 默认为系统时间 
	clock Clock		
}
```

在 **Example** 中分别使用了 `NewProduction` 和 `NewExample` ，接下来以这两个函数开始分析。下图表示 A 函数调用了 B 函数，其中箭头表示调用关系。图中函数都会分析到。

![](http://img.golang.space/shot-1620608635804.png)

**NewProduction**

从下面代码中可以看出，此函数是对 `NewProductionConfig().Build(...)` 封装的快捷方式

```go
func NewProduction(options ...Option) (*Logger, error) {
	return NewProductionConfig().Build(options...)
}
```

**NewProductionConfig**

在 InfoLevel 及更高版本上启用了日志记录。它使用 JSON 编码器，写入 stderr，启用采样。

```go
func NewProductionConfig() Config {
	return Config{
    // info 日志级别
		Level:       NewAtomicLevelAt(InfoLevel),
    // 非开发模式
		Development: false,
    // 采样设置
		Sampling: &SamplingConfig{
			Initial:    100, // 相同日志级别下相同内容每秒日志输出数量
			Thereafter: 100, // 超过该数量，才会再次输出
		},
    // JSON 编码器
		Encoding:         "json",
    // 稍后介绍
		EncoderConfig:    NewProductionEncoderConfig(),
    // 输出到 stderr
		OutputPaths:      []string{"stderr"},
		ErrorOutputPaths: []string{"stderr"},
	}
}
```

**Config** 结构体

```go
type Config struct {
	// 日志级别
	Level AtomicLevel `json:"level" yaml:"level"`
	// 开发模式
	Development bool `json:"development" yaml:"development"`
	// 停止使用调用方的函数和行号
	DisableCaller bool `json:"disableCaller" yaml:"disableCaller"`
	// 完全停止使用堆栈跟踪，默认为  >=WarnLevel 使用堆栈跟踪
	DisableStacktrace bool `json:"disableStacktrace" yaml:"disableStacktrace"`
	// 采样设置策略
	Sampling *SamplingConfig `json:"sampling" yaml:"sampling"`
	// 记录器的编码，有效值为 'json' 和 'console' 以及通过 `RegisterEncoder` 注册的有效编码
	Encoding string `json:"encoding" yaml:"encoding"`
	// 编码器选项
	EncoderConfig zapcore.EncoderConfig `json:"encoderConfig" yaml:"encoderConfig"`
	// 日志的输出路径
	OutputPaths []string `json:"outputPaths" yaml:"outputPaths"`
	// zap 内部错误的输出路径
	ErrorOutputPaths []string `json:"errorOutputPaths" yaml:"errorOutputPaths"`
	// 添加到根记录器的字段的集合
	InitialFields map[string]interface{} `json:"initialFields" yaml:"initialFields"`
}
```

**NewDevelopment**

从下面代码中可以看出，此函数是对 `NewDevelopmentConfig().Build(...)` 封装的快捷方式

```go
func NewDevelopment(options ...Option) (*Logger, error) {
	return NewDevelopmentConfig().Build(options...)
}
```

**NewDevelopmentConfig**

此函数在DebugLevel及更高版本上启用日志记录，它使用 console 编码器，写入 stderr，禁用采样。

```go
func NewDevelopmentConfig() Config {
	return Config{
    // debug 等级
		Level:            NewAtomicLevelAt(DebugLevel),
    // 开发模式
		Development:      true,
    // console 编码器
		Encoding:         "console",
		EncoderConfig:    NewDevelopmentEncoderConfig(),
    // 输出到 stderr
		OutputPaths:      []string{"stderr"},
		ErrorOutputPaths: []string{"stderr"},
	}
}
```

`NewProductionEncoderConfig` 和 `NewDevelopmentEncoderConfig` 是返回针对性的编码器配置。

```go
type EncoderConfig struct {
	// 设置 编码为 JSON 时的 KEY
	MessageKey    string `json:"messageKey" yaml:"messageKey"`
	LevelKey      string `json:"levelKey" yaml:"levelKey"`
	TimeKey       string `json:"timeKey" yaml:"timeKey"`
	NameKey       string `json:"nameKey" yaml:"nameKey"`
	CallerKey     string `json:"callerKey" yaml:"callerKey"`
	FunctionKey   string `json:"functionKey" yaml:"functionKey"`
	StacktraceKey string `json:"stacktraceKey" yaml:"stacktraceKey"`
  // 配置行分隔符
	LineEnding    string `json:"lineEnding" yaml:"lineEnding"`
	// 配置常见复杂类型的基本表示形式。
	EncodeLevel    LevelEncoder    `json:"levelEncoder" yaml:"levelEncoder"`
	EncodeTime     TimeEncoder     `json:"timeEncoder" yaml:"timeEncoder"`
	EncodeDuration DurationEncoder `json:"durationEncoder" yaml:"durationEncoder"`
	EncodeCaller   CallerEncoder   `json:"callerEncoder" yaml:"callerEncoder"`
	// 日志名称，此参数可选
	EncodeName NameEncoder `json:"nameEncoder" yaml:"nameEncoder"`
	// 配置 console 编码器使用的字段分隔符，默认 tab
	ConsoleSeparator string `json:"consoleSeparator" yaml:"consoleSeparator"`
}
```

**NewProductionEncoderConfig**

```go
func NewProductionEncoderConfig() zapcore.EncoderConfig {
	return zapcore.EncoderConfig{
		TimeKey:        "ts",
		LevelKey:       "level",
		NameKey:        "logger",
		CallerKey:      "caller",
		FunctionKey:    zapcore.OmitKey,
		MessageKey:     "msg",
		StacktraceKey:  "stacktrace",
    // 默认换行符 \n
		LineEnding:     zapcore.DefaultLineEnding,
    // 日志等级序列为小写字符串，如:InfoLevel被序列化为 "info"
		EncodeLevel:    zapcore.LowercaseLevelEncoder,
    // 时间序列化成浮点秒数
		EncodeTime:     zapcore.EpochTimeEncoder,
    // 时间序列化，Duration为经过的浮点秒数
		EncodeDuration: zapcore.SecondsDurationEncoder,
    // 以 包名/文件名:行数 格式序列化
		EncodeCaller:   zapcore.ShortCallerEncoder,
	}
}
```

该配置会输出如下结果，此结果出处参见 Example 中的例一

```bash
{"level":"info","ts":1620367988.461055,"caller":"test/use_test.go:24","msg":"Failed to fetch URL: https://baidu.com"}
```

**NewDevelopmentEncoderConfig**

```go
func NewDevelopmentEncoderConfig() zapcore.EncoderConfig {
	return zapcore.EncoderConfig{
		// keys 值可以是任意非空的值
		TimeKey:        "T",
		LevelKey:       "L",
		NameKey:        "N",
		CallerKey:      "C",
		FunctionKey:    zapcore.OmitKey,
		MessageKey:     "M",
		StacktraceKey:  "S",
     // 默认换行符 \n
		LineEnding:     zapcore.DefaultLineEnding,
    // 日志等级序列为大写字符串，如:InfoLevel被序列化为 "INFO"
		EncodeLevel:    zapcore.CapitalLevelEncoder,
    // 时间格式化为  ISO8601 格式
		EncodeTime:     zapcore.ISO8601TimeEncoder,
		EncodeDuration: zapcore.StringDurationEncoder,
    // // 以 包名/文件名:行数 格式序列化
		EncodeCaller:   zapcore.ShortCallerEncoder,
	}
}
```

该配置会输出如下结果，此结果出处参见 Example 中的例二

```bash
2021-05-07T14:14:12.434+0800	INFO	test/use_test.go:31	failed to fetch URL	{"url": "https://baidu.com", "attempt": 3, "backoff": "1s"}
```

`NewProductionConfig` 和 `NewDevelopmentConfig` 返回 config 调用 `Build` 函数返回 logger，接下来我们看看这个函数。

```go
func (cfg Config) Build(opts ...Option) (*Logger, error) {
  enc, err := cfg.buildEncoder()
	if err != nil {
		return nil, err
	}

	sink, errSink, err := cfg.openSinks()
	if err != nil {
		return nil, err
	}
	
	if cfg.Level == (AtomicLevel{}) {
		return nil, fmt.Errorf("missing Level")
	}
	
	log := New(
		zapcore.NewCore(enc, sink, cfg.Level),
		cfg.buildOptions(errSink)...,
	)
	if len(opts) > 0 {
		log = log.WithOptions(opts...)
	}
	return log, nil
}
```

Build 通过对 config 内的参数解析、封装，最后调用 New 方法来创建 Logger，你也可以调用 `New` 方法来自定义创建 logger，其中 core 用于输出日志，非常重要，后面有空着重分析。

#### SugaredLogger

加了语法糖的 logger。

```go
type SugaredLogger struct {
	base *Logger
}
```

对于每个日志级别，提供了三种方法

```go
// Info 使用 fmt.Sprint 构造和记录消息。 
func (s *SugaredLogger) Info(args ...interface{}) {
	s.log(InfoLevel, "", args, nil)
}

// Infof 使用 fmt.Sprintf 记录模板消息。
func (s *SugaredLogger) Infof(template string, args ...interface{}) {
	s.log(InfoLevel, template, args, nil)
}

// Infow 记录带有其他上下文的消息
func (s *SugaredLogger) Infow(msg string, keysAndValues ...interface{}) {
	s.log(InfoLevel, msg, nil, keysAndValues)
}
```

首先在 `sugar.Infof("...")` 打上断点，从这开始追踪源码。  (此断点位置参见 Example 中的例一)

三个函数都调用了 `log` 函数，函数如下

```go
func (s *SugaredLogger) log(lvl zapcore.Level, template string, fmtArgs []interface{}, context []interface{}) {
	// 判断是否启用的日志级别
	if lvl < DPanicLevel && !s.base.Core().Enabled(lvl) {
		return
	}
	// 将参数合并到语句中
	msg := getMessage(template, fmtArgs)
  // Check 可以帮助避免分配一个分片来保存字段。
	if ce := s.base.Check(lvl, msg); ce != nil {
		ce.Write(s.sweetenFields(context)...)
	}
}
```

函数的第一个参数 `InfoLevel` 是日志级别，其源码如下

```go
const (
	// Debug 应是大量的，且通常在生产状态禁用
	DebugLevel = zapcore.DebugLevel
	// Info 是默认的记录优先级
	InfoLevel = zapcore.InfoLevel
	// Warn 比 info 更重要
	WarnLevel = zapcore.WarnLevel
	// ErEor 是高优先级的,如果程序顺利不应该产生任何 err 级别日志
	ErrorLevel = zapcore.ErrorLevel
	// DPanic 特别重大的错误 
	DPanicLevel = zapcore.DPanicLevel
	// Panic 记录消息后调用 panic
	PanicLevel = zapcore.PanicLevel
	// Fatal 记录消息后调用 os.Exit(1).
	FatalLevel = zapcore.FatalLevel
)
```

`getMessage` 函数处理 `template` 和 `fmtArgs` 参数，主要为不同的参数选择最合适的方式合并消息

```go
func getMessage(template string, fmtArgs []interface{}) string {
  // 没有参数直接返回 template
	if len(fmtArgs) == 0 {
		return template
	}
	
  // 此处调用 Sprintf 会使用反射
	if template != "" {
		return fmt.Sprintf(template, fmtArgs...)
	}
	
  // 消息为空并且有一个参数，返回该参数
	if len(fmtArgs) == 1 {
		if str, ok := fmtArgs[0].(string); ok {
			return str
		}
	}
  // 返回所有 fmtArgs
	return fmt.Sprint(fmtArgs...)
}
```

### Doc

https://pkg.go.dev/go.uber.org/zap

### QA

### 比较

> 说明 : 以下资料来源于 zap 官方

Log a message and 10 fields:

| Package         | Time        | Time % to zap | Objects Allocated |
| --------------- | ----------- | ------------- | ----------------- |
| ⚡ zap           | 862 ns/op   | +0%           | 5 allocs/op       |
| ⚡ zap (sugared) | 1250 ns/op  | +45%          | 11 allocs/op      |
| zerolog         | 4021 ns/op  | +366%         | 76 allocs/op      |
| go-kit          | 4542 ns/op  | +427%         | 105 allocs/op     |
| apex/log        | 26785 ns/op | +3007%        | 115 allocs/op     |
| logrus          | 29501 ns/op | +3322%        | 125 allocs/op     |
| log15           | 29906 ns/op | +3369%        | 122 allocs/op     |

Log a message with a logger that already has 10 fields of context:

| Package         | Time        | Time % to zap | Objects Allocated |
| --------------- | ----------- | ------------- | ----------------- |
| ⚡ zap           | 126 ns/op   | +0%           | 0 allocs/op       |
| ⚡ zap (sugared) | 187 ns/op   | +48%          | 2 allocs/op       |
| zerolog         | 88 ns/op    | -30%          | 0 allocs/op       |
| go-kit          | 5087 ns/op  | +3937%        | 103 allocs/op     |
| log15           | 18548 ns/op | +14621%       | 73 allocs/op      |
| apex/log        | 26012 ns/op | +20544%       | 104 allocs/op     |
| logrus          | 27236 ns/op | +21516%       | 113 allocs/op     |

Log a static string, without any context or `printf`-style templating:

| Package          | Time       | Time % to zap | Objects Allocated |
| ---------------- | ---------- | ------------- | ----------------- |
| ⚡ zap            | 118 ns/op  | +0%           | 0 allocs/op       |
| ⚡ zap (sugared)  | 191 ns/op  | +62%          | 2 allocs/op       |
| zerolog          | 93 ns/op   | -21%          | 0 allocs/op       |
| go-kit           | 280 ns/op  | +137%         | 11 allocs/op      |
| standard library | 499 ns/op  | +323%         | 2 allocs/op       |
| apex/log         | 1990 ns/op | +1586%        | 10 allocs/op      |
| logrus           | 3129 ns/op | +2552%        | 24 allocs/op      |
| log15            | 3887 ns/op | +3194%        | 23 allocs/op      |

### 相似的库

[logrus](https://github.com/sirupsen/logrus)  功能强大

[zerolog](https://github.com/rs/zerolog) 性能相当好的日志库



