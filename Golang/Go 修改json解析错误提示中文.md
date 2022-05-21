# Go 修改Json解析错误提示中文

当使用json.Unmarshal时如果出现字段类型不匹配、json字符串错误则会返回一个内容为以下提示的Error。

```
json: cannot unmarshal xx into Go struct field ...
```

go中使用err.Error()将error类型转为字符串类型，

跳转到Error()方法，发现是个interface{}中的方法。

```go
type error interface {
    Error() string
}
```

所以具体实现Error()方法应该在返回的这个Error实际所属的类型。这个类型实现了Error()方法。

所以查看json包中decode.go文件，查看到其中定义的错误类型有三个

```go
// An UnmarshalTypeError describes a JSON value that was
// not appropriate for a value of a specific Go type.
type UnmarshalTypeError struct {
    Value  string       // description of JSON value - "bool", "array", "number -5"
    Type   reflect.Type // type of Go value it could not be assigned to
    Offset int64        // error occurred after reading Offset bytes
    Struct string       // name of the struct type containing the field
    Field  string       // the full path from root node to the field
}
// An UnmarshalFieldError describes a JSON object key that
// led to an unexported (and therefore unwritable) struct field.
//
// Deprecated: No longer used; kept for compatibility.
type UnmarshalFieldError struct {
    Key   string
    Type  reflect.Type
    Field reflect.StructField
}
// An InvalidUnmarshalError describes an invalid argument passed to Unmarshal.
// (The argument to Unmarshal must be a non-nil pointer.)
type InvalidUnmarshalError struct {
    Type reflect.Type
}
```

我使用的是Unmarshal所以对应的错误是UnmarshalTypeError，查看UnmarshalTypeError实现的Error()方法

```go
func (e *UnmarshalTypeError) Error() string {
    if e.Struct != "" || e.Field != "" {
        return "json: cannot unmarshal " + e.Value + " into Go struct field " + e.Struct + "." + e.Field + " of type " + e.Type.String()
    }
    return "json: cannot unmarshal " + e.Value + " into Go value of type " + e.Type.String()
}
```

参考方法就可以编写自己的Error方法了：

```go
/*
* @Content : 返回入参错误
* @Param   :
* @Return  :
* @Author  : LiJunDong
* @Time    : 2022-04-15
 */
func Error(err error) (msg string) {
    jsonErr, ok := err.(*json.UnmarshalTypeError)
    if ok {
        if jsonErr.Struct != "" || jsonErr.Field != "" {
            msg = "参数[" + jsonErr.Field + "]错误，要求[" + jsonErr.Type.String() + "]类型"
            // "json: cannot unmarshal " + e.Value + " into Go struct field " + e.Struct + "." + e.Field + " of type " + e.Type.String()
            // json: cannot unmarshal string into Go struct field DetailUserParam.id of type int64
        } else {
            msg = "请求参数解析错误"
            // "json: cannot unmarshal " + e.Value + " into Go value of type " + e.Type.String()
        }
    } else {
      	// 以下对针对Gin的ValidationErrors处理
        errs, ok := err.(validator.ValidationErrors)
        if ok {
            errList := make([]string, 0, len(errs))
            for _, e := range errs {
                errList = append(errList, e.Translate(Trans))
            }
            msg = strings.Join(errList, ",")
        } else {
            msg = "请求参数格式错误"
        }
    }
    return msg
}
```

使用

```go
err := json.Unmarshal(jsonStr, &data)
if err != nil {
  errStr := Error(err)
}
```

