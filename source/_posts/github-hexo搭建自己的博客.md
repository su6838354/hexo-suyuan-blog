---
title: 搭建自己的个人博客
tags: ['nodejs','hexo']
categories: ['工具']
date: 2017-08-25 00:02:42
---

# github上建立个人主页
- 到github上建立个人主页仓库
	![image](/images/1503591079880.png)

	![image](/images/1503591185611.png)
>注意仓库名字必须是你的 `个人用户名.github.is`

- 这时候你可以访问你的github个人主页了
![image](/images/1503591377977.png)
>参考	<https://pages.github.com/>

# hexo写博客

- 本地有nodejs环境，安装hexo-cli 
			npm install -g hexo-cli
>参考[hexo中文](https://hexo.io/zh-cn/docs/)

- 用hexo建立自己的博客项目<https://hexo.io/zh-cn/docs/setup.html>
			hexo init yourname
			cd yourname
			npm install
                   
- hexo中配置github账户，可自动发布；以文本编辑器打开_config.yml文件，并滚动到最下面添加如下配置信息，如又该字段，修改为你自己的github仓库

      # Deployment
      ## Docs: https://hexo.io/docs/deployment.html
      deploy:
        type: git
        repo: https://github.com/su6838354/su6838354.github.io.git
        branch: master
- 预览hexo中的博客
			hexo s
   ![image](/images/1503592345617.png)
   
   ![image](/images/1503592386857.png)

- 编译发布
			hexo clean
			hexo g
			hexo d
此时会提示要求你输入github 的账户密码，使用邮箱为账户的正常输入邮箱即可

- 到githu仓库可以看到提交的编译的博客文章，打开自己的主页<https://su6838354.github.io>也可以看到初始化的博客

# hexo换主题
- 换个next主题<https://github.com/iissnan/hexo-theme-next>

# hexo写博客
- 用hey，<https://github.com/nihgwu/hexo-hey>图片可以直接截图后粘贴到编辑器中，改下url中到域名为下图

	![image](/images/1503592858550.png)
    
    写完后正常 hexo clean， hexo g， hexo d

# 域名绑定
不想用https://su6838354.github.io  ，想换成http://blog.suleyan.com/
- 到阿里云申请个域名

- 解析域名到github到个人主页上
![image](/images/1503593100609.png)
![image](/images/1503593191977.png)

- github个人主页添加CNAME配置
在hexo项目的source下面添加CNAME
![image](/images/1503593344161.png)
重新编译发布
再次访问https://su6838354.github.io 会自动转到http://blog.suleyan.com/
>注意cnd域名解析可能又延迟生效



参考
http://xiaopingblog.cn/2016/04/08/untitled-1460084538799/
http://www.worldhello.net/gotgithub/03-project-hosting/050-homepage.html
https://segmentfault.com/a/1190000003101692











