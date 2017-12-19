---
layout: post
title:  "Swift中的可选类型?和隐式解析可选类型!"
date:   2017-10-17 19:15:00 +0800
categories: swift
tags: swift
---
# Swift中的可选类型`?`和隐式解析可选类型`!`

### 可选类型`?`
开发中经常会需要处理变量值为空的情况，Swift作为一种类型安全的语言很重视值为空的情况。于是引入了可选类型。
可选类型的值可能有值，也可能为空值。所以必须要处理可能为空值`nil`时的情况

```swift
let possibleString: String? = "Hello"
if let actualString = possibleString {
    // actualString是普通（非可选）字符串，其值与possibleString中的值相同
    print(actualString)     //输出"Hello\n"
} else {
    // possibleString为空值时的情况
}
```
可选类型的值是可选值，这在字符串类型中需要特别注意

```swift
let possibleString: String? = "Hello"
print(possibleString)  //编译器会提示“Expression implicitly coerced from 'String?' to Any”；
// 此时打印出来的值是"Optional("Hello")"
```
如上例所示，如果直接使用可选字符串，那么值将会包含在`Optional("")`中。此时就需要使用强制解析符号`!`

```swift
let possibleString: String? = "Hello"
print(possibleString!)   // "Hello\n"
```
使用可选类型可以让代码更加严谨，并且在每个使用的地方强迫你去思考这里是否可能为空。

```swift
// 应用场景
// 错误写法: let url: URL = URL(string: "http://www.baidu.com")
// 因为URL的构造函数可能会失败，所以它的返回值是URL?类型
// 正确写法: let url: URL? = URL(string: "http://www.baidu.com")

// 使用类型推导
let url = URL(string: "http://www.baidu.com")

/* 不推荐写法
if url != nil {
    URLRequest(url: url!)
}
*/

// 该语法称为可选绑定（如果url有值就解包赋值给tempUrl，并执行{}）
if let tempUrl = url {
    URLRequest(url: tempUrl)
}
```

### 隐式解析可选类型
如上所示，可选类型在使用中需要判断是否为空值，那么这就带来一个问题：怎么确定一个可选类型是有值的呢？
比如像这样

```swift
var dog: String? = "wangcai"
let cat: String = dog   // Error: Value of optional type 'String?' not unwrapped; did you mean to use '!' or '?'?
```

如上，变量cat在接受赋值时无法确定dog是不是nil，因为它是可选类型，所以编译器报错。
所以，就像上文所说使用强制解析`!`来表示我有值。或者cat也声明为可选类型

```swift
var dog: String? = "wangcai"
let cat: String? = dog
```
为了省心，我们使用强制解析`!`，但是以下情况就不省心了

```swift
var dog: String? = "wangcai"    //dog变量想要给一个非可选类型的cat变量赋值
let cat: String = dog!          //cat说，嗯，我已经拿到啦，么么哒！
let tiger: String = dog!        //hello，我是旺财～
let fish: String = dog!         //我是旺财～
let bird: String = dog!         //旺财。。。
let sheep: String = dog!        //为什么每次都得验证一遍
let mouse: String = dog!        //心好累
```
为了不出现满屏的`!`，我们可以采用隐式解析可选类型

```swift
var dog: String! = "wangcai"    //dog变量想要给一个非可选类型的cat变量赋值
let cat: String = dog           //cat说，嗯，我已经拿到啦，么么哒！
let tiger: String = dog         //ok
let fish: String = dog          //ok
let bird: String = dog          //ok
let sheep: String = dog         //ok
let mouse: String = dog         //ok
```
看，将dog声明为隐式解析可选类型`String!`省去了满世界的`!`。

那么我们是否一直使用隐式解析可选类型就万事大吉了呢？答案是*No*。因为隐式解析可选类型还是可选类型，它是可以为空的。如果在空值时被使用，程序就会**Crash**。

```swift
var url: URL! = URL(string: "www.baidu.com")
url = nil   // nothing happened
URLRequest(url: url)    // fatal error: unexpectedly found nil while unwrapping an Optional value
```

#### 什么时候使用隐式解析可选类型？
1. 初始化过程中不能定义的常量

每一个成员常量在初始化完成后一定会有一个值。然而有的时候在初始化过程中不能确定会有正确的值，但是又可以确保在被访问前一定会有值。<br>
使用可选变量不会有问题，因为可选变量在初始化时会自动被赋予`nil`，并且在初始化结束时被赋予正确的值。但是使用时的强制解包会很痛苦，满屏幕都会是`!`。<br>
以下展示了一个成员变量在视图加载完成之前不能完成初始化的例子：

```swift
class MyController: UIViewController {
    @IBOutlet var button: UIButton!
    var buttonOriginalWidth: CGFloat!
    
    override func viewDidLoad() {
        self.buttonOriginalWidth = self.button.frame.size.width
    }
}
```
这里有2个隐式解析可选类型。IBOutlet类型是Xcode强制为可选类型的，因为它不是在初始化时赋值的，而是在加载视图的时候。你可以把它设置为普通可选类型，但是如果这个视图加载正确，它是不会为空的。<br>
在视图加载完成之前，我们无法知道button的初始宽度。但是我们知道`viewDidLoad`方法会在任何方法之前被调用（除了初始化函数）。与其在所有地方强制解包，不如定义成隐式解析可选类型。

2. 当程序不能在某个变量为`nil`后恢复
这种情况非常罕见。如果应用程序在访问某个空的变量时无法继续运行，那么也就无需再判空了。通常情况下，如果程序必须符合某个条件才能继续运行，则可以使用断言。

#### 什么时候不使用隐式解析可选类型？

1. 延迟加载计算类型成员
有时候我们需要永远不为空的成员变量，但是它可能在初始化时不能被正确赋值。一种方法是使用隐式解析可选类型。不过更好的方法是使用延迟加载：

```swift
class FileSystemItem {
}

class Directory: FileSystemItem {
    lazy var contents: [FileSystemItem] = {
        var loadedContents = [FileSystemItem]()
        // load contents and append to loadedContents
        return loadedContents
    }()
}
```
如上代码所示，`contents`不会被初始化直到它第一次被访问。

2. 另外任何地方
在大多数情况下，要避免使用隐式解析可选类型，因为一旦使用错误就会造成程序**Crash**。如果不确定一个变量是否会是`nil`，就默认使用普通可选类型。去解包一个不会是`nil`的变量不会有什么副作用。


## 参考资料
1. [an drewagner: Uses for Implicitly Unwrapped Optionals in Swift](https://drewag.me/posts/2014/07/05/uses-for-implicitly-unwrapped-optionals-in-swift)
2. [Liwx: Swift可选类型](http://www.jianshu.com/p/6f74174ff81f)
3. [民谣程序员：Swift- 可选类型，隐式解析可选类型](http://www.jianshu.com/p/3a2f66be4399)

