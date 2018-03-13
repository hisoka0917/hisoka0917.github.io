---
layout: post
title: Swift编码规范
date: 2018-03-13
categories: swift
tags: [swift, ios, code]
---

根据SwiftLint的规则自己撸了一份Swift编码规范。整个顺序都是按照SwiftLint的顺序，基本上属于搬运。


#### 1. 委托(Delegate)协议应当为class类，可以被弱引用

#### 2. 闭合小括号内有闭合大括号中间不能有任何空格

```swift
// Correct
[].map({ })
[].map(
 { }
 )
 
 // Wrong
 [].map({ } )
 [].map({ }.   )
```

#### 3. 闭包的封闭端和开始端应当有相同的缩进

#### 4. 闭包的参数应当和左括号在同一行

```swift
// Correct
[1, 2].map { number in
 number + 1 
}

// Wrong
[1, 2].map {
 ↓number in
 number + 1 
}
```

#### 5. 闭包表达式在每个大括号内应该有一个空格

```swift
// Correct
map({ $0 })
[].map ({ $0.description })

// Wrong
map({$0 })
map({ $0})
map({$0})
[].map ({$0.description     })
```

#### 6. 冒号`:`应当紧靠所定义的标识符，即左侧没有空格，右侧与类型之间必须只有一个空格。如果在`Dictionary`中，紧靠Key，与Value之间必须有且只有一个空格

```swift
// Correct
let abc: String = "jun"
let abc = [1: [3: 2], 3: 4]

// Wrong
let jun:Void
let jun : Void
let jun :Void
let jun:   Void
```

#### 7. 逗号`,`前面没有空格，后面有且只有一个空格

```swift
// Correct
[a, b, c, d]
func abc(a: String, b: String) { }
func abc(
  a: String,
  bcd: String
) {
}

// Wrong
[a ,b]
func abc(a: String↓ ,b: String) { }
let result = plus(
    first: 3↓ , // Comment
    second: 4
)
```

#### 8. 编译器协议中声明的初始化函数不应该直接调用

```swift
// Correct
let set: Set<Int> = [1, 2]
let set = Set(array)

// Wrong
let set = Set(arrayLiteral: 1, 2)
let set = Set.init(arrayLiteral: 1, 2)
```

#### 9. 控制语句 for，while，do，catch语句中的条件不能包含在()中

```swift
// Correct
if condition {
if (a, b) == (0, 1) {

// Wrong
if (condition) {
if(condition) {
if ((a || b) && (c || d)) {
```

#### 10. 函数体复杂度应当被限制

```swift
// 建议
func f1() {
	if true {
		for _ in 1..5 { } 
	}
	if false { }
}

func f(code: Int) -> Int {
	switch code {
	case 0: fallthrough
	case 0: return 1
	case 0: return 1
	case 0: return 1
	case 0: return 1
	case 0: return 1
	case 0: return 1
	case 0: return 1
	case 0: return 1
	default: return 1
	}
}

func f1() {if true {}; if true {}; if true {}; if true {}; if true {}; if true {}
func f2() {
if true {}; if true {}; if true {}; if true {}; if true {}
}}

// 不建议
func f1() {
  if true {
    if true {
      if false {}
    }
  }
  if false {}
  let i = 0

  switch i {
  case 1: break
  case 2: break
  case 3: break
  case 4: break
 default: break
  }
  for _ in 1...5 {
    guard true else {
      return
    }
  }
}
```

#### 11. 使用block注册通知时应当储存返回的观察者，以便之后移除

```swift
// Correct
let foo = nc.addObserver(forName: .NSSystemTimeZoneDidChange, object: nil, queue: nil) { }

let foo = nc.addObserver(forName: .NSSystemTimeZoneDidChange, object: nil, queue: nil, using: { })

func foo() -> Any {
   return nc.addObserver(forName: .NSSystemTimeZoneDidChange, object: nil, queue: nil, using: { })
}

// Wrong
nc.addObserver(forName: .NSSystemTimeZoneDidChange, object: nil, queue: nil) { }

nc.addObserver(forName: .NSSystemTimeZoneDidChange, object: nil, queue: nil, using: { })

@discardableResult func foo() -> Any {
   return nc.addObserver(forName: .NSSystemTimeZoneDidChange, object: nil, queue: nil, using: { })
}
```

#### 12. 不推荐直接初始化可能有害的类型

