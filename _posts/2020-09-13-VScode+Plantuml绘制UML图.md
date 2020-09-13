---
ruxilayout: post
title: "VScode+PlantUML画UML图"
description: ""
categories: []
tags: [Essential Skill]
redirect_from:
  - /2020/09/13/
typora-root-url: ..
---

前两天发现了UML图对看源代码理解源代码还是非常有用处的，所以就花时间折腾了一下。希望成为一个神器


* Kramdown table of contents
{:toc .toc}
因为踩到了坑，所以记录下来

### 1. 安装java环境

官网[https://www.oracle.com/java/technologies/javase-jre8-downloads.html](https://www.oracle.com/java/technologies/javase-jre8-downloads.html)

注：主要注册一下再下载，免费的。不想注册的话自行在网上找安装包

> 然后把java的bin文件夹添加到环境变量即可
>
> 后面证明这一步并没用，但添加进去总没坏处

```
C:\Program Files\Java\jre1.8.0_261\bin
```

### 2.下载Graphviz

官网：[https://graphviz.gitlab.io/](https://graphviz.gitlab.io/)

> 同样把bin文件夹添加到环境变量

### 3. 下载plantuml

官网：[https://plantuml.com/zh/starting](https://plantuml.com/zh/starting)

下载得到一个.jar包，放在任意文件夹即可

### 4. VScode 上安装插件

- PlantUML
- Graphviz Preview

然后对其进行设置

点击`VScode 右下角小齿轮->设置->用户->扩展->PlantUML`配置

![set](/images/posts/2020-09-13/set.png)

修改选项

- `Jar (自定义 plantuml.jar 的位置。留空则使用内置的版本。)`修改为刚才下载的plantuml.jar文件所在的位置
- `Java(Java 可执行文件位置。)`修改为java可执行文件的位置

再转到`Graphviz Preview`的设置，点击`在setting.json中编辑`， 添加Graphviz的dot可执行文件的位置

```shell
 "graphvizPreview.dotPath": "D:/Programs/Graphviz 2.44.1/bin/dot.exe",
```

如下图所示

![set2](/images/posts/2020-09-13/set2.png)

> 注意：
>
> 必须设置java可执行文件的位置，不设置的话我就报错了。我也添加环境变量了
>
> ```shell
> Error: write EPIPE
> at afterWriteDispatched (internal/stream_base_commons.js:149:25)
> at writeGeneric (internal/stream_base_commons.js:140:3)
> at Socket._writeGeneric (net.js:776:11)
> at Socket._write (net.js:788:8)
> at doWrite (_stream_writable.js:435:12)
> at writeOrBuffer (_stream_writable.js:419:5)
> at Socket.Writable.write (_stream_writable.js:309:11)
> at c:\Users\Albert\.vscode\extensions\jebbs.plantuml-2.13.13\out\src\plantuml\renders\local.js:112:35
> at processTicksAndRejections (internal/process/task_queues.js:94:5)
> ```

### 5. 开始使用

随便新建一个文件，写入代码

> 哈哈，专门挑了一个花里胡哨的代码，其实普通的使用没这么复杂的

```shell
@startuml
skinparam backgroundColor #EEEBDC
skinparam handwritten true

skinparam sequence {
ArrowColor DeepSkyBlue
ActorBorderColor DeepSkyBlue
LifeLineBorderColor blue
LifeLineBackgroundColor #A9DCDF

ParticipantBorderColor DeepSkyBlue
ParticipantBackgroundColor DodgerBlue
ParticipantFontName Impact
ParticipantFontSize 17
ParticipantFontColor #A9DCDF

ActorBackgroundColor aqua
ActorFontColor DeepSkyBlue
ActorFontSize 17
ActorFontName Aapex
}

actor User
participant "First Class" as A
participant "Second Class" as B
participant "Last Class" as C

User -> A: DoWork
activate A

A -> B: Create Request
activate B

B -> C: DoWork
activate C
C --> B: WorkDone
destroy C

B --> A: Request Created
deactivate B

A --> User: Done
deactivate A

@enduml
```

然后按快捷键`Alt+D`

完美得到效果图

![show](/images/posts/2020-09-13/show.png)

注意：

- 修改文件后缀名为.plantuml就能有代码高亮了
- 官网有超级多示例：[https://plantuml.com/zh/](https://plantuml.com/zh/)

