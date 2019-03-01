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

##### 1.1 `CDVViewController.h`

`CDVViewController.m`如下所示（留意额外添加的注释）：

```ObjC
#import <UIKit/UIKit.h>
#import <Foundation/NSJSONSerialization.h>
#import "CDVAvailability.h"
#import "CDVInvokedUrlCommand.h"
#import "CDVCommandDelegate.h"
#import "CDVCommandQueue.h"
#import "CDVScreenOrientationDelegate.h"
#import "CDVPlugin.h"
#import "CDVWebViewEngineProtocol.h"

// CDVViewController是UIViewController的子类，可以想起它的生命周期会调用的方法
@interface CDVViewController : UIViewController <CDVScreenOrientationDelegate>{
    @protected
    id <CDVWebViewEngineProtocol> _webViewEngine;
    @protected
    id <CDVCommandDelegate> _commandDelegate;
    @protected
    CDVCommandQueue* _commandQueue;
    NSString* _userAgent;
}

// 注意webView在这里只作为UIView的角色存在
@property (nonatomic, readonly, weak) IBOutlet UIView* webView;

@property (nonatomic, readonly, strong) NSMutableDictionary* pluginObjects;
@property (nonatomic, readonly, strong) NSDictionary* pluginsMap;
@property (nonatomic, readonly, strong) NSMutableDictionary* settings;
@property (nonatomic, readonly, strong) NSXMLParser* configParser;

@property (nonatomic, readwrite, copy) NSString* configFile;
@property (nonatomic, readwrite, copy) NSString* wwwFolderName;
@property (nonatomic, readwrite, copy) NSString* startPage;
@property (nonatomic, readonly, strong) CDVCommandQueue* commandQueue;
@property (nonatomic, readonly, strong) id <CDVWebViewEngineProtocol> webViewEngine;
@property (nonatomic, readonly, strong) id <CDVCommandDelegate> commandDelegate;

/**
 The complete user agent that Cordova will use when sending web requests.
 */
@property (nonatomic, readonly) NSString* userAgent;

/**
 The base user agent data that Cordova will use to build its user agent.  If this
 property isn't set, Cordova will use the standard web view user agent as its
 base.
 */
@property (nonatomic, readwrite, copy) NSString* baseUserAgent;

/**
	Takes/Gives an array of UIInterfaceOrientation (int) objects
	ex. UIInterfaceOrientationPortrait
*/
@property (nonatomic, readwrite, strong) NSArray* supportedOrientations;

/**
 The address of the lock token used for controlling access to setting the user-agent
 */
@property (nonatomic, readonly) NSInteger* userAgentLockToken;

- (UIView*)newCordovaViewWithFrame:(CGRect)bounds;

- (NSString*)appURLScheme;
- (NSURL*)errorURL;

- (UIColor*)colorFromColorString:(NSString*)colorString;
- (NSArray*)parseInterfaceOrientations:(NSArray*)orientations;
- (BOOL)supportsOrientation:(UIInterfaceOrientation)orientation;

- (id)getCommandInstance:(NSString*)pluginName;
- (void)registerPlugin:(CDVPlugin*)plugin withClassName:(NSString*)className;
- (void)registerPlugin:(CDVPlugin*)plugin withPluginName:(NSString*)pluginName;

- (void)parseSettingsWithParser:(NSObject <NSXMLParserDelegate>*)delegate;

@end
```

##### 1.2 `CDVViewController.m`

&emsp;&emsp;`CDVViewController.m`如下，实现代码较多，后面以生命周期为线进行梳理（看不过来建议双屏对比查看）。

