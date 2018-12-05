---
title: lipo
date: 2017-04-05 11:12:13
tags:
---


根据公司项目的需求，需要把核心代码封装成SDK(framework)。为了能够使得真机和模拟器共用，需要将SDK(framework)做成共用版本。这时候lipo就派上用场了。

### 查看SDK支持的架构(模拟器)

``` bash
ifuwo:B ifuwo$ lipo -info MeasureSDK.framework/MeasureSDK_B 
Architectures in the fat file: MeasureSDK.framework/MeasureSDK_B are: i386 x86_64
```

### 查看SDK支持的架构(真机)

``` bash
ifuwo:B ifuwo$ lipo -info MeasureSDK_B.framework/MeasureSDK_B 
Architectures in the fat file: MeasureSDK_B.framework/MeasureSDK_B are: armv7 arm64
```

### 合并

``` bash
ifuwo:B ifuwo$ lipo -create MeasureSDK.framework/MeasureSDK_B MeasureSDK_B.framework/MeasureSDK_B -output MeasureSDK_B
```
### 合并之后查看SDK支持的架构(模拟器真机共用)

``` bash
ifuwo:B ifuwo$ lipo -info MeasureSDK_B.framework/MeasureSDK_B 
Architectures in the fat file: MeasureSDK_B.framework/MeasureSDK_B are: armv7 arm64 i386 x86_64
```

### 遇到的问题

SDK的使用过程中没有任何问题，模拟器和真机都可以正常使用。但是在提交苹果审核的时候，却报错了。大概意思是说苹果不支持x86_64和i386架构。后来的解决方案就是把x86_64和i386重新剔除掉了。
	
``` bash
ERROR ITMS-90087:"Unsupported Architectures.The executable for ..../..SDK.framework contains unsupported architectures '[x86_64, i386]'"
```

### 后来得知一种解决方法:
 1. 制作一个release版本的真机SDK
 2. 制作一个debug版本的模拟器SDK
 3. 合并