```swift
// 建议写法
let foo = UIDevice.current
let foo = Bundle.main
let foo = Bundle(path: "bar")
let foo = Bundle(identifier: "bar")

// 不建议写法
let foo = UIDevice()
let foo = Bundle()
let foo = bar(bundle: Bundle(), device: UIDevice())
```

#### 13. 避免一起使用`dynamic`和`@inline(__always)`

```swift
// Correct
class C {
	dynamic func f() {}
}

class C {
	@inline(__always) func f() {}
}

class C {
	@inline(never) dynamic func f() {}
}

// Wrong
class C {
	@inline(__always) dynamic func f() {}
}

class C {
	@inline(__always) public dynamic func f() {}
}

class C {
	@inline(__always) dynamic internal func f() {}
}

class C {
	@inline(__always)
	dynamic func f() {}
}

class C {
	@inline(__always)
	dynamic
	func f() {}
}
```

#### 14. 建议使用`isEmpty`，而不是使用`count == 0`比较

```swift
// 建议写法
if number.isEmpty {
    print("为空")
} else {
    print("不为空")
}

// 不建议写法
let number = "long"
if number.characters.count == 0 {
    print("为空")
} else {
    print("不为空")
}
```

#### 15. 将枚举与关联类型进行匹配时，如果不使用它们，则可省略参数

```swift
// 建议写法
switch foo {
    case .bar: break
}
switch foo {
    case .bar(let x): break
}
switch foo {
    case let .bar(x): break
}
switch (foo, bar) {
    case (_, _): break
}
switch foo {
    case "bar".uppercased(): break
}

// 不建议写法
switch foo {
    case .bar(_): break
}
switch foo {
    case .bar(): break
}
switch foo {
    case .bar(_), .bar2(_): break
}
```

#### 16. 闭包参数为空时建议使用`() -> Void`而不是`Void -> Void`

```swift
// Correct
let abc: () -> Void
func foo(completion: () -> Void) {
}

// Wrong
let bcd: Void -> Void
func foo(completion: Void -> Void) {
}
```

#### 17. 使用尾随闭包时，避免使用空的圆括号

```swift
// Correct
[1, 2].map { $0 + 1 }
[1, 2].map({ $0 + 1 })
[1, 2].reduce(0) { $0 + $1 }
[1, 2].map { number in
	number + 1 
}
let isEmpty = [1, 2].isEmpty()
UIView.animateWithDuration(0.3, animations: {
   self.disableInteractionRightView.alpha = 0
}, completion: { _ in
   ()
})

// Wrong
[1, 2].map() { $0 + 1 }
[1, 2].map( ) { $0 + 1 }
[1, 2].map() { number in
	number + 1 
}
[1, 2].map(  ) { number in
	number + 1 
}
```

#### 18. 在声明类属性和方法时注意访问控制(ACL)，`private`,`fileprivate`,`internal`,`public`,`open` 必要时明确指定修饰关键字

#### 19. 尽量避免显式调用`.init()`

#### 20. 避免使用`fallthrough`

```swift
// Correct
switch foo {
case .bar, .bar2, .bar3:
    something()
}

// Wrong
switch foo {
case .bar:
    fallthrough
case .bar2:
    something()
}
```

#### 21. 每个文件大于700行时应考虑重构。文件行数禁止大于1200行。

#### 22. `for`循环中如果只有单个`if`条件建议使用`where`表达式

```swift
// 建议写法
for user in users where user.id == 1 { }
for user in users {
   if let id = user.id { }
}
for user in users {
   if var id = user.id { }
}
for user in users {
   if user.id == 1 { } else { }
}
for user in users {
   if user.id == 1 { }
   print(user)
}
for user in users {
   let id = user.id
   if id == 1 { }
}
for user in users {
   if user.id == 1 && user.age > 18 { }
}

// 不建议写法
for user in users {
   if user.id == 1 { return true }
}
```

#### 23. 避免强制类型转换(force cast) `as!`

```swift
// Correct
NSNumber() as? Int

// Wrong
NSNumber() as! Int
```

#### 24. 对会抛出异常(throw)的方法，避免使用`try!`强解

```swift
// Correct
func a() throws {}; do { try a() } catch {}

// Wrong
func a() throws {}; try! a()
```

#### 25. 尽量避免使用强制解包。

```swift
// 建议写法
navigationController?.pushViewController(myViewController, animated: true)
		
// 不建议写法
navigationController!.pushViewController(myViewController, animated: true)

let url = NSURL(string: "http://www.baidu.com")!
print(url)

return cell!
```

#### 26. 函数体长度限制。大于60行应当重构。单个函数体长度禁止大于100行。

