---
layout: post
title: Implementing a Cache layer using Observer Pattern
---
When talking about performance improvements of an application, first thing that comes to our mind is cache. Next thought that follows is, what is the easiest and quickest way to implement it. This is the stage that everybody get together and start pointing fingers at each other, blaming didn't have enough time to think about it due tight deadlines, too much changes to be done and don't want to touch the code etc.

My point on this is performance of the application is we should think about performance at the very start of the project. Even before we as developers open the code editor and write the very first line.

To explain this well, I'll take an example from the project I'm currently working on. 

This application is built using nested set model (tree architecture).[link wiki]. We use MongoDB to store data and application lifts the weight of object loading.

In simple, we have only one class which is responsible for all data objects in the system (there are few more classes that help handling data validation, related data loading etc.) but the core of the application is simply one class. Any data point in the system is an instance of a the main class.

The Problem

This architecture has helped us scale the system without having any core changes (core of the application hasn't been touched in years). Even though this architecture has lots of pros in our case, it has its drawbacks when it comes to performance (our benchmark was to load any given endpoint within less than a second).

Application is very data heavy and some endpoints took up to 4 seconds when doing calculations against big datasets. 

As we look into the issue, it turned out to be that some helper classes that were responsible for calculations were called few times. Especially when pulling data from 3 or 4 level down the chain. 

To fix this problem the first solution was to optimize methods that were loading data objects. We used lazy loading on all the nested data relationships to reduce the stress. Second solution was to cache data is been already called during the process.

Before we dive deeper in the Cache implementation let's have a look at the application structure.

Application Architecture.

Backend API - Laravel
Frontend - VueJS
Database - mongoDB
Cache - Redis

Application is deployed using docker and deployment is been done as separate container.

This gave is plenty of freedom to optimise at so many levels.

Cache Implementation
We put couple of rules before come up with a solution.

Change the core only if it's the last resort.
Utilities existing resources
Implementation shouldn't take long to implement (To avoid any major refactoring)

By looking at the above constraints it was clear that we had to use Observer pattern if we are to implement a solution without touching the core and build a solution as a wrapper to the application. 

Implementation

Knowing that we will be building the cache layer using Observer Pattern, we started utilising Laravel's IoC container. 

As we found that bottleneck was constant data load from the DB
First we started injecting concrete data related class.

Service Provider code with concrete class

Then we created interfaces with data fetching methods. The reason behind this approach was to have class that can load data from cache with same method names. That way there will be no changes to the core code.

Data load interfaces

Once we have all those heavy data loading methods been extracted to interfaces, now it's time to implement Cache supported class.

Protip: Be wise when creating cache tags and use short cache tag names. Redis will thank you a lot in the long run.

Cache supported class

Once we done creating cache supported class we can easily replace them with our concrete class with the help of the IoC container.


IoC code that replace the concrete class


There you have it. 

Now when the application calls any data loading methods it'll look in the cache first before it talks to the DB.

Here we have implemented a caching wrapper without disrupting the core of the application. 

Cache expiration

In out application we have one core class that it responsible of talking to DB. We implemented cache expiration using boot method in Elequent. But if you are not using Laravel you can use DB model events that comes with the library that you use.

Boot method goes here

Protip
When cache invalidation we need to make sure we use correct cache keys. Otherwise wise it'll drop the whole cache on every modification to the data.
