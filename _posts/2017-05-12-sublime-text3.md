---
layout: post
title: sublime text3安装c语言环境插件
date:   2017-05-12 23:30:39
categories: just-for-fun
---
以前在linux下看源码一直习惯用source insight，然而最近发现sublime text的插件好多好强大，完全替代了source insight，转战sublime还有个重要原因是SI的字体太丑了。今天就来说说怎么在sublime下搭建c语言的环境吧，主要是自动补全还有跳转定义等功能。

刚安装好的sublime是没有装package control的，需要先在tools的下拉菜单栏里安装：
![安装package control.png](picture/sublime1.png)

安装完成后，在去preference的下来菜单里找到package control，选择package control：Install package
![使用package control安装插件.png](picture/sublime2.png)

在输入栏里写上c，会自动补全cmake这样的插件，这里我们依次安装c improved，c+11，cMake，cMakeBuilder，cMakeEditor
![安装插件.png](picture/sublime3.png)

完成后，我们从File→Open Folder打开一个工程看看效果，如下图：
![跳转定义.png](picture/sublime4.png)

![自动补全.png](picture/sublime5.png)

已经有了自动补全和跳转定义的效果了，大功告成。另外注意到，右下角有选择语言，点开发现sublime真强大，支持的语言连一些很少用的脚本语言都有。
![支持的语言.png](picture/sublime6.png)

这次换回了新笔记本，上面的双系统给装的deepin，一个国产的操作系统，就是在debian版本上加了自己的UI以及一些软件支持，UI像极mac，现在更新的版本，毛玻璃效果，还有8.9的QQ，真心不错，放张图～

![deepin系统.png](picture/sublime7.png)