#### 27. 函数参数数量应该尽量少。一般不超过5个。禁止大于8个。

#### 28. 范型类型名称只能包含字母数字，以大写字母开头，长度介于1～20个字符之间

```swift
// Correct
func foo<T>() {}
func foo<T>() -> T {}
func foo<T, U>(param: U) -> T {}
func foo<T: Hashable, U: Rule>(param: U) -> T {}

// Wrong
func foo<T_Foo>() {}
func foo<T, U_Foo>(param: U_Foo) -> T {}
func foo<TTTTTTTTTTTTTTTTTTTTT>() {}
func foo<type>() {}
```

#### 29. 变量标识符名称应该只包含字母数字字符，并以小写字母开头或只应包含大写字母。在上述例外情况下，当变量名称被声明为静态且不可变时，变量名称可能以大写字母开头。变量名称不应该太长或太短

#### 30. 只读属性不使用get关键字

```swift
// Correct
class Foo {
  var foo: Int {
    get {
      return 3
    }
    set {
      _abc = newValue 
    }
  }
}

class Foo {
  var foo: Int {
     return 20 
  } 
}

class Foo {
  static var foo: Int {
     return 20 
  } 
}


// Wrong
class Foo {
  var foo: Int {
    get {
      return 20 
    } 
  } 
}

class Foo {
  var foo: Int {
    get{
      return 20 
    } 
  } 
}
```

#### 31. 尽可能避免使用隐式解析可选类型

```swift
// Correct
@IBOutlet var label: UILabel!
let int: Int? = 42
let int: Int? = nil

// Wrong
let label: UILabel!
let int: Int! = 42
let int: Int! = nil
var ints: [Int!] = [42, nil, 42]
func foo(int: Int!) {}
let int: ImplicitlyUnwrappedOptional<Int>
```

#### 32. 初始化集合Set时,推荐使用Set.isDisjoint(), 不建议:Set.intersection

```swift
// 建议写法
_ = Set(syntaxKinds).isDisjoint(with: commentAndStringKindsSet)
let isObjc = !objcAttributes.isDisjoint(with: dictionary.enclosedSwiftAttributes)
_ = Set(syntaxKinds).intersection(commentAndStringKindsSet)
_ = !objcAttributes.intersection(dictionary.enclosedSwiftAttributes)

// 不建议写法
_ = Set(syntaxKinds).intersection(commentAndStringKindsSet).isEmpty
let isObjc = !objcAttributes.intersection(dictionary.enclosedSwiftAttributes).isEmpty
```

#### 33. 元组(tuple)成员个数不要太多，一般不超过3个

#### 34. 文件开头不应有空格或空行

```swift
/// Correct
//
// ViewController.swift

/// Wrong
 //......................这里有个空格
// ViewController.swift

/// Wrong
.........................这里是个空行
//
// ViewController.swift
```

#### 35. 使用结构体扩展属性和方法而不是传统函数

```swift
// Correct
rect.width
rect.height
rect.isNull
rect.isEmpty
rect.insetBy(dx: 5.0, dy: -7.0)

// Wrong
CGRectGetWidth(rect)
CGRectGetHeight(rect)
CGRectIsNull(rect)
CGRectIsEmpty(rect)
CGRectInset(rect, 10, 5)
```

#### 36. 使用结构体范围的常量而不是传统的全局常量

```swift
// Correct
CGRect.infinite
CGPoint.zero
CGRect.zero
CGRect.null
CGFloat.pi

// Wrong
CGRectInfinite
CGPointZero
CGRectZero
CGRectNull
CGFloat(M_PI)
```

#### 37. 使用Swift构造函数而不是传统便利函数

```swift
// Correct
CGPoint(x: 10, y: 10)
CGPoint(x: xValue, y: yValue)
CGSize(width: 10, height: 10)

// Wrong
CGPointMake(10, 10)
CGPointMake(xVal, yVal)
CGSizeMake(10, 10)
```

#### 38. 使用结构体扩展属性和方法而不是传统NS函数

```swift
// Correct
rect.width
rect.height
rect.isNull
rect.isEmpty
rect.insetBy(dx: 5.0, dy: -7.0)

// Wrong
NSWidth(rect)
NSHeight(rect)
NSIsEmptyRect(rect)
NSIntersectsRect(rect1, rect2)
```

#### 39. 单行代码长度不能超过120个字符

#### 40. 数组(Array)和字典(Dictionary)字面量结束处应该与它开始处有相同的缩进

