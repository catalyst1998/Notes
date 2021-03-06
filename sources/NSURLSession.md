# URL Loading System
## 1 概述

![overview](https://github.com/catalyst1998/Notes/blob/main/sources/image/URLSession/overview.png)

URL 加载系统（URL Loading System）使用标准协议（如 https）或自定义协议提供对 URL 标识资源进行访问。URL Loading System 是异步执行的，这样 app 可以保持响应，并在 response 到达时处理数据或错误。
使用URLSession实例创建一个或多个URLSessionTask实例，URLSessionTask实例可以拉取数据并将数据返回到 app、下载文件，或将文件、数据上传到远程服务器。使用URLSessionConfiguration对象配置URLSession的实例 session（会话），URLSessionConfiguration对象可以配置 caches、cookies 策略，以及是否允许使用数据流量等。
可以使用一个 session 重复创建 task。例如，浏览器为正常浏览和无痕模式使用单独的 session，无痕浏览不会保存数据到磁盘。



## 2 URLSessionConfiguration
URLSessionConfiguration相当于是管家，负责为seesion配置在网络交互过程中的各种属性。
URLSessionConfiguration 允许通过各种属性来设置configuration，包括setting cookie policies、setting security policies、setting caching policies、Setting HTTP Policy and Proxy Properties、Supporting Background Transfers等

![configuration属性](https://github.com/catalyst1998/Notes/blob/main/sources/image/URLSession/属性.png)

从上图上可以看到常用的属性：`allowsCellularAccess`--是否允许访问蜂窝网, `HTTPMaximumConnectionsPerHost`--最大连接数, `requestCachePolicy`--cache策略等

apple提供了三种类型的configuration

```objective-c
@property (class, readonly, strong) NSURLSessionConfiguration *defaultSessionConfiguration;
@property (class, readonly, strong) NSURLSessionConfiguration *ephemeralSessionConfiguration;
+ (NSURLSessionConfiguration *)backgroundSessionConfigurationWithIdentifier:(NSString *)identifier API_AVAILABLE(macos(10.10), ios(8.0), watchos(2.0), tvos(9.0));
```
官方对这几种类型的configuration的解释：
> - The singleton shared session (which has no configuration object) is for basic requests. It’s not as customizable as sessions that you create, but it serves as a good starting point if you have very limited requirements. You access this session by calling the shared class method. See that method’s discussion for more information about its limitations.
> - Default sessions behave much like the shared session (unless you customize them further), but let you obtain data incrementally using a delegate. You can create a default session configuration by calling the default method on the URLSessionConfiguration class.
> - Ephemeral sessions are similar to default sessions, but they don’t write caches, cookies, or credentials to disk. You can create an ephemeral session configuration by calling the ephemeral method on the URLSessionConfiguration class.
> - Background sessions let you perform uploads and downloads of content in the background while your app isn’t running. You can create a background session configuration by calling the backgroundSessionConfiguration(_:) method on the URLSessionConfiguration class.

对于shared session和default其实比较相似，只不过default可以自定义一些参数。default，it uses the disk-persisted global cache,credential and cookie storage objects. Ephemeral, store all data in memory.
其中`backgroundSessionConfiguration:`的`identifier`参数在整个app中必须是唯一的，如果出现重复其后果是严重的！
通过该identifier，我们可以创建新的background session，用于获取与其绑定的download or upload response。




## 3 URLSessionTask
> NSURLSessionTask - a cancelable object that refers to the lifetime of processing a given request.
URLSessionTask是真正干活的人，但他并没有初始方法，必须归属于seesion。
URLSessionTask是URL会话任务的抽象类，有四个具体的子类
![3](https://github.com/catalyst1998/Notes/blob/main/sources/image/URLSession/task.png)

- `URLSessionDataTask`：Use this task for GET requests to retrieve data from servers to memory.     
使用 `dataTask(with:)`方法创建`URLSessionDataTask`实例，datatask 用于请求资源(如：JSON、XML格式的数据），将服务器的响应作为一个或多个`NSData`对象返回到内存中。Default、ephemeral、shared session 支持`URLSessionDataTask`，`backgroundsession` 不支持`URLSessionDataTask`。

- `URLSessionUploadTask`：Use this task to upload a file from disk to a web service via a POST or PUT method.
使用 `uploadTask(with:from:)`方法创建`URLSessionUploadTask`实例，用于通过HTTP协议向服务器上传数据或文件,URLSessionUploadTask继承自URLSessionDataTask。使用URLSessionUploadTask 可以很方便为 request 提供 body（例如，POST 或 PUT），还可以在收到 response 前上传数据。此外，upload task 支持后台会话。
在 iOS 中，为 background session 创建 upload task 时，系统会将文件复制到临时目录，然后从临时目录上传。

- URLSessionDownloadTask：Use this task to download a file from a remote service to a temporary file location.
使用`downloadTask(with:)`方法创建URLSessionDownloadTask 实例，download task 将资源直接下载到磁盘上的文件。Download task 支持任何类型的会话。download task可以pause、resume

- URLSessionStreamTask：使用streamTask(withHostName:port:)或streamTask(with:)方法创建URLSessionStreamTask实例。流任务（stream task）从主机、端口或网络服务建立 TCP/IP连接。

*NOTE，在创建完任务之后，需要调用resume 方法才会启动任务。在任务完成或失败前，session 会强引用 task。如果没有特别用途，不需要维护对任务的引用。*


## 4 URLSession
### 4.1 URLSession overview
URLSession is both a class and a suite of classes for handling HTTP- and HTTPS-based requests.
URLSession是整个体系中的领导。
![2](https://github.com/catalyst1998/Notes/blob/main/sources/image/URLSession/2.png)

*NOTE：一个session可以创建多个task，并且session中的所有的task共享configuration*

### 4.2 Create URLSession 
[官方文档](https://developer.apple.com/documentation/foundation/nsurlsessionconfiguration?language=objc#topics)

> Configuration options for an NSURLSession.  When a session is created, a copy of the configuration object is made - you cannot modify the configuration of a session after it has been created.
可以通过单例sharedSession来使用session，也可以通过URLSessionConfiguration来创建configuration，配置session。

**创建NSURLSession的三个类方法**

```obejctive-c
// 使用default configuration并且该session中的task必须提供completionHandler
+ (NSURLSession *)sharedSession;

// 该session中的task必须提供completionHandler
+ (NSURLSession *)sessionWithConfiguration:(NSURLSessionConfiguration *)configuration;

+ (NSURLSession *)sessionWithConfiguration:(NSURLSessionConfiguration *)configuration delegate:(id<NSURLSessionDelegate>)delegate delegateQueue:(NSOperationQueue *)queue;
```
* NOTE：
- 在创建session的时候会拷贝configuration，并且在创建session之后，不允许修改configuration。
- NSURLSession对delegate是strong reference，直到我们显式地调用 invalidateAndCancel 或 resetWithCompletionHandler:方法。因此，在使用过程中要注意retain cycle。*

## 5 URLSessionDelegate
> NSURLSessionDelegate specifies the methods that a session delegate may respond to.  There are both session specific messages (for example, connection based auth) as well as task based messages.

![3](https://github.com/catalyst1998/Notes/blob/main/sources/image/URLSession/delegate.png)

URLSessionDelegate定义了URLSession实例调用delegate处理session事件的方法。除实现URLSessionDelegate协议内方法，大部分 delegate 还需要实现URLSessionTaskDelegate、URLSessionDataDelegate、URLSessionDownloadDelegate中的一个或多个协议，以便处理 task 级事件，
URLSessionDelegate 有三个optional方法
1. `- (void)URLSession: didBecomeInvalidWithError: `该方法通知session是否失效
> The last message a session receives.  A session will only become invalid because of a systemic error or when it has been explicitly invalidated, in which case the error parameter will be nil.

如果通过调用finishTasksAndInvalidate()方法使会话无效，会话会在最后一个 task 完成或失败后调用该方法
如果通过调用invalidateAndCancel()方法使会话无效，会话立即调用该方法。

> If implemented, when a connection level authentication challenge has occurred, this delegate will be given the opportunity to provide authentication credentials to the underlying connection. Some types of authentication will apply to more than one request on a given connection to a server (SSL Server Trust challenges). If this delegate message is not implemented, the behavior will be to use the default handling, which may involve user interaction.
2.` - (void)URLSession: didReceiveChallenge: completionHandler:` 该方法主要是处理远程服务器的会话级身份验证请求。遇到以下两种情况时会调用该方法：
  1. 远程服务器请求客户端证书，或 Windows NT LAN Manager（NTLM）认证时会调用该方法以提供适当的凭据。
  2. 当 session 与使用 SSL 或 TLS 的远程服务器首次建立连接时，使用该方法验证服务器的证书链。
如果未实现该方法，session 会调用URLSessionTaskDelegate协议中urlSession(_:task:didReceive:completionHandler:)方法，采用 task 级认证。

> If an application has received an application: handleEventsForBackgroundURLSession: completionHandler:  message, the session delegate will receive this message to indicate that all messages previously enqueued for this session have been delivered.  At this time it is safe to invoke the previously stored completion handler, or to begin any internal updates that will result in invoking the completion handler.
3. `- (void)URLSessionDidFinishEventsForBackgroundURLSession:`

## 6 URLSessionTaskDelegate
URLSessionTaskDelegate协议定义了 URL Session 实例调用 delegate 处理 task 级事件的方法。URLSessionTaskDelegate继承自URLSessionDelegate。
### 6.1 处理task生命周期
当任务完成是会调用`- (void)URLSession: task: didCompleteWithError:`方法。如果发生错误，error参数会包含失败的原因。

### 6.2 处理重定向
当远程服务器请求http重定向时，会调用`- (void)URLSession:task:willPerformHTTPRedirection:newRequest: completionHandler:`方法。
在该方法内必须调用 completion handler。如果允许重定向，为 completion handler 传入 request 参数；如果需要修改重定向，传入修改后的 request 对象；如果禁止重定向，则参数传 nil，此时得到的 response 就是重定向。

### 6.3 处理upload task
上传文件时会定期调用`- (void)URLSession: task: didSendBodyData: totalBytesSent: totalBytesExpectedToSend:`方法，以显示上传进度。

### 6.4 采集数据
调用 `- (void)URLSession: task: didFinishCollectingMetrics:`方法，收集task完成的信息。其中materics参数封装了task的指标

`URLSessionTaskMetrics`类包含以下三个属性：
- `taskInterval：`任务发起至任务完成的时间。
- `redirectCount：`任务执行过程中重定向次数。
- `transactionMetrics`：数组内元素为任务执行期间每个 request-response 事务度量标准。元素类型为URLSessionTaskTransactionMetrics。

`URLSessionTaskTransactionMetrics`对象封装执行会话任务期间收集的性能指标。每个`URLSessionTaskTransactionMetrics`对象包含了一个 request 和 response 属性，对应于 task 的 request 和 response。其也包含时间指标（temporal metrics），以fetchStartDate开始，以responseEndDate结束，以及其他特性，例如：networkProtocolName和resourceFetchType

![4](https://github.com/catalyst1998/Notes/blob/main/sources/image/URLSession/4.png)

可以使用该方法查看请求各阶段所占用的时间，优化性能。

## 7 URLSessionDataDelegate
`URLSessionDataDelegate`协议定义了 URL session 实例处理 data task、upload task 任务级事件方法。URLSessionDataDelegate继承自URLSessionTaskDelegate协议。

Data task 接收到服务器的初始回复（header）时，会调用`- (void)URLSession: dataTask：didReceiveResponse:  completionHandler:`方法
只有在接收到 response header 后需要取消任务，或将任务转变为 download task 时才需要实现该方法。未实现该方法时，默认允许继续传输数据。
可以在completionHandler中传入URLSession.ResponseDisposition常量。
- `URLSession.ResponseDisposition.allow：`任务继续作为 data task 执行。
- `URLSession.ResponseDisposition.cancel：`取消任务。
- `URLSession.ResponseDisposition.becomeDownload：`调用`urlSession(_:dataTask:didBecome:)`方法，创建一个 download task 取代当前的 data task。

Data task 接收到数据时会调用urlSession(_:dataTask:didReceive:)方法。该方法可能被调用多次，每次调用提供上次调用后的数据，你的 app 负责将所需数据拼接起来。

Data task 或 upload task 在接收完所有数据后会调用urlSession(_:dataTask:willCacheResponse:completionHandler:)方法，以决定是否将响应存储到缓存中。如果没有实现该方法，则根据会话的 configuration 决定是否保存。该方法的主要用途在于阻止指定 URL 缓存响应，或修改缓存的 userInfo 字典。实现该方法后必须调用 completionHandler，传入 proposed response 或修改后的 response 缓存数据，或nil禁止缓存 response。
只有在URLProtocol协议允许缓存 response 时，才会调用该方法。下面所有条件均成立时才会缓存响应：
- 请求是 HTTP 或 HTTPS 类型，也可以是支持缓存的自定义网络协议。
- 请求成功，即状态码在200至299区间。
- response 来自服务器，而非缓存。
- 会话配置允许缓存。
- URLRequest缓存策略允许缓存。
- 服务器响应中与缓存相关的 header 允许缓存。
- 响应大小足够小，能够进行缓存。例如，如果提供磁盘缓存，则响应不得大于磁盘缓存大小的5%。

## 8 CompletionHandler 还是 delegat
### 8.1 如何选择
前文说到的用于创建URLSeeion的类方法中，一个需要delegate，两个需要completionHandler。那么什么时候用delegate什么时候用completionHandler？


以下情况需要提供completionHandler，其余情况可以交给delegate处理
- 当App处于background、suspended以及out-of-process状态时，需要由background sessions上传或下载数据；
- 自定义网络鉴权；
- 自定义SSL验证；
- 由服务器返回的MIME type决定一个网络传输是下载到磁盘还是直接display；
- 通过body stream上传数据；
- 自定义cache；
- 自定义HTTP redirects；

### 8.2 区别
![5](https://github.com/catalyst1998/Notes/blob/main/sources/image/URLSession/5.png)
完成处理程序需要处理以下三件事：
1. 验证 error 参数是否为 nil。如果不为 nil，则传输时发生错误。此时应处理错误并退出。
2. 检查响应的状态码（status code）是否指示成功，以及 MIME 类型是否为预期值。如果不符合，处理服务器错误并退出。
3. 根据需要使用返回的 data。


![6](https://github.com/catalyst1998/Notes/blob/main/sources/image/URLSession/6.png)


## 9 总结
使用NSURLSession加载数据流程：
0. 创建configuration
1. 创建session
2. 通过url和参数创建task
3. 调用resume开始task
4. 在handler、delegate中处理回调



参考：
- [YouTube视频](https://www.youtube.com/watch?v=zvfViYmETuc&t=8s)
- [URLSession详解 · pro648/tips Wiki](https://github.com/pro648/tips/wiki/URLSession%e8%af%a6%e8%a7%a3)
- [URLSession Tutorial: Getting Started](https://www.raywenderlich.com/3244963-urlsession-tutorial-getting-started)
- [NSURLSession——网络框架新生代](http://zxfcumtcs.github.io/2014/08/02/NSUrlSession/)


