一个完整的 Core Data 应用 <sup>[1]</sup>
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

###创建 Table View Controller###

到了这里，我们可以使用我们刚才创建的类来创建一个 View Controller 展示 Item 列表了。在栗子应用中，我们用到了一个 Storyboard，在 Storyboard 中添加了一个以 Table View Controller 为根的 Navigation Controller，这样会自动把这个 Table View Controller 设为数据源，但是我们不想这么做。所以，在 viewDidLoad 中，我们要添加下面这些代码：

``` objective-c
fetchedResultsControllerDataSource =
    [[FetchedResultsControllerDataSource alloc] initWithTableView:self.tableView];
self.fetchedResultsControllerDataSource.fetchedResultsController = 
    self.parent.childrenFetchedResultsController;
fetchedResultsControllerDataSource.delegate = self;
fetchedResultsControllerDataSource.reuseIdentifier = @"Cell";
```

在 Fetched Results Controller Data Source 的初始化函数中会设置 Table View 的 Data Source，Reuse Identifer 也需要与 Storyboard 中的对应上。我们还需要实现代理方法：

``` objective-c
- (void)configureCell:(id)theCell withObject:(id)object
{
    UITableViewCell* cell = theCell;
    Item* item = object;
    cell.textLabel.text = item.title;
}
```

当然，除了设置 TextLabel，你还可以对许多其它的东西进行设置，你随便了。现在，我们已经为显示数据做好了一切准备，但是还是空空如也，什么都没有显示出来。

添加交互
----
我们会添加几种数据的交互方法：首先，会加入一个能添加 Item 的方法；然后，会实现 Fetched Results Controller 的代理来刷新 Table View；最后添加对删除和恢复数据的支持。

###添加 Item###

