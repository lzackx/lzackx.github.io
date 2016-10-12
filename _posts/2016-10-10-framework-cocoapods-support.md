---
layout: post
title:  "为Framework添加CocoaPods支持"
date:   2016-10-10 09:00:00
categories: Framework
---
**为Framework添加CocoaPods支持的步骤**

**[CocoaPods](https://cocoapods.org/)**

CocoaPods是很受开源社区青睐的Xcode集成开源Framework的管理工具，开源社区很多开源Framework基本都支持通过以下方式进行集成
(例如[Alamofire](https://github.com/Alamofire/Alamofire)、[SwiftyJSON](https://github.com/SwiftyJSON/SwiftyJSON)等)。

`pod '***'`

而这里将写下为Framework添加CocoaPods支持的具体步骤和可能遇到的坑。

**1. 在Github上为Framework项目创建仓库**

Github创建仓库过程相当简单，右上角的`'+'`号，按照提示创建就可以了，没难度。

这里留下一些`README.md`可能会用到的工具: [Shields.io](http://shields.io/), [Travis CI](https://travis-ci.org/)。

需要注意的是License是要选一下的，关乎这个Framework的仓库的开源说明。
具体不同的License怎么选，可以查看这个[Choose A License](http://choosealicense.com/)。

创建完毕后，以我的一个开源Framework为例[SwiftyRoutes](https://github.com/lzackx/SwiftyRoutes)。
在自己的terminal上进入想要放这个仓库的目录，例如

```
cd ~/Work/Repositories
git clone https://github.com/lzackx/SwiftyRoutes.git
```

等待后，仓库就在自己想要放的目录里了(这里是`~/Work/Repositories`)。

**2. 构建Framework目录**

然后，可以把Framework的Xcode相关项目文件放到这个目录里。
这里放的方法比较多样化，有的开源项目会另外创建一个source目录放置构成Framework的源码。
我个人比较喜欢创建一个Xcode的workspace，把framework和demo放在里面，方便`git clone`到本地的仓库在写好Framework后能添加到demo里。
但具体想要怎么样的目录结构都问题不大，需要关注的是接下来要讲解的podspec文件。

**3. 填写podspec文件**

podspec文件是让CocoaPods知道当前Framework相关信息并展示出去的文件。想要让Framework支持CocoaPods集成，首先就要填写这个文件。
在这里，用SwiftyRoutes.podspec做例子，在仓库根目录下(也就是`~/Work/Repositories/SwiftyRoutes`)创建这个文件，其内容如下:

```
Pod::Spec.new do |s|
  s.name             = 'SwiftyRoutes'
  s.version          = "1.0.0"
  s.summary          = 'A simple routing library for UIViewController written in Swift.'
  s.description      = 'SwiftyRoutes can make the transition between different UIViewControllers easier with URL.'

  s.homepage         = "https://github.com/lzackx/SwiftyRoutes"
  s.license          = { :type => 'MIT', :file => 'LICENSE' }
  s.author           = { 'lzackx' => 'zackx@foxmail.com' }
  s.source           = { :git => 'https://github.com/lzackx/SwiftyRoutes.git', :tag => s.version }

  s.requires_arc = true
  s.ios.deployment_target = '8.0'
  s.source_files = 'SwiftyRoutes/SwiftyRoutes/*.swift'
  s.frameworks = 'UIKit'
  
  s.pod_target_xcconfig =  {
        'SWIFT_VERSION' => '3.0',
  }
end
```

这里不得不说，podspec文件这里有坑，还不少，现在这里能看到的各种字段意思都比较好理解。
但需要多注意的分别是`version`、`source`、`deployment_target`、`source_files`等
(还有些没列出，假如Framework需要相关的Bundle文件，那还得注意resources什么的)。

*version*:这个是Framework将会被推到CocoaPods上的版本库说明，需要注意的是Framework项目的Info里要同步这个版本号，
并且并且并且(重要的话说三遍)，在仓库`commit`后，需要记得出一个对应这个版本号的Release，不然`pod trunk push`的过程，甚至`pod spec lint`都会不成功。

*source*:这个是直接填当前的GitHub相关仓库地址就OK，但是后面的参数则是指定此次push的版本，或许不一定是tag，可能可以是branch什么的，这个具体没试，但应该是这个道理。

*deployment_target*这个是Framework支持的版本号，如果是多平台支持的话，字段可能要添加watchOS什么的，想要支持什么就填什么。

*source_files*这个是核心中的核心，就是这个字段决定了`pod install`后Framework有什么文件被集成到项目中的，
有些开源Framework喜欢把这些文件放到另开的一个`sources`目录里就是因为这么做方便。此处一定要准确指定对应的源码文件，不然跟不放没区别。

**4. Swift 3.0的支持**

CocoaPods是支持Swift 3.0的Framework的，但要做一点额外的工作

*1)pod工具*:

这里不得不说，祖国防火墙的强大让开发者走弯路，由于Xcode8.0的发布，如果开发者用的Xcode是8.0的话，pod必须得用`1.1.0.rc.2`这个以上的版本。

如果`gem source -l`是`https://rubygems.org/`那么可能被墙，假如是`https://ruby.taobao.org/`那么现在此源(经目前测试)的pod版本只到`1.1.0.beta.2`，
现在，想要不翻墙获得最新版本，那就`gem sources --remove ***`去掉上面这2个略坑的源，
添加[Ruby China](https://ruby-china.org/)这个源`gem source -a https://gems.ruby-china.org`，然后再更新pod工具。

另外还有，如果很早以前安装过`gem`，那不得不还要额外做一些事情来避免`Operation not permitted`的发生。
然后通过`gem install -n /usr/local/bin cocoapods --pre`指定位置来更新。

*2).swift-version文件*:

对于Framework是swift 3.0写的源码，还得在仓库根目录(例如SwiftyRoutes例子的是:`~/Work/Repositories/SwiftyRoutes`)添加一个`.swift-version`文件，
文件内容就是`3.0`。

*3)Warning: Always Embed Swift Standard Libraries*

这里顺便说一下这个问题，Xcode 8有时候`pod install`后会出现这个警告(例如集成EaseMob的库)，这个是没办法的事情，强迫症也得忍一忍，
CocoaPods的issue中提到过这问题，说是还想支持Xcode 7的版本，Xcode智能地警告你需在target里添加这个字段，其实不加也应该没问题。

**5. 提交仓库**

在上面这些准备都好了以后，就可以`commit`当前所有的修改，并同步到GitHub仓库去了。

**6. 验证podspec**

这里也是有坑的，但在这之前，要先保证上面说到的pod工具已经是最新的。

首先，验证当前的`~/.cocoapods/repos/master`是否存在，并进入到此目录。即:

`cd ~/.cocoapods/repos/master`

假如没有，就执行如下命令来创建:

```
cd ~/.cocoapods/repos
git clone https://github.com/CocoaPods/Specs.git master
```

然后，注册CocoaPods的推送资格

`pod trunk register YOUREMAIL 'USERNAME' --description='DESCRIPTION'`

CocoaPods会发来验证邮件，检查上面的邮箱，进行验证。

之后，验证上面所写的podspec文件是否正确。即(以SwiftyRoutes为例子):

`pod spec lint ~/Work/Repositories/SwiftyRoutes/SwiftyRoutes.podspec --verbose` 

当发现pass后，就可以进行推送了，如下:

`pod trunk push ~/Work/Repositories/SwiftyRoutes/SwiftyRoutes.podspec -- verbose`

然后等待，直到完成，就可以通过

`pod search SwiftyRoutes`

查看到自己的Framework了。
