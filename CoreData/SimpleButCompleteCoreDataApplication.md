一个完整的 Core Data 应用
====   

在本文中，我们将会创建一个小而完整的 Core Data 应用，这个应用允许你建立一组层次化的列表，列表的每一个表项可以有它的子列表，这个列表的层次化结构可以是非常非常深的。为了完全的理解 Core Data 的运行机制，我们会手工建立起这个 Core Data 的堆栈结构，而不是使用 Xcode 提供的模板。示例代码可以到 [GitHub](https://github.com/objcio/issue-4-full-core-data-application) 上查看。

我们会怎么建立它
---

首先，我们会使用一个 Core Data 模型和一个文件名来创建一个 PersistentStack 对象，它会返回一个上下文；然后，我们要建立我们自己的 Core Data 模型；接下来，我们会新建一个简单的 UITableViewController来展示列表项的根节点，这会用到 NSFetchedResultsController，列表同时会一步步的提供添加、导航、删除、恢复等功能的支持。

建立栈结构
---
我们首先在主线程队列创建一个上下文，在以前的代码中，你可能会看到```[[NSManagedObjectContext alloc] init]```，但是现在，如果你使用了基于队列的并行模型，你应该使用```initWithConcurrencyType:```这个初始化方法：

``` objective-c
- (void)setupManagedObjectContext
{
    self.managedObjectContext = 
        [[NSManagedObjectContext alloc] initWithConcurrencyType:NSMainQueueConcurrencyType];
    self.managedObjectContext.persistentStoreCoordinator = 
        [[NSPersistentStoreCoordinator alloc] initWithManagedObjectModel:self.managedObjectModel];
    NSError *error;
    [self.managedObjectContext.persistentStoreCoordinator
        addPersistentStoreWithType:NSSQLiteStoreType
                     configuration:nil
                               URL:self.storeURL
                           options:nil
                             error:&error];
    if (error) {
        NSLog(@"error: %@", error);
    }
    self.managedObjectContext.undoManager = [[NSUndoManager alloc] init];
}
```

检查错误很重要，因为在开发中很可能产生失败。比如当你修改了你的数据模型，Core Data 检测到了这一点便不再会继续下去了，你也可以给 options 传一个值来高速 Core Data 这种情况下应该怎么做，Martin 在他的[数据迁移](CustomCoreDataMigrations.md)这篇文章中详细描述了这点。在最后一行代码中添加的一个 undo manager 会在以后被使用到，不同于在 Mac，在 iOS 上你需要显示的添加一个 undo manager，不然它不会被默认加上。

上段代码创建了一个简单的 Core Data 栈结构：一个上下文，一个上下文关联的 PSC 和一个 PSC 关联的持久化存储。设置[可以更复杂](CoreDataOverview.md)，通常可能有许多上下文，每个上下文在一个单独的队列中。