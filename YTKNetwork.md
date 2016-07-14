##源码解析之--[YTKNetwork](https://github.com/yuantiku/YTKNetwork)网络层
####[blog](http://www.jianshu.com/p/521a6437a0b6)

![IMG_0974.JPG](http://upload-images.jianshu.io/upload_images/1121012-9e8c5f2f08343caf.JPG?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#####首先
　　关于网络层最先可能想到的是```AFNetworking```，或者Swift中的```Alamofire```，直接使用起来也特别的简单，但是稍复杂的项目如果直接使用就显得不够用了，首先第三方耦合不说，就光散落在各处的请求回调就难以后期维护，所以一般会有针对性的再次封装，往往初期可能业务相对简单，考虑的方面较少，后期业务增加可能需要对网络层进行重构，一个好的架构也一定是和业务层面紧密相连的，随业务的增长不断健壮的。
　　最近也是看了YTKNetwork的源码和相关博客，站在前辈的肩膀上写下一些自己关于网络层的解读。
#####与业务层对接方式
常见的与业务层对接方式两种：
- 集约型：
　　最典型就属于上面说的```AFNetworking```、```Alamofire```，发起网络请求都集中在一个类上，请求回调通过Block、闭包实现的，Block、闭包回调有比较好的灵活性，可以方便的在任何位置发起请求，同时也可能是不好的地方，网络请求回调散落在各处，不便于维护。
　　下面是一个集约型的网络请求，大家使用集约型网络请求有没有遇到这么一个场景，请求回调后需要做比较多的处理，代码量多的时候，会再定义一个私有方法把代码写在里面，在Block中调用在私有方法。其实这样处理本质上和通过代理回调本质上是一样的。

```
[_manager GET:url parameters:param success:^(AFHTTPRequestOperation *operation, id responseObject) {
     //The data processing, Rendering interface
 } failure:^(AFHTTPRequestOperation *operation, NSError *error) {
                   
 }];

```
- 离散型：
　　对应的是接下来的YTKNetwork，离散型最大的特点就是一个网络请求对应一个单独的类，在这个类内部封装请求地址、方式、参数、校验和处理请求回来的数据，通常代理回调，需要跨层数据传递时也使用通知回调，比较集中，因为数据处理都放在内部处理了，返回数据的形式（模型化后的数据还是其他）不需要控制器关心，控制器只需要在代理返回的数据可以直接对渲染UI，让Controller更加轻量化。

发起请求

```
    NSString *userId = @"1";
    GetUserInfoApi *api = [[GetUserInfoApi alloc] initWithUserId:userId];
    [api start];
    api.delegate = self;

```
Delegate回调
```
- (void)requestFinished:(YTKBaseRequest *)request {
    NSLog(@"----- succeed ---- %@", request.responseJSONObject);
    //Rendering interface
}
- (void)requestFailed:(YTKBaseRequest *)request {
    NSLog(@"failed");
}

```
#####YTKNetwork解析
首先看下YTKNetwork的类文件：
![YTKNetwork.png](http://upload-images.jianshu.io/upload_images/1121012-64fa9892df3c891b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

图解它们之间的调用关系，注意还是理顺关系，看懂这个图应该对源码的理解没有太多问题：

![Scrren.png](http://upload-images.jianshu.io/upload_images/1121012-8d6b66e1e262782c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

- ```YTKBaseRequest```：YTKRequest的父类，定义了Request的相关属性，Block和Delegate。给对外接口默认的实现，以及公共逻辑。


- ```YTKRequest```：主要对缓存做处理，更新缓存、读取缓存、手动写入缓存，是否忽略缓存。这里采用归档形式缓存，请求方式、根路径、请求地址、请求参数、app版本号、敏感数据拼接再MD5作为缓存的文件名，保证唯一性。还提供设置缓存的保存时长，主要实现是通过获取缓存文件上次修改的时刻距离现在的时间和设置的缓存时长作比较，来判断是否真正发起请求，下面是发起请求的一些逻辑判断：

```
- (void)start {
    if (self.ignoreCache) {
        //如果忽略缓存 -> 网络请求
        [super start];
        return;
    }

    // check cache time
    if ([self cacheTimeInSeconds] < 0) {
        //验证缓存有效时间 -> 网络请求
        [super start];
        return;
    }

    // check cache version
    long long cacheVersionFileContent = [self cacheVersionFileContent];
    if (cacheVersionFileContent != [self cacheVersion]) {
        //验证缓存版本号，如果不一致 -> 网络请求
        [super start];
        return;
    }

    // check cache existance
    NSString *path = [self cacheFilePath];  //
    NSFileManager *fileManager = [NSFileManager defaultManager];
    if (![fileManager fileExistsAtPath:path isDirectory:nil]) {
        //根据文件路径，验证缓存是否存在，不存在 -> 网络请求
        [super start];
        return;
    }

    // check cache time 上次缓存文件时刻距离现在的时长 与 缓存有效时间 对比
    int seconds = [self cacheFileDuration:path];
    if (seconds < 0 || seconds > [self cacheTimeInSeconds]) {
        //上次缓存文件时刻距离现在的时长 > 缓存有效时间
        [super start];
        return;
    }

    // load cache
    _cacheJson = [NSKeyedUnarchiver unarchiveObjectWithFile:path];
    if (_cacheJson == nil) {    //取出缓存，如果没有 -> 网络请求
        [super start];
        return;
    }

    _dataFromCache = YES;
    //缓存请求成功后的数据
    
    [self requestCompleteFilter];   //代理
    
    YTKRequest *strongSelf = self;
    [strongSelf.delegate requestFinished:strongSelf];
    
    if (strongSelf.successCompletionBlock) {    //block回调
        strongSelf.successCompletionBlock(strongSelf);
    }
    [strongSelf clearCompletionBlock];
}

```

通过归档存储网络请求的数据：
```
- (void)saveJsonResponseToCacheFile:(id)jsonResponse {
    if ([self cacheTimeInSeconds] > 0 && ![self isDataFromCache]) {
        NSDictionary *json = jsonResponse;
        if (json != nil) {
            [NSKeyedArchiver archiveRootObject:json toFile:[self cacheFilePath]];
            [NSKeyedArchiver archiveRootObject:@([self cacheVersion]) toFile:[self cacheVersionFilePath]];
        }
    }
}

```





- ```YTKNetworkAgent```：真正发起网络请求的类，在```addRequest```方法里调用AFN的方法，这块可以方便的更换第三方库，还包括一些请求取消，插件的代理方法调用等，所有网络请求失败或者成功都会调用下面这个方法：

```
- (void)handleRequestResult:(AFHTTPRequestOperation *)operation {
    NSString *key = [self requestHashKey:operation];
    YTKBaseRequest *request = _requestsRecord[key];
    YTKLog(@"Finished Request: %@", NSStringFromClass([request class]));
    if (request) {
        
        BOOL succeed = [self checkResult:request];
        if (succeed) {  //请求成功
            [request toggleAccessoriesWillStopCallBack];    //调用执行加载动画插件
            [request requestCompleteFilter];
            if (request.delegate != nil) {  //请求成功代理回调
                [request.delegate requestFinished:request];
            }
            if (request.successCompletionBlock) {   //请求成功Block回调
                request.successCompletionBlock(request);
            }
            [request toggleAccessoriesDidStopCallBack];
        } else {            //请求失败
            YTKLog(@"Request %@ failed, status code = %ld",
                     NSStringFromClass([request class]), (long)request.responseStatusCode);
            [request toggleAccessoriesWillStopCallBack];    //调用执行加载动画插件
            [request requestFailedFilter];
            if (request.delegate != nil) {      //请求失败代理回调
                [request.delegate requestFailed:request];
            }
            if (request.failureCompletionBlock) {   //请求失败Block回调
                request.failureCompletionBlock(request);
            }
            [request toggleAccessoriesDidStopCallBack];
        }
    }
    [self removeOperation:operation];
    [request clearCompletionBlock];
}
```

- ```YTKNetworkConfig```：配置请求根路径、DNS地址。

- ```YTKNetworkPrivate```：可以理解为一个工具类，拼接地址，提供加密方法，定义分类等。

- ```YTKBatchRequest```、```YTKChainRequest```：这是YKTNetwork的两个高级用法，批量网络请求和链式的网络请求，相当于一个存放Request的容器，先定义下面属性，finishedCount来记录批量请求的完成的个数：

```
@interface YTKBatchRequest() <YTKRequestDelegate>
@property (nonatomic) NSInteger finishedCount;
@end
```
每完成一个请求finishedCount++，直到finishedCount等于所有请求的个数时才回调成功。
```
#pragma mark - Network Request Delegate

- (void)requestFinished:(YTKRequest *)request {
    _finishedCount++;
    if (_finishedCount == _requestArray.count) {
        [self toggleAccessoriesWillStopCallBack];
        if ([_delegate respondsToSelector:@selector(batchRequestFinished:)]) {
            [_delegate batchRequestFinished:self];
        }
        if (_successCompletionBlock) {
            _successCompletionBlock(self);
        }
        [self clearCompletionBlock];
        [self toggleAccessoriesDidStopCallBack];
    }
}

```
给Request绑定一个Index
```
@interface YTKChainRequest()<YTKRequestDelegate>
@property (assign, nonatomic) NSUInteger nextRequestIndex;
@end
```
从requestArray数组中依次取出发起网络请求，同时nextRequestIndex++，只要一个请求失败则触发失败的回调：
```
- (void)start {
    if (_nextRequestIndex > 0) {
        YTKLog(@"Error! Chain request has already started.");
        return;
    }
    if ([_requestArray count] > 0) {
        [self toggleAccessoriesWillStartCallBack];
        [self startNextRequest];
        [[YTKChainRequestAgent sharedInstance] addChainRequest:self];
    } else {
        YTKLog(@"Error! Chain request array is empty.");
    }
}

```

```
//下一个网络请求
- (BOOL)startNextRequest {
    if (_nextRequestIndex < [_requestArray count]) {
        YTKBaseRequest *request = _requestArray[_nextRequestIndex];
        _nextRequestIndex++;
        request.delegate = self;
        [request start];
        return YES;
    } else {
        return NO;
    }
}
```

