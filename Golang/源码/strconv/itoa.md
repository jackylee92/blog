## FormatUnit


> 将unit64位（无符号整型，范围0到2^31）转为string类型
>
> 参数1:需要处理的unit64类型数据
>
> 参数2:进制数，
>
> strconv.FormatUint(data, 10) // 转为10进制数字字符串
>
> strconv.FormatUint(data, 16) //转为16进制数字字符串

__Exmaple：__

```go
var data uint64
data = 123456
result := strconv.FormatUint(data, 10)
fmt.Printf("Type: %T, Value: %v \n", result, result)
```

__源码：__

> /usr/local/go/src/strconv/itoa.go

```go
// FormatUint returns the string representation of i in the given base,
// for 2 <= base <= 36. The result uses the lower-case letters 'a' to 'z'
// for digit values >= 10.
func FormatUint(i uint64, base int) string {
        if fastSmalls && i < nSmalls && base == 10 {
                return small(int(i))
        }
        _, s := formatBits(nil, i, base, false, false)
        return s
}
```

接手两个参数：第一个为需要处理的Uinit64位值，第二个为进制数，返回一个string

* ``fastSmalls && i < nSmalls && base == 10 `` 

  fastSmalls:为是否开启小数字快速处理

  ``  11  const fastSmalls = true // enable fast path for small integers``

  nSmalls:设置的小数字范围 100

  ``  68  const nSmalls = 100``

  ``base == 10``近支持转10进制时可以快速处理小数字

  __处理小的数字逻辑入下：__

  ```go
  // small returns the string for an i with 0 <= i < nSmalls.
  func small(i int) string {
          if i < 10 {
                  return digits[i : i+1]
          }
          return smallsString[i*2 : i*2+2]
  }
  ```

  ``<10``时会取digits下标的字符串直接返回
  
  ```go
  20 const digits = "0123456789abcdefghijklmnopqrstuvwxyz"
  ```
  
  * ``>=10 && <100`` 时会取smallsString中对应下表范围的字符串
  
  ```go
  70  const smallsString = "00010203040506070809" +
        "10111213141516171819" +
        "20212223242526272829" +
        "30313233343536373839" +
        "40414243444546474849" +
        "50515253545556575859" +
        "60616263646566676869" +
        "70717273747576777879" +
        "80818283848586878889" +
        "90919293949596979899"
  ```

  如果不是小数字或者转换的进制不是十进制，则使用formatBits方法处理
  
  ``_, s := formatBits(nil, i, base, false, false)``
  
  我们看看formatBits函数内部：
  
  ```go
  // formatBits computes the string representation of u in the given base.
  // If neg is set, u is treated as negative int64 value. If append_ is
  // set, the string is appended to dst and the resulting byte slice is
  // returned as the first result value; otherwise the string is returned
  // as the second result value.
  //
  func formatBits(dst []byte, u uint64, base int, neg, append_ bool) (d []byte, s string) {
      if base < 2 || base > len(digits) {
          panic("strconv: illegal AppendInt/FormatInt base")
      }
      // 2 <= base && base <= len(digits)
  
      var a [64 + 1]byte // +1 for sign of 64bit value in base 2，多一位为了处理负数情况前面需要加-
      i := len(a)
  
      if neg {
          u = -u // 如果是负数，则先转为正数
      }
  
      // convert bits
      // We use uint values where we can because those will
      // fit into a single register even on a 32bit machine.
      if base == 10 {
          // common case: use constants for / because
          // the compiler can optimize it into a multiply+shift
  
          if host32bit {
              // convert the lower digits using 32bit operations
              for u >= 1e9 {
                  // Avoid using r = a%b in addition to q = a/b
                  // since 64bit division and modulo operations
                  // are calculated by runtime functions on 32bit machines.
                  q := u / 1e9
                  us := uint(u - q*1e9) // u % 1e9 fits into a uint
                  for j := 4; j > 0; j-- {
                      is := us % 100 * 2
                      us /= 100
                      i -= 2
                      a[i+1] = smallsString[is+1]
                      a[i+0] = smallsString[is+0]
                  }
  
                  // us < 10, since it contains the last digit
                  // from the initial 9-digit us.
                  i--
                  a[i] = smallsString[us*2+1]
  
                  u = q
              }
              // u < 1e9
          }
  
          // u guaranteed to fit into a uint
          us := uint(u)
          for us >= 100 {
              is := us % 100 * 2
              us /= 100
              i -= 2
              a[i+1] = smallsString[is+1]
              a[i+0] = smallsString[is+0]
          }
  
          // us < 100
          is := us * 2
          i--
          a[i] = smallsString[is+1]
          if us >= 10 {
              i--
              a[i] = smallsString[is]
          }
  
      } else if isPowerOfTwo(base) {
          // Use shifts and masks instead of / and %.
          // Base is a power of 2 and 2 <= base <= len(digits) where len(digits) is 36.
          // The largest power of 2 below or equal to 36 is 32, which is 1 << 5;
          // i.e., the largest possible shift count is 5. By &-ind that value with
          // the constant 7 we tell the compiler that the shift count is always
          // less than 8 which is smaller than any register width. This allows
          // the compiler to generate better code for the shift operation.
          shift := uint(bits.TrailingZeros(uint(base))) & 7
          b := uint64(base)
          m := uint(base) - 1 // == 1<<shift - 1
          for u >= b {
              i--
              a[i] = digits[uint(u)&m]
              u >>= shift
          }
          // u < base
          i--
          a[i] = digits[uint(u)]
      } else {
          // general case
          b := uint64(base)
          for u >= b {
              i--
              // Avoid using r = a%b in addition to q = a/b
              // since 64bit division and modulo operations
              // are calculated by runtime functions on 32bit machines.
              q := u / b
              a[i] = digits[uint(u-q*b)]
              u = q
          }
          // u < base
          i--
          a[i] = digits[uint(u)]
      }
      // add sign, if any 负数需要加上-
      if neg {
          i--
          a[i] = '-'
      }
  
      if append_ { // 如果是追加，就append到[]byte中
          d = append(dst, a[i:]...)
          return
      }
      s = string(a[i:])
      return
  }
  func isPowerOfTwo(x int) bool {
      return x&(x-1) == 0
  }
  ```
  
  比较长。
  
  第一个判断：对base(需要转换的进制数)做验证 需要在2-36之间，否则panic
  
  这个地方需要注意，在你程序处理数据时，由于这个panic会导致进程挂掉，所以对数值不确定的情况下最好在上层捕捉这个panic，而不是直接panic程序挂掉。
  
  ``var a [64 + 1]byte ``返回字符串的byte类型
  
  
  
  __当转换为十进制时，在32上对byte的截取逻辑不同于64位__
  
  在64位上会循环处理
  
  ``is := us % 100 * 2``为最后两个数字在smallsString上的位置，两个字符的位置分别时``is~ is+1``，通过下标获取到的是ascll码，将获取到ascll码一次插入[]byte类型a中，最后将a强转为string类型。
  
  
  
  __对于非转10进制的处理方式如下：__
  
  isPowerOfTwo判断入参是否是2的n次方，这里面步骤为将x和(x-1)转位二进制后位运算(&)为0则表示是2的n次方。
  
  ``bits.TrailingZeros``返回入参转二进制后的尾随零个数
  
  后面这个算法 有点复杂。。。（各种位运算）
  
  ``const uintSize = 32 << (^uint(0) >> 32 & 1) // 32 or 64``判断当前机器是32位还是64位

