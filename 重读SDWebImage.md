#### 重读SDWebImage

很早以前刚接触iOS开发的时候就知道SDWebImage的强大和流行，在大部分的iOS项目中都可以看到她的身影，也是相当经典的第三方库，非常值得阅读，只是一直以来都只是大致了解其运行的流程，并没有深入阅读源码，其中的许多设计和细节都非常值得学习，所以让我们再次去一探究竟。

首先我们从一张时序图来了解SDWebImage最主要的工作流程。![SDWebImageSequenceDiagram](SDWebImageSequenceDiagram.png)

其工作流程可以分为下列的步骤：

1. 控件调用setImageWithURL接口
2. 追溯到UIView的loadImageWithURL方法
3. 在SDWebImageManager中首先通过queryDiskCacheForKey方法寻找所请求的图片是否在沙盒中缓存
4. 如果有，则直接返回Image，否则通过downloadImage方法使用另外一个类SDWebImageDownloader下载图片，返回结果后，通过storeImage缓存图片，而后再返回Image
5. 把Image数据set到相应的控件上

了解了基本流程后，我们再看看具体的类结构。![SDWebImageClassDiagram](SDWebImageClassDiagram.png)

初看起来是比较复杂的一张类图，但仔细看看后可以发现其实他们之间的关系是比较清晰的。

- 首先我们从左上角开始看起，对UIButton、UIImageView等控件添加分类方法作为下载图片的入口，所有的方法最终走他们父类UIview的方法sd_internalSetImage
- SDWebImageManager 作为连接所有类的核心，在sd_internalSetImage方法中通过调用loadImageWithURL进入。
- SDWebImageCombinedOperation 描述的是一个在Manager层面的下载操作，里面包含了取消block还有真正的下载操作Operation。
- SDWebImageManagerDelegate 提供一些代理方法给外部使用
- SDImageCache 看其名字就知道是提供缓存功能的
- SDImageCacheConfig 配置缓存设置信息的属性及方法
- SDWebImageDownloader 看其名字也知道是用来下载图片的
- SDWebImageDownloaderOperation 一个真正的下载任务
- SDWebImageDownloadToken 用来描述一个下载任务的特征
- SDWebImageDownloaderOperationInterface 如果想要自定义一个下载任务的话，必须遵守的协议
- SDWebImagePrefetcher 用来预下载图片的，可以下载完先不使用

这么多类中，其实支撑SDWebImage运作的最主要的三个部分，他们分别是：SDWebImageManager、SDImageCache、SDWebImageDownloader下面我们逐一对其进行细细分析，看看他们到底是怎么工作的。

###### SDWebImageManager

如果我们查看SDWebImageManager.m的源码，会发现其中近一半多的篇幅都被单独一个方法占据了，他就是之前我们提过的：

~~~objective-c
- (id <SDWebImageOperation>)loadImageWithURL:(nullable NSURL *)url options:(SDWebImageOptions)options progress:(nullable SDWebImageDownloaderProgressBlock)progressBlock completed:(nullable SDInternalCompletionBlock)completedBlock;
~~~

很明显这个方法中的内容是SDWebImageManager中最核心的工作流程，下面我们逐一梳理其中的逻辑。

1. 检查URL的合法性

2. 检查URL是否曾经失败过

3. 结合上面的信息，选择是否提前回调。

   ~~~objective-c
   [self callCompletionBlockForOperation:operation completion:completedBlock error:[NSError errorWithDomain:NSURLErrorDomain code:NSURLErrorFileDoesNotExist userInfo:nil] url:url];
   ~~~

