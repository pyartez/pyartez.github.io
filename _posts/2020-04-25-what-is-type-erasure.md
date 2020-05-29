---
layout: post
date: 2020-04-20 21:00
title:  "What is Type Erasure in Swift"
mood: happy
category: 
- swift
- advanced
- protocols
- generics
tags:
- swift
- advanced
- protocols
- generics
---

## Intro

- Wtf is type erasure?
- Why do I need it?
- This seems complicated?
- Isn't there a simpler way?

These are are just some of the questions I found myself asking once I first starting exploring Type Erasure. Like many other developers, I have been making use of protocols in my code to remove dependencies and make my classes easy to unit test. It wasn't until I then started to add generics to my protocols that I discovered the need to apply Type Erasure.

Having read many blog posts and guides about Type Erasure I still came away confused as to what it was, why it was needed and why it seemed to add so much complexity. By trying to add generics to protocols in a project I was working on I finally saw the light! I am going to try and walk you through the topic using an example which is similar to the one I was trying to solve in my project. Hopefully this will help those of you who are looking to understand the topic further in the same way it helped me.

## Generics and Associated Types

I am assuming that as you are here you have a fairly advanced knowledge of swift and have potentially begun or have been using protocols with Generics in your code. Below is a simple protocol called Fetchable. Idea of the protocol is go and fetch some objects of type T from somewhere and call the completion handler with the result once it's finished whatever it is doing.

{% highlight swift %}

protocol Fetchable {
    associatedtype T

    func fetch(completion: ((Result<T, Error>) -> Void)?)
}

{% endhighlight %}

Now that we have our protocols lets create a couple of structs to implement the protocol.

{% highlight swift %}

struct User {
    let id: Int
    let name: String
}

struct userFetch: Fetchable {
    typealias T = User

    func fetch(completion: ((Result<T, Error>) -> Void)?) {
        let user = User(id: 1, name: "Phil")
        completion?(.success(user))
    }
}

/* ======== */

struct News {
    let id: Int
    let title: String
}

struct newsFetch: Fetchable {
    typealias T = News

    func fetch(completion: ((Result<T, Error>) -> Void)?) {
        let news = News(id: 1, title: "Some news title here")
        completion?(.success(news))
    }
}

{% endhighlight %}

So here we have created a couple of dummy data classes, User and News. Our fetch structs have implemented the Generic protocol and have specified the type of object that will be returned in the protocol using a typealias. Everything here is great, we can implement this protocol as many times as we like and return whatever object types we want without the need to create a new protocol for each one.

## The problem

Now, here in lies the problem. 