---
layout:     post
title:      WebViewJavascriptBridge学习
author:     慢慢
tags: 		WebViewJavascriptBridge iOS
subtitle:  	
category:  blog
---
<!-- Start Writing Below in Markdown -->

参考：
[WebViewJavascriptBridge源码分析](http://blog.csdn.net/mociml/article/details/47701133)
[WebViewJavascriptBridge 原理分析](http://www.2cto.com/kf/201503/384998.html)

源码地址：https://github.com/marcuswestin/WebViewJavascriptBridge

上面两个原理已经讲的很清楚了，重点讲了JS如何调用Native的代码，即使用了URL拦截的方法，就不再详细分析源码了。大概流程如图：
![这里写图片描述](http://img.blog.csdn.net/20160402190402265)
1 Native端通过registerHandler注册方法，即保存到messageHandlers的Dictionary里
```
- (void)registerHandler:(NSString *)handlerName handler:(WVJBHandler)handler {
    _base.messageHandlers[handlerName] = [handler copy];
}
```
2 js端调用callHandler方法，通过iframe发送网络请求，Native端通过UIViewDelegate的方法拦截，如果和WebViewJavascriptBridge的scheme相同则执行flushMessageQueue，参数相关的信息是通过调用js中的_fetchQueue取到的
3 在flushMessageQueue会判断responseId和callbackId，responseId标识的是Native调用js的方法后，js端执行Native的回调；callbackId表示的是js直接调用Native的代码

在以前的版本中，是WebView的Delegate方法中通过判断numRequestsLoading计数来注入injectJavascriptFile的代码，但是计数可能会存在问题，导致没有调用Bridge相关的代码
```c
- (void)webViewDidFinishLoad:(UIWebView *)webView {
    if (webView != _webView) { return; }
    
    _numRequestsLoading--;
    
    if (_numRequestsLoading == 0 && ![[webView stringByEvaluatingJavaScriptFromString:[_base webViewJavascriptCheckCommand]] isEqualToString:@"true"]) {
        [_base injectJavascriptFile:YES];
    }
    [_base dispatchStartUpMessageQueue];
    
    
    __strong WVJB_WEBVIEW_DELEGATE_TYPE* strongDelegate = _webViewDelegate;
    if (strongDelegate && [strongDelegate respondsToSelector:@selector(webViewDidFinishLoad:)]) {
        [strongDelegate webViewDidFinishLoad:webView];
    }
}
```
新版本里增加了新的scheme，不需要Native通过计数的方案来实现了，所以需要在js里增加

```
function setupWebViewJavascriptBridge(callback) {
    if (window.WebViewJavascriptBridge) { return callback(WebViewJavascriptBridge); }
    if (window.WVJBCallbacks) { return window.WVJBCallbacks.push(callback); }
    window.WVJBCallbacks = [callback];
    var WVJBIframe = document.createElement('iframe');
    WVJBIframe.style.display = 'none';
    WVJBIframe.src = 'wvjbscheme://__BRIDGE_LOADED__';
    document.documentElement.appendChild(WVJBIframe);
    setTimeout(function() { document.documentElement.removeChild(WVJBIframe) }, 0)
}
```
同时在UIViewViewDelegate的
```
- (BOOL)webView:(UIWebView *)webView shouldStartLoadWithRequest:(NSURLRequest *)request navigationType:(UIWebViewNavigationType)navigationType
```
方法中增加了
```c
if ([_base isBridgeLoadedURL:url]) {
    [_base injectJavascriptFile];
} 
```

这些都是通过URL拦截来实现的，但是苹果从iOS7开始引入了JavaScriptCore库，比如JSPatch就是这样实现的

