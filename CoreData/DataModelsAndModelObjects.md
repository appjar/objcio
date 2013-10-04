Data Model 和 Model Object<sup>1</sup>
====

我们会在这文章中仔细探究一下 Core Data Model 和 Managed Object 类，本文不会是一个关于这个话题的简单介绍，而会专注于在使用 Core Data 过程中一些少有人知道或经常被忽略的方面。如果你想了解更多的细节，请移步 [苹果的 Core Data 编程指南](https://developer.apple.com/library/mac/documentation/cocoa/Conceptual/CoreData/cdProgrammingGuide.html#//apple_ref/doc/uid/TP30001200-SW1)

Data Model（数据模型）
---

存在 *.xcdatamodel 文件中的数据模型定义了应用中所会用到的数据类型（也就是 Core Data 中的 Entity——实体）。通常我们用 Xcode 的图形界面来定义数据模型，但是你也可以在代码中完成这些事情：你可以首先创建一个[NSManagedObjectContext](https://developer.apple.com/library/ios/documentation/cocoa/Reference/CoreDataFramework/Classes/NSManagedObjectModel_Class/Reference/Reference.html)对象，然后通过[NSEntityDescription](https://developer.apple.com/library/mac/documentation/Cocoa/Reference/CoreDataFramework/Classes/NSEntityDescription_Class/NSEntityDescription.html)对象来创建实体，实体中可能会包含关系和属性，分别可以用[NSRelationshipDescription](https://developer.apple.com/library/ios/documentation/cocoa/Reference/CoreDataFramework/Classes/NSRelationshipDescription_Class/NSRelationshipDescription.html)和[NSAttributeDescription](https://developer.apple.com/library/mac/documentation/Cocoa/Reference/CoreDataFramework/Classes/NSAttributeDescription_Class/reference.html)来表示。虽然你几乎用不到这种方法，但是了解一下这些类还是有好处的。

###Attribute（属性）###

创建了实体时候，还要定义实体上的属性。定义属性很简单，但是有一些方面值得我们注意。

####Default（默认值）/Optional（可选）####

一个属性可以被定义为是可选的还是必选的。如果一个必选的属性被设为空值，那么在保存时会失败。同时，我们也可以为每个属性设置一个默认值。可选的属性也可以有默认值，但是这没有太大的意义，而且可能会带来一些困扰，所以我还是建议你对于有默认值的属性，还是把它的 optional 选项给勾掉吧。

####Transient（临时的）####

这是一个经常被忽略的选项，如果一个属性被定义为 Transient，那么它就不会被写入持久化存储中，但是它运行时，它和普通的属性一样：也就是说它也会参与到[验证][Validation]、恢复、错误处理等过程之中。当你想把较多的模型逻辑代码放到 Managed Object 子类中时，transient 属性时很有用的，具体的使用技巧可以看[这里][TransientInsteadOfIvars]。

[TransientInsteadOfIvars:]
####Managed Object 子类中的变量####

[Validation:]
####Validation（验证）####

本话题的更多文章
---
* [Core Data概述](CoreDataOverview.md)
* [一个简单却完整的 Core Data 应用](SimpleButCompleteCoreDataApplicati\
on.md)
* [使用 SQLite 和 FMDB 替代 Core Data](UsesSQLiteInsteadOfCoreData.md)
* [导入大量数据]()
* [查询](PerformantFetching.md)
* [自定义 Core Data 的数据迁移](CustomCoreDataMigrations.md)

备注
---
1. 原文链接：http://www.objc.io/issue-4/core-data-models-and-model-objects.html
