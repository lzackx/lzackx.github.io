---
layout: post
title:  "Z系列工具汇总"
date:   2022-06-30 08:00:00
categories: Repositories
---
**Z系列工具汇总**

- [前言](#前言)
- [Frameworks](#frameworks)
  - [Zouter](#zouter)
  - [Zequest](#zequest)
  - [Zopup](#zopup)
  - [Zodifier](#zodifier)
  - [Zoo](#zoo)
  - [Zlement](#zlement)
  - [Zind](#zind)
  - [Zcripts](#zcripts)
  - [ZOpenWrtAction](#zopenwrtaction)

# 前言

最近发现模块多了起来, 所以做个汇总, 作为索引, 也作为介绍.


# Frameworks

## [Zouter](https://github.com/lzackx/Zouter)

 iOS的一个基于Target-Action的路由库, 有以下特性:

 * 支持`Objective-C`和`Swift`
 * 解偶的同时, 可集中管理注册的Target-Action
 * 支持执行的优先级处理
 * 支持参数透传
 * 支持执行完成回调

## [Zequest](https://github.com/lzackx/Zequest)

iOS的一个基于`AFNetworking`封装的请求库, 有以下特性:

* 支持`Objective-C`和`Swift`
* 支持请求数据缓存
* 支持自动JSON数据模型转换
* 支持统一注册公共的`Header`、`Post Body`和并发请求数等请求参数

## [Zopup](https://github.com/lzackx/Zopup)

iOS的一个基于`UIWindow`实现的队列弹窗库, 有以下特性:

* 支持`Objective-C`和`Swift`
* 单独弹出
* 支持队列弹出, FIFO
* 支持自定义弹出的对象, 可以是`UIView`或`UIViewController`
  
配合[SwiftEntryKit](https://github.com/huri000/SwiftEntryKit)基本覆盖所有弹窗场景

## [Zodifier](https://github.com/lzackx/Zodifier)

iOS的一个修改运行时变量的库, 适用于以下一些大量使用KVO响应慢的场景:

* 皮肤主题切换
* 多语言切换
* 登录状态切换

## [Zoo](https://github.com/lzackx/Zoo)

iOS的一个运行时调试控制台页面, 从`DoraemonKit`中重构而来, 有以下特性:

* 在原有可自定义添加工具的基础上, 可自定义集成选择自带的工具, 包括:
  * [Zoo.General](https://github.com/lzackx/Zoo.General)
  * [Zoo.Logger](https://github.com/lzackx/Zoo.Logger)
  * [Zoo.GPS](https://github.com/lzackx/Zoo.GPS)
  * [Zoo.UI](https://github.com/lzackx/Zoo.UI)
  * [Zoo.Performance](https://github.com/lzackx/Zoo.Performance)
  * [Zoo.MemoryLeakFinder](https://github.com/lzackx/Zoo.MemoryLeakFinder)
* 所有自带工具不会有任何的自发数据请求行为(`DoraemonKit`中有的请求都移除了)
* 通过其他仓库, 而非子Spec集成自带工具, 好处是可以在Podfile中指定工程的`configuration`安装线上运行时必要的, 可以立即检查问题的工具.

## [Zlement](https://github.com/lzackx/Zlement)

iOS的一个处理集合视图高度缓存的库, 优化`Cell`的渲染

## [Zind](https://github.com/lzackx/Zind)

iOS的一个运行FLutter页面的封装, 有以下特性:

* 管理FLutter容器的生命周期
* 支持复用Flutter的Engine
* 支持Flutter容器的弹窗

## [Zcripts](https://github.com/lzackx/Zcripts)

平常常用的一些脚本, 包括:

* 定时Crontab更新CocoaPods
* MacOS新机开发环境安装脚本
* ReactNative的Metro分包打包配置和脚本
* Raspberry Pi的OpenMediaVault安装和配置脚本

## [ZOpenWrtAction](https://github.com/lzackx/ZOpenWrtAction)

小米路由刷OpenWrt的配置和固件