```swift
// Correct
[1, 2, 3]

[1,
 2
]

[
   1,
   2
]

[
   1,
   2]
   
   let x = [
       1,
       2
   ]
   
[key: 2, key2: 3]

[key: 1,
 key2: 2
]

[
   key: 0,
   key2: 20
]

// Wrong
let x = [
   1,
   2
    ]
   let x = [
       1,
       2
 ]
let x = [
   key: value
    ]
```

#### 41. MARK注释应当采用有效格式`// MARK:...`或`// MARK: -...`

```swift
/// Correct
// MARK: good
// MARK: - good
// MARK: -
// BOOKMARK

/// Wrong
//MARK: bad
// MARK:bad
//MARK:bad
//  MARK: bad
// MARK:  bad
// MARK: -bad
// MARK:- bad
// MARK:-bad
//MARK: - bad
```

#### 42. 当参数闭包多于一个的时候不使用尾随闭包语法

```swift
// Correct
foo.map { $0 + 1 }

foo.reduce(0) { $0 + $1 }

if let foo = bar.map({ $0 + 1 }) {

}

foo.something(param1: { $0 }, param2: { $0 + 1 })

UIView.animate(withDuration: 1.0) {
    someView.alpha = 0.0
}

// Wrong
foo.something(param1: { $0 }) { $0 + 1 }
UIView.animate(withDuration: 1.0, animations: {
    someView.alpha = 0.0
}) { _ in
    someView.removeFromSuperview()
}
```

#### 43. 一个对象作为观察者应该只在`deinit`中移除自己

```swift
// Correct
class Foo { 
   deinit {
       NotificationCenter.default.removeObserver(self)
   }
}
class Foo { 
   func bar() {
       NotificationCenter.default.removeObserver(otherObject)
   }
}

// Wrong
class Foo { 
   func bar() {
       NotificationCenter.default.removeObserver(self)
   }
}
```

#### 44. 左大括号前边应有一个单独空格，并与声明语句位于同一行

```swift
// Correct 
func abc() {
}
[].map() { $0 }
[].map({ })
if let a = b { }
while a == b { }
guard let a = b else { }

// Wrong
func abc(){
}
func abc()
	{ }
[].map(){ $0 }
[].map( { } )
if let a = b{ }
while a == b{ }
guard let a = b else{ }
```

#### 45. 运算符左右应当各有一个空格

```swift
// Correct
let foo = 1 + 2
let foo = 1 > 2
let foo = !false

// Wrong
let foo = 1+2
let foo = 1   + 2
let foo = 1   +    2
let foo = 1 +    2
let foo=1+2
let foo=1 + 2
let foo=bar
let foo = bar   ?? 0
let foo = bar??0
```

#### 46. 在协议中声明属性时，应该设置访问器的顺序为`get set`

```swift
// Correct
protocol Foo {
	var bar: String { get set }
}
protocol Foo {
	var bar: String { get }
}
protocol Foo {
	var bar: String { set }
}

// Wrong
protocol Foo {
	var bar: String { set get }
}
```

#### 47. 在弃置函数结果时，优先使用`_ = foo()`而不是`let _ = foo()`

```swift
// Correct
_ = foo()
if let _ = foo() { }
guard let _ = foo() else { return }

// Wrong
let _ = foo()
if _ = foo() { let _ = bar() }
```

#### 48. 用nil初始化可选变量是冗余的

```swift
// Correct
var myVar: Int?
let myVar: Int? = nil
var myVar: Int? = 0

// Wrong
var myVar: Int? = nil
```

#### 49. 当字符串枚举值与枚举名称相等时，可以省略。

```swift
// Correct
enum Numbers: String {
	case one
	case two
}
enum Numbers: Int {
 	case one = 1
 	case two = 2
}
enum Numbers: String {
 	case one = "ONE"
 	case two = "TWO"
}
enum Numbers: String {
 	case one = "ONE"
 	case two = "two"
}
enum Numbers: String {
 	case one, two
}

// Wrong
enum Numbers: String {
 	case one = "one"
 	case two = "two"
}
enum Numbers: String {
 	case one = "one", two = "two"
}
enum Numbers: String {
 	case one, two = "two"
}
```

#### 50. 函数声明返回Void是冗余的

```swift
// Correct
func foo() {}
func foo() -> Int {}

// Wrong
func foo() -> Void {}
protocol Foo {
 	func foo() -> Void
}
func foo() -> () {}
```

#### 51. 函数返回箭头和返回类型间应该用一个空格分隔，或者在独立一行

