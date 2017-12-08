---
layout:     post
title:      Freeline加快Android Studio编译速度
subtitle:   
date:       2016-11-23
author:     XiaoZhui Wang
header-img: img/post-bg-2.jpg
catalog: true
tags:
    - Android
---

本文只是对[Freeline](https://github.com/alibaba/freeline) 官方使用说明文档的一些细节的补充，大部分使用说明是直接Copy过来的。

Freeline是由蚂蚁聚宝Android团队开发的一款针对Android平台的增量编译工具。它可以充分利用缓存文件，在几秒钟内迅速地对代码的改动进行编译并部署到设备上，有效地减少了日常开发中的大量重新编译与安装的耗时。
虽然Android Studio的Instant Run也能加快项目的编译速度，但就目前来看，它还存在一些问题，有时编译会很慢，甚至卡死。Freeline相比Instant Run编译速度更快也更稳定，Freeline编译原理详情可以看[原理说明](https://yq.aliyun.com/articles/59122?spm=5176.8091938.0.0.1Bw3mU)。

#### 一.配置工程
配置project-level的build.gradle，加入freeline-gradle的依赖：
```source-groovy-gradle
buildscript {
    repositories {
        jcenter()
    }
    dependencies {
        classpath 'com.antfortune.freeline:gradle:0.8.2'
    }
}
```

然后，在你的主module的build.gradle中，应用freeline插件的依赖：
```source-groovy-gradle
apply plugin: 'com.antfortune.freeline'

android {
    ...
}
```

最后，在命令行执行以下命令来下载 freeline 的 python 和二进制依赖。
*   Windows[CMD]: gradlew initFreeline
*   Linux/Mac: ./gradlew initFreeline

对于国内的用户来说，如果你的下载的时候速度很慢，你也可以加上参数，执行`gradlew initFreeline -Pmirror`，这样就会从国内镜像地址来下载。
你也可以使用参数`-PfreelineVersion={your-specific-version}`来下载特定版本的 freeline 依赖。

如果你的工程结构较为复杂，在第一次使用freeline编译的时候报错了的话，你可以添加一些freeline提供的配置项，来适配你的工程。在moudle的gradle文件增加如下代码
````
freeline {
    hack true
    productFlavor 'your-flavor'
    //.....其他配置项
}
````
配置项具体可以看[Freeline DSL References](https://github.com/alibaba/freeline/wiki/Freeline-DSL-References)。

#### 二.执行编译

##### 方法一：通过Android Studio插件Freeline来编译

在Android Studio中，通过以下路径Preferences → Plugins → Browse repositories，搜索“freeline”，并安装。

[![](https://camo.githubusercontent.com/8c2cd6b22e85207884dc843cb9a0b758e636953c/687474703a2f2f7777342e73696e61696d672e636e2f6c617267652f3635653466316536677731663832656b6e616575646a3230746b30316f6d78652e6a7067)](https://camo.githubusercontent.com/8c2cd6b22e85207884dc843cb9a0b758e636953c/687474703a2f2f7777342e73696e61696d672e636e2f6c617267652f3635653466316536677731663832656b6e616575646a3230746b30316f6d78652e6a7067)

直接点击 `Run Freeline`的按钮，就可以享受Freeline带来的开发效率的提升啦（当然，你可能会先需要一个较为耗时的全量编译过程）。在Freeline工具栏和Build选项里的Freeline还可以以`-d` 或`-f`方式执行编译。
插件也会提示你Freeline最新的版本是多少，你也可以通过插件来对Freeline进行更新。

##### 方法二：直接在命令行执行python脚本来编译
* python freeline.py        正常模式编译
* python freeline.py -d        调试模式编译
* python freeline.py -f        全量编译，就是整个工程重新编译

> 由于编译使用的是python脚本，所以要安装Python环境并设置环境变量，否则会提示command python not found。只支持Python 3.0以下环境。

> 如果项目被Freeline编译后，想使用Android Studio自带的编译去编译项目，启动应用时可能会发生闪退，这时候只需要先clean一下项目再编译项目，就不会发生闪退了。

补充：类似的插件还有JRebel，也很好用，但它是收费的。