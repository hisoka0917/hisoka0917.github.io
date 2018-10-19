---
layout: post
title: 利用Swift的反射将Struct转换成Dictionary
date: 2018-10-19
categories: swift
tags: [ios, swift, reflection, codable]
---

最近在做接口参数校验的时候碰到了一个问题。在参数格式转换的时候某些中文会变成编码的字符串。下面就来详细描述一下这个问题。

我们的接口参数校验需要将参数升序排列后拼接起来。在项目中接口的参数是以 Json 的格式传递的，所以这里我用了 `Swift 4` 中的 `JsonEncoder` 来做 Json 序列化，这样就可以直接将一个 `struct` 转成 `Json` 。那么这里我们需要将参数排序，所以我们需要将参数转成 `Dictionary` 类型，因为 `Dictionary` 有排序方法。于是最开始的时候我使用了系统提供的简单的转换函数，如下所示：

```swift
// 定义数据类型
struct ContactSimpleModel: Codable {
    var relation: String
    var name: String
}

// 扩展 Encodable 协议
extension Encodable {

    var dictionary: [String: Any]? {
        if let data = try? JSONEncoder().encode(self) {
            if let dict = try? JSONSerialization.jsonObject(with: data) as? [String: Any] {
                return dict
            }
            return nil
        }
        return nil
    }

}
```

来试着运行一下：

```swift
let contact1 = ContactSimpleModel(relation: "朋友", name: "宙斯")

if let dict = contact1.dictionary {
    print(dict)
}

// 打印: ["relation": 朋友, "name": 宙斯]
```

OK，到目前为止都没有问题。但是当遇到复杂一点的参数时，问题就出来了。

```swift
struct PersonModel: Codable {
    var job: String?
    var contacts: [ContactSimpleModel]
    var manager: ManagerSimpleModel?
}

struct ContactSimpleModel: Codable {
    var relation: String
    var name: String
}

struct ManagerSimpleModel: Codable {
    var name: String
    var age: Int
}
```

这里我们定义了一个嵌套的数据结构。然后我们试着像刚才那样来处理。

```swift
let contact1 = ContactSimpleModel(relation: "朋友", name: "宙斯")
let contact2 = ContactSimpleModel(relation: "同学", name: "奥丁")
let manager = ManagerSimpleModel(name: "拉斐尔", age: 31)
let job = "火枪手"

let person = PersonModel(job: job, contacts: [contact1, contact2], manager: manager)

if let dict = person.dictionary {
    print(dict)
}
```

打印出来的结果为：

```
["contacts": <__NSArrayI 0x600002471980>(
{
    name = "\U5b99\U65af";
    relation = "\U670b\U53cb";
},
{
    name = "\U5965\U4e01";
    relation = "\U540c\U5b66";
}
)
, "manager": {
    age = 31;
    name = "\U62c9\U6590\U5c14";
}, "job": 火枪手]
```

问题出现了，中文被转成了utf-8编码的字符串。这个问题是在 `JSONSerialization.jsonObject()` 这一步，这个方法会将嵌套结构中的中文字符转成编码的字符串，第一层 `key-value` 中的中文值不会受影响。那么这样一来我如果拿这个字典去做排序拼接就会和后端计算的值会不一样。于是就要想办法来手动解析，以保证字符的编码不变。

于是我准备利用 `Swift` 的反射机制来做这件事情。反射的机制在这篇文章里就不介绍了，感兴趣的同学可以搜索相关文章。这里我介绍一下具体怎么实现。

我们来看一段代码：

```swift
func allProperties(model: Codable) throws -> [String: Any] {

    var result: [String: Any] = [:]

    let mirror = Mirror(reflecting: model)

    guard let style = mirror.displayStyle, style == Mirror.DisplayStyle.struct || style == Mirror.DisplayStyle.class else {
        //throw some error
        throw NSError(domain: "hris.to", code: 777, userInfo: nil)
    }

    for (labelMaybe, valueMaybe) in mirror.children {
        guard let label = labelMaybe else {
            continue
        }

        result[label] = valueMaybe
    }

    return result
}
```

首先这段代码中新建了一个 `Mirror` 类。这个 `Mirror` 类可以将一个对象反射到一个结构中。它有一个 `displayStyle` 属性，其定义如下：

```swift
public enum DisplayStyle {

        case `struct`

        case `class`

        case `enum`

        case tuple

        case optional

        case collection

        case dictionary

        case set
}
```

基本上涵盖了所有对象类型。这个属性可以用来判断当前对象是什么类型。

然后还有一个 `children` 属性，这个属性是 `(label: String?, value: Any)` 类型的一个集合。

这个函数做的事情就是将一个对象的所有属性转成一个 `Dictionary` 对象返回。以 `label` 为键，`value` 作为值。那么我们来看一下之前定义的 `Person` 对象转成字典后的结果。

```swift
if let dict = try? allProperties(model: person) {
    print(dict)
}

// 打印结果：["job": Optional("火枪手"), "manager": Optional(__lldb_expr_1.ManagerSimpleModel(name: "拉斐尔", age: 31)), "contacts": [__lldb_expr_1.ContactSimpleModel(relation: "朋友", name: "宙斯"), __lldb_expr_1.ContactSimpleModel(relation: "同学", name: "奥丁")]] 
```

