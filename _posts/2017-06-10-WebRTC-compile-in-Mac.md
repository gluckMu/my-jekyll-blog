---
layout: post
title:  "Mac编译WebRTC"
date:   2017-06-10
desc: "Mac编译WebRTC"
keywords: "WebRTC"
categories: [iOS]
tags: [iOS, WebRTC]
---

# Mac编译WebRTC

> [官方文档](https://webrtc.org/native-code/ios/)

### 安装depot_tools

``` 
git clone https://chromium.googlesource.com/chromium/tools/depot_tools.git
export PATH=`pwd`/depot_tools:"$PATH"
```

### 下载代码

```
fetch --nohooks webrtc_ios
gclient sync
```
由于国内网络环境影响，下载时请挂代理，由于下载代码时不仅涉及到git，还有python等，所以最好使用vpn，避免出现各种问题。
	
在gclient sync时，会根据DEPS文件中的hooks配置进行相应的操作，可以根据相关平台，删除相应的操作。
	
使用gclient help查看更多命令。
	
### 生成项目

生成项目时需要使用[GN](https://chromium.googlesource.com/chromium/src/+/master/tools/gn/README.md)，有关参数可以去官网查看。
	
```
gn gen out/xxx(指定生成地址) --args='target_os="ios" target_cpu="arm64" clang_use_chrome_plugins=false is_debug=false'
```
	
上述命令生成的是ninja工程，如果想要生成xcode工程，在命令后加上`-ide="xcode"`即可，但是在实际编译时还是使用的ninja编译。
	
### 编译

编译时使用了[Ninja](https://ninja-build.org/)
	
```
ninja -C out/xxx objc_framework
```
	
objc_framework是指定编译WebRTC开发包的target，想要查看有哪些target，使用命令`gn ls out/xxx`。



