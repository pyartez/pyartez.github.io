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
    associatedtype FetchType

    func fetch(completion: ((Result<FetchType, Error>) -> Void)?)
}

{% endhighlight %}

Now that we have our protocols lets create a couple of structs to implement the protocol.

{% highlight swift %}

struct User {
    let id: Int
    let name: String
}

struct UserFetch: Fetchable {
    typealias FetchType = User

    func fetch(completion: ((Result<FetchType, Error>) -> Void)?) {
        let user = User(id: 1, name: "Phil")
        completion?(.success(user))
    }
}

{% endhighlight %}

So here we have created a dummy data class, User. Our fetch struct has implemented the Generic protocol and has specified the type of object that will be returned in the protocol using a typealias. Everything here is great, we can implement this protocol as many times as we like and return whatever object types we want without the need to create a new protocol for each one.

## The problem

Now, here in lies the problem. If we wish to hold a reference to an object that has implemented this protocol. How do we know what type it is going to return? See the below example:

{% highlight swift %}

struct SomeStruct {
    let userFetch: Fetchable
}

{% endhighlight %}

Now we could do something like below, however this creates a dependency between SomeStruct and the userFetch object. If we are following the SOLID principles we want to avoid having dependencies and hide implementation details.

{% highlight swift %}

struct SomeStruct {
    let userFetch: UserFetch
}

{% endhighlight %}

Ok, so let's try adding a type like we do with other Generic types such as arrays and dictionaries:

{% highlight swift %}

struct SomeStruct {
    let userFetch: Fetchable<User>
}

{% endhighlight %}

If you try the above you will probably end up with an error something like this:

> Cannot specialize non-generic type 'Fetchable'

See generic protocols, unlike generic types cannot have their type inferred in the type declaration. The type is only specified during implementation. So if we are following SOLID principles of programming we should have something like below, however this still does not tell us what type the fetchable will return:

{% highlight swift %}

struct SomeStruct {
    let userFetch: Fetchable
}

{% endhighlight %}

What you will also find here is that you will see an error, something like the below

> Protocol 'Fetchable' can only be used as a generic constraint because it has Self or associated type requirements

So we can't even use this type as a reference to the object, as it has an associated type which we cannot see at this point and have no way of specifying.

### Type Erasure to the rescue

So this is where type erasure comes in. In order for us to know the type returned we need to implement a new class that can be used to infer the type of object returned so that we know what to expect when we call fetch.

{% highlight swift %}

// 1
struct AnyFetchable<T>: Fetchable {
    // 2
    typealias FetchType = T

    // 3
    private let _fetch: (((Result<T, Error>) -> Void)?) -> Void

    // 4
    init<U: Fetchable>(_ fetchable: U) where U.FetchType == T {
        _fetch = fetchable.fetch
    }

    // 5
    func fetch(completion: ((Result<T, Error>) -> Void)?) {
        _fetch(completion)
    }
}

{% endhighlight %}

Whoooa! There is a lot going on here so let us go through it piece by piece to explain what is happening.

1. Here our AnyFetchable class is implementing the Fetchable protocol. But also we see that we now have a Generic type specification. This means that we can specify what type is being used while storing a reference to this struct.
2. Our generic type T being specified in the line above is then used in the typealias and mapped to the FetchType associated value of the protocol.
3. Now this is where things get a fiddly. In order for us to erase the type if the injected class we must first **create an attribute which is a closure with a matching signature for each function in the protocol**. In this scenario we only have 1 method which is the fetch method. Here you can see the fetch attribute has the same method signature as the one in the protocol.
4. Lets break this down a bit. First of all we saying that this init method is only available for an object that has implemented Fetchable, called U. The where clause at the end of the line is a generic type restriction which states that the FetchType of the Fetchable U must be the same as the one being used in this class. This might not make too much sense right now, but stay with me. When the fetchable type U is passed in, we store a reference to its fetch method in our own internal variable. This is what helps us erase the type, we store a reference to all of the objects methods without actually storing a reference to the object. That way we don't need to know the type.
5. Here is our implementation of the Fetchable protocols fetch method, however all we are doing is calling the reference to passed in objects fetch method and calling that instead.

Hopefully some of this makes sense, some of this may be new or confusing especially point 4. Let's show how we can use our class in this example.

{% highlight swift %}

struct SomeStruct {
    let userFetch: AnyFetchable<User>
}

// 1
let userFetch = UserFetch()

// 2
let anyFetchable = AnyFetchable<User>(userFetch)

// 3
let someStruct = SomeStruct(userFetch: anyFetchable)

// 4
someStruct.userFetch.fetch { (result) in
    switch result {
    case .success(let user):
        print(user.name)
    case .failure(let error):
        print(error)
    }
}

{% endhighlight %}

1. First we create an instance of our UserFetch object from earlier in the example that returns our example user.
2. We pass this into our AnyFetchable object. Now remember we had a generic type constraint on our init in point 4 of the previous example. This is being satisfied because we have specified that the AnyFetchable should return a User type, and the UserFetch object we are passing in has the FetchType User.
3. We can now pass in the AnyFetchable to our struct.

### Why?

Now you are probably thinking, why do all of this? Well, let's try another example:

{% highlight swift %}

// New Dave user struct
struct DaveFetch: Fetchable {
    typealias FetchType = User

    func fetch(completion: ((Result<FetchType, Error>) -> Void)?) {
        let user = User(id: 2, name: "Dave")
        completion?(.success(user))
    }
}

// Example implementation 2
let daveFetch = DaveFetch()
let anyDaveFetchable = AnyFetchable<User>(daveFetch)
let someDaveStruct = SomeStruct(userFetch: anyDaveFetchable)

someDaveStruct.userFetch.fetch { (result) in
    switch result {
    case .success(let user):
        print(user.name)
    case .failure(let error):
        print(error)
    }
}

{% endhighlight %}

So here we have created a new object that implements Fetchable and returns a user called Dave. We can then pass this into our SomeStruct using our type erasure class and it works exactly the same. The SomeStruct class doesn't need to be changed in order to work with the new dave class as it's type has been erased. In a production app we could inject any class we want as long as it fetches a User, whether that comes from the web, core data, the file system. It doesn't matter we could switch it at any time without making changes to our SomeStruct class.

### Finally

The last example here is that we can use our Any class for other types, not just User. See the example below:

{% highlight swift %}

// Product Type
struct Product {
    let id: Int
    let title: String
    let price: String
}

struct ProductFetch: Fetchable {
    typealias FetchType = Product

    func fetch(completion: ((Result<FetchType, Error>) -> Void)?) {
        let product = Product(id: 1, title: "My Product", price: "10.99")
        completion?(.success(product))
    }
}

let productFetch = ProductFetch()
let anyProductFetch = AnyFetchable<Product>(productFetch)
anyProductFetch.fetch { (result) in
    switch result {
    case .success(let product):
        print(product.title)
    case .failure(let error):
        print(error)
    }
}

{% endhighlight %}

Similar to our user example, we have created a new Product object and a fetcher that returns a product object. However we can re-use our AnyFetchable here but specifying the return type as Product.

There is a lot to cover and understand here and hopefully this helps make some sense of type erasure and what it is used for. More importantly how to implement your own Any type erasure class for your own protocols so that they can be referenced in your code.

[Download the playground](/files/type-erasure.playground.zip) and play around with the examples yourself