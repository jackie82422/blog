---
title: golang 變數型態雜記
toc: true
categories: golang
date: 2022-10-18 09:33:59
tags: golang
---

雜記

<!-- more -->

## 數字型態

1. 如果使用二進制檔格式、網路協定...etc 那就使用對應的整數格式例如byte -> uint8
2. 如果在寫lib，可能需要寫上多個不同整數格式
3. 其他：大部分情況用默認`int`
4. 計算金額時因為golang跟大多數語言一樣使用IEEE754規格存放浮點數，計算上精準不足，建議使用`decimal` lib.

## 字串

1. default value is string.empty.
2. is immutable.
3. for compiler, rune equals int32.


## 型別轉換

golang不提供自動型態轉換（automatic type promotion），例如c-sharp可以透過內建或自訂的implict,explict operator做自動的型別轉換，但golang不行。
好處是不用去記憶那些型別轉換規則，也不用去擔心精度喪失；缺點是這些東西都得自己handle.


## 變數宣告

``` golang
// 最完整
var x int = 10

// 編譯後確定型別
var x = 10

// 宣告變數並賦予default value
var x int

// 宣告多個變數
var x,y int = 1,2
var x,y int

// 宣告串
var (
    x int
    y      = 2
    z int  = 3
    d, e   = 4, "Test"
)

// 短宣告
x := 10
x, y := 10,"Test"
```

短宣告在package-level無法使用。

在function內如果宣告default值建議用：`var x int`而非短宣告方式（option, coding style)

## 未定型態變數

未定型態變數在無法推斷確切型態的時候會給一個預設型態，通常未定型態變數比較彈性
```go
const x = 10
var y int = x
var z float32 = x
var a byte = x
```
以上通通合法。

## Array

1. 宣告
```go
// 包含3個integer的array，值都為default , 0
var x [3]int

// 宣告同時賦值
var x [3]int{2,3,4}

// 稀疏陣列
var x [11]int{1,5:4,11:100}
// [1,0,0,0,4,0,0,0,0,0,100]

// 不定義大小 
var x [...]int{2,3,4}
```
2. golang不支援N維陣列，只能用模擬的
```golang
var x [5][5]int
```
代表有五個`[5]int`。

3. 不同大小的陣列被視為不同的型態，例如`[5]int` != `[3]int`，而型態在編譯期就會確定，所以無法將大小當作變數傳入陣列中。且無法使用型態轉換將大小不同的陣列轉換成一樣的大小：所以除非知道陣列的真正大小不然平常陣列用到的機會不多。

## Slice

#### 宣告，slice不用宣告大小，大小並非型態的一部分。
```go
var x = []int{1,2,3}

var x [10]int{1,5:4,11:100}
// [1,0,0,0,4,0,0,0,0,0,100]

var x []int
// x's value is nil

fmt.printf(x == nil)
// output : true

var y []int{}
// y's value is []
fmt.printf(y == nil)
//output : false
```
#### append function
```go
var x []int
x = append(x, 1)
x = append(x, 2,3,4,5,6)

y := []int{7,8,9}
x = append(x, y...)
// x = [1,2,3,4,5,6,7,8,9]
```

### capacity 

slice有容量限制，當加入的值大過容量限制則go runtime會自動擴展這個slice，它會將slice的內的值複製到新的slice，將新的值加到結尾然後回傳新的slice，這樣的操作是肯定會消耗一點效率的。
所以在知道明確大小的情況下（或者可以預估最大大小的情況下）可以用`make`函數指定容量

```go
x := make([]int, 5)
fmt.Println(cap(x))
// output : 5
```

但這個操作建立的是長度為5且容量為5的slice，預設的值為0，當新增值的時候會將長度擴展為6，容量翻倍（容量1024下預設翻倍，後續每次增加25%）

```go
x.append(x,6)
// [0,0,0,0,0,6]
```

如果要實現這個效果可以這樣用：

```go
x := make([]int, 0 ,5)
x.append(x,6)
// [6]
```

`x := make([]int, 0 ,5)` 可以產生長度為0，容量為5的slice.

> append  ！一！定！ 會增加長度

### 切割

