## ParseBool

通过字符串判断true/false，函数很简单，就是通过一个switch case 返回true或者false，如果没有case命中，则会返回false，syntaxError("ParseBool", str)。

返回的错误是一个结构体

```go
// ErrSyntax indicates that a value does not have the right syntax for the target type.
var ErrSyntax = errors.New("invalid syntax")

// A NumError records a failed conversion.
type NumError struct {
    Func string // the failing function (ParseBool, ParseInt, ParseUint, ParseFloat)
    Num  string // the input
    Err  error  // the reason the conversion failed (e.g. ErrRange, ErrSyntax, etc.)
}

func (e *NumError) Error() string {
    return "strconv." + e.Func + ": " + "parsing " + Quote(e.Num) + ": " + e.Err.Error()
}

func syntaxError(fn, str string) *NumError {
    return &NumError{fn, str, ErrSyntax}
}
```

当调用syntaxError时，实例话一个NumError的结构体，结构体中Err是Error()方法返回的string，经过errors,New()转为error类型。



true："1", "t", "T", "true", "TRUE", "True"

fales: "0", "f", "F", "false", "FALSE", "False"

```go
// ParseBool returns the boolean value represented by the string.
// It accepts 1, t, T, TRUE, true, True, 0, f, F, FALSE, false, False.
// Any other value returns an error.
func ParseBool(str string) (bool, error) {
    switch str {
    case "1", "t", "T", "true", "TRUE", "True":
        return true, nil
    case "0", "f", "F", "false", "FALSE", "False":
        return false, nil
    }
    return false, syntaxError("ParseBool", str)
}
```



## FormatBoll

将bool值转为string(true/false)类型，代码很简单就是一个if判断

```go
// FormatBool returns "true" or "false" according to the value of b.
func FormatBool(b bool) string {
    if b {
        return "true"
    }
    return "false"
}
```



## AppendBool

将bool值转为[]byte追加到指定[]byte中，代码很简单一个if加一个append

```go
// AppendBool appends "true" or "false", according to the value of b,
// to dst and returns the extended buffer.
func AppendBool(dst []byte, b bool) []byte {
    if b {
        return append(dst, "true"...)
    }
    return append(dst, "false"...)
}
```



## equalIgnoreCase

内部一个忽略大小写对比字符串是否相等的函数。

主要是将两个字符串中每个字符如果是大写字母就转为小写字母，在逐个比较

```go
func equalIgnoreCase(s1, s2 string) bool {
    if len(s1) != len(s2) {
        return false
    }
    for i := 0; i < len(s1); i++ {
        c1 := s1[i]
        if 'A' <= c1 && c1 <= 'Z' {
            c1 += 'a' - 'A'
        }
        c2 := s2[i]
        if 'A' <= c2 && c2 <= 'Z' {
            c2 += 'a' - 'A'
        }
        if c1 != c2 {
            return false
        }
    }
    return true
}
```

## ParseFloat

将字符串转为float64