4. 根据URL获取CacheKey，通过CacheKey去本地寻找图片，现在内存中找，如果内存中没有就去沙盒中找，这里使用了GCD异步函数，专门在一个IO线程里做沙盒存取的操作。

   ~~~objective-c
   - (nullable NSOperation *)queryCacheOperationForKey:(nullable NSString *)key done:(nullable SDCacheQueryCompletedBlock)doneBlock {
       // First check the in-memory cache...
       UIImage *image = [self imageFromMemoryCacheForKey:key];
       if (image) {
           NSData *diskData = nil;
           if ([image isGIF]) {
               diskData = [self diskImageDataBySearchingAllPathsForKey:key];
           }
           if (doneBlock) {
               doneBlock(image, diskData, SDImageCacheTypeMemory);
           }
           return nil;
       }

       NSOperation *operation = [NSOperation new];
       dispatch_async(self.ioQueue, ^{
           if (operation.isCancelled) {
               // do not call the completion if cancelled
               return;
           }
           @autoreleasepool {
               NSData *diskData = [self diskImageDataBySearchingAllPathsForKey:key];
               UIImage *diskImage = [self diskImageForKey:key];
               if (diskImage && self.config.shouldCacheImagesInMemory) {
                   NSUInteger cost = SDCacheCostForImage(diskImage);
                   [self.memCache setObject:diskImage forKey:key cost:cost];
               }

               if (doneBlock) {
                   dispatch_async(dispatch_get_main_queue(), ^{
                       doneBlock(diskImage, diskData, SDImageCacheTypeDisk);
                   });
               }
           }
       });
   ~~~

5. 如果缓存中有，则回调，没有的话则去网络下载

   - 这里有个值得一说的点就是作者通过位移运算符做SDWebImageDownloaderOptions的枚举值，通过对枚举值进行或运算可以通过一个值描述多个枚举。

     ~~~objective-c
     SDWebImageDownloaderLowPriority = 1 << 0 //00000001
     SDWebImageDownloaderProgressiveDownload = 1 << 1 //00000010
     SDWebImageDownloaderUseNSURLCache = 1 << 2 //00000100
     options =  SDWebImageDownloaderLowPriority || SDWebImageDownloaderProgressiveDownload || SDWebImageDownloaderUseNSURLCache
     options = 00000111
     ~~~

   - 设置好配置信息后去网络下载

6. 如果下载成功后就把图片cache起来，然后回调

   ~~~objective-c
    if (downloadedImage && finished) {
   	[self.imageCache storeImage:downloadedImage imageData:downloadedData forKey:key toDisk:cacheOnDisk completion:nil];}
   [self callCompletionBlockForOperation:strongOperation completion:completedBlock image:downloadedImage data:downloadedData error:nil cacheType:SDImageCacheTypeNone finished:finished url:url];
   ~~~

以上就是SDWebImageManager的主要功能和流程，在这其中包含了对imageCache和imageDownloader的调用，下面我们就先说一说imageCache。

##### SDImageCache

其实SDImageCache的功能很简单，就是对图片的缓存，缓存有两个地方，一个是内存memory，另外一个是沙盒disk。SDImageCache分别提供了对这两个地方进行图片的存、取、删的功能。我们就通过逐一介绍这些方法作为切入点对SDImageCache进行分析。

- 存储Store

  ~~~objective-c
  - (void)storeImage:(nullable UIImage *)image
           imageData:(nullable NSData *)imageData
              forKey:(nullable NSString *)key
              toDisk:(BOOL)toDisk
          completion:(nullable SDWebImageNoParamsBlock)completionBlock;
  ~~~

  这些参数都很好理解，如果cacheconfig中设置了缓存在内存中，则先回把数据缓存进内存，然后再缓存至disk沙盒中。[self checkIfQueueIsIOQueue]检查当前queue是不是IOqueue。然后再把数据写入对应的路径中。

- 取Query

  ~~~objective-c
  - (nullable NSOperation *)queryCacheOperationForKey:(nullable NSString *)key done:(nullable SDCacheQueryCompletedBlock)doneBlock;
  ~~~

  最主要的方法就是这个，其中先去内存中寻找，如果没有命中，则异步去disk中再找，如果找到的话，则把当前图片缓存在内存里方便下一次寻找的的同时，返回图片。

- 删除Remove

  ~~~objective-c
  - (void)removeImageForKey:(nullable NSString *)key fromDisk:(BOOL)fromDisk withCompletion:(nullable SDWebImageNoParamsBlock)completion
  ~~~

  删除逻辑就比较简单了，如果是cacheconfig是shouldCacheImagesInMemory的话，先删除内存中的，再异步删除disk中的数据，然后跳回主线程执行回调。

##### SDWebImageDownloader

在SDWebImage中另一个非常重要的部分就是下载器SDWebImageDownloader了，总体其使用了NSURLSession和NSOperation进行设计和实现。下面这个方法就是整个下载器最主要的了。

~~~objective-c
- (nullable SDWebImageDownloadToken *)downloadImageWithURL:(nullable NSURL *)url options:(SDWebImageDownloaderOptions)options progress:(nullable SDWebImageDownloaderProgressBlock)progressBlock completed:(nullable SDWebImageDownloaderCompletedBlock)completedBlock;
~~~

这个方法首先会转入一个新的方法addProgressCallback，并且直接返回其的返回值。在addProgressCallback中会根据URL去URLOperations里面寻找是否有对应的Operation,没有的话，则把createCallback赋值给Operation。

~~~objective-c
operation = createCallback();
~~~

并且加入到数组URLOperations中保存。在URLOperations的完成回调函数里写，在Operation执行完毕后，从URLOperations中删除。

~~~objective-c
id downloadOperationCancelToken = [operation addHandlersForProgress:progressBlock completed:completedBlock];
token = [SDWebImageDownloadToken new];
token.url = url;
token.downloadOperationCancelToken = downloadOperationCancelToken;
~~~

通过addHandlersForProgress方法，把过程回调progressBlock和完成回调completedBlock封装在一个downloadOperationCancelToken里面，并且在callbackBlocks中保存。然后再生成一个SDWebImageDownloadToken，分别包含URL和Token信息。最终返回Token

那么到这里，让我们看看前面createCallback()中到底做了什么？

在createCallback()中，我们根据options设置了request的缓存策略

~~~objective-c
NSURLRequestCachePolicy cachePolicy = NSURLRequestReloadIgnoringLocalCacheData;
        if (options & SDWebImageDownloaderUseNSURLCache) {
            if (options & SDWebImageDownloaderIgnoreCachedResponse) {
                cachePolicy = NSURLRequestReturnCacheDataDontLoad;
            } else {
                cachePolicy = NSURLRequestUseProtocolCachePolicy;
            }
        }
~~~

生成了request，设置requestheader，cookies

~~~objective-c
NSMutableURLRequest *request = [[NSMutableURLRequest alloc] initWithURL:url cachePolicy:cachePolicy timeoutInterval:timeoutInterval];
        request.HTTPShouldHandleCookies = (options & SDWebImageDownloaderHandleCookies);
        request.HTTPShouldUsePipelining = YES;
        if (sself.headersFilter) {
            request.allHTTPHeaderFields = sself.headersFilter(url, [sself.HTTPHeaders copy]);
        }
        else {
            request.allHTTPHeaderFields = sself.HTTPHeaders;
        }
~~~

根据request和session生成SDWebImageDownloaderOperation。对Operation的一些属性进行设置，例如credentials（认证），queuePriority（队列中优先级）等等。

~~~objective-c
SDWebImageDownloaderOperation *operation = [[sself.operationClass alloc] initWithRequest:request inSession:sself.session options:options];
~~~

最后加入到downloadQueue队列中，并且添加上一个加入的lastAddedOperation对当前Operation的依赖。保证先进后出的顺序。（可以想象一下，你快速刷新Tableview的时候，同时有很多图片下载，但是如果是FIFO的话，你已经在底部了，图片还是从上往下，按照请求的顺序返回是不是不合理，合理的应该是你当前眼睛看到的那部分图片，也就是最后发出的那些请求先返回，这样就很好理解了。）

~~~objective-c
[sself.downloadQueue addOperation:operation];
        if (sself.executionOrder == SDWebImageDownloaderLIFOExecutionOrder) {
            [sself.lastAddedOperation addDependency:operation];
            sself.lastAddedOperation = operation;
        }
~~~

最后返回operation。

此外，还有另外一个方法，就是取消任务的功能。

~~~objective-c
- (void)cancel:(nullable SDWebImageDownloadToken *)token
~~~

首先进入到dispatch_barrier_async中执行任务，栅栏函数的作用是，在其队列中，保证在其之前的所有任务都执行完毕后才会执行，并且在其执行完毕后，后面的其他任务才能正常执行。倘若一个队列中所有的函数都是栅栏函数的话，那么这个队列虽然本身是并行队列，但实际效果是串行队列的效果。在dispatch_barrier_async中分别删除callbackBlocks和URLOperations中和token所对应的对象。

##### SDWebImageDownloaderOperation

首先，SDWebImageDownloaderOperation是继承于NSOperation的，把其加入一个队列中，就会自动执行，这是NSOperation的特点。这里的SDWebImageDownloaderOperation是一个自定义NSOperation，自定义NSOperation需要自己实现start方法，自己维护isFinished和isExecuting等属性。

~~~objective-c
- (void)start
~~~

在start方法里，大致做了生成unownedSession，dataTask，执行dataTask等工作。

~~~objective-c
[self.dataTask resume]
~~~

在开始执行之后，进行调用保存在callbackBlocks中的所有progressBlock，并且设置self.executing = YES。最后发送通知SDWebImageDownloadStartNotification，表示任务已经开始。

同时也要实现done函数，表示任务执行完毕。

~~~objective-c
- (void)done {
    self.finished = YES;
    self.executing = NO;
    [self reset];
}
~~~

一个小tips：

~~~~objective-c
@synthesize finished = _finished;
- (void)setFinished:(BOOL)finished {
    [self willChangeValueForKey:@"isFinished"];
    _finished = finished;
    [self didChangeValueForKey:@"isFinished"];
}
~~~~

这样写的好处是，当我们设置当前的finished时，我们相当于设置了isFinished，所有对其的KVO都会生效。

##### 一些阅读代码过程中的tips

- 使用@synchronized保证线程安全，下面的例子中就是相当于对runningOperations加了锁，保证的其在不同线程访问是不会产生错误。

~~~objective-c
@synchronized (self.runningOperations) {
        [self.runningOperations addObject:operation];
    }
~~~


