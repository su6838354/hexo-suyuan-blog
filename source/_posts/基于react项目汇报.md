---
title: 基于react项目架构
tags:
  - 项目
  - reactjs
categories:
  - 个人项目
date: 2017-12-16 22:45:53
---

这是一个关于公司项目的前端技术路线的描述，内容不会涉及具体代码。

项目提供了看板、工作流、统计分析等主要的项目管理工具，同时前端接入了环境管理，持续集成，case管理等服务页面。

前端基于react技术栈，采用webpack3.x进行工程化管理，搭配常规的babel编译工具。基于chrom+react-devtools+redux-tools搭建开发工具。

![](/images/lu_1/lu_1.001.jpeg)
---
![](/images/lu_1/lu_1.002.jpeg)
---
![](/images/lu_1/lu_1.003.jpeg)
---
![](/images/lu_1/lu_1.004.jpeg)
---
![](/images/lu_1/lu_1.005.jpeg)

---
![](/images/lu_1/lu_1.006.jpeg)

- Webpack+babel+es6为核心的前端工程
- 基于react的Jsx页面构建
- 单页面app模式，react-router路由管理
- Redux的单向数据流，action事件驱动，reducer纯函数处理器,
- 网络代理抽象，通用网络处理
- 性能优化：imtuable state；网络缓存；优化资源传送；
- Reudx-saga处理异步事件
---
![](/images/lu_1/lu_1.007.jpeg)
---
![](/images/lu_1/lu_1.008.jpeg)
---
![](/images/lu_1/lu_1.009.jpeg)
