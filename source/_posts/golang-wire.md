---
title: Dependency Injection -- Wire
toc: true
date: 2022-10-31 16:09:47
categories: golang
tags: golang
---

自從dotnet後面出了net core framework之後DI已經內建在了框架之中，體驗過了DI帶來的低耦合跟可測試性之後換個語言還是會想把DI也一起弄了；這邊是看Google - Wire的雜記。

不會有過多介紹DI本身，主要還是這個DI框架的應用。

<!-- more -->


## 在框架之前

DI是完成了S.O.L.I.D裡面的D - Dependency Inversion(控制反轉)的實現，在框架之前你完全可以透過自己寫code去完成這一點，它並不是很高深的魔法。在net core中我們幾乎是綁定了DI在我們的code裡面，這是因為在net core中的DI除了完成了控制反轉，它還帶給我們很多的方便。

譬如它高度集成框架自帶的一些工具，net core的一些工具已經透過DI集成在框架裡面方便你隨時抽換，而且官方的DI還提供了物件的Life cycle控制，你可以控制物件在啥時被實體化，複用還是每次都重新實體化。

在GOLANG寫一個簡單的DI實現其實很容易。

```go
func main() {
	log := LogToConsole{ModuleName: "TestProject"}
	repository := ProductRepository{}
	paymentService := NewPaymentService(log, repository)
	paymentService.Log.LogInfo("Test")
}

type PaymentService struct {
	Log        Logger
	Repository Repository
}

func NewPaymentService(log *LogToConsole, repository *ProductRepository) *PaymentService {
	fmt.Println(&log.ModuleName)
	return &PaymentService{
		Log:        log,
		Repository: repository,
	}
}

type Logger interface {
	LogInfo(s string)
	LogError(s string)
	LogWarning(s string)
}

type LogToConsole struct {
	ModuleName string
}

func (l *LogToConsole) LogInfo(s string) {
	fmt.Printf("%v , %v\n", l.ModuleName, s)
}

func (l *LogToConsole) LogError(s string) {
	//TODO implement me
	panic("implement me")
}

func (l *LogToConsole) LogWarning(s string) {
	//TODO implement me
	panic("implement me")
}
```

如此一來我的上層邏輯不用依賴於底層的具體實做，只要符合interface定義的我就通通可以拿來用；後續我要把LOG存到DB存到FILE還是存到哪裡，都只需要完成該interface的實做，並且一開始告知我要啥就可以了。

