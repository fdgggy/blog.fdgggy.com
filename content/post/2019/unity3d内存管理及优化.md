---
title: "unity内存管理及优化"
date: 2019-09-10T18:05:08+08:00
description: "unity内存管理及优化"
tags:
- 内存管理
- 优化
categories:
 - unity3d
keywords:
 - 内存管理
toc: false
url: "/2019/09/10/memmgr/"

---

## unity为什么能跨平台
参考：https://segmentfault.com/a/1190000004355051
### MONO
* CLI 通用语言基础架构，是一个技术规范，定义了与语言无关的跨体系结构的运行环境。开发者只需要按照规范内各种高级语言来开发软件，即可实现跨平台。规范中包括CIL，可读性较低的通用中间语言。
* .NET是微软对CLI的一种实现，只能在windows运行
* Mono是跨平台的，是对CLI的另一种实现，mono运行时基于堆栈。运行的是CIL语言。
* CIL 通用中间语言，能运行在所有支持CLI的环境中。也就是可以同时运行在mono或者.net，和具体平台无关，这样就可以跨平台。
0.选择的语言(c#,vb,boo)用不同的编译器编译为CIL
1.代码编译为CIL
2.运行时(mono)把CIL用不同的编译器编译成不同平台的本机原生代码，实现跨平台。
* mono 编译
1.JIT即时编译，动态编译，程序执行时才编译代码，并解释执行。会对编译过的代码缓存。IOS平台不允许JIT。所以Unity官方无法提供热更方案。
2.AOT静态编译，在程序执行之前就编译好了。  

![9f7fe79dbabd425af8835eb028170664](/img/hugo/2019/unity3d内存管理及优化.resources/E43881D6-B3A3-4F9F-B0D3-9FF8170B3BDA.png)
### IL2CPP
MONO为啥要换成IL2CPP：
1.mono维护成本大
2.mono版本授权受限。
3.提高运行效率，换成IL2CPP后，程序运行效率提升1.5-2.0倍

![317a6739face44234e28297c994794e3](/img/hugo/2019/unity3d内存管理及优化.resources/DFDD8E91-8FE6-4834-B209-9A7D912EB836.png)
IL2CPP将IL变回C++，再由各个平台的C++编译器编译成本地能执行的原生汇编，在IL2CPP虚拟机环境运行。

* IL变回CPP，可以利用各平台编译器对代码执行编译期优化。程序更小更快。
* IL2CPP VM 负责提供GC管理，内存管理逻辑和mono一样，线程创建等。
* 用了IL2CPP编译就是AOT静态编译了，IOS，Xbox，ps都不支持JIT。

## MONO内存/托管堆内存
参考：https://wetest.qq.com/lab/view/135.html
![79c93df49c80bb647cd6398dd6013819](/img/hugo/2019/unity3d内存管理及优化.resources/50AE2645-144B-4076-AD97-7B96B50338B7.png)
c#代码编译为IL中间语言，运行在MONO运行时，内存由MONO进行分配和管理。由MONO自动改变堆的大小，适时调用垃圾回收(GC)操作释放不需要的内存。
### MONO内存管理策略
![2a1cefe5888c798544932661e9ea2e91](/img/hugo/2019/unity3d内存管理及优化.resources/59A820E9-727C-4ED6-87CC-6E9A650C4DA7.png)

* mono内存分为已用内存(used)和堆内存，堆内存指mono向OS申请的内存。
* 当需要mono分配内存时，会先查看空闲内存是否足够，不够的话mono会进行一次GC操作释放更多空闲内存，如果还是不够，则mono会向OS申请内存，并扩充堆内存。  

#### GC
GC的主要作用是从已使用内存中找出不再需要使用的内存，并进行释放。步骤：  
1.停止所有需要mono申请内存的线程   
2.遍历所有已用内存，找到那些不再需要使用的内存，并标记    
3.释放被标记的内存称为空闲内存   
4.重新开始被停止的线程
##### 触发GC的条件
* 空闲内存不足，mono会自动调用GC
* 手动通过GC.Collect()触发   
1.GC本身比较耗时，因为会暂停那些需要mono分配内存的线程(c#创建的线程和主线程)，因此无论是否在主线程调用，GC都会导致游戏卡顿，帧率下降。   
2.GC释放的内存只会留给Mono，并不会还给os，因此mono堆内存是只增不减的。

### mono内存泄露分析
#### GC原理
![4429cec77348d03715b00090af706ed5](/img/hugo/2019/unity3d内存管理及优化.resources/93CE99C0-42E7-4FD6-A3FC-C777AA1A98B2.png)

* mono是通过对象的引用关系来判断不需要的内存。每次分配内存时，维护一个分配对象表，当GC的时候，以全局数据区和当前寄存器中的对象为根节点，按照引用关系遍历，遍历到的对象标记为alive，没有遍历到的对象则进行回收。
* 上图中的E,F将会在GC过程中回收。
#### 内存泄露
##### 代码侧泄漏及优化
对象不再使用，超出其作用域时，但没有被GC回收称为内存泄露，MONO内存泄露会使空闲内存减少，GC频繁，MONO堆不断扩充，最终导致游戏内存占用升高,导致应用崩溃。

* 猜测+不断修改代码测试，对比内存分配情况，效率很低，可以用cube工具对mono内存快照对比，定位mono内存泄露。
注意：  
1.尽量少使用静态变量，静态变量是GC的根节点，不会被GC  
2.对于不再使用的对象将其设置为Null，使其可以被GC.  
3.尽量复用对象，减少new的次数。    
4.不同类型使用string +操作时使用StringBuilder代替String，减少不必要的字符串操作，使用for代替foreach。    
5.局部变量或非常驻变量用struct代替class.    
6.避免在update里面new对象(class,container,array)，会导致gc频繁.   
7.检查游戏标签gameObject.CompareTag("Enemy")代替gameObject.tag=="Enemy"，会多余拷贝字符串，造成gc.   
8.不要在update中调用getcomponent，在awake或start中缓存变量。    
9.不要使用unity GUI,会造成大量gc alloc，禁用主摄像机的GUI Layer组件。    
10.替换Debug.Log，unity所有log类型都会跟踪堆栈调用。
##### 资源侧泄漏
资源加载后占用了内存，在资源不用后，没有将资源卸载导致内存泄漏。存在于本机堆(Native堆)中。
* 加载场景时，unity编辑器场景里所有asset,都会自动加载。切换场景时，场景中所使用的所有资源将会被unity自动卸载。
* DontDestroyOnLoad的资源本身，及依赖的资源都不会卸载。
* Resource.Load的资源，不再使用后用Resource.UnloadAsset()或Resources.UnloadUnusedAssets()卸载。尽量不要使用，因为是一个遍历操作，造成卡顿，在合适的时机卸载。会卸载AssetBundle加载后的资源和resource.load后的资源。
* Resources.UnloadUnusedAssets()内部会调用GC.Collect()，建议在加载环节调用。
* AssetBundle.load后，延迟卸载,unload(false);
* 注意AssetBundle的资源冗余，可通过工具查看。
* 对资源或者代码，在生命期结束后就要释放。
* 对于在本地缓存资源，一定要在切换场景时remove或clear掉，否则Resources.UnloadUnusedAssets无法卸载，造成资源泄漏。   

##### 优化
###### 纹理优化
1.纹理格式,根据硬件的种类选择硬件支持的文理根式。android平台用ETC1，IOS用PVRTC  

* etc1不支持Alpha透明通道，opengl es2.0的设备只支持etc1。将透明贴图分拆为2张，一张RGB24位纹理记录原始颜色部分，一张Alpha8纹理记录透明通道部分，然后将2张贴图分别转化为etc1格式的文理，并通过shader渲染。      
* pvrtc的纹理尺寸需要正方形，否则显示不出来
2.减少纹理尺寸，512\*512的显示效果足够了，就没必要用1024\*1024
3.禁用Mipmap功能。mipmap会根据摄像机的远近选择不同精度的贴图，较远的显示的贴图像素低，优点是优化显存带宽减少渲染，缺点是占用内存。
4.禁用Read&Write，决定纹理是在内存上还是显存上，开启时会使纹理内存增大。
####### AB加载
1.尽量用CreateFromFile加载，因为www会产生webstream流，里面包含ab包本身和包含的资源
2.对不用的ab包，unload(false)
3.场景切换时，先跳转空场景，再跳到目标场景。避免两个场景叠加产生内存峰值。        
### 库代码优化
参考：https://onevcat.com/2012/11/memory-in-unity3d/
* unity库和第三方库，在player setting面板里的optimization中api level设置为.NET 2.0 Subset，不需要把.NET全部的api包含进去。
* 库剥离去掉System.xml，如果要用，引入一个轻量级的xml库，如mono.xml。
* 降低打包后程序的尺寸和代码的内存占用
### 查找资源泄漏
* 通过Profile的内存快照比较，将2次内存的状态截取进行比较，寻找内存的增量和泄漏点。可以在进关卡前和出关卡后做2次dump比较。






































