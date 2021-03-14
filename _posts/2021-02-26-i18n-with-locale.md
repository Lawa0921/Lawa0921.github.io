---
title: 關於 I18n 的 with_locale
layout: post
tag:
- Ruby
- Rails
- I18n
- Object Model
author: lawa0921
hidden: false
category: blog
date: '2021-02-27 10:09:20'
---

大家應該或多或少都有在工作當中接觸到 I18n 的經驗  
如果你看到了這一段程式碼  
你覺得他會輸出什麼語系？  

```
I18n.default_locale = "en"

I18n.with_locale("zh-TW") { object = SomeClass.new }

object.some_column
```

答案是 en 的語系  
有些人會認為不是已經在 initialize 的時候指定語系給 object 了嗎 ？  
但是其實並不是指定  
有關 I18n 的狀態  
其實並不在 object 身上  
而是在 I18n 這個物件身上  

讓我們來看看 with_locale 這個 method 做了什麼：  
```
I18n.method(:with_locale).source.display

    def with_locale(tmp_locale = nil)
      if tmp_locale == nil
        yield
      else
        current_locale = self.locale
        self.locale = tmp_locale
        begin
          yield
        ensure
          self.locale = current_locale
        end
      end
    end
```

如果對 self 的對象不理解的話推薦可以看看這篇由我的強者朋友 Fred 所寫的 [Ruby 的繼承鍊 (1) - 如何實踐物件導向](https://www.spreered.com/ruby-object-model-1/)  
簡單的說就是 I18n 在處理時  
先暫時把你所指定的語系存在自己身上  
利用這個暫存的語系來處理 block 裡面的事情  
處理完之後再把語系設定回原本自己身上的語系  
  
再說的白話點就是  
  
當生出一個 object 時我們有請 I18n 跟 zh-TW 來生  
但是其實 object 被生完之後並不知道他實際上屬於哪個語系的  
所以他就會再去問 I18n，是哪一個語系把它生出來的  
但是這時候 I18n 這個傢伙生活比較亂，他有很多個對象分別是 en zh-TW zh-HK ... etc，他已經忘記生出 object 拿到的語系是什麼了  
於是就隨便塞了一個他現在的對象給 object 跟他說這個就是你爸爸  
於是才造成了這個誤會  
  
因為問題的根源在於 object 本身並不知道要拿出來的語系是什麼  
因此類似於 `I18n.with_locale("zh-TW") { object = SomeClass.new }` 這樣的寫法其實是沒有用處的  

### 提示
如果下次有遇到類似的問題  
可能可以先想想  
目前你需要的這則訊息  
是誰負責知道的？  
是 I18n 嗎？  
還是 object 呢？  
搞清楚當前的 self 是誰可以幫助你快速釐清問題的根源在哪邊
