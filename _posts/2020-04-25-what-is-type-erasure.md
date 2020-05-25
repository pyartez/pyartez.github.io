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
    func fetch(completion: ((Result<Post, Error>) -> Void)?) {
        // Fetch code here
    }
}

struct UserService {
    func fetch(completion: ((Result<User, Error>) -> Void)?) {
        // Fetch code here
    }
}

{% endhighlight %}

To make the requests to these 2 services so that we can combine them and send them to the UI for display we are going to use something called a ViewModel. This pattern is commonly seen in things like MVVM architecture, we won't go into detail about those here but feel free to read up on them. It may help with your understanding of the described problem.

Our ViewModel may look something like this:

{% highlight swift %}

struct PostViewModel {
    let postService: PostService
    let userService: UserService

    func load() {
        postService.fetch({ result in 
            // Process data here
        })

        userService.fetch({ result in 
            // Process data here
        })
    }
}

{% endhighlight %}

Now we have a view model here, that is holding 2 references to the services it needs. Now what is the problem here?

* Testing - We are linking to the implementations of the post service and user service classes. We can't inject a mock inside a test to see how our view model behaves if either service throws an error for example.
* Dependencies - This class now has 2 other dependencies. If we wanted to re-use this or move it into another project / framework we would need to take the service objects. And the dependencies of those objects, and their dependencies and so on and so on.

### Refactoring to use protocols

The first thing we can do to resolve this is to use protocols to remove the dependencies on the service classes.

{% highlight swift %}

protocol PostService {
    func fetch(completion: ((Result<Post, Error>) -> Void)?)
}

protocol UserService {
    func fetch(completion: ((Result<User, Error>) -> Void)?)
}

/* Implementations */

struct remotePostService: PostService {
    func fetch(completion: ((Result<Post, Error>) -> Void)?) {
        // Fetch data
    }
}

struct remoteUserService: UserService {
    func fetch(completion: ((Result<User, Error>) -> Void)?) {
        // Fetch data
    }   
}

{% endhighlight %}

Here we have created a protocol for each of the services and then provided implementations in our remote service classes. Now let's update the View Model

{% highlight swift %}

struct PostViewModel {
    let postService: PostService
    let userService: UserService

    func load() {
        postService.fetch({ result in 
            // Process data here
        })

        userService.fetch({ result in 
            // Process data here
        })
    }
}

{% endhighlight %}

Wait a minute? Our ViewModel is exactly the same. Except now when we create it, we can inject mock services in our unit tests to make sure things are working as expected. I won't go into mocks and unit testing in this post but feel free to read further on the subject if you don't quite understand the above

Now looking at the 2 protocols we created above there is a common theme. Both of the methods in the service protocols are the same. The only difference is the return type in the result. What if we needed to add further services for other return types? Then we would have a bunch of protocols that are all the same except the type that is being returned. How can we fix this? Generics.

### Generics

Now that we have identified our 
