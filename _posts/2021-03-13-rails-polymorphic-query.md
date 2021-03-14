---
title: 關於 Rails 多型關聯的 query 方法 (query polymorphic associations)
layout: post
tag:
- Ruby
- Rails
- SQL
- ActiveRecord
- ORM
author: lawa0921
category: blog
date: '2021-03-13 15:08:11'
---

# 關於多型關聯的 joins

當我們在 Rails 需要進行多個 table 的查詢時  
我們會使用 joins 的語法  
```
Class WatchingInfo
  belongs_to :user_course
end

Class UserCourse
  has_one :watching_info
end

>>  WathcingInfo.joins(:user_course).where(user_courses: { is_active: true })
  >> ActiveRecord::Relation [......]
```

如此一來就可以將 table 拼在一起  
並找出所有 WatchingInfo 對應的 user_course 是 is_active 的資料了  

但是如果今天這個是多型關聯  
這件事情就沒辦法這樣子做了  

```
Class WatchingInfo
  belongs_to :watchable, polymorphic: true
end

Class UserCourse
  has_one :watching_info, as: :watchable
end

Class UserWebinar
    has_one :watching_info, as: :watchable
end

>> WathcingInfo.joins(:watchable)
  >> ActiveRecord::EagerLoadPolymorphicError (Cannot eagerly load the polymorphic association :watchable)
```

看起來 Rails 本身並沒有支援 joins 多型關聯  
不過這並不代表這件事情做不到  

要搞清楚這件事情  
首先我們要知道 joins 這個 method 幫我們做了什麼  

其實 ActiveRecord 這個 ORM 在串起一段 method 時  
是先把所有的查詢串接起來  
等到真正執行這段程式碼時才進行 query   
並且不管你的 where 怎麼串  
只要內容是一樣的  
組出的 query 所查詢的結果都會是一樣的  

```
>> User.where(id: 1).where(username: "abc").where(email: "abc@gmail.com").to_sql
  >> "SELECT `users`.* FROM `users` WHERE `users`.`id` = 1 AND `users`.`username` = 'abc' AND `users`.`email` = 'abc@gmail.com'"
>> User.where(email: "abc@gmail.com").where(id: 1).where(username: "abc").to_sql
  >> "SELECT `users`.* FROM `users` WHERE `users`.`email` = 'abc@gmail.com' AND `users`.`id` = 1 AND `users`.`username` = 'abc'"

# 這兩段 query 所回傳的結果會是一樣的
```

首先知道了 where 的順序沒有影響之後  
接著來看看 joins 做了什麼  

```
Class WatchingInfo
  belongs_to :user_course
end

Class UserCourse
  has_one :watching_info
end

>>  WathcingInfo.joins(:user_course).to_sql
  >> "SELECT `watching_infos`.* FROM `watching_infos` INNER JOIN `user_courses` ON `user_courses`.`id` = `watching_infos`.`user_course_id`
```

在 joins 的時候  
其實會組出一個 SQL 查詢  
會把 watching_infos 這張表與 user_courses 拼在一起  
條件是 user_courses 的 id 與 watching_infos 的 course_id 相同時  
也就是說第一筆 watching_info 會把其所對應的 user_course 拉出來與這筆 watching_infos 拼在一起  

既然 rails 沒辦法自動幫我們拼多型關聯  
我們可以直接寫 raw SQL 來解除這個限制   

```
>> WatchingInfo.joins("INNER JOIN user_courses ON user_courses.id = watching_infos.watchable_id")
  >> ActiveRecord::Relation [ ...... ] 
```

如此一來就可以正確 joins 出想要的表並針對 joins 進來的 table 進行查詢  

# joins 多個多型關聯的表
 
如果你只需要針對多型裡面的其中一種進行查詢的話到這邊就已經足夠了  
但是今天你可能需要同時對兩個甚至三個多型關聯的表進行篩選  
這時候該怎麼做呢？  

這是今天我在工作上實際遇到的使用情境  
```
Class WatchingInfo
  belongs_to :user
  belongs_to :watchable, polymorphic: true
end

Class UserWebinarSet
  belongs_to :webinar_set
  has_one :watching_info, as: :watchable
end

Class UserCourse
  belongs_to :course
  has_one :watching_info, as: :watchable
end

Class Course
  has_many :user_courses
end

Class WebinarSet
  has_many :user_webinar_sets
end
```

