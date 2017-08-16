---
layout: post
title:  "iOS 11 — WKWebView"
date:   2017-08-15
desc: "iOS 11 WKWebView新特性"
keywords: "WKWebView"
categories: [iOS]
tags: [iOS]
---

## cookie

增加对cookie的增删改查操作，并且能够接收cookie变化的通知。

```
let cookie = HTTPCookie(properties: [
    .domain : "baidu.com",
    .path : "/",
    .secure : true,
    .name : "login",
    .value: "test"])
let websiteDataStore = WKWebsiteDataStore.nonPersistent()
websiteDataStore.httpCookieStore.setCookie(cookie!) {
    let configuration = WKWebViewConfiguration()
    configuration.websiteDataStore = websiteDataStore
    
    self.webview = WKWebView(frame: self.view.frame, configuration: configuration)
    self.webview.load(URLRequest(url: URL(string: "https://www.baidu.com")!))
}

```

## content filter

屏蔽自定义过滤规则的资源内容，至于过滤的规则，参见[WKWebView内容过滤规则详解](http://www.jianshu.com/p/8af24e9dc82e)

```
// json规则
let jsonString = """
    [{
        "trigger": {
            "url-filter": ".*"
        },
        "action": {
            "type": "make-https"
        }
    }]
    """

// 	编译规则，只需要编译一次            
WKContentRuleListStore.default().compileContentRuleList(forIdentifier: "demoRuleList", encodedContentRuleList: jsonString, completionHandler: { (list, error) in
    guard let contentRuleList = list else { return }
    let configuration = WKWebViewConfiguration()
    configuration.userContentController.add(contentRuleList)
    self.webview = WKWebView(frame: self.view.frame, configuration: configuration)
    self.webview.load(URLRequest(url: URL(string: "https://www.baidu.com")!))
})

// 后续可以查找规则使用
WKContentRuleListStore.default().lookUpContentRuleList(forIdentifier: "demoRuleList") { (list, error) in
    guard let contentRuleList = list else { return }
    let configuration = WKWebViewConfiguration()
    configuration.userContentController.add(contentRuleList)
    self.webview = WKWebView(frame: self.view.frame, configuration: configuration)
    self.webview.load(URLRequest(url: URL(string: "https://www.baidu.com")!))
}
```

## custom schema handler

允许对页面上自定义的schema进行处理，但是对于像http、https、ftp这种是不能处理的。

```
// 实现自定义的schema处理类
class CustomWKURLSchemeHandler: NSObject, WKURLSchemeHandler {
    func webView(_ webView: WKWebView, start urlSchemeTask: WKURLSchemeTask) {
        // 处理自定义schema完成时 先构造好response
        // 然后调用相应方法 通知处理完成
        urlSchemeTask.didReceive(response)
        urlSchemeTask.didReceive(data)
        urlSchemeTask.didFinish()
    }
    
    func webView(_ webView: WKWebView, stop urlSchemeTask: WKURLSchemeTask) {
        // task结束时调用
    }
}

// 在configuration中注册schema
let configuration = WKWebViewConfiguration()
let handler = CustomWKURLSchemeHandler()
config.setURLSchemeHandler(handler, forURLScheme: "custom-schema")
self.webview = WKWebView(frame: self.view.frame, configuration: config)
self.webview.load(URLRequest(url: URL(string: "https://www.baidu.com")!))
```

## 缓存

iOS 11里，WKWebView还是与URLProtocol不兼容，也不走NSURLCache，所以之前的利用URLProtocol实现的缓存策略，无法在WKWebView上使用，而新增的schema handler由于禁止注册http、https这类系统保留schema，所以也无法使用，暂时来看，想在WKWebView上做缓存还是没什么好办法。

### WKWebView与URLProtocol兼容解决办法

利用WebKit里面的WKBrowsingContextController私有API可以使WKWebView兼容URLProtocol，但是由于使用了私有API，所以上架AppStore可能就存在问题。

```
Class cls = NSClassFromString(@"WKBrowsingContextController");
SEL sel = NSSelectorFromString(@"registerSchemeForCustomProtocol:");
if ([(id)cls respondsToSelector:sel]) {
#pragma clang diagnostic push
#pragma clang diagnostic ignored "-Warc-performSelector-leaks"
    NSLog(@"Hack method");
    [(id)cls performSelector:sel withObject:@"http"];
#pragma clang diagnostic pop
}
```

## 参考

* [WWDC 2017 Session 220 Customized Loading in WKWebView](https://developer.apple.com/videos/play/wwdc2017/220/)