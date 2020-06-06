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

extension URLSession {

    enum SessionError: Error {
        case noData
    }

    /// Wraps the standard dataTask method with a JSON decode attempt using the passed generic type.
    /// Throws an error if decoding fails
    /// - Parameters:
    ///   - url: The URL to be retrieved.
    ///   - completionHandler: The completion handler to be called once decoding is complete / fails
    /// - Returns: The new session data task

    // 1 
    func dataTask<T: Decodable>(with url: URL,
                                completionHandler: @escaping (T?, URLResponse?, Error?) -> Void) -> URLSessionDataTask {

        //2
        return self.dataTask(with: url) { (data, response, error) in
        	// 3
            guard error == nil else {
                completionHandler(nil, response, error)
                return
            }

            // 4
            guard let data = data else {
                completionHandler(nil, response, SessionError.noData)
                return
            }

            // 5
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

