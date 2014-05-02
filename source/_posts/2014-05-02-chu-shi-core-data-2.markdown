---
layout: post
title: "初识Core Data(2)"
date: 2014-05-02 20:17:04 +0800
comments: true
categories: 
- Objective-C
- Core Data

---
##自定义NSManagedObject
在上一篇教程中我们每条数据都是通过`NSManagedObject`对象装载，通过KVC方式使用`valueForKey：`方法访问对象属性，但是使用KVC要比使用访问器效率低一点。 只在必要时使用KVC，比如你需要动态选择key或keyPath。  

``` objc
[newEmployee setValue:@”Stig” forKey:firstName];
[aDepartment setValue:@1000 forKeyPath:manager.salary];
``` 

下面我们将自定义`NSManagedObject`类，通过对它的继承拓展，使得我们有自己的Event类，并通过访问器方法代替KVC方式来访问对象的属性。  
按CMD+N或者在可视化建模工具下选择菜单中Editor->Create NSManagedObject Subclass：  

![](/images/blog/QQ20140502-1@2x.png)  

![](/images/blog/QQ20140502-2@2x.png)  

选中需要子类化的Entity（当然我们只有一个Event，自动勾选了）：  

![](/images/blog/QQ20140502-3@2x.png)  

最后点击Create，于是Event类就创建好了，可以看到属性timeStamp已经自动生成了，并且实现为`@dynamic`  

熟悉Objective-C语法的都知道`@synthesize`实际的意义就是自动生成属性的setter和getter方法。  

`@dynamic`就是要告诉编译器，代码中用`@dynamic`修饰的属性，其getter和setter方法会在程序运行的时候或者用其他方式动态绑定，以便让编译器通过编译。其主要的作用就是用在`NSManagerObject`对象的属性声明上，由于此类对象的属性一般是从Core Data的属性中生成的，Core Data框架会在程序运行的时候为此类属性生成getter和setter方法。  

好的，下面我们改写以前的代码，这次我们将使用Event类的对象完成以前的任务：  

在MasterViewController.m文件中加入`#import "Event.h"`，然后将`insertNewObject:`方法替换如下  

``` 
- (void)insertNewObject:(id)sender
{
    NSManagedObjectContext *context = [self.fetchedResultsController managedObjectContext];
    NSEntityDescription *entity = [[self.fetchedResultsController fetchRequest] entity];
    Event *newManagedObject = [NSEntityDescription insertNewObjectForEntityForName:[entity name] inManagedObjectContext:context];
    
    // If appropriate, configure the new managed object.
    // Normally you should use accessor methods, but using KVC here avoids the need to add a custom class to the template.
//    [newManagedObject setValue:[NSDate date] forKey:@"timeStamp"];
    newManagedObject.timeStamp = [NSDate date];
    // Save the context.
    NSError *error = nil;
    if (![context save:&error]) {
         // Replace this implementation with code to handle the error appropriately.
         // abort() causes the application to generate a crash log and terminate. You should not use this function in a shipping application, although it may be useful during development. 
        NSLog(@"Unresolved error %@, %@", error, [error userInfo]);
        abort();
    }
}
``` 
嗯，英文注释还告诉我们通常你应该用访问器方法呢，还说但是现在在这用KVC就避免了向模板添加自定义类的需求，真逗啊  
依此类推，更改`prepareForSegue: sender:`方法：    

``` 
- (void)prepareForSegue:(UIStoryboardSegue *)segue sender:(id)sender
{
    if ([[segue identifier] isEqualToString:@"showDetail"]) {
        NSIndexPath *indexPath = [self.tableView indexPathForSelectedRow];
        Event *object = [[self fetchedResultsController] objectAtIndexPath:indexPath];
        [[segue destinationViewController] setDetailItem:object];
    }
}
``` 

还有`configureCell: atIndexPath:`方法：  

``` 
- (void)configureCell:(UITableViewCell *)cell atIndexPath:(NSIndexPath *)indexPath
{
    Event *object = [self.fetchedResultsController objectAtIndexPath:indexPath];
//    cell.textLabel.text = [[object valueForKey:@"timeStamp"] description];
    cell.textLabel.text = [object.timeStamp description];
}
``` 
相应地我们也可以针对`DetailViewController`进行改造：  
DetailViewController.h:  

``` 
#import <UIKit/UIKit.h>
@class Event;
@interface DetailViewController : UIViewController

@property (strong, nonatomic) Event *detailItem;

@property (weak, nonatomic) IBOutlet UILabel *detailDescriptionLabel;
@end
``` 

DetailViewController.m:  

``` 
#import "DetailViewController.h"
#import "Event.h"
@interface DetailViewController ()
- (void)configureView;
@end

@implementation DetailViewController

#pragma mark - Managing the detail item

- (void)setDetailItem:(Event *)newDetailItem
{
    if (_detailItem != newDetailItem) {
        _detailItem = newDetailItem;
        
        // Update the view.
        [self configureView];
    }
}

- (void)configureView
{
    // Update the user interface for the detail item.

    if (self.detailItem) {
//        self.detailDescriptionLabel.text = [[self.detailItem valueForKey:@"timeStamp"] description];
        self.detailDescriptionLabel.text = [self.detailItem.timeStamp description];
    }
}

- (void)viewDidLoad
{
    [super viewDidLoad];
	// Do any additional setup after loading the view, typically from a nib.
    [self configureView];
}

- (void)didReceiveMemoryWarning
{
    [super didReceiveMemoryWarning];
    // Dispose of any resources that can be recreated.
}

@end
``` 
这部分比较简单，就不详细解释了，运行程序，跟以前一样（不截图了）  

如果想复用MasterViewController里面那些代码，需要做些大改动，具体可以参看[更轻量的 View Controllers](http://objccn.io/issue-1-1/)这篇文章  

