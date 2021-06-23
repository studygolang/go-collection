# 装饰Go的错误

Go语言十分强大并且流行 — 许多项目都是用Go语言来实现的，如Kubernetes。Go语言一个有趣的特性是他能返回多个值， 这为Go语言提供了一种和其他编程语言不同的错误处理方式. Go将error视为具有预定义类型的值， 从技术上来说是一个interface。 然而，编写多层体系结构应用程序并使用api暴露应用的特性需要使用更多上下文信息进行错误处理，而不仅仅是一个值。 本文我们将探讨如何装饰Go错误类型以在应用程序中引入更多值。

## 用户自定义类型

我们将重写的Go里自带的`error` 类型 ，首先从一个自定义的错误类型开始，该错误类型将在应用程序中解释为`error`类型。 因此，我们将引入新的自定义错误接口，封装了Go的 `error` 。

```go
type GoError struct {
   error
}
```

## 上下文数据

当我们在Go中说错误是一个值时，它是字符串值 - 任何实现了`Error() string`函数的类型都可以视作error类型。将字符串值视为error会使跨层的错误解释复杂化，因为解释错误字符串不是正确的方法。因此让我们把字符串和错误码解耦：

```go
type GoError struct {
   error
   Code    string
}
```

现在对错误的解释将基于错误码`Code`字段而不是字符串。让我们通过上下文数据进一步对错误字符串进行解耦，在上下文数据中允许使用`i18n`包进行国际化。

```go
type GoError struct {
   error
   Code    string
   Data    map[string]interface{}
}
```

`Data`包含用于构造错误字符串的上下文数据。错误字符串可以通过数据模板化:

```
//i18N def
"InvalidParamValue": "Invalid parameter value '{{.actual}}', expected '{{.expected}}' for '{{.name}}'"
```

在i18N定义文件中，错误码将会映射到使用`Data`构建的模板化的错误字符串中。

## 错误原因

错误可能发生在任何一层，有必要为每一层提供解释错误的选项，并在不丢失原始错误值的情况下进一步使用附加的上下文信息对错误进行包装。`GoError`结构体可以用`Causes`进一步修饰，它将保存整个错误堆栈。

```
type GoError struct {
   error
   Code    string
   Data    map[string]interface{}
   Causes  []error
}
```

如果必须保存多个错误数据，则`causes`是一个数组类型，并将其设置为基本`error`类型，以便在应用程序中包含该原因的第三方错误。

## 组件

标记层组件将有助于识别错误发生在哪一层，并且可以避免不必要的错误包装。例如，如果`service`类型的错误组件发生在服务层，则可能不需要包装错误。检查组件信息将有助于防止暴露不应该通知用户的错误，比如数据库错误：

```go
type GoError struct {
   error
   Code      string
   Data      map[string]interface{}
   Causes    []error
   Component ErrComponent
}

type ErrComponent string
const (
   ErrService  ErrComponent = "service"
   ErrRepo     ErrComponent = "repository"
   ErrLib      ErrComponent = "library"
)
```

## 响应类型

添加错误响应类型将支持错误分类，以便于解释。例如，可以根据响应类型(如`NotFound`)对错误进行分类，像`DbRecordNotFound`、`ResourceNotFound`、`UserNotFound`等等的错误都可以归类为 `NotFound`错误。这在多层应用程序开发过程中非常有用，而且是可选的装饰：

```go
type GoError struct {
   error
   Code         string
   Data         map[string]interface{}
   Causes       []error
   Component    ErrComponent
   ResponseType ResponseErrType
}

type ResponseErrType string

const (
   BadRequest    ResponseErrType = "BadRequest"
   Forbidden     ResponseErrType = "Forbidden"
   NotFound      ResponseErrType = "NotFound"
   AlreadyExists ResponseErrType = "AlreadyExists"
)
```

## 重试

在少数情况下，错误将被重试。retry字段可以通过设置`Retryable`标记来决定是否为错误重试:

```
type GoError struct {
   error
   Code         string
   Message      string
   Data         map[string]interface{}
   Causes       []error
   Component    ErrComponent
   ResponseType ResponseErrType
   Retryable    bool
}
```

## GoError 接口

通过定义一个带有`GoError`实现的显式错误接口，可以简化错误检查:

```go
package goerr

type Error interface {
   error

   Code() string
   Message() string
   Cause() error
   Causes() []error
   Data() map[string]interface{}
   String() string
   ResponseErrType() ResponseErrType
   SetResponseType(r ResponseErrType) Error
   Component() ErrComponent
   SetComponent(c ErrComponent) Error
   Retryable() bool
   SetRetryable() Error
}
```

## 抽象错误

有了上述的装饰方式，更重要的是对错误进行抽象，将这些装饰保存在同一地方，并提供错误函数的可重用性

```go
func ResourceNotFound(id, kind string, cause error) GoError {
   data := map[string]interface{}{"kind": kind, "id": id}
   return GoError{
      Code:         "ResourceNotFound",
      Data:         data,
      Causes:       []error{cause},
      Component:    ErrService,
      ResponseType: NotFound,
      Retryable:    false,
   }
}
```

这个错误函数抽象了`ResourceNotFound`这个错误，开发者可以使用这个函数来返回错误对象而不是每次创建一个新的对象：

```go
//UserServiceuser, err := u.repo.FindUser(ctx, userId)
if err != nil {
   if err.ResponseType == NotFound {
      return ResourceNotFound(userUid, "User", err)
   }
   return err
}
```

## 结论

我们演示了如何使用添加上下文数据的自定义Go的错误类型，从而使得错误在多层应用程序中更有意义。你可以在[这里](https://gist.github.com/prathabk/744367cbfc70435c56956f650612d64b)看到完整的代码实现和定义。

## 原文链接

https://medium.com/spectro-cloud/decorating-go-error-d1db60bb9249