为了添加Item，我从 [Clear](http://www.realmacsoftware.com/clear/) 剽窃了一种交互方式，顺便说一句，Clear 是我认为的最美的应用之一。我们把一个 Text Field 设置为 Table View 的 header，然后修改 Table View 的 Content Inset 来使 Text Field 默认为隐藏状态，要看设置的方法可以戳[这里](../Views/InternalOfScrollView.md)。跟往常一样，完整的代码放在 GitHub 上，下面这几行是在 textFieldShouldReturn 中比较关键的：

``` objective-c
[Item insertItemWithTitle:title 
                   parent:self.parent
   inManagedObjectContext:self.parent.managedObjectContext];
textField.text = @"";
[textField resignFirstResponder];
```

###监听数据变化###

下一步是确保在有新的 Item 插入时，你的 Table View 也会插入一行。有几种方法，可以达成这一目的，我们选择了实现 Fetched Results Controller 的代理的方法：

``` objective-c
- (void)controller:(NSFetchedResultsController*)controller
   didChangeObject:(id)anObject
       atIndexPath:(NSIndexPath*)indexPath
     forChangeType:(NSFetchedResultsChangeType)type
      newIndexPath:(NSIndexPath*)newIndexPath
{
    if (type == NSFetchedResultsChangeInsert) {
        [self.tableView insertRowsAtIndexPaths:@[newIndexPath]
                              withRowAnimation:UITableViewRowAnimationAutomatic];
    }
}
```

当有删除、更改或者移动的时候，Fetched Results Controller 调用这个方法。如果你同时有多个更改的话，你需要再实现另外两个方法来让 Table View 把这些改变一次性展示出来。对于一次简单的插入或删除一个 Item 的操作，可能表现得不明显，但是当你实现数据同步时，这样做会变得很漂亮。

``` objective-c
- (void)controllerWillChangeContent:(NSFetchedResultsController*)controller
{
    [self.tableView beginUpdates];
}

- (void)controllerDidChangeContent:(NSFetchedResultsController*)controller
{
    [self.tableView endUpdates];
}
```

###使用 Collection View###

Fetched Results Controller 不仅仅可以与 Table View 合作，它可以用到任何种类的 View 中。因为它是基于 Index Path 的，所以它也可以与 Collection View 一起工作得相当完美，尽管你可能要悲催的修改一些代码，因为 Collection View 没有 beginUpdates 和 endUpdates，只有一个 performBatchUpdates。所以你要把你获得的所有的更改都收集起来，然后再 controllerDidChangeContent 中一次性把改变都展示出来，想看栗子可以戳[这里](https://github.com/AshFurrow/UICollectionView-NSFetchedResultsController)。

###实现你自己的 Fetched Results Controller###

你可以不用 NSFetchedResultsController，实际上，你可以为你的应用定制一些东西，能变得更有效率。你可以监听 NSManagedObjectContextOjbectsDidChangeNotification，当你接收到这个事件时，userInfo 字典会包含更改的、插入的、删除的对象，那么你可以随意处理它们了。

传递模型对象
---

有了添加和显示 Item 的功能之后，可以着手开始开发添加子列表功能了。你可以在 Storyboard 里把一个 cell 拖到 View Controller 中来创建一个 Segue，最好给这个 Segue 命名，如果我们在同一个 View Controller 中有许多 Segue 的话，我们可以找到要找的 Segue。

我处理 Segue 的方法如下：首先，要找到要找的 Segue；然后对每一个 Segue，为它们的目标 View Controller，创建一个单独的方法。

``` objective-c
- (void)prepareForSegue:(UIStoryboardSegue *)segue sender:(id)sender
{
	[super prepareForSegue:segue sender:sender];
 	if ([segue.identifier isEqualToString:selectItemSegue]) {
		[self presentSubItemViewController:segue.destinationViewController]
	}
}

- (void)presentSubItemViewController:(ItemViewController *)subItemViewController
{
	Item *item = [self.fetchedResultsControllerDataSource selectedItem];
	subItemViewController.parent = item;
}
```

子 View Controller 需要的唯一的东西就是 Item，它可以从这个 Item 中获得到 Managed Object Context。我们从 Data Source 中得到被选中的 Item，然后把它传递给子 View Controller 就行了，就是这么简单。

一个非常常见的坏习惯就是把 Managed Object Context 作为应用代理的一个属性，然后从任意的地方都可以访问到它。如果你想在你的代码中的某一个特定的部分使用另外一个 Managed Object Context 的话，按照上面的方法做回是代码变得非常难以重构，单元测试做起来也会很困难。

现在，试着把一个 Item 添加到子列表中，你的应用可能会华丽的跪掉，因为我们现在有两个 Fetched View Controller —— 一个用于最顶部的 View Controller，另一个用于根 View Controller —— 第二个会试图在未被显示时刷新屏幕，这样会导致崩溃。解决方法是告诉 Data Source 停止对 Fetched Results Controller 的监听：

``` objective-c
- (void)viewWillAppear:(BOOL)animated
{
    [super viewWillAppear:animated];
    self.fetchedResultsControllerDataSource.paused = NO;
}

- (void)viewWillDisappear:(BOOL)animated
{
    [super viewWillDisappear:animated];
    self.fetchedResultsControllerDataSource.paused = YES;
}
```

实现以上目的的一个方式是在 Data Source 中把 Fetched Results Controller 的代理设为 nil，这样就不会再收到任何更新的通知了：

``` objective-c
- (void)setPaused:(BOOL)paused
{
    _paused = paused;
    if (paused) {
        self.fetchedResultsController.delegate = nil;
    } else {
        self.fetchedResultsController.delegate = self;
        [self.fetchedResultsController performFetch:NULL];
        [self.tableView reloadData];
    }
}
```

PerformFetch 会确保你的 Data Source 是最新的，当然，还有更好的实现方法，就是不把代理设为空，而是在暂停状态下记录下所有的变化，然后在离开暂停状态的时候刷新 Table View。

删除
---

我们需要通过几个步骤来添加删除功能：首先，找到要提供删除功能的 Table View；然后，把对象从 Core Data 中删除，并确保剩下的 Item 的 Order 属性仍然配置正确。

为了添加滑动删除的支持，需要实现 Data Source 的下面两个方法：

``` objective-c
- (BOOL)tableView:(UITableView*)tableView canEditRowAtIndexPath:(NSIndexPath*)indexPath
{
    return YES;
}
 
- (void)tableView:(UITableView *)tableView commitEditingStyle:(UITableViewCellEditingStyle)editingStyle forRowAtIndexPath:(NSIndexPath *)indexPath 
{
    if (editingStyle == UITableViewCellEditingStyleDelete) {
        id object = [self.fetchedResultsController objectAtIndexPath:indexPath];
        [self.delegate deleteObject:object];
    }
}
```

我们需要告诉代理（那个 View Controller）来删除对象，而不是直接把对象删除了，这样做就可以不必与 Data Source 共享持久化存储层，也可以使自定义动作更容易维护；相应的 View Controller 仅仅需要调用 Managed Object Context 的 deleteObject: 即可。

还有另外两个重要的问题没有解决：如果处理我们删除的 Item 的 Children；在删除之后，如何处理其他 Item 的 Order。对于前者，Core Data 提供了良好的删除支持：在我们的数据模型中，我们可以把 Children 关系的删除规则设为 Cascade。对于后者，我们可以重写 prepareForDeletion 方法，在方法中把所有 Order 比被删除的 Item 高的邻居 Item 的 Order 都更新：

``` objective-c
- (void)prepareForDeletion
{
    NSSet *siblings = self.parent.children;
    NSPredicate *predicate = [NSPredicate predicateWithFormat:@"order > %@", self.order];
    NSSet *siblingsAfterSelf = [siblings filteredSetUsingPredicate:predicate];
    [siblingsAfterSelf enumerateObjectsUsingBlock:^(Item* sibling, BOOL *stop)
    {
        sibling.order = @(sibling.order.integerValue - 1);
    }];
}
```

现在我们几乎已经完成了对删除的支持，可以触控 Table View 的 Cell 来删除对象了，最后一步就是实现一些必须的代码，来确保当有 Item 被删除时，它对应的 Cell 也要被删除。在我们的 Data Source 的 controller:didChangeObject: 方法中，需要添加一个 if 分支：

``` objective-c
...
else if (type == NSFetchedResultsChangeDelete) {
    [self.tableView deleteRowsAtIndexPaths:@[indexPath]
                          withRowAnimation:UITableViewRowAnimationAutomatic];
}
```

添加 Undo 支持
---

Core Data 另一个非常漂亮的功能就是它整合了 Undo 支持。我们需要添加 Shake to Undo （晃动以恢复，iPhone 的一个功能）功能，第一步就是告诉应用我们能做这个：

``` objective-c
application.applicationSupportsShakeToEdit = YES;
```

现在，当用户晃动了手机，应用就会去找第一个响应者要它的 Undo Manager，然后来执行 Undo 操作。在[上个月的这篇文章](../Views/CreatingCustomControls.md)中，我们看到一个 View Controller 也是在响应链中的，这就是我们下面要用到的特性。我们要在 View Controller 中重写 UIResponder 的下面两个方法：

``` objective-c
- (BOOL)canBecomeFirstResponder {
    return YES;
}

- (NSUndoManager*)undoManager
{
    return self.managedObjectContext.undoManager;
}
```

现在，当有晃动手势事件发生时，Managed Object Context 的 Undo Manager 会获得一个 Undo 消息，并 Undo 最近的一次变化。记住，在 iOS 上，一个 Managed Object Context 默认是没有 Undo Manager 的（但是在 Mac 上是有的），所以我们在配置 Core Data 栈结构时，需要手动添加这个 Undo Manager：

``` objective-c
self.managedObjectContext.undoManager = [[NSUndoManager alloc] init];
```

现在基本上已经完成了 Undo 功能了，当你晃动手机时，iOS 会弹出一个对话框，对话框有两个按钮：Undo 和 Cancel。Core Data 会自动把数据的变化分组，举个栗子，addItem:parent 会被分为一个 Undo 动作，而删除会被分为另外的 Undo 动作。

为了能使用户更好的使用 Undo 功能，我们可以给动作命名，只要稍稍修改一下 textFieldShouldReturn: 就可以了：

``` objective-c
NSString* title = textField.text;
NSString* actionName = [NSString stringWithFormat:
    NSLocalizedString(@"add item \"%@\"", @"Undo action name of add item"), title];
[self.undoManager setActionName:actionName];
[self.store addItem:title parent:nil];
```

这样的话，当用户晃动手机时，弹框上的文字就不仅仅是一个 Undo 了。

编辑
---
在栗子代码中是不支持编辑的，但是在实际使用中，仅仅修改 Item 的属性也是常见场景。比如，只修改 Item 的标题，这个栗子看起来影响不大；但是把一个 Item foo 的 Parent 变成 bar，这会导致许多变化。

排序
---
对 Cell 进行重新排序也没有支持，但是很容易实现。不过这有一个需要注意的地方：当你允许用户手动对 Item 进行排序，你必须更新 Item 的 Order 属性，这会使 Fetched Results Controller 调用它的相关代理方法。这一过程在 [NSFetchedResultsControllerDelegate documentation](https://developer.apple.com/library/ios/documentation/CoreData/Reference/NSFetchedResultsControllerDelegate_Protocol/Reference/Reference.html#//apple_ref/doc/uid/TP40008228-CH1-SW14) 中有解释。

保存
---
保存很简单，只需要调用 Managed Object Context 的 save 方法即可，唯一困难的地方就是保存时机，苹果的示例代码在 applicationWillTerminate: 中保存，但是你也可以根据情况把它写在 applicationDidEnterBackground: 或者在运行时保存。

本话题的更多文章
---
* [Core Data概述](CoreDataOverview.md)
* [使用 SQLite 和 FMDB 替代 Core Data](UsesSQLiteInsteadOfCoreData.md)
* [Data Model 和 Model Objects](DataModelsAndModelObjects.md)
* [导入大量数据]()
* [查询](PerformantFetching.md)
* [自定义 Core Data 的数据迁移](CustomCoreDataMigrations.md)

备注
---
[1] 原文链接：http://www.objc.io/issue-4/full-core-data-application.html  
[2] 由于我没有实践过 Storyboard 方面的编程，文中关于 Storyboard 的相关段落翻译可能会不尽准确，请见谅。有兴趣的可以移步原文。
