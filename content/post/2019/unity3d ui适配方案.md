---
title: "unity3d的ui适配方案"
date: 2019-09-16T12:05:08+08:00
description: "unity3d的ui适配方案"
tags:
- ui适配
- ugui
- unity3d
categories:
 - unity3d
keywords:
 - ui适配
toc: true
url: "/2019/09/16/uiscaler/"

---

#### 分辨率
##### 显示分辨率
###### 概念
屏幕图像的精密度，显示器所能显示的像素数量。
- 显示分辨率一定的情况下，显示屏越小(ppi越高)图像越清晰。
- 显示屏大小固定，显示分辨率越高(ppi越高)图像越清晰。
###### 单位
- dpi 点每英寸
- lpi 线每英寸
- ppi 像素每英寸

##### 图像分辨率
- 单位英寸中所包含的像素点数，单位为ppi，像素每英寸.
- 水平像素数\*垂直像素数   

#### 为什么要做屏幕适配
市面上手机厂商很多，屏幕分辨率很多种，如果同样的图片放到不同分辨率手机上，展现出来的大小会不同。不可能给每一种分辨率都做一套UI。
#### 自适应
如：一张图片按照1280\*720(16:9)做了ui，但是屏幕尺寸是2048\*1536(4:3)，所说的自适应就是ui尺寸要适应现在的屏幕尺寸，宽度和高度都要小于屏幕实际宽度和高度，如果这张ui按照像素一一对应放在屏幕上，肯定不能充满整个屏幕。
为了ui充满屏幕，且不变形，要将ui的一个像素对应屏幕大于一个像素，也就是ui要放大。这时可以根据ui尺寸比例和屏幕尺寸比例来计算缩放比例。
#### 宽或高等比缩放
ui 尺寸是16/9=1.778，ipad 屏幕尺寸4/3=1.33
屏幕尺寸宽高比小于ui尺寸宽高比，屏幕宽/ui宽相对于屏幕高/ui高变化小，则以宽度适配，ui宽与高等比缩放,放大相同比例。
ui w = 1280
ui h = 1280\(2048/1536)=960 设计分辨率高于设计的高度，则把背景图的高度做成960即可在缩放时铺满屏幕。

宽与高同时扩大2048/1280倍，充满屏幕，ui就是等比扩大充满屏幕，不变形，一般用于全屏背景ui.

市面上手机,pad分辨率
//todo
- 当屏幕宽高比小于设计分辨率宽高比时(如ipad)，以设计分辨率的宽来适配，等比(设备宽/设计宽)缩放宽与高。
- 当屏幕宽高比大于设计分辨率宽高比时(如全面屏),以设计分辨率的高来适配，等比(设备高/设计高)缩放宽与高。

#### UGUI屏幕适配策略
##### canvas
ui布局和渲染的抽象空间，所有的ui都必须在canvas结点下才能被渲染出来。canvas大小就是所支持的设备屏幕大小。
##### render mode
- screen space -overlay 覆盖模式
ui直接显示在所有图形之上，跟camera无关

- screen space -camera 摄像机
使用camera作为参照，将ui平面放置在camera前一定距离，屏幕大小，分辨率，camera视锥体改变时ui平面会自动调整大小。

- world space
直接把对象作为世界坐标中的平面，也就是当作3d物体显示ui。

#### canvas scaler 缩放核心
canvas scaler是ui系统中控制ui元素的总体大小和像素密度的组件，canvas scaler的缩放比例影响canvas下的所有元素，包括字体和图像便捷。

##### reference resolution
预设屏幕大小，也就是设计分辨率，美术是按照此分辨率设计ui。ui适配方案都是按照屏幕分辨率和设计分辨率来计算缩放的。一般市面上很多手机的宽高比是16:9，可以设置为1280\*720, 最近几年出的全面屏是19:9(1792\*828), 适配时在1280\*720大小上整体缩放x倍。

##### scale factor
缩放因子，用于缩放整个canvas，调整canvas size与screen size一样大小。
```c#
protected void SetScaleFactor(float scaleFactor)
{
    if (scaleFactor == m_PrevScaleFactor)
        return;
 
    m_Canvas.scaleFactor = scaleFactor;
    m_PrevScaleFactor = scaleFactor;
}
```
screen size = canvas size * scale factor
##### ui scale mode

###### constant pixel size
固定像素大小
##### scale with screen size

![ba1bb26632026f182ebb06bf0358823d](/img/hugo/2019/unity3d ui适配方案.resources/0F744757-A1F9-46A8-9433-6C6647860C13.png) 

通过设定的设计分辨率来缩放，match为0则以宽适配,为1则以高适配
```c#
Vector2 screenSize = new Vector2(Screen.width, Screen.height);
 
float scaleFactor = 0;
switch (m_ScreenMatchMode)
{
    case ScreenMatchMode.MatchWidthOrHeight:
    {
        float logWidth = Mathf.Log(screenSize.x / m_ReferenceResolution.x, kLogBase);
        float logHeight = Mathf.Log(screenSize.y / m_ReferenceResolution.y, kLogBase);
        float logWeightedAverage = Mathf.Lerp(logWidth, logHeight, m_MatchWidthOrHeight);
        scaleFactor = Mathf.Pow(kLogBase, logWeightedAverage);
        break;
    }
    case ScreenMatchMode.Expand:
    {
        scaleFactor = Mathf.Min(screenSize.x / m_ReferenceResolution.x, screenSize.y / m_ReferenceResolution.y);
        break;
    }
    case ScreenMatchMode.Shrink:
    {
        scaleFactor = Mathf.Max(screenSize.x / m_ReferenceResolution.x, screenSize.y / m_ReferenceResolution.y);
        break;
    }
}
```
##### expand 扩大
将canvas size进行宽或高扩大，屏幕宽相较于设计分辨率宽变化较小，以宽适配，相反以高适配。

![189732b820457009e1568d19ad2fcbd9](/img/hugo/2019/unity3d ui适配方案.resources/D4ED1D96-4413-4CFB-9DE0-EB785B5C7FBC.png)

reference resolution 为1280\*720，screen size 为800\*600
scalefactor width 800/1280=0.625
scalefactor height 600/720=0.833
canvas size = screen size/scale factor
canvas width = 800/0.625=1280
canvas height = 600/0.625=960
高度从720变成了960，最大程度的放大显示所有元素，缩放因子则为0.625。
##### shrink 收缩
与expand刚好相反

##### 最终适配方案

reference size 1280\*720 宽高比16:9
ui scale mode expand模式









