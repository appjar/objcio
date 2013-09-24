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

创建模型
---
创建模型很简单，我们只需要在工程中新建一个文件，选择 Core Data 模板（在 Core Data 分类下面）。这个文件会被编译为一个后缀名为 momd 的文件，持久化存储需要一个 NSManagedObjectModel 的对象，当我们在运行时需要创建这个对象时，需要载入这个 momd 文件。模型的源文件是一个简单的 XML 文件，根据经验，当你把它放到版本控制中时，你一般不会遇到需要合并的问题。如果你愿意的话，你也可以在代码中创建一个模型。

当你创建了模型之后，你就可以添加一个有两个属性的 Item 了；接着，你要给它添加两个关系：parent 和 children；你需要把这两个关系设置为互逆的，也就是说如果你把 a 的 parent 设置成了 b，那么 b 就自动把 a 设置成了 它的 children 之一了。

通常情况下，你还可以使用 order 关系，或把 order 属性置空，但是当你使用 NSFetchedResultsController 查询是，这个 order 关系就表现得不是特别理想了。我们要做的是：要么重新实现一部分 NSFetchedResultsController 的；要么重新实现排序，我选择了后者。

现在在菜单中选择“编辑” > “创建 NSManagedObject 子类”，创建了一个绑定到这个实体的子类，这样你会得到两个新建的文件：Item.h 和 Item.m，在头文件中还有一个由于历史原因保留下来的额外的 category，我们要立即把这个东西删掉。

创建一个 Store 类
---
我们要创建一个根节点作为 item 树的起点，为了创建它并在稍后找出它，我们需要创建一个简单的 Store 类来帮助完成这一任务。它有一个 managed object context 和一个找到 rootItem 的方法。在我们应用的 app delegate 中，我们要找到这个 rootItem 并把它传递给根 view controller。你也可以把这个 rootItem 存储到 user defaults 中，以便能够更快的找到它：

``` objective-c
- (Item *)rootItem 
{
    NSFetchRequest *request = [NSFetchRequest fetchRequestWithEntityName:@"Item"];
    request.predicate = [NSPredicate predicateWithFormat:@"parent = %@", nil];
    NSArray *objects = [self.managedObjectContext executeFetchRequest:request error:NULL];
    Item *rootItem = [objects lastObject];
    if (rootItem == nil) {
        rootItem = [Item insertItemWithTitle:nil
                                      parent:nil
                      inManagedObjectContext:self.managedObjectContext];
    }
    return rootItem;
}
```

添加一个新的 item 是很简单的，有一点需要注意的是，当我们设置一个新的 item 的 order 属性时，我们要把这个属性的值设置得比它的兄弟 item 的 order 属性值都大：这样一来，第一个孩子的 order 为 0，之后的孩子的 order 属性的值都要比它前面的大1。我们会在 Item 类中自定义一个方法，并把上数设置 order 的逻辑写到这个自定义的方法中：

``` objective-c
+ (instancetype)insertItemWithTitle:(NSString *)title
                             parent:(Item *)parent
             inManagedObjectContext:(NSManagedObjectContext *)managedObjectContext
{
    NSUInteger order = parent.numberOfChildren;
    Item *item = [NSEntityDescription insertNewObjectForEntityName:self.entityName
                                            inManagedObjectContext:managedObjectContext];
    item.title = title;
    item.parent = parent;
    item.order = @(order);
    return item;
}

- (NSUInteger)numberOfChildren
{
    return self.children.count;
}
```

为了自动刷新表的内容，我们还会用到一个 fetched results controller，这个东西会管理一个 fecth request 和许多 item，能很好的配合 table view，我们会在下一部分介绍到它：

```objective-c
- (NSFetchedResultsController *)childrenFetchedResultsController
{
    NSFetchRequest *request = [NSFetchRequest fetchRequestWithEntityName:[self.class entityName]];
    request.predicate = [NSPredicate predicateWithFormat:@"parent = %@", self];
    request.sortDescriptor = @[[NSSortDescriptor sortDescriptorWithKey:@"order" ascend:YES]];
    return [[NSFetchResultsController alloc] initWithFetchRequest:request
                                             managedObjectContext:self.mangedObjectContext
                                               sectionNameKeyPath:nil
                                                        cacheName:nil];
}
```

添加一个可以接收 Fetched Results Controller 反馈的 Table View
---
下一步我们要创建一个根 View Controller：一个 Table View，它可以从一个 NSFetchedResultsController 中获取数据。这个 Fetched Results Controller 会管理你的 Fetch Request，如果你还给它设置了一个代理，那么当 Managed Object Context 中有数据变动时，它会及时的通知你，这意味着如果你实现了它的代理方法，那么当你感兴趣的数据发生变化时，你可以自动更你的 Table View。举个栗子，如果你在后台有一个同步线程，而且把数据变化保存到了数据库中，那么你的 Table View 就会自动更新了。

###创建 Table View 的 Data Source###
在[清爽的 View Controller](../LighterViewControllers/LighterViewController.md)中，我们演示了如何把 Data Source 从 Table View 中剥离出来，我们要对 Fetched Results Controller 做相同的事情；我们创建了一个单独的类 FetchedResultsControllerDataSource 来扮演 Table View 的 Data Source，同时它也负责监听 Fetched Results Controller 并更新 Table View。我们用 Table View 来初始化这个 FetchedResultsControllerDataSource，像这样：

``` objective-c
- (id)initWithTableView:(UITableView *)tableView
{
    self = [super init];
    if (self) {
        self.tableView = tableView;
        self.tableView.dataSource = self;
    }
    return self;
}
```

在我们设置 Fetch Results Controller 的时候，我们也必须给它设置代理，同时执行第一次查询操作。此时很容易忘记调用 `perFormFetch:`，如果那样的话，你不会获得到任何结果，当然也不会产生任何错误。

``` objective-c
- (void)setFetchedResutlsController:(NSFetchedResultsController *)fetchedResultsController
{
    _fetchedResultsController = fetchedResultsController;
    fetchedResultsController.delegate = self;
    [fetchedResultsController performFetch:NULL];
}
```

既然我们要实现 UITableViewDataSource 协议，那么下面两个方法是一定要实现的：

``` objective-c
- (NSInteger)numberOfSectionsInTableView:(UITableView *)tableView
{
    return self.fetchedResultsController.sections.count;
}

- (NSInteger)tableView:(UITableView *)tableView numberOfRowsInSection:(NSInteger)sectionIndex
{
    id<NSFetchedResultsSectionInfo> section = self.fetchedResultsController.sections[sectionIndex];
    return section.numberOfObjects;
}
```

在创建 cell 时，需要两步：

* 通过 Fetched Results Controller 找到对应的对象
* 找到一个可复用的 cell，并通过 Table View 的代理来设置这个 cell

现在，我们已经有了一个很好的职责分离的设计，View Controller 只需要负责使用传入的对象来更新这个 cell：

``` objective-c
- (UITableViewCell*)tableView:(UITableView*)tableView 
        cellForRowAtIndexPath:(NSIndexPath*)indexPath
{
    id object = [self.fetchedResultsController objectAtIndexPath:indexPath];
    id cell = [tableView dequeueReusableCellWithIdentifier:self.reuseIdentifier
                                             forIndexPath:indexPath];
    [self.delegate configureCell:cell withObject:object];
    return cell;
}
```