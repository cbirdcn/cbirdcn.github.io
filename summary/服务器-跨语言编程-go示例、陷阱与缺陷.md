# 服务端重点-跨语言编程-Go示例、陷阱与缺陷

## 语法陷阱

### 初始化零值

未进行初始化的变量被声明为零值而不是`<nil>`。例如，int类型的零值为0。

```go
var e int
fmt.Println(e)

// 输出
0
```

### 常量定义

Go supports constants of character, string, boolean, and numeric values.

也就是说，常量是在编译期检查，需要确定的基本类型，不能出现函数、struct等

const定义可以出现在函数内或函数外，任何能使用`var`定义变量的位置都能用`const`定义常量

```go
const s string = "constant"
const n = 500000000
const d = 3e20 / n
const n = math.Pow(10,2) // 报错，函数

type Node struct {}
var node1 = Node{} // 不报错
const node2 = Node{} // 报错
```

### 条件与循环

#### for

go的循环只有for一种语法。

for支持
- `initial/condition/after`
- `condition`，类似与其他语言的`while`
- 当`condition`被省略时表示`true`进入死循环，直到遇到break才能跳出循环，或者遇到return返回函数

continue进入本循环语句的下一次迭代iteration

注意loop语句中变量的作用域，如果是在for语句内才定义的变量，就不能被loop外访问。就像第二个loop中的j

语句不包含`()`，`:`等

左花括号需要与for语句同行，因为跨行且没有任何连接符(比如`,`)的语句会被Go认为是两个语句，报错语法错误

```go
    i := 1
    for i <= 3 {
        fmt.Println(i)
        i = i + 1
    }
    fmt.Println(i) // 不报错

    for j := 7; j <= 9; j++ {
        fmt.Println(j)
    }
    fmt.Println(j) // 报错

    for {
        fmt.Println("loop")
        break
    }
```

Go支持`i++`语法，但不支持`++i`语法，避免歧义。

如果for语句包含两个变量要注意：
- initial部分只能是一个完整的表达式，也就是说有一个赋值符号，要写成`i , j := 1, 1.0`
- `after`部分也是一个完整的表达式，由于`i++`本身就是一个表达式，所以多个变量都要写成`i, j = i+1, j+1`

这叫做`平行赋值`

```go
    // for语句包含多个变量

    // 错误的示范
    for j := 7, k := 7; j <= 9 && k <= 9; j, k = j++, k++ {
        fmt.Println(j)
        fmt.Println(k)
    }

    // 正确的示范
    for j, k := 7, 7.0; j <= 9 && k <= 9; j, k = j+1, k+1 {
        fmt.Println(j)
        fmt.Println(k)
    }
```

#### if else

注意`else if`是分开的，不是`elseif`

多个条件用`&&`或`||`

在Go中没有三元if，所以即使对于基本条件，也需要使用完整的if语句。这是为了避免出现复杂的三元if语句。

```go
 if num := 9; num < 0 {
     fmt.Println(num, "is negative")
 } else if num < 10 {
     fmt.Println(num, "has 1 digit")
 } else {
     fmt.Println(num, "has multiple digits")
 }
```

if语句中同样可以定义变量，但是只有特殊场景才这么做，比如打开文件或取字典项。同样只支持平行赋值，而不支持多次赋值

```go
if err := file.Chmod(0664); err != nil {
    fmt.Println(err)
    return err
}

if v, err :=myMap[key]; err != nil {
    fmt.Println(err)
    return err
}else {
    //do something with v
}
```

注意，在早期的Go版本不支持只在`if...else`中return而不在函数结尾返回，这其实是希望大家使用更清晰的代码风格
```go
    // 不推荐
    funt f() string{
        id := 2
        if id == 1 {
            return "YourName"
        }else {
            return "MyName"  
        }
    }

    // 推荐
    funt f() string{
        id := 2
        if id == 1 {
            return "YourName"
        }
        return "MyName"
    }
```

#### `for...range...`迭代器

rang是迭代器，能根据不同的类型，返回不同的内容。

```go
for index,char := range string {}
for index,value := range array {}
for index,value := range slice {}
for key,value := range map {}
for key,value := range channel {}
```

