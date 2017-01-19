---
layout: post
title:  "iOS开发基础——日志"
date:   2017-01-12
desc: "iOS开发基础——日志"
keywords: "iOS开发"
categories: [iOS]
tags: [iOS]
---

## 前言

日志是每个开发人员都会与之打交道的，是调试程序的一种有效办法，并且在出现问题时，日志就是程序员的福音，所以，日志的重要性不言而喻。在这种情况下，一个应用的日志设计就非常重要。

## 日志类型

iOS开发中，日志打印有以下几种方法

### ASL（Apple System Logging）

ASL是管理和存储系统日志信息的守护进程，它有一套API去操作系统日志，包括创建、查询等，ASL记录的日志分为不同的级别，包括

```
ASL_LEVEL_EMERG
ASL_LEVEL_ALERT
ASL_LEVEL_CRIT
ASL_LEVEL_ERR
ASL_LEVEL_WARNING
ASL_LEVEL_NOTICE
ASL_LEVEL_INFO
ASL_LEVEL_DEBUG
```

其中，只有ASL_LEVEL_NOTICE级别以上的日志，才会被记入系统日志，然后从Console.app上能够查看。

如果想使用ASL API的话，需要先引入<asl.h>，其中asl_log函数能够写入日志，asl_add_log_file(NULL, STDERR_FILENO)能够将日志额外输出到stderr，如果是联机调试的话，就是展示在XCode的console中。

**注意：ASL在iOS 10以后已经被新的日志工具（Unified Logging: os_log）替代了，ASL提供的API都将不推荐使用，部分功能失效**

### os_log

iOS 10以后提供的新的日志操作工具，待补充。

### printf

printf是个C语言里面的标准方法，它接受的入参是C语言的String常量(`const char *format`)，它会输出到标准的stdout里面，效率高，但是它在真机上输出的日志不能通过Console.app查看，因为printf没有调用ASL。

### NSLog

NSLog是Foundation框架下提供的一个日志方法，它接受的入参是Objective-C的NSString，苹果官方的解释是`Logs an error message to the Apple System Log facility`，这里面的关键字有两个，一个是error message，一个是Apple System Log。前者意思是NSLog输出的是stderr，这一点同printf是不同的，后者则说明NSLog还会向ASL写入日志。

NSLog会将日志写进ASL，ASL会自动添加时间戳、进程名称，同时向stderr输出。因此，NSLog输出的日志，可以通过Console.app进行查看，但另一方面，也导致其性能的下降。

### 总结

对于一般应用来说，使用NSLog已经能满足需求，但是注意NSLog输出的日志实际上是一种error级别的日志，它会存在设备的系统日志中，因此在应用发布时应当屏蔽，同时NSLog输出的日志比较粗粒度，导致开发人员使用时可能就会比较随意；如果有更精细的日志需求，那么ASL就比较合适，它提供了8个级别的日志等级；printf适用于C，对于Objective-C来说，它在性能上很好，但是开发人员在开发时更关注的是日志提供的信息，而在发布时，日志通常是去掉的，所以性能这一方面可以忽略，另一方面，printf在真机上输出的日志，无法通过Console.app查看到，如果要查看需要额外的处理，并不如NSLog来的方便，所以printf在大多场景下并不适用，当然如果对性能有要求的话，printf是个不错的选择。

## 应用日志实现

### 需求

* 开发时需要有不同的日志级别，以区分不同的日志信息
* 测试时能够打印出日志，但发布时应当屏蔽日志
* 测试人员上报问题时，能够实时查看日志，并且上报日志，方便开发人员定位问题
* 调试时，日志能够实时输出到XCode的console中

### 日志级别

在后台开发时，经常使用的日志工具log4j，其提供了不同的日志级别，包括debug、info、warning等，但是在前端开发过程中，可能就是简单的使用NSLog打印日志，这两者的区别就在于后台在运行出错时，还可以看日志信息来判断错误原因，但是前端是没法看到日志信息的，所以后台对于日志的要求更加精确，因为生产环境应该只打印需要打印的信息，而前端在生产上反正看不到信息，所以开发时对日志的处理可能更随意。但是这种习惯对于开发人员来说，并不是什么好事情，日志分级还是有必要的，其带来的好处有以下：

