---
layout: post
title:  "CocoaPods二进制"
date:   2017-09-11
desc: "CocoaPods二进制的一种新方法"
keywords: "CocoaPods"
categories: [iOS]
tags: [iOS]
--- 

## 为什么需要二进制

iOS项目依赖管理工具目前主流的有CocoaPods和Carthage两种，我们的项目中使用了CocoaPods，因为配置方便，不需要像Carthage那样还要自己配置，但是缺点是CocoaPods管理的是库的源码，因此在编译主工程的同时需要先编译Pods，虽然XCode会缓存编译结果，但是开发过程中还是会经常clean项目后再编译，导致编译时间过长。但实际情况是，Pods下的库只需要编译一次就好，基本不会存在变动，因此我们需要将CocoaPods管理的依赖库二进制化，减少编译时间，同时还需要能够切换回原代码，方便开发调试。

## 已有方案

针对CocoaPods的二进制化，业内已经有很多方案，例如[CocoaPods组件平滑二进制化解决方案](http://www.infoq.com/cn/articles/cocoapods-binarization)，[基于 CocoaPods 进行 iOS 开发](https://blog.dianqk.org/2017/05/01/dev-on-pod/)，还有些提出将CocoaPods整合Carthage一起使用，例如[I have a pod, I have a carthage](https://mp.weixin.qq.com/s/wV68OWGB3fiWc1hJW-o59g)，[CarthagePods](http://www.jianshu.com/p/6db0187cc50b)，其实本质上它们都是一样的原理，将代码编译成lib，然后创建新的pod，通过依赖新的pod，实现二进制化，通过依赖的pod的切换，实现源代码与二进制库的切换。

理解上述方案的实现原理后，就清楚这些方案的缺点很明显，就是需要重新创建一个pod，用来存放编译好的lib文件，也就是说每一个pod库文件的变更，都必须先编译一次，然后将编译好的库放进与之对应的二进制的pod里，更新升级，这一步比较麻烦。而在使用时，如果不要求切换回源码的话，那么podfile是不需要任何特殊配置的，只要依赖的是二进制化的pod库即可，这也是这种方法的优势，但是如果需要切换源码的话，就需要修改podfile。

那么有没有其他的方法，不需要再额外创建一个pod库呢，在[基于 CocoaPods 进行 iOS 开发](https://blog.dianqk.org/2017/05/01/dev-on-pod/)这篇文章中，作者提到过一种思路，就是使用CocoaPods提供的pre_install hooks里面去搞事情，于是我就想到能否在hooks中编译代码，然后修改相应的配置文件，达到二进制化的目的，最后就有了下面的新方法。

## CocoaPods准备知识

### 概述

CocoaPods是用Ruby开发出来的一套iOS依赖库管理工具，在使用时，工程里必须提供一个Podfile，里面描述了相应的依赖关系。实际上，Podfile里的配置就是Ruby代码，理解这一点有利于加深认识本文提出的方案。

```ruby
platform:ios, '8.0'
target 'Test' do

    pod 'SDWebImage', '~> 4.1.0'
    pod 'WeixinSDK', '~> 1.4.3'

end
```

### 原理

CocoaPods到底是怎么替我们管理依赖的，详细的原理可以看看[这篇文章](https://bestswifter.com/cocoapods/)。在iOS工程中使用静态库文件，主要就是配置`Build Setting`里面的`Header Search Path`、`Library Search Path`和`Other Linker Flags`，CocoaPods会自动替我们修改主工程的相应配置，实现库依赖，配置文件就是`Pods/Target Support Files/Pods-XXX(工程名)/Pods-XXX.debug(release).xcconfig`，里面存放了相应的配置。

CocoaPods管理的库有两种，一种是源代码形式的（如AFNetworking），一种是二进制化的包（如WeixinSDK），对于这两种库，CocoaPods会分别处理。

* 对于非二进制化的pod，CocoaPods会在自动生成的Pods项目下创建一个target，这个target指明了需要编译的源文件，然后Pods项目下会有一个以Pods-开头的target，它会依赖所有其他的target，保证所有的target都会被编译，如果这里的依赖被删除，那么对应的target就不会被编译。编译好的静态库文件存放路径是`PODS_CONFIGURATION_BUILD_DIR`，因此配置文件里面`LIBRARY_SEARCH_PATHS`里指定的就是`$PODS_CONFIGURATION_BUILD_DIR/XXX`。

<div align='center'>
<img src="{{site.baseurl}}{{ site.img_path }}/CocoaPods_Binary/dependency.jpg" alt="CocoaPods dependency"/>
</div>

* 对于二进制化的pod，由于不需要再编译，因此不会生成target，只需要在配置文件指定正确的LIBRARY_SEARCH_PATHS即可，一般是`$(PODS_ROOT)/XXX`。

### Hooks

CocoaPods提供了三种hook方法，分别是pre_install、post_install和plugin，我们在pod install --verbose时，注意打印出来的日志，即可找到对应的调用。

* pre_install是在资源下载完成后调用的，唯一入参是Pod::Installer，注意这时候还没有生成Pods项目文件以及相关配置文件

	```ruby
	pre_install do |installer|
	  # Do something fancy!
	end
	```

* post_install是在Pods项目文件已经创建完成但还没有写入本地磁盘时调用的，唯一入参是Pod::Installer

	```ruby
	post_install do |installer|
	  # Do something fancy!
	end
	```
	
* plugin hooks，CocoaPods中还提供了一个Pod::HooksManager.register方法，这个方法是专门为开发CocoaPods plugin提供的，它接收三个入参，第一个是plugin名称，第二个是调用时机，有三种:pre_install，:source_provider，和:post_install，:pre_install会在准备阶段调用，主要是完成插件加载后；:source_provider会在准备完成后，解析依赖前调用；:post_install会在Pods项目写入本地磁盘后调用。第三个参数是回调函数。

<div align='center'>
<img src="{{site.baseurl}}{{ site.img_path }}/CocoaPods_Binary/hooks.jpg" alt="CocoaPods hooks"/>
</div>

## CocoaPods二进制

二进制化首先要编译源代码，生成静态库，不过这一步我们先不说，先假设已经生成好静态库文件。

### 删除依赖

编译好之后，就需要修改Pods项目的配置，因为已经有了静态库，所以主工程编译时就不应该再编译pod源码，这一步可以通过删除前面介绍的依赖来实现。因为要修改Pods项目配置，所以只能选择在post_install hooks中去实现，而修改Pods配置文件，就需要用到CocoaPods提供的[Xcodeproj](https://github.com/CocoaPods/Xcodeproj)工具。

```ruby
def deleteDependency(installer, libs) # installer: post_install入参 libs: 需要二进制化的pod数组 
	installer.pods_project.targets.each do |target| #installer post_install的入参
		if (target.name.start_with?('Pods-'))
		    podTarget = target
		    target.dependencies.delete_if { |dependency| libs.include?(dependency.name) }
			break
		end
	end
end
```

修改过后的Pods项目中的Pods-开头的target就不再依赖二进制化的target，这样每次主工程编译时就不会再编译该Pod。

### 修改XCConfig

现在静态库已经生成好了，依赖也删除了，但是主工程里面的配置还是CocoaPods自动生成的那一套，在去找静态库的时候就会报错，因为路径不对，也就是`Library Search Path`的配置不正确，这就需要去修改CocoaPods生成的xcconfig里面的配置。因为post_install hooks调用时，xcconfig文件已经生成，所以我们可以直接去读取该文件然后修改，同样是利用Xcodeproj。

```ruby
def PodStatic.updateConfig(path, libs) # libs: 需要二进制化的pod数组 path: xcconfig文件路径，类似Pods/Target Support Files/Pods-XXX(工程名)/Pods-XXX.debug(release).xcconfig
	config = Xcodeproj::Config.new(path)
	libSearchPath = config.attributes['LIBRARY_SEARCH_PATHS']
	libRegex = libs.join('|')
	newLibSearchPath = libSearchPath.gsub(/\$PODS_CONFIGURATION_BUILD_DIR\/(#{libRegex})/) {
		|str| str.gsub(/\$PODS_CONFIGURATION_BUILD_DIR/, STATIC_LIBRARY_DIR)  #STATIC_LIBRARY_DIR是编译好的静态库文件存放路径
	}
	config.attributes['LIBRARY_SEARCH_PATHS'] = newLibSearchPath
	config.save_as(Pathname.new(path))
end
```

完成xcconfig文件修改后，主工程的XCode配置就会随着变化，由于只修改了`LIBRARY_SEARCH_PATHS `，而`Header Search Path`和`Other Linker Flags`还是由CocoaPods自动生成，并且相应的源文件也没有修改，所以主工程能够正确的找到头文件，并且也能够查看源代码，而且编译时会直接使用编译好的静态库，减少编译时间。

### 编译

最后，我们回过头来介绍编译，编译静态库本身并不复杂，使用XCode自带工具xcodebuild即可实现，但是xcodebuild需要指定target，这就要求必须有Pods.xcodeproj文件，但是post_install hooks还没有写入文件，这就导致这时候根本无法使用xcodebuild。所以剩下的可以用的就是plugin hooks（调用Pod::HooksManager.register方法），但是这个hooks是专门为plugin使用的，CocoaPods在执行plugin hooks的回调时是会有白名单判断的，还记得register方法的第一个入参么，就是插件名称，它会去判断这个插件名称是否在已注册的插件列表里面，然后再执行回调。当然，我们可以自己写一个CocoaPods plugin，但是我这里投机取巧了一下，因为我的CocoaPods里面已经有了一个cocoapods-stats的插件，我就在register方法中将插件名称传入cocoapods-stats，利用已有注册插件，实现了回调，由于方法比较骇客，所以可能不适用于所有人。至于调用register方法还是在post_install hooks中。

```ruby
def buildLibs(libs)  #libs 需要二进制化的pod
	if libs.length > 0
		Pod::HooksManager.register('cocoapods-stats', :post_install) do |context, _|
			Dir.chdir(PODS_ROOT_DIR){ # PODS_ROOT_DIR: Pods工程目录
				libs.each do |lib|
					Pod::UI.message "- PodStatic: building #{lib}"
					build_dir = STATIC_LIBRARY_DIR + File::SEPARATOR + lib # STATIC_LIBRARY_DIR: 编译好的静态库存放路径
					`xcodebuild clean -scheme #{lib}`
					`xcodebuild -scheme #{lib} -configuration release build CONFIGURATION_BUILD_DIR=#{build_dir}`
					`rm -rf #{build_dir + File::SEPARATOR + '*.h'}`
				end
			}
			Pod::UI.message "- PodStatic: removing derived files"
			`rm -rf build`
		end
	end
end

```

编译时还存在另一个问题，虽然现在可以编译一次，然后后续主工程编译都不需要再编译pod，但是在pod update时，是否需要每次都去重新编译一个静态库，因为很多时候都是添加或者删除组件，并不会影响已经编译好的静态库，除非是要删除，那么这时候再编译，只会增加pod update的时间。实际上，CocoaPods也需要解决这个问题，所以它已经给出了解决方法，我们只需要调用它的接口即可。

```ruby
def libsNeedBuild(installer, libs)
	unchangedLibs = installer.analysis_result.podfile_state.unchanged
	changedLibs = libs.delete_if { |lib| unchangedLibs.include?(lib) }
	changedLibs
end
```

这样我们就得到了变化的pod，然后编译即可，减少了pod update的时间。

### 切换源码

上面三步实现了CocoaPods的二进制化，并且由于只修改了CocoaPods的配置文件，源码文件并没有更改，相当于同时提供了源码和静态库，因此开发过程中可以随时查看三方库的源代码，而且编译时间减少，唯一的缺点是不支持debug，如果需要debug的话就需要切换为源码，而且在打包上架AppStore时，也最好使用源码导出，因为在pod install时编译好的静态库，主要是为了开发使用，不一定使用的是上架时候的配置。因此，我们还需要能够随时切换回源码模式。

由于我们修改了Pods项目的配置，所以每次执行pod update时，CocoaPods每次都会重新生成Pods项目文件，但是已下载的依赖库不会再重新下载，这样的话，切换源码非常简单，只要我们在post_install hooks不再修改配置文件即可，达到这一目的可以使用CocoaPods提供的ENV环境变量。

```ruby
post_install do |installer|
	libs = ['SDWebImage']  # libs: 需要二进制化的pod
	if ENV['ENABLE_STATIC_LIB']
		if !ENV['FORCE_BUILD']
			libs = libsNeedBuild(installer, libs)
		end
		buildLibs(libs)
		updatePodProject(installer.pods_project, libs)
	end
end
```

使用`ENABLE_STATIC_LIB=1 FORCE_BUILD=1 pod update`更新时，就会自动二进制化相关pod，并且强制重新编译，如果直接`pod update`就会切换回正常源码模式，使用极其简单。

## 改进

1. 编译需要使用plugin hooks，而目前使用的方法比较骇客，不太通用，所以可能还是修改成CocoaPods插件形式更适合，至于删除依赖和修改config文件，不确定能否一起放在插件中运行，但是尽量封装在一起比较好；
2. 目前无法通过在需要二进制化的pod后面增加配置项来标识，因为CocoaPods会去校验，所以现在只能通过额外设置一个数组来指定是否需要二进制化，然后通过环境配置项ENABLE_STATIC_LIB统一控制是否二进制化，后续如果改成插件需要研究是否能够通过增加配置项来达到单独控制的效果；
3. 目前还只支持静态库.a的封装，后续还要支持framework；
4. 删除pods时还没有增加删除静态库的操作；
5. 依赖库怎么解决，目前还是需要手动指定；

## 参考

* [细聊 Cocoapods 与 Xcode 工程配置](https://bestswifter.com/cocoapods/)
* [CocoaPods 都做了什么？](http://draveness.me/cocoapods.html)
* [基于 CocoaPods 进行 iOS 开发](https://blog.dianqk.org/2017/05/01/dev-on-pod/)
* [CocoaPods组件平滑二进制化解决方案](http://www.infoq.com/cn/articles/cocoapods-binarization)