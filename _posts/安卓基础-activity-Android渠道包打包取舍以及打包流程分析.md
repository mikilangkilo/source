---
title: 安卓基础-Android渠道包打包取舍以及打包流程分析
date: 2018-10-20 16:20:28
tag : "activity"
category : "安卓基础"
description : "本文主要是对Android事件分发机制的学习"
---

# 前述

刚开始接手好搭的时候，用的是美团的多渠道打包方案，但是当时前人写的代码有些问题。

当时的流程大致是：打包一个包，然后解压，修改META-INFO,之后重新压缩，过程中会生成一个额外的包含渠道名的文件。

这样的好处是解压之后重新压缩，并不需要重新编译，也不需要重新签名。

但是当时在apk启动的时候使用的读取渠道包的方式也是和官网提供的一样，没有变化。是启动时解压apk，然后读取相对应的文件。

# 过去方式的缺点

由于需要启动的时候解压apk，整个过程耗时大约200ms，而且是每次都会出现的。加上前人对于性能的优化也不是很在心，当时刚接手的时候启动速度十分慢（大约1-2秒的卡顿）。

后来不得已，出于性能的考虑，对这个动了刀子。

# 变化

由于刚开始的渠道包并不算太多，因此第一个考虑就是使用gradle的变体来进行更改。

变体是官方提供的，使用变体的话，平时打包也可以直接选定打什么包。对于项目来讲，省了不少时间。

# 变体带来的缺点

后期由于项目规模大了起来，增加了5个渠道，全渠道打包时间一下子长了很多，由刚开始的40分钟，猛增到一小时20多分钟。同时市场那边的意思是后期还会继续扩充渠道。

因此现在必须要更改一下这个打包方式。

# 想法1-raw

第一种想法是脱离美团打包，通过自己设置资源文件夹下面的raw目录，来进行打包，raw文件在Android里面是不会编译成二进制文件的，会原样保留到apk里面。在app启动的时候直接读raw文件。
但是这种想法刚说出来就被进奎否了，当时的问题出在raw文件需要进行重新编译，否则不会出现在R.java中，而如果使用apktool重新编译，效率基本上和变体是55开。

# 想法2-asset

第二种想法是第一种想法的补充，既然raw文件需要通过编译来生成r，那我们找一个不需要编译的，想法很简单，打包之后直接塞一个文件到asset里面，这个文件就是配置文件，每次打包之后，解压，然后复制相对应渠道的配置文件到asset里面，这样就可以避免需要回编译的目的了。但是这个方法需要进行重新签名。

# 想法3-美团方案优化

在目前不需要配置额外的配置目录的情况下，想法2其实是有些牛刀小用了。事实上我们完全可以在美团的打包方案上面进行优化。美团提供的打包方案，当时细细想想，完全没有必要每次启动apk的时候都去读取。仅仅需要第一次打开app的时候，读取并写入sp中，之后每次读取sp，如果没有在去重新读取即可。这样可以省了好多事情。

# 总结

鉴于目前项目的规格以及需求，仅仅在需要扩充渠道的时候，仅仅需要重启美团的优化方案，同时优化每次编译的选项即可。但是在后续需要用到不同配置文件的时候，就需要使用想法2的方法，美团的方案不够支持那么详细。

同时鉴于此，也算研究了一下app打包流程，顺便总结一下。

# 打包流程

![打包流程](/images/Android/Androidpackageimage.png)

从官方图可以很明显的看出流程变化。

1. 资源文件是通过aapt来进行打包的，aapt全称是Android Asset Packaging Tool，可以看出来，aapt打包生成了2个东西，一个是R,一个是compiled resources，像想法一之中的raw就是生成在r里面，compiled resources 是asset文件夹下面的东西，这些东西不会参与打包，而是直接被压缩进apk.

关于asset和raw我之前一直认为raw是原封不动，不会编译成二进制，今天才知道两个相同点都是原封不动不会被编译成二进制。差别就是读写方式不同。因此想法1本来就是错误的。

2. java编译器同时编译三者，一是代码，二是r文件，三是aidl编译下来的java接口。编译成class之后，然后和第三方class一起并到dex中（dex是好几个class合并起来的）。
看到这边有个疑问，为什么是class？我们所知道的第三方包，有时候是使用aar的形式来进行依赖的，aar包含了资源文件，但是我查了一下，如果是aar的话。将会被打包到class.jar中，应该就是先打包到jar中，然后又会被放到dex中

3. asset， resource， dex 三者就可以打包成一个apk了，此时使用apkbuilder即可，打包成一个未签名的apk。

4. apk好了之后使用jarsigner来进行签名

5. 按理说签名完之后就应该结束了，为什么还会有个zipaligh呢？

zipalign的效果：zipalign是一个对齐工具，Android基于linux，因此资源也是仿照linux来的，在多进程需要寻找资源的时候，最好的方式是按照linux架构来设计数据摆放位置，因此最好的是放在4字节层，这样子的话系统就不需要读取所有的文件，而直接可以类似链表的形式知道什么资源在什么地方。这样节省了大量的内存。因此这个步骤我们称之为对齐

