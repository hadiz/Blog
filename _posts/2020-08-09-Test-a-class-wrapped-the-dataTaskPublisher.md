---
layout: post
title: Test a class wrapped the dataTaskPublisher
tags: [combine, unit test]
---
Apple introduced the [Combine framework](https://developer.apple.com/documentation/combine) in [WWDC19](https://developer.apple.com/videos/play/wwdc2019/722/).
>The Combine framework provides a declarative Swift API for processing values over time. These values can represent many kinds of asynchronous events. Combine declares publishers to expose values that can change over time, and subscribers to receive those values from the publishers.

If you are interested in knowing more about the Combine framework,  check [ this excellent, comprehensive mini-book by Matt Neuburg.](https://www.apeth.com/UnderstandingCombine/start.html)

Using the Combine framework, we can cover the whole process fetching data, decode it, handle the errors, and assign it to the views.  

```swift
URLSession.shared.dataTaskPublisher(for: url)
    .map { $0.data }
    .decode(type: Person.self, decoder: JSONDecoder())
    .receive(on: DispatchQueue.main)
    .sink(receiveCompletion: { _ in }) { [weak self] person in
        guard let self = self else { return }
        
        self.name.text = person.name
}.store(in:&self.storage)
```

But this code violates the single responsibility principle; It's better to separate the concerns. Let's define a SimpleNetwork class to take care of the network call.
```swift
class SimpleNetwork {
    func fetchData(from address: String) -> URLSession.DataTaskPublisher {
        let url = URL(string: address)
        
        return URLSession.shared.dataTaskPublisher(for: url!)
    }
}
```
We have some issues here. First, force unwrapping the URL is not a good idea. It's better to handle it by the optional binding. So when we face a bad URL, we can return an Error. To do that, We need to change our return type.  

```swift
func fetchData(from address: String) -> AnyPublisher<(data: Data,
    response: URLResponse), URLError> {
    
    guard let url = URL(string: address) else {
        return Fail<(data: Data, response: URLResponse),
            URLError>(error:
            URLError(URLError.badURL)).eraseToAnyPublisher()
    }
    
    return URLSession.shared.dataTaskPublisher(for:
        url).eraseToAnyPublisher()
}
```
### Writing Tests

Second, we need to provide some tests for our method. The first case would be to check the behavior of ```fetchData``` when we pass an invalid address.

```swift
func testInvalidAddressPublishesFailure() throws {
    let network = SimpleNetwork()
    let pub = network.fetchData(from: "Invalid URL")

    pub.sink(receiveCompletion: { completion in
        switch completion {
        case .finished:
            XCTFail()
        case .failure(let error):
           XCTAssertEqual(error.errorCode, URLError.badURL.rawValue)
        }
    }) {
        XCTAssertNil($0)
    }.store(in:&self.storage)
}
```

### The Problem

In the second test, we can check the behavior of ```fetchData``` when valid data returns. To achieve that, we need to mock the ```URLSession```. The first way would be inheriting our ```MockURLSession``` from ```URLSession``` and override the method that we need to mock. But the ```dataTaskPublisher``` is not open, so we can't override its behavior. The second thought would be declaring a Protocol, conform the ```URLSession``` to it, and use that protocol. Let's experiment that.

```swift
protocol URLSessionProtocol {
    func dataTaskPublisher(for url: URL) -> URLSession.DataTaskPublisher
}

class SimpleNetwork {
    var session: URLSessionProtocol!

    internal func fetchData(from address: String) -> AnyPublisher<(data: Data,
        response: URLResponse), URLError> {
    
    ...

        return session.dataTaskPublisher(for: url).eraseToAnyPublisher()
    }
}

extension URLSession: URLSessionProtocol {
}
```

Now we need to create a ```MockURLSession``` and conform it to ```URLSessionProtocol``` and change the ```dataTaskPublisher``` behavior. ```dataTaskPublisher``` returns a ```URLSession.DataTaskPublisher``` instance; ```DataTaskPublisher``` does the real job, and if we want to change the behavior, we need to override its behavior. But guess what, ```DataTaskPublisher``` is a Struct. We can not inherit from and override the behavior. So even it was possible to override ```dataTaskPublisher```, we couldn't change its behavior.

### The Solution

It seems that we have a dead-end here. But we can have a workaround. What if we define a new method in ```URLSessionProtocol```, conform the ```URLSession``` to it and then wrap the ```dataTaskPublisher``` method in the new method. Let's see this:

```swift
protocol URLSessionProtocol {
    func dataTaskAnyPublisher(for: URL) -> AnyPublisher<(data: Data, response:
        URLResponse), URLError>
}

class SimpleNetwork {
var session: URLSessionProtocol!

func fetchData(from address: String) -> AnyPublisher<(data: Data, response:
    URLResponse), URLError> {
    
    ...

    return session.dataTaskAnyPublisher(for: url)
    }
}

extension URLSession: URLSessionProtocol {
    func dataTaskAnyPublisher(for url: URL) -> AnyPublisher<(data: Data,
        response: URLResponse), URLError> {
        return self.dataTaskPublisher(for: url).eraseToAnyPublisher()
    }
}
```

Now we can make our ```MockURLSession``` conform to ```URLSessionProtocol``` and return any publisher that we need. So in our test side, we can write this:

```swift
class URLSessionMock : URLSessionProtocol {
    func dataTaskAnyPublisher(for: URL) -> AnyPublisher<(data: Data,
        response: URLResponse), URLError> {
        FakeURLSession.DataTaskPublisher().eraseToAnyPublisher()
    }
}

class FakeURLSession {
    struct DataTaskPublisher: Publisher {
        typealias Output = (data: Data, response: URLResponse)
        typealias Failure = URLError
        
        func receive<S>(subscriber: S)
            where S : Subscriber, Self.Failure == S.Failure, Self.Output == S.Input 
                {
                var subscription: Subscription
                
                subscription = SuccessInner(downstream: subscriber)
                
                subscriber.receive(subscription: subscription)
        }
        
        class SuccessInner<S>: Subscription
        where S : Subscriber, Failure == S.Failure, Output == S.Input {
            var downstream: S?
            
            init(downstream: S) {
                self.downstream = downstream
            }
            
            func request(_ demand: Subscribers.Demand) {
                _ = downstream?.receive((data: Data(), response: URLResponse()))
                downstream?.receive(completion: .finished)
                downstream = nil
                return
            }
            
            func cancel() {
                downstream = nil
            }
        }
    }
}
```

And finally, we can write our test:

```swift
func testValidURLPublishesResults() throws {
    let network = SimpleNetwork()
    network.session = URLSessionMock()
    
    let pub = network.fetchData(from: "https://apple.com")
    
    pub.sink(receiveCompletion: { completion in
        switch completion {
        case .finished:
            break
        case .failure( _ ):
           XCTFail()
        }
    }) {
        XCTAssertNotNil($0)
    }.store(in:&self.storage)
}
```

### Conclusion

  In our busy day full of distractions, we need to take some measures to delegate our works. One way is delegating repetitive tasks and automate them. With this automation, we also need to adopt some protection to avoid breaking our work by mistake. One of them is protecting our mainline to be healthy and ready to release. With enabling Status Checks for our mainline (the master branch in GitHub), we can be sure that we can not push our commits directly to the master branch,  and merging other branches to it is safe.
