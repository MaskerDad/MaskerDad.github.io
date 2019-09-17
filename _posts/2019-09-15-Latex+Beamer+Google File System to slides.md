---
layout: post
title: "Latex+Beamer+Google File System Model to make slides"
description: ""
categories: [LaTexManuals]
tags: [LaTex]
redirect_from:
  - /2019/09/15/
typora-root-url: ..
---

这是我第一次用LaTex+Beamer做slides

## 一、挑选使用的Beamer主题模板

这里推荐[LaTex开源小屋提供的模板](https://www.latexstudio.net/archives/category/tex-slides/beamer-theme-template)，还是有一部分感觉很不错的。

我挑选了[The Google File System的beamer主题](https://www.latexstudio.net/archives/12234.html)，比较简洁，还是挺适合做学术类的Presentation的。

![](/images/posts/2019-09-15/GFS.png)

## 二、直接拿着模板就是写

> 前提是安装好TexLive(各种LaTex编译器的集合版本)+TeXstudio(非常好用的LaTex IDE)。网上大把安装教程，并且安装TexLive时提供附带安装TeXstudio，注意勾选
>
> 之前我安装好了，这里我就不重新安装去截图了。自行Google安装

普通的用法模板里都有，不再赘述，这里只记录我不会的内容

### 1.在Slides中插入视频（MP4/H.264或FLV格式的视频+网络视频+Mp3）

```latex
\usepackage{media9}

%01、插入一个本地视频
\includemedia[
  width=1.0\linewidth,height=0.675\linewidth, % #视频的长和宽16:9
  addresource=filename.mp4, %视频路径
  transparent,%Indicates whether underlying page content is visible through transparent areas of theembedded media.
  activate=pageopen,%怎么样激活这个Media
  passcontext,%If set, user right-clicks are passed through to the context menu of theembedded Flash application, replacing the default Adobe Reader context menu
  flashvars={
    source=filename.mp4%视频路径
   &autoPlay=true % start playing on activation 自动播放吗？
   &loop=true%循环播放吗？
  }
]{}{VPlayer.swf}%视频播放插件
% 下面是两个按钮
%开始/暂停
\mediabutton[
  mediacommand=some_dice:playPause,
  overface=\color{blue}{\fbox{\strut Play/Pause}},
  downface=\color{red}{\fbox{\strut Play/Pause}}
]{\fbox{\strut Play/Pause}}
%Media标题，如果点击的话点开什么文件
\mediabutton[
  mediacommand=some_dice:setSource [(filename.mp4)]%这里再设置一个文件路径
]{\fbox{\strut Media Caption}}


%02、播放远程视频
\includemedia[
  width=0.6\linewidth,height=0.3375\linewidth,
  activate=pageopen,
  flashvars={
  modestbranding=1 % no YT logo in control bar
  &autohide=1 % controlbar autohide
  &showinfo=0 % no title and other info before start
  &rel=0 % no related videos after end
}
]{\includegraphics[width=0.6\linewidth]{048}}{https://www.youtube.com/v/g8Ejj0T0yG4?rel=0}

%03、播放MP3
\includemedia[
  transparent,
  passcontext,
  addresource=SampleAudio.mp3,
  flashvars={source=SampleAudio.mp3},
]{\color{blue}\framebox[0.4\linewidth][c]{Applause}}{APlayer.swf}

```

> 注意！！！
>
> 1、媒体文件的播放使用Adobe Reader的内置Flash Player，所以必须用Adobe Reader才能正常打开PDF中的视频。这里提供一个下载链接[Adobe Reader免安装版](http://www.downxia.com/downinfo/3790.html)，免安装版可以放在U盘里，便于在别的电脑上用Slides做Presentation
>
> 2、Media9必须用pdflatex编译，一般TexStdio默认编译器是xelatex，编译出来的pdf中的视频不能正常播放，所以要加上
>
> ```latex
> % !TeX TXS-program:compile = txs:///pdflatex/
> % !BIB program = biber
> ```
>
> 然后允许本文档使用pdflatex构建即可。(这两行代码可以放在文件最前面，也可以放在其他任何位置，TexStudio会自动识别)

### 2、左右排版文字和图片

之前只用`figure`和`minifigure`排版过多个图片，这次做Slides突然需要左边显示文字，右边显示图片的功能。网上没有找到现成的用法，本来尝试用`table`的，类似于将屏幕分成$1 \times 2$的表格，左边放文字，右边放图片。但是没成功，总是出现错误，我觉得应该是可以的，只是我的姿势错了。

然后我无意间看到了用`figure`和`minifigure`也可以排版文字。

所以

```latex
\begin{frame}
\frametitle{北斗卫星天线}
\framesubtitle{手持机天线——微带天线}
\begin{figure}[] 
	\begin{minipage}{0.45\linewidth}
		\begin{itemize}
			\item 频率范围：Tx:L Rx:S
			\item 极化方式：Tx:LHCP Rx:RHCP
			\item 天线增益：Tx:4 dBi Rx:3.5 dBi
			\item 配合手持设备使用的天线，该系列天线能满足手持设备高精度的要求，亦可配合各种OEM板卡使用。
		\end{itemize}
	\end{minipage}
	\begin{minipage}{0.5\linewidth}
		\centering
		\includegraphics[scale=0.6]{figures/bdsa6.jpg}
	\end{minipage}
\end{figure}
\end{frame}
```

大功告成

![](/images/posts/2019-09-15/minifigure.png)

我也试了左边放表格，也是可以的。所以`minifigure`不一定只能放图片哦！

### 3、插入gif动图

**情况1**、需要插入的gif是类似于视频的，只需要将他播放一遍就好。这样的话可以直接将gif转成mp4，然后用上面的插入mp4的方法插入即可。

**情况2**、需要控制播放速度，和逐帧播放的情况。请使用下面的方法。

#### 第一步，将gif文件分解为一帧一帧的多个图片。

方法1：使用在线分解，百度搜索`gif分解即可`。这里放两个链接备用 [动态图分解](http://tool.chuanxincao.net/fenjie/)和[动态图分解](https://tu.sioe.cn/gj/fenjie/)。

方法2：下载工具分解

1. 下载[ImageMagick](https://imagemagick.org/script/download.php)
2. 安装好后，进入安装目录，打开cmd窗口，输入 `convert -coalesce 输入的gif文件的路径 输出的png文件的路径`。或者直接将ImageMagick添加到环境变量，在gif文件的路径下打开cmd窗口输入 `convert -coalesce test.gif test.png`
3. 然后会生成数个由test.gif分解来的test-[i].png文件（i为gif帧的个数）

方法3：觉得上面都太麻烦，就用Python自己写个程序

#### 第二步

```latex
\usepackage{graphicx}%打开照片要用这个包
\usepackage{animate}%播放多个gif

\begin{frame}
\frametitle{500米口径球面射电望远镜}
\framesubtitle{以下内容源自500米口径球面射电望远镜工程官网http://www.cas.cn/zt/kjzt/fast}
\centering
\animategraphics[autoplay,loop,controls,%自动播放，循环播放，显示控制按钮
scale=0.35%也可以用  width,height控制大小
]{0.5}{figures/fast}{1}{3}%以每秒0.5张的速度播放fast1到fast3的图片(注意图片名字除了最后面的数字，前面都要一样)
\end{frame}
```

> 注 :用PDFLaTex或XeLaTex编译。要用adobe reader等支持javascript的阅读器才能正常播放（福昕不行）。



以上就是这次做Slides中遇到的问题和解决方法啦。

这次是老师让我们根据一个题目做一个Presentation，我被分到的是卫星通信天线(地面端)。找资料和整理资料其实就加起来花了几个小时，但做Slides用了我两天时间。简直丧心病狂。一方面是我没有全神贯注的做，另一方面就是我第一次用Beamer做Slides，其中遇到了很多问题，每解决一个问题都要排查好久。不过最终每个问题都让我解决了，还不不错的。加深了对Latex的理解。



最后记忆一个很有用的命令，在cmd窗口执行（前提是texlive/bin/win32路径已经被添加到环境变量了）

```shell
texdoc package名
```

就是查看Package的帮助文档，非常有用哦，很多情况下百度都不能详细的帮助你，而帮助文档却是对某个命令最详细的解释。

比如`texdoc media9`去看看添加视频的命令还有什么可以详细设置的地方吧

### 4、表格过大(或过小)的调整方法

这里借鉴了[Latex 表格过大(或过小)的调整方法](https://blog.csdn.net/wbl90/article/details/52597429)，感谢大佬。

#### 01.表格过窄

使用`\setlength{\tabcolsep}{7mm}{XXXX}`调整表格宽度, 效果为”按页面宽度调整表格“

```latex
\setlength{\tabcolsep}{7mm}{
\begin{tabular}{cccccc} \toprule
Models  &  $\hat c$  &  $\hat\alpha$  &  $\hat\beta_0$  &  $\hat\beta_1$  &  $\hat\beta_2$  \\ \hline
model  & 30.6302  & 0.4127  & 9.4257  & - & - \\
model  & 12.4089  & 0.5169  & 18.6986  & -6.6157  & - \\
model  & 14.8586  & 0.4991  & 19.5421  & -7.0717  & 0.2183  \\
model  & 3.06302  & 0.41266  & 0.11725  & - & - \\
model  & 1.24089  & 0.51691  & 0.83605  & -0.66157  & - \\
model  & 1.48586  & 0.49906  & 0.95609  & -0.70717  & 0.02183  \\
\bottomrule
\end{tabular}}
```

#### 02.表格宽过宽

用`\resizebox{\textwidth}{15mm}{XXXX}`调整表格宽度, 效果为”按内容调整表格”.

```latex
\resizebox{\textwidth}{15mm}{
\begin{tabular}{cccccccccccc} \toprule
Models  &  $\hat c$  &  $\hat\alpha$  &  $\hat\beta_0$  &  $\hat\beta_1$  &  $\hat\beta_2$ & Models  &  $\hat c$  &  $\hat\alpha$  &  $\hat\beta_0$  &  $\hat\beta_1$  &  $\hat\beta_2$  \\ \hline
model  & 30.6302  & 0.4127  & 9.4257  & - & -  & model  & 30.6302  & 0.4127  & 9.4257  & - & -\\
model  & 12.4089  & 0.5169  & 18.6986  & -6.6157  & - & model  & 30.6302  & 0.4127  & 9.4257  & - & - \\
model  & 14.8586  & 0.4991  & 19.5421  & -7.0717  & 0.2183 & model  & 30.6302  & 0.4127  & 9.4257  & - & - \\
model  & 3.06302  & 0.41266  & 0.11725  & - & - & model  & 30.6302  & 0.4127  & 9.4257  & - & - \\
model  & 1.24089  & 0.51691  & 0.83605  & -0.66157  & - & model  & 30.6302  & 0.4127  & 9.4257  & - & - \\
model  & 1.48586  & 0.49906  & 0.95609  & -0.70717  & 0.02183  & model  & 30.6302  & 0.4127  & 9.4257  & - & - \\
\bottomrule
\end{tabular}}
```















