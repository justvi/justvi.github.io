---
layout: post
title: 一个Rss阅读器的实现-model引入CoreData
date: 2014-11-17
categories: [iOS]
---

之前留下一个问题就是，masterViewController的tableViewCell是写死的三个rss地址，不能添加，删除它。那么现在引入model层，使用Core Data来存储定于的rss地址。

先在工程中加入CoreData框架，然后新建一个Data Model，将其命名为Rss.xcdatamodeld。打开它，增加一个Entity，命名为Subscribe，为它添加一个属性rssUrl，类型为String。然后新建一个NSManagedObject subclass，选择data model中的Subscribe，自动生成了这个类。

接着在app delegate中引入core data支持。在AppDelegate.h中加入如下代码：

	@property (nonatomic, readonly, strong) NSManagedObjectContext *managedObjectContext;
	@property (nonatomic, readonly, strong) NSManagedObjectModel *managedObjectModel;
	@property (nonatomic, readonly, strong) NSPersistentStoreCoordinator *persistentStoreCoordinator;

	- (void)saveContext;
	- (NSURL *)applicationDocumentsDirectory;

并且在AppDelegate.m中编写core data stack代码，这里不列出。注意在managedObjectModel和persistentSotreCoordinator中修改resource参数为自己命名的data model名称，这里为@"Rss"和@"Rss.sqlite"。 

接着进行添加订阅源的代码编写。在MasterViewController.m中添加属性subscribes，用其来存储所有的订阅源

	@property (nonatomic, strong) NSMutableArray *subscribes;

在viewDidLoad中进行初始化
	
	self.subscribes = [NSMutableArray array];

修改numberOfRowsInSection为：

	return [_subscribes count];
	
cellForRowAtIndexPath中设置cell的text为subscribe的rssUrl：

	Subscribe *sub = [_subscribes objectAtIndex:indexPath.row];
    cell.textLabel.text = sub.rssUrl;

在MasterViewController.h中添加增加订阅的方法，然后在.m中实现：

	- (void)addSubscribe:(NSString *)url
	{
    	Subscribe *sub = [NSEntityDescription insertNewObjectForEntityForName:@"Subscribe" inManagedObjectContext:[APP_DELEGATE managedObjectContext]];
    	if (sub) {
        	sub.rssUrl = url;
        	NSError *err = nil;
        	[[APP_DELEGATE managedObjectContext] save:&err];
        	if (!err) {
            	[self.subscribes addObject:sub];
            	[self.tableView reloadData];
        	}
    	}
	}

接着在AddRssViewController中的键盘返回处理中想master发送这个消息：

	MasterViewController *controller = [self.navigationController.viewControllers objectAtIndex:0];
	if (self.textInput.text) {
    	[controller addSubscribe:self.textInput.text];
    }
    [self.navigationController popViewControllerAnimated:YES];

这样，添加订阅源的处理就完成了。接下来处理删除操作。

首先修改master view controller的canEditRowAtIndexPath返回YES使table处于可编辑状态。然后修改editingStyleForRowAtIndexPath返回UITableViewCellEditingStyleDelete使之之能进行删除的操作。然后修改commitEditingStyle来进行提交时的处理。

	if (editingStyle == UITableViewCellEditingStyleDelete) {
        // Delete the row from the data source
        [self.subscribes removeObjectAtIndex:indexPath.row];
        UITableViewCell *cell = [self.tableView cellForRowAtIndexPath:indexPath];
        [APP_DELEGATE deleteSubscribe:cell.textLabel.text];
        [tableView deleteRowsAtIndexPaths:@[indexPath] withRowAnimation:UITableViewRowAnimationFade];
    }

这里去的url作为参数传入appdelegate处理删除的方法中。在appdelegate中实现删除如下：

	- (void)deleteSubscribe:(NSString *)url
	{
    	NSArray *subs = [self getSubscribes];
    	for (Subscribe *sub in subs) {
        	if ([sub.rssUrl isEqualToString:url]) {
            	[self.managedObjectContext deleteObject:sub];
            	if ([sub isDeleted]) {
                	NSError *err = nil;
                	[self.managedObjectContext save:&err];
            	}
        	}
    	}
	}

那么删除的处理也完成了。但是当添加了一些订阅源，然后重启应用程序时，添加的并没有在master中显示出来，因为在master初始化显示时并没有取出这些数据。现在来添加它：

在AppDelegate中实现取得所有订阅的方法：

	- (NSArray *)getSubscribes
	{
    	NSFetchRequest *req = [[NSFetchRequest alloc] init];
    	NSEntityDescription *entity = [NSEntityDescription entityForName:@"Subscribe" inManagedObjectContext:self.managedObjectContext];
    	[req setEntity:entity];
    
    	NSError *err = nil;
    	NSArray *subs = [self.managedObjectContext executeFetchRequest:req error:&err];
    
    	return subs;
	}

然后再master view controller的viewDidLoad中subscribes初始化后添加：

	[self.subscribes addObjectsFromArray:[APP_DELEGATE getSubscribes]];

这样，重启时已添加的订阅也显示出来了。

至此，rss阅读器大致的功能都实现了。
