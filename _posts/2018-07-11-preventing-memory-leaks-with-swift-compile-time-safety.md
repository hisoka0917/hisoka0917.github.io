---
layout: post
title: 你经常忘记写[weak self]吗？这里有一个解决方案
date: 2018-07-11
categories: swift
tags: [swift, delegation, closure, retain cycles, generics]
---

> 感谢原文作者Oleg Dreyman授权翻译本文
> 原作者：Oleg Dreyman
> 原文链接：https://medium.com/anysuggestion/preventing-memory-leaks-with-swift-compile-time-safety-49b845df4dc6

*这篇文章将讨论基于闭包的委托、循环引用和范型*

今天我们讨论一下如何让Swift的委托(delegation)变得更优雅。我们先来看一下Swift中标准的Cocoa风格的委托例子。

1.首先我们创建一个仅限于类的委托协议

```swift
protocol ImageDownloaderDelegate: class {

	func imageDownloader(_ imageDownloader: ImageDownloader, didDownload image: UIImage)
	
}
```

2.然后我们来实现`ImageDownloader`

```swift
class ImageDownloader {
    
    weak var delegate: ImageDownloaderDelegate?
    
    func downloadImage(for url: URL) {
        download(url: url) { image in
            self.delegate?.imageDownloader(self, didDownload: image)
        }
    }
    
}
```