## FormatInt

类似``FormatUnit``函数处理，只是在small和formatBites传参时将int64强转位int类型

```go
// FormatInt returns the string representation of i in the given base,
// for 2 <= base <= 36. The result uses the lower-case letters 'a' to 'z'
// for digit values >= 10.
func FormatInt(i int64, base int) string {
    if fastSmalls && 0 <= i && i < nSmalls && base == 10 {
        return small(int(i))
    }
    _, s := formatBits(nil, uint64(i), base, i < 0, false)
    return s
}
```

## Itoa

调用了FormatInt，只是在FormatInt入参强转位int64位处理

```go
// Itoa is equivalent to FormatInt(int64(i), 10).
func Itoa(i int) string {
    return FormatInt(int64(i), 10)
}
```

## AppendInt

将一个int64位数字追加到[]byte后面，返回[]byte

当需要追加的int64数字位为0-100范围内，使用small返回该int64位数字的字符串类型，并append到[]byte中。

```go
// AppendInt appends the string form of the integer i,
// as generated by FormatInt, to dst and returns the extended buffer.
func AppendInt(dst []byte, i int64, base int) []byte {
    if fastSmalls && 0 <= i && i < nSmalls && base == 10 {
        return append(dst, small(int(i))...)
    }
    dst, _ = formatBits(dst, uint64(i), base, i < 0, true)
    return dst
}
```

## AppendUint

和AppendInt类似，只是在调用small、formatBits时入参强转位int类型。

```go
// AppendUint appends the string form of the unsigned integer i,
// as generated by FormatUint, to dst and returns the extended buffer.
func AppendUint(dst []byte, i uint64, base int) []byte {
    if fastSmalls && i < nSmalls && base == 10 {
        return append(dst, small(int(i))...)
    }
    dst, _ = formatBits(dst, i, base, false, true)
    return dst
}
```

