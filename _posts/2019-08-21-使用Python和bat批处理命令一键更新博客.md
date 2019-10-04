---
layout: post
title: "使用Python和bat批处理命令一键更新博客"
description: "使用Windows的bat命令和Python的selenium库调用websever，实现了一键将新增的博客部署到GitHub和Gitee远程库以及自动化更新部署GiteePage"
categories: []
tags: [Essential Skill,Python,Efficient Application]
redirect_from:
  - /2019/08/21/
typora-root-url: ..
---

# 使用Python和bat批处理命令一键更新博客

使用jekyll+GitHub Page/GiteePage打造自己的博客已经一段时间了。除了写博客，还要把写完的文件更新到远程仓库，这样的更新操作只是简单重复性工作，早就想着能不能自动化更新。今天闲来无事（啊呸！就是不想写论文）就实现了一键就处理好所有操作。正好践行了之前听罗老师的课学到的工具意识。

其中的问题包括

1. 把md文件写作环境的image文件夹复制到博客网站的image文件夹

   > 这个我觉得是jekyll的Bug，写博客是在\_post文件夹，\_post文件夹又在网站的根目录下，其中的md文件引用图片时始终不能做到与网站的路径同步，也就是在本地引用图片很正常，但部署到网站之后就找不到文件路径在哪了。emmm...说的有点晕。
   >
   > 比如我现在的结构就是在\_post下新建一个image文件夹用来放图片。同样在根目录(与\_post同级的目录)也建一个image。这样的话在本地md文件中引用图片`/image/posts/2019-08-21/figure.png`。上传到远程仓库再经过编译之后，依然能根目录的image文件夹下找到图片。
   >
   > 有点绕，不好用语言描述出来，具体写博客的时候会遇到这个问题的！

2. 将文件修改提交给本地git仓库

3. 将修改同步到远程GitHub仓库和Gitee仓库

   > GitHub在国内访问太慢，所以我也在国内的码云Gitee部署了同样的博客

4. 更新部署GiteePage

   > 将新写的md文件上传到远程仓库后，GitHub会自动重新编译成网页，但Gitee自动编译部署要钱，免费的需要自己手动编译。所以我基于Python的selenium自动化的去更新部署

## bat文件代码

因为我的所有环境都在Window下，所以使用了bat批处理文件执行命令。

```bash
xcopy _posts\images images  /E /D /Y#2019-09-03修改，这句不用加了，下面介绍新的方法

git status

git add .

git commit -m "post"

git push origin master

python DeployGiteePage.py

pause
```

接下来一句一句的解释这些命令。

> #2019-09-03修改删除xcopy
>
> ### xcopy命令
>
> ```
> xcopy _posts\images images  /E /D /Y
> ```
>
> 这句的意思就是将_posts\images文件夹下的所有文件拷贝到images文件夹下。也就是实现上面说的第一个需求。
>
> 也就是将AlbertYZP.github.io\_posts\images中的所有文件复制到AlbertYZP.github.io\images中，因为AlbertYZP.github.io\_posts\images下的图片是给写博客时的md文件用的，AlbertYZP.github.io\images中的图片是生成静态网站后图片的存放地址
> 如果只放在AlbertYZP.github.io\images中，md引用文件的链接会混乱。造成生成的静态网站无法正确加载图片
> 所以使用这个方法，使引用图片的链接在本地与静态网站中一致，所以需要将在本题编写博客用的图片复制到静态网站调用图片的文件夹中
>
> xcopy就是复制文件夹的命令
> /E是复制所有子目录，包括空目录
> /D是复制最新的文件。为了不复制已存在的文件的目的
> /Y是禁止询问

为了解决问题1，将Typora的根目录设为`AlbertYZP.github.io`即可(注：Typora是一个优秀的开源markdown编辑器，强烈推荐)

只需要加一行代码即可`typora-root-url: ..`即为：

```shell
layout: post
title: "使用Python和bat批处理命令一键更新博客"
description: "使用Windows的bat命令和Python的selenium库调用websever，实现了一键将新增的博客部署到GitHub和Gitee远程库以及自动化更新部署GiteePage"
categories: []
tags: [Essential Skill,Python]
redirect_from:
	- /2019/08/21/
typora-root-url: ..
```