注意遍历string时得到的是字节索引位置和UTF-8格式`rune`类型数据(int32)

```go
    s := make([]string, 0, 2)
    s = append(s, "你")
    s = append(s, "好")
    for index,value := range s {
        fmt.Println(index, value)
    }
    // 打印
    // 0 你
    // 1 好
```

index和value都可以省略，但是方式不同。省略key要用`_`代替。省略value只写key即可

注意：和其他语言不同，只取一个值表示key值而不是value值。

```go
    // 省略index
    for _,value := range s {
        fmt.Println(value)
    }
    

    // 省略value
    for key := range s {
        fmt.Println(key)
    }
    // 强烈建议，把省略value写成_
    for key, _ := range s {
        fmt.Println(key)
    }
```

range迭代器有感知迭代结束的作用。

range可用于从Channel接收值，直到它关闭为止。但是如果Channel没有被关闭就会导致迭代陷入死锁DeadLock，因为range不知何时退出。

```go
    queue := make(chan string, 2)
    queue <- "one"
    queue <- "two"
    close(queue) // 在range前关闭channel可以正常迭代
    // go close(queue) // 在range的同时并发关闭channel也可以正常迭代

    for elem := range queue {
        fmt.Println(elem)
    }
    // close(queue) // 在range后才关闭channel将导致迭代过程死锁
```

#### switch

switch的表达式可以不是常量或者字符串，也可以沒有表达式，如果没有表达式则如同if-else-else。

switch用`{}`而不是其他语言

每个case中隐藏了`break`，相当于默认避免了连续进入多个case分支

`fallthrough`表示可以继续执行后面的case。但是`fallthrough`必须是这个case中的最后一条语句。

多个case同样逻辑时，可以用`,`分隔

`default`不一定要放到最后，也可以没有

```go
// 无实际意义

// switch提供表达式，case判断表达式的值
switch ch {
    case '0', '1', '2', '3', '4', '5', '6', '7', '8', '9':
        ret = "Int"
    case 'A', 'B', 'C', 'D':
    case 'a', 'b', 'c', 'd', 'e':
        ret = "String"
    default:
        ret = "Other Char"
}

// switch无表达式，case提供表达式
switch {
    case '0' <= ch && ch <= '9':
        ret = "Int"
    case ('a' <= ch && ch <= 'z') || ('A' <= ch && ch <= 'Z'):
        ret = "ABC"
        fallthrough
    case 'a' <= ch && ch <= 'z':
        ret = "abc"
    default:
        ret = "Other Char"
}

switch time.Now().Weekday() {
    case time.Saturday, time.Sunday:
        fmt.Println("It's the weekend")
    default:
        fmt.Println("It's a weekday")
}
```

`i.(type)`是一个特殊用法，表示`获取interface{}类型的变量i的类型`，返回一个表示类型的值，注意不是字符串。这个语法只能在`switch`语句中使用，否则编译报错。

这在判断interface{}的底层类型时很有用。当然也可以用`reflect.TypeOf(t).Name()`获取反射对象的类型名称，具体不展开。

注意类型断言`x.(T)`不同于接口底层类型`i.(type)`。类型断言没有被限制在`switch`中才能使用，并且得到的是`T`类型的返回值，但是如果提供的`T`是错误的会导致编译错误。

switch 判断 interface{}的底层类型：
```go
// switch 判断 interface{}的底层类型
    whatAmI := func(i interface{}) {
        switch t := i.(type) {
        case bool:
            fmt.Println("I'm a bool")
        case int:
            fmt.Println("I'm an int")
        default:
            fmt.Printf("Don't know type %T\n", t)
        }
    }
    whatAmI(true)
    whatAmI(1)
    whatAmI("hey")
```

关于类型和反射的细节可以看`interface{} 接口类型`以及`reflect 反射`章节

### 类型

#### array 数组

数组是特定长度和类型的元素的序列。

数组中的元素类型是一致的。在内存中分配了一段同类型大小的连续的内存空间。所以数组中的元素可以是任何数据类型，包括值类型和引用类型，但是不能混用。

元素类型和长度都是数组类型的一部分。注意，长度是固定的, 不能动态变化，访问无效索引会报越界错误。

