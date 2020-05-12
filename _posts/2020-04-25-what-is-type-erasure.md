---
layout: post
date: 2020-04-20 21:00
title:  "What is Type Erasure in Swift"
mood: happy
category: 
- swift
- advanced
- protocols
tags:
- swift
- advanced
- protocols
---

## Intro

- Wtf is type erasure?
- Why do I need it?
- This seems complicated?
- Isn't there a simpler way?

These are are just some of the questions I found myself asking once I first starting exploring Type Erasure. Like many other developers, I have been making use of protocols in my code to remove dependencies and make my classes easy to unit test. It wasn't until I then started to add generics to my protocols that I discovered the need to apply Type Erasure.

Having read many blog posts and guides about Type Erasure I still came away confused as to what it was, why it was needed and why it seemed to add so much complexity. By trying to add generics to protocols in a project I was working on I finally saw the light! I am going to try and walk you through the topic using an example which is similar to the one I was trying to solve in my project. Hopefully this will help those of you who are looking to understand the topic further in the same way it helped me.

## Setting the scene

Let's take some example JSON so that you can follow along more easily.

[Users List]: https://jsonplaceholder.typicode.com/users
[Posts List]: https://jsonplaceholder.typicode.com/posts

If you open the posts list URL in the JSON viewer of your choice you will see that it contains a list of posts looking something like this:

{% highlight json %}

{
    "userId": 1,
    "id": 1,
    "title": "sunt aut facere repellat provident occaecati excepturi optio reprehenderit",
    "body": "quia et suscipit\nsuscipit recusandae consequuntur expedita et cum\nreprehenderit molestiae ut ut quas totam nostrum rerum est autem sunt rem eveniet architecto"
}

{% endhighlight %}

As you can see, each post contains a userId, however we don't have any user objects so we won't be able to display who made the post. In order to do so we will need the list of users.

### Codable Objects

To begin, we will create some codable structs to represent the data in the JSON urls so we can quickly parse them using a JSONDecoder

{% highlight swift %}

struct User {
    let id: Int
    let name: String
}

struct Post {
    let userId: Int
    let id: Int
    let title: String
}

{% endhighlight %}

To keep this example simple we are just going to parse a few of the attributes for each, we don't need all of them for what we are doing here.

### Services and View Model

To continue this example we are going to create 2 service objects, one for each of the lists we are going to fetch.

{% highlight swift %}

struct PostService {

​	

}

{% endhighlight %}