```ObjC
#import <objc/message.h>
#import "CDV.h"
#import "CDVPlugin+Private.h"
#import "CDVUIWebViewDelegate.h"
#import "CDVConfigParser.h"
#import "CDVUserAgentUtil.h"
#import <AVFoundation/AVFoundation.h>
#import "NSDictionary+CordovaPreferences.h"
#import "CDVLocalStorage.h"
#import "CDVCommandDelegateImpl.h"
#import <Foundation/NSCharacterSet.h>

@interface CDVViewController () {
    NSInteger _userAgentLockToken;
}

@property (nonatomic, readwrite, strong) NSXMLParser* configParser;
@property (nonatomic, readwrite, strong) NSMutableDictionary* settings;
@property (nonatomic, readwrite, strong) NSMutableDictionary* pluginObjects;
@property (nonatomic, readwrite, strong) NSMutableArray* startupPluginNames;
@property (nonatomic, readwrite, strong) NSDictionary* pluginsMap;
@property (nonatomic, readwrite, strong) id <CDVWebViewEngineProtocol> webViewEngine;

@property (readwrite, assign) BOOL initialized;

@property (atomic, strong) NSURL* openURL;

@end

@implementation CDVViewController

@synthesize supportedOrientations;
@synthesize pluginObjects, pluginsMap, startupPluginNames;
@synthesize configParser, settings;
@synthesize wwwFolderName, startPage, initialized, openURL, baseUserAgent;
@synthesize commandDelegate = _commandDelegate;
@synthesize commandQueue = _commandQueue;
@synthesize webViewEngine = _webViewEngine;
@dynamic webView;

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

- (void)printVersion
{
    NSLog(@"Apache Cordova native platform version %@ is starting.", CDV_VERSION);
}

- (void)printPlatformVersionWarning
{
    if (!IsAtLeastiOSVersion(@"8.0")) {
        NSLog(@"CRITICAL: For Cordova 4.0.0 and above, you will need to upgrade to at least iOS 8.0 or greater. Your current version of iOS is %@.",
            [[UIDevice currentDevice] systemVersion]
            );
    }
}

- (void)printMultitaskingInfo
{
    UIDevice* device = [UIDevice currentDevice];
    BOOL backgroundSupported = NO;

    if ([device respondsToSelector:@selector(isMultitaskingSupported)]) {
        backgroundSupported = device.multitaskingSupported;
    }

    NSNumber* exitsOnSuspend = [[NSBundle mainBundle] objectForInfoDictionaryKey:@"UIApplicationExitsOnSuspend"];
    if (exitsOnSuspend == nil) { // if it's missing, it should be NO (i.e. multi-tasking on by default)
        exitsOnSuspend = [NSNumber numberWithBool:NO];
    }

    NSLog(@"Multi-tasking -> Device: %@, App: %@", (backgroundSupported ? @"YES" : @"NO"), (![exitsOnSuspend intValue]) ? @"YES" : @"NO");
}

-(NSString*)configFilePath{
    NSString* path = self.configFile ?: @"config.xml";

    // if path is relative, resolve it against the main bundle
    if(![path isAbsolutePath]){
        NSString* absolutePath = [[NSBundle mainBundle] pathForResource:path ofType:nil];
        if(!absolutePath){
            NSAssert(NO, @"ERROR: %@ not found in the main bundle!", path);
        }
        path = absolutePath;
    }

    // Assert file exists
    if (![[NSFileManager defaultManager] fileExistsAtPath:path]) {
        NSAssert(NO, @"ERROR: %@ does not exist. Please run cordova-ios/bin/cordova_plist_to_config_xml path/to/project.", path);
        return nil;
    }

    return path;
}

- (void)parseSettingsWithParser:(NSObject <NSXMLParserDelegate>*)delegate
{
    // read from config.xml in the app bundle
    NSString* path = [self configFilePath];

    NSURL* url = [NSURL fileURLWithPath:path];

    self.configParser = [[NSXMLParser alloc] initWithContentsOfURL:url];
    if (self.configParser == nil) {
        NSLog(@"Failed to initialize XML parser.");
        return;
    }
    [self.configParser setDelegate:((id < NSXMLParserDelegate >)delegate)];
    [self.configParser parse];
}

- (void)loadSettings
{
    CDVConfigParser* delegate = [[CDVConfigParser alloc] init];

    [self parseSettingsWithParser:delegate];

    // Get the plugin dictionary, whitelist and settings from the delegate.
    self.pluginsMap = delegate.pluginsDict;
    self.startupPluginNames = delegate.startupPluginNames;
    self.settings = delegate.settings;

    // And the start folder/page.
    if(self.wwwFolderName == nil){
        self.wwwFolderName = @"www";
    }
    if(delegate.startPage && self.startPage == nil){
        self.startPage = delegate.startPage;
    }
    if (self.startPage == nil) {
        self.startPage = @"index.html";
    }

    // Initialize the plugin objects dict.
    self.pluginObjects = [[NSMutableDictionary alloc] initWithCapacity:20];
}

- (NSURL*)appUrl
{
    NSURL* appURL = nil;

    if ([self.startPage rangeOfString:@"://"].location != NSNotFound) {
        appURL = [NSURL URLWithString:self.startPage];
    } else if ([self.wwwFolderName rangeOfString:@"://"].location != NSNotFound) {
        appURL = [NSURL URLWithString:[NSString stringWithFormat:@"%@/%@", self.wwwFolderName, self.startPage]];
    } else if([self.wwwFolderName rangeOfString:@".bundle"].location != NSNotFound){
        // www folder is actually a bundle
        NSBundle* bundle = [NSBundle bundleWithPath:self.wwwFolderName];
        appURL = [bundle URLForResource:self.startPage withExtension:nil];
    } else if([self.wwwFolderName rangeOfString:@".framework"].location != NSNotFound){
        // www folder is actually a framework
        NSBundle* bundle = [NSBundle bundleWithPath:self.wwwFolderName];
        appURL = [bundle URLForResource:self.startPage withExtension:nil];
    } else {
        // CB-3005 strip parameters from start page to check if page exists in resources
        NSURL* startURL = [NSURL URLWithString:self.startPage];
        NSString* startFilePath = [self.commandDelegate pathForResource:[startURL path]];

        if (startFilePath == nil) {
            appURL = nil;
        } else {
            appURL = [NSURL fileURLWithPath:startFilePath];
            // CB-3005 Add on the query params or fragment.
            NSString* startPageNoParentDirs = self.startPage;
            NSRange r = [startPageNoParentDirs rangeOfCharacterFromSet:[NSCharacterSet characterSetWithCharactersInString:@"?#"] options:0];
            if (r.location != NSNotFound) {
                NSString* queryAndOrFragment = [self.startPage substringFromIndex:r.location];
                appURL = [NSURL URLWithString:queryAndOrFragment relativeToURL:appURL];
            }
        }
    }

    return appURL;
}

- (NSURL*)errorURL
{
    NSURL* errorUrl = nil;

    id setting = [self.settings cordovaSettingForKey:@"ErrorUrl"];

    if (setting) {
        NSString* errorUrlString = (NSString*)setting;
        if ([errorUrlString rangeOfString:@"://"].location != NSNotFound) {
            errorUrl = [NSURL URLWithString:errorUrlString];
        } else {
            NSURL* url = [NSURL URLWithString:(NSString*)setting];
            NSString* errorFilePath = [self.commandDelegate pathForResource:[url path]];
            if (errorFilePath) {
                errorUrl = [NSURL fileURLWithPath:errorFilePath];
            }
        }
    }

    return errorUrl;
}

- (UIView*)webView
{
    if (self.webViewEngine != nil) {
        return self.webViewEngine.engineWebView;
    }

    return nil;
}

// Implement viewDidLoad to do additional setup after loading the view, typically from a nib.
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

- (void)setLockToken:(NSInteger)lockToken
{
	_userAgentLockToken = lockToken;
	[CDVUserAgentUtil setUserAgent:self.userAgent lockToken:lockToken];
}

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

-(void)viewWillTransitionToSize:(CGSize)size withTransitionCoordinator:(id<UIViewControllerTransitionCoordinator>)coordinator
{
    [super viewWillTransitionToSize:size withTransitionCoordinator:coordinator];
    [[NSNotificationCenter defaultCenter] postNotification:[NSNotification notificationWithName:CDVViewWillTransitionToSizeNotification object:[NSValue valueWithCGSize:size]]];
}

- (UIColor*)colorFromColorString:(NSString*)colorString
{
    // No value, nothing to do
    if (!colorString) {
        return nil;
    }
    
    // Validate format
    NSError* error = NULL;
    NSRegularExpression* regex = [NSRegularExpression regularExpressionWithPattern:@"^(#[0-9A-F]{3}|(0x|#)([0-9A-F]{2})?[0-9A-F]{6})$" options:NSRegularExpressionCaseInsensitive error:&error];
    NSUInteger countMatches = [regex numberOfMatchesInString:colorString options:0 range:NSMakeRange(0, [colorString length])];
    
    if (!countMatches) {
        return nil;
    }
    
    // #FAB to #FFAABB
    if ([colorString hasPrefix:@"#"] && [colorString length] == 4) {
        NSString* r = [colorString substringWithRange:NSMakeRange(1, 1)];
        NSString* g = [colorString substringWithRange:NSMakeRange(2, 1)];
        NSString* b = [colorString substringWithRange:NSMakeRange(3, 1)];
        colorString = [NSString stringWithFormat:@"#%@%@%@%@%@%@", r, r, g, g, b, b];
    }
    
    // #RRGGBB to 0xRRGGBB
    colorString = [colorString stringByReplacingOccurrencesOfString:@"#" withString:@"0x"];
    
    // 0xRRGGBB to 0xAARRGGBB
    if ([colorString hasPrefix:@"0x"] && [colorString length] == 8) {
        colorString = [@"0xFF" stringByAppendingString:[colorString substringFromIndex:2]];
    }
    
    // 0xAARRGGBB to int
    unsigned colorValue = 0;
    NSScanner *scanner = [NSScanner scannerWithString:colorString];
    if (![scanner scanHexInt:&colorValue]) {
        return nil;
    }
    
    // int to UIColor
    return [UIColor colorWithRed:((float)((colorValue & 0x00FF0000) >> 16))/255.0
                           green:((float)((colorValue & 0x0000FF00) >>  8))/255.0
                            blue:((float)((colorValue & 0x000000FF) >>  0))/255.0
                           alpha:((float)((colorValue & 0xFF000000) >> 24))/255.0];
}

- (NSArray*)parseInterfaceOrientations:(NSArray*)orientations
{
    NSMutableArray* result = [[NSMutableArray alloc] init];

    if (orientations != nil) {
        NSEnumerator* enumerator = [orientations objectEnumerator];
        NSString* orientationString;

        while (orientationString = [enumerator nextObject]) {
            if ([orientationString isEqualToString:@"UIInterfaceOrientationPortrait"]) {
                [result addObject:[NSNumber numberWithInt:UIInterfaceOrientationPortrait]];
            } else if ([orientationString isEqualToString:@"UIInterfaceOrientationPortraitUpsideDown"]) {
                [result addObject:[NSNumber numberWithInt:UIInterfaceOrientationPortraitUpsideDown]];
            } else if ([orientationString isEqualToString:@"UIInterfaceOrientationLandscapeLeft"]) {
                [result addObject:[NSNumber numberWithInt:UIInterfaceOrientationLandscapeLeft]];
            } else if ([orientationString isEqualToString:@"UIInterfaceOrientationLandscapeRight"]) {
                [result addObject:[NSNumber numberWithInt:UIInterfaceOrientationLandscapeRight]];
            }
        }
    }

    // default
    if ([result count] == 0) {
        [result addObject:[NSNumber numberWithInt:UIInterfaceOrientationPortrait]];
    }

    return result;
}

- (BOOL)shouldAutorotate
{
    return YES;
}

// CB-12098
#if __IPHONE_OS_VERSION_MAX_ALLOWED < 90000  
- (NSUInteger)supportedInterfaceOrientations  
#else  
- (UIInterfaceOrientationMask)supportedInterfaceOrientations
#endif
{
    NSUInteger ret = 0;

    if ([self supportsOrientation:UIInterfaceOrientationPortrait]) {
        ret = ret | (1 << UIInterfaceOrientationPortrait);
    }
    if ([self supportsOrientation:UIInterfaceOrientationPortraitUpsideDown]) {
        ret = ret | (1 << UIInterfaceOrientationPortraitUpsideDown);
    }
    if ([self supportsOrientation:UIInterfaceOrientationLandscapeRight]) {
        ret = ret | (1 << UIInterfaceOrientationLandscapeRight);
    }
    if ([self supportsOrientation:UIInterfaceOrientationLandscapeLeft]) {
        ret = ret | (1 << UIInterfaceOrientationLandscapeLeft);
    }

    return ret;
}

- (BOOL)supportsOrientation:(UIInterfaceOrientation)orientation
{
    return [self.supportedOrientations containsObject:[NSNumber numberWithInt:orientation]];
}

- (UIView*)newCordovaViewWithFrame:(CGRect)bounds
{
    NSString* defaultWebViewEngineClass = [self.settings cordovaSettingForKey:@"CordovaDefaultWebViewEngine"];
    NSString* webViewEngineClass = [self.settings cordovaSettingForKey:@"CordovaWebViewEngine"];

    if (!defaultWebViewEngineClass) {
        defaultWebViewEngineClass = @"CDVUIWebViewEngine";
    }
    if (!webViewEngineClass) {
        webViewEngineClass = defaultWebViewEngineClass;
    }

    // Find webViewEngine
    if (NSClassFromString(webViewEngineClass)) {
        self.webViewEngine = [[NSClassFromString(webViewEngineClass) alloc] initWithFrame:bounds];
        // if a webView engine returns nil (not supported by the current iOS version) or doesn't conform to the protocol, or can't load the request, we use UIWebView
        if (!self.webViewEngine || ![self.webViewEngine conformsToProtocol:@protocol(CDVWebViewEngineProtocol)] || ![self.webViewEngine canLoadRequest:[NSURLRequest requestWithURL:self.appUrl]]) {
            self.webViewEngine = [[NSClassFromString(defaultWebViewEngineClass) alloc] initWithFrame:bounds];
        }
    } else {
        self.webViewEngine = [[NSClassFromString(defaultWebViewEngineClass) alloc] initWithFrame:bounds];
    }

    if ([self.webViewEngine isKindOfClass:[CDVPlugin class]]) {
        [self registerPlugin:(CDVPlugin*)self.webViewEngine withClassName:webViewEngineClass];
    }

    return self.webViewEngine.engineWebView;
}

- (NSString*)userAgent
{
    if (_userAgent != nil) {
        return _userAgent;
    }

    NSString* localBaseUserAgent;
    if (self.baseUserAgent != nil) {
        localBaseUserAgent = self.baseUserAgent;
    } else if ([self.settings cordovaSettingForKey:@"OverrideUserAgent"] != nil) {
        localBaseUserAgent = [self.settings cordovaSettingForKey:@"OverrideUserAgent"];
    } else {
        localBaseUserAgent = [CDVUserAgentUtil originalUserAgent];
    }
    NSString* appendUserAgent = [self.settings cordovaSettingForKey:@"AppendUserAgent"];
    if (appendUserAgent) {
        _userAgent = [NSString stringWithFormat:@"%@ %@", localBaseUserAgent, appendUserAgent];
    } else {
        // Use our address as a unique number to append to the User-Agent.
        _userAgent = localBaseUserAgent;
    }
    return _userAgent;
}

- (void)createGapView
{
    CGRect webViewBounds = self.view.bounds;

    webViewBounds.origin = self.view.bounds.origin;

    UIView* view = [self newCordovaViewWithFrame:webViewBounds];

    view.autoresizingMask = (UIViewAutoresizingFlexibleWidth | UIViewAutoresizingFlexibleHeight);
    [self.view addSubview:view];
    [self.view sendSubviewToBack:view];
}

- (void)didReceiveMemoryWarning
{
    // iterate through all the plugin objects, and call hasPendingOperation
    // if at least one has a pending operation, we don't call [super didReceiveMemoryWarning]

    NSEnumerator* enumerator = [self.pluginObjects objectEnumerator];
    CDVPlugin* plugin;

    BOOL doPurge = YES;

    while ((plugin = [enumerator nextObject])) {
        if (plugin.hasPendingOperation) {
            NSLog(@"Plugin '%@' has a pending operation, memory purge is delayed for didReceiveMemoryWarning.", NSStringFromClass([plugin class]));
            doPurge = NO;
        }
    }

    if (doPurge) {
        // Releases the view if it doesn't have a superview.
        [super didReceiveMemoryWarning];
    }

    // Release any cached data, images, etc. that aren't in use.
}

- (void)viewDidUnload
{
    // Release any retained subviews of the main view.
    // e.g. self.myOutlet = nil;

    [CDVUserAgentUtil releaseLock:&_userAgentLockToken];

    [super viewDidUnload];
}

#pragma mark CordovaCommands

- (void)registerPlugin:(CDVPlugin*)plugin withClassName:(NSString*)className
{
    if ([plugin respondsToSelector:@selector(setViewController:)]) {
        [plugin setViewController:self];
    }

    if ([plugin respondsToSelector:@selector(setCommandDelegate:)]) {
        [plugin setCommandDelegate:_commandDelegate];
    }

    [self.pluginObjects setObject:plugin forKey:className];
    [plugin pluginInitialize];
}

- (void)registerPlugin:(CDVPlugin*)plugin withPluginName:(NSString*)pluginName
{
    if ([plugin respondsToSelector:@selector(setViewController:)]) {
        [plugin setViewController:self];
    }

    if ([plugin respondsToSelector:@selector(setCommandDelegate:)]) {
        [plugin setCommandDelegate:_commandDelegate];
    }

    NSString* className = NSStringFromClass([plugin class]);
    [self.pluginObjects setObject:plugin forKey:className];
    [self.pluginsMap setValue:className forKey:[pluginName lowercaseString]];
    [plugin pluginInitialize];
}

/**
 Returns an instance of a CordovaCommand object, based on its name.  If one exists already, it is returned.
 */
- (id)getCommandInstance:(NSString*)pluginName
{
    // first, we try to find the pluginName in the pluginsMap
    // (acts as a whitelist as well) if it does not exist, we return nil
    // NOTE: plugin names are matched as lowercase to avoid problems - however, a
    // possible issue is there can be duplicates possible if you had:
    // "org.apache.cordova.Foo" and "org.apache.cordova.foo" - only the lower-cased entry will match
    NSString* className = [self.pluginsMap objectForKey:[pluginName lowercaseString]];

    if (className == nil) {
        return nil;
    }

    id obj = [self.pluginObjects objectForKey:className];
    if (!obj) {
        obj = [[NSClassFromString(className)alloc] initWithWebViewEngine:_webViewEngine];

        if (obj != nil) {
            [self registerPlugin:obj withClassName:className];
        } else {
            NSLog(@"CDVPlugin class %@ (pluginName: %@) does not exist.", className, pluginName);
        }
    }
    return obj;
}

#pragma mark -

- (NSString*)appURLScheme
{
    NSString* URLScheme = nil;

    NSArray* URLTypes = [[[NSBundle mainBundle] infoDictionary] objectForKey:@"CFBundleURLTypes"];

    if (URLTypes != nil) {
        NSDictionary* dict = [URLTypes objectAtIndex:0];
        if (dict != nil) {
            NSArray* URLSchemes = [dict objectForKey:@"CFBundleURLSchemes"];
            if (URLSchemes != nil) {
                URLScheme = [URLSchemes objectAtIndex:0];
            }
        }
    }

    return URLScheme;
}

#pragma mark -
#pragma mark UIApplicationDelegate impl

/*
 This method lets your application know that it is about to be terminated and purged from memory entirely
 */
- (void)onAppWillTerminate:(NSNotification*)notification
{
    // empty the tmp directory
    NSFileManager* fileMgr = [[NSFileManager alloc] init];
    NSError* __autoreleasing err = nil;

    // clear contents of NSTemporaryDirectory
    NSString* tempDirectoryPath = NSTemporaryDirectory();
    NSDirectoryEnumerator* directoryEnumerator = [fileMgr enumeratorAtPath:tempDirectoryPath];
    NSString* fileName = nil;
    BOOL result;

    while ((fileName = [directoryEnumerator nextObject])) {
        NSString* filePath = [tempDirectoryPath stringByAppendingPathComponent:fileName];
        result = [fileMgr removeItemAtPath:filePath error:&err];
        if (!result && err) {
            NSLog(@"Failed to delete: %@ (error: %@)", filePath, err);
        }
    }
}

/*
 This method is called to let your application know that it is about to move from the active to inactive state.
 You should use this method to pause ongoing tasks, disable timer, ...
 */
- (void)onAppWillResignActive:(NSNotification*)notification
{
    // NSLog(@"%@",@"applicationWillResignActive");
    [self.commandDelegate evalJs:@"cordova.fireDocumentEvent('resign');" scheduledOnRunLoop:NO];
}

/*
 In iOS 4.0 and later, this method is called as part of the transition from the background to the inactive state.
 You can use this method to undo many of the changes you made to your application upon entering the background.
 invariably followed by applicationDidBecomeActive
 */
- (void)onAppWillEnterForeground:(NSNotification*)notification
{
    // NSLog(@"%@",@"applicationWillEnterForeground");
    [self.commandDelegate evalJs:@"cordova.fireDocumentEvent('resume');"];

    /** Clipboard fix **/
    UIPasteboard* pasteboard = [UIPasteboard generalPasteboard];
    NSString* string = pasteboard.string;
    if (string) {
        [pasteboard setValue:string forPasteboardType:@"public.text"];
    }
}

// This method is called to let your application know that it moved from the inactive to active state.
- (void)onAppDidBecomeActive:(NSNotification*)notification
{
    // NSLog(@"%@",@"applicationDidBecomeActive");
    [self.commandDelegate evalJs:@"cordova.fireDocumentEvent('active');"];
}

/*
 In iOS 4.0 and later, this method is called instead of the applicationWillTerminate: method
 when the user quits an application that supports background execution.
 */
- (void)onAppDidEnterBackground:(NSNotification*)notification
{
    // NSLog(@"%@",@"applicationDidEnterBackground");
    [self.commandDelegate evalJs:@"cordova.fireDocumentEvent('pause', null, true);" scheduledOnRunLoop:NO];
}

// ///////////////////////

- (void)dealloc
{
    [[NSNotificationCenter defaultCenter] removeObserver:self];

    [CDVUserAgentUtil releaseLock:&_userAgentLockToken];
    [_commandQueue dispose];
    [[self.pluginObjects allValues] makeObjectsPerformSelector:@selector(dispose)];
}

- (NSInteger*)userAgentLockToken
{
    return &_userAgentLockToken;
}

@end

```

