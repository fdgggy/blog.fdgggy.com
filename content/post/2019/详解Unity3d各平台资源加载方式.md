---
title: "详解unity各平台资源加载"
date: 2019-09-10T12:05:08+08:00
description: "详解unity各平台资源加载"
tags:
- 资源加载
- unity
categories:
 - unity3d
keywords:
 - 资源加载
toc: false
url: "/2019/09/16/resource/"

---

相信大家在基于Unity3d开发游戏时会对各种平台资源加载方式会有疑惑，至少我曾经就有这样的疑惑，这里分享出来。

一、Resources目录加载    

* Resources.Load() 只能加载Resources目录的资源，可以有多个子Resources目录。
* 加载资源方式可以是同步或异步。
* Resources目录是Unity自动识别目录，打包时会把目录下所有资源全部包含打包进安装包，而且不会处理资源依赖关系，也就是相同依赖资源会存在多份。
* 打包进安装包的资源加密且压缩。

二、StreamingAssets目录加载(只读目录)  

* 在把资源(预制件)生成AssetBundle时产生的目录，这个文件夹中的资源在打包时会原封不动的打包进去，不会压缩
* Android平台下的目录地址：jar:file:///data/app/xxx.xxx.xxx.apk/!/assets
* 只能www读取，直接new www(Application.streamingAssetsPath )，会自动加jar:file://协议 (安卓下该文件夹在jar包内部，协议是jar:file://, Application.streamingAssetsPath直接返回该协议)
* Windows平台下的目录地址: Assets/Game/GameName
* 直接用FileUtils.readFileBytes读取
* www读取，www协议加"file:///", path : "file:///"+Application.streamingAssetsPath
* ios平台下的目录地址: Application/xxx.app/Data/Raw 加载方式同windows

三、持久化目录加载(可读可写)  

* Android平台下的目录地址：/data/data/xxx.xxx.xxx/files，根据手机不同地址略有不同
* Windows: C: ...\Game\game
* IOS:Application.../Documents

以上平台都可以通过   

* FileUtils.readFileBytes读取
* www读取，www协议加"file:///", path : "file:///"+Application.persistentDataPath
* AssetBundle.Load，同步或者异步加载，建议用这个新接口加载AB包。
* 涉及热更新时资源一般都是放在持久化目录，可读可写的。这一篇不是讲热更新，后期会有专门的针对资源和代码的热更新内容。

四、开发环境资源加载   

一般大型游戏，资源同时存在于Resources目录和非Resources目录，非Resources目录的资源一般会生成为AB包供热更使用，开发环境时不可能每次去生成AB包，那样效率太低。

用这个API就能解决非Resources目录资源的加载，UnityEditor.AssetDatabase.LoadAssetAtPath<Object>(path);注意，路径是相对Assets的路径，且加载具体资源的后缀要带上，如 .../asset.prefab