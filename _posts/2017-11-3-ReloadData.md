---
layout: post
comments: true
title:  "UITableView 各个reload性能分析"
date:   2017-11-2 22:03:03 +0800
categories: iOS
---

苹果原生的UITableView 当数据源改变时有三个reload方法，分别是reloadData、reloadSections:withRowAnimation:、reloadRowsAtIndexPaths:withRowAnimation，本文主要是分析三个方法在Reload非动画时(animation参数使用UITableViewRowAnimationNone)的性能比较，主要是比较主线程的时间。凭感觉这三个方法的性能最好是ReloadRow，最差是ReloadData,下面实际比较一下是不是这样。Demo传送门：[DJTestTableViewReloadData](https://github.com/Dokay/DJTestDemo/tree/master/DJTestTableViewReloadData)

### 比较的方法
reloadSections:withRowAnimation:和reloadRowsAtIndexPaths:withRowAnimation是两个同步方法，reloadData是一个异步刷新方法。直接在方法调用前后埋点采集时间是不准确的，所以需要结束时间的采集在TableView的View刷新之后，而View的刷新在Runloop的beforeWarting回调里面，所以调用reloadData方法后的下个Runloop开始时TableView已经刷新完毕。dispatch_after是避免其他事件对检测的干扰。
```
- (void)onTouchTest{
    dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(1 * NSEC_PER_SEC)), dispatch_get_main_queue(), ^{
        __block NSTimeInterval begin, end;
        begin = CACurrentMediaTime();
        printf("start....\n");
        dispatch_async(dispatch_get_main_queue(), ^{
            printf("reload....\n");
//            [self.tableView reloadData];
            NSIndexPath *indexPath = [NSIndexPath indexPathForRow:3 inSection:0];
            [self.tableView reloadSections:[NSIndexSet indexSetWithIndex:0] withRowAnimation:UITableViewRowAnimationNone];
//            [self.tableView reloadRowsAtIndexPaths:@[indexPath] withRowAnimation:UITableViewRowAnimationNone];
            dispatch_async(dispatch_get_main_queue(),^{
                printf("end....\n");
                end = CACurrentMediaTime();
                printf(" %8.2f ms\n",((end - begin) * 1000));
            });
        });
    });
    
}
```

### 结果数据比较
使用测试机型分别是iPhone4S,系统版本iOS8.1.1，iPhone7，系统版本iOS11.1。由于第一次调用函数和从第二次开始后数据差异比较大，分开比较。先比较第二次开始后的稳定数据。

以下图表是稳定状态下的iPhone4S数据：
![](/images/posts/reloaddata/Reload_4S_2.png)

iPhone7数据：
![](/images/posts/reloaddata/Reload_7_2.png)

结论：
在iPhone 4S机器上当Cell数量小于1000时，ReloadData的性能是优于ReloadRow,当Cell数量大于1000时ReloadData的性能开始优于ReloadRow。ReloadSection的性能始终最差。
在iPhone 7机器上ReloadData的性能始终是优于ReloadRow,ReloadSection的性能始终最差。


下面比较第一次函数调用的数据：
第一次函数调用ReloadRow和ReloadSection都有一个特点就是相较于ReloadData，ReloadRow会多调用一次Cell的初始化方法，ReloadSection会多调用一屏数量的Cell的初始化方法。至于为什么会多调用一次初始化方法根据苹果的文档猜测，新初始化的Cell用来做动画了,下面是苹果关于reloadSections:withRowAnimation:的说明：
![](/images/posts/reloaddata/ReloadRow_apple.jpg)

此处包含Nib文件的Cell比较简单，只包含一个UIImageView和一个UILabel。具体可见文末的测试Demo。
以下图表是第一次与第二次开始后稳定的iPhone4S数据：
![](/images/posts/reloaddata/Reload_4S_1.png)

iPhone7数据：
![](/images/posts/reloaddata/Reload_7_1.png)

结论：
由于要多初始化一个含Nib的Cell，ReloadRow的性能劣于ReloadData在iPone4S和iPhone7上表现是一致的。

### 总结
#### 1.当Cell更新没有动画要求：
* 只有当Cell的数量比较少（少于1000）时，并且Cell的初始化比较简单时，在iPhone4S iOS8.1.1上ReloadRow性能最优;其他Cell稍复杂或者Cell的数量比较多时ReloadData性能最优。ReloadSection始终是效率最差的。
* 猜测ReloadRow慢的原因是即使使用了UITableViewRowAnimationNone，苹果仍然做了动画，只是这个动画的时间为0，由于要做动画，对整个TableView的调整复要高于直接刷新整个TableView。
* 由于我们平时使用的Cell的复杂度一般高于一个简单的Nib文件Cell初始化的时间，建议在平时的使用中优先使用ReloadData,其次ReloadRow,最后使用ReloadSection。

#### 2.当Cell有动画要求时：
* 当Cell更新有动画需求时，比如Cell的高度变化，使用reloadData将不会有任何动画，而reloadRow和ReloadSection有具体的平滑动画。


原创文章，转载请注明出处。

