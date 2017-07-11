---
layout: post
title: 一个Rss阅读器的实现-UISplitViewController的使用
date: 2014-10-25 00:00
categories: [iOS]
---

缘起：学习iOS开发一段时间了，一直想找个东西练练手，正好平时阅读blog都用rss阅读器，所以索性就选定这个了。

首先建立一个新的Single View Application（为什么不用Master-Detail Application，因为不想用storyboard或者xib的方式，而直接采用编写代码的方式，这样可以对iOS开发中涉及到的技术有更好的理解），选定语言为Objective-c，设备为iPad，暂不使用core data，这样就新建了一个工程。

如果通过代码方式来进行编写，其中的Main.storyboard和LaunchSereen.xib，甚至生成的ViewController.h(.m)都是不需要的，所以将这四项删掉。这个时候运行app时会报如下的错误：
> Terminating app due to uncaught exception 'NSInvalidArgumentException', reason: 'Could not find a storyboard named 'Main' in bundle NSBundle

在工程设置的Info选项中，找到"Main storyboard file base name"，将其删掉，再运行时就不会报错了。此时的程序界面是一片黑色，这时将UISplitViewController作为根视图控制器：在AppDelegate.m的didFinishLaunchingWithOptions中添加如下代码：

	UISplitViewController *splitViewController = [[UISplitViewController alloc] init];
    UITableViewController *master = [[UITableViewController alloc] init];
    UITableViewController *detail = [[UITableViewController alloc] init];
    splitViewController.viewControllers = [NSArray arrayWithObjects:master, detail, nil];
    
	self.window = [[UIWindow alloc] initWithFrame:[[UIScreen mainScreen] bounds]];
    self.window.rootViewController = splitViewController;
    self.window.backgroundColor = [UIColor whiteColor];
    [self.window makeKeyAndVisible];

加入master及detail是因为UISplitViewController必须设置它的viewControllers属性。

这样，一个基础的界面就出来了。但是有个问题，当旋转设备为纵向时，并没有显示master表格的按钮。

新建两个UITableViewController：MasterViewController和DetailViewController做为UISplitViewController的viewController设置。并且修改didFinishLaunchingWithOptions如下：

	MasterViewController *master = [[MasterViewController alloc] init];
    DetailViewController *detail = [[DetailViewController alloc] init];
    UINavigationController *masterNav = [[UINavigationController alloc] initWithRootViewController:master];
    UINavigationController *detailNav = [[UINavigationController alloc] initWithRootViewController:detail];
    splitViewController.viewControllers = [NSArray arrayWithObjects:masterNav, detailNav, nil];

加入navigation controller是为了后面的操作做准备，并且引入了导航栏。

为了能够在detail视图中点击按钮时弹出master视图，要将DetailViewController实现UISplitViewControllerDelegate的关于显示与隐藏的方法。在DetailViewController中声明如下属性：

	@property (nonatomic, strong) UIPopoverController *navigationPopoverController;

在appDelegate中设置splitViewController的代理为detail：

	[splitViewController setDelegate:detail];

实现以下两个方法：

	- (void)splitViewController:(UISplitViewController *)svc willHideViewController:(UIViewController *)aViewController withBarButtonItem:(UIBarButtonItem *)barButtonItem forPopoverController:(UIPopoverController *)pc
	{
    	barButtonItem.title = @"master";
    	self.navigationPopoverController = pc;
    	self.navigationItem.leftBarButtonItem = barButtonItem;
	}

	- (void)splitViewController:(UISplitViewController *)svc willShowViewController:(UIViewController *)aViewController invalidatingBarButtonItem:(UIBarButtonItem *)barButtonItem
	{
    	self.navigationPopoverController = nil;
    	self.navigationItem.leftBarButtonItem = nil;
	}

这样，当选择设备为竖向时，点击master按钮，就从左侧弹出master视图。

至此，一个大致的表现形式就出来了。



