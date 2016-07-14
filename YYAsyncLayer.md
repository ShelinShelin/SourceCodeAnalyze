
## 源码解析之--[YYAsyncLayer](https://github.com/ibireme/YYAsyncLayer)异步绘制
####[blog](http://www.jianshu.com/p/7a353962b3d8)

#####前言
　　YYAsyncLayer是异步绘制与显示的工具。最初是从YYKitDemo中接触到这个工具，为了保证列表滚动流畅，将视图绘制、以及图片解码等任务放到后台线程，在YYAsyncLayer之前还是想从YYKitDemo中性能优化说起，虽然些跑题了...

##### YYKitDemo
　　对于列表主要对两个代理方法的优化，一个与绘制显示有关，另一个与计算布局有关：
```
- (UITableViewCell *)tableView:(UITableView *)tableView cellForRowAtIndexPath:(NSIndexPath *)indexPath;
- (CGFloat)tableView:(UITableView *)tableView heightForRowAtIndexPath:(NSIndexPath *)indexPath;  
```
　　常规逻辑可能觉得应该先调用```tableView : cellForRowAtIndexPath :```返回```UITableViewCell```对象，事实上调用顺序是先返回```UITableViewCell```的高度，是因为```UITableView```继承自```UIScrollView```，滑动范围由属性```contentSize```来确定，```UITableView```的滑动范围需要通过每一行的```UITableViewCell```的高度计算确定，复杂cell如果在列表滚动过程中计算可能会造成一定程度的卡顿。
　　假设有20条数据，当前屏幕显示5条，```tableView : heightForRowAtIndexPath :```方法会先执行20次返回所有高度并计算出滑动范围，```tableView : cellForRowAtIndexPath :```执行５次返回当前屏幕显示的cell个数。
　　

