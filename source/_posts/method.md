---
title: golang 雜記 - method
toc: true
date: 2022-10-25 10:21:07
categories: golang
tags: golang
---


golang 雜記。

<!-- more -->


## Method

golang的method沒有overload可以用，宣告的方式跟函式很像。

```go
type Person struct {
	FirstName string
	LastName  string
	Age       int
}

func (p *Person) AddAge() {
	p.Age++
}

func (p Person) String() string {
	return fmt.Sprintf("%s %s, age is %d", p.FirstName, p.LastName, p.Age)
}
```

我們宣告了一個叫做`String()`的方法，該方法會回傳一個`string`常值，接收子則是`p`。函式的宣告則接受子在名稱與輸出型別之間，可以參考 [function](https://blog.chalin.ninja/golang-function/)。

有趣的部份是我們在寫function或method時常常都會有修改傳入值的時候，在golang上就是以傳遞指標變數然後取得指標值然後修改該位置的值。在function裡面可以參考上一篇關於function的雜記，在method中基本一樣，但在method裡面有一點語法糖可以使用。

```go
func main() {
	person := lib.Person{
		FirstName: "Chacha",
		LastName:  "Lin",
		Age:       29,
	}
	person.AddAge()
	fmt.Println(person.String())
}
// output: Chacha Lin, age is 30
```

這個呼叫是承接上一個範例的呼叫，應該很容易發現`person.AddAge()`這行明顯看起來是呼叫值變數，但卻在下一行的複本中看到原值被修改了，這是因為golang在讀`person.AddAge()`這行時做的是`(&person).AddAge()`。

只要receiver寫的是指標型態，則呼叫時會自動去接收這個值的指標位址。

至於用pointer的時機，我節錄一下`Go Document`的建議：

>First, and most important, does the method need to modify the receiver? If it does, the receiver must be a pointer. (Slices and maps act as references, so their story is a little more subtle, but for instance to change the length of a slice in a method the receiver must still be a pointer.).

>Second is the consideration of efficiency. If the receiver is large, a big struct for instance, it will be much cheaper to use a pointer receiver.

>Next is consistency. If some of the methods of the type must have pointer receivers, the rest should too, so the method set is consistent regardless of how the type is used.

簡言之是如果你需要改變傳入的值，用pointer；如果你的值非常大，用pointer，因為pointer大小永遠是固定的，你傳的是這個值的指標位置；最後是一致性，如果這個類型的某些方法是用指標接收，那你應該使其一致。


> TODO 一致性

另外，`nil`是可以被傳遞到method中的，但前提是`nil`必須為指標接收。

```go
func (p *Person) IsNil() bool {
	if p == nil {
		return true
	}
	return false
}
```

原因也很簡單，以值為接收子的話實際是沒有指向任何地方的，但以指標為值的話只是收到一個指向`nil`的指標值而已。

method同時也也跟function一樣可以指派到一個變數上面，這個行為稱作`method value`。

```go
func main() {
	var person lib.Person
	person.FirstName = "Chacha"
	person.LastName = "Lin"
	result := person.String
	fmt.Println(result())
    //output: Chacha Lin, age is 0
}
```

也可以用型態本身建立函式，這個行為稱為`method expression`。

```go
func main() {
	var person lib.Person
	person.FirstName = "Chacha"
	person.LastName = "Lin"
	result := lib.Person.String
	fmt.Println(result(person))
    //output: Chacha Lin, age is 0
}
```

## IOTA

iota並不是縮寫，而是希臘字母的其中一個字，當它為數學符號的時候它代表著幾個意思：
1. iterator of sum
2. 下標索引
3. 複數的虛

在golang中是從APL這個編成語言借鑒過來的用法，它代表生成連續整數序列。

在其他語言中很類似的東西是`enum`。

```go
	const (
		Stop = iota
		Start
		Restart
	)

	fmt.Println(Stop)
    //output 0
    fmt.Println(Start)
    //output 1
```

當const block中宣告了iota，該值會從0開始，後面的值不需要定義iota則會照順序給予1,2,3...etc.


## Inheritance

繼承主要目的有兩個，一是程式碼重用，二是polymorphism(多型、多態)，golang是沒有所謂繼承的，golang真的是很新的語言，它跟現在多數工程師給出的建議一樣：多用組合(Composition)取代繼承(Inheritance)，甚至是從語言層面支持這種作法。

這個可以體現在他的內嵌語法：

```go
type Person struct {
	FirstName string
	LastName  string
	Age       int
}

func (p *Person) AddAge() int {
	p.Age++
	return p.Age
}

func (p *Person) String() string {
	return fmt.Sprintf("%s %s, age is %d", p.FirstName, p.LastName, p.Age)
}

type Student struct {
	Person
	Score int
    Age   int
}

func (s *Student) GetScore() int {
	return s.Score
}
```

在`Student`的struct裡面有一個`Person`，這個Person沒有指定名稱，因此他是一個內嵌欄位(embedded)，在這邊它會被提昇到`Student`這個Struct並且展開，所以在使用上可以像這樣：

```go
func main() {
	s := lib.Student{
		Score: 100,
		Person: lib.Person{
			FirstName: "Chacha",
			LastName:  "Lin",
			Age:       29,
		},
	}
	result := s.String
	fmt.Println(result())
    fmt.Println(s.LastName)
	fmt.Println(s.Person.Age)
    //output: Chacha Lin, age is 29
    //output: Lin
    //output: 29
}
```

這看起來是繼承的一種實現方式，但它本質上跟其他語言的繼承還是有所差別的。在這邊的`Student`看似繼承了`Person`，但它並不是`Person`的子類，無法將之賦予給`Person`。


```go
s := lib.Student{}
var p Person
p = s
// 編譯失敗
```

## Interface

golang中的interface宣告上跟其他語言差不了多少：

```go
type Personer interface {
	String(p *Person) string
}
```

習慣上golang的interface會用`-er`作為結尾。

但是golang跟其他語言的interface有一個很大的差異，在一般動態語言(ex. javascript)一樣是有interface，但他們採用的是鴨子定型(Duck typing)，這個概念很有趣；它指的是當有個生物走起來像鴨子、游泳像鴨子、叫聲也像鴨子，那這東西就是鴨子沒錯了。

白話是說當這個東西有相同的property跟method，那他們就是一樣的，可以參考這篇[Duck Typing in Javascript - stackoverflow](https://stackoverflow.com/questions/3379529/duck-typing-in-javascript)，最高投票數的回覆很簡潔易懂。

在靜態型別(ex. C#)中這個interface是很嚴格的，他不能只是看起來像鴨子，它必須就得是鴨子。

```C#
	public static void Main()
	{
		IDuck duck = new Duck();
		duck.Quack();
	}

	interface IDuck{
		void Quack();
	}
	
	internal class Duck : IDuck{
		public void Quack(){
			Console.WriteLine("quack-quack!");
		}
	}
```

所以我們做的不是檢查property或者method，而是一開始就約束了class到底是屬於啥東西。

而golang的interface顯然是比較偏向duck typing的。儘管Learning Go一書中說明這並非duck typing而是一個混合體，原因在於golang的interface是一種隱性實做，他不用在implement interface的struct上面明確宣告它是實做了哪個interface，但實際在使用上這種沒有明確約束卻默默幫你連結好的行為跟duck typing實際的作法其實差異不大。


```go
type Personer interface {
	String() string
	AddAge() int
}

type Person struct {
	FirstName string
	LastName  string
	Age       int
}

func (p *Person) AddAge() int {
	p.Age++
	return p.Age
}

func (p *Person) String() string {
	return fmt.Sprintf("%s %s, age is %d", p.FirstName, p.LastName, p.Age)
}

func main() {
	var a lib.Person = lib.Person{
		LastName: "Lin",
	}

	fmt.Println(a.String())
	//output:  Lin, age is 0
}
```

如果你的IDE有支持Go to implement的功能，你會發現`Personer`這個interface已經有了`Person`這個implement，與javascript不同，golang定義的interface是type safe的，這就是最大最大的差別，縱使它看起來很像。

interface也跟struct一樣可以內嵌

```go
type Studenter interface {
	Personer
}
```

前面提到golang interface的隱性實做(implimented implicitly)可以帶出一個神奇的狀態，空介面(empty interface)，從剛剛的例子可以知道只要實做了interface method的struct就會自動被認作該interface的實做，而空的介面正正是所有型態的實做。

所以空的interface可以代表任何的型態。

```go
	var a interface{}
	a = 20
	a = "Hello"
	a = struct{ id int }{id: 20}
	fmt.Println(a)
	//output 20
```

在1.18版本後的golang新增了泛形，也提供了關鍵字`any`這個`any`等效於`interface{}`，所以可以這樣寫

```go
func main() {
	var a any
	a = 20
	a = "Hello"
	a = struct{ id int }{id: 20}
	fmt.Println(a)
	//output 20
}
```

當不確定資料格式時可以拿來用，但並不能當作常態，身為強型態語言這不是一個很好的作法。

既然`any`可以代表所有的型態，在使用上我們還是常常會需要判斷這個型態到底是啥，這時可以使用type assertion型態推斷去判斷具體型態到底是啥，推斷失敗則throw panic.

```go
	var i interface{} = "Test"
	c := i.(string)
	fmt.Println(c)
	//output Test
	d := i.(int)
	//panic: interface conversion: interface {} is string, not int
 
```

golang對型別判斷非常嚴格，如果即使操作的型別底層型別一致但外層不一致一樣會噴panic.

```go
type CustomInt int

func main() {
	var i any
	var c CustomInt = 20
	i = c
	d := i.(int)
	//panic: interface conversion: interface {} is main.CustomInt, not int
}
```

或者使用ok寫法：

```go
type CustomInt int

func main() {
	var i any
	var c CustomInt = 20
	i = c
	d, ok := i.(int)
	if !ok {
		fmt.Println("Error")
	}
	fmt.Println(d)

	//output Error
	//output 0
}
```
失敗會得到false,同時將值預設為default value(0).

或者使用switch-case

```go
switch j := i.(type){
	case int:
	case string
	...
}
```