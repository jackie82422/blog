---
title: Vim綁不到想要用的Rider Action?可以多繞點路
date: 2021-06-06 23:59:16
categories: IDE
tags: [IDE,Rider,Vim]
---

我相信用到雙平台的人最大的痛點就是IDE的Shortcut都長得不一樣了，想透過VIM去綁定Action還發現Action Name好像不太對啊?!本篇文教你怎麼找到對的Action Name.

<!-- more -->

一般我們再綁定Vim action的時候可以透過在Vim command輸入
```bash
:actionlist
```
![](A.png)
去取得所有的Action Name
![](B.png)

數量是多得離譜，可以COPY到Editor比較方便

然後就可以開心在.ideavimrc裡面做編輯，從而去綁定Action了。(通常.ideavimrc會放在 ~/ 下面，這點在雙平台都是一樣的，沒有的話就自己建立吧！)

編輯的方式大致上如下
![](C.png)

第9行是設定leader鍵，在16行你可以看到如果我要執行GotoAction的時候要下的按鍵是 "space"? ，而我設定的leader鍵是"space"，也就是我打"space"? 的時候會執行GotoAction的動作。

而同樣16行後面你可以看到"CR"，這個是Enter的意思，因為指令打完之後Vim會跳出提示問你是不是要執行GotoAction，操作上非常不順暢，通常會直接補上Enter去確定(跳過)它。

至此你已經知道如何去綁定Action了。

但如果你實際試過綁定Action這件事情的話，你會發現有一點非常麻煩；通常你會在ide的Setting中找到你要的快捷鍵，並將這個快捷鍵的Action綁定到Vim上面，那時你會發現怎麼找不到對應的Action Name呀？！

沒錯，就是長得不大一樣！

我們舉個例子吧，例如我可以斷言是所有使用Jetbrains IDEs產品的使用者最常用的Action “Show context actions”在Windows系統上它的快捷鍵預設為alt + Enter；在Mac os系統上它的快捷鍵預設為option + Enter

![](D.png)

在Actionlist上他叫做甚麼呢？ “ShowIntentionActions”

雖然從語意上感覺能大概猜一下，但是當你在actionlist搜尋Action的時候有近五百個結果，搜尋Show有近兩百個結果，看了就累人。

我提供一個我自己的作法，如果有更好的方式麻煩分享一下…

我的作法是這樣，如果我要綁定”Show Context Action”這個快捷鍵的Action，我會先替它在加一組快捷鍵，像是這樣

![](E.png)

然後到特定的folder找設定

Windows : C:\users\\""userName""\AppData\Roaming\JetBrains\Rider ""version""\keymaps\ ..
Mac OS : ~\Library\Application Support\Jetbrains\Rider ""version""\keymaps\ ..

下面應該可以找到一個xml檔案，開啟

![](F.png)

當啷，你找到ActionName了，恭喜恭喜，雖然這個方式有點迂迴，但還好只需要設定一次。