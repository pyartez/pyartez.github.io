---
layout: post
date: 2020-06-04 10:00
title:  "Simple JSON decoder in Swift and Combine"
mood: happy
category: 
- swift
- combine
- generics
- networking
tags:
- swift
- combine
- generics
- networking
---

## Intro

Pretty much every app nowadays requires you to connect to the internet to access some content. The majority of those apps use [JSON](https://www.w3schools.com/whatis/whatis_json.asp) to communicate that data from the backend systems.

There is high chance you will have some code in your app to download, parse and return objects for your app to use from an endpoint (unless you are using a network library such as [Alamofire](https://github.com/Alamofire/Alamofire))

In this post we are going to demonstrate how we can use [Generics](https://docs.swift.org/swift-book/LanguageGuide/Generics.html) and [Codable](https://developer.apple.com/documentation/foundation/archives_and_serialization/encoding_and_decoding_custom_types) to help us build a simple reusable JSON decoder to download and parse responses from our endpoints.

## Building our codable objects

The first thing we need to do is build our codable objects. Objects that implement the Codable protocol allow [Encoders](https://developer.apple.com/documentation/swift/encoder) and [Decoders](https://developer.apple.com/documentation/swift/decoder) to encode or decode them to and from an external representation such as JSON.

Let's take the response from the sample endpoint below as an example:

<https://jsonplaceholder.typicode.com/users>

{% highlight JSON %}

{
    "id": 1,
    "name": "Leanne Graham",
    "username": "Bret",
    "email": "Sincere@april.biz",
    "address": {
      "street": "Kulas Light",
      "suite": "Apt. 556",
      "city": "Gwenborough",
      "zipcode": "92998-3874",
      "geo": {
        "lat": "-37.3159",
        "lng": "81.1496"
      }
    },
    "phone": "1-770-736-8031 x56442",
    "website": "hildegard.org",
    "company": {
      "name": "Romaguera-Crona",
      "catchPhrase": "Multi-layered client-server neural-net",
      "bs": "harness real-time e-markets"
    }
  }

{% endhighlight %}

You can create codable classes yourself by hand. In simple examples this can be fairly straight forward, however if you have a response that has a more complex structure, doing so can be time consuming and error prone.

To create our codable objects we can use a generator, my weapon of choice is [QuickType](https://app.quicktype.io). We just paste in the JSON that is returned from the posts endpoint and it automatically generates the Codable structs for us. Easy!

If we paste in our post response, we should end up with some code looking like this:

{% highlight Swift %}

// MARK: - User
struct User: Codable {
    let id: Int
    let name, username, email: String
    let address: Address
    let phone, website: String
    let company: Company
}

// MARK: - Address
struct Address: Codable {
    let street, suite, city, zipcode: String
    let geo: Geo
}

// MARK: - Geo
struct Geo: Codable {
    let lat, lng: String
}

// MARK: - Company
struct Company: Codable {
    let name, catchPhrase, bs: String
}

typealias Users = [User]

{% endhighlight %}

How easy was that?! Obviously we will still need to check the structs, in the example above none of the fields are optional which means data must be passed in otherwise our decoding will fail. We don't need to worry about that here, but worth remembering when checking the generated code in your examples.

## URLSession extension and Generics

To solve our problem we are going to wrap the existing URLSession [dataTask](https://developer.apple.com/documentation/foundation/urlsession/1410330-datatask) method. I'm sure if you have done any kind of request work in pure swift you will have used this method in some form so we aren't going to go into the details of how it works.

{% highlight Swift %}

extension HTTPURLResponse {
    enum ResponseError: Error {
        case statusCode
    }
}

extension URLSession {

	// 1
    enum SessionError: Error {
        case noData
        case statusCode
    }

    /// Wraps the standard dataTask method with a JSON decode attempt using the passed generic type.
    /// Throws an error if decoding fails
    /// - Parameters:
    ///   - url: The URL to be retrieved.
    ///   - completionHandler: The completion handler to be called once decoding is complete / fails
    /// - Returns: The new session data task

    // 2 
    func dataTask<T: Decodable>(with url: URL,
                                completionHandler: @escaping (T?, URLResponse?, Error?) -> Void) -> URLSessionDataTask {

        // 3
        return self.dataTask(with: url) { (data, response, error) in
        	// 4
            guard error == nil else {
                completionHandler(nil, response, error)
                return
            }

            // 5
            if let response = response as? HTTPURLResponse,
                (200..<300).contains(response.statusCode) == false {
                completionHandler(nil, response, SessionError.statusCode)
            }

            // 6
            guard let data = data else {
                completionHandler(nil, response, SessionError.noData)
                return
            }

            // 7
            do {
                let decoded = try JSONDecoder().decode(T.self, from: data)
                completionHandler(decoded, response, nil)
            } catch(let error) {
                completionHandler(nil, response, error)
            }
        }
    }
}

{% endhighlight %}

So let's step through this code sample step by step:

1. First of we have defined a custom error for this extension, this is returned when no data has been returned from the request, covered in point 6. We also have an error case if we get an HTTPURLResponse with an incorrect status code, covered in point 5.
2. Here we are making use of [Generics](https://docs.swift.org/swift-book/LanguageGuide/Generics.html) to allow any type T being returned from this function as long as type T implements the [Decodable](https://developer.apple.com/documentation/swift/decodable) protocol (which we need it to inorder to use the [JSONDecoder](https://developer.apple.com/documentation/foundation/jsondecoder))
3. As discussed, here we are calling the existing [dataTask](https://developer.apple.com/documentation/foundation/urlsession/1410330-datatask) method to run our request.
4. First thing we do once the request has returned is check to see if there was a request error, if so we call the completion handler with the response and the error.
5. The second check we perform is to check the status code if we have received an HTTPURLResponse. Note we aren't stopping the code here if we don't get a HTTPURLResponse as you could use this function to load a local JSON file for example, not just a remote URL. Any status code in the 200-299 range is considered a successful request, if we receive a status code outside this range we return an error along with the response for further processing by whoever passed the completion handler.
6. The third check we perform is to unwrap data ready for decoding. If this fails (as in it's nil) then we call the completionHandler with the response and our custom error defined in step 1.
7. The final piece of the puzzle is to attempt to decode the data into type T we defined in the method signature as part of step 2. If this succeeds we can call our completion handler with our decoded type and response. If it throws an error we capture the error and return it using the catch block below.

## See it in action

Now that we have put our function together, let's take it for a test drive.

{% highlight Swift %}

let url = URL(string: "https://jsonplaceholder.typicode.com/users")!
let task = URLSession.shared.dataTask(with: url, completionHandler: { (users: Users?, response, error) in
    if let error = error {
        print(error.localizedDescription)
        return
    }

    users?.forEach({ print("\($0.name)\n") })
})
task.resume()

{% endhighlight %}

This shouldn't look too scarey, infact if you have used the standard dataTask functions in your code previously this should look very familiar. The only different here being that our completion handler now returns our Codable User objects rather than just a blob of Data like before.

Hopefully that example makes sense and gives you a nice simple way to perform a request and have it decode some JSON into a struct / class. Now let's have a look at some reactive programming using Combine.

## Combine

Hopefully you have at least heard of [Combine](https://developer.apple.com/documentation/combine) even if you haven't had chance to use it yet in a production app. It is Apple's own version of a reactive framework. Those of you who have already been using [RxSwift](https://github.com/ReactiveX/RxSwift) will be right at home. We aren't going to go into too much detail about what Combine is but here is a definition of what reactive programming is:

>In computing, reactive programming is a declarative programming paradigm concerned with data streams and the propagation of change

In more simplistic terms, reactive programming uses a observer pattern to allow classes to monitor different streams of data or state. When this state changes it emits an event with the new value which can trigger other streams to perform work or update such as UI code. If you are familier with [KVO](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/KeyValueObserving/KeyValueObserving.html) you will understand the basic concept. However reactive programming is far less painful and a lot more powerful than KVO.

Now let's take the previous pure Swift example and see how we can use it in Combine. The Combine framework adds new reactive functionality to the URLSession in the form of the [dataTaskPublisher](https://developer.apple.com/documentation/foundation/urlsession/3329708-datataskpublisher) function.

{% highlight Swift %}

extension URLSession {

	// 1
    enum SessionError: Error {
        case statusCode(HTTPURLResponse)
    }

    /// Function that wraps the existing dataTaskPublisher method and attempts to decode the result and publish it
    /// - Parameter url: The URL to be retrieved.
    /// - Returns: Publisher that sends a DecodedResult if the response can be decoded correctly.

    // 2
    func dataTaskPublisher<T: Decodable>(for url: URL) -> AnyPublisher<T, Error> {
    	// 3
        return self.dataTaskPublisher(for: url)
        	// 4
            .tryMap({ (data, response) -> Data in
                if let response = response as? HTTPURLResponse,
                    (200..<300).contains(response.statusCode) == false {
                    throw SessionError.statusCode(response)
                }

                return data
            })
            // 5
            .decode(type: T.self, decoder: JSONDecoder())
            // 6
            .eraseToAnyPublisher()
    }
}

{% endhighlight %}

Similar to our previous example we have extended URLSession to provide this functionality. Let's step through it:

1. As with the pure Swift example we are defining a custom error here to handle when we receive a status code that is not a success. The difference being here we are attaching the response to the error as we don't have a completionHandler in Combine. That way whoever is handling the error can inspect the response and see why it failed.
2. Here we are defining the method, again using generics to only accept a type T that has implemented the  Decodable protocol. The function returns a publisher who returns our decoded object.
3. As discussed previously, we are simply wrapping the existing dataTaskPublisher method.
4. Now here is where things start to become reactive. The [tryMap](https://developer.apple.com/documentation/combine/publishers/merge/3209016-trymap) function is similar to the standard map function in that it attempts to convert / transfrom elements from one type to another. However the difference here being that it is almost wrapped in a try. In this case you can include code in the closure which throws errors and they will be pushed downstream and handled later instead of needing a do block. Similar to our pure Swift example, we are checking we have a valid status code, if not we throw our custom error. If not we map our data ready to be decoded.
5. Here we are using the built in [decode](https://developer.apple.com/documentation/combine/publishers/merge/3208943-decode) method to attempt to decode our custom type using the JSONDecoder. Similar to the tryMap function above, any errors are pushed downstream to be handler later.
6. The final piece of the puzzle is to use [type erasure](https://pyartez.github.io/swift/apdvanced/protocols/generics/what-is-type-erasure.html). This removes the publisher class type and makes it AnyPublisher. For more info on type erasure see my [previous post](https://pyartez.github.io/swift/apdvanced/protocols/generics/what-is-type-erasure.html)

## Combine in action

Now that we have built our wrapper class 