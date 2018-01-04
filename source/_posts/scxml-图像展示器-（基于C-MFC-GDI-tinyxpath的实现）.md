---
title: scxml 图像展示器 （基于C++ MFC GDI tinyxpath的实现）
tags:
  - C++
  - scxml
categories:
  - 个人项目
date: 2015-01-13 22:50:12
---

Scxml是w3c出来的基于状态机的对话脚本语言标准，具体内容可以谷歌到，这里讲述自己开发的一个把scxml转化为可交互图形的程序。

源代码上传到了[github](https://github.com/su6838354/scxml_exec)
 
 ### 整体设计
基本原则是把具有状态机关系的xml语言转换为矩形、矩形之间的线、矩形的子父级关系。

整个模块由下而上分为 5部分

- 1.Scxml 脚本
- 2.Parser 层（依赖Tinyxpath）
- 3.Model 层
- 4.Layout 层 (Model转化为虚拟图形对象)
- 5.View 图形（MFC和GDI [ Gdiplus::Graphics]实现 ）

Parser层会通过tinyxpth解析scxml脚本并产出Modal对象，并对上层提供getState，getTransitions，GetFinals等接口，接口之间的参数类型就是Model层定义的；

Layout层获取所有的state和transition，然后转化为虚拟图形对象ScxmlRectangle和ScxmlLine等；

View层通过MFC实现，将虚拟图形对象进行描绘和渲染

### 包设计
下面这个是包设计图，UI从LayOut中获得图形信息画图，LayOut从IGetScxmlObject获得解析信息，
![](/images/cpp/scxml_1.png)
Parser层通过Iread可以读取到scxml文件中的元素，ModelFactory将获取的元素转换为自定义对象，提供IGetModel给layout

### 用例图
下面是layout层和parser层的用例图，
- layout用于描述自动状态机布局的过程，包含从scxml_parser模块获取对象，根据对象内容计算出整个图形布局
	- Rectangle Scxml   用于输出图形的中心点、宽度、高度，线条起始点等内容
	- GetScxmlObject   解析scxml对象，生成矩形和有向线段
- Parser描述从scxml格式解析成对象的过程，以及和外部模块之间的关系，
	- Read scxml     主要用于按照需求读取scxml文件内容，其调用tinyxpath模块执行自定义的xpath语法
	- create model          将读取的内容构建成对象
    
   ![](/images/cpp/scxml_2.png) 
    
### Layout的类图如下

- Line        线，包含起始点和终点
- Rectangle       矩形，包含中心点、宽度、高度
- ScxmlLayout         包含所有矩形和线条的数据，拥有计算整个图形布局的方法
      
  ![](/images/cpp/scxml_3.png) 
    
 ### Layout时序图

- 根据scxml对象，执行LayOut算法，生成图形信息
- 调用scxml_parser模块，获取自定义的scxml对象，生成相应的图形内容，执行布局算法，输出图形信息
    
   ![](/images/cpp/scxml_4.png) 
   
### Parser 类图如下

- Xtinyxpath     调用Xpath语言查找scxml元素
- ScxmlParser          调用tinyxpath获取元素，封装为scxmlobject对象
  
     ![](/images/cpp/scxml_5.png)
     
     
###  Parser时序图：

scxml文件解析过程     调用tinyxpath模块，实现在C++中内嵌使用xpath语言，按要求获取scxml元素，转化成自定义的对象，用于layout画图

  ![](/images/cpp/scxml_6.png)
  
  
 ### 展示效果如下

下面为一个相对复杂的scxml，包含了并行、多层的嵌套关系

  ![](/images/cpp/scxml_7.png)