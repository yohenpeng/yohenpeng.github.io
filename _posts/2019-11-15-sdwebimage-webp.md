---
layout: post
title: SDWebImage支持加载Webp格式图片
subTitle: SDWebImage支持加载Webp格式的几种办法
description: SDWebImage要支持webp格式，必须依赖libwebp库，但是由于libwebp原有源是在chromium.googlesource.com上，翻墙pod install安装也会失败
catalog: true
tags:
    - iOS
    - 代码集成
---

# YHLibwebp
**前言：** *由于webp格式和png同等清晰度下，数据量更小，项目组决定要支持加载wbep图片，SDWebImage要支持webp格式，必须依赖libwebp库，但是由于libwebp原有源是在chromium.googlesource.com上，翻墙pod install安装也会失败。报错信息如下：*

`Cloning into'/var/folders/9d/jkc05y752h1csv8s91s27pg80000gn/T/d20170503-52118-q8kwcb'...fatal: unable to access'https://chromium.googlesource.com/webm/libwebp/': Failed to connect to chromium.googlesource.com port443: Operation timed out`

*由于项目里面都是使用SDWebImage框架，改用YYImage成本也比较大，时间也不充裕，目前通过查阅网上的办法并测试发现目前就三个办法可行，可以看文章末尾，都不太适合我们工程。比较了几个方法之后，我在想是不是可以把依赖的libwebp库下载下来推送到私有pod仓库，测试用enkins也不用翻墙，毕竟libwebp库的更新频率是比较低的，能保证SDWebImage每次打包都更新就可以，于是就有了这个仓库*



**使用方法**

1、podfile新增：

```objective-c
source https://github.com/yohenpeng/YHSpecs.git
pod 'YHLibwebp'
pod 'SDWebImage'
```

2、将仓库里的 SDImageWebPCoder引入到工程中，并将其加入到解码管理类

```objective-c
SDImageWebPCoder *coder = [SDImageWebPCoder sharedCoder];
[[SDImageCodersManager sharedManager] addCoder:coder];
```



-------

### 现有解决方案

#### A.翻墙 + 替换host文件

[host文件链接](https://raw.githubusercontent.com/racaljk/hosts/master/hosts)

优缺点：替换一次host文件即可，可以使用Cocoapod集成，但是需要翻墙



#### B.无需翻墙 + 修改镜像源

1. podfile增加`SDWebImageWebPCoder` 运行pod install，肯定会失败的，不过可以查看SDWebImage依赖的libwebp版本，比如0.6.1
2. find ~/.cocoapods/repos/master -iname libwebp输出路径
3. cd 上述路径，找到对应的版本目录，进去编辑libwebp.podspec.json
4. 找到 "source": { "git": "https://chromium.googlesource.com/webm/libwebp","tag": "v0.6.1”}, 将源替换为https://github.com/webmproject/libwebp.git
5. 修改为镜像源之后指定pod install

优缺点：不用翻墙，可以使用Cocoapod集成，每次更新pod或者打包都要进行这样的操作，非常繁琐



#### C. 直接将SDWebImage源代码、libwep库拖进工程

1. 从github上下载SDWebImage源代码
2. 从[webp库](<http://downloads.webmproject.org/releases/webp/index.html>)这里下载对应系统的webp库framework集成到工程中

优缺点：不用翻墙，无法使用Cocoapod集成，会经常遗漏升级版本，而且各组件工程也无法使用