## [Viper](https://github.com/spf13/viper)

![Viper](https://github.com/spf13/viper/raw/master/.github/logo.png?raw=true)

### 一句话描述

> Go configuration with fangs

### 简介

#### viper是什么？

> Viper is a complete configuration solution for Go applications including 12-Factor apps. It is designed to work within an application, and can handle all types of configuration needs and formats. It supports:

Viper 是一个完整的Go应用程序配置解决方案的库。它被设计成应用在应用程序中，并且可以处理所有类型的配置需求和格式。它支持:

- 设置默认值
- 支持读取 JSON、 TOML、 YAML、 HCL、 envfile 和 Java properties 属性配置文件
- 实时监视和重读配置文件(可选)
- 读取环境变量
- 读取远程配置系统(etcd 或 consul) ，并观察更改
- 读取命令行flags
- 读取缓冲区
- setting explicit values 设置显式值

可以将 Viper 作为所有应用程序配置需求的注册中心。

#### 为什么选择viper？

> When building a modern application, you don’t want to worry about configuration file formats; you want to focus on building awesome software. Viper is here to help with that.

在构建现代应用程序时，您不需要担心配置文件格式; 您需要专注于构建出色的软件。”毒蛇”是来帮忙的。

Viper 为你做了以下事情:

1. 查找、加载和解析 JSON、 TOML、 YAML、 HCL、 INI、 envfile 或 Java 属性格式的配置文件
2. 提供一种机制来设置不同配置选项的默认值提供一种机制，用于为通过命令行flags指定的选项设置重写值
3. 提供一个别名系统，以便在不破坏现有代码的情况下轻松重命名参数
4. 使用户提供与默认值相同的命令行或配置文件时，可以很容易地区分两者之间的不同

Viper uses the following precedence order. Each item takes precedence over the item below it:

Viper 使用以下优先顺序。每个项优先级高于下面的项:

- 显式地调用Set
- flag
- env
- config 
- key/value store
- default 

**重要提示:** Viper 配置keys不区分大小写。目前正在讨论将其设置为可选的。

### Example 

> 备注：可以对例子进行扩展；

[往viper中赋值](https://github.com/spf13/viper#putting-values-into-viper)

[从viper中获取值](https://github.com/spf13/viper#getting-values-from-viper)

### 源码分析

> 备注：可以对部分源码进行分析

### Doc

https://github.com/spf13/viper/blob/master/README.md

### QA

**为什么叫viper?**

Viper是被设计成 [Cobra](https://github.com/spf13/cobra)的伴侣。虽然两者都可以完全独立地运行，但它们构成了一个强大的组合，可以处理很多应用程序基础需求。

### 比较

> 备注： 可以到https://go.libhunt.com/ 这个网站进行库对比或者链接到其他博客网站

[viper VS ini](https://go.libhunt.com/compare-viper-vs-ini)

### 相似的库

[ini](https://github.com/go-ini/ini)


