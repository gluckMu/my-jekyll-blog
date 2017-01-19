---
layout: post
title:  "iOS开发基础——配置"
date:   2017-01-19
desc: "iOS开发基础——配置"
keywords: "iOS开发"
categories: [iOS]
tags: [iOS]
---

## 引言

在公司开发项目时，面临着多套环境（开发、测试、准生产、生产等）维护的问题，对客户端来说，不同的环境，意味着与后台相关的服务的配置可能不同，例如接口地址、页面地址等，还有可能三方的sdk配置也会有影响，在这种情况下，客户端如何在一套代码里干净、有效的管理不同配置就非常重要。

## 目的

* 一套代码
* 导出不同环境的ipa
* 尽量少的影响业务代码开发

## 方法

现有的环境配置方法，主要有两种，一个是基于target，一个是基于configuration，在介绍具体的方法前，先说明下target和configuration到底是什么。

### 基础

#### target

一个target对应一个product，它包含了编译这个product所需要的一系列信息，包括build setting、build phases，通常一个project下可以有多个target，并且target之间可以互相依赖，xcode在打包时可以自动检测依赖，并且按照相应的顺序来build对应的target，当然，你也可以手动设置依赖，即使这两个target之间没有依赖。（CocoPods就是利用了手动设置，具体的原理见[这里](http://www.jianshu.com/p/ad2e37e741bb)）

#### configuration

一个configuration是一系列配置信息的集合，这些配置信息都是在build setting下设置的，一个projcet可以有有多个configuration，并且configuration都是被所有target共享的，默认情况下，xcode会自动生成两个configuration，即debug和release。

#### scheme

一个scheme定义了打包时使用的target和对应的configuration，在导出ipa时必须指定且只能够指定一个scheme，当选中一个scheme时，build所需要的全部信息就已经设置好了。

#### 总结

从scheme的作用来看，打包的时候必须指定一个scheme，那如果想在一套代码里，导出不同环境的ipa，就必须有多个scheme。但是这多个scheme之间，可以通过修改target，也可以通过修改configuration来实现，因此就有了之前说的两种方法。

### 基于Configuration实现

#### 创建configuration

在项目的PROJECT->Info下的Configuration栏下，点击新增，会出现Duplicate选项，这里就涉及到要怎么创建Configuration，如果说你需要有两个环境，一个是开发，一个是生产，那么每个环境必须有Debug和Release两个Configuration，这样总共就会有四个，在新建的时候，每个环境的Debug Configuration必须Duplicate From Debug，Release Configuration必须Duplicate From Release，这个Debug和Release是工程默认就会有的。

<div align='center'>
<img src="{{site.baseurl}}{{ site.img_path }}/iOS_Develop_Foundation_Configuration/configuration-create.png" alt="Create Configuration"/>
</div>

#### Build Settings

创建configuration后，在PROJECT->具体TARGET->General里面的所有信息，应当都不能直接修改，因为在这里修改的话，意味着所有的configuration都改了，影响面比较大，除非是你就是想所有configuration都一样的话，可以在这里修改，否则千万不要动。这里的信息都可以在Build Settings里面找到对应的配置，可以通过Build Settings去修改，支持更细粒度的修改。在Build Settings里，一般涉及到的配置有以下几个。

##### Bundle ID && Info.plist

* Bundle ID：这个影响是否可以在真机上同时存在 
* Info.plist：Info.plist里面主要设置了Display Name（测试版可能需要名字区分），Bundle ID（这个和前面的设置一样，plist默认就是引用上面的那个，不过你也可以直接在plist改），Version（版本号可以不同环境不同），还有些其他设置，因此对于不同的configuration，建议设置不同的plist文件，Info.plist的具体信息可以参见[苹果官方文档](https://developer.apple.com/library/content/documentation/General/Reference/InfoPlistKeyReference/Articles/AboutInformationPropertyListFiles.html)，注意Info.plist文件必须首字母大写。 

<div align='center'>
<img src="{{site.baseurl}}{{ site.img_path }}/iOS_Develop_Foundation_Configuration/configuration-build-settings-packaging.png" alt="Build Settings Packaging"/>
</div>

##### Preprocessor Marcos

所有的宏定义都可以放在这，例如如果我们要区分环境的话，就可以在这里添加环境变量的宏定义，然后在代码中直接使用即可。

<div align='center'>
<img src="{{site.baseurl}}{{ site.img_path }}/iOS_Develop_Foundation_Configuration/configuration-preprocessor-marcos.png" alt="Build Settings Preprocessor Marcos"/>
</div>

##### App Icon

如果想让开发版本的app上的图标特殊的话，就需要在这里配置了，但是前提是你必须先创建好Asset Catalog。

<div align='center'>
<img src="{{site.baseurl}}{{ site.img_path }}/iOS_Develop_Foundation_Configuration/configuration-app-icon.png" alt="Build Settings App Icon"/>
</div>

##### 更多

上面的这几项都是经常使用的，但是如果你有更多特殊的需求的话，可以自行探索，例如Other Link Flags、Header Search Paths等。

#### 创建scheme

配置完configuration之后，剩下最后一步，就是对不同的环境，创建出各自的scheme，这样在打包的时候，只需要选择特定的scheme即可。记住，这里的scheme对应的是不同环境，并不是一个configuration就对应一个scheme，通常是两个configuration（Debug和Release）对应一个scheme。

scheme的创建，这里就不细说了，不会请自行百度，注意的是，在scheme配置时，Run、Test、Analyze里面的Bulid Configuration选择相应的Debug Configuration，而Profile和Archive选择相应的Release Configuration。

<div align='center'>
<img src="{{site.baseurl}}{{ site.img_path }}/iOS_Develop_Foundation_Configuration/configuration-scheme.png" alt="Create Scheme"/>
</div>

#### 打包

至此，所有的配置都已经结束，然后后续在使用时，只需要选择相应的scheme打包即可，如果是要配合持续集成工具的话，还需要新建不同的打包项目，然后在各自项目下使用不同环境的打包脚本即可。

### 基于Target实现

基于Target实现的方法实际上和Configuration基本上是一样的，不过不同的Target代表不同的环境而已，同样也是在修改Build Settings下面的相关配置项，然后创建scheme，打包即可，这里就不再赘述。

### 两种方案对比

* 根据target和configuration的定义来看，target应当被用来区分不同的product，就像CocoaPods那样，如果只是用来区分环境配置的话，显得有些过于臃肿了，而configuration就是为了配置来设计的，所以会更加优雅点
* 使用target区分环境的话，对于开发人员来说，要注意的是，在往project中添加文件时，需要指定正确的target，否则会出现编译错误，而configuration则完全没有这个问题，只需要一次设置即可

<div align='center'>
<img src="{{site.baseurl}}{{ site.img_path }}/iOS_Develop_Foundation_Configuration/target-add-file.png" alt="Add Files to Target"/>
</div>

* target比configuration要多出个build phases的设置，因此在配置功能上要更加自由和强大，但是如果是执行脚本的话，由于能够拿到configuration，所以只需要加个if判断，也能够实现脚本执行，所以可以忽略，但是如果你的项目中有这种需求时，那么可以考虑target
* 使用configuration来区分环境要比target更加复杂，因为在target中，只需要将所有的configuration都配置成一致的即可，不用考虑到底有什么configuration，但是在使用configuration来区分环境时，则需要更细粒度的设置，不同的configuration有不同的设置，尤其是当你的project还引用了其他project，这时候你在其他project里面也需要添加相应的configuration（当然也可以在当前的project里面进行设置，但是同样复杂），所以这就导致了使用configuration的复杂性，但是这种设置是一次性的，跟这种复杂性带来的好处相比，个人觉得还是值得的
* 使用target的一个额外好处，是CocoaPods支持针对target的不同配置，意思是可以在开发环境依赖某些库，到生产可以不打包进去，而configuration的话，也可以使用other link flags，配合lib search path来实现（这个又可以另开一篇介绍），相对来说，还是更复杂些，

综合上面这几点，个人推荐还是优先使用configuration来实现环境的切换。