```go
	x := []int{1, 2, 3, 4}
	y := x[:2]
	z := x[1:]
	d := x[1:3]
	e := x[:]
	fmt.Println("x:", x)
	fmt.Println("y:", y)
	fmt.Println("z:", z)
	fmt.Println("d:", d)
	fmt.Println("e:", e)

// output:
// x: [1 2 3 4]
// y: [1 2]
// z: [2 3 4]
// d: [2 3]
// e: [1 2 3 4]
```

slice與切割出來的sub-slice共用記憶體，換言之對sub-slice值做改變一樣會影響到main-slice，而切割出來的slice的容量與main-slice是一樣的。

因為共用記憶體，同理在對sub-slice做append的時候也會影響到main-slice.

比較特殊的作法是在切slice的時候同時把容量定成切出來的大小，再做append的時候因為超出容量所以會回傳一個容量更大的copy slice，就不會影響到main-slice.

```go
    x := []int{1,2,3,4}
    y := x[:2:2]
    y = append(y, 30)
    fmt.Println("x:", x)
	fmt.Println("y:", y)

    // output :
    // x: [1 2 3 4]
    // y: [1 2 30]
```

但更好的作法我覺得用copy會更漂亮

```go
	x := []int{1, 2, 3, 4}
	y := make([]int, 4)
	copy(y, x)
	fmt.Println(y)

    // [1,2,3,4]
```

也可以部份複製

```go
	x := []int{1, 2, 3, 4}
	y := make([]int, 2)
	copy(y, x[:2])
	fmt.Println(y)

    // [1,2] 
```


## String, rune, byte.

從string的源碼註釋可以看到

> string is the set of all strings of 8-bit bytes, conventionally but not
necessarily representing UTF-8-encoded text. A string may be empty, but
not nil. Values of string type are immutable.

其實作法跟c-sharp差不多，由byte陣列組成string

golang
```go
    var s string = "Test"
    var b byte = s[1]
    // b = 'e'
```

c-sharp
```c#
    var s = "string";
    char b = s[1];
```

一樣可以跟slice做切片運算。

`var s2 string = s[:2]`

而且字串是不可變的，它沒有slice那種修改sub影響main的問題，但它也有其他的問題；例如有些符號或emoji佔了多個byte，當然中文也是，因此字串切可能切不完整。

也因為這個原因，golang提供了`rune`去做字串的處理。

```go
	var s string = "哈囉，golang"
	var b []byte = []byte(s)
	var r []rune = []rune(s)
	fmt.Println(b)
	fmt.Println(r)
	fmt.Println(s)

    // Output
    // [229 147 136 229 155 137 239 188 140 103 111 108 97 110 103]
    // [21704 22217 65292 103 111 108 97 110 103]
    // 哈囉，golang

    var c string = string(r[0])
    fmt.Println(c)
    // Output
    // 哈
```

比較[]byte, []rune就可以發現差異。

## MAP

```go
var nilMap map[string]int
fmt.Println(nilMap == nil)
// output : true

nonNilMap := map[string]int{
    "Test":1,
}

var mapWithMakeFunc := make(Map[string]int, 10)
// 長度為0, 容量為10
```

map跟slice有多個相同的點
1. map同slice會在容量不足時自動擴展
2. 可以用make function建造自定義容量的map
3. default value為nil(指的是map本身)
4. map call by ref, 無法直接用`'=='`做比較

增刪改查map

```go
var shopItem map[string]int
shopItem = make(map[string]int,10)
shopItem["Gin"] = 300
fmt.Println(shopItem["Gin"])

// output 300
```

當map搜尋不到指定的key時會回傳0，在不確定0是找不到值還是值就是0的狀況下，可以使用`ok`寫法。

```go
v, ok := shopItem["Whisky"]
fmt.Println(v,ok)
//output 0 false
```

刪除特定值可以使用delete function，如果key不存在則不發生任何事情。

```go
delete(shopItem, "Whisky")
```

Java跟C#語言有的`Set`結構則golang並不提供，但可以用map模擬`Set`結構，key值（index value)可以放欲放入set的值，而value則放boolean值，給定true。

```go
mockSet := make(map[int]bool)
mockSet[5] = true
mockSet[1] = true

v,ok := mockSet[2]
fmt.Println(v,ok)
// output false false

v,ok := mockSet[5]
fmt.Println(v,ok)
// outout true true

fmt.println(mockSet[2])
// output false
```

也用放空struct的方式，可以進一步降低記憶體使用量，但再取值就非得使用`ok`寫法了。差別只有給定boolean會消耗 1 byte... 

