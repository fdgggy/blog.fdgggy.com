---
title: "unity3d资源管理"
date: 2019-09-18T12:05:08+08:00
description: "unity3d资源管理"
tags:
- 资源管理
- AB包
categories:
 - unity3d
keywords:
- 资源管理
- AB包
toc: true
url: "/2019/09/18/absystem/"

---
# 一、Assetbundle原理
## 简介
参考：https://blog.csdn.net/lodypig/article/category/6315960
https://zhuanlan.zhihu.com/p/25683486
https://www.jianshu.com/p/2a7c4a48aaee
http://blog.shuiguzi.com/categories/UnityKB/
https://blog.csdn.net/swj524152416/article/details/73348296
http://blog.shuiguzi.com/2017/04/18/AssetBundle_usage_pattern_1/
https://blog.csdn.net/qq_33337811/article/details/73849019

UWA 上面很多AB方面的干货
https://blog.uwa4d.com/archives/ABTheory.html

内存优化：https://blog.csdn.net/gtofei013/article/details/60580857

源码：ABsystem: https://github.com/tangzx/ABSystem

KEngine:https://github.com/mr-kelly/KEngine

unity资源包，可以把自定义的游戏对象预制件或者资源以二进制形式保存在ab文件中。支持的格式:模型，纹理，音频，场景等，推荐方式为将关联的内容制作成prefab，将整个prefab导出到AssetBundle，Unity会收集该prefab使用的关联文件，将其一并打入AssetBundle文件，并保留prefab中资源和脚本相互关联。
![ce1fdde848174259878e38905bab1cfd](/img/hugo/2019/Unity3d资源管理.resources/877833F6-8B31-42FE-A54F-3BEDDF0FEDBA.png)

## 内部格式
![c77b73c7b30879d51686d9f8c474a290](/img/hugo/2019/Unity3d资源管理.resources/15464F92-1DFA-4A93-9CD1-123C8BB14BDB.png)

![3e19969e3d959ca549de7c6e2dd581a7](/img/hugo/2019/Unity3d资源管理.resources/62315657-2B9C-44E3-BAF1-CC7221C9A651.png)
![3d91ebdb77daab8a05c6d9df826cc9b7](/img/hugo/2019/Unity3d资源管理.resources/9A9584E1-7BEA-4D9E-9718-5DF084E5EFA5.png)

LZMA就是ZIP格式，相比于LZ4压缩格式，占用空间小，Unity解析慢
# 二、Assetbundle导出
## 编辑器导出
参考：
https://blog.csdn.net/lodypig/article/details/51871510
## 脚本导出
1.遍历指定目录下的预制件，把共有依赖资源单独记录，每一个AB记录其实是一个AssetBundleBuild结构体，最后统一打包
```csharp
    //  AssetBundle building map 实体.
    public struct AssetBundleBuild       
    {
        public string assetBundleName;         // AssetBundle 名称.
        public string assetBundleVariant;      // AssetBundle 变体.     
        public string[] assetNames;            // 该AssetBundle包含的资源路径列表。
    }
```
2.打包相关
把第一步统计的待打包向用如下API打包
```csharp
    //  AssetBundle building map 实体.
    public struct AssetBundleBuild       
    {
        public string assetBundleName;         // AssetBundle 名称.
        public string assetBundleVariant;      // AssetBundle 变体.     
        public string[] assetNames;            // 该AssetBundle包含的资源路径列表。
    }

```
3.打包选项
![2abfa3b717abee61d55bda2e8e6f24cc](/img/hugo/2019/Unity3d资源管理.resources/5A994F59-08A1-4220-8A8C-78E0AABB6BE5.png)
注意：热更资源(lua/Asset)下载接口时有个传版本参数的接口，可以自己在打包的时候做版本管理，或者直接加入打包选项DeterministicAssetBundle，通过资源HashID来做版本号，不然会出现热更失败的现象，切记。  