1. 先看构造函数，对于`UIViewController`的各个构造函数，类中都调用同一个方法`__init`，在做一些初始化属性的操作外，也用通知中心对App的生命周期做了监听，这些监听也同时做了能让Web中相应App生命周期的通知。
2. 然后是`viewDidLoad`中对整个网页与原生交互的初始化
   1. 通过`[self loadSettings]`加载配置
      1. 优先寻找`configFile`属性指向的文件名，若无，默认取`config.xml`文件
      2. 初始化`configParser`属性（`NSXMLParser`解析器），用于解析1中的配置文件，解析完后的数据存于`CDVConfigParser`（实现了`NSXMLParserDelegate`协议）实例的属性中，Cordova中的插件就是在这里做映射的。
    2. 通过`[self createGapView]`初始化`webView`，之前从声明中知道`webView`是个`UIView`，这时候会通过`[self newCordovaViewWithFrame:webViewBounds]`创建这个视图，声明成`UIView`的好处就是可以让开发者自定义配置文件中的`CordovaWebViewEngine`和Web View的插件，当前版本则缺省使用`CDVUIWebViewEngine`作为`webViewEngine`，在这个类的内部则是强制转换`webView`为`UIWebView`来使用了。
    3. 通过从1中获取的配置解析出来的变量`startupPluginNames`获得启动就要加载的插件，在此处进行注册初始化（调用插件的`pluginInitialize`方法），过程中会记录每个插件的注册初始化所花的时间，并在最后记录到`pluginObjects`数组属性中，做成强引用，使插件不会被释放。
    4. 通过从1中获取并解析出来的属性中尝试读取Web View的初始页页面，页面存在就加载，不存在就报错误页面，假如链错误页面也没指定，就单纯加载一个显示错误信息的body标签。
