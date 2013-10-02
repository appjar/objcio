一个完整的 Core Data 应用 <sup>[1]</sup>
====

__我找不到一个好理由在说服你不去使用 Core Data，它实在是太优秀了，目前被大量的 Cocoa 开发者使用，这样当你的小组有新成员加入时，或者有别人要接替你的工作时，开发维护工作会变得非常顺利。__

__更重要的是，没有必要花时间或精力去开发一个你自己的数据系统，总之享受 Core Data 吧。__

为什么我不用 Core Data
===

[Mike Ash 说](http://www.mikeash.com/pyblog/friday-qa-2013-08-30-model-serialization-with-property-lists.html)：

> 从个人角度上讲，我不是特别喜欢 Core Data，它的 API 有些笨重，类库本身呢又显得比较慢，尤其是对一些数据量较大的应用来说。

一个现实的栗子：10000 条数据
===

比如说我现在有一个 RSS 阅读器，用户可以右键单击某一个订阅源，然后把这个订阅源下所有的条目标记为已读。

这种情况下，会有一个有 read 属性的 Article 实体，为了把每一个条目标记为已读，应用需要把这个订阅源下所有的文章都载入到内存中，然后把 read 属性设置为 YES。

通常情况下，上面的处理的方法还说得过去，但是假设这个订阅源下面有200篇文章，你就要考虑把这项工作放在后台进行以免阻塞了主线程了（尤其是在 iPhone 应用中），这样你就必须考虑多线程的 Core Data，事情变得麻烦了起来。

看起来可能还不至于因为这点儿事儿就弃用了 Core Data，那么在考虑一下下面的情况呢？

RSS 订阅源是需要同步的，我现在有两种不同的 RSS 同步 API，他们都返回一个已读文章的 ID 数组，其中一个可能会返回多达 10000 个 ID。你不想把这 10000 篇文章都载入到内存然后在主线程里把它们的 read 属性设为 NO；你也不想在后台线程中这样做：因为无论任何一种做法都会耗费太大代价（比如电池负荷）。

实际上一个比较好的解决办法是：告诉数据库把数组中的每一篇文章的 read 属性设为 NO。SQLite 就可以做到这一点，而且只需要一步操作。如果你又在表的 uniqueID 字段建立了索引，那么这个操作时很快的，你也可以像在主线程一个轻松的完成这一任务。

另外一个栗子：快速启动
===



本话题的更多文章
---
* [Core Data概述](CoreDataOverview.md)
* [一个简单却完整的 Core Data 应用](SimpleButCompleteCoreDataApplication.md)
* [Data Model 和 Model Objects](DataModelsAndModelObjects.md)
* [导入大量数据]()
* [查询](PerformantFetching.md)
* [自定义 Core Data 的数据迁移](CustomCoreDataMigrations.md)

备注
---
[1] 原文链接：http://www.objc.io/issue-4/SQLite-instead-of-core-data.html 
[2] 由于我没有实践过 FMDB 方面的编程，文中关于 FMDB 的相关段落翻译可能会不尽准确，请见谅。有兴趣的可以移步原文。