```swift
// Correct
func abc() -> Int {}
func abc() -> [Int] {}
func abc() ->
    Int {}
func abc()
    -> Int {}

// Wrong
func abc()->Int {}
func abc()->[Int] {}
func abc()->(Int, Int) {}
func abc()-> Int {}
func abc() ->Int {}
func abc()  ->  Int {}
```

#### 52. 在执行运算操作和赋值时，优先使用简写操作符（+ =， - =，* =，/ =）

```swift
// Correct
foo -= 1
foo -= variable
foo -= bar.method()

// Wrong
foo = foo - 1
foo = foo - aVariable
foo = foo - bar.method()
```

#### 53. `else`和`catch`应该在前一个声明的同一行上，并以一个空格分隔

```swift
// Correct
} else if {

} else {

} catch {

// Wrong
}else if {

}  else {

}
catch {

}
	  catch {
```

#### 54. `case`语句应该与封闭的`switch`语句垂直对齐

```swift
// Correct
switch someBool {
case true: // case 1
    print('red')
case false:
    /*
    case 2
    */
    if case let .someEnum(val) = someFunc() {
        print('blue')
    }
}
enum SomeEnum {
    case innocent
}

if aBool {
    switch someBool {
    case true:
        print('red')
    case false:
        print('blue')
    }
}

switch someInt {
// comments ignored
case 0:
    // zero case
    print('Zero')
case 1:
    print('One')
default:
    print('Some other number')
}

// Wrong
switch someBool {
    case true:
         print('red')
    case false:
         print('blue')
}

if aBool {
    switch someBool {
        case true:
            print('red')
    case false:
        print('blue')
    }
}

switch someInt {
    case 0:
    print('Zero')
case 1:
    print('One')
    default:
    print('Some other number')
}
```

#### 55. 应该使用速记语法糖，即[Int]而不是Array

```swift
// Correct
let x: [Int]
let x: [Int: String]

// Wrong
let x: Array<String>
let x: Dictionary<Int, String>
```

#### 56. 禁止在数组和字典中出现尾随逗号

```swift
// Correct
let foo = [1, 2, 3]
let foo = []
let foo = [:]
let foo = [1: 2, 2: 3]

// Wrong
let foo = [1, 2, 3,]
let foo = [1, 2, 3, ]
let foo = [1, 2, 3   ,]
let foo = [1: 2, 2: 3, ]
```

#### 57. 每个文件应该以一个并且只有一个空行结束

#### 58. 代码行不能有尾随分号

```swift
// Correct
let a = 0

// Wrong
let a = 0;

let a = 0;
let b = 1

let a = 0;;

let a = 0;    ;;

let a = 0; ; ;
```

#### 59. 行末不能有空格

```swift
// Correct
let name: String
//
let name: String //

// Wrong
let name: String ←这里有空格
```

#### 60. 单个类型定义长度不能超过500行，如单个类代码行数不能超过500行

#### 70. 类型名称只能包含字母数字字符，以大写字母开头，长度介于3到40个字符之间

```swift
// Correct
class MyType {}
class AAAAAAAAAAAAAAAAAAAAAAAAAAAAA {}
struct MyType {}

// Wrong
class myType {}
class _MyType {}
private class MyType_ {}
class My {}
class AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA {}
struct myType {}
```

#### 71. 避免使用不需要的`break`语句

```swift
// Correct
switch foo {
case .bar:
    break
}

switch foo {
default:
    break
}

switch foo {
case .bar:
    for i in [0, 1, 2] { break }
}

switch foo {
case .bar:
    if true { break }
}

switch foo {
case .bar:
    something()
}

// Wrong
switch foo {
case .bar:
    something()
    ↓break
}

switch foo {
case .bar:
    something()
    ↓break // comment
}

switch foo {
default:
    something()
    ↓break
}

switch foo {
case .foo, .foo2 where condition:
    something()
    ↓break
}
```

#### 72. 闭包中未使用的参数应该用`_`替换

```swift
// Correct
[1, 2].map { $0 + 1 }
[1, 2].map { number in
 number + 1 
}
[1, 2].map { _ in
 3 
}

// Wrong
[1, 2].map { number in
 return 3
}
[1, 2].map { number in
 return numberWithSuffix
}
```

#### 73. 当index或item未使用时，`.enumerated()`可以删除