3. 后面的生命周期方法，都是简单继承实现并加上了通知中心对生命周期的通知。
4. 析构函数中释放一些内存（含插件，即插件是跟随页面析构做析构的）。

**小结**：`CDVViewController`类及其子类的对象在建立页面的时候，为了让Web端能知道App情况会监听App中的生命周期，也会通过在页面的生命周期发送通知让原生部分可以获得页面生命周期，还会获取配置、注册带有`onload`为`true`参数标签的插件。进一步的话，更可以自定义`webViewEngine`实例的实现。


#### 1. `CDVPlugin`

&emsp;&emsp;在知道了插件的加载后，就需要知道插件里面的实现要做什么。

&emsp;&emsp;`CDVPlugin.h`如下所示：

```ObjC
#import <Foundation/Foundation.h>
#import <UIKit/UIKit.h>
#import "CDVPluginResult.h"
#import "NSMutableArray+QueueAdditions.h"
#import "CDVCommandDelegate.h"
#import "CDVWebViewEngineProtocol.h"

@interface UIView (org_apache_cordova_UIView_Extension)

@property (nonatomic, weak) UIScrollView* scrollView;

@end

// 可用于获取监听页面生命周期的信息
extern NSString* const CDVPageDidLoadNotification;
extern NSString* const CDVPluginHandleOpenURLNotification;
extern NSString* const CDVPluginHandleOpenURLWithAppSourceAndAnnotationNotification;
extern NSString* const CDVPluginResetNotification;
extern NSString* const CDVViewWillAppearNotification;
extern NSString* const CDVViewDidAppearNotification;
extern NSString* const CDVViewWillDisappearNotification;
extern NSString* const CDVViewDidDisappearNotification;
extern NSString* const CDVViewWillLayoutSubviewsNotification;
extern NSString* const CDVViewDidLayoutSubviewsNotification;
extern NSString* const CDVViewWillTransitionToSizeNotification;

/*
 * The local and remote push notification functionality has been removed from the core in cordova-ios 4.x,
 * but these constants have unfortunately have not been removed, but will be removed in 5.x.
 * 
 * To have the same functionality as 3.x, use a third-party plugin or the experimental
 * https://github.com/apache/cordova-plugins/tree/master/notification-rebroadcast
 */
extern NSString* const CDVLocalNotification CDV_DEPRECATED(4.0, "Functionality removed in 4.0, constant will be removed in 5.0");
extern NSString* const CDVRemoteNotification CDV_DEPRECATED(4.0, "Functionality removed in 4.0, constant will be removed in 5.0");
extern NSString* const CDVRemoteNotificationError CDV_DEPRECATED(4.0, "Functionality removed in 4.0, constant will be removed in 5.0");

@interface CDVPlugin : NSObject {}

// 防止循环引用的弱引用
@property (nonatomic, readonly, weak) UIView* webView;
@property (nonatomic, readonly, weak) id <CDVWebViewEngineProtocol> webViewEngine;

@property (nonatomic, weak) UIViewController* viewController;
@property (nonatomic, weak) id <CDVCommandDelegate> commandDelegate;

@property (readonly, assign) BOOL hasPendingOperation;

- (void)pluginInitialize;

- (void)handleOpenURL:(NSNotification*)notification;
- (void)handleOpenURLWithApplicationSourceAndAnnotation:(NSNotification*)notification;
- (void)onAppTerminate;
- (void)onMemoryWarning;
- (void)onReset;
- (void)dispose;

/*
 // see initWithWebView implementation
 - (void) onPause {}
 - (void) onResume {}
 - (void) onOrientationWillChange {}
 - (void) onOrientationDidChange {}
 */

- (id)appDelegate;

@end
```

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
- (void)webViewDidStartLoad:(UIWebView *)webView;
- (void)webViewDidFinishLoad:(UIWebView *)webView;
- (void)webView:(UIWebView *)webView didFailLoadWithError:(NSError *)error;
```

&emsp;&emsp;前端的JavaScript在加载后已经能通过拦截时注入的JavaScript代码进行对交互调用了。