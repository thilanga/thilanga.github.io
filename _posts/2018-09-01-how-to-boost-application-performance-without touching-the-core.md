---
layout: post
title: "How to boost application performance (without touching the core)"
date: 2018-09-01
comments: true
tags: design-patterns performance clean-code php
description: How to boost application performance (without touching the core) 
---
```
When talking about making performance improvements to an application, the first thing that comes to mind is caching.

The next thought that usually follows is, ‘What is the easiest and quickest way to implement it?’.
```

When building software, inevitably there comes the stage where the development team will start talking about performance
improvements. And that’s when fingers get pointed.

Some in the team will say didn't have enough time to think about it, due to the tight deadlines. Others will say there
are too many changes to be made, and now they don't want to touch the code.

That’s why we, as developers, should think about the application’s performance at the very start of the project. Yes,
even before we open the code editor and write the very first line.

I'll take an example from the project I'm currently working on.

In my team, we were fortunate that performance improvements could be made without making changes to the core code base.
The application is built using nested set model (tree architecture). [link wiki]. We use MongoDB to store the data and
the application lifts the weight of object loading.

In simple terms, we have only one class which is responsible for all data objects in the system. Yes,
there are few more classes that help handling data validation, related data loading, and more, but the core of the
application is one class. Any data point in the system is an instance of that main class.

## The Problem

This architecture has helped us scale the system without having to make any core changes to the code base.
In fact, the core of the application hasn't been touched in years.

But even though this architecture has lots of pros in our case, it has its drawbacks when it comes to performance.

Our benchmark was for any endpoint to load within less than a second. Our application is very data heavy,
and some endpoints were taking up to 4 seconds when doing calculations against big datasets.

As we looked into the issue, it turned out to be that some helper classes responsible for calculations were regularly being calling.
Especially when pulling data from 3 or 4 levels down the chain.

## We made two key changes to improve performance.

1. The first solution was to optimise the methods that load data objects. We used lazy loading on all the nested data
relationships to reduce this stress.
2. The second solution was to cache data that was already being called during the process.

### Architecture of the application
Before we dive deeper into cache implementation, let's have a look at the application structure.

 - Backend API - Laravel
 - Frontend - VueJS
 - Database - mongoDB
 - Cache - Redis
 - The application is also deployed using Docker and deployment is done in separate containers.


> Looking at the architecture of the application, we had plenty of freedom to optimise along many levels.

## Cache Implementation

>Before we got started, we made few rules:
1. We 	could only change the core code as a last resort.
2. We 	needed to utilise our existing resources
3. Implementation shouldn't take long, to avoid any major refactoring.

Given the above constraints, it was clear we had to use an observer pattern as a wrapper to the application so we wouldn’t touch the core code.

## Implementation

