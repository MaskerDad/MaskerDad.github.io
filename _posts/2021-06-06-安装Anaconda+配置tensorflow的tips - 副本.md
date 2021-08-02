---
layout: post
title: "安装Anaconda+配置支持显卡的tensorflow，pytorch的tips"
description: ""
categories: []
tags: [Python]
redirect_from:
  - /2021/06/06/
typora-root-url: ".."
---



* Kramdown table of contents
{:toc .toc}
## 1.Anaconda 安装包下载地址

官网：[Anaconda | Individual Edition](https://www.anaconda.com/products/individual#Downloads)

国内源： [Index of /anaconda/archive/ | 北京外国语大学开源软件镜像站 | BFSU Open Source Mirror](https://mirrors.bfsu.edu.cn/anaconda/archive/)

**Tips：**

- 官网一定是最新版，国内源不一定是最新版，这个北京外国语大学的目前来看一直是最新版
- 清华大学、中科大的源已经失效了，都是给了个链接给到北京外国语大学的源了。这个北外的源好像也是清华源的团队做的

## 2.Anaconda 安装

windows上双击，Linux上`bash Anaconda***.sh`即可

**Tips:**

- 安装在哪不重要，安装的时候注意看提醒添加到PATH环境变量，如果没有自动添加，则按照下面步骤需要手动添加
  - Linux上
    - 执行命令` vim ~/.bashrc`（vim是命令行的文本编辑器，如果有图形化的界面的话用其他也行比如`gedit ~/.bashrc`）
    - 添加`export PATH=/home/(username)/anaconda3/bin:$PATH`到文件最后。重点是这个安装路径`/home/(username)/anaconda3/bin`，因人而异
    - 保存后执行命令` source ~/.bashrc`应用刚才修改的环境变量
  - windows上，参考[Windows添加环境变量_xue_nuo的博客-CSDN博客](https://blog.csdn.net/xue_nuo/article/details/114793534) 。把类似于`/home/(username)/anaconda3/bin`的安装文件夹下的bin文件夹添加进去进行。

## 3.Conda切换国内源

conda 是Anaconda的包管理器，一般默认的源是国外的，很慢。

参考[anaconda | 镜像站使用帮助 | 北京外国语大学开源软件镜像站 | BFSU Open Source Mirror](https://mirrors.bfsu.edu.cn/help/anaconda/)这个链接设置为国内北京外国语大学的源，我也试了阿里源，中科大源，豆瓣源，感觉都停止服务不更新了，目前2021年6月，这个北外的最好用。

- Linux上

  - 执行命令`conda config --set show_channel_urls yes`生成`.condarc`配置文件

  - 执行命令` vim ~/.condarc`

  - 把里面的内容替换为

    ```sh
    channels:
      - defaults
    show_channel_urls: true
    default_channels:
      - https://mirrors.bfsu.edu.cn/anaconda/pkgs/main
      - https://mirrors.bfsu.edu.cn/anaconda/pkgs/r
      - https://mirrors.bfsu.edu.cn/anaconda/pkgs/msys2
    custom_channels:
      conda-forge: https://mirrors.bfsu.edu.cn/anaconda/cloud
      msys2: https://mirrors.bfsu.edu.cn/anaconda/cloud
      bioconda: https://mirrors.bfsu.edu.cn/anaconda/cloud
      menpo: https://mirrors.bfsu.edu.cn/anaconda/cloud
      pytorch: https://mirrors.bfsu.edu.cn/anaconda/cloud
      simpleitk: https://mirrors.bfsu.edu.cn/anaconda/cloud
    ```

  - 执行命令`conda clean -i`清除索引缓存（刚安装的anaconda就没必要执行这个命令了）

- Windows 也是修改用户目录下的`.condarc`文件， 一般在`C:\Users\username`文件夹下。其他与linux相同

## 4.pip切换国内源

pip是python 的包管理器，主要功能与conda相同，但是我不喜欢用conda安装包，conda我基本只用来创建虚拟环境。因为之前我用conda安装的包的时候，因为版本依赖，conda会强行改变依赖的包的版本号。本来python里面的版本问题就非常复杂，他改的话很容易另外一个包就不能用了。然而pip不会。pip只会提示依赖包的版本不对，安装失败，而不是去强行改他的版本。这样我可以根据依赖自行选择安装合适的版本。

pip的源也是参考[pypi | 镜像站使用帮助 | 北京外国语大学开源软件镜像站 | BFSU Open Source Mirror](https://mirrors.bfsu.edu.cn/help/pypi/)

- 临时使用

  ```shell
  pip install -i https://mirrors.bfsu.edu.cn/pypi/web/simple
  ```

- 永久使用（设置为默认源）

  ```
  pip config set global.index-url https://mirrors.bfsu.edu.cn/pypi/web/simple
  ```



其他的国内源可以留作备用，自行根据情况选择

中国科学技术大学 ：http://pypi.mirrors.ustc.edu.cn/simple/

阿里云：http://mirrors.aliyun.com/pypi/simple/

豆瓣(douban)：http://pypi.douban.com/simple/

清华大学： https://pypi.tuna.tsinghua.edu.cn/simple/

华中科技大学： https://pypi.hustunique.com/



## 5.安装显卡驱动、cuda、cuDNN, tensorflow, pytorch

注意！！！一定先看好版本再安装！版本错的话tensorflow是不能正常调用的

- 最新的tensorflow版本适配看官网 ： [从源代码构建  | TensorFlow](https://www.tensorflow.org/install/source)

- Pytorch提供了不同cuda版本的安装命令 ：[Previous PyTorch Versions | PyTorch](https://pytorch.org/get-started/previous-versions/)

- cuda（cudaToolkit）与NVIDIA显卡驱动的版本关系：[Release Notes :: CUDA Toolkit Documentation (nvidia.com)](https://docs.nvidia.com/cuda/cuda-toolkit-release-notes/index.html)

- cudnn与cuda的版本关系： [cuDNN Archive | NVIDIA Developer](https://developer.nvidia.com/rdp/cudnn-archive)

> 注意：conda提供了 `conda install cudnn=7.6.5 cudatoolkit=10.1` 按道理可以一键安装呀，可是我试了一下，丝毫没搞懂他安装到哪了，之后有时间可以再研究一下。

所以，我们需要做的是

1. 根据自己借鉴的代码或者自己喜欢用的版本选择tensorflow的版本或pytorch版本（或者直接选最新版就行）

2. 根据tensorflow版本在 [从源代码构建  | TensorFlow](https://www.tensorflow.org/install/source)找对应的cuda和cuDNN 版本（pytorch在[Previous PyTorch Versions | PyTorch](https://pytorch.org/get-started/previous-versions/)找）

3. 根据cuda版本看显卡驱动需求[Release Notes :: CUDA Toolkit Documentation (nvidia.com)](https://docs.nvidia.com/cuda/cuda-toolkit-release-notes/index.html)

4. 下载显卡驱动并安装：[官方 GeForce 驱动程序 | NVIDIA](https://www.nvidia.cn/geforce/drivers/)

5. 下载cuda并安装：[CUDA Toolkit Archive | NVIDIA Developer](https://developer.nvidia.com/cuda-toolkit-archive) （注：安装的时候记得取消安装驱动，因为上面已经安装过了，为什么不直接cuda和驱动一起安装呢？好像容易出现报错，我就报错了）

6. 下载cudnn并安装[https://developer.nvidia.com/rdp/cudnn-download](https://developer.nvidia.com/rdp/cudnn-download)（注：需要登录，下载后把文件复制到对应文件夹即可，可参考[win10安装CUDA和cuDNN的正确姿势 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/94220564)，Linux类似）

7. 查看cuda和cudnn是否安装成功

   ```shell
   nvcc --version
   cat /usr/local/cuda/version.txt
   cat /usr/local/cuda/include/cudnn.h | grep CUDNN_MAJOR -A 2
   ```

8. 用pip安装对应版本的tensorflow或pytorch

9. 检查GPU是否可用

   ```
   import tensorflow as tf
   tf.test.is_gpu_available()
   
   或者
   import torch
   torch.cuda.is_available()
   ```

**Tips:**  

​		我遇到的一个情况，本机已经安装好的的cuda是10.2, 我想安装tensorflow, 但是没有tensorflow版本支持cuda10.2 。重新安装cuda10.1又太麻烦了。我找到了一个方法，强行改版本号。参考[Ubuntu安装cuda10.2来用tensorflow2.x和pytorch_wm9028的专栏-CSDN博客](https://blog.csdn.net/wm9028/article/details/110380095)，亲测可用

```shell
#1.前提是已经正确安装cuda10.2
nvidia-smi
nvcc -V

# 2.干正事（重点就这二步）：
cd /usr/local/cuda-10.2/targets/x86_64-linux/lib/
sudo ln -s libcudart.so.10.2.89 libcudart.so.10.1
sudo ln -s libcupti.so.10.2.75 libcupti.so.10.1

# 如果使用的是cudnn8.X 如果是cudnn7.x 请略过
cd /usr/local/cuda-10.2/lib64
sudo ln -s libcudnn.so.8 libcudnn.so.7


# 检查添加路径：
sudo vi ~/.bashrc

export CUDA_HOME=/usr/local/cuda-10.2
export LD_LIBRARY_PATH=${CUDA_HOME}/lib64:${CUDA_HOME}/extras/CUPTI/lib64
export PATH=${CUDA_HOME}/bin:${PATH}

source ~/.bashrc

# 3. 检查GPU
>python
>>>import tensorflow as tf
>>>tf.__version__
2.3.1
>>>tf.test.is_gpu_available()
True
>>>tf.config.list_physical_devices('GPU')
[PhysicalDevice(name='/physical_device:GPU:0', device_type='GPU'),
 PhysicalDevice(name='/physical_device:GPU:1', device_type='GPU'),
 PhysicalDevice(name='/physical_device:GPU:2', device_type='GPU'),
 PhysicalDevice(name='/physical_device:GPU:3', device_type='GPU')]

```



