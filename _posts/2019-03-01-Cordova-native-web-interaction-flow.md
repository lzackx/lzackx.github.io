---
layout: post
title:  "Cordova原生与网页的交互流程"
date:   2019-03-01 09:00:00
categories: Framework
---
**Cordova原生与网页的交互流程**

&emsp;&emsp;[Cordova](https://cordova.apache.org/)是从PhoneGap中抽出的核心代码，用于方便开发跨平台的应用程序。本文将对iOS中Cordova的原生与网页的交互流程进行梳理，充分了解Cordova是如何利用HTML，CSS和JavaScript的技术进行iOS平台的开发。

#### 1. `CDVViewController`

&emsp;&emsp;在Cordova中，原生与网页的关系从`CDVViewController`中说起。

&emsp;&emsp;`CDVViewController.m`如下，实现代码较多，后面以生命周期为线进行梳理（看不过来建议双屏对比查看）。

1. 先看构造函数，对于`UIViewController`的各个构造函数，类中都调用同一个方法`__init`，在做一些初始化属性的操作外，也用通知中心对App的生命周期做了监听，这些监听也同时做了能让Web中相应App生命周期的通知。
```ObjC
- (void)__init
{
    if ((self != nil) && !self.initialized) {
        _commandQueue = [[CDVCommandQueue alloc] initWithViewController:self];
        _commandDelegate = [[CDVCommandDelegateImpl alloc] initWithViewController:self];
        [[NSNotificationCenter defaultCenter] addObserver:self selector:@selector(onAppWillTerminate:)
                                                     name:UIApplicationWillTerminateNotification object:nil];
        [[NSNotificationCenter defaultCenter] addObserver:self selector:@selector(onAppWillResignActive:)
                                                     name:UIApplicationWillResignActiveNotification object:nil];
        [[NSNotificationCenter defaultCenter] addObserver:self selector:@selector(onAppDidBecomeActive:)
                                                     name:UIApplicationDidBecomeActiveNotification object:nil];

        [[NSNotificationCenter defaultCenter] addObserver:self selector:@selector(onAppWillEnterForeground:)
                                                     name:UIApplicationWillEnterForegroundNotification object:nil];
        [[NSNotificationCenter defaultCenter] addObserver:self selector:@selector(onAppDidEnterBackground:)
                                                     name:UIApplicationDidEnterBackgroundNotification object:nil];

        // read from UISupportedInterfaceOrientations (or UISupportedInterfaceOrientations~iPad, if its iPad) from -Info.plist
        self.supportedOrientations = [self parseInterfaceOrientations:
            [[[NSBundle mainBundle] infoDictionary] objectForKey:@"UISupportedInterfaceOrientations"]];

        [self printVersion];
        [self printMultitaskingInfo];
        [self printPlatformVersionWarning];
        self.initialized = YES;
    }
}

- (id)initWithNibName:(NSString*)nibNameOrNil bundle:(NSBundle*)nibBundleOrNil
{
    self = [super initWithNibName:nibNameOrNil bundle:nibBundleOrNil];
    [self __init];
    return self;
}

- (id)initWithCoder:(NSCoder*)aDecoder
{
    self = [super initWithCoder:aDecoder];
    [self __init];
    return self;
}

- (id)init
{
    self = [super init];
    [self __init];
    return self;
}
```
2. 然后是`viewDidLoad`中对整个网页与原生交互的初始化
   1. 通过`[self loadSettings]`加载配置
      1. 优先寻找`configFile`属性指向的文件名，若无，默认取`config.xml`文件
      2. 初始化`configParser`属性（`NSXMLParser`解析器），用于解析1中的配置文件，解析完后的数据存于`CDVConfigParser`（实现了`NSXMLParserDelegate`协议）实例的属性中，Cordova中的插件就是在这里做映射的。
    1. 通过`[self createGapView]`初始化`webView`，之前从声明中知道`webView`是个`UIView`，这时候会通过`[self newCordovaViewWithFrame:webViewBounds]`创建这个视图，声明成`UIView`的好处就是可以让开发者自定义配置文件中的`CordovaWebViewEngine`和Web View的插件，当前版本则缺省使用`CDVUIWebViewEngine`作为`webViewEngine`，在这个类的内部则是强制转换`webView`为`UIWebView`来使用了。
    2. 通过从1中获取的配置解析出来的变量`startupPluginNames`获得启动就要加载的插件，在此处进行注册初始化（调用插件的`pluginInitialize`方法），过程中会记录每个插件的注册初始化所花的时间，并在最后记录到`pluginObjects`数组属性中，做成强引用，使插件不会被释放。
    3. 通过从1中获取并解析出来的属性中尝试读取Web View的初始页页面，页面存在就加载，不存在就报错误页面，假如链错误页面也没指定，就单纯加载一个显示错误信息的body标签。
```ObjC
- (void)viewDidLoad
{
    [super viewDidLoad];

    // Load settings
    [self loadSettings];

    NSString* backupWebStorageType = @"cloud"; // default value

    id backupWebStorage = [self.settings cordovaSettingForKey:@"BackupWebStorage"];
    if ([backupWebStorage isKindOfClass:[NSString class]]) {
        backupWebStorageType = backupWebStorage;
    }
    [self.settings setCordovaSetting:backupWebStorageType forKey:@"BackupWebStorage"];

    [CDVLocalStorage __fixupDatabaseLocationsWithBackupType:backupWebStorageType];

    // // Instantiate the WebView ///////////////

    if (!self.webView) {
        [self createGapView];
    }

    // /////////////////

    /*
     * Fire up CDVLocalStorage to work-around WebKit storage limitations: on all iOS 5.1+ versions for local-only backups, but only needed on iOS 5.1 for cloud backup.
        With minimum iOS 7/8 supported, only first clause applies.
     */
    if ([backupWebStorageType isEqualToString:@"local"]) {
        NSString* localStorageFeatureName = @"localstorage";
        if ([self.pluginsMap objectForKey:localStorageFeatureName]) { // plugin specified in config
            [self.startupPluginNames addObject:localStorageFeatureName];
        }
    }

    if ([self.startupPluginNames count] > 0) {
        [CDVTimer start:@"TotalPluginStartup"];

        for (NSString* pluginName in self.startupPluginNames) {
            [CDVTimer start:pluginName];
            [self getCommandInstance:pluginName];
            [CDVTimer stop:pluginName];
        }

        [CDVTimer stop:@"TotalPluginStartup"];
    }

    // /////////////////
    NSURL* appURL = [self appUrl];
    __weak __typeof__(self) weakSelf = self;

    [CDVUserAgentUtil acquireLock:^(NSInteger lockToken) {
        // Fix the memory leak caused by the strong reference.
        [weakSelf setLockToken:lockToken];
        if (appURL) {
            NSURLRequest* appReq = [NSURLRequest requestWithURL:appURL cachePolicy:NSURLRequestUseProtocolCachePolicy timeoutInterval:20.0];
            [self.webViewEngine loadRequest:appReq];
        } else {
            NSString* loadErr = [NSString stringWithFormat:@"ERROR: Start Page at '%@/%@' was not found.", self.wwwFolderName, self.startPage];
            NSLog(@"%@", loadErr);

            NSURL* errorUrl = [self errorURL];
            if (errorUrl) {
                errorUrl = [NSURL URLWithString:[NSString stringWithFormat:@"?error=%@", [loadErr stringByAddingPercentEncodingWithAllowedCharacters:NSCharacterSet.URLPathAllowedCharacterSet]] relativeToURL:errorUrl];
                NSLog(@"%@", [errorUrl absoluteString]);
                [self.webViewEngine loadRequest:[NSURLRequest requestWithURL:errorUrl]];
            } else {
                NSString* html = [NSString stringWithFormat:@"<html><body> %@ </body></html>", loadErr];
                [self.webViewEngine loadHTMLString:html baseURL:nil];
            }
        }
    }];
    
    // /////////////////
    
    NSString* bgColorString = [self.settings cordovaSettingForKey:@"BackgroundColor"];
    UIColor* bgColor = [self colorFromColorString:bgColorString];
    [self.webView setBackgroundColor:bgColor];
}
```
3. 后面的生命周期方法，都是简单继承实现并加上了通知中心对生命周期的通知。
```ObjC
-(void)viewWillAppear:(BOOL)animated
{
    [super viewWillAppear:animated];
    [[NSNotificationCenter defaultCenter] postNotification:[NSNotification notificationWithName:CDVViewWillAppearNotification object:nil]];
}

-(void)viewDidAppear:(BOOL)animated
{
    [super viewDidAppear:animated];
    [[NSNotificationCenter defaultCenter] postNotification:[NSNotification notificationWithName:CDVViewDidAppearNotification object:nil]];
}

-(void)viewWillDisappear:(BOOL)animated
{
    [super viewWillDisappear:animated];
    [[NSNotificationCenter defaultCenter] postNotification:[NSNotification notificationWithName:CDVViewWillDisappearNotification object:nil]];
}

-(void)viewDidDisappear:(BOOL)animated
{
    [super viewDidDisappear:animated];
    [[NSNotificationCenter defaultCenter] postNotification:[NSNotification notificationWithName:CDVViewDidDisappearNotification object:nil]];
}

-(void)viewWillLayoutSubviews
{
    [super viewWillLayoutSubviews];
    [[NSNotificationCenter defaultCenter] postNotification:[NSNotification notificationWithName:CDVViewWillLayoutSubviewsNotification object:nil]];
}

-(void)viewDidLayoutSubviews
{
    [super viewDidLayoutSubviews];
    [[NSNotificationCenter defaultCenter] postNotification:[NSNotification notificationWithName:CDVViewDidLayoutSubviewsNotification object:nil]];
}
```
4. 析构函数中释放一些内存（含插件，即插件是跟随页面析构做析构的）。
```ObjC
- (void)dealloc
{
    [[NSNotificationCenter defaultCenter] removeObserver:self];

    [CDVUserAgentUtil releaseLock:&_userAgentLockToken];
    [_commandQueue dispose];
    [[self.pluginObjects allValues] makeObjectsPerformSelector:@selector(dispose)];
}
```

**小结**：`CDVViewController`类及其子类的对象在建立页面的时候，为了让Web端能知道App情况会监听App中的生命周期，也会通过在页面的生命周期发送通知让原生部分可以获得页面生命周期，还会获取配置、注册带有`onload`为`true`参数标签的插件。进一步的话，更可以自定义`webViewEngine`实例的实现。


#### . `CDVPlugin`

&emsp;&emsp;在知道了插件的加载后，就需要知道插件里面的实现要做什么。

&emsp;&emsp;插件内有对页面生命周期的监听处理，但没有多做具体的实现，算是个可以参考的模版，值得注意的是`CDVViewController`的`webViewEngine`默认就是一个实现了`CDVWebViewEngineProtocol`协议的`CDVPlugin`实例。

&emsp;&emsp;插件的初始化方法中通常会为交互做些准备，而其他方法则是对监听事件的处理了。

#### 3. 补充

&emsp;&emsp;在网页与原生的交互中，只用上述Cordova中的代码还不够，还需要使用`NSURLProtocol`作为补充，目的是把一些需要注入到网页的代码根据网页的URL进行有条件的注入。这样的本地注入优于从服务器加载Javascript，但只适用与`UIWebView`的拦截，对于`WKWebView`的拦截用些奇技淫巧也是可行的，但Apple看样子并不推荐这么做。

&emsp;&emsp;`NSURLProtocol`的对象会在`UIWebView`的`UIWebViewDelegate`协议中的

```ObjC
- (BOOL)webView:(UIWebView*)webView shouldStartLoadWithRequest:(NSURLRequest*)request navigationType:(UIWebViewNavigationType)navigationType
```
方法允许后进行调用`NSURLProtocol`中的

```ObjC
/*! 
    @method startLoading
    @abstract Starts protocol-specific loading of a request. 
    @discussion When this method is called, the protocol implementation
    should start loading a request.
*/
- (void)startLoading;
```

&emsp;&emsp;这时候就可以通过判断网络请求的URL，筛选出需要注入的页面，并读取存放在本地的JavaScript文件，注入进Web View中了。此后才走`UIWebView`的`UIWebViewDelegate`协议中的

```ObjC
- (BOOL)webView:(UIWebView *)webView shouldStartLoadWithRequest:(NSURLRequest *)request navigationType:(UIWebViewNavigationType)navigationType;
- (void)webViewDidStartLoad:(UIWebView *)webView;
- (void)webViewDidFinishLoad:(UIWebView *)webView;
- (void)webView:(UIWebView *)webView didFailLoadWithError:(NSError *)error;
```

&emsp;&emsp;前端的JavaScript在加载后已经能通过拦截时注入的JavaScript代码进行对交互调用了。