默认情况下，数组为零值，对于整型数意味着0。对于字符串意味着""。bool对应着false。

数组默认情况下是值传递， 因此会进行值拷贝。数组间不会相互影响。如想在其它函数中，去修改原来的数组，可以使用引用传递(指针方式)

```go
    var a [5]int // 注意数组类型包含长度（不带长度是切片slice），写到变量后，长度在前元素类型在后。遇到特殊类型比如二维数组就是[m][n]int
    fmt.Println("emp:", a) // 默认为零值，int类型意味着都是0
    fmt.Println("emp:", a[0])
    // fmt.Println("emp:", a[5]) // 越界
    // 输出
    // emp: [0 0 0 0 0]
    // emp: 0
    // invalid array index 5 (out of bounds for 5-element array)

    a = [5]int{1, 2, 3, 4, 5} // 初始化用{}，类型一致
    a = [5]int{1, 2, 3, 4, 5.0} // 不会报错，会自动转成int
    // a = [5]int{1, 2, 3, 4, 5.1} // 报错，constant 5.1 truncated to integer
    a = [5]int{1, 2, 3, 4, 'a'} // 不报错
    // a = [5]int{1, 2, 3, 4, "a"} // 报错
    fmt.Println("emp:", a)
    // emp: [1 2 3 4 97]

    // 二维数组
    var b [2][3]int
    for i := 0; i < 2; i++ {
        for j := 0; j < 3; j++ { // 注意i、j顺序，i是母数组的索引，j是子数组的索引
            b[i][j] = i + j
        }
    }
    fmt.Println(b)
    // [[0 1 2] [1 2 3]] // 注意打印的格式和顺序
```

数组指针，指一个数组的指针，也就是数组起始位置的地址，用`*[size]Type`表示。注意有些情况下需要`(*arrPtr)[i]`访问元素，而不能用`*arrPtr[i]`，因为优先级会导致错误。

指针数组，表示一个数组的每个元素都是指针，用`[size]*Type`表示，不太常用。