```swift
// 推荐写法
for (idx, foo) in bar.enumerated() { }
for (_, foo) in bar.enumerated().something() { }
for (_, foo) in bar.something() { }
for foo in bar.enumerated() { }
for foo in bar { }
for (idx, _) in bar.enumerated().something() { }
for (idx, _) in bar.something() { }
for idx in bar.indices { }

// 不推荐写法
for (_, foo) in bar.enumerated() { }
for (_, foo) in abc.bar.enumerated() { }
for (_, foo) in abc.something().enumerated() { }
for (idx, _) in bar.enumerated() { }
```

#### 74. 可选绑定值不使用时，用`!= nil`优于`let _ =`

```swift
// 推荐写法
if let bar = Foo.optionalValue {
}
if let (_, second) = getOptionalTuple() {
}
if let (_, asd, _) = getOptionalTuple(), let bar = Foo.optionalValue {
}
if foo() { let _ = bar() }
if foo() { _ = bar() }
if case .some(_) = self {}
if let point = state.find({ _ in true }) {}

// 不推荐写法
if let ↓_ = Foo.optionalValue {
}
if let a = Foo.optionalValue, let ↓_ = Foo.optionalValue2 {
}
guard let a = Foo.optionalValue, let ↓_ = Foo.optionalValue2 {
}
if let (first, second) = getOptionalTuple(), let ↓_ = Foo.optionalValue {
}
if let (first, _) = getOptionalTuple(), let ↓_ = Foo.optionalValue {
}
if let (_, second) = getOptionalTuple(), let ↓_ = Foo.optionalValue {
}
if let ↓(_, _, _) = getOptionalTuple(), let bar = Foo.optionalValue {
}
func foo() {
if let ↓_ = bar {
}
if case .some(let ↓_) = self {}
```

#### 75. @IBInspectable只应用于变量，其类型是显式的，并且是受支持的类型

```swift
// Correct
class Foo {
  @IBInspectable private var x: Int
}

// Wrong
class Foo {
  @IBInspectable private let count: Int
}
```

#### 76. 如果函数参数在声明中位于多行中，则应垂直对齐

```swift
// Correct
func validateFunction(_ file: File, kind: SwiftDeclarationKind,
                      dictionary: [String: SourceKitRepresentable]) { }

func foo(data: (size: CGSize,
                identifier: String)) {}

func validateFunction(
   _ file: File, kind: SwiftDeclarationKind,
   dictionary: [String: SourceKitRepresentable]) -> [StyleViolation]

// Wrong
func validateFunction(_ file: File, kind: SwiftDeclarationKind,
                  ↓dictionary: [String: SourceKitRepresentable]) { }
func validateFunction(_ file: File, kind: SwiftDeclarationKind,
                       ↓dictionary: [String: SourceKitRepresentable]) { }
func validateFunction(_ file: File,
                  ↓kind: SwiftDeclarationKind,
                  ↓dictionary: [String: SourceKitRepresentable]) { }
```

#### 77. 调用函数时如果参数位于多行，则应垂直对齐

#### 78. 垂直空白分隔仅限单个空行

```swift
// Correct
let abc = 0
let cba = 0

/* bcs 



*/

// bca 

// Wrong
let aaaa = 0


↑ 2个空行
struct AAAA {}



↑ 3个空行
```

#### 79. 使用`-> Void`而不是`-> ()`

```swift
// Correct
let abc: () -> Void = {}
let abc: () -> (VoidVoid) = {}
func foo(completion: () -> Void)
let foo: (ConfigurationTests) -> () throws -> Void)

// Wrong
let abc: () -> ↓() = {}
let abc: () -> ↓(Void) = {}
let abc: () -> ↓(   Void ) = {}
func foo(completion: () -> ↓())
func foo(completion: () -> ↓(   ))
func foo(completion: () -> ↓(Void))
let foo: (ConfigurationTests) -> () throws -> ↓())
```

#### 80. Delegate应该为weak来避免循环引用

```swift
// Correct
class Foo {
  weak var delegate: SomeProtocol?
}
class Foo {
  weak var someDelegate: SomeDelegateProtocol?
}
class Foo {
  weak var delegateScroll: ScrollDelegate?
}

// Wrong
class Foo {
  var delegate: SomeProtocol?
}
class Foo {
  var scrollDelegate: ScrollDelegate?
}
```

#### 81. 使用可选绑定而不是强制解包

```swift
// Correct
if let constantName = someOptional { }

// Wrong
let constants = someOptional!
```

#### 82. 使用可选绑定语法`if let value =`来判空而不是`if value != nil`

```swift
// Correct
if let value = result {
	print(value)
}

// Wrong
if value != nil {
	print(value)
}
```



