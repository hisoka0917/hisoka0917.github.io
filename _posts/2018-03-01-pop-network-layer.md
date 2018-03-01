---
layout: post
title: 面向协议编程(Protocol Oriented Programming)的网络层代码简述
date: 2018-03-01
categories: swift
tags: [swift, ios, pop]
---

在WWDC2015上苹果介绍了面向协议编程（Protocol Oriented Programming）这一思想在Swift上的应用。这给了我们一种新的思路。接下来我们尝试用POP的思想来构建一个网络层。

首先我们定义一个Request协议

```swift
enum HTTPMethod: String {
    case GET
    case POST
}

protocol Request {
    var path: String { get }
    var method: HTTPMethod { get }
}
```

这是一个最原始的Request协议。显然这个协议还缺少很多东西。比如说还需要参数和返回数据的处理。那么在这里我们使用Swift4的Codable协议来定义Request所需的参数以及返回数据的解析。完善一下上面的协议。

```swift
protocol Parsable {
    associatedtype Result: Decodable
    static func parse(data: Data) throws -> Result?
}

protocol Request {
    var path: String { get }
    var method: HTTPMethod { get }
    var query: Query? { get set }

    associatedtype Response: Parsable
    associatedtype Query: Encodable
}
```

这里我们使用了`associatedtype`来让协议实现者自己来定义类型。

接下来我们需要一个发送Request的对象，于是定义一个Client。

```swift
enum ClientError: Error {
    case requestFailed
    case jsonConversionFailure
    case invalidData
    case responseUnsuccessful
    case jsonParsingFailure
}

protocol Client {
    var host: String { get }

    func fetch<R>(_ req: R, handler: @escaping (R.Response.Result?, ClientError?) -> Void) where R: Request
}
```

在Client协议里面定义了一个`host`变量和一个`fetch`方法，这个`fetch`方法接收一个`Request`类型的参数，完成之后异步回调`handler`。

然后来实现Client协议。

```swift
struct URLSessionClient: Client {
    let host = "http://gank.io/"

    func fetch<R>(_ req: R, handler: @escaping (R.Response.Result?, ClientError?) -> Void) where R: Request {
        let url = URL(string: self.host.appending(req.path))!
        var request = URLRequest(url: url)
        request.httpMethod = req.method.rawValue

        let task = URLSession.shared.dataTask(with: request) { (data, response, _) in
            guard let httpResponse = response as? HTTPURLResponse else {
                handler(nil, .requestFailed)
                return
            }
            if httpResponse.statusCode == 200 {
                DispatchQueue.main.async {
                    if let data = data {
                        do {
                            let res = try R.Response.parse(data: data)
                            handler(res, nil)
                        } catch {
                            handler(nil, .jsonConversionFailure)
                        }
                    } else {
                        handler(nil, .invalidData)
                    }
                }
            } else {
                handler(nil, .responseUnsuccessful)
            }
        }
        task.resume()
    }

}
```

这里使用原生的URLSession来实现一个Client。当然这里可以用任何的网络框架。那么这里的一个关键步骤就是使用`Request`协议里的`Response`属性的`parse()`方法来解析网络返回的数据。

实现了Client以后我们需要实现Request和Response。

```swift
struct ArticleModel: Codable {
    var _id: String
    var createdAt: String
    var desc: String
    var publishedAt: String
    var images: [String]?
    var source: String
    var type: String
    var url: String
    var used: Bool
    var who: String?
}

struct ArticleResponseModel: Codable {
    var error: Bool?
    var results: [ArticleModel]?
}

struct ArticleResponse: Parsable {
    typealias Result = ArticleResponseModel

    static func parse(data: Data) throws -> ArticleResponseModel? {
        do {
            let responseJson = try JSONDecoder().decode(Result.self, from: data)
            return responseJson
        } catch {
            throw ClientError.jsonParsingFailure
        }
    }
}

struct ArticleRequest: Request {
    var path: String = "api/data/iOS/20/2"
    var method: HTTPMethod = .GET
    var query: Query?
    typealias Response = ArticleResponse
    typealias Query = String
}
```

`ArticleResponse`遵守`Parsable`协议，定义了`Result`的类型为`ArticleResponseModel`。并且实现了`parse()`方法，这里使用`JSONDecoder`来进行数据解析。

在`ArticleRequest`里面定义Response的类型为`ArticleResponse`。

接下来我们只需要构造Request示例并传递给Client就行了。很多时候我们喜欢用一个`RequestManager`来干这个事情。

```swift
enum FetchResult<T, U> where U: Error {
    case success(T)
    case failure(U)
}

class RequestManager {
    static func fetch<R>(_ request: R,
                         handler: @escaping (FetchResult<R.Response.Result, ClientError>) -> Void) where R: Request {
                         
        URLSessionClient().fetch(request) { (result, error) in
            if let result = result {
                handler(FetchResult.success(result))
            } else {
                handler(FetchResult.failure(.jsonParsingFailure))
            }
        }
    }
}
```

到这里为止整个网络层的基本框架已经完成。在遵循了面向协议编程的思想之后现在的这个网络层弹性非常好。如果不喜欢用URLSession，可以整个替换掉，使用Alamofire或者别的你喜欢的网络框架，只需要遵守Client协议就行。Response的parse方法的实现也可以替换掉。Request的Query部分都是对象化操作，很方便。

查看完整的[示例代码](https://github.com/hisoka0917/POPNetworkExample)


