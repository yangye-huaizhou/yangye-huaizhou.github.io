---
layout: post
title: 如何把ubuntu玩成windows
date:   2017-05-01 13:50:39
categories: just-for-fun
---

Linux系统作为一款优秀且开源的现代操作系统，被科研教育界广泛应用。其中使用最多且最适合新手的发行版就是ubuntu。在ubuntu操作系统中，最困扰的一点就是，无法像windows系统一样有QQ、微信以及音乐播放器等经常使用的工具。那么这篇文章就是告诉你怎样安装这些常用软件，解决一些基本的问题，更好地玩转ubuntu系统~

一般来说，在ubuntu系统中，选择带有LTS（long term support）版，从而获得三年以上的更新支持，比方说ubuntu 14.04LTS、ubuntu16.04LTS。我们这里就以ubuntu16.04为例。


![图片.png](http://upload-images.jianshu.io/upload_images/5971286-77d1d83033e6279f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


在安装完系统以后，需要手动安装的软件如下，可以根据自己需要有选择性地安装：

## 1.安装Vim
```sudo apt-get install vim```

## 2.安装谷歌浏览器
谷歌浏览器的下载地址：<https://dl.google.com/linux/direct/google-chrome-stable_current_amd64.deb>

命令行安装：
```
sudo apt-get install libappindicator1 libindicator7  
sudo dpkg -i google-chrome-stable_current_amd64.deb   
sudo apt-get -f install
```

## 3.安装搜狗输入法和WPS
直接去官网<http://pinyin.sogou.com/linux/> 下载对于系统（32位/64位）的安装包deb文件

命令行安装：
```
sudo dpkg -i sogoupinyin_2.1.0.0086_amd64.deb   
sudo apt-get -f install  
```

系统自带的libreoffice虽然是开源的，但是Java写出来的office又丑又慢，果断删除，换用WPS。
由于上一步安装搜狗输入法的时候，解包时已经将加进了ubuntu kylin的源，所以直接apt-get就可以安装WPS。

先删除libreoffice：
```
sudo apt-get remove libreoffice-common
```
然后安装WPS:
```
sudo apt-get install wps-office
```

**安装完WPS以后，第一次打开，会提示字体缺失：
下载字体包并加入系统字体文件夹：**
链接: <https://pan.baidu.com/s/1dELTDBj> 密码: fn98

将整个wps_symbol_fonts目录拷贝到 /usr/share/fonts/目录下
注意，wps_symbol_fonts目录要有可读可执行权限
权限设置,执行命令如下
```
cd /usr/share/fonts/
chmod 755 wps_symbol_fonts
cd /usr/share/fonts/wps_symbol_fonts 
chmod 644 *
```
生成缓存配置信息
```
cd /usr/share/fonts/wps_symbol_fonts
mkfontdir
mkfontscale
fc-cache
```

另外，此过程还有可能有报一个错，**ttf-mscorefonts-installer安装失败**，这是微软的字体，据说年代久远了，所以下载的时候，经常下不下来，所以会安装失败。这里给了传送门：
链接: <https://pan.baidu.com/s/1miLVEbA> 密码: xaen

在终端输入：
```
sudo dpkg-reconfigure ttf-mscorefonts-installer
```
会弹出界面，输入fonts文件夹的路径，自动安装完成。


此时的WPS中中文字体很少，对应的官方也给出了中文字体的安装包，华文XXX等字体都可以用了。当然，我们也可以把windows下好看的字体直接复制到ubuntu系统的字体文件夹，修改权限后就可以使用。

官方中文字体包：
链接: <https://pan.baidu.com/s/1pL8suaJ> 密码: y8as


## 4.安装git、cmake、openssh-server
命令行输入：
```
sudo apt-get install git cmake openssh-server
```

## 5.安装Sublime Text 3编辑器
Sublime text3编辑器的字体非常好看，而且还有很多强大的插件。安装如下:
```
sudo add-apt-repository ppa:webupd8team/sublime-text-3    
sudo apt-get update    
sudo apt-get install sublime-text
```

## 6.安装oracle java
```
sudo add-apt-repository ppa:webupd8team/java    
sudo apt-get update    
sudo apt-get install oracle-java8-installer
```
安装时间取决于下载速度

## 7.安装eclipse
直接去官网下载免安装的压缩包，或者直接走这里的传送门：
链接: <https://pan.baidu.com/s/1qYbgo5e> 密码: dwf8
下载解压后，由于是免安装，直接运行文件夹下的eclipse即可，同时我们也可以建立链接，从而在命令行输入eclipse就直接打开软件：
```
cd /usr/local/bin
sudo ln -s /home/torronto/eclipse/eclipse  (后面路径是你eclipse可执行文件的位置)
```

## 8.安装cuda
64位传送门：
链接: <https://pan.baidu.com/s/1slPjpG5> 密码: k9v6
使用命令，将deb文件解包：
```
sudo dpkg -i cuda-repo-ubuntu1504_7.5-18_amd64.deb
```
安装，需要下载1GB多文件，要个半天时间
```
sudo apt-get update
sudo apt-get install cuda
```
 
安装好了后，还需要配置环境变量，才可以使用：
在conf里面加入库文件，新建一个cuda.conf
```
sudo gedit /etc/ld.so.conf.d/cuda.conf
```
输入一行:```/usr/local/cuda/lib64```,保存退出
使用命令```sudo ldconfig```重新加载一次配置文件

将bin和path加进profile文件
```
sudo gedit /etc/profile
```
在最后两行加上：
```
PATH=$PATH:/usr/local/cuda/bin
LD_LIBRARY_PATH=$LA_LIBRARY_PATH:/usr/local/cuda/lib64
```
保存退出，source一下：
```
sudo source /etc/profile
```
 
查看命令是否可用
```nvcc --version```出现版本号，即可用了


## 9.安装QQ：
安装包传送门：
链接: <https://pan.baidu.com/s/1i4HHOzV> 密码: 18rj

如果是64位系统，先添加对32位库的支持：
```
sudo dpkg --add-architecture i386
sudo apt-get update
```
可能需要添加下列32位库
```
sudo apt-get install lib32z1 lib32ncurses5
```
安装crossover
如果要安装14版本，从上面的分享地址里下载crossover_14.1.11-1_all.deb、crossover_14.1.11-1_all-free.deb、deepin-crossover_0.5.14_all.deb三个文件。依次安装
如果安装15，crossover-15_15.0.3-1_all.deb crossover-15_15.0.3-1_all-free.deb、deepin-crossover-helper_1.0deepin0_all.deb 并依次安装。
可能出现的依赖问题，如果crossover不能使用(不能创建容器)，安装libp11-kit-gnome-keyring_3.18.3-0ubuntu2_i386.deb，还是不能的话的试试64位版的
注意：
QQ 8.x，需要Crossover 15
QQ 7.x 支持Crossover 14

都是deb包，可以直接下载并用```dpkg -i```安装。

通常叉掉qq后，可能再次打开会无法登录，提示已经登录了qq，在debian8 gnome3下，qq无法最小化到托盘。此时必须kill掉qq相关的进程。
存在问题，可以语音，但是视频存在bug。
注意问题：
偶尔出现卡顿或者无法使用QQ的远程协助
需要安装以下32位库：
```
sudo apt-get install  libgl1-mesa-glx:i386 libssl1.0.0:i386 libgphoto2-6:i386 
```
无法输入中文：
因为QQ以及WPS等软件默认是采用的ibus输入系统，而我们的系统输入是fcitx，所以修改这些应用使用fcitx：
```
sudo vim ~.profile
```
在最后加上两行：
```
export XMODIFIERS="@im=fcitx"
export QT_IM_MODULE="fcitx"
```
保存重启。
以后每次重启若出现没有输入法，只需在命令行输入 fcitx启动输入法即可。


## 10.安装微信：
此版本是基于网页版的微信，免安装，解压即用，传送门：
链接: <https://pan.baidu.com/s/1pKVcOiB> 密码: k7cf


## 11.安装网易云音乐：
网易云音乐现在推出了linux版，真是一个绝好的消息，这比以前在ubuntu下勉强安装深度音乐要好多了。官网：<http://music.163.com/#/download>选择linux版，下载然后dpkg -i安装。

## 12.安装视频播放器：
Ubuntu下视频播放器首推SMplayer，安装：
```
sudo apt-get install smplayer
```

至此，恭喜你已经把ubuntu玩成windows了，enjoy it！