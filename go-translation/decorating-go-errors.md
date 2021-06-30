# 封装Go中的error

Go语言很强大并且现在也十分流行 — 许多项目都是用Go语言来实现的，如Kubernetes。Go语言的一个有趣特性是它的多值返回功能提供了一种与其他编程语言不同的错误处理方法。 Go将error视为具有预定义类型的值，其本身是一个interface类型。然而，编写多层体系结构应用程序并使用api暴露应用的特性需要有包含更多上下文信息的error处理，而不仅仅是一个值。 本文我们将探讨如何封装Go的error类型以在应用程序中带来更大的价值。

## 用户自定义类型

我们将重写的Go里自带的`error`类型，首先从一个自定义的错误类型开始，该错误类型将在程序中识别为`error`类型。 因此，我们引入一个封装了Go的 `error`的新自定义Error类型。

```go
type GoError struct {
   error
}
```

## 上下文数据

当我们在Go中说error是一个值时，它是字符串值 - 任何实现了`Error() string`函数的类型都可以视作error类型。
将字符串值视为error会使跨层的error处理复杂化，因此处理error字符串信息并不是正确的方法。所以我们可以把字符串和错误码解耦：

```go
type GoError struct {
   error
   Code    string
}
```

现在对error的处理将基于错误码`Code`字段而不是字符串。让我们通过上下文数据进一步对错误字符串进行解耦，在上下文数据中可以使用`i18n`包进行国际化。

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

在i18N定义文件中，错误码`Code`将会映射到使用`Data`构建的模板化的错误字符串中。

## Causes

error可能发生在任何一层，有必要为每一层提供处理error的选项，并在不丢失原始error值的情况下进一步使用附加的上下文信息对error进行包装。`GoError`结构体可以用`Causes`进一步封装，用来保存整个错误堆栈。

```
type GoError struct {
   error
   Code    string
   Data    map[string]interface{}
   Causes  []error
}
```

如果必须保存多个error数据，则`causes`是一个数组类型，并将其设置为基本`error`类型，以便在程序中包含该原因的第三方错误。


## 组件(Component)

标记层组件将有助于识别error发生在哪一层，并且可以避免不必要的error wrap。例如，如果`service`类型的error组件发生在服务层，则可能不需要wrap error。检查组件信息将有助于防止暴露给用户不应该通知的error，比如数据库error：

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

## 响应类型(ResponseType)

添加一个错误响应类型这样可以支持error分类，以便于了解什么错误类型。例如，可以根据响应类型(如`NotFound`)对error进行分类，像`DbRecordNotFound`、`ResourceNotFound`、`UserNotFound`等等的error都可以归类为 `NotFound` error。这在多层应用程序开发过程中非常有用，而且是可选的封装：

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

在少数情况下，出现error会进行重试。retry字段可以通过设置`Retryable`标记来决定是否要进行error重试:

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

通过定义一个带有`GoError`实现的显式error接口，可以简化error检查:

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

## 抽象error

有了上述的封装方式，更重要的是对error进行抽象，将这些封装保存在同一地方，并提供error函数的可重用性

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

这个error函数抽象了`ResourceNotFound`这个error，开发者可以使用这个函数来返回error对象而不是每次创建一个新的对象：

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

我们演示了如何使用添加上下文数据的自定义Go的error类型，从而使得error在多层应用程序中更有意义。你可以在[这里](https://gist.github.com/prathabk/744367cbfc70435c56956f650612d64b)看到完整的代码实现和定义。

## 原文链接

https://medium.com/spectro-cloud/decorating-go-error-d1db60bb9249