![TableViewOfPerformanceOptimization.png](http://upload-images.jianshu.io/upload_images/1121012-d95b3294b3ca0fc3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
　　从图中简单看下流程，从网络请求返回JSON数据，将Cell的高度以及内部视图的布局封装为Layout对象，Cell显示之前在异步线程计算好所有布局对象，并存入数组，每次调用```tableView: heightForRowAtIndexPath :```只需要从数组中取出，可避免重复的布局计算。同时在调用```tableView: cellForRowAtIndexPath :```对Cell内部视图异步绘制布局，以及图片的异步绘制解码，这里就要说到今天的主角YYAsyncLayer。

#####YYAsyncLayer

　　首先介绍里面几个类：
- YYAsyncLayer：继承自CALayer，绘制、创建绘制线程的部分都在这个类。
- YYTransaction：用于创建RunloopObserver监听MainRunloop的空闲时间，并将YYTranaction对象存放到集合中。
- YYSentinel：提供获取当前值的```value```（只读）属性，以及```- (int32_t)increase```自增加的方法返回一个新的```value```值，用于判断异步绘制任务是否被取消的工具。

![AsyncDisplay.png](http://upload-images.jianshu.io/upload_images/1121012-86524da67fb8b094.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

　　
　　上图是整体异步绘制的实现思路，后面一步步说明。现在假设需要绘制Label，其实是继承自UIView，重写```+ (Class)layerClass``` ，在需要重新绘制的地方调用下面方法，比如```setter```，```layoutSubviews```。
```
+ (Class)layerClass {
    return YYAsyncLayer.class;
}
- (void)setText:(NSString *)text {
    _text = text.copy;
    [[YYTransaction transactionWithTarget:self selector:@selector(contentsNeedUpdated)] commit];
}
- (void)layoutSubviews {
    [super layoutSubviews];
    [[YYTransaction transactionWithTarget:self selector:@selector(contentsNeedUpdated)] commit];
}
```
　　YYTransaction有```selector ```、```target ```的属性，```selector ```其实就是```contentsNeedUpdated ```方法，此时并不会立即在后台线程去更新显示，而是将YYTransaction对象本身提交保存在```transactionSet```的集合中，上图中所示。
```
+ (YYTransaction *)transactionWithTarget:(id)target selector:(SEL)selector{
    if (!target || !selector) return nil;
    YYTransaction *t = [YYTransaction new];
    t.target = target;
    t.selector = selector;
    return t;
}
- (void)commit {
    if (!_target || !_selector) return;
    YYTransactionSetup();
    [transactionSet addObject:self];
}
```
　　同时在YYTransaction.m中注册一个RunloopObserver，监听MainRunloop在```kCFRunLoopCommonModes```（包含```kCFRunLoopDefaultMode```、```UITrackingRunLoopMode```）下的```kCFRunLoopBeforeWaiting```和```kCFRunLoopExit```的状态，也就是说在一次Runloop空闲时去执行更新显示的操作。

>```kCFRunLoopBeforeWaiting```：Runloop将要进入休眠。
```kCFRunLoopExit```：即将退出本次Runloop。

```

static void YYTransactionSetup() {
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        transactionSet = [NSMutableSet new];
        CFRunLoopRef runloop = CFRunLoopGetMain();
        CFRunLoopObserverRef observer;
        observer = CFRunLoopObserverCreate(CFAllocatorGetDefault(),
                                           kCFRunLoopBeforeWaiting | kCFRunLoopExit,
                                           true,      // repeat
                                           0xFFFFFF,  // after CATransaction(2000000)
                                           YYRunLoopObserverCallBack, NULL);
        CFRunLoopAddObserver(runloop, observer, kCFRunLoopCommonModes);
        CFRelease(observer);
    });
}
```

　　下面是RunloopObserver的回调方法，从transactionSet取出transaction对象执行SEL的方法，分发到每一次Runloop执行，避免一次Runloop执行时间太长。

```
static void YYRunLoopObserverCallBack(CFRunLoopObserverRef observer, CFRunLoopActivity activity, void *info) {
    if (transactionSet.count == 0) return;
    NSSet *currentSet = transactionSet;
    transactionSet = [NSMutableSet new];
    [currentSet enumerateObjectsUsingBlock:^(YYTransaction *transaction, BOOL *stop) {
#pragma clang diagnostic push
#pragma clang diagnostic ignored "-Warc-performSelector-leaks"
        [transaction.target performSelector:transaction.selector];
#pragma clang diagnostic pop
    }];
}
```
　　接下来是异步绘制，这里用了一个比较巧妙的方法处理，当使用GCD时提交大量并发任务到后台线程导致线程被锁住、休眠的情况，创建与程序当前激活CPU数量（```activeProcessorCount```）相同的串行队列，并限制```MAX_QUEUE_COUNT```，将队列存放在数组中。
　　YYAsyncLayer.m有一个方法```YYAsyncLayerGetDisplayQueue```来获取这个队列用于绘制（这部分YYKit中有独立的工具YYDispatchQueuePool）。创建队列中有一个参数是告诉队列执行任务的服务质量quality of service，在iOS8+之后相比之前系统有所不同。
- iOS8之前队列优先级：

>```DISPATCH_QUEUE_PRIORITY_HIGH 2```                 高优先级
     ```DISPATCH_QUEUE_PRIORITY_DEFAULT 0```              默认优先级
     ```DISPATCH_QUEUE_PRIORITY_LOW (-2)```               低优先级
     ```DISPATCH_QUEUE_PRIORITY_BACKGROUND INT16_MIN```  后台优先级

- iOS8+之后：

>```QOS_CLASS_USER_INTERACTIVE 0x21```,              用户交互(希望尽快完成，不要放太耗时操作)
     ```QOS_CLASS_USER_INITIATED 0x19```,                用户期望(不要放太耗时操作)
     ```QOS_CLASS_DEFAULT 0x15```,                        默认(用来重置对列使用的)
     ```QOS_CLASS_UTILITY 0x11```,                        实用工具(耗时操作，可以使用这个选项)
    ``` QOS_CLASS_BACKGROUND 0x09```,                     后台
     ```QOS_CLASS_UNSPECIFIED 0x00```,                    未指定

```
/// Global display queue, used for content rendering.
static dispatch_queue_t YYAsyncLayerGetDisplayQueue() {
#ifdef YYDispatchQueuePool_h
    return YYDispatchQueueGetForQOS(NSQualityOfServiceUserInitiated);
#else
#define MAX_QUEUE_COUNT 16
    
    static int queueCount;
    static dispatch_queue_t queues[MAX_QUEUE_COUNT];            //存放队列的数组
    static dispatch_once_t onceToken;
    static int32_t counter = 0;
    dispatch_once(&onceToken, ^{
        //程序激活的处理器数量
        queueCount = (int)[NSProcessInfo processInfo].activeProcessorCount;        
        queueCount = queueCount < 1 ? 1 : (queueCount > MAX_QUEUE_COUNT ? MAX_QUEUE_COUNT : queueCount);
        if ([UIDevice currentDevice].systemVersion.floatValue >= 8.0) {
            for (NSUInteger i = 0; i < queueCount; i++) {
                dispatch_queue_attr_t attr = dispatch_queue_attr_make_with_qos_class(DISPATCH_QUEUE_SERIAL, QOS_CLASS_USER_INITIATED, 0);
                queues[i] = dispatch_queue_create("com.ibireme.yykit.render", attr);
            }
        } else {
            for (NSUInteger i = 0; i < queueCount; i++) {
                queues[i] = dispatch_queue_create("com.ibireme.yykit.render", DISPATCH_QUEUE_SERIAL);
                dispatch_set_target_queue(queues[i], dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0));
            }
        }
    });
    //给counter +1
    int32_t cur = OSAtomicIncrement32(&counter);
    if (cur < 0) cur = -cur;
    //每次从数组取出
    return queues[(cur) % queueCount];
#undef MAX_QUEUE_COUNT
#endif
}
```
　　接下来是关于绘制部分的代码，对外接口```YYAsyncLayerDelegate```代理中提供```- (YYAsyncLayerDisplayTask *)newAsyncDisplayTask```方法用于回调绘制的代码，以及是否异步绘制的BOOl类型属性```displaysAsynchronously```，同时重写CALayer的```display``` 方法来调用绘制的方法```- (void)_displayAsync:(BOOL)async```。
　　这里有必要了解关于后台的绘制任务何时会被取消，下面两种情况需要取消，并调用了YYSentinel的```increase```方法，使```value```值增加（线程安全）：
- 在视图调用```setNeedsDisplay```时说明视图的内容需要被更新，将当前的绘制任务取消，需要重新显示。
- 以及视图被释放调用了```dealloc```方法。

　　在YYAsyncLayer.h中定义了```YYAsyncLayerDisplayTask```类，有三个block属性用于绘制的回调操作，从命名可以看出分别是将要绘制，正在绘制，以及绘制完成的回调，可以从block传入的参数```BOOL(^isCancelled)(void)```判断当前绘制是否被取消。
```
@property (nullable, nonatomic, copy) void (^willDisplay)(CALayer *layer); 
@property (nullable, nonatomic, copy) void (^display)(CGContextRef context, CGSize size, BOOL(^isCancelled)(void));
@property (nullable, nonatomic, copy) void (^didDisplay)(CALayer *layer, BOOL finished);
```
　　下面是部分```- (void)_displayAsync:(BOOL)async```绘制的代码，主要是一些逻辑判断以及绘制函数，在异步执行之前通过```YYAsyncLayerGetDisplayQueue```创建的队列，这里通过```YYSentinel```判断当前的```value```是否等于之前的值，如果不相等，说明绘制任务被取消了，绘制过程会多次判断是否取消，如果是则return，保证被取消的任务能及时退出，如果绘制完毕则设置图片到```layer.contents```。

```
if (async) {  //异步
        if (task.willDisplay) task.willDisplay(self);
        YYSentinel *sentinel = _sentinel;
        int32_t value = sentinel.value;
        NSLog(@" --- %d ---", value);
        //判断当前计数是否等于之前计数
        BOOL (^isCancelled)() = ^BOOL() {
            return value != sentinel.value;
        };
        
        CGSize size = self.bounds.size;
        BOOL opaque = self.opaque;
        CGFloat scale = self.contentsScale;
        CGColorRef backgroundColor = (opaque && self.backgroundColor) ? CGColorRetain(self.backgroundColor) : NULL;
        if (size.width < 1 || size.height < 1) {    //视图宽高小于1
            CGImageRef image = (__bridge_retained CGImageRef)(self.contents);
            self.contents = nil;
            if (image) {
                dispatch_async(YYAsyncLayerGetReleaseQueue(), ^{
                    CFRelease(image);
                });
            }
            if (task.didDisplay) task.didDisplay(self, YES);
            CGColorRelease(backgroundColor);
            return;
        }
        //异步绘制
        dispatch_async(YYAsyncLayerGetDisplayQueue(), ^{
            if (isCancelled()) {
                CGColorRelease(backgroundColor);
                return;
            }
            //开启图片上下文
            UIGraphicsBeginImageContextWithOptions(size, opaque, scale);
            CGContextRef context = UIGraphicsGetCurrentContext();
            if (opaque) {   //不透明
                CGContextSaveGState(context);
                {
                    if (!backgroundColor || CGColorGetAlpha(backgroundColor) < 1) {
                        CGContextSetFillColorWithColor(context, [UIColor whiteColor].CGColor);
                        CGContextAddRect(context, CGRectMake(0, 0, size.width * scale, size.height * scale));
                        CGContextFillPath(context);
                    }
                    if (backgroundColor) {
                        CGContextSetFillColorWithColor(context, backgroundColor);
                        CGContextAddRect(context, CGRectMake(0, 0, size.width * scale, size.height * scale));
                        CGContextFillPath(context);
                    }
                }
                CGContextRestoreGState(context);
                CGColorRelease(backgroundColor);
            }
            task.display(context, size, isCancelled);
            if (isCancelled()) {
                
                UIGraphicsEndImageContext();
                dispatch_async(dispatch_get_main_queue(), ^{
                    if (task.didDisplay) task.didDisplay(self, NO);
                });
                return;
            }
            //从当前图片上下文获取图片
            UIImage *image = UIGraphicsGetImageFromCurrentImageContext();
            UIGraphicsEndImageContext();
            if (isCancelled()) {
                dispatch_async(dispatch_get_main_queue(), ^{
                    if (task.didDisplay) task.didDisplay(self, NO);
                });
                return;
            }
            dispatch_async(dispatch_get_main_queue(), ^{
                if (isCancelled()) {
                    if (task.didDisplay) task.didDisplay(self, NO);
                } else {
                    self.contents = (__bridge id)(image.CGImage);
                    if (task.didDisplay) task.didDisplay(self, YES);
                }
            });
        });

```

　　
