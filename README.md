Spring cache abstraction
=======

At work, we have a Service Oriented Architecture wrapped around a mainframe legazy system.
Basicly all our new web applications therefore communicates with web services and generate some type of webpage based on that data.
The web services are not flamingly fast; typically they have a response time between 100ms and 1500ms.
When creating webapplications based on data from these web services, we want to cache often used data in our webapp to speed things up.
Normally, the data is quite static and eventual consistency between web and mainframe data is good enough.

Google guava cache
-------

Not long ago we started using [Google Guava project](http://code.google.com/p/guava-libraries/) for our simple caching needs.
This is a very nice cache system that can be plugged in to any class structure to cache method responses.
From the outside, you call, for instance, an interface function. But in the implementation class you use a builder to create a static cache for the class method.
You delegate your actual call to Guava, and the cache will look up a key and return a value if it exists in the cache, otherwise it will compute the actual method.

A typical, simple implementation of a cache :

	@Component
	public class DataService(){
		private static Cache<CacheKey, Document> cache;
		private static final int CACHE_SIZE = 10;
		private static final int CACHE_EXPIRATION_IN_HOURS = 24;

		@Autowire
		private MyWebServiceBean myConnectorBean;

		public Document getDataFromService(Document request){
			if (cache == null) initializeCache();

			Document response = null;
			try {
				CacheKey key = new CacheKey(request.hashCode(), request);
				ettProduktResponse = cache.get(key);
			} catch (ExecutionException e) {
				throw new RuntimeException("Error fetching from cache", e);
			}
			return response;
		}

		private void initializeCache() {
			cache = CacheBuilder.newBuilder()
					.maximumSize(1000)
					.expireAfterWrite(CACHE_EXPIRATION_IN_HOURS, TimeUnit.HOURS)
					.build(new CacheLoader<CacheKey, Document>() {
						@Override
						public Document load(CacheKey cacheKey) throws Exception {
							return myConnectorBean.getDataFromService(request)
						}
					});
		}
	}

Guava have all the functions you want in a cache: Time to live, max elements, eviction, invalidation, statistics, eviction events and much much more.
However, implementations tend to be quite complex and new developers struggle to get the grasp of what is going on when they see the code. It can also be difficult to
write code that is unintrusive in the classes that need caching.
The example above could probably be written much smarter that this, but the complexity of the code is quite obvious.


Spring Cache Abstraction
=============

Spring Cache Abstraction is new in [Spring 3.1](http://static.springsource.org/spring/docs/3.1.0.M1/spring-framework-reference/html/cache.html).
It seems to be built on previous work done with [spring ehcache annotations](http://code.google.com/p/ehcache-spring-annotations/).

Spring cache abstraction is just that, an abstraction. It is based on some very powerful annotations like __@Cacheable__, __@CachePut__, __@CacheEvict__ and more.
I will not show examples for all annontations in this blog, just the few that helped me replace Guava.

Basicly what you do is annotate a method, or class for that sake, with **@Cacheable("name-of-cache")**. Spring will then proxy the class and any calls done to the methods
will be cached in the named cache. Default key will be based on input to the method. Next time you call the method with the same parameters, the cached data
will be returned. This is basicly all you need to do with the class to enable caching for this method, or class.
_(This means of course that the method should be consistent in input/output. Costly calculations being an example.)_

	@Component
	public class DataService(){

		@Autowire
		private MyWebServiceBean myConnectorBean;

		@Cacheable("name-of-cache")
		public Document getDataFromService(Document request){
			return myConnectorBean.getDataFromService(request);
		}
	}

But to cache something, you need a cache engine!
--------

We use annotated spring beans to wire up the application. __(NO XML FTW)__
In our __@Configuration__ class we added the __@EnableCaching__ annotation. __This is new in [Spring 3.1.1](name_of_cache)!__
We then create a __@Bean__ for __CacheManager__. This class is used by Spring Cache Abstraction to control the caching.
We can choose to use a simple __ConcurrentMapCache__, with a name equal to the one we use in __@Cacheable__, for very simple caching:

	@Configuration
	@EnableCaching
	public class ServiceContext{

		@Bean
		public CacheManager cacheManager(){
			CacheManager cacheManager = new SimpleCacheManager();
         	cacheManager.addCaches(Arrays.asList(new ConcurrentMapCache("name_of_cache")));
         	return cacheManager;
		}
	}

And that is it! Your applications should now be able to cache the class methods.

What if I don't want the cache to live for ever?
---------
An easy way could be to annotate a method with __@CacheEvict("name_of_cache", allEntries=true)__
When this method is called, the cache will be emptied.

However, in our web application, we typically want to cache the xml response from a slow web service. This data is quite static, but it CAN change.
The question is when does it change? When should we evict data from the cache? And what do we do when the concurrent map begins to eat up our memory?
We need functionality that Guava gives us. At this point, there are no ready made implementations of Guava with Spring Cache Abstraction that I have found. (Lets start implementing)
Spring provides an implementation of [EHcache](http://ehcache.org/), a well proven cache system.

What we need to make use of this is to replace our concurrentMapCache with a EHcache manager.
Notice that we have added an interface to the __@Configuration__ class. This is to help Spring understand that the __@Bean keyGenerator__
is the implementation that we want to use to generate default keys.

The KeyGenerator is an interface that take the actual object that is doing caching, the actual method that is being called and all parameters to that method as input.
Based on that input, it generates a key for the cache and returns it. The DefaultKeyGenerator that is used above creates a hashcode as key. This should work very good
on methods that is consistent in input and output.

	@Configuration
	@EnableCaching
	public class CoreServiceConfig implements CachingConfigurer {

		@Bean
		@Override
		public CacheManager cacheManager(){

			EhCacheManagerFactoryBean factory = new EhCacheManagerFactoryBean();
			factory.setCacheManagerName("My cache manager");
			factory.setConfigLocation(new ClassPathResource("ehcache.xml"));
			factory.setShared(true);
			try {
				factory.afterPropertiesSet();
			} catch (IOException e) {
				throw new RuntimeException("Init cache crashed", e);
			}

			EhCacheCacheManager managerEH = new EhCacheCacheManager();
			managerEH.setCacheManager(factory.getObject());

			return managerEH;
		}

		@Bean
		@Override
		public KeyGenerator keyGenerator(){
			return new DefaultKeyGenerator();
		}
	}

To configure the EHcache we use ehcache.xml on the classpath:

	<ehcache>
		<diskStore path="java.io.tmpdir"/>
		<defaultCache
			name="name_of_cache"
			maxEntriesLocalHeap="1000"
			eternal="false"
			timeToIdleSeconds="120"
			timeToLiveSeconds="120"
			overflowToDisk="true"
			maxEntriesLocalDisk="10000000"
			diskPersistent="false"
			diskExpiryThreadIntervalSeconds="120"
			memoryStoreEvictionPolicy="LRU"
			/>
	</ehcache>

There are of cource classes that somehow can do this in our java code, but some xml could be nice just to be able to easily configure our caches.
_Remember to add EHcache jar on you classpath.._

And that is it!

Security
------
If we cache up web service responses that are unique to a spesific user or somehing like that,
a user could easily fiddle with some params and get data that the user should not. This is a serious security issue!
We need to make the cache smarter. What if we could control the cache keys and add data to the key the user cannot control.
Our web application is actually a rest api using Jersey resources.
On top of everything, we have a single sign-on solution to control user access to data.

We implement a custom key generator:

	public class SecureKeyGenerator implements KeyGenerator {

		@Override
		public Object generate(Object target, Method method, Object... params) {
			Object keyPostFix = new DefaultKeyGenerator().generate(target, method, params);

			if (target instanceof SecureKeyAware && ((SecureAware) target).getSecureKey() != null) {
				return ((SecureKeyAware) target).getSecureKey() + keyPostFix;
			} else {
				throw new RuntimeExeption("Target for key generation is not secure key aware.");
			}
		}
	}

The single signon solutions helps us here, because our the backend have unique identifiers on each customer. We just add this identifier to the key, and we have a secure cache because
method parameters is no longer the only provider for key generation. Since the key is any subclass of object, we could create the
key as a MyKeyImplementation if you need to do something smart with the keys in the cache.

---

In this blog, web have shown how to add caching to a class or method in very uintrusive way with Spring. The example is quite simple,
and for this application it works very well. Spring cache abstraction can be used in much more complicated ways. Check out the [Spring documentation](http://static.springsource.org/spring/docs/3.1.0.M1/spring-framework-reference/html/cache.html).
Remember that Spring only abstract away the caching engine, and that spring does not handle the actual caching but leaves that to the proven implementation.