Now we knew we were building the cache layer using an observer pattern, we started utilising [Laravel's Service Container](https://laravel.com/docs/master/container/){:target="_blank"}.

And since the bottleneck was the constant data load from the database, we started by injecting a concrete implimentation of the data class.

> Inject concrete class with the help of the service provider

```php
class BusinessPlansServiceProvider extends ServiceProvider
{

    public function register()
    {
        $this->app->singleton(Plan::class, function () {
            // return new Plan object;
            return new Plan();
        });

        $this->app->singleton(PlanDataList::class, function () {
            // return new Table;
            return new PlanDataList();
        });
    }
}
```

> Resolving injected class within the logic class

```php
    protected function addBucket($name, $key)
    {
        $data = app(PlanDataList::class);
        $data->setPlanId($this->getPlanId());
        $loadedData = $data->loadData($name);

        $this->addRow(is_string($key) ? $key : $name, $loadedData);
    }
```

The next step was to create interfaces with data fetching methods. The reason behind this approach was to have a class that can load data from the cache with the same method names. This way, there was no need to change the core code.

## Interfaces

Once we had all those heavy data loading methods being extracted to interfaces, it was time to implement cache supported class.

> `PlanDataListInterface.php`

```php
interface PlanDataListInterface
{
    public function setPlanId(string $planId);
    public function loadData(string $bucket);
}
```

> `PlanInterface.php`

```php
interface PlanInterface
{
    public function getPlan(string $planId);
}
```

> Protip: Be wise when creating your cache tags and remember to use short cache tag names. Redis will thank you in the long run.

## Cache supported class

Once we created the cache supported classes, we could easily replace them with our concrete class with the help of the [Service Container](https://laravel.com/docs/master/container/){:target="_blank"}.


```php
class BusinessPlansServiceProvider extends ServiceProvider
{
    public function register()
    {
        $this->app->singleton(Plan::class, function () {
            // return new Plan object;
            return new PlanCache(new Plan());
        });

        $this->app->singleton(PlanDataList::class, function () {
            // return new Table;
            return new PlanDataCache(new PlanDataList());
        });
    }
}
```

> `PlanCache.php`

```php
class PlanCache implements PlanInterface
{
    use CacheTracker;
    const CACHE_LIFE_TIME = 8;

    private $next;

    public function __construct(Plan $next)
    {
        $this->next = $next;
    }

    public function getPlan(string $planId)
    {
        $expiresAt = Carbon::now()->addHours(self::CACHE_LIFE_TIME);

        return Cache::tags(CacheTracker::genCacheTags())
                ->remember(CacheTracker::genCacheKey(['get_plan',]), $expiresAt,
                    function () use ($planId) {
                        return $this->next->getPlan($planId);
                    });
    }
}
```


> `PlanDataCache.php`

```php
class PlanDataCache implements PlanDataListInterface
{
    use CacheTracker;
    const CACHE_LIFE_TIME = 8;

    private $next;

    public function __construct(PlanDataList $next)
    {
        $this->next = $next;
    }

    public function setPlanId(string $planId)
    {
        $expiresAt = Carbon::now()->addHours(self::CACHE_LIFE_TIME);

        return Cache::tags(CacheTracker::genCacheTags())
                ->remember(CacheTracker::genCacheKey(['setPlanId']), $expiresAt,
                    function () use ($planId) {
                        return $this->next->setPlanId($planId);
                    });
    }

    public function loadData(string $bucket)
    {
        $expiresAt = Carbon::now()->addHours(self::CACHE_LIFE_TIME);

        return Cache::tags(CacheTracker::genCacheTags())
                ->remember(CacheTracker::genCacheKey(['loadData', $bucket]), $expiresAt,
                    function () use ($bucket) {
                        return $this->next->loadData($bucket);
                    });
    }
}
```


And there you have it.

> We have implemented a caching wrapper without disrupting the core of the application.

Now when the application calls any data loading methods it'll look in the cache first before it talks to the database.

## A final word: Cache expiration

As I mentioned earlier, in our application there is only one core class responsible for talking to database. We implemented cache expiration using the boot method in Elequent. But if you are not using Laravel, you can use the database model events that come with the library that you use.

`CacheTracker.php` was used as a helper class to support the cache implimentation

> `CacheTracker.php`

```php
trait CacheTracker
{
    public static function bootCacheTracker()
    {
        static::created(function ($instance) {
            self::flushCache();
        });

        static::deleted(function ($instance) {
            self::flushCache();
        });

        static::saved(function ($instance) {
            self::flushCache();
        });


        static::updated(function ($instance) {
            self::flushCache();
        });
    }

    public static function flushCache(array $tags = [])
    {
        if (count($tags) === 0) {
            $tags = self::genCacheTags();
        }

        Cache::tags($tags)->flush();
    }

    public static function genCacheTags()
    {
        $arrForKey = [
            'module',
            'business'
        ];

        return $arrForKey;
    }

    public static function genCacheKey($args = [])
    {
        $filterQuery = (app(FilterQuery::class));
        $filterArray = $filterQuery->getFilterArray(request());

        $businessId = $filterQuery->pluckBusinessIdFromFilter(collect($filterArray))->first();
        $planId = $filterQuery->pluck(collect($filterArray), 'id')->first();

        $arrForKey = [
            'bi_',
            $businessId,
            $planId
        ];

        $cacheKey = implode('_', array_merge($arrForKey, $args));

        return md5($cacheKey);
    }
}
```

> Protip: When it comes to cache invalidation we you need to make sure you use correct cache keys. Otherwise, it'll drop the whole cache when you transact with the database each and every time.