今天我所需要的是查出所有對應的 course 與 webinar_set 的 is_active 是 true 的 watching_info  
或許你會以為可以這樣做  
```
WatchingInfo.
  joins("INNER JOIN user_courses ON user_courses.id = watching_infos.watchable_id INNER JOIN courses ON courses.id = user_courses.course_id").
  joins("INNER JOIN user_webinar_sets ON user_webinar_sets.id = watching_infos.watchable_id INNER JOIN webinar_sets ON webinar_sets.id = user_webinar_sets.webinar_set_id").
  where("courses.is_active = TRUE OR webinar_sets.is_active = TRUE")

>> ActiveRecord::Relation [ ...... ] 
```

看起來是可以正常運作  
但是其實這並不是你所需要的結果  
原因是 joins 預設是使用 INNER JOIN  
在大部分的情況也確實使用 INNER JOIN 即可   
可以幫助我們過濾掉一些我們所不需要資料  
但是在這邊的情形  
因為我們要同時同時需要兩邊的表來進行查詢  
但是如果這樣子做的話  
就變成在 joins 的過程就已經把不少資料過濾掉了  
這樣子 joins 最後會留下的 wathcing_info 只有 watching_info 對應的 user_course 及 user_webinar_set 都存在的才會留下來查詢  
也就是說 watchable_id 要對應到兩個表都有一樣的資料才會留下來  
但這並不是我們想要的結果  

# LEFT OUTER JOIN

當我們不想在 joins 階段過濾掉資料時  
可以使用 `LEFT OUTER JOIN` 來拼表  

```
joins("LEFT OUTER JOIN user_courses
      ON user_courses.id = watching_infos.watchable_id' LEFT OUTER JOIN courses ON courses.id = user_courses.course_id").
    joins("LEFT OUTER JOIN user_webinar_sets
      ON user_webinar_sets.id = watching_infos.watchable_id LEFT OUTER JOIN webinar_sets ON webinar_sets.id = user_webinar_sets.webinar_set_id").
    where("courses.is_active = TRUE OR webinar_sets.is_active = TRUE")
		
>> ActiveRecord::Relation [ ...... ] 
```

這樣一來就可以拼出需要的表並進行查詢了  
在拼表時不會過濾掉任何資料  
缺少資料的部分會自動讓該欄位空缺  
也就不會發生 `INNER JOIN` 時發生的問題了  

# raw SQL 好難看喔

不得不說 raw SQL 的寫法雖然可以運作  
但是可讀性實在很難說得上是高  
為了不要造成後續維護夥伴的困擾  
後來決定還是採用類似原先 joins 的概念來寫這段 query   
不過不能 joins 原來的多型關聯那該怎麼辦呢？  

### 答案是在寫一個關聯給它

```
class WatchingInfo
  belongs_to :watchable, polymorphic: true
  belongs_to :user_course, -> { where(watching_infos: { watchable_type: 'UserCourse'}).includes(:watching_info) }, foreign_key: "watchable_id", optional: true
  belongs_to :user_webinar_set, -> { where(watching_infos: { watchable_type: 'UserWebinarSet'}).includes(:watching_info) }, foreign_key: 'watchable_id', optional: true
end
```

如此一來你在 joins 的時候的對象就不是多型關聯了  
所以你就變得可以這樣操作  

```
WatchingInfo.joins(user_course: :course)
```

這邊要補充一個小地方  
在 model 裡面寫的關聯可以接上一個 lambda   
在這個 lambda 裡面的 self 並不是這個 model 本身  
而是 belongs_to 的對象  

```
belongs_to :user_course, -> { where(watching_infos: { watchable_type: 'UserCourse'}).includes(:watching_info) }, foreign_key: "watchable_id", optional: true

# 以上這段的意思其實等同於下面這段

belongs_to :user_course, -> { UserCourse.where(watching_infos: { watchable_type: 'UserCourse'}).includes(:watching_info) }, foreign_key: "watchable_id", optional: true
```

因此我們才會需要 includes 當前的 model 來做查詢  

而要做到類似於上面的查詢則可以使用 ActiveRecord 所準備的 `left_outer_joins` 與 *`or` method 來做到  

```
WatchingInfo.
  left_outer_joins(user_course: :course, user_webinar_set: :webinar_set).merge(WebinarSet.active).
  or(WatchingInfo.left_outer_joins(user_course: :course, user_webinar_set: :webinar_set).merge(Course.active))
```

感謝所有協助我解決這個問題的同事大大們   
特此紀錄  

* `or` method 為 rails 5 後才有的 method ，若是你使用的版本低於 5 則無法使用該 method