从这个打印的结果中我们可以看到，值的类型很清楚的打印了出来。到这里为止，这个字典还是不能用的，因为这个里面的值有太多东西了，有类型名称，还有 `Optional`。所以要彻底的把嵌套的子结构也解析出来。于是我们用递归的方法来修改一下这个方法。

```swift
func allProperties(model: Codable) -> Any {
	var result: [String: Any] = [:]
    let mirror = Mirror(reflecting: model)

    guard let style = mirror.displayStyle, style == Mirror.DisplayStyle.struct || style == Mirror.DisplayStyle.class else {
        //throw some error
        return mirror
    }
    
    for (labelMaybe, valueMaybe) in mirror.children {
        guard let label = labelMaybe else { 
            continue 
        }
        result[label] = allProperties(model: valueMaybe as! Codable)
    }
    return result
}
```

然后运行一下，打印结果如下：

```plaint
["contacts": Mirror for Array<ContactSimpleModel>, "manager": Mirror for Optional<ManagerSimpleModel>, "job": Mirror for Optional<String>]
```

嗯，类型很准确，但是值没了。为什么会这样呢？原因是这个函数中我们只处理了 `struct` 和 `class` 两种类型。当我们将 `contacts` 、`manager` 和 `job` 3个属性的值递归调用的时候都在 `guard` 语句这里直接返回了。

我们仔细地来看这3个属性的类型。第一个 `contacts` 它是一个数组，所以它的 `displayStyle` 应该是 `collection` 。第二个 `manager` 它应该是个 `struct` 啊，那它应该可以走到下面的代码并且解析出来。实际上不是，它的类型是 `ManagerSimpleModel?`，所以它的 `displayStyle` 实际上是 `optional`。同理，第三个 `job` 属性它的类型是 `Optional<String>`，所以它也是 `optional` 类型。所以说这3个属性都被提前返回了，打印出来的结果显示它们是一个 `Mirror` 对象。

于是接下来我们就要来解决其它类型时的处理。这里我们先来分析几种情况。

第一，`String`，`Int` 这类基础类型怎么办？从上面的 `DisplayStyle` 枚举中我们可以看到，基础类型没有在枚举里面。所以基础类型的 `displayStyle` 是 `nil`。

第二，数组我们怎么处理？`DisplayStyle` 枚举中有 `collection`，所以数组可以用这个类型来处理。要注意的是，数组是没有 `key` 的，所以 `collection` 类型的对象的每一个子元素是没有 `label` 的。我们只需将 `value` 取出来递归处理就行了。

第三，`optional`要怎么处理？这里我们就需要将值先解包后再处理。于是我们先实现解包的方法。

```swift
func unwrap<T>(_ any: T) -> Any
{
    let mirror = Mirror(reflecting: any)
    guard mirror.displayStyle == .optional, let first = mirror.children.first else {
        return any
    }
    return first.value
}
```

好，`optional` 的问题解决了，接下来我们要修改之前的函数来解决上述的那些问题。

```swift
func convert2Dict<T>(_ any: T) -> Any {
    let mirror = Mirror(reflecting: any)

    if let style = mirror.displayStyle {
        if style == .collection {
            var array: [Any] = []
            for (_, valueMaybe) in mirror.children {
                let value = unwrap(valueMaybe)
                array.append(convert2Dict(value))
            }
            return array
        } else {
            var dict: [String: Any] = [:]
            for (labelMaybe, valueMaybe) in mirror.children {
                guard let label = labelMaybe else { continue }
                let value = unwrap(valueMaybe)
                dict[label] = convert2Dict(value)
            }
            return dict
        }
    } else {
        return any
    }
}
```

修改完之后的方法处理了之前提到的几个问题。来看一下运行的结果。

```swift
if let dict = convert2Dict(person) as? [String: Any] {
    print(dict)
}

// 打印结果
// ["contacts": [["relation": "朋友", "name": "宙斯"], ["relation": "同学", "name": "奥丁"]], "job": "火枪手", "manager": ["name": "拉斐尔", "age": 31]]
```

打印的结果完全符合我们的预期。那么到此为止，我们利用反射来将 `struct` 转成 `Dictionary` 的功能已经完全实现了。[Gist地址可以点击这里查看](https://gist.github.com/hisoka0917/efcfbe006db929abecbe37376cf9e56b)。

再说点额外的。

接下来就只是我这个项目中的工程问题了。因为我要把字典按 `key` 排序。我们试着来给转换以后的字典排序。

```swift
if let dict = convert2Dict(person) as? [String: Any] {
    print(dict)
    let sortedQuery = dict.sorted(by: {$0.0 < $1.0})
    print(sortedQuery)
}

// 打印结果
// ["contacts[[\"relation\": \"朋友\", \"name\": \"宙斯\"], [\"relation\": \"同学\", \"name\": \"奥丁\"]]", "job火枪手", "manager[\"age\": 31, \"name\": \"拉斐尔\"]"]
```

这个结果并不是很好，因为数组里面的子结构体对象并没有按升序排列。所以还要继续处理里面嵌套结构的排序。当然这里处理方法并不唯一。由于我的项目需要，我的要求是把参数排序、拼接、删除多余字符集。于是在这个项目中我采用了比较 hack 的做法。就是排序、拼接、删除多余字符集的操作直接在转换的过程中完成。也就是说之前的转换函数不返回字典，而是直接返回一个已经处理完了的字符串。这么处理在我这个工程中是适用的。

如果同学们在工程中有类似需求，也可以寻找更好的实现方法。


