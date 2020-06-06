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

<https://jsonplaceholder.typicode.com/posts>

{% highlight JSON %}

{
    "userId": 1,
    "id": 1,
    "title": "sunt aut facere repellat provident occaecati excepturi optio reprehenderit",
    "body": "quia et suscipit\nsuscipit recusandae consequuntur"
}

{% endhighlight %}