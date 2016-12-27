# CocoPods使用

## 安装

首先必须安装好Ruby（MacOS自带），然后在国内由于墙的存在，需要更改下源。


```
$ gem sources --add https://gems.ruby-china.org/ --remove https://rubygems.org/
$ gem sources -l
https://gems.ruby-china.org
# 确保只有 gems.ruby-china.org
```

然后执行


```
$ sudo gem install cocoapods (-pre #此参数用来安装最新版)
```

CocoPods的安装就这么简单

## 使用

### 下载Specs Repo

CocoPods在使用时，会先在本地下载Specs Repo，这个Specs Repo里面存储了所有的可用的pod列表，之后的search都是通过Specs Repo来实现的。

Specs Repo是放在github上的，由于国内网络环境问题，导致github不稳定，且速度极慢，在网上搜索的几个国内的镜像也已经挂了，个人推荐还是挂上VPN直接git clone吧。

git本身可以设置代理，具体如下


```
git config --global http.proxy 'socks5://127.0.0.1:1080' 
git config --global https.proxy 'socks5://127.0.0.1:1080'
```

设置完代理后，就可以将github上的Specs Repo直接clone到本地，Specs Repo存放本地的路径是~/.cocoapods/repos/


```
git clone https://github.com/CocoaPods/Specs.git master
```

当然，以上这些步骤，也可以不手动执行，当你在使用时，它也会自动执行，但是这个初始化是肯定会进行的，建议手动执行下，后续使用时可以省时间。

### 项目使用

进入需要使用CocoPods的iOS工程目录，然后执行

```
pod init
```

在工程目录下，会生成一个Podfile文件，内容如下

```
# Uncomment this line to define a global platform for your project
# platform :ios, '9.0'

target 'FMComponent' do
  # Comment this line if you're not using Swift and don't want to use dynamic frameworks
  use_frameworks!

  pod "Alamofire", "4.2.0"
  pod "SwiftyJSON", "3.1.3"

  # Pods for FMComponent

  target 'FMComponentTests' do
    inherit! :search_paths
    # Pods for testing
  end

  target 'FMComponentUITests' do
    inherit! :search_paths
    # Pods for testing
  end

end
```

具体的pod语法不在这里介绍，可以直接去[CocoPods官网](https://cocoapods.org/)查看。

然后，执行

```
pod install
```

工程目录下会自动生成一些文件，包括一个.xcworkspace文件，之后都从这个文件打开工程，不是之前的.xcodeproj文件。在install时，会去github下载第三方依赖，因此建议可以设置代理。

### 更新

以后每一次修改Podfile时，都必须执行

```
pod update (--verbose --no-repo-update #建议带上，省去更新过程)
```

### 其他

```
pod env  #查看CocoPods环境
pod --version #查看CocoPods版本
pod search *** #搜索包
```