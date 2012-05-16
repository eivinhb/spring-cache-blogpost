Spring cache abstraction
=====================

Using caching to speed up web applications in a SOA environment.
--------------------

At work, we have a Service Oriented Architecture wrapped around a mainframe legazy system.
Basicly all our new web applications therefore communicates with web services. They are not flaming fast; typically they have a response time between 100ms and 1500ms.
When creating webapplications based on data from these web services, we want to cache often used data in our webapp to speed things up.

Not long ago we started using the fine Google Guava project http://code.google.com/p/guava-libraries/
This is a very nice cache system that can be plugged in to any class structure to cache method responses.
From the outside, you call an interface function. But in the implementation class you use a builder to create a static cache for the class method.
You delegate to Guava, and the cache will look up a key and return a value if it exists in the cache.
If not, guava will call a specified method that gets the data, stores the result in the cache and returns it through the interface.

A typical, simple implementation:
CODE HERE

Guava have all the functions you want in a cache: Time to live, max elements, eviction, invalidation, statistics and more.

This is great and all. But what if I dont want to put all this code in my implementation class?

Enter Spring Cache Abstraction
=============

Spring Cache Abstraction is new in Spring 3.1. It seems to be built on previous work done with spring ehcache annotations.
Spring cache abstraction is just that, an abstraction. It is based on some very powerful annotations like @Cacheable, @CachPut, @CacheEvict and more.
I will not show examples for all annontations in this blog, just the few that helped me replace Guava.

Basicly what you do is annotate a method, or class, with @Cacheable("name_of_cache"). Spring will then proxy the class and any calls done to the methods
will web cached in the named cache. Default key will be based on input to the method. Next time you call the method with the same parameters, the cached dat
will be returned. This is basicly all you need to do with the class to enable caching for this class or method.
(This means of course that the method should be consistent in input/output. But there are ways around everything. Costly calculations could be an example.)

CODE HERE

But to cache something, you need a cache!
We use annotated spring beans to wire up the application.

In our @Configuration class we added the @EnableCaching annotation. This is new in spring 3.1.1!
We then create a @Bean for CacheManager. This class is used by Spring Cache Abstraction to control the caching.
We can choose to use a simple ConcurrentMapCache, with a name equal to the one we use in @Cacheable, for very simple caching:
CODE HERE

And that is it! Your applications should now be able to cache the class methods.

But wait! What if I dont want the cache to live for ever?
An easy way could be to annotate a method with @CacheEvict("name_of_cache", allEvict=true) *** Check code ***
When this method is called, the cache will be emptied.

In our web application, we typically want to cache the xml response from a slow web service. This data is quite static, but it CAN change.
The question is when does it change? When should we evict data from the cache? And what do we do when the concurrent map begins to eat up our memory?
We need functionality that Guava gives us. At this point, there are no ready made implementations of Guava with Spring Cache Abstraction that I have found. (Lets start implementing)
Spring provides an implementation of EHcache, a well proven cache.

What we need to make use of this is to replace our concurrentMapCache with a EH cache manager:
CODE HERE!

To configure the eh cache we use this xml:
CODE HERE

There are of cource classes that can do this in our java code somehow, but some xml could be nice just to be able to easily configure our caches.


Now, lets say that we need some security on this application. Our web application is actually a rest api using Jersey resources and on top of that, web have a single signon solution to controll
user access to data. If we cache up web service responses, a user could easily fiddle with some params and get data that the user should not. This i a serious sequrity issue.@
We need to make the cache smarter. What if we could control the cache keys and add data to the key the user cant control from web.

We need a custom key generator:
CODE HERE

The single signon solutions helps us here, because our the backend have unique identifiers on each customer. We just add this identifier to the key, and we have a secure cache.
Notice that we have added an interface to the @Configuration class. This is to help Spring understand that the @Bean keyGenerator is the implementation that we want to use to generate defalut keys.



In this blog, web have shown how to add caching to a class in very uintrusive way. The example is quite simple,
and for this application it works very well. Spring cache abstraction can be used in much more complicated ways. Check out Spring documentation.
Remembering that Spring only abstract away the caching regime, and that spring does not handle the actual caching but leaves that to the proven implementation, I see no reason
why not to implement this.