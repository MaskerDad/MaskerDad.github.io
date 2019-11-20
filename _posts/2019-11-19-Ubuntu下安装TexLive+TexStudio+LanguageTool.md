---
layout: post
title: "Ubuntu下安装TexLive+TexStudio+LanguageTool"
description: ""
categories: [LaTeXManuals]
tags: [LaTeX]
redirect_from:
  - /2019/11/19/
typora-root-url: ..
---


>今天在办公室电脑上安装好了ubuntu系统，其中需要配置写论文的环境。特此记录一下。

TexLive和TexStudio的配置完全按照的 [Install-LaTeX.pdf](https://github.com/OsbertWang/install-latex/releases/download/v2019.11.13/Install-LaTeX.pdf), 感谢大佬的奉献，此处不再赘述。

LanguageTool是一个开源的语法拼写检查工具，写论文的时候非常好用，所以需要单独配置一下。Windows下的配置可完全按照[对TexStudio配置拼写和语法检查LanguageTool功能]( https://blog.csdn.net/yinqingwang/article/details/54583541 )。

Ubuntu下的配置不太一样。参考[Github源代码里的README.md]( https://github.com/languagetool-org/languagetool )在命令行执行

```shell
curl -L https://raw.githubusercontent.com/languagetool-org/languagetool/master/install.sh | sudo bash
```

脚本会自动下载java环境和languagetool。等着就行（languagetool的下载很慢，将近一个小时，服务器在国外）

> 全部完成后

进入当前文件夹下的LanguageTool-4.7-stable文件夹，看到languagetool.jar等文件就好了。

在LanguageTool-4.7-stable下执行

```shell
java -cp languagetool-server.jar org.languagetool.sever.HTTPServer -p 8081
```

得到这个就没问题了，很成功

> 2019-11-20 08:12:41 +0000 Not setting up database access, dbDriver is not configured
> 2019-11-20 08:12:41 +0000 WARNING: running in HTTP mode, consider using org.languagetool.server.HTTPSServer for encrypted connections
> 2019-11-20 08:12:41 +0000 Setting up thread pool with 10 threads
> 2019-11-20 08:12:41 +0000 Starting LanguageTool 4.7 (build date: 2019-09-28 10:09, 64f87c1) server on http://localhost:8081...
> 2019-11-20 08:12:41 +0000 Server started

接下来

就需要配置TeXstudio了

`Options->Configure TeXstudio->Language Checking`

![texstudio](/images/posts/2019-11-19/texstudio.png)

绿色圈住的是默认填好的

> 注：上面三个是英文的拼写检查，需要中文的自行搜索

红色部分填好`languagetool-server.jar`的绝对路径即可。

这时候已经配置好了，检查一些可不可以

重启TeXstudio，新建一个文件（在文件编辑状态才会触发打开LanguageTool）

`Help->Check LanguageTool`

> which java: /usr/bin/java
>
> JAVA: java -version
> openjdk version "11.0.4" 2019-07-16
> OpenJDK Runtime Environment (build 11.0.4+11-post-Ubuntu-1ubuntu218.04.3)
> OpenJDK 64-Bit Server VM (build 11.0.4+11-post-Ubuntu-1ubuntu218.04.3, mixed mode, sharing)
>
> Real-time checking is enabled.
> Grammar checking is enabled.
>
> Tries to start automatically.
>
> LT current status: working
>
> LT-URL: http://localhost:8081/v2/check

主要看`LT current status: working`这一行，如果是working就是正常了，如果是`error`证明不正常

注意排查

1. java环境是否添加到环境变量，刚才安装的时候一般会自己弄好的，小概率出现这种问题。如果没办法，可以把java的绝对路径放到设置中`java`那一行
2. languagetool-server.jar的绝对路径是否正确