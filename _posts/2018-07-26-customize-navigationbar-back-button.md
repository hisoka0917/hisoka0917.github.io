---
layout: post
title: 全局自定义Navigation Bar的返回按钮
date: 2018-07-26
categories: swift
tags: [ios, swift, navigation]
---

在开发中更改导航栏返回按钮样式是非常常见的设计。当然我们不希望在每个ViewController里设置返回按钮。有一种解决方案是写一个ViewController基类，在基类里面设置返回按钮，然后项目中所有的ViewController都从这个基类中继承。这样做并不优雅。我们希望全局设置返回按钮的图片。

最简单的全局设置返回按钮图片的方法是在`AppDelegate`的`didFinishLaunchingWithOptions`方法中写入以下代码

```swift
let barApperance = UINavigationBar.appearance()
barApperance.backIndicatorImage = UIImage(named: "navigation_back")
barApperance.backIndicatorTransitionMaskImage = UIImage(named: "navigation_back")
//barApperance.backItem?.title = " "  //在这里设置返回按钮文本并没有用
```

当然如果你的UINavigationController是在Storyboard里设置的，也可以去IB里面设置，有对应的属性。

这样就全局替换了返回按钮的图片。但是我们会遇到一个问题就是有时候我们需要获取用户点击返回按钮的事件，这样我们不得不新建一个`UIBarButtonItem`设置为`leftBarButtonItem`。但是新建的item的图片如果和我们全局设置的图片一样的话那么在页面跳转的时候2个返回按钮的图片会错位。因为新建的`UIBarButtonItem`放到NavigationBar左边的时候系统会默认留出8pt的margin。而全局默认的`backButton`左边是不留空的。所以2个不同的按钮位置就会错位。于是我们只能切2张图，一张左边有8pt的空白，一张左边不留空。全局的图片使用左边留空的那张，自定义的按钮使用不留空的图片。这样一来水平方向就对齐了。但是垂直方向还是不对的。因为系统默认的`backButton`在图片右边有一个`UILabel`，于是图片的垂直位置会往下1pt。所以我们左边不留空的那张图在垂直方向上还要往下1pt才能和系统的按钮完全对齐。

我们处理完返回按钮的图片后再来处理返回文本。因为很多设计都不想要返回按钮上的文本。我们可以用以下代码来让返回按钮上的文本不显示

```swift
let barButtonAppearance = UIBarButtonItem.appearance()

barButtonAppearance.setBackButtonTitlePositionAdjustment(UIOffset(horizontal: -500, vertical: 0), for: UIBarMetrics.default)
```

这段代码是让返回按钮的文本往左偏移500pt，这样文本就在屏幕外面了。不过这样做有一点点小的问题，就是在页面跳转动画的时候是看得到title的横向移动的。强迫症表示不是很舒服。

于是我们就想把返回按钮的文本颜色设置成透明

```swift
let barButtonAppearance = UIBarButtonItem.appearance()

barButtonAppearance.setTitleTextAttributes([NSAttributedStringKey.foregroundColor: UIColor.clear], for: .normal)

barButtonAppearance.setTitleTextAttributes([NSAttributedStringKey.foregroundColor: UIColor.clear], for: .highlighted)
```

这样一来转场动画时title的移动就舒服多了。

