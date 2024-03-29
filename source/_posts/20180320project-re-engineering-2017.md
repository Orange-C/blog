---
title: 记一次项目重构
date: 2018-03-20 14:52:30
categories: notes
tags:
- JavaScript
- React
- Koa
- nodejs
- Reengineering
---

## 前言

为什么要写这么一篇好像和业务关系比较大的博客呢？一方面这算是去年的主要工作，前前后后加上探索和收尾大概花了5个月的时间（除了中间正式迁移的一个月之外，其他时间都是和日常业务开发并行一起在做），打算好好写一下就当是去年的工作小结了，另一方面，这次重构的确给我带来了很大的提升，想写一下学到的东西，不光是技术方面的探索，还有看问题的角度，以及在重构工作中各个阶段的把控，都学到了很多东西，有些可能可以水一篇博客出来，有读者感兴趣的话再写吧（逃）。

所以这篇文章可能有那么一点碎碎念，我会尽量提取一些关键的内容在各章节的开始部分。

<!-- more -->

## 重构之前

后续部分中会将项目简称rainbow。rainbow技术上是一个前端基于react，中台基于koa的内部项目，中台node对接数据端API。因为历史原因，rainbow包含了rainbow-red，rainbow-yellow，rainbow-blue等多个项目用于支持不同的业务线，每个都是独立的node服务和前端页面，但是各个项目之间功能又大同小异，维护和扩展一直以来基本是靠复制粘贴，要开新业务线就要复制一个项目，然后花费大量时间改耦合部分，同理在新业务线中开功能也是这个步骤。

这次重构的宏观目标就是把rainbow-red，rainbow-yellow，rainbow-blue多个项目合并成为rainbow-platform，统一维护功能模块，同时实现业务线的快速扩展。

## 一切开始之前——基础设施改善

> 工欲善其事，必先利其器

一个优秀的项目应该是从本地开发到上线整个流程都有良好的体验，这样才能在后续的开发和重构工作中获得更高的效率。

### 本地开发环境

优秀的开发环境应该包含以下几点：
1. 一键搭建
2. 开发时编译时间占比小
2. 方便调试

#### 项目开发环境

因为历史原因，最大的rainbow项目在webpack编译时大概有4000多个module，用的还是webpack1和很老的配置，初次打包和rebuild都需要好几分钟，基本是处于一个开发十分不方便的状态。而且因为用的是dev-middleware，有时候在需要只调试node代码的时候，每次node重启都会进行webpack编译，十分浪费时间。

将webpack从1升级到了3，用dll拆出了公共模块放在cdn上，用来加快编译，还做了一些资源拆分和异步加载，加上了hash值缓存等等，大部分还是一些webpack配置工程师的活。说实话webpack一直被人诟病配置麻烦，主要还是黑盒式的配置加上自由度太大，导致很多东西文档实在写不全（不过webpack的文档的确有点糟糕），多看看相关文章积累经验后面就轻车熟路了，不算什么有技术难度的事情。

一顿操作下来本地开发初次build大概在40秒左右，rebuild3秒左右，本来还打算修复hotReload，但是因为当时react-hot-loader问题有点多，而且开会讨论之后大部分人觉得有了hotReload不太方便在chrome上面调试样式，所以就没做了。

然后加入了nobuild模式，配合node的--inspect和vscode的调试工具，调试node的体验也是大大提升。

#### 组件开发环境

团队内有一些公共组件是以npm包的形式放在内部npm上的，但是基本不能本地开发，之前的开发都是靠发布到npm上再下载下来调试，<del>以至于出现了99.0.124这种神奇的版本号</del>。而且每个组件都打包了所有依赖到index.js中，以至于每个组件大概都有1m左右。