* 分级后的日志，能够更清晰的呈现给开发人员，并且能够让更重要的信息及时通知给开发人员。如果所有的error和普通的debug信息都一个样子，很容易会让开发者忽略，而分级后的日志，会更有针对性，甚者可以通过后期日志文件的处理，将所有error级别的日志过滤出来，更容易发现问题，而这如果是不分级的话，是很难达到的。
* 日志分级，能够让开发人员在代码开发时，更加审慎的思考，在写日志的时候到底是用什么级别的日志来写入，而不是像之前那样想写日志就直接NSLog就好了，这多一步的思考，能够更加严谨开发人员的逻辑，清楚知道在这里写日志的目的。

日志级别的实现，实际上在上文已经提到了，ASL本身就已经提供了各种级别的日志了，我们要做的就是在ASL的基础上进行封装即可，当然iOS 10以后，新的os_log已经替换了ASL，推荐使用os_log实现。具体的实现方法，这里不再赘述，可以参考[这篇文章](http://doing-it-wrong.mikeweller.com/2012/07/youre-doing-it-wrong-1-nslogdebug-ios.html)。当然，现有的三方日志框架，例如[CocoaLumberjack](https://github.com/CocoaLumberjack/CocoaLumberjack)，也是不错的选择，不过其原理也是同上。

### 日志开关

日志开关，就是根据打包的不同，来关闭或者打开日志。这里实际上牵涉到另一个问题，就是如何管理不同的配置打包，例如开发环境、生产环境、AppStore等，这个准备在另外一篇文章说明。

日志开关的实现，实际上就是用宏定义来替换相应的日志实现。一般来说，都是在发布AppStore时，关闭日志，这时候就添加如下代码

```
#define NSLog(fmt,...)
```

就能够将所有NSLog输出的日志关闭，当然这只是个简单示例，真正在项目中使用的时候可能还需要处理下，如果是ASL的话，那么又是另外一套宏定义了，但是原理大致如此。

### 日志上报

往往测试人员在测试过程中，会遇到各种问题，这时候报给开发人员的时候，开发人员还需要再复现问题，如果能复现还好，不能复现的就痛苦了，这时候如果能够将日志同时上报的话，就能解决开发人员的一大部分问题，因此日志的实时查看和上报是很切实的需要。

#### 日志获取

真机测试时的日志实时查看，就涉及到如何读取系统日志（这里我们默认日志是采用NSLog输出的，如果是printf输出的话，后面另行说明），这就有两种方法。

##### 重定向

前面我们介绍了NSLog会输出到stderr，在C语言中，有三个默认的句柄

```
#define	stdin	__stdinp
#define	stdout	__stdoutp
#define	stderr	__stderrp
```

其对应的iOS系统层面的三个文件句柄如下

```
#define	 STDIN_FILENO	0	/* standard input file descriptor */
#define	STDOUT_FILENO	1	/* standard output file descriptor */
#define	STDERR_FILENO	2	/* standard error file descriptor */
```

既然NSLog是写到STDERR_FILENO中去的,那么根据Unix的知识,我们可以重定向这个文件,让NSLog直接写到我们自己的文件中去，由于重定向到的文件是我们沙盒中的文件,那么我们就可以随意操作这个文件了，包括日志的实时查看以及日志的上报。

重定向的方法有以下几种

* freopen

```
NSArray *paths = NSSearchPathForDirectoriesInDomains(NSDocumentDirectory, NSUserDomainMask, YES);
NSString *documentsPath = [paths objectAtIndex:0];
NSString *loggingPath = [documentsPath stringByAppendingPathComponent:@"/mylog.log"];
//redirect NSLog
freopen([loggingPath cStringUsingEncoding:NSASCIIStringEncoding], "a+", stderr);
```

注意： freopen是没有办法将NSLog重定向回去的，因为我们不知道stderr在不同平台下的具体文件路径，而且iOS中由于沙盒机制，也没有办法访问沙盒外的文件，所以如果想重定向回去的话，只能用下面介绍的dup2

* dup2

```
- (void)redirectLog {
	NSPipe* pipe = NSPipe.pipe;
   NSFileHandle* fhr = [pipe fileHandleForReading];
   dup2([[pipe fileHandleForWriting] fileDescriptor], fileno(stderr));
   [NSNotificationCenter.defaultCenter addObserver:self selector:@selector(readCompleted:) name:NSFileHandleReadCompletionNotification object:fhr];
   [fhr readInBackgroundAndNotify];
}

- (void)readCompleted:(NSNotification*)notification
{
    [((NSFileHandle*)notification.object) readInBackgroundAndNotify];
    NSString* logs = [NSString.alloc initWithData:notification.userInfo[NSFileHandleNotificationDataItem] encoding:NSUTF8StringEncoding];
    // 这里就拿到了日志 可以自己发挥做些东西了
    dispatch_async(dispatch_get_main_queue(), ^{
        [self addLogMessage:logs timestamp:[[NSDate date] timeIntervalSinceNow]];
    });
}
```

如果想要重定向回去的话，只要记住原始的句柄，然后用dup2即可。

```
int originH1 = dup(STDERR_FILENO);
//这里是重定向到自己的文件
[self redirectLog];
...
//这里是重定向回来
dup2(originH1, STDERR_FILENO);
```

* GCD的dispatch Source

```

- (dispatch_source_t)_startCapturingWritingToFD:(int)fd  {
    int fildes[2];
    pipe(fildes);  // [0] is read end of pipe while [1] is write end
    dup2(fildes[1], fd);  // Duplicate write end of pipe "onto" fd (this closes fd)
    close(fildes[1]);  // Close original write end of pipe
    fd = fildes[0];  // We can now monitor the read end of the pipe
    char* buffer = malloc(1024);
    NSMutableData* data = [[NSMutableData alloc] init];
    fcntl(fd, F_SETFL, O_NONBLOCK);
    dispatch_source_t source = dispatch_source_create(DISPATCH_SOURCE_TYPE_READ, fd, 0, dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_HIGH, 0));
    dispatch_source_set_cancel_handler(source, ^{
        free(buffer);
    });
    dispatch_source_set_event_handler(source, ^{
        @autoreleasepool {
            while (1) {
                ssize_t size = read(fd, buffer, 1024);
                if (size <= 0) {
                    break;
                }
                [data appendBytes:buffer length:size];
                if (size < 1024) {
                    break;
                }
            }
            NSString *aString = [[NSString alloc] initWithData:data encoding:NSUTF8StringEncoding];
            //printf("aString = %s",[aString UTF8String]);
            //NSLog(@"aString = %@",aString);
            //读到了日志,可以进行我们需要的各种操作了
        }
    });
    dispatch_resume(source);
    return source;
}

```

注意：在使用的时候，记得保存返回的dispatch_source_t对象，不然释放了就拿不到日志了。

重定向的方法是一种侵入式的方法，因为它会导致日志在debug console中完全无法显示，这样在联机debug的时候就会比较麻烦。

##### ASL

前面介绍了NSLog会去调用ASL记录日志，因此所有的日志都可以在ASL中找到，因此通过ASL提供的查询方法就可以查出当前所有的日志。ASL的好处是没有重定向文件,所以不会影响Xcode等控制台的输出,它是一种非侵入式的读取的方式，对于开发人员来说完全无感。（CocoaLumberjack也是根据此实现）

下面是[BugshotKit](https://github.com/marcoarment/BugshotKit)使用ASL的代码

```
- (BOOL)updateFromASL
{
    pid_t myPID = getpid();
    
    aslmsg q, m;
    q = asl_new(ASL_TYPE_QUERY);
    aslresponse r = asl_search(NULL, q);
    BOOL foundNewEntries = NO;
    
    while ( (m = SystemSafeASLNext(r)) ) {
        if (myPID != atol(asl_get(m, ASL_KEY_PID))) continue;

        // dupe checking
        NSNumber *msgID = @( atoll(asl_get(m, ASL_KEY_MSG_ID)) );
        if ([_collectedASLMessageIDs containsObject:msgID]) continue;
        [_collectedASLMessageIDs addObject:msgID];
        foundNewEntries = YES;
        
        NSTimeInterval msgTime = (NSTimeInterval) atol(asl_get(m, ASL_KEY_TIME)) + ((NSTimeInterval) atol(asl_get(m, ASL_KEY_TIME_NSEC)) / 1000000000.0);

        const char *msg = asl_get(m, ASL_KEY_MSG);
        if (msg == NULL) { continue; }
        [self addLogMessage:[NSString stringWithUTF8String:msg] timestamp:msgTime];
    }
    
    if ([UIDevice currentDevice].systemVersion.floatValue >= 8.0f) {
        asl_release(r);
    } else {
#pragma clang diagnostic push
#pragma clang diagnostic ignored "-Wdeprecated-declarations"
        // The deprecation attribute incorrectly states that the replacement method, asl_release()
        // is available in __IPHONE_7_0; asl_release() first appears in __IPHONE_8_0.
        // This would require both a compile and runtime check to properly implement the new method
        // while the minimum deployment target for this project remains iOS 7.0.
        aslresponse_free(r);
#pragma clang diagnostic pop
    }
    asl_free(q);

    return foundNewEntries;
}

asl_object_t SystemSafeASLNext(asl_object_t r) {
    if ([UIDevice currentDevice].systemVersion.floatValue >= 8.0f) {
        
        return asl_next(r);
    }
#pragma clang diagnostic push
#pragma clang diagnostic ignored "-Wdeprecated-declarations"
    // The deprecation attribute incorrectly states that the replacement method, asl_next()
    // is available in __IPHONE_7_0; asl_next() first appears in __IPHONE_8_0.
    // This would require both a compile and runtime check to properly implement the new method
    // while the minimum deployment target for this project remains iOS 7.0.
    return aslresponse_next(r);
#pragma clang diagnostic pop
}
```

注意：ASL在iOS 10以后被替代了，导致部分api在iOS 10以上会出现问题，比如上面代码中使用到的asl_search，其结果就是拿不到日志，但是新的os_log中暂时还没有提供相应的api来替代它（具体可以见[这里](https://forums.developer.apple.com/thread/67390)），所以在iOS 10以上的系统中，目前可能还得使用重定向来实现，当然还需要继续关注苹果后续的行动。

#### 上报

不论是通过重定向还是ASL，当我们拿到日志后，可操作的空间便大了很多，我们可以自定义一个View来实时显示日志，还可以将日志上传到指定的服务器，或者邮箱等，这里可以使用第三方的库[BugshotKit](https://github.com/marcoarment/BugshotKit)，它的实现原理也就是上面介绍的，它也同时提供了一个展示的view，并且能够截屏，同日志一起发送到邮箱中。

但是BugshotKit有一个问题，就是当应用突然crash的时候，日志就没法再上传，因此这点需要自行改进。

### 日志输出到Debug Console

这个问题，在没有重定向时，并不是个问题，因为NSLog会自动将日志输出到Debug Console中（就是说，在连接XCode的时候会显示在下面的console中），但是如果使用了重定向的话，这时候就会出现问题，当然这也是由iOS 10带来的问题，因为iOS 10以前可以使用ASL。

实际上，iOS 10带来的ASL问题暂时也没什么好的解决办法（如果有的话，请告诉我），在这种情况下，为了实现第三个功能，就只能在iOS 10上用重定向来实现日志获取，如果需要在Debug Console中查看日志时，那就只能将stderr重定向到原来的位置，这也是一种临时的解决办法。

##参考

* [重定向NSLog](http://titm.me/?cat=1)
