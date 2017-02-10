#使用 CTMediator（Target-Action）方式实现组件解耦
##一、普通页面跳转用法
假设我们有个页面叫 OneViewController，当前页面为 HomeViewController，普通情况下页面的间的跳转方式如下：
<pre><code>#import "HomeViewController.h"
 #import "OneViewController.h"
@implementation HomeViewController
- (void)aButtonClick:(UIButton *)sender {
    OneViewController *viewController = [[OneViewController alloc] init];
    viewController.name = @"普通用法";  //传递必要参数
    [self.navigationController pushViewController:viewController animated:YES];
}
@end
</code></pre>

这样做看上去没什么问题，实际也没什么问题。 但是，考虑以下情况：

1、如果 HomeViewController 里有 N 个这样的 button 事件，每个点击后的跳转都是不同的页面，那么则 HomeViewController 里，需要导入 N 个这样的 OneViewController.h;
2、如果 HomeViewController 是一个可以移植到其它项目的业务模块，在拖出首页 HomeVC 相关的业务代码时，难道还要把 'HomeViewController.m' 导入的 N 个其它 XxxViewController.h 都一块拖到新项目中么？

这点就是因为代码的耦合导致了首页 HomeVC 没法方便的移植。

说这样没有问题，是因为普通情况下，我们并没有移植 HomeVC 到其它项目的需求。

##二、Target-Action 实现页面跳转
采用的是 CTMediator 这套方案

还是假设我们有个页面叫 NewsViewController, 当前页面为HomeViewController 那么，我们按照CTMediator设计的架构来写一遍这个流程

###1、创建Target-Action
创建一个 Target_News 类，在这个文件里，我们主要生成 NewsViewController 实例并为其进行一些必要的赋值。例如:

<pre><code>// Target_News.h

 #import "Foundation/Foundation.h"
 #import "UIKit/UIKit.h"

@interface Target_News : NSObject

- (UIViewController *)Action_NativeToNewsViewController:(NSDictionary *)params;

@end
</code></pre>

这个类需要直接 #import "NewsViewController.h"

<pre><code>// Target_News.m

#import "Target_News.h"
#import "NewsViewController.h"

@implementation Target_News

- (UIViewController *)Action_NativeToNewsViewController:(NSDictionary *)params {
    NewsViewController *newsVC = [[NewsViewController alloc] init];

    if ([params valueForKey:@"newsID"]) {
        newsVC.newsID = params[@"newsID"];
    }

    return newsVC;
}

@end
</code></pre>

###2、创建 CTMediator 的Category
CTMediator+NewsActions.这个Category利用Runtime调用我们刚刚生成的Target_News。

由于利用了Runtime，导致我们完全不用#import刚刚生成的Target_News即可执行里面的方法，所以这一步，两个类是完全解耦的。也即是说，我们在完全解耦的情况下获取到了我们需要的NewsViewController。例如：

<pre><code>// CTMediator+NewsActions.h

#import "CTMediator.h"
#import "UIKit/UIKit.h"

@interface CTMediator (NewsActions)

- (UIViewController *)yt_mediator_newsViewControllerWithParams:(NSDictionary *)dict;

@end
</code></pre>

<pre><code>// CTMediator+NewsActions.m

#import "CTMediator+NewsActions.h"

NSString * const kCTMediatorTarget_News = @"News";
NSString * const kCTMediatorActionNativTo_NewsViewController = @"NativeToNewsViewController";

@implementation CTMediator (NewsActions)

- (UIViewController *)yt_mediator_newsViewControllerWithParams:(NSDictionary *)dict {

    UIViewController *viewController = [self performTarget:kCTMediatorTarget_News
                                                    action:kCTMediatorActionNativTo_NewsViewController
                                                    params:dict];
    if ([viewController isKindOfClass:[UIViewController class]]) {
        return viewController;
    } else {
        NSLog(@"%@ 未能实例化页面", NSStringFromSelector(_cmd));
        return [[UIViewController alloc] init];
    }
}

@end
</code></pre>

###3、最终使用
由于在Target中，传递值得方式采用了去Model化得方式，导致我们在整个过程中也没有#import任何Model。所以，我们的每个类都与Model解耦。

<pre><code>// HomeViewController.m

#import "HomeViewController.h"
#import "CTMediator+NewsActions.h"

@implementation HomeViewController

- (void)bButtonClick:(UIButton *)sender {    
    UIViewController *viewController = [[CTMediator sharedInstance] yt_mediator_newsViewControllerWithParams:@{@"newsID":@"123456"}];
    [self.navigationController pushViewController:viewController animated:YES];
}
@end
</code></pre>

###4、不足之处
这里其实唯一的问题就是，Target_Action里不得不填入一些 Hard Code，就是对创建的VC的赋值语句。不过这也是为了达到最大限度的解耦和灵活度而做的权衡。

<pre>
//  1. kCTMediatorTarget_News字符串 是 Target_xxx.h 中的 xxx 部分
NSString * const kCTMediatorTarget_News = @"News";

//  2. kCTMediatorActionNativTo_NewsViewController 是 Target_xxx.h 中 定义的 Action_xxxx 函数名的 xxx 部分
NSString * const kCTMediatorActionNativTo_NewsViewController = @"NativeToNewsViewController";
</pre>




