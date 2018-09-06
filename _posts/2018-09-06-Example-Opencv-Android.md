---
layout: post
title: "android opencv JNI开发环境搭建"
date: 2018-09-06
description: "android opencv JNI开发环境搭建"
tag: opencv
---  

> 千里之行，始于足下。

网上很多opencv4android的教程都是通过导入opencv的java module进行开发的，因为之前一直在看opencv c++ 的开发流程，所以采用了JNI的方法直接编写c++代码来学习opencv。
### 新建项目

首先需要安装CMake以及NDK环境，没有安装的的可以直接百度。

新建一个包含c++ support的项目，如下图所示，一路next就可以了。

![新建项目](http://papfkhd3v.bkt.clouddn.com/QQ截图20180906154706.png)

### 导入opencv

需要下载opencv4android的sdk

[opencv 3.4.1下载地址](https://opencv.org/opencv-3-4-1.html)

下载完成后将.so文件以及include导入项目的
src/main/jnilibs文件中。

.so文件在sdk的“\sdk\native\libs”目录下。

include文件在“\sdk\native\jni\include”目录下。

![导入opencv依赖](http://papfkhd3v.bkt.clouddn.com/QQ截图20180906161327.png)

为了手机的abi，我还加了armeabi-v7a，所以需要修改app的build.gradle,如果想在虚拟机上跑还需要加入x86的abi。

```
externalNativeBuild {
            cmake {
                cppFlags ""
                abiFilters "armeabi","armeabi-v7a"
            }
        }
```

### 修改CMakeLists.txt

但是现在还无法在我们的native.cpp文件中直接引用opencv的c++代码，还需要修改CMakeLists.txt。

根据自己的路径添加，libopencv_java3是你的so链接库的名字。
```
include_directories(src/main/jniLibs/include)

add_library(libopencv_java3 SHARED IMPORTED)
set_target_properties(libopencv_java3 PROPERTIES IMPORTED_LOCATION
            ${CMAKE_SOURCE_DIR}/src/main/jniLibs/${ANDROID_ABI}/libopencv_java3.so)
```
在target_link_libraries中也加入libopencv_java3
```
target_link_libraries( # Specifies the target library.
                       native-lib libopencv_java3

                       # Links the target library to the log library
                       # included in the NDK.
                       ${log-lib} )
```
完成以上步骤sync 项目之后，我们就可以开始愉快的编码了。
如下图所示，可以使用opencv c++的api了。

![QQ截图20180906163141.png](http://papfkhd3v.bkt.clouddn.com/QQ截图20180906163141.png)

[github地址](https://github.com/Zellerpooh/OpencvExample)
### 参考文章
[Android智能识别 - 银行卡区域裁剪](https://www.jianshu.com/p/a763eeb8159f)
