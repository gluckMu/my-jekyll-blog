---
layout: post
title:  "React Native通信原理详解"
date:   2016-12-02
desc: "React Native通信原理详解"
keywords: "React Native"
categories: [iOS]
tags: [iOS, React Native]
---

>基于2016-11-17的[React Native](https://github.com/facebook/react-native)源码

## 简介

* What: React Native并不是开发Hybrid App，它是为开发人员提供了一套使用JavaScript开发Native App的开发框架。

* Why: React Native通过将不同平台的界面实现‘封装’起来，对开发人员提供一套JavaScript调用，允许开发人员像使用React开发Web页面一样来开发Native App，这就是`Learn once，write anywhere`，其本质思想上同`Write once, run anywhere`是一致的，都是通过屏蔽底层的多样性，来提高开发人员的效率。

* How: 不同于JSPatch利用Objective-C的Runtime机制动态创建新的类型或方法，React Native是预先使用Native实现了一堆可用的组件，然后通过JavaScript去调用这些组件，在这一点上，React Native有点类似于Hybrid App里面JavaScript去调用Native方法的做法。这里面的重点就是JavaScript如何与Native通信，实现相互调用。

## Native与JavaScript通信

&emsp;&emsp;在iOS中，Native与JavaScript通信有两种方法，一种是通过UIWebView（iOS 8.0以后又有了WKWebView），一种是通过JavaScriptCore，后者是iOS 7.0以后才提供的，不过根据苹果官方的统计数据，iOS 9.0以上占有率92%，所以可以忽略系统要求。

### 1. UIWebView

&emsp;&emsp;最早开始开发Hybrid App时，JavaScript就需要与Native通信，那时候出现的一些开源项目，例如EasyJSWebView、Cordova，它们就是利用UIWebView来实现JavaScript与Native的通信。

+ Native执行JavaScript：利用UIWebView提供的`stringByEvaluatingJavaScriptFromString:`接口，可以直接执行JavaScript代码，并且返回执行结果。
+ JavaScript调用Native：JavaScript没有可以直接调用Native的方法，只能通过在Native端捕获UIWebView的请求URL来间接实现调用，具体做法就是通过在Web端使用隐藏iframe来实现URL跳转，这个URL是的地址格式是预先与Native端定义好的，例如：XXXBridge://methodName?{params}，这个格式是不定的，只要Native能够解析即可。

### 2. JavaScriptCore

&emsp;&emsp;JavaScriptCore库把WebKit的JavaScipt引擎用Objective-C封装，提供简单、快速、安全的方式接入JavaScript，这就使得Native与JavaScript的互相调用更加方便。

* Native执行JavaScript：利用JavaScriptCore提供的JSContext，调用其`evaluateScript:`方法。

```javascript
  [jsContext evaluateScript:@"js代码"];
```

或者是通过JSContext先获取JavaScript方法名，然后通过JSValue的`callWithArguments:`方法，但是这个只能调用方法，不能执行JavaScript片段。

```javascript
  JSValue *method = jsContext[@"方法名"];
  [method callWithArguments:@[@"参数"]];
```

* JavaScript调用Native：利用JSContext的`setObject:forKeyedSubscript:`方法，可以将Native对象或者Block塞进JavaScript运行的上下文中，然后Web端通过这个上下文就可以获取Native对象，然后直接执行即可。这里值得有两点需要注意，第一是怎么获取Web端运行时的JSContext，这个通过KVC可以从UIWebView中获取，但是在app审核可能被苹果拒绝，因为这是一种hack方法，依赖了UIWebView的内部对象，不过还是有牛人发现可以通过WebKit的内部机制来实现，这个比较复杂，具体不再此阐述，详见[这里](https://github.com/TomSwift/UIWebView-TS_JavaScriptContext)，对应的中文版看[这里](http://hao.jser.com/archive/9062/)。

```objc
  JSContext *jsContext = [webView valueForKeyPath:@"documentView.webView.mainFrame.javaScriptContext"];
```

&emsp;&emsp;第二，如果是set对象的话，这个对象暴露给JavaScript的方法要封装成协议，并且这个协议需要遵从JSExport协议，最后这个对象只要实现这个自定义的协议即可。

&emsp;&emsp;JavaScriptCore相对于UIWebView来说，更加的方便，因为不用再通过隐式的URL请求去调用Native，直接就可以调用对象方法，而且更加安全，在执行js时，可以添加exceptionHandler，所以前者更加吸引人，但是现在苹果官方并没有开放获取UIWebView的JSContext的方法，所以导致在Hybrid App的开发中，可能需要些黑科技，但是React Native并不是开发Hybrid App，它实际上就是去执行JavaScript代码，所以使用JavaScriptCore是最优的。当然，React Native也提供了使用自定义JavaScript执行引擎的功能。

## 通信实现

### 1. JavaScript调用Native

&emsp;&emsp;JavaScript调用Native方法有几个难点需要解决

* **JavaScript如何知道有哪些Native对象和方法可以调用；**
* **JavaScript知道了Native对象和方法后，怎么去通知Native，然后调用起来对应的方法。**

&emsp;&emsp;React Native针对以上问题的解决方案，就是实现了一个模块配置表，然后在Native和JavaScript端都实现一个Bridge，在这个Bridge中各自保存了一份相同的模块配置表，这个配置表中包含了所有的模块列表信息，包括模块名称，模块ID，模块提供的方法列表信息（方法列表中又包括方法名称，方法ID，方法类型），模块提供的常量。注意，这里的这个模块配置表只是逻辑上的，真正实现的时候，在Native端，这个模块配置表比较简单，就是个存储模块信息对象的数组；JavaScript端会根据这个配置表，生成出一个NativeModules对象，这里面包括了所有的模块对象，并且根据模块ID、方法ID、方法类型去生成一个真正的js函数，然后赋给模块对象，大致如下：

```javascript
  "NativeModules": {
     "RCTAlertManager": {
       "alertWithArgs": func(...args: Array<any>) {
           // 这里的function是根据method的type来区分的
           // type有sync，promise，其他
       }
       ...other methods
       ...other export constants
     },
     ...
  }
```

&emsp;&emsp;根据方法类型生成方法的代码如下：

```javascript
  NativeModules.js
	
  function genMethod(moduleID: number, methodID: number, type: MethodType) {
    let fn = null;
    if (type === 'promise') {
      fn = function(...args: Array < any > ) {
        return new Promise((resolve, reject) => {
          BatchedBridge.enqueueNativeCall(moduleID, methodID, args, (data) => resolve(data), (errorData) => reject(createErrorFromErrorData(errorData)));
        });
      };
    } else if (type === 'sync') {
      fn = function(...args: Array < any > ) {
        return global.nativeCallSyncHook(moduleID, methodID, args);
      };
    } else {
      fn = function(...args: Array < any > ) {
        const lastArg = args.length > 0 ? args[args.length - 1] : null;
        const secondLastArg = args.length > 1 ? args[args.length - 2] : null;
        const hasSuccessCallback = typeof lastArg === 'function';
        const hasErrorCallback = typeof secondLastArg === 'function';
        hasErrorCallback && invariant(hasSuccessCallback,
          'Cannot have a non-function arg after a function arg.'
        );
        const onSuccess = hasSuccessCallback ? lastArg : null;
        const onFail = hasErrorCallback ? secondLastArg : null;
        const callbackCount = hasSuccessCallback + hasErrorCallback;
        args = args.slice(0, args.length - callbackCount);
        BatchedBridge.enqueueNativeCall(moduleID, methodID, args, onFail, onSuccess);
      };
    }
    fn.type = type;
    return fn;
  }
```

&emsp;&emsp;针对第一个问题，JavaScript有了这个配置表后，就自然知道Native提供了哪些模块和方法可以调用，但是这里所谓的知道，仅仅是在NativeModules中注册了而已，开发人员真正在使用的时候，还是不知道有什么模块和方法，因此，React Native在JavaScript端对NativeModules中的对象又封装了一层，衍生出了对应的模块类，方便开发人员使用，需要注意的是，React Native封装的都是它在Native端实现了的模块，所以当开发人员自己扩展Native的模块时，最好还在JavaScript端也实现对应的类，方便开发。

&emsp;&emsp;针对第二个问题，JavaScript在知道所有Native提供的模块信息后，在调用Native方法时，就通过JavaScript Bridge去“通知”Native Bridge，提供相应的模块ID、方法ID、入参（包括回调），Native Bridge在“接收”调用信息后，根据模块ID、方法ID就能从自己的模块配置表中找到对应的实现方法，然后将入参传递过去，执行方法即可，实现方法的调用。

<div align='center'>
<img src="{{site.baseurl}}{{ site.img_path }}/ReactNative_Bridge/JSInvokeNative.png" alt="JavaScritp Invoke Native"/>
</div>

&emsp;&emsp;这里又引入另一个问题，就是**JavaScript Bridge如何去“通知”Native Bridge现在有方法调用?**

&emsp;&emsp;这一点在前面JavaScriptCore介绍过，可以通过在Native端设置JSContext的方法，然后在JavaScript里面就可以直接调用，React Native里面也使用到这一点，但是只是作为其中一种备用方法。React Native在JavaScript Bridge维护了一个队列，这个队列里面放置所有要调用的Native方法（ModuleID、MethodID、Params），除非方法类型是sync的，这种类型的方法会直接去调用JSContext里面的Native方法`nativeCallSyncHook`。当Native通过JavaScript Bridge来调用JavaScript方法或者回调时，JavaScript Bridge会在方法执行完毕时，顺带将这个队列里的所有调用一同返回给Native，Native收到返回就会去解析，然后调用相应的方法，实现了方法调用的“通知”。核心代码如下：

```javascript
  MessageQueue.js
	
  enqueueNativeCall(moduleID: number, methodID: number, params: Array < any > , onFail: ? Function, onSucc : ? Function) {
    if (onFail || onSucc) {
      onFail && params.push(this._callbackID);
      this._callbacks[this._callbackID++] = onFail;
      onSucc && params.push(this._callbackID);
      this._callbacks[this._callbackID++] = onSucc;
    }
    this._callID++;

    this._queue[MODULE_IDS].push(moduleID);
    this._queue[METHOD_IDS].push(methodID);
    this._queue[PARAMS].push(params);

    const now = new Date().getTime();
    if (global.nativeFlushQueueImmediate &&
      now - this._lastFlush >= MIN_TIME_BETWEEN_FLUSHES_MS) {
      global.nativeFlushQueueImmediate(this._queue);
      this._queue = [
        [],
        [],
        [], this._callID
      ];
      this._lastFlush = now;
    }
  }
```

&emsp;&emsp;**那么React Native为什么这样设计呢，它这样能保证JavaScript能够实时调用起来Native方法么？**

&emsp;&emsp;根据代码来看，React Native在所有Native调用JavaScript的函数入口处（实际上就三个，callFunctionReturnFlushedQueue、callFunctionReturnResultAndFlushedQueue和invokeCallbackAndReturnFlushedQueue），都记录了一个当前时间作为flushTime，然后同步执行相应的JavaScript方法，执行完毕后，把队列中的所有Native调用当做结果返回，这样只要所有的JavaScript逻辑都是由Native去触发的，那么理论上JavaScript里面的Native方法调用，都会在结果中直接被返回，然后执行。事实上，`React Native中所有的JavaScript方法也的确都是由Native事件触发的，这个事件可能是用户操作，或者定时器等系统事件`，所以React Native采用这种方法，可以保证Native方法被JavaScript调用起来，但是由于JavaScript的执行是存在性能消耗的，如果一个JavaScript方法执行超过一定时间，那么前面调用的Native方法可能在队列里面一直排队等待，所以React Native在执行Native方法时，会去判断当前时间距离之前记录的flushTime是否超过了一定时间差（现在是5ms），没超过就塞进队列，超过了就直接调用JSContext里Native方法`nativeFlushQueueImmediate`，告诉Native你要去取我队列了，在这里我们就明白了JavaScript的执行是有性能损耗的，这个临界值就是5ms。同时，采用队列的方法，就可以让一次JavaScript方法产生的多次Native调用，一次性返回给Native去执行，减少了Naitve和JavaScript上下文的切换。这样来看，React Native的设计是很巧妙的，如果每一次Native调用，都去直接调用JSContext，就会产生频繁的Native和JavaScript环境切换，虽然单次的交互可能更及时，但是从整体上看，并不一定保证调用的实时性；还有一种方法，就是Native启用定时器，去定时处理队列，但是这样做就会导致在这个定时器的间隔怎么确定，空闲时会导致冗余的调用，在实时性上可能又会有所缺失。

### 2. Native调用JavaScript

&emsp;&emsp;React Native在实现Native调用JavaScript方法时，就简单很多，没有了模块配置表，虽然JavaScript在实现的时候还是有模块表的，但是并没有为Native去提供这样一个模块表。为什么JavaScrip缓存了Native端的模块配置表，而Native端没有缓存JavaScript模块表，React Native采用这一做法的考虑，我认为主要原因是二者职责的不同导致，在React Native中，JavaScript是负责页面组织和逻辑处理，Native是负责根据规则去展示和接收反馈并转发，在这种设计下，JavaScript处于主导地位，有点类似于分层设计，这样来看，理论上JavaScript完全不需要提供方法供Native调用，应当只有回调才对，但是由于各种限制，例如APP启动时的通知，点击事件等，导致JavaScript必须提供少量方法，供Native去调用来通知自己某些事件的发生，但是这种方法应该是尽量少并且不易扩展的，这就导致JavaScript提供的方法在Native端就没有必要再去缓存配置表了，直接调用即可，而处于“底层”的Native，则必须提供很多方法，而且可能还随时需要扩展，来满足JavaScript的开发需求，因此实现配置表就省去了扩展时带来的麻烦。

&emsp;&emsp;Native在调用JavaScript方法和回调原理上是一致的，都是通过调用“代理方法”去实现具体方法的调用，但是也还有些许不同，一个是入口不同，调用JavaScript方法的入口函数是callFunctionReturnFlushedQueue和callFunctionReturnResultAndFlushedQueue两个方法，这两个方法的唯一区别就是后者会返回JavaScript方法执行结果，执行JavaScript回调的入口函数是invokeCallbackAndReturnFlushedQueue，还有一个是调用方法时，“代理方法”的入参是moduleName、methodName、args，而调用回调时，传递的是callbackID、args，这个callbackID是JavaScript在调用Native方法时，将回调函数注册在JavaScript Bridge中，然后将callbackID传递给Native，最后Native调用回调时的凭据就是这个callbackID。

&emsp;&emsp;根据前面的介绍，JavaScriptCore提供了直接调用JavaScript方法的方法，如JSContext的evaluateScript方法，但是为了更好的调用速度（github上[提交记录](https://github.com/facebook/react-native/pull/1037/files)），React Native采用了JavaScriptCore里面的JSObjectRef下面的方法来执行JavaScript方法。核心代码如下：

```objc
  Native RCTJSCExecutor.m
	
  -(void) _executeJSCall: (NSString * ) method
  arguments: (NSArray * ) arguments
  unwrapResult: (BOOL) unwrapResult
  callback: (RCTJavaScriptCallback) onComplete {
    __weak RCTJSCExecutor * weakSelf = self;
    [self executeBlockOnJavaScriptQueue: ^ {
      RCTJSCExecutor * strongSelf = weakSelf;
      if (!strongSelf || !strongSelf.isValid) {
        return;
      }
      RCTJSCWrapper * jscWrapper = strongSelf - > _jscWrapper;
      JSContext * context = strongSelf - > _context.context;
      JSGlobalContextRef contextJSRef = context.JSGlobalContextRef;
      // get the BatchedBridge object
      JSValueRef errorJSRef = NULL;
      JSValueRef batchedBridgeRef = strongSelf - > _batchedBridgeRef;
      if (!batchedBridgeRef) {
        JSStringRef moduleNameJSStringRef = jscWrapper - > JSStringCreateWithUTF8CString("__fbBatchedBridge");
        JSObjectRef globalObjectJSRef = jscWrapper - > JSContextGetGlobalObject(contextJSRef);
        batchedBridgeRef = jscWrapper - > JSObjectGetProperty(contextJSRef, globalObjectJSRef, moduleNameJSStringRef, & errorJSRef);
        jscWrapper - > JSStringRelease(moduleNameJSStringRef);
        strongSelf - > _batchedBridgeRef = batchedBridgeRef;
      }
      NSError * error;
      JSValueRef resultJSRef = NULL;
      if (batchedBridgeRef != NULL && errorJSRef == NULL && !jscWrapper - > JSValueIsUndefined(contextJSRef, batchedBridgeRef)) {
        // get method
        JSStringRef methodNameJSStringRef = jscWrapper - > JSStringCreateWithCFString((__bridge CFStringRef) method);
        JSValueRef methodJSRef = jscWrapper - > JSObjectGetProperty(contextJSRef, (JSObjectRef) batchedBridgeRef, methodNameJSStringRef, & errorJSRef);
        jscWrapper - > JSStringRelease(methodNameJSStringRef);
        if (methodJSRef != NULL && errorJSRef == NULL && !jscWrapper - > JSValueIsUndefined(contextJSRef, methodJSRef)) {
          JSValueRef jsArgs[arguments.count];
          for (NSUInteger i = 0; i < arguments.count; i++) {
            jsArgs[i] = [jscWrapper - > JSValue valueWithObject: arguments[i] inContext: context].JSValueRef;
          }
          resultJSRef = jscWrapper - > JSObjectCallAsFunction(contextJSRef, (JSObjectRef) methodJSRef, (JSObjectRef) batchedBridgeRef, arguments.count, jsArgs, & errorJSRef);
        } else {
          if (!errorJSRef && jscWrapper - > JSValueIsUndefined(contextJSRef, methodJSRef)) {
            error = RCTErrorWithMessage([NSString stringWithFormat: @ "Unable to execute JS call: method %@ is undefined", method]);
          }
        }
      } else {
        if (!errorJSRef && jscWrapper - > JSValueIsUndefined(contextJSRef, batchedBridgeRef)) {
          error = RCTErrorWithMessage(@ "Unable to execute JS call: __fbBatchedBridge is undefined");
        }
      }
      id objcValue;
      if (errorJSRef || error) {
        if (!error) {
          error = RCTNSErrorFromJSError([jscWrapper - > JSValue valueWithJSValueRef: errorJSRef inContext: context]);
        }
      } else {
        // We often return `null` from JS when there is nothing for native side. [JSValue toValue]
        // returns [NSNull null] in this case, which we don't want.
        if (!jscWrapper - > JSValueIsNull(contextJSRef, resultJSRef)) {
          JSValue * result = [jscWrapper - > JSValue valueWithJSValueRef: resultJSRef inContext: context];
          objcValue = unwrapResult ? [result toObject] : result;
        }
      }
      onComplete(objcValue, error);
    }];
  }
```

## 模块配置表

&emsp;&emsp;React Native中JavaScript调用Native方法的核心便是模块配置表，实际上Native端和JavaScript端提供的方法都有个模块配置表，但是其中JavaScript端的模块配置表只会存在JavaScript Bridge中，不会传递给Native端，不过两者的思想是一致的，都是模块自行去Bridge中注册自己，只不过由于语言特性的区别，导致在实现上会有差异，在这里我们着重讨论的是Native端的这个模块配置表是怎么生成的。模块配置表的生成，主要包括两个部分，一是模块的注册，二是模块具体对外开放的方法的注册。

### 1. 模块注册

&emsp;&emsp;模块注册过程十分简单，就是所有对外开放的模块，在初始化的时候，自行去Native Bridge那里注册下自己，告诉它我需要对JavaScript开放，这就完成了模块列表的获取，这里React Native利用了NSObject的load方法去实现注册。

### 2. 方法注册

&emsp;&emsp;方法注册相对来说要复杂些，React Native根据每一个要对外开放的方法去生成一个对应的“声明方法”，这个声明方法是以__rct_export__开头的，方便后续识别，该方法会返回一个包括两个元素的数组，第一个元素是对外开放给JavaScript调用时的方法名称，第二个元素是方法在Native端的真实声明，后续在获取模块的对外开放的方法列表时，只需要遍历模块类的所有方法，只要是以__rct_export__开头的，就直接调用它，获取真实的对外开放方法，然后添加进列表即可。

&emsp;&emsp;经过以上两步，Native端就可以生成出模块配置表，剩下的就是要传给JavaScript即可。在传递过程中，React Native采用了“延迟加载”，在绝大多数情况下（也就是不修改默认的JavaScriptExecutor），Native利用JavaScriptCore向JavaScript端注入模块列表时，只会将模块的名称告诉JavaScript，JavaScript Bridge会根据这个列表来生成相应的模块对象，不过这时候这个对象是空的，当真正需要使用这个模块对象时，就会去根据模块名称，调用JSContext里Native方法`nativeRequireModuleConfig`，返回模块的所有配置信息，实现了模块在使用时才加载。这种“延迟加载”在React Native中被广泛使用，并不仅仅是这一处。

## 总结

&emsp;&emsp;React Native中Native与JavaScript通信的过程大致就这些，但是其它还有很多的细节，例如队列的管理、JavaScriptExecutor的处理、性能的检测等，这些每一项都相当复杂，可以自行去查看源码。
