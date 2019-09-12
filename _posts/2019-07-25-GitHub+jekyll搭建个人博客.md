---
layout: post
title: "GitHub+jekyll个人博客搭建"
description: ""
categories: []
tags: [Essential Skill]
redirect_from:
  - /2019/07/25/
typora-root-url: ..
---

## GitHub+jekyll个人博客搭建

今天又听老师说，分享知识特别重要，不仅是社会价值的体现，更是加深了自己对其的理解。之前我也是为了这个目的，写了一段时间博客，但是觉得越来越不方便，就放弃了。最近又看到了这个个人博客的搭建，既可在GitHub Page上部署，以后有服务器了也可以部署在服务器上，所以我就再次准备写博客。

此篇文章记录搭建的步骤
### 第一步：选一个自己感觉好看的主题
这两个网址都有好多特别漂亮的主题，耐心选一下即可。

http://jekyllthemes.org/
https://jekyllthemes.dev/

我选的是http://jekyllthemes.org/themes/leopard/

![](/images/posts/2019-07-25/leopard.png)

点Homepage即可看到其在GitHub上的源代码。

```shell
git clone https://github.com/leopardpan/leopardpan.github.io.git
```

> 注：git的使用可参见[廖雪峰的git教程](https://www.liaoxuefeng.com/wiki/896043488029600)，可以带你快速的学会git的使用以及与GitHub的配合使用。以后的博客管理也会特别方便。

### 第二步：搭建本地jekyll环境用来调试博客

按照Ruby中国官网的[jekyll安装教程](https://ruby-china.org/wiki/install_ruby_guide)，即可正常的安装好ruby以及jekyll。我没有学过Ruby，所以也仅仅是按部就班。

1.我是在CentOS6.9虚拟机环境下配置的。一切正常，就是有点慢。

```shell
[albert@localhost AlbertYZP.github.io]$ rvm -v
rvm 1.29.9 (latest) by Michal Papis, Piotr Kuczynski, Wayne E. Seguin [https://rvm.io]
[albert@localhost AlbertYZP.github.io]$ ruby -v
ruby 2.3.0p0 (2015-12-25 revision 53290) [x86_64-linux]
[albert@localhost AlbertYZP.github.io]$ jekyll -v
jekyll 3.8.5

```

2.在AlbertYZP.github.io文件夹内启动jekyll，它会自动读取配置文件的。

```shell
$ jekyll serve
```

但出现了报错

```shell
Could not find proper version of jekyll (3.8.5) in any of the sources Run `bundle install` to install missing gems
```

意思就是目前版本的jekyll与gemfile中的依赖版本不一样。需要执行bundle install来解决依赖问题。那就执行它。

```shell
$ bundle install
```

但有出现了好长时间没一点动静的问题，并且怎么修改gems源都没用。试了好多种方法，在[一个大佬的笔记](https://cj1406942109.github.io/2018/11/17/bundle-install-no-response/)中找到一个解决方法。顺利解决问题

将bundle的源修改了，而不是去修改gems的源，虽然迷之又迷，但是问题还是顺利解决了，有动静了，并且很快就安装好了

```shell
$ bundle config mirror.https://rubygems.org https://gems.ruby-china.com
```

随后便可再次启动jekyll

```shell
[albert@localhost AlbertYZP.github.io]$ jekyll serve
Configuration file: /mnt/hgfs/code/AlbertYZP.github.io/_config.yml
            Source: /mnt/hgfs/code/AlbertYZP.github.io
       Destination: /mnt/hgfs/code/AlbertYZP.github.io/_site
 Incremental build: disabled. Enable with --incremental
      Generating... 
                    done in 0.678 seconds.
RubyDep: WARNING: Your Ruby is outdated/buggy.
RubyDep: WARNING: Your Ruby is: 2.3.0 (buggy). Recommendation: upgrade to 2.3.1.
RubyDep: WARNING: (To disable warnings, see:http://github.com/e2/ruby_dep/wiki/Disabling-warnings )
 Auto-regeneration: enabled for '/mnt/hgfs/code/AlbertYZP.github.io'
    Server address: http://127.0.0.1:4000
  Server running... press ctrl-c to stop.

```

运行成功

在虚拟机中本地浏览器输入http://127.0.0.1:4000即可访问博客主页(在同一个局域网下通过IP地址也可以访问，以后可以部署在自己的公网服务器上），我没有截图。但效果与模板作者的demo一模一样。

### 第三步修改博客配置文件

jekyll的文件组织结构为

- _includes：其中包括了很多页面文件，修改其中的文件可以修改博客页面细节。
  - JB：一般用不到
  - styles：里面有一个css样式文件，没学过，所以没动
  - comments.html：评论区配置文件，可以给博客加上评论的功能
  - external.html：一般用不到
  - footer.html：博客底部显示的内容，包括各种账号，版权，访问量等东西的设置。
  - head.html等其他的，我都没改。术业有专攻，我没学过前端。所以。。。我只能自己用模板。
- _layouts：排版布局设置，对于用模板的人也用不到

- _posts：这个很重要，所有自己写的博客放在这个文件中，按照命名0000-00-00-***.md命名（e.g,2019-07-25-GitHub+jekyll搭建个人博客.md）即可。

- _site这个，据我观察，是jekyll编译所有配置文件后生成的静态网站站点。
- css、js：不会用，保持不动就行
- images：这个存放了网站中用到的所有图片，修改这里面的一些图片即可生成自己独一无二的界面
  - payimg：存放的打赏有关的图片
  - posts：存放的自己的博客md文件需要的图片源文件
  - readme：readme文件中引用的图片源文件
  - avatar.jpg：博客中间的头像
  - background-cover.jpg：博客中间头像后面的背景图
  - favicon.png：标签栏中显示的小图标

- _config：jekyll的配置文件，等会的配置全部在这里面设置，非常重要
- about.md：博客中"关于我"界面中显示的内容
- 其他文件都不太需要修改。

#### 1、设置_config文件

作者已经注释了很多了，我再加一点注释而已。可参考简书里面的一个教程[利用 GitHub Pages 快速搭建个人博客](https://www.jianshu.com/p/e68fba58f75c)。比较全面，但有些细节没有写出来。

其中的评论区设置和统计访问我会另外说。

```xml
# Basic
#头像下面的标题
title: 杨展鹏的个人博客
#头像下面的副标题
subtitle: 整理、分享知识的殿堂
#头像下面的描述
description: 欢迎来到我的个人博客~
# 头像里面的标题
avatarTitle: AlbertYZP
# 头像里面的描述
avatarDesc: CS Pre-postGraduate
#也就是你博客的域名，没有自己买域名就直接设置GitHub默认的域名就好
url: "http://albertyzp.github.io"

# Comment 评论区配置，disqus和gitment都依赖国外的服务器，都比较慢，但也没办法。有言我没找到官网
comment:
    #disqus: 
    gitment: AlbertYZP #需要修改comments.html中的client_id和client_secret
    # uyan:   # 有言id,登录有言官网申请

# Social，博客底栏的联系方式
social:
    #weibo: 
    github: AlbertYZP
    # zhihu: 
    # juejin: 
    #jianshu: 
    # twitter: 
    mail: albert.yzp@gmail.com


# 百度统计 
# 使用博客模板请去掉或者更换 id 
baidu:
    id: 02825e06557bac1812d33b39b1c36868
# Google Analytics
# 使用博客模板请去掉或者更换 id
#ga:
#    id: xxxx  
#    host: auto  


#下面的如果没看懂就不用改即可
#链接的格式
permalink: /:year/:month/:title/
# Format
highlighter: rouge

textColor: #FF0000

# supported colors: blue, green, purple, red, orange or slate. If you need clear, leave it empty.
cover_color: clear

# 博客右上角的按钮设置
blog_button:
    title: 博客主页

# Navigation buttons in the front page.s
nav:
    - {title: 所有文章, description: archive, url: '/archive'}   
    - {title: 标签, description: tags, url: '/tags'}      
    - {title: 关于我, description: about, url: '/about'}      
    

# Pagination，翻页的设置
plugins: [jekyll-paginate,jekyll-sitemap]
paginate: 10
paginate_path: "page/:num/"

```

#### 2、评论区设置

这个模板只支持了三种评论插件。disqus服务器在国外，太慢了，gitment是利用的GitHub的issue功能，也比较慢。uyan已经倒下了。

所以我就选用了gitment。

可参考[Gitment](https://blog.csdn.net/anttu/article/details/77688004),写的很详细，也把很多错误指出来了，在此不再赘述，我也花了很长时间才把评论区调试正常。

友情提示，加载的很慢！

#### 3、访问统计

在国内，谷歌的肯定不靠谱，毫不犹豫选择百度。

https://tongji.baidu.com/登录后右上角我的报告，新赠网址

![](/images/posts/2019-07-25/leopard.png)

![](/images/posts/2019-07-25/baidutongji.png)

注意域名与网站首页的输入格式即可。

确定后会生成一段代码

![](/images/posts/2019-07-25/daima.png)

将问号后面的一串字符复制到_config文件中的id:后面即可。随后即可看到访问报告。



将所有设置完毕后得到我自己的博客如下(在虚拟机中调试的)

![](/images/posts/2019-07-25/albertyzp.png)

### 第四步：将所有文件部署到github

1. ssh-keygen -t rsa -C "youremail@example.com"

   生成SSH密钥部分见[廖雪峰的git教程](https://www.liaoxuefeng.com/wiki/896043488029600)。

2. git remote add origin git@github.com:AlbertYZP/AlbertYZP.github.io.git

   一定别直接push，先把远程库修改为自己github上的仓库！(先在github上新建好名为yourname.github.io的仓库)

3. git add .

   将所有修改过的文件放进缓冲区

4. git commit -m "init"

   将修改过的文件从缓冲区提交给仓库，-m是添加备注

5. git push -u origin master

   最后便可以push到仓库

### 第五步 享受成功的喜悦吧

嘻嘻嘻

![](/images/posts/2019-07-25/end.png)

花了我总共快一天的时间才完全搭建好的。主要是遇到了各种各样的小问题，一个一个解决挺费时间的。

