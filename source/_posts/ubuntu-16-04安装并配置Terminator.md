---
title: ubuntu 16.04安装并配置terminator
date: 2021-10-10 16:40:32
tags:
categories: 其他
---
#  概述
最近在使用ros时经常要同时打开多个终端，在各个终端间来回切换很不方便，而ubuntu自带的终端不能在一个终端中切分多个窗口，因此改用Terminator，本文档主要记录ubuntu16.04下安装并配置terminator的过程。
<!--more-->

# 安装

  使用apt包管理工具轻松安装
  ```
  sudo apt-get install terminator
  ```
  此时**CTRL+ALT+T**默认打开的终端已经是terminator了。

# 美化

  如下，默认的terminator外观实在是丑的让人不忍直视
  <div align="center">
  <img src="/all_images/ubuntu1604安装并配置terminator/terminator默认配置.png">
  </div>

  于是上网找了一个外观不错的配置模板，修改或者创建.config/terminator/config文件，添加如下配置

  ```
  [global_config]
    title_font = Ubuntu Mono 11[keybindings]
  [keybindings]
  [layouts]
    [[default]]
      [[[child1]]]
        parent = window0
        type = Terminal
      [[[window0]]]
        parent = ""
        type = Window
  [plugins]
  [profiles]
    [[default]]
      background_color = "#002b36"
      background_darkness = 0.91
      background_image = None
      background_type = transparent
      font = Monospace 12
      foreground_color = "#e0f0f1"
      show_titlebar = False
      use_system_font = False
  ```
  这些设置也可以在：窗口右键->首选项->profiles中手动调整，此时terminator的外观如下：

  <div align="center">
  <img src="/all_images/ubuntu1604安装并配置terminator/配置1.png">
  </div>

  此时，命令提示符颜色并不突出，通过修改PS1值将terminator中的命令提示符颜色设置为与系统自带的终端一样的颜色：

  ```
  gedit ~/.bashrc #打开设置文件
  ```
  找到.bashrc中PS1赋值的位置，注释掉其中的if, else分支，并修改PS1的值：
  ```
  PS1='${debian_chroot:+($debian_chroot)}\[\033[01;32m\]\u@\h\[\033[00m\]:\[\033[01;34m\]\w\[\033[00m\]\$ '
  ```

  <div align="center">
  <img src="/all_images/ubuntu1604安装并配置terminator/命令提示符.png">
  </div>

  保存，在终端输入如下命令后便可永久更改
  ```
  source  ~/.bashrc
  ```

  最终的terminator的外观如下
  <div align="center">
  <img src="/all_images/ubuntu1604安装并配置terminator/最终.png">
  </div>

# 添加右键打开方式
  安装完成后，**Ctrl+Alt+T** 命令就会默认打开terminator，但是在文件浏览器中，默认鼠标右键打开终端还是打开的系统的terminal而非terminator，以下方法使用**nautilus-actions**这个软件实现在不删除系统的终端的情况下添加terminator的启动方式

## 安装
```
sudo apt-get install nautilus-actions
```
## 配置
终端打开nautilus-actions软件：
```
nautilus-actions-config-tool
```
选择新建动作，

对动作选项卡设置如下项：
<div align="center">
<img src="/all_images/ubuntu1604安装并配置terminator/添加右键打开方式1.png">
</div>

对其中的命令选项卡设置如下项：
```
/usr/bin/terminator
--working-directory=%d/%b
```

<div align="center">
<img src="/all_images/ubuntu1604安装并配置terminator/添加右键打开方式2.png">
</div>

点击编辑栏的首选项，去掉Create a root ‘Nautilus-Actions’ menu：

<div align="center">
<img src="/all_images/ubuntu1604安装并配置terminator/添加右键打开方式3.png">
</div>

最后进行配置保存。全部配置完成后，重启电脑，或者在终端运行：
```
nautilus -q
```

此时打开任何一个文件，鼠标右键，就会有一个Open in terminator选项，点击该选项，就会直接打开terminator了

# 常用快捷键

```
Ctrl+Shift+T                    新建一个terminator窗口
Ctrl+Shift+E                    垂直分割窗口
Ctrl+Shift+O                    水平分割窗口
Ctrl+Shift+W                    关闭当前的terminator窗口
Ctrl+Shift+Q                    退出当前窗口，当前窗口的所有终端都将被关闭
Ctrl+Shift+N or Ctrl+Tab        移动到下一个终端
Ctrl+Shift+P or Ctrl+Shift+Tab  移动到之前的一个终端

Alt+Up                          移动到上面的终端
Alt+Down                        移动到下面的终端
Alt+Left                        移动到左面的终端
Alt+Right                       移动到右面的终端

F11                             全屏开关
```

# 参考

* [Ubuntu（16.04）中安装terminator以及美化](https://www.jianshu.com/p/5d1999d05d36)

* [ubuntu下Terminator终端的使用及配置](https://blog.csdn.net/Tansir94/article/details/81410450)

* [Ubuntu16.04或Ubuntu18.04设置右键打开terminator而非系统terminal](https://blog.csdn.net/zhanghm1995/article/details/89419109)

* [Terminator快捷键汇总](https://blog.csdn.net/laviolette/article/details/52703605)
