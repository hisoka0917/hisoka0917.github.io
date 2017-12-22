---
layout: post
title:  在内联block中如何使用unowned和weak
date:   2017-10-17 19:50:00 +0800
categories: swift
tags: swift
---

本文将讨论一下在Swift中，closure中引用self时的循环引用问题，以及unowned和weak的用法。

关于iOS内存管理中经典的循环引用问题在这篇文章中就不再描述，可以参考onevcat的这篇文章:[《内存管理，weak 和 unowned》
](http://swifter.tips/retain-cycle/)

我们知道当一个闭包作为类成员的时候，如果这个闭包内访问了`self`那么就会引起循环引用，这时我们需要用`unowned`或者`weak`来避免循环引用。具体内容在onevcat的文章中有详细描述。
在这篇文章里我想讨论一下当闭包作为函数参数时会是个什么情况。为此我们写一个vc如下所示：

```swift
class SecondViewController: UIViewController {

    override func viewDidLoad() {
        super.viewDidLoad()
        self.title = "Second ViewController"
    }
    
    deinit {
        print("deinit \(self.title ?? "default value")")
    }

    @IBAction func getData(_ sender: Any) {
        ApiManager.getAsyncMockData {(dict: NSDictionary) in
            let action = UIAlertAction(title: "Cancel", style: .cancel, handler: nil)
            let actionController = UIAlertController(title: "Second ViewController Alert", message: dict.description, preferredStyle: .alert)
            actionController.addAction(action)
            self.present(actionController, animated: true, completion: nil)
        }
        self.navigationController?.popViewController(animated: true)
    }
}

class ApiManager: NSObject {
    class func getMockData() -> NSDictionary? {
        if let path = Bundle.main.path(forResource: "data", ofType: "json") {
            if let jsonData = NSData.init(contentsOfFile: path) {
                do {
                    let dict = try JSONSerialization.jsonObject(with: jsonData as Data, options: .mutableContainers)
                    return dict as? NSDictionary
                } catch {
                    print("Json parse error: ", error)
                }
            }
        }
        return nil
    }
    
    class func getAsyncMockData(complete: @escaping (_ result: NSDictionary) -> Void) {
        DispatchQueue.main.asyncAfter(deadline: .now() + 1) {
            if let dict = self.getMockData() {
                complete(dict)
            }
        }
    }
}
```
在这里模拟了异步获取数据并返回的情况。在界面上按下Button调用`getData()`，该函数中会异步获取数据，并做一些操作。在调用`ApiManager`之后直接pop自己，也就是说当异步调用数据返回的时候这个vc已经被释放了。此时会发生什么呢？在模拟器上运行这段代码，界面返回后并没有发生什么，看上去一切正常。但是控制台打出了一行字`Warning: Attempt to present <UIAlertController: 0x7fb1b204da00> on <MVVMExample.SecondViewController: 0x7fb1b0623e40> whose view is not in the window hierarchy!`
并且没有出现`deinit`字样。我们来分析一下，控制台打出这段warning说明`self.present()`被执行了，而没有`deinit`说明vc没有被释放。为了验证这一点，我们使用Instruments里的Leak工具来查看这里是否有内存泄漏。结果显示这个地方果然有内存泄漏。

![leak1]({{ site.url }}/assets/images/posts/closure-unowned-weak-self/leak1.png)
![leak2]({{ site.url }}/assets/images/posts/closure-unowned-weak-self/leak2.png)

由于我们这里是用的`self.present()`这种对view hierarchy的操作导致了内存泄漏，那么如果是一些比较轻的操作呢？

```swift
private func printTitle() {
    print("I am \(self.title ?? "default value")")
}
    
@IBAction func getData(_ sender: Any) {
    ApiManager.getAsyncMockData {(dict: NSDictionary) in
        self.printTitle()
    }
    self.navigationController?.popViewController(animated: true)
}
```
我们将代码改成这样，并且运行。控制台打印出如下信息
    
    I am Second ViewController
    deinit Second ViewController

在这里vc完成了回调的闭包操作后正确释放，似乎并没有任何问题。但是如果异步回调闭包里面有大量计算或者会阻塞主线程的操作，那么将会影响到App的运行。举个典型的例子，从网络获取数据，处理数据并reload tableView。这种情况下就会影响到主线程。

显然，我们希望vc在pop之后立即释放。于是很自然的就想到了使用weak
```swift
@IBAction func getData(_ sender: Any) {
    ApiManager.getAsyncMockData { [weak self] (dict: NSDictionary) in
        let action = UIAlertAction(title: "Cancel", style: .cancel, handler: nil)
        let actionController = UIAlertController(title: "Second ViewController Alert", message: dict.description, preferredStyle: .alert)
        actionController.addAction(action)
        self?.present(actionController, animated: true, completion: nil)
    }
    self.navigationController?.popViewController(animated: true)
}
```
再次运行。正如我们所期望的，vc正确释放并且闭包中对self的操作并没有执行。

那么如果我们使用`[unowned self]`会怎么样？结果是**Crash**。因为`unowned`相当于以前的`unsafe_unretained`，当对象被释放时，unowned引用地址不会被指向`nil`，而是维持原内存地址，而实际上该地址上的对象已经被释放，此时去访问这个地址，程序当然会崩溃。而`weak`的对象在释放后会指向`nil`，这样不会造成crash。

如何正确选择这两者的使用，Apple给我们的建议是**如果能够确定在访问时不会已被释放的话，尽量使用 unowned，如果存在被释放的可能，那就选择用 weak**。

