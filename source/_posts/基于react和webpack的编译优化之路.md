---
title: 基于react和webpack的编译优化之路
tags:
  - reactjs
  - 项目
categories:
  - 前端
date: 2017-09-03 22:09:29
---

本文是针对公司项目的一个前端编译方面的迭代记录，主要目的是减少编译时间，优化开发体验

### 背景
最初项目前端工程化比较初级，通过webpack+jsx-loader对jsx进行处理，通过gulp构建clean和build任务；并且他是多页面多入口，每个页面都需要进行webpack编译
```
gulpfile.js

    gulp.task('clean', function (cb) {
      var rimraf = require('rimraf');
      rimraf('./build/', cb);
    });
    
    gulp.task('build',['clean'], function() {
        var compSrc = './src/html/*',
            compDst = './build/html/';
    
        gulp.src(compSrc)
            .pipe(gulp.dest(compDst));
    
        gulp.src('')
            .pipe(webpack(webpackConfig))
            .pipe(gulp.dest(''));
    });
    
webpack.config

    module.exports = {
        entry: {
            //'lts-irelease2-nav':'./src/jsx/lts-irelease2-nav.jsx',
            'lts-irelease2-create-project':'./src/jsx/lts-irelease2-create-devplan.jsx',
            'lts-irelease2-list-project':'./src/jsx/lts-irelease2-list-devplan.jsx',
            ...
        },
        output: {
            filename: './build/js/[name].js'
        },
        //devtool: 'inline-source-map',
        module: {
            loaders: [
                { test: /\.jsx$/, loader: 'jsx-loader' }
            ]
        },
        resolve: {
            extensions: ['', '.js','.jsx']
        }
    };
```
问题：

 1. 不支持es6等最新语法
 2. 不支持css,less等编译
 3. 没有实现SPA模式
 4. 前后端没有分离，后端还是通过模板渲染做了一次无用功
 5. 自研的组件库存在一些问题，state 状态管理混乱，使用semantic作为UI层，组件封装成本高，react和semantic 的js事件之间的时序存在问题 等，随着react、webpack等升级，没有精力来维护自研组件
 
### 分析

- 打包过程分析
我们知道，webpack 在打包过程中会针对不同的资源类型使用不同的loader处理，然后将所有静态资源整合到一个bundle里，以实现所有静态资源的加载。webpack最初的主要目的是在浏览器端复用符合CommonJS规范的代码模块，而CommonJS模块每次修改都需要重新构建(rebuild)后才能在浏览器端使用。
那么， webpack是如何进行资源的打包的呢？总结如下：

	-	对于单入口文件，每个入口文件把自己所依赖的资源全部打包到一起，即使一个资源循环加载的话，也只会打包一份
	- 对于多入口文件的情况，分别独立执行单个入口的情况，每个入口文件各不相干
我们的项目使用的就是多入口文件。在入口文件中，webpack会对每个资源文件进行配置一个id，即使多次加载，它的id也是一样的，因此只会打包一次。
但是每个入口都需要做一遍打包流程，本身就很不合适。

- 如何定位webpack打包速度慢的原因
首先需要定位webpack打包速度慢的原因，才能因地制宜采取合适的方案。我们可以在终端中输入：
```
$ webpack --profile --json > stats.json
```
然后将输出的json文件到如下两个网站进行分析
https://github.com/webpack/analyse
http://alexkuz.github.io/webpack-chart/

### 改进路线

- nodejs升级到8.3.X

- 使用yarn而不是npm

- 去除gulp,直接采用webpack

- webpack中有选择的编译
```
include: path.join(__dirname, 'src'),
exclude: /node_modules/,
```

- webpack 编译支持react,es6和一些新语法，比如static class，解构语法
其中加入babel编译会增长编译时间

- eslint开发某些路径下的代码

- webpack 自动集成html和js
自动在html中加入script tag， 增量发版，

```
const HtmlWebpackPlugin = require('html-webpack-plugin');
new HtmlWebpackPlugin(
{
  filename: path.join('html', `${realName}.html`),
  template: path.join(realHtmlPath, `${name.split('.')[0]}.html`),
  hash: true,
  inject: true,
  chunks: ['babel-polyfill', realName, 'common'],
  chunksSortMode: orderByList(['common', 'babel-polyfill']),
}
);
	
```
- commonjs 提取公共js
保证每个jsx里面有公共部分
```
new webpack.optimize.CommonsChunkPlugin({name: "common", filename: "js/common.js"}),
```
这个改为后，每个js文件会变小，会有个比较大的common.js
此时编译可能很慢了

- 自动入口分析
通过js获取html下的入口，自动引入对应的jsx，不需要修改webpack.config.js
>增加html后，nodejs自动扫描到对应的jsx文件载入webpack入口，无需再修改webpack.config.js
>根据编译参数，支持开发环境和生成环境 两种配置
>指定参数后可以编译单文件，提高开发过程中的编译效率
>支持版本号更新

- 提取css
```
const ExtractTextPlugin = require("extract-text-webpack-plugin");
new ExtractTextPlugin({filename:"styles.css", allChunks:true}),
```

- spa 单页app模式升级(工作量比较大)
只保留一个html,原本的html代码转为jsx
react-router管理前端路由，后端路由暂时不去除

- 添加watch,支持修改代码后自动编译

- webpack添加dashboard 编辑进度和性能可视化分析

- 生产环境优化
这个webpack2 的命名参数-p 可以自动实现 
压缩，去重，treeshark，合并、去错，transform-runtime支持浏览器的模拟es6环境

- webpack2升级为webpack3，react 升级到15.X

- CND引入
react等基础包不需要编译，通过引入cnd的方式
webpack的 externals 特性
注意区分开发和线上环境引入不同包

- 支持sourcemap；
- sourcemap 建议用轻量级，因为该功能影响编译速度


- 多线程编译,HappyPack多进程处理loader编译；

- 热加载编译，增量编译；处理好redux 和redux-saga的对应逻辑

- 添加antd组件库，antd按需引入



参考
https://github.com/hawx1993/tech-blog/issues/3