# Go 构造函数

## 优点

* 扩展性强，可对单独结构体添加字段，单独方法进行扩展

* 支持自定义默认值

## 思路

将不定参数独立成一个结构体A

定义一个结构体S，这个结构体包含一个函数属性，这个函数入参是A的指针

将A中每个属性写一个设置函数，这个函数结构和S中的函数相同，将函数设置S的函数属性，这样每个属性都对应一个S的实例，只是实例中的函数分别是修改入参A指针中的自己属性。

## 代码：

```go
// 用户基本属性
type User struct {
    Name string
    Age int
    Ext userExt
}
// 用户附加属性
type userExt struct {
    CheZi string
    Qian string
    FangZi string
}
// 附加属性默认值
const (
    defaultCheZi = "无车"
    defaultQian = "无存款"
    defaultFangZi = "无房"
)
// 对外，通过apply方法设置入参的这个属性，这个属性是个指针
type UserExtITF interface{
    apply(*userExt)
}
// 定义的设置属性的函数类型
type setFunc func(*userExt)
// 定义一个结构体，f属性是设置属性的方法
// 关键：
//   每个新增的字段都实例化一个这个类型的变量，只是其中的f对应的方法实现不同
//   实现了UserExtITF，所以对外类型可以统一
type setFuncS struct{
    f setFunc
}
// 上面定义的这个结构体实现了UseExtITF接口，
// 实现的内容是执行这个结构体的方法属性，这个方法其实就是设置属性值
func (s setFuncS)apply(ext *userExt){
    s.f(ext)
}
// 每个属性都会调用这个方法创建setFuncS实例
// 传入的f是对应设置本身的方法(方法的入参是所有属性同一个实例的指针)
func newSetFuncS(f setFunc) *setFuncS {
    return &setFuncS{
        f: f,
    }
}
// 获取默认值
func getDefaultExt()(data userExt){
    data.FangZi = defaultFangZi
    data.CheZi = defaultCheZi
    data.Qian = defaultQian
    return data
}
// 重要：
//   返回个UseExtITF interface，实际是setFuncS，只是f属性的函数是设置CheZi
func WithCheZi(data string) UserExtITF {
    return newSetFuncS(func(ext *userExt){
        ext.CheZi = data
    })
}
// 重要：
//   返回个UseExtITF interface，实际是setFuncS，只是f属性的函数是设置FangZi
func WithFangZi(data string) UserExtITF {
    return newSetFuncS(func(ext *userExt){
        ext.FangZi = data
    })
}
// 重要：
//   返回个UseExtITF interface，实际是setFuncS，只是f属性的函数是设置Qian
func WithQian(data string) UserExtITF {
    return newSetFuncS(func(ext *userExt){
        ext.Qian = data
    })
}
// 对外暴露的创建用户方法
func NewUser(name string, age int, exts ...UserExtITF) *User {
    user := User{
        Name: name,
        Age:  age,
        // 获取初始属性
        Ext:  getDefaultExt(),
    }
    for _, item := range exts {
        // item为每个实现了Interface的setFuncS实例，入参为属性共同的对象指针
        item.apply(&user.Ext)
    }
    return &user
}
```

```go
e := []UserExtITF{
    WithCheZi("奥迪"),
    WithFangZi("小区"),
}
u := NewUser("aaa", 100, e...)
log.Println(u)
```

## 扩展属性(YanZhi)

* userExt 中添加字段 YanZhi

* 为这个字段添加个WithYanZhi方法

* 可以设置默认值，defaultYanZhi , getDefaultExt()中添加设置YanZhi

* 使用就可以在``WithFangZi("小区"),`` 后面添加``WithYanZhi("高"),``
