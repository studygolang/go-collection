一个用于字符串、数字、切片和结构体的校验库和过滤库。基于[validator.js](https://github.com/chriso/validator.js)。

### 安装

在终端中输入以下命令：

```bash
go get github.com/asaskevich/govalidator
```

或者你可以get指定版本的包与`gopkg.in`:

```bash
go get gopkg.in/asaskevich/govalidator.v10
```

### 开启此功能，默认情况下会校验所有字段

`SetFieldsRequiredByDefault`

当结构体中的字段不包括valid或未明确标记为忽略（即使用`valid:"-"`或`valid:"email,optional"`）都会导致校验失败，另外开启此功能的话可以把它放在一个包的 init() 方法或main()方法。

`SetNilPtrAllowedByRequired` 

允许结构体中`required`标记的字段为nil。默认情况下，为了保持一致性，会禁用此功能。但是一些在`nil`值和`zero value`中间状态的值的包是可以使用的。如果禁用的话，`nil`值和`zero`值都会校验失败。

```go
import "github.com/asaskevich/govalidator"

func init() {
  govalidator.SetFieldsRequiredByDefault(true)
}
```

下面是一些代码示例：

```go
// 这个使用govalidator.ValidateStruct()会校验失败(参数的值不匹配)
type exampleStruct struct {
  Name  string ``
  Email string `valid:"email"`
}

// 当email为空或无效地址的话会校验失败
type exampleStruct2 struct {
  Name  string `valid:"-"`
  Email string `valid:"email"`
}

// email不为空且地址无效会校验失败
type exampleStruct2 struct {
  Name  string `valid:"-"`
  Email string `valid:"email,optional"`
}
```

### 最近的重大更改（见[#123](https://github.com/asaskevich/govalidator/pull/123))

##### 自定义validator函数

可以将context上下文作为第二个参数，如果以struct对象传参的话就会被校验-这使得依赖验证成为可能。。

```go
import "github.com/asaskevich/govalidator"

// 旧函数签名
func(i interface{}) bool

// 新函数签名
func(i interface{}, o interface{}) bool
```

##### 添加自定义validator

这是为了防止访问自定义validator时出现数据竞争。

```go
import "github.com/asaskevich/govalidator"

// 旧的方法
govalidator.CustomTypeTagMap["customByteArrayValidator"] = func(i interface{}, o interface{}) bool {
  // ...
}

// 新的方法
govalidator.CustomTypeTagMap.Set("customByteArrayValidator", func(i interface{}, o interface{}) bool {
  // ...
})
```

### 函数列表

```go
func Abs(value float64) float64
func BlackList(str, chars string) string
func ByteLength(str string, params ...string) bool
func CamelCaseToUnderscore(str string) string
func Contains(str, substring string) bool
func Count(array []interface{}, iterator ConditionIterator) int
func Each(array []interface{}, iterator Iterator)
func ErrorByField(e error, field string) string
func ErrorsByField(e error) map[string]string
func Filter(array []interface{}, iterator ConditionIterator) []interface{}
func Find(array []interface{}, iterator ConditionIterator) interface{}
func GetLine(s string, index int) (string, error)
func GetLines(s string) []string
func HasLowerCase(str string) bool
func HasUpperCase(str string) bool
func HasWhitespace(str string) bool
func HasWhitespaceOnly(str string) bool
func InRange(value interface{}, left interface{}, right interface{}) bool
func InRangeFloat32(value, left, right float32) bool
func InRangeFloat64(value, left, right float64) bool
func InRangeInt(value, left, right interface{}) bool
func IsASCII(str string) bool
func IsAlpha(str string) bool
func IsAlphanumeric(str string) bool
func IsBase64(str string) bool
func IsByteLength(str string, min, max int) bool
func IsCIDR(str string) bool
func IsCRC32(str string) bool
func IsCRC32b(str string) bool
func IsCreditCard(str string) bool
func IsDNSName(str string) bool
func IsDataURI(str string) bool
func IsDialString(str string) bool
func IsDivisibleBy(str, num string) bool
func IsEmail(str string) bool
func IsExistingEmail(email string) bool
func IsFilePath(str string) (bool, int)
func IsFloat(str string) bool
func IsFullWidth(str string) bool
func IsHalfWidth(str string) bool
func IsHash(str string, algorithm string) bool
func IsHexadecimal(str string) bool
func IsHexcolor(str string) bool
func IsHost(str string) bool
func IsIP(str string) bool
func IsIPv4(str string) bool
func IsIPv6(str string) bool
func IsISBN(str string, version int) bool
func IsISBN10(str string) bool
func IsISBN13(str string) bool
func IsISO3166Alpha2(str string) bool
func IsISO3166Alpha3(str string) bool
func IsISO4217(str string) bool
func IsISO693Alpha2(str string) bool
func IsISO693Alpha3b(str string) bool
func IsIn(str string, params ...string) bool
func IsInRaw(str string, params ...string) bool
func IsInt(str string) bool
func IsJSON(str string) bool
func IsLatitude(str string) bool
func IsLongitude(str string) bool
func IsLowerCase(str string) bool
func IsMAC(str string) bool
func IsMD4(str string) bool
func IsMD5(str string) bool
func IsMagnetURI(str string) bool
func IsMongoID(str string) bool
func IsMultibyte(str string) bool
func IsNatural(value float64) bool
func IsNegative(value float64) bool
func IsNonNegative(value float64) bool
func IsNonPositive(value float64) bool
func IsNotNull(str string) bool
func IsNull(str string) bool
func IsNumeric(str string) bool
func IsPort(str string) bool
func IsPositive(value float64) bool
func IsPrintableASCII(str string) bool
func IsRFC3339(str string) bool
func IsRFC3339WithoutZone(str string) bool
func IsRGBcolor(str string) bool
func IsRegex(str string) bool
func IsRequestURI(rawurl string) bool
func IsRequestURL(rawurl string) bool
func IsRipeMD128(str string) bool
func IsRipeMD160(str string) bool
func IsRsaPub(str string, params ...string) bool
func IsRsaPublicKey(str string, keylen int) bool
func IsSHA1(str string) bool
func IsSHA256(str string) bool
func IsSHA384(str string) bool
func IsSHA512(str string) bool
func IsSSN(str string) bool
func IsSemver(str string) bool
func IsTiger128(str string) bool
func IsTiger160(str string) bool
func IsTiger192(str string) bool
func IsTime(str string, format string) bool
func IsType(v interface{}, params ...string) bool
func IsURL(str string) bool
func IsUTFDigit(str string) bool
func IsUTFLetter(str string) bool
func IsUTFLetterNumeric(str string) bool
func IsUTFNumeric(str string) bool
func IsUUID(str string) bool
func IsUUIDv3(str string) bool
func IsUUIDv4(str string) bool
func IsUUIDv5(str string) bool
func IsULID(str string) bool
func IsUnixTime(str string) bool
func IsUpperCase(str string) bool
func IsVariableWidth(str string) bool
func IsWhole(value float64) bool
func LeftTrim(str, chars string) string
func Map(array []interface{}, iterator ResultIterator) []interface{}
func Matches(str, pattern string) bool
func MaxStringLength(str string, params ...string) bool
func MinStringLength(str string, params ...string) bool
func NormalizeEmail(str string) (string, error)
func PadBoth(str string, padStr string, padLen int) string
func PadLeft(str string, padStr string, padLen int) string
func PadRight(str string, padStr string, padLen int) string
func PrependPathToErrors(err error, path string) error
func Range(str string, params ...string) bool
func RemoveTags(s string) string
func ReplacePattern(str, pattern, replace string) string
func Reverse(s string) string
func RightTrim(str, chars string) string
func RuneLength(str string, params ...string) bool
func SafeFileName(str string) string
func SetFieldsRequiredByDefault(value bool)
func SetNilPtrAllowedByRequired(value bool)
func Sign(value float64) float64
func StringLength(str string, params ...string) bool
func StringMatches(s string, params ...string) bool
func StripLow(str string, keepNewLines bool) string
func ToBoolean(str string) (bool, error)
func ToFloat(str string) (float64, error)
func ToInt(value interface{}) (res int64, err error)
func ToJSON(obj interface{}) (string, error)
func ToString(obj interface{}) string
func Trim(str, chars string) string
func Truncate(str string, length int, ending string) string
func TruncatingErrorf(str string, args ...interface{}) error
func UnderscoreToCamelCase(s string) string
func ValidateMap(inputMap map[string]interface{}, validationMap map[string]interface{}) (bool, error)
func ValidateStruct(s interface{}) (bool, error)
func WhiteList(str, chars string) string
type ConditionIterator
type CustomTypeValidator
type Error
func (e Error) Error() string
type Errors
func (es Errors) Error() string
func (es Errors) Errors() []error
type ISO3166Entry
type ISO693Entry
type InterfaceParamValidator
type Iterator
type ParamValidator
type ResultIterator
type UnsupportedTypeError
func (e *UnsupportedTypeError) Error() string
type Validator
```

### 例子

#### 判断是否为URL(IsURL)

```go
println(govalidator.IsURL(`http://user@pass:domain.com/path/page`)) // false
```

#### 判断类型(IsType)

```go
println(govalidator.IsType("Bob", "string")) // true
println(govalidator.IsType(1, "int")) // true
i := 1
println(govalidator.IsType(&i, "*int")) // true
```

IsType也可以在struct tag 用type使用， 这对map校验很有必要的：

```go
type User	struct {
  Name string      `valid:"type(string)"`
  Age  int         `valid:"type(int)"`
  Meta interface{} `valid:"type(string)"`
}
result, err := govalidator.ValidateStruct(User{"Bob", 20, "meta"})
if err != nil {
	println("error: " + err.Error())
}
println(result) // true
```

#### ToString

```go
type User struct {
	FirstName string
	LastName string
}

str := govalidator.ToString(User{"John", "Juan"})
println(str) // {John Juan}
```

#### 遍历(Each)，Map，过滤，切片计数

遍历迭代array和slice会调用Iterator

```go
data := []interface{}{1, 2, 3}
var fn govalidator.Iterator = func(value interface{}, index int) {
	println(value.(int))
}
govalidator.Each(data, fn)
//1
//2
//3
```

```go
data := []interface{}{1, 2, 3, 4, 5}
var fn govalidator.ResultIterator = func(value interface{}, index int) interface{} {
	return value.(int) * 3
}
_ = govalidator.Map(data, fn) // result = []interface{}{1, 6, 9, 12, 15}
```

```go
data := []interface{}{1, 2, 3, 4, 5, 6, 7, 8, 9, 10}
var fn govalidator.ConditionIterator = func(value interface{}, index int) bool {
	return value.(int)%2 == 0
}
_ = govalidator.Filter(data, fn) // result = []interface{}{2, 4, 6, 8, 10}
_ = govalidator.Count(data, fn) // result = 5
```

#### 校验struct

如果要校验structs，你可以在struct的任何字段中使用`valid` tag。使用此字段的所有validator在一个tag中均按逗号进行分隔。如果你想跳过校验，请将`-`放在你的tag中。如果下面列表中没有你需要的validator，你可以按下面例子添加：

```go
govalidator.TagMap["duck"] = govalidator.Validator(func(str string) bool {
	return str == "duck"
})
```

有关完全自定义validator（基于接口），请参阅下文。

下面是在结构体字段中的可以使用的validator列表（validator - 使用方法）：

```go
"email":              IsEmail,
"url":                IsURL,
"dialstring":         IsDialString,
"requrl":             IsRequestURL,
"requri":             IsRequestURI,
"alpha":              IsAlpha,
"utfletter":          IsUTFLetter,
"alphanum":           IsAlphanumeric,
"utfletternum":       IsUTFLetterNumeric,
"numeric":            IsNumeric,
"utfnumeric":         IsUTFNumeric,
"utfdigit":           IsUTFDigit,
"hexadecimal":        IsHexadecimal,
"hexcolor":           IsHexcolor,
"rgbcolor":           IsRGBcolor,
"lowercase":          IsLowerCase,
"uppercase":          IsUpperCase,
"int":                IsInt,
"float":              IsFloat,
"null":               IsNull,
"uuid":               IsUUID,
"uuidv3":             IsUUIDv3,
"uuidv4":             IsUUIDv4,
"uuidv5":             IsUUIDv5,
"creditcard":         IsCreditCard,
"isbn10":             IsISBN10,
"isbn13":             IsISBN13,
"json":               IsJSON,
"multibyte":          IsMultibyte,
"ascii":              IsASCII,
"printableascii":     IsPrintableASCII,
"fullwidth":          IsFullWidth,
"halfwidth":          IsHalfWidth,
"variablewidth":      IsVariableWidth,
"base64":             IsBase64,
"datauri":            IsDataURI,
"ip":                 IsIP,
"port":               IsPort,
"ipv4":               IsIPv4,
"ipv6":               IsIPv6,
"dns":                IsDNSName,
"host":               IsHost,
"mac":                IsMAC,
"latitude":           IsLatitude,
"longitude":          IsLongitude,
"ssn":                IsSSN,
"semver":             IsSemver,
"rfc3339":            IsRFC3339,
"rfc3339WithoutZone": IsRFC3339WithoutZone,
"ISO3166Alpha2":      IsISO3166Alpha2,
"ISO3166Alpha3":      IsISO3166Alpha3,
"ulid":               IsULID,
```

带参数的validator

```go
"range(min|max)": Range,"length(min|max)": ByteLength,"runelength(min|max)": RuneLength,"stringlength(min|max)": StringLength,"matches(pattern)": StringMatches,"in(string1|string2|...|stringN)": IsIn,"rsapub(keylength)" : IsRsaPub,"minstringlength(int): MinStringLength,"maxstringlength(int): MaxStringLength,
```

带任何类型参数的validator

```go
"type(type)": IsType,
```

下面是使用的小例子：

```go
type Post struct {
	Title    string `valid:"alphanum,required"`
	Message  string `valid:"duck,ascii"`
	Message2 string `valid:"animal(dog)"`
	AuthorIP string `valid:"ipv4"`
	Date     string `valid:"-"`
}
post := &Post{
	Title:   "My Example Post",
	Message: "duck",
	Message2: "dog",
	AuthorIP: "123.234.54.3",
}

// 可以添加自己想要检验的tags
govalidator.TagMap["duck"] = govalidator.Validator(func(str string) bool {
	return str == "duck"
})

// 可以添加自己想要检验的参数tags
govalidator.ParamTagMap["animal"] = govalidator.ParamValidator(func(str string, params ...string) bool {
    species := params[0]
    return str == species
})
govalidator.ParamTagRegexMap["animal"] = regexp.MustCompile("^animal\\((\\w+)\\)$")

result, err := govalidator.ValidateStruct(post)
if err != nil {
	println("error: " + err.Error())
}
println(result)
```

#### 校验Map

如果你想要校验Map，可以用需要校验的Map和包含在ValidateStruct使用的相同Tag的校验Map，并且两个Map必须是`map[string]interface{}`的类型

下面是使用的小例子：

```go
var mapTemplate = map[string]interface{}{
	"name":"required,alpha",
	"family":"required,alpha",
	"email":"required,email",
	"cell-phone":"numeric",
	"address":map[string]interface{}{
		"line1":"required,alphanum",
		"line2":"alphanum",
		"postal-code":"numeric",
	},
}

var inputMap = map[string]interface{}{
	"name":"Bob",
	"family":"Smith",
	"email":"foo@bar.baz",
	"address":map[string]interface{}{
		"line1":"",
		"line2":"",
		"postal-code":"",
	},
}

result, err := govalidator.ValidateMap(inputMap, mapTemplate)
if err != nil {
	println("error: " + err.Error())
}
println(result)
```

#### 白名单

```go
// 从一个string中移除a-z之外的所有字符
println(govalidator.WhiteList("a3a43a5a4a3a2a23a4a5a4a3a4", "a-z") == "aaaaaaaaaaaa")
```

#### 自定义校验函数

允许使用你自定义特定域的validators - 例子如下：

```go
import "github.com/asaskevich/govalidator"

type CustomByteArray [6]byte // 支持自定义类型并可以进行校验

type StructWithCustomByteArray struct {
  ID              CustomByteArray `valid:"customByteArrayValidator,customMinLengthValidator"` // 多个自定义validator也是可以的，并将按顺序进行处理
  Email           string          `valid:"email"`
  CustomMinLength int             `valid:"-"`
}

govalidator.CustomTypeTagMap.Set("customByteArrayValidator", func(i interface{}, context interface{}) bool {
  switch v := context.(type) { 
  case StructWithCustomByteArray:
   
  case SomeOtherType:
    // ...
  default:
    // 
  }

  switch v := i.(type) { 
  case CustomByteArray:
    for _, e := range v { 
      if e != 0 {
        return true
      }
    }
  }
  return false
})
govalidator.CustomTypeTagMap.Set("customMinLengthValidator", func(i interface{}, context interface{}) bool {
  switch v := context.(type) { // 会根据另一个字段中的值校验一个字段，即依赖的校验
  case StructWithCustomByteArray:
    return len(v.ID) >= v.CustomMinLength
  }
  return false
})
```

#### 遍历Error

默认情况下，Error()在单个字符串中返回所有错误。要获取每个error，可以这样做：

```go
  if err != nil {
    errs := err.(govalidator.Errors).Errors()
    for _, e := range errs {
      fmt.Println(e.Error())
    }
  }
```

#### 自定义error消息

通过添加`~`分隔号可以自定义错误消息-例子如下

```go
type Ticket struct {
  Id        int64     `json:"id"`
  FirstName string    `json:"firstname" valid:"required~First name is blank"`
}
```

### 相关库比较

| 库名                                       | 介绍                                                         | star数 |
| ------------------------------------------ | ------------------------------------------------------------ | ------ |
| https://github.com/gookit/validate         | Go通用的数据验证与过滤库，使用简单，内置大部分常用验证、过滤器，支持自定义验证器、自定义消息、字段翻译。咱国人开发的。文档可参考 https://github.com/gookit/validate/blob/master/README.zh-CN.md | 500    |
| https://github.com/go-ozzo/ozzo-validation | 一个常用的Go验证包。支持使用普通语言构造而不是容易出错的struct标记的可配置和可扩展的验证规则的校验库。 | 2.1k   |
| https://github.com/go-playground/validator | Go 结构体和字段校验库, 包括跨字段, 跨Struct, Map, Slice和Array。另外gin也默认支持。可参考大俊大佬[Go 每日一库之 validator](https://mp.weixin.qq.com/s/_cqeNbKeNayW0dHx6ROaNA) | 7.8k   |
| https://github.com/asaskevich/govalidator  | 一个用于字符串、数字、切片和结构体的校验库和过滤库。基于[validator.js](https://github.com/chriso/validator.js)。 | 4.7k   |

