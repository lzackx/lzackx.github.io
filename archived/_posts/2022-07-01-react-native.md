---
layout: post
title:  "React Native"
date:   2022-07-01 08:00:00
categories: ReactNative
---
**[React Native](https://reactnative.dev/)**

- [0. 前言](#0-前言)
- [1. `Javascript`与`Native`的交互](#1-javascript与native的交互)
  - [1.1 初始化参数](#11-初始化参数)
  - [1.2 原生功能模块](#12-原生功能模块)
  - [1.3 原生UI模块](#13-原生ui模块)
- [2. `Redux`的配合](#2-redux的配合)
  - [2.1 `react-redux`](#21-react-redux)
  - [2.2 `toolkit`](#22-toolkit)
- [3. 打包与集成](#3-打包与集成)
  - [3.1 打包](#31-打包)
  - [3.2 集成](#32-集成)
- [4. 热更新](#4-热更新)
  - [4.1 `CodePush`](#41-codepush)
- [5. `Fabric`理解](#5-fabric理解)
- [相关链接](#相关链接)

# 0. 前言

`React Native`是很早就有的App跨平台开发框架了, 后来因为`Javascript`线程的性能问题渐渐进入瓶颈. 

到了2018年后, `Facebook`团队通过2年的努力, 把渲染器重构, 诞生出`Fabric`, 才再次焕发新的生机.

本文记录使用`React Native`的一些开发的关键知识点, 帮助唤醒记忆.

# 1. `Javascript`与`Native`的交互

`React Native`框架主要包括3个部分, `Javascript`侧、`C++`侧和`Native`侧, UI上的最小表达方式是`RCTRootView`, 而文件的最小表达方式是`bundle`.

## 1.1 初始化参数

`RCTRootView`在初始化时, 可以传递`Native`侧的初始化参数到`Javascript`侧, 用于初始化`Javascript`侧的运行所需数据.

在使用多`bundle`方式开发的混合App中, 初始化参数会带上App的公共参数与业务所需的私有参数.

## 1.2 原生功能模块

由于业务也需要使用到原生的某些能力, 例如基于原生的路由, 又或者事件通知, 这时候就需要两侧的数据传递.

在`Native`侧遵循`RCTBridgeModule`协议并利用宏导出方法, 就可以让`Javascript`侧调用同名方法.

对于这些交互, 有几种场景:

* 导出变量
* 方法的直接调用
* 有回调的方法调用
* 事件发送

另外, 使用Swift的时候, 需要额外声明导出的方法.

最后, 还有一些小细节, 就是交互数据传递的执行线程可以在模块中决定, 还有遵循`RCTBridgeDelegate`协议下的模块初始化方法后, 可以动态注册依赖的模块.

## 1.3 原生UI模块

当`Javascript`侧想要使用平台特定的一些UI组件时, 可以通过继承实现`RCTViewManager`类, 并声明导出属性或方法来处理. 这种情况下, 在`Javascript`侧的组件就可以通过`property`来传递参数或方法了.

# 2. `Redux`的配合

`Redux`是状态管理工具, `React Native`中使用`Redux`能让数据调理更清晰, 同时还有非常丰富的插件生态.

## 2.1 `react-redux`

`Redux`可以配合很多种框架, `react-redux`就是`Redux`配合`React`使用的一种, 由于它仅涉及`Javascript`侧的状态管理, 不会有原生处理, 所以它的集成与原生无关.

`Redux`的理解中, 有几个关键定义:

* Property, 描述Component属性的对象
* State, 描述Component状态的对象
* Store, 通常在一个注册的`Javascript`侧的App中只有一个, 通过若干个Reducer初始化, 所有 Component 都可以从此处获取到State
* Selector, 从 Store 中获取 State 的一个指定数据的方法
* Reducer, 输入当前 State 和 Action, 输出新的 State 的一个行为对象.
* Action, 描述行为的对象, Store 通过 Dispatch 一个 Action 到 Reducer, 触发 State 更新

![Redux数据流](/assets/images/2022-07-01-redux-flow.gif)

使用的流程通常是:
1. 新建 State
2. 创建 Reducer
3. 创建 Action
4. 注册 Reducer 到 Store 内
5. Store connect Component
6. 利用 Selector 获取 State
7. mapDispatchToProps,暴露 Store dispatch Action 给 Property, 
8. 更新 State
9. mapStateToProps, 更新Property

## 2.2 `toolkit`

`Redux`官方推出的傻瓜式使用`Redux`的工具库, 自带了中间件`redux-thunk`, 再来看看几个关键定义:

* Slice, 由`toolkit`定义的一个快速创建 Reducer 和 Action 的对象, 一个Slice通常管理一个单一原则模块的 State
* Thunk, 数据流与 Action 相似, 但是一种可以延迟执行一些操作的对象
* entityAdapter, 对一个数据集合的封装管理, 内部提供了很多处理集合数据的方法

对比原本的`Redux`数据流, `toolkit`提供的封装极大地简化了使用流程.

# 3. 打包与集成

`React Native`的开发伴随着很多依赖库的集成, 在开发过程中, 开发环境、持续集成、加载打开都需要维护.

本文开端提到过, 文件的最小表达方式是`bundle`, 意思是不论是`Javascript`侧的打包代码, 还是所需的本地资源文件, 都会放在`bundle`格式的目录中, 在加载时, 提供它在沙盒中的`URL`, 进行加载展示.

## 3.1 打包

`React Native`的打包器是`Metro`, 打包时, 提供其配置文件, 可以做到纵向的分包打包, 减少`bundle`文件中的`jsbundle`, 即`Javascript`侧的代码大小.

在加载使用时, 通过分段加载基础包和业务包来运行界面, 使得一个项目工程中的基础部分重复使用.

这可能在内嵌`bundle`的`React Native`项目中没什么太多的差别, 但是对于可以热更新的`React Native`来说, 可以减少`bundle`的体积, 等于减少网络带宽, 提升加载速度, 是比HTML页面体验好很多的一种优化.

配置`Metro`打包后, 必须要先打包基础包, 然后根据基础包产出的基础代码依赖声明文件, 配置业务包读取并打包剩下的`Javascript`代码.

这里的打包方案社区也有不少, 但因为`React Native`的版本一直在迭代, 所以高版本的打包配置细节可能要有些许的调整, 但这个方案的思路是一致的.

## 3.2 集成

`React Native`的集成在其自生成的项目文件中, 时通过2个Ruby文件引入`CocoaPods`依赖本地的库集成的, 对于现有项目集成`React Native`有些不友好, 因为生硬搬依赖或者搬脚本都不是好维护的办法.

所以其实一个一劳永逸的办法是, 创建一个现有项目中所用的所有`bundle`都用到的依赖的空壳项目, 安装所有依赖, 然后分别创建`package.json`和`node_modules`的软链接到现有项目中去, 这样子就能在现有项目中, 通过`CocoaPods`引入相对路径正确的`React Native`依赖. 

而所有`bundle`的依赖增加, 都要统一注册到空壳项目中去, 这样子的处理有点取巧, 但是对于统一管理总依赖, 统一管理`React Native`的升级都很方便.

# 4. 热更新

## 4.1 `CodePush`

`CodePush`是推送`React Native`热更新的开源工具, 有服务端、命令行工具和平台SDK, 由于高级版已经融合进了微软的`App Center`, 所以可以使用2的版本, 实测无影响.

需要注意的是, `CodePush`的热更新版本匹配逻辑是按照版本范围大小来匹配的, 不是时间排序匹配, 所以在多`bundle`集成原生的项目中, 可能需要根据需要修改服务端的匹配机制.

另外, 对于多`bundle`集成的项目来说, `CodePush`的SDK也是要作出大量修改的, 因为其本身的设计思路是1对1的关系, 用了很多的静态变量, 所以想要支持多`bundle`的项目, 需要调整多包支持.

# 5. `Fabric`理解

`Fabric`是渲染器, 内部设计比较多的定义.

从渲染线程来说, 如下:

* `Javascript`线程
* 后台线程
* UI线程

从渲染对象的角度来说, 如下:

* React 元素树（React Element Trees）
* React 影子树（React Shadow Tree）
* 宿主视图树（Host View Tree）

从渲染流水线的角度来说, 如下:

* 渲染(Render)
* 提交(Commit)
* 挂载(Mount)
  
以上的各个角度差不多是一一对应的关系.

其中, `Fiber`是影子树渲染过程中的算法, 而`yoga`是布局计算库.

而对于元素树中只参与布局的元素, 在影子树的计算中, 会被拍平, 使得视图树中的嵌套不会跟元素树中的一一对应.

# 相关链接

* [React Native](https://reactnative.dev/)
* [React Native 中文网](https://www.react-native.cn/)
* [NPM](https://www.npmjs.com/)
* [Redux](https://redux.js.org/)
* [Redux 中文官网](http://cn.redux.js.org/)