具体可以看：[Golang-数组指针和指针数组](https://learnku.com/articles/44096)

```go
func main() {
    var a [5]int
    a = [5]int{1, 2, 3, 4, 5} // 初始化用{}，类型一致
    
    change(&a)
    fmt.Println(a)
    // 输出：[100 2 3 4 5]
}

func change(x *[5]int) {
    x[0] = 100 // 正确，go支持省略指针符号
    // *x[1] = 200 // 错误，因为x[1]优先于*，所以*int是错误的。提示：invalid indirect of x[1] (type int)
    (*x)[2] = 300 // 正确，因为*x表示数组的值的列表，然后再取arr[2]。
}
```

#### Slices 切片

与数组不同的是，片仅根据其包含的元素(而不是元素的数量)进行类型划分。未初始化的切片等于nil，长度为0。

要创建一个非零长度的空切片，请使用内置的make命令。这里我们创建了一个长度为3(初始值为0)的字符串切片。默认情况下，新切片的容量等于它的长度

当切片的cap不足以容纳len时需要扩容，切片的扩容逻辑在不同Go版本不同。一般理解为元素数量在1024以下直接翻倍，以上每次增加1M。

注意，一些可能导致切片扩容的操作有可能返回一个新的切片，所以需要接受返回值。就比如append()

切片也可以被复制`copy(dst, src)`，但是要求dst必须和src的cap、len相同，否则将只能拷贝一部分，因为dst不支持自动扩容。所以直接让`dst := make([]int, len(src))`即可

slices包提供了很多使用的切片函数，比如`slices.Equal(s, p) -> bool`

切片可以组成多维数据结构。与多维数组不同，内部切片的长度可以变化。

```go
    s0 := []int{1, 2, 3} // 切片的简短声明赋值，和数组类似，但是没有长度。但是一般不这样用
    fmt.Println("emp:", s0, "len:", len(s0), "cap:", cap(s0))
    // emp: [1 2 3] len: 3 cap: 3

    var s []int
    s = make([]int, 3) // 初始化分配cap=3。再给3个元素都分配了初始值0，所以len=3
    fmt.Println("emp:", s, "len:", len(s), "cap:", cap(s))
    // emp: [0 0 0] len: 3 cap: 3

    s = append(s, 1) // 向slice追加一个元素，由于原cap不足，所以直接翻倍扩容cap=6。再将第4个元素初始化为追加的值，所以len=4。注意
    fmt.Println("emp:", s, "len:", len(s), "cap:", cap(s))
    // emp: [0 0 0 1] len: 4 cap: 6

    var s2 []int
    s2 = make([]int, 0, 3) // 初始化cap=3，但是不初始化，所以len=0
    s2 = append(s2, 1, 2) // 追加多个元素
    fmt.Println("emp:", s2, "len:", len(s2), "cap:", cap(s2)) // 追加2个元素后，len=2
    // emp: [1 2] len: 2 cap: 3

    s = append(s, s2...) // 切片相加，第二个切片需要用...表示这是一个切片，然后Go会将切片拆分成
    fmt.Println("emp:", s, "len:", len(s), "cap:", cap(s))
    // emp: [0 0 0 1 1 2] len: 6 cap: 6
    
    // 拷贝
    var c []int
    c = make([]int, len(s)) // 注意：初始化cap=len=len(src)，这样元素数和src一样，且都将被初始化为零值可以进行拷贝。固定写法，不能改。
    fmt.Println("before cpy:", c, "len:", len(c), "cap:", cap(c))
    // before cpy: [0 0 0 0 0 0] len: 6 cap: 6
    copy(c, s) // 没有返回值。
    fmt.Println("cpy1:", c, "len:", len(c), "cap:", cap(c))
    // cpy: [0 0 0 1 1 2] len: 6 cap: 6
    var c2 []int
    c2 = make([]int, 0, len(s)) // 注意只有c中len长度的元素才能被拷贝，所以如果c的cap足够，但是len太少将只会拷贝一部分。所以不能让len=0
    copy(c2, s)
    fmt.Println("cpy2:", c2, "len:", len(c2), "cap:", cap(c2))
    // cpy2: [0 0 0 1 1] len: 5 cap: 6
    var c3 []int
    c3 = make([]int, len(s)-1) // 注意如果cap和len都不够，也会只拷贝一部分，而不会对目标切片进行扩容
    copy(c3, s)
    fmt.Println("cpy3:", c3, "len:", len(c3), "cap:", cap(c3))
    // cpy3: [0 0 0 1 1] len: 5 cap: 5
    var c4 []int
    c4 = make([]int, cap(s)+1) // 拷贝到更大的dst中，这时就应该指定合适的len了，否则将出现补位的零值。所以len(s)写法是最简单且合适的
    copy(c4, s)
    fmt.Println("cpy4:", c4, "len:", len(c4), "cap:", cap(c4))
    // cpy4: [0 0 0 1 1 2 0] len: 7 cap: 7

    // 切片操作符
    low := 0
    high := len(s)
    var p []int
    p = s[low:high] // 含前不含后
    fmt.Println("part:", p, "len:", len(p), "cap:", cap(p))
    // part: [0 0 0 1 1 2] len: 6 cap: 6
    p = s[:high] // 前省略就是0
    fmt.Println("part:", p, "len:", len(p), "cap:", cap(p))
    // part: [0 0 0 1 1 2] len: 6 cap: 6
    p = s[:] // 后省略就是len(s)
    fmt.Println("part:", p, "len:", len(p), "cap:", cap(p))
    // part: [0 0 0 1 1 2] len: 6 cap: 6
    // p = s[:-1] // 报错，不支持负数。invalid slice index -1 (index must be non-negative)
    
    // slices包比较
    if slices.Equal(s, p) {
        fmt.Println("s == p")
    }

    // 二维切片(2x3)
    outerLen := 2 // row
    innerLen := 3 // column
    twoD := make([][]int, outerLen) // 给二维切片的外部切片初始化，内部到循环中再初始化
    for i := 0; i < outerLen; i++ { // 注意外部长度和内部长度
        twoD[i] = make([]int, innerLen) // 由于外部循环时，内部需要初始化一个slice，类型为[]int，才能继续赋值，否则将报错
        for j := 0; j < innerLen; j++ {
            twoD[i][j] = i + j
        }
    }
    fmt.Println(twoD) // 打印顺序和数组一样
    // [[0 1 2] [1 2 3]]
```

切片的底层实现就是数组，只是这个数组的扩容、销毁、增删改查都是Go在后台处理的。所以如果两个切片共用同一个底层数组空间，对数组的写操作是能影响另一个切片的。

所以会出现下列诡异的情况：
- 在一个函数内部对切片参数进行了排序后也会改变函数外部原来的切片中元素的顺序
- 在函数内向切片增加了元素后在函数外的原切片却没有新增元素
- 添加并排序后，外部的切片有可能元素数量和元素顺序都不会变


当函数参数为slice值时，在函数中交换元素。在函数外，切片也被影响了。

原因：虽然按值传递切片，但是切片本身只保存底层数组的地址，所以函数中局部变量s指向的地址和函数外s指向的地址仍然是一样的，共享同一个数组。所以函数内的切片操作就影响到了函数外

```go
func main() {
  var s []int
  for i := 1; i <= 3; i++ {
    s = append(s, i)
  }
  reverse(s)
  fmt.Println(s) // [3 2 1]
}

func reverse(s []int) {
  for i, j := 0, len(s) - 1; i < j; i++ {
    j = len(s) - (i + 1)
    s[i], s[j] = s[j], s[i]
  }
}
```

现在让函数内对s进行append扩容操作，发现函数外排序受到影响，并且丢失了最后1个元素。

原因：函数内局部变量s本来指向的是和外部同样的底层空间，但是append操作创建了一个新切片，新切片的len+1，在函数内翻转过程中按照len+1翻转。但是到了函数外变量s的len仍然是原len，无法获得第4个元素。

```go
func main() {
    var s []int
    for i := 1; i <= 3; i++ {
      s = append(s, i)
    }
    fmt.Println(len(s)) // 3
    fmt.Println(cap(s)) // 4 注意：第三次append导致cap从2变成4
    reverse(s)
    fmt.Println(s) // [999 3 2]
    fmt.Println(len(s)) // 3
    fmt.Println(cap(s)) // 4 注意：函数外的len不变，所以没有取到最后一个元素1
}

func reverse(s []int) {
    s = append(s, 999)
    for i, j := 0, len(s) - 1; i < j; i++ {
      j = len(s) - (i + 1)
      s[i], s[j] = s[j], s[i]
    }
    fmt.Println(len(s)) // 4
    fmt.Println(cap(s)) // 4 注意：由于cap比len大1，所以函数内没有进行扩容，也就不会产生新数组，函数内外就还是同一个底层数组空间，只是append导致函数内len+1
}
```

现在为函数内切片追加多个元素导致底层数组扩容呢？

函数内，切片扩容，len从3变成6，cap就从4翻倍到8，这时append返回的新切片s底层就是一个全新的数组了，变量s和函数外s指向的底层数组就不是同一个地址了，函数内的翻转影响的是新数组。函数外切片对应的数组不会受到任何影响。

```go
func main() {
    var s []int
    for i := 1; i <= 3; i++ {
      s = append(s, i)
    }
    fmt.Println(len(s)) // 3
    fmt.Println(cap(s)) // 4 注意：随着append进行，cap:2->4
    reverse(s)
    fmt.Println(s) // [1 2 3] 不会受到影响
    fmt.Println(len(s)) // 3
    fmt.Println(cap(s)) // 4
}

func reverse(s []int) {
    s = append(s, 999, 1000, 1001)
    fmt.Println(len(s)) // 6
    fmt.Println(cap(s)) // 8
    for i, j := 0, len(s)-1; i < j; i++ {
      j = len(s) - (i + 1)
      s[i], s[j] = s[j], s[i]
    }
}
```

解决办法就是让函数将新切片（代表新数组的地址）返回，并赋值给函数外原始切片。这样函数内外都是新切片，对应新的底层数组。

```go
func main() {
    var s []int
    for i := 1; i <= 3; i++ {
      s = append(s, i)
    }
    fmt.Println(len(s)) // 3
    fmt.Println(cap(s)) // 4 注意：随着append进行，cap:2->4
    s = reverse(s)
    fmt.Println(s) // [1001 1000 999 3 2 1] 拿到新切片
    fmt.Println(len(s)) // 6
    fmt.Println(cap(s)) // 8
}

func reverse(s []int) []int{
    s = append(s, 999, 1000, 1001)
    fmt.Println(len(s)) // 6
    fmt.Println(cap(s)) // 8
    for i, j := 0, len(s)-1; i < j; i++ {
      j = len(s) - (i + 1)
      s[i], s[j] = s[j], s[i]
    }
    return s // 返回新切片
}
```

遍历切片时，只需要注意一点。range 创建了每个元素的副本，而不是直接返回对该元素的引用。要想获取每个元素的地址，可以使用切片变量和索引值

```go
myNum := []int{10, 20, 30, 40, 50}
// 修改切片元素的值
// 使用空白标识符(下划线)来忽略原始值
for index, _ := range myNum {
    myNum[index] += 1
}
for index, value := range myNum {
    fmt.Printf("index: %d value: %d\n", index, value)
}
```

#### map 映射 字典 哈希 关联数组

创建空map，和slice的语法类似，`make(map[keyType]valueType)`

```go
func main() {
    m := make(map[string]int)
    m["k1"] = 1
    m["k2"] = 2
    fmt.Println(m)
    // map[k1:1 k2:2]
    fmt.Println(len(m))
    // 2
    
    delete(m, "k2")

    v1, ok := m["k1"] // ok语法判断是否存在
    fmt.Println(v1, ok)
    // 1 true
    v2, ok := m["k2"]
    fmt.Println(v2, ok)
    // 0 false

    clear(m) // 清除整个map。注意：老版本不支持。新版本为了解决`for{delete()}`无法删除NaN问题才决定引入clear()
    fmt.Println(m)
    // map[]

    // maps包提供map函数
    n := map[string]int{"foo": 1, "bar": 2}
    fmt.Println("map:", n)
    n2 := map[string]int{"foo": 1, "bar": 2}
    if maps.Equal(n, n2) {
        fmt.Println("n == n2")
    }
}
```

注意map中key是无序的，所以遍历时无法按序输出。一定要按序，需要用一个数据结构，比如slice，向map写入时也要按序写入key值到slice，再按序遍历slice挨个去map中查找对应的value。

另外由于map遍历的无序性，在遍历map期间，如果有一个新的key被创建，那么在循环遍历过程中可能会被输出，也可能会被跳过。对于每一个创建的key在迭代过程中是选择输出还是跳过都是不同。所以请不要在遍历map时还新增key。

```go
func main() {
    m := map[int]bool {
        0: true,
        1: false,
        2: true,
    }
    
    for k, v := range m {
        if v {
            m[10+k] = true
        }
    }
    fmt.Println(m)
    // 输出举例：
    // map[0:true 1:false 2:true 10:true 12:true 20:true 22:true 30:true 32:true 40:true]
    // map[0:true 1:false 2:true 10:true 12:true 22:true 32:true 42:true 52:true 62:true]
}
```

map和slice一样，在容量不足时会进行自动扩容以及rehash操作。所以map如果能确定初始容量，会对性能有很大帮助。`make(map[keyType]valueType, initialCap)`

map是非并发安全的，sync包的sync.Map类型才是并发安全的，但是函数有很大不同。或者构造一个带有map和sync.RWMutex成员的struct。

```go
package main

import "fmt"
import "sync"

func main() {
    var m sync.Map
    m.Store("k1", "v1") // 写

    v1, err := m.Load("k1") // 读
    fmt.Println(v1, err)
    // v1 true

    m.Store("k2", "v2")
    m.Range(func(k, v interface{}) bool{ // Range遍历，注意匿名函数的固定格式、类型、返回值等都是固定的
        fmt.Println(k.(string), v.(string))
        return true
    })
    // k1 v1
    // k2 v2

    m.Delete("k1")
    v1, err = m.Load("k1") // 读
    fmt.Println(v1, err)
    // <nil> false
}
```

实现原理不在这里描述（dirty）

#### interface{} 接口类型

[类型断言](https://go.dev/ref/spec#Type_assertions)

interface{}的类型断言 i.(T)：
```go
// interface{}的类型断言：i.(T)
var x interface{} = 7 // x has dynamic type int and value 7
fmt.Println(x.(int)) // 正确
fmt.Println(x.(string)) // 报错：interface {} is int, not string
fmt.Println(x.(type)) // 报错：use of .(type) outside type switc
fmt.Println(reflect.TypeOf(x).Name()) // 正确。获取反射对象的类型名称。不展开
y := 3
fmt.Println(y.(int)) // 错误：只能对interface{}类型进行断言
```

switch中通过`i.(type)`判断interface{}的底层类型：
```go
// switch 判断 interface{}的底层类型
    whatAmI := func(i interface{}) {
        switch t := i.(type) { // i.(type)只能在switch中
        case bool:
            fmt.Println("I'm a bool")
        case int:
            fmt.Println("I'm an int")
        default:
            fmt.Printf("Don't know type %T\n", t)
        }
    }
    whatAmI(true)
    whatAmI(1)
    whatAmI("hey")
```

比较两个interface{}的方法
```go
func Compare(a, b interface{}) (int, error) {
    aT := reflect.TypeOf(a)
    bT := reflect.TypeOf(b)
    if aT != bT {
        return -2, errors.New("进行比较的数据类型不一致")
    }

    switch av := a.(type) {

    default:
        return -2, errors.New("该类型数据比较没有实现")
    case string:
        switch bv := b.(type) {
        case string:
            return ByteCompare(av, bv), nil
        }
    case int:
        switch bv := b.(type) {
        case int:
            return NumCompare(av, bv), nil
        }

    }

    return -2, nil
}
```

反射的细节要看`relect 反射`一节

##### relect 反射

Golang 类型断言(Type Assertion)与反射(Reflect)都是用于动态处理类型的方法，但是它们有一些很重要的区别。

[golang 类型断言与反射区别](https://juejin.cn/s/golang%20%E7%B1%BB%E5%9E%8B%E6%96%AD%E8%A8%80%E4%B8%8E%E5%8F%8D%E5%B0%84%E5%8C%BA%E5%88%AB)

类型断言：
- 可以用于检测空接口变量的具体类型，并将其转换为某个具体类型。
- 它是通过检测一个接口变量的具体类型来进行的，因此它不需要知道运行时类型的名称。
- **类型断言是在编译时进行的**，因此它对于运行时的性能影响比较小。

反射：
- 可以用于动态获取和修改类型信息和变量的值。
- 它通过分析某个类型的信息来实现，因此它**需要知道运行时类型的名称**。
- **反射是在运行时实现的**，因此它对于运行时的性能影响比较大。

总的来说，如果你只需要获取或修改一个变量的值，那么使用类型断言是更好的选择；如果你需要获取或修改一个类型的所有信息，那么使用反射是更好的选择。

```go
var x interface{} = 7
fmt.Println(x.(int))
fmt.Println(reflect.TypeOf(x).Name())
```

更多：TODO

#### struct 结构体

结构体是一组数据类型的集合

```go
package main

import "fmt"
import "sync"

type Person struct { // 大写可被外部访问
    sync.RWMutex // 并发读写锁
    Name string // 大写可被外部访问
    Age int
}

func NewPerson() *Person{ // 注意，一般都是返回结构体指针，New函数也是开头大写
    return &Person{}
}

func (p *Person) SetAttr(name string, age int) {
    p.Lock()
    defer p.Unlock()
    p.Name = name
    p.Age = age
}

func (p *Person) GetAttr() (string, int){
    p.RLock()
    defer p.RUnlock()
    return p.Name, p.Age
}

func main() {
    p := NewPerson()
    p.SetAttr("andy", 10)
    name, age := p.GetAttr()
    fmt.Println(name, age)
    // andy 10

    p.Age = 11
    fmt.Println(p)
    // &{{{0 0} 0 0 0 0} andy 11}
}

```

## 工程陷阱

### 导入包但未使用

导入但未使用会在编译器报错。所以如果导入包内部初始化，但是并未在当前文件中使用，可以用`_`导入但不引用。mysql驱动实现database/sql就是这样导入的。



## 主要参考

[Go by Example](https://gobyexample.com/)