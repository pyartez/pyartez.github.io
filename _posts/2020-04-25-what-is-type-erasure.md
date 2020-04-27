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