4.BuildTarget  
4.1 AB包在不同平台不兼容，需要分别导出，打包时设置BuildTarget即可.  
4.2 在windows下设置Unity的平台，可选Android或IOS，打包成相应资源，再打成APK，IOS需要导出成XCode工程，苹果包很麻烦.  
4.3 注意：切成Android平台在编辑器下加载的AB资源会出现材质丢失现象，那是因为打出来的AB包针对Android格式，不针对Windows平台
## mainfest
1.每生成一个AB包，就会生成一个.mainfest文件，记录这个AB包的相关信息，包括包含资源和依赖资源.  
2.同时在输出AB包的目录下会生成一个同名.mainfest文件，这个清单文件记录所有AB包的信息，在加载AB包时会用到
## 开发模式，发布模式
问题：如果用AB作为热更方案，那在开发阶段每次调试都要打成AB包，再测试？疯了。达到的目的是，开发时直接加载预制件，发布时加载AB包。   
解决：   
1.第一种方案是把预制件放入Resource目录，开发时调用同步加载接口，发布时把Resource目录改名，导出AB，再打成apk。略显麻烦。   
2.第二种方案是预制件不放入Resource目录，每次修改了资源，比如增加了一个UI预制件，通过脚本扫描一边，作用是记录每个资源的对应关系
```json
"player1_atk_pugong1_Sf_E":{
"abName":"player1_atk_pugong1_sf_e.unity3d", 
"md5":"9a89798fca5cc782f42b9ed2051fcbe6", 
"path":"Assets/EffecRes/Prefab/rolePrefab/player1/player1_atk_pugong1_Sf_E.prefab"
}
```
有了这个配置文件，在编辑模式下用ResourceMgr加载时可以通过如下方式同步加载，移动端时通过AB加载的API加载。
```csharp
if (m_pf2Asset.ContainsKey(prefabName))
            {
                PrefabToAbData preabdata = m_pf2Asset[prefabName];
                Object prefab = UnityEditor.AssetDatabase.LoadAssetAtPath<Object>(preabdata.path);

                if (callBack != null)
                {
                    callBack(prefab);
                    callBack = null;
                }

            }
            else
            {
                DebugLog.LogError("点击->生成预制件索引,prefabName get null:" + prefabName);
            }
```
# 三、Assetbundle依赖&加载
### 依赖
Unity 巨坑之一，稍不注意，就会留下难忘的踩坑记忆
#### 打包时收集依赖
Unity5默认开启自动收集依赖，打包选项为BuildAssetBundleOptions.CollectDependencies，比如一个prefab引用一张图片，unity5会自动将该图片也一起打包，避免遗漏，可是，如果多个预制件依赖这个图片，那么这个图片就存在2份，造成资源冗余。

解决：打包时将公用资源，比如图片，材质，单独打成AB包，引用到的AB依赖这个新包
### 加载
依赖关系：A->B-C.  
1.加载顺序，先加载C，再加载B，再加载A，加载依赖时只需要AB对象在内存即可，不需要加载里面具体的资源.  
2.依赖查找
缓存总的清单文件，使用manifest.GetAllDependencies（）来获取具体依赖AB包.  
3.Shader依赖加载
![aace09adf769f48391c45ca68a3caede](/img/hugo/2019/Unity3d资源管理.resources/24E7BC12-3C06-48FC-A45F-F3629BA13B1B.png)
4.加载方式.  
    4.1 特殊路径.  
    Resources可以在工程根目录，也可以在自目录，目录下所有资源将打包进游戏存放的archive中。所有资源会被压缩，只读不可写。使用Resource.Load()加载.  
    StreamingAssetPath必须在Asset根目录下，也会打包进游戏，不压缩，只读不可写
    ![a68b6025f54f63d545f6b43efb3d2276](/img/hugo/2019/Unity3d资源管理.resources/0DBE23B8-6C5E-4895-875D-A7DB5E28C05C.png)
    可以通过www加载，必须加上文件协议，安卓下必须用www加载，因为在安卓下时在jar包内file:///+streamingPath+"/"+path    
    persistentDataPath持久化目录，可读可写
    ![481e2831f44986a025b9ad5185883161](/img/hugo/2019/Unity3d资源管理.resources/8054F2EA-52FB-48BB-AD63-681554AA2F73.png)    
    DataPath应用程序目录，即Asset
    ![e1b349dcb41e74d840cf1dba66a8a703](/img/hugo/2019/Unity3d资源管理.resources/05E2F96F-7077-4F58-9556-7B86CAE8056C.png)    
    4.2 同步加载
    ![9fca897cc1172e8894914d999781dcfa](/img/hugo/2019/Unity3d资源管理.resources/1A13B9AA-8B1B-470C-8DB1-0088916FF657.png)    
    4.3 异步加载
    ![e894decf0e46bb1a3c430268319d5fda](/img/hugo/2019/Unity3d资源管理.resources/C747C21F-5C3D-4014-B6EA-8FAD948056C8.png)    
    ![043bfdf27c8c5a2fdbc0542553da78b2](/img/hugo/2019/Unity3d资源管理.resources/00D2D494-49BD-4298-B780-5D64C8C09FB5.png)
    4.4 AB内的Asset加载
    ![8122fb8e8b19f483a7eeac0d444caf71](/img/hugo/2019/Unity3d资源管理.resources/B657AA3C-9C42-49C0-AEDC-5F95A12D4208.png)
    注意：一般流程是把Streaming目录下的zip包拷贝到持久化目录，然后调用异步加载接口，用www加载时都要统一加前缀， file:///
