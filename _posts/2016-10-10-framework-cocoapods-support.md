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

`podspec`文件的语法可以在[这里](https://guides.cocoapods.org/syntax/podspec.html)查到。

*备注：CocoaPods对于`xcdatamodeld`文件的处理很奇怪，想动态与静态集成都兼容的话，需要把此文件当成resources且使用手动创建类文件来进行集成*

*1)pod工具*:

尽量配合Xcode使用较新的`pod`工具。

```
gem source -l
```

可以查看`gem`源。

默认为：`https://rubygems.org/`

也可以通过以下方式换源（`https://ruby.taobao.org/`、 `https://gems.ruby-china.org`([Ruby China](https://ruby-china.org/))）,再更新pod工具

```
gem sources --remove xxx
gem source -a xxx
```

另外还有，如果很早以前安装过`gem`，那不得不还要额外做一些事情来避免`Operation not permitted`的发生。
然后通过以下指定位置来更新。

```
gem install -n /usr/local/bin cocoapods --pre
```

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