首先用项目配置里的loader配置同样做了一份用于组件的webpack配置，支持本地watch rebuild，不打包外部依赖，最后生成umd规范的组件和对应的css文件，然后是分享了一下npm link/unlink的开发模式以及package.json中peerdependence、dependence、devDependence的用法，版本号的规范根据[Semantic Versioning 2.0.0](http://semver.org/lang/zh-CN/)，并要求组件发布要写CHANGELOG。

这样下来公共组件总算是有序了一点，最近打算将各个视图组件和antd一样统一起来，用babel-plugin-import实现按需加载。

#### 内部脚手架

在上面两份配置成熟之后用对应配置写了一个脚手架工具rainbow-init，用于快速生成新项目，这样算是统一了各个项目的开发环境配置。

### 构建和部署

体验良好的部署流程应该是一键式黑盒式的，不需要开发者关心部署的环境问题，部署顺序问题。

首先是同步了各个机器的环境，包括node版本和pm2版本等等（顺便修复了node7.6版本导致的内存泄漏问题）。公司内部机器的部署工具主要是用于后端服务的，所以对前端的部署不太友好，之前也是一直没有区分前端和node的部署，都是直接机器上强覆盖重启。

我重新写了一下构建和部署的shell和node脚本，前端静态资源自动发布到内部cdn上，加上了html部署的流程，保留了原来node服务部署的流程。这样倚赖就能做到前端和node的分开部署，也防止了前端部署代码导致node服务重启的问题。部署流程如下：

* 前端：静态资源发布cdn，html部署
* node：node代码全量覆盖，重启服务

### 整合工具

最后将上述的所有配置和脚本整合到了rainbow-tools里面，所有项目和组件直接依赖这个包，做了一个强统一。不过所有配置都支持传自定义配置覆盖默认，以便部分项目的特殊需求。

到此为止基本项目的开发和部署体验已经满足了大部开发者的需求，并且也做到了统一的管理，在重写配置的时候顺便解决了资源加载问题和一些bug，可以说项目质量和团队的整体开发效率都有了很大的提升。

## 升级依赖 && 重写基础模块

在工具完善之后，接下来要做的便是夯实地基。在大型项目中，往往有很多基础的模块，被各个业务模块依赖，理想状态下这些模块应该是完全业务隔离的，但随着项目不断迭代，开发团队更替，这些模块难免会混入许多业务逻辑，增加了复杂度，导致越来越难扩展，间接拖累了业务开发。所以在重构时如果忽视了这些模块，只重构看起来有问题的上层模块，那就是治标不治本。

rainbow在我来之前有过几次重构的尝试，但传统主义的开发者因为担心基础模块改动可能带来的问题一直没有改动，导致前几次重构基本都是换汤不换药，问题还在，项目质量没有明显的提升。这次重构前刚好有一个小型项目要开发，我申请将它从rainbow项目中脱离出来独立开发，用它做试验重写了基础模块，下面我分前端和node两部分讲讲重写的基础部分。

### web前端
1. 重写了封装的请求模块，支持async/await，将错误情况手动throw到promise的catch中统一处理
3. 将前端路由从react-router3升级到了4，支持组件内配置路由，不依赖主页面的路由配置，保证组件独立性方便后面拆分

### node
1. 将koa1升级到koa2，支持async/await
2. 重写了请求模块，更方便地支持转发和组合请求
2. 因为node作为BFF层大部分请求都是转发java，将原先一个个注册的转发路由全部统一至`router.all('/redirect/*')`下
3. 重写了log模块，依赖`log4js`，分级别分时生成log文件，同时整理了log信息

## 业务架构探索

到这一步，基本非业务的内容都整理的差不多了，到这一步才真正开始处理棘手的业务模块问题，不过有了前面的工作，这边的难度也是减小了不少。我们再来回顾一下之前的重构目标：

> 这次重构的宏观目标就是把rainbow-red，rainbow-yellow，rainbow-blue多个项目合并成为rainbow-platform，统一维护功能模块，同时实现业务线的快速扩展。

可以看到，要实现这么一个统一的平台，肯定不能和原来一样以业务线划分项目，而是要弱化业务线的角色，将其作为一个变量，把重心放到功能模块上面，以功能模块为基础构建项目，功能模块中各个业务线feature的区分用模块封装的featureMap实现，这就是platform项目的业务线入口和功能库概念。

### 业务线入口 && 功能库

业务线入口在代码层的体现就是一份配置，代表这个业务线有哪些功能以及功能中的哪些feature，具体如下：
```js
const APP_CONFIG = [{
    name: 'red', // app名称
    features: [ // app功能模块
        'overview',
        'map',
        'geofence',
        {
            value: 'tide',
            options: { // options表示该功能的feature配置，具体实现封装在模块中
                lock: true,
                search: true
            }
        },
    ]
}, {
    name: 'blue',
    features: [
        'overview',
        'map', 
    ]
}
```

初始化时候直接读取这份配置即可生成包含对应功能的app入口，每个名字对应功能库中的一个功能模块，一般模块都是交给一个具体的开发负责，封装了模块各种feature，可以用featureMap选择开启或关闭。

### 弱化node层业务逻辑

node作为rainbow的BFF层，在项目里较多的应用就是承担了一个转发和合并请求的角色。之前在重写基础模块的时候我重写了node的请求模块，统一了node层的转发请求，其实目的就是为了弱化
node层的业务逻辑，因为合并请求的场景并不是特别多，如果转发统一的话，那么在开发时就不需要修改node层的代码，直接更新前端资源即可，这样在上线时也有较好的用户体验。

当然除了请求之外，node层还承担了其他业务，包括登录和文件下载。
1. 登录模块用的是公司内部的登录系统，我将其中的业务线配置放在了前端，这样的话，node层登录接口只需要接受不同的配置信息就可以实现各个业务线的登录。
2. 文件下载模块，主要是各种表格，我将`xlsx`库放在了前端，实现了前端的文件下载，避免了在服务器上进行文件操作，也是减少了很多复杂度。不过为了数据安全考虑，还是保留了一个下载的log接口，以便前端文件下载时记录用户信息。

当然，最后难免还是有部分业务请求要在node层进行拆分和聚合，这样的case也是在重构中大部分能挪到前端的都放在了前端。node层业务逻辑的减轻对于整体水平不高的团队来说还是有不错的减负效果。

## 正式迁移

改动在新项目试水了半个月，也是逐渐趋于稳定，终于要开始正式的迁移，真正到正式的这部分，探索工作也是基本完成了，剩下的都是乏味的工作量，包括整合各业务线模块，整合不同的技术栈，抽离模块耦合，检查回归问题等等，这些太过细节就不赘述了。这边主要还有一个工程问题：平台迁移作为内部升级的项目，要和业务开发并行，两者同步进行会产生大量冗余工作，还会增加业务开发难度。

解决方案首先是以不影响业务开发为前提，这边主要通过两方面进行处理：
1. 减少迁移持续时间 —— **将迁移分为前/中/后期**，各业务线逐个迁移。真正并行开发的只有中期。前期利用新项目测试迁移方案，包含框架升级基础、模块重写等。中期迁移时允许项目合并有部分重复代码，留给迁移后再处理。整体缩短迁移中期的持续时间，减少冗余工作。
2. 减少迁移同步的工作量 —— 用`git submodul`实现各个git项目代码共用，现有开发任务尽量都在submodule下进行，保证业务开发代码和平台迁移代码同步，减少迁移时的同步工作。

## 后期收尾

正式迁移之后，经过一个月的修修补补，项目也是逐渐趋于稳定，之后就是对迁移的遗留产物进行一个收尾，包括去除合并时的冗余代码，移除submodule。整合之后rainbow作为plarform也是更便于开发和管理了，就可以做一些之前很难进行的展望，包括和设计一起推动项目的主题统一，和产品一起讨论功能下线方案等等。

## 小结

在我实习和正式工作的时间中，和rainbow项目接触了接近两年，对这个已经接近20w代码的项目也是又爱又恨。由于种种原因，整个重构过程都是由我一人完成，痛苦和收获都很多，完成之后也算是了却了之前的一个心结。大部分程序员工作时难免要与遗留项目打交道，希望这篇文章能有那么一点帮助吧。