# 四、Assetbundle内存
![40ae8a54c3dcdb33a8d0bdafb3e65b66](/img/hugo/2019/Unity3d资源管理.resources/70C8E1A8-1BB0-47BF-9297-F8C766949ABF.png)
![3f9a61ee222831c6e913b2a09febc506](/img/hugo/2019/Unity3d资源管理.resources/9262A457-3619-4CF6-A6DC-F605F69301AF.png)
![fd7317e861d07714c6f3e6e894c7d1e8](/img/hugo/2019/Unity3d资源管理.resources/212D563E-92A7-495B-B7BC-1028F40DB5B2.png)
![955f0a5cc5d805eaa6f27314576c90ef](/img/hugo/2019/Unity3d资源管理.resources/010819A8-9679-45AB-BC90-8CC8F987AFF0.png)
![77d64be7a1669c8cae99531b7008a5ae](/img/hugo/2019/Unity3d资源管理.resources/32311259-37BF-4C55-8F2C-512261DD0D51.png)
![3cc073dd9697377fb4529414f61be89f](/img/hugo/2019/Unity3d资源管理.resources/A0D0166E-FD2D-4931-91DF-54EBE42F29C6.png)
![02944bfa70b4e8e0ea58a4285347ddc8](/img/hugo/2019/Unity3d资源管理.resources/C84DB0E7-8CBD-4F1F-9CE2-3BBB9F970EBC.png)
![68116a9482fdf2ebe52473595cf3a5f8](/img/hugo/2019/Unity3d资源管理.resources/0CED6B66-F2DA-4E16-BE1E-3A219D4A9DCF.png)
![c75f5c94ff83ddf24f70d1654897ab3e](/img/hugo/2019/Unity3d资源管理.resources/DD617C25-0754-4F7D-AF30-995CA9BC911A.png)

# 五、AB释放策略
参考：
http://blog.shuiguzi.com/2017/04/18/AssetBundle_usage_pattern_1/
对官方ab包讲解的翻译

## 目的
手动管理内存，清空不需要的内存
## 手段
经过上述文章介绍后得出一个基本结论，AB.unload(true)卸载AB镜像和加载出来的Asset，不包括通过Asset实例化得对象，AB.unload(false)只卸载AB镜像。
## 策略
1.AB加载好并LoadAsset后，就可以AB.Unload(false)，采用延迟卸载，防止过段时间再次加载，延迟卸载还有个原因就是不能打断unity的异步加载，如果切换场景时，打断未加载完成的操作，则unity会崩溃，unity的bug,所以采用延迟卸载，自己来管理。  
2.引用计数，A->B->C 加载A时需先加载B,加载B时先加载C,最后加载A,如果B被卸载，则A加载失败，固每一个AB当引用计数为0时才卸载.  
3.轮询检测要卸载的资源，切换场景并不手动处理引用计数，只删掉每个AB包外部加载请求，因为是异步回调，又在释放对象池，具体看代码。  
4.AB在内存可以通过profile查看，内存取样，NOT_SAVED查看，手机通过wifi连接查看，因为最终AB都会卸载，实例化后引用计数减1，小于1时就加入卸载到期时间，间隔时间轮询到时引用计数为0且过期就会卸载，同时会把所有引用资源的引用计数-1，引用减为0的时候也会加入卸载时间。如果发现有未卸载的AB则为代码bug,场景AB和lua AB除外，这一点会引起bug,深刻的领悟。   
5.切换场景时，unity会去清空所有未引用的对象，只有ab需要自己unload，所以不用Resource.unloadunuse，如果使用了，unity偶尔还会引起丢失引用的报错，且这个函数开销也很大，慎用。

资源管理花费了很多精力来理解，涉及unity内存管理，代码已撸完，可以去https://github.com/fdgggy/supershoot 项目查看资源管理模块, 查看了很多大佬写的文章，希望对未来有帮助.