---
layout: post
title: "Windows下Sublime Text+ LaTex的配置"
description: ""
categories: [LaTeXManuals]
tags: [LaTeX]
redirect_from:
  - /2020/07/22/
typora-root-url: ..
---

在此之前我用过TeXStudio和VS code了（注意编译器与编辑器的区别）。但是TeXStudio界面有点难看，写代码没感觉。并且我要写书用的模板，在TeXStudio下总是编译有问题（虽然跟编辑器无关，应该是编译器顺序的问题，但我没能力排查）。VS code界面超级好看，我也配置好了，但还是用Latex插件辅助编译有问题。然后又转战到Sublime Text，配置完就编译很正常。。。。玄学。。。



所以记录一下Sublime Text的配置。

* Kramdown table of contents
{:toc .toc}
主要参考[https://noc-leung.gitbook.io/techdocs/](https://noc-leung.gitbook.io/techdocs/)

## 一、 配置Sublime

1. 下载最新版Sublime [https://www.sublimetext.com/](https://www.sublimetext.com/)
2. 在Sublime上安装**Package Control**[https://packagecontrol.io/installation](https://packagecontrol.io/installation)
   1. 在线安装：按住快捷键`Ctrl+Shift+P`或者点击`Tool->Command Palette... `进入控制台再输入`Install Package Control`，点击`Enter`确认。这个方法。。。很可能安装失败。需要翻墙。
   2. 推荐使用手动安装方法，在[https://packagecontrol.io/installation](https://packagecontrol.io/installation)的Manual下，过程非常详细。先下载[Package Control.sublime-package](https://packagecontrol.io/Package Control.sublime-package)包; 再点击`Preferences -> Browse Packages…`，在打开的文件夹的上一层文件夹下找到`Installed Packages/`文件夹，把刚才下载的包文件复制进去重启Sublime即可。
3. 安装**LaTeXTools **
   	1. 安装好Package Control就能安装LaTeXTools 了。继续按按住快捷键`Ctrl+Shift+P`或者点击`Tool->Command Palette... `进入控制台。
    	2. 输入`install`，看到`Package Control: Install Package`。再按`Enter`, 可能需要等一会儿，别急。等他加载出来一个输入框。
    	3. 在输入框里输入`latextools`再安装即可。重启Sublime生效

此时Sublime是配置好了，再点击`Preferences -> Settings`, 在User设置里加上一行

```
"keep_focus": false
```

目的是为了之后编译latex后让pdf显示在前面

## 二、 安装TeXLive

TeXLive安装过程完全按照的 [Install-LaTeX.pdf](https://github.com/OsbertWang/install-latex/releases/download/v2019.11.13/Install-LaTeX.pdf)即可, 感谢大佬的奉献，此处不再赘述。

安装完后再配置**LaTeXTools **

1. 点击`Preferences -> Package Settings ->LatexTools ->Setting-User`进入LaTeXTools 的设置。

2. 安装SumatraPDF用来打开编译生成的PDF，以及之后进行正反向搜索。[https://www.sumatrapdfreader.org/free-pdf-reader.html](https://www.sumatrapdfreader.org/free-pdf-reader.html)

3. 在打开的文件里按`Ctrl+F`查找关键词`windows`, 找到windows平台下的相关配置。

   ```c
   	"windows": {
   		// Path used when invoking tex & friends; "" is fine for MiKTeX
   		// For TeXlive 2011 (or other years) use
   		// "texpath" : "C:\\texlive\\2011\\bin\\win32;$PATH",
   		"texpath" : "D:\\Software\\texlive\\2020\\bin\\win32;$PATH",//Latex编译器的 的路径，按照TeXLive的安装路径即可。
   		// TeX distro: "miktex" or "texlive"
   		"distro" : "texlive",//用的miktex还是texlive
   		// Command to invoke Sumatra. If blank, "SumatraPDF.exe" is used (it has to be on your PATH)
   		"sumatra": "D:\\Software\\SumatraPDF\\SumatraPDF.exe",//SumatraPDF的路径
   		// Command to invoke Sublime Text. Used if the keep_focus toggle is true.
   		// If blank, "subl.exe" or "sublime_text.exe" will be used.
   		"sublime_executable": "D:\\Software\\Sublime Text 3\\sublime_text.exe", //sublime_text的路径
   		// how long (in seconds) to wait after the jump_to_pdf command completes
   		// before switching focus back to Sublime Text. This may need to be
   		// adjusted depending on your machine and configuration.
   		"keep_focus_delay": 0.5
   	},
   ```

   务必配置好上面的路径。

## 三、使用Sublime+Latex

### 1.编译

打开`.tex`文件后按`Ctrl+B`或者点击`Tools -> Build`进行编译。注意选择编译器是xelatex还是pdflatex哦。

### 2. 正向搜索

也就是从LaTex代码跳转到PDF文件的位置。

在配置好上面SumatraPDF路径的情况下点击`Ctrl+L, J`

注意！！！这里是`Ctrl+L`和`J`快速先后连按。我这个就纠结了好长时间。怎么按`Ctrl+L`都跳转不了，还以为是`Ctrl+L`或`J`都行呢。

### 3. 反向搜索

也就是从PDF文件的位置代码跳转到LaTex源代码的位置。

先配置一下

1. 打开SumatraPDF点击`设置->选项`搜索命令行下面的框里输入

2. 在`设置反向搜索命令行`下面的框里输入

   `"D:/Software/Sublime Text 3/sublime_text.exe"  "%f":"%l"`

   注意！！！

   sublime text的路径要用双引号括住，里面要用斜杠`/`分开，用反斜杠`\`就不行。都是踩的坑呀。
   
   > 2020.08.11更新
   >
   > 我发现有的时候安装，并没有`设置反向搜索命令行`这个选项，可能是版本问题？
   >
   > 解决方案：点击`设置->高级选项`，打开一个txt文本文件，在最后面加上一句
   >
   > ```shell
   > InverseSearchCmdLine = "D:/Software/Sublime Text 3/sublime_text.exe"  "%f":"%l"
   > ```
   >
   > 就好啦，效果跟上面是等价的。

然后。在PDF文件里`双击`某个位置就好啦

### 四、使用技巧

### 1. 分多个tex文件编写

一般一个书的每一章都单独写一个tex文件，然后再主文件`\include{}`即可。但是编译的时候，在每个子tex文件按编译的话。texstudio会自动找到主文件，可是sublime不会自动找，它只编译当前的子tex文件，正向搜索也会出问题。我也困惑了好久，睡一觉就想到了解决方法。

只需要在子tex文件上最开始加上一行即可。也就是

```latex
%!TEX root = ../book.tex
```

指定一下主tex文件是谁就好啦。

> 注： ..的意思是上一层文件夹，所以../book.tex就是上一层文件夹的book.tex文件。根据自己的文件组织方式自行修改即可。

### 2. 添加拼写检查

点击`Preferences -> Settings`打开用户设置

加上一行

```shell
"spell_check": true,
```

如果想改字体什么的加上

```shell
"caret_style": "phase",
"font_face": "YaHei Consolas Hybrid",
"font_size": 12,
"highlight_line": true,
"highlight_modified_tabs": true,
```

