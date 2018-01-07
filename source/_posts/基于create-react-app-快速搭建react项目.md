---
title: 基于create-react-app快速搭建react项目
tags:
  - reactjs
  - 工程
categories:
  - 前端
date: 2017-09-03 18:28:31
---

# 安装环境
本身安装了nodejs，npm或者yarn

    yarn global add create-react-app
或者
    
    npm install -g create-react-app
    
 # 创建项目
		create-react-app react-demo

![image](/images/1504432144489.png)

![image](/images/1504432202611.png)

# 运行
查看scripts 中可以执行等命令

		npm run-script
![image](/images/1504432536022.png)

我们可以猜想npm start 是开发模式下，启动服务热加载编译；npm run build 自然是编译程序

执行

		npm start 

会发现自动打开了浏览器http://localhost:3000/ 属于你等react web 已经出现了
可以打开App.js编写代码；
另外该开发套件本身没有包含react-router，你需要自己引入
	
    yarn add react-router
    yarn add react-router-dom
    
# 例子
https://github.com/su6838354/react-demo

# 分析
这时候整个react项目已经搭建好了，不需要知道任何的webpack等工程化技术，在node_modules/react-scripts 目录下我们可以发现该开发套件的真谛。
![image](/images/1504432860140.png)

> bin	 执行react-scripts 命令的入口
>config 根据环境变量和设置，webpack/环境/jest/polyfill等一些配置
>scripts 各种脚本，编译,测试，初始化，开发等，为package.json中定义的scripts和最初创建项目的命令服务
>temple 最初创建项目等时候 采用这套模版生成我们最初的项目

在未来项目配置不符合你等需求等时候，可以查看 https://github.com/facebookincubator/create-react-app/blob/master/packages/react-scripts/template/README.md
修改配置以满足需求；

比如 需要修改引入等js，css等相对路径，修改下自己等环境变量即可
export PUBLIC_URL='/home/static/'，当然还有其他方式比如修改package.json的homepage
参考
https://github.com/facebookincubator/create-react-app/blob/master/packages/react-scripts/template/README.md#adding-assets-outside-of-the-module-system

![image](/images/1504434422598.png)



# 参考
https://github.com/facebookincubator/create-react-app