注意`delegate`被标记为`weak`来防止循环引用。假如你在阅读本文，那么你应该知道为什么我们需要这么做。如果你不知道，那么你应该看看[NatashaTheRobot](https://medium.com/@natashatherobot)写的这篇文章[iOS: How To Make Weak Delegates In Swift.](https://www.natashatherobot.com/ios-weak-delegates-swift/#)

3.接下来我们实现一个`ImageDownloader`的使用者

```swift
class Controller {
    
    let downloader = ImageDownloader()
    var image: UIImage?
    
    init() {
        downloader.delegate = self
    }
    
    func updateImage() {
        downloader.downloadImage(for: /* some image url */)
    }
    
}

extension Controller: ImageDownloaderDelegate {

    func imageDownloader(_ imageDownloader: ImageDownloader, didDownload image: UIImage) {
        self.image = image
    }
    
}

```

好了，我们有了一个干净、熟悉的API，同时我们不必担心内存泄漏因为我们的`delegate`是`weak`的。说明一下，这个例子非常好，完全没有问题。

那么这篇文章的点在哪儿呢？

### 现代Swift

如今这种Cocoa风格的委托模式在Swift开发者中越来越不流行了。原因很简单：这种代码看起来不是很**“现代”**，它需要大量的样板并且缺乏灵活性（举个例子：尝试一下给范型编写这样的委托）。

这就是为什么越来越多的开发者选择通过闭包委托（delegation through closure）模式。我们来应用到`ImageDownloader`的例子中。

首先，我们删除`ImageDownloaderDelegate`协议以及`ImageDownloader`中的委托。取而代之的是一个闭包属性。

```swift
class ImageDownloader {
    
    var didDownload: ((UIImage) -> Void)?
    
    func downloadImage(for url: URL) {
        download(url: url) { image in
            self.didDownload?(image)
        }
    }
    
}
```

相应的修改我们的`Controller`

```swift
class Controller {
    
    let downloader = ImageDownloader()
    var image: UIImage?
    
    init() {
        downloader.didDownload = { image in
            self.image = image
        }
    }
    
    func updateImage() {
        downloader.downloadImage(for: /* some image url */)
    }
    
}
```

我们的代码现在更加紧凑，并且（可能）更容易阅读和推理。但是有经验的开发者会注意到这里有个问题：**内存泄漏了！**

当我们取消`ImageDownloader`里的`delegate`属性时，我们同时失去了`weak`的特性。所以现在`Controller`持有一个`ImageDownloader`的引用，同时`ImageDownloader`的`didDownload`闭包里面也持有一个`Controller`的引用。这是经典的循环引用的定义并且伴随着内存泄漏。

有经验的开发者会知道一个简单的解决方案：使用`[weak self]`！

```swift
class Controller {
    
    let downloader = ImageDownloader()
    var image: UIImage?
    
    init() {
        downloader.didDownload = { [weak self] image in
            self?.image = image
        }
    }
    
    func updateImage() {
        downloader.downloadImage(for: /* some image url */)
    }
    
}
```

现在代码又可以正常运行了。

### 但是...

你可以看到，从API设计的角度来讲，这种新方法实际上使事情变得更糟。之前`ImageDownloader`的设计者负责防止产生内存泄漏。现在这是API使用者的独家义务。

而`[weak self]`是非常容易被忽视的东西。我确信这个世界上有很多代码（即使在生产环境）会遇到这个简单却无所不在的问题。

[Swift API 设计指南](https://swift.org/documentation/api-design-guidelines/#fundamentals)指出：

> 方法和属性等实体仅声明一次，但可以重复使用。

这点很重要。我们甚至无法保证在自己的app里每个需要的地方都加上`weak self`。我们当然不能指望我们所有的潜在API用户都这样做。作为API设计者，我们需要关注使用时的安全性。

当然，Swift可以让我们使用更好的方法。

我们来看问题的核心点：在分配委托回调时，99%的时候都应当有一个`[weak self]`捕获列表。但实际上并没有什么能阻止我们忽略它。没有error，没有warning，什么都没有。我们如何才能强制执行正确的行为呢？

这正是我在讨论的最基本的方式：

```swift
class ImageDownloader {
    
    private var didDownload: ((UIImage) -> Void)?
    
    func setDidDownload<Object : AnyObject>(delegate: Object, callback: @escaping (Object, UIImage) -> Void) {
        self.didDownload = { [weak delegate] image in
            if let delegate = delegate {
                callback(delegate, image)
            }
        }
    }
    
    func downloadImage(for url: URL) {
        download(url: url) { image in
            self.didDownload?(image)
        }
    }
    
}
```

于是现在我们的`didDownload`是私有的了，并且使用者需要调用`setDidDownload(delegate:callback:)`，它将传进来的委托对象包装成弱引用，从而强制执行我们想要的行为。以下是`Controller`的样子：

```swift
class Controller {
    
    let downloader = ImageDownloader()
    var image: UIImage?
    
    init() {
        downloader.setDidDownload(delegate: self) { (self, image) in
            self.image = image
        }
    }
    
    func updateImage() {
        downloader.downloadImage(for: /* some image url */)
    }
    
}
```

欢呼吧，没有循环引用也没有内存泄漏，非常棒！我们的代码仍然紧凑且易读，而且在内存泄漏方面更安全。 并且没有丑陋的`[weak self]`！

注意：这种智能技术实际上也被`Foundation`的[UndoManager](https://developer.apple.com/documentation/foundation/undomanager/2427208-registerundo)（以及其他一些Cocoa API）使用。

### 更深入一步

不过这还是不够理想。我们需要在`ImageDownloader`里面写很多额外的、不能重用的代码。利用Swift范型的强大功能，我们可以做得更好：

```swift
struct DelegatedCall<Input> {
    
    private(set) var callback: ((Input) -> Void)?
    
    mutating func delegate<Object : AnyObject>(to object: Object, with callback: @escaping (Object, Input) -> Void) {
        self.callback = { [weak object] input in
            guard let object = object else {
                return
            }
            callback(object, input)
        }
    }
    
}
```

这样样板就消失了，我们重新回到了一个非常薄的API。

```swift
class ImageDownloader {
    
    var didDownload = DelegatedCall<UIImage>()
    
    func downloadImage(for url: URL) {
        download(url: url) { image in
            self.didDownload.callback?(image)
        }
    }
    
}
```

而且`Controller`的代码看上去更好了（从我的观点看）

```swift
class Controller {
    
    let downloader = ImageDownloader()
    var image: UIImage?
    
    init() {
        downloader.didDownload.delegate(to: self) { (self, image) in
            self.image = image
        }
    }
    
    func updateImage() {
        downloader.downloadImage(for: /* some image url */)
    }
    
}
```

现在我们只用了14行代码就帮助我们避免了大量无意的内存泄漏。

这就是我喜欢Swift和她的类型系统的地方。它挑战我用有创意的设计来对抗常见的陷阱。`DelegateCall`API说明了一切，它使我们能够编写干净、富有表现力和安全的代码。

这项技术结合了Cocoa风格的委托和基于闭包的委托的最好的部分，同时解决了它们的大多数问题。我一直在使用它。于是我决定将它作为一个包发布。它具有更通用的语法，称为`Delegated`。一定要看一看：[dreymonde/Delegated](https://github.com/dreymonde/Delegated)。

感谢阅读这篇文章。有任何问题或建议可以写在评论里。