这样就直接在md文件中直接调用`/images/posts`中的文件了，不用像我之前自己想的方法那样再维护一个文件夹了。

### git命令

```bash
git status#查看仓库状态，其实没什么用，但提交之前查看状态是个好习惯

git add .#把所有修改的文件提交到缓冲区

git commit -m "post"#把缓冲区的文件放到仓库。并注释是post，其实我就是图省事，就注释了个post，如果其他地方提交的时候最好写清楚做了哪些修改

git push origin master#把本地仓库提交到远程仓库
```

这四句常用git的人就非常熟悉了，但注意把git的bin文件夹放到环境变量里。要不然git命令没法执行。

这里就又牵扯到了将仓库同时提交到GitHub仓库和Gitee仓库两个仓库的问题了。

首先就是`git remote add origin ***.git`x先设置一个名为origin的远程仓库（什么名都行），然后就可以看看当前连接的仓库有哪些`git remote -v`

![](/images/posts/2019-08-21/git1.png)

然后执行`git remote --seturl origin 另外一个.git`也就是把另外一个远程仓库也添加到origin 下。然后再查看就可以看到

![](/images/posts/2019-08-21/git2.png)

有两个push的远程仓库。这就可以了，直接运行`git push origin master`就可以同时上传到这两个远程仓库了。

### python命令

这个也是要注意把Python添加到环境变量，`python DeployGiteePage.py`就是用Python执行DeployGiteePage.py中的脚本代码。DeployGiteePage.py中的代码下面会说到。

### pause命令

这个就是让执行bat的命令行窗口在执行后不要立即关闭。其实没有实际用处，我就是看不惯他不搭理我就自己关了。执行这个命令他会出现学C语言的时候最熟悉的`按任意键继续...`。哈哈哈，就不让他直接把窗口关了。然后随便按一个键盘就关了。

其实也没有那么随便，主要是怕程序执行出现问题，不让他关的话你可以翻看一下上面的命令的执行情况。

## Python脚本

```python
import time
from selenium import webdriver

#option = webdriver.ChromeOptions()
#option.add_argument('headless')#option 是用来设置不弹出chrome界面的

url="https://gitee.com/zpyang/zpyang/pages"
driver = webdriver.Chrome()

#打开链接
driver.get(url)

#下面就是登陆自己的账号。登陆账号之后才能部署
#找到右上角登陆的按钮并点击
driver.find_element_by_xpath('//*[@id="git-nav-user-bar"]/a[2]').click()


#输入用户名
driver.find_element_by_xpath('//*[@id="user_login"]').send_keys("**********")

#输入密码
driver.find_element_by_xpath('//*[@id="user_password"]').send_keys("**********")


#点击登录
driver.find_element_by_xpath('//*[@id="new_user"]/div[2]/div/div/div[4]/input').click()

#等加载好，如果不等加载好可能会点击不上
time.sleep(2)

#点击更新按钮
driver.find_element_by_xpath('//*[@id="pages-branch"]/div[7]').click()


#处理提示框点确定
alert = driver.switch_to_alert()
alert.accept()

#等待部署完成

time.sleep(30)

driver.quit()

```

这个就不一句一句的解释了，感兴趣了自己学习Python爬虫，会的一看就懂，不会的就不是我一句两句能解释清楚的。每个函数的功能我已经卸载上面了。

参考[selenium官方文档](https://www.seleniumhq.org/docs/03_webdriver.jsp#introducing-the-selenium-webdriver-api-by-example)



最后将这两个bat、py文件放在根目录即可

![](/images/posts/2019-08-21/dir.png)

写完之后直接双击这个bat文件。就等着他执行完毕吧。主要是Python脚本那执行有点慢。因为调用了webdriver和网络原因，但是一分钟之内肯定可以执行完。



这样的话就不用自己复制粘贴图片，执行命令上传文件，部署更新博客了。完美！也可以节省很多时间呢！

> 哈哈，写代码一个小时左右（因为在此之前没用过selenium），加上写这个博客大概花了两个小时。成功荒废了半个下午