---
title: Laravel - Bootstrap time optimization by using a hashtable to store providers
date: 2024-05-28
categories: [Programming, PHP, Laravel]
permalink: /2024/05/laravel-bootstrap-time-optimization-by-using-a-hashtable-to-store-providers/
image:
  path: /assets/img/2024-05-28/featured.png
---
Having a profiler and performance monitoring is crucial for maintaining the efficiency and reliability of any application. Profilers help engineers identify bottlenecks by providing detailed insights into the application’s performance, such as memory usage, CPU load, and execution time of various functions.

## Problem
In one project that I maintain, we use a profiler Blackfire to monitor performance and create profiles on demand. Recently, I was checking what’s going on with the Laravel bootstrap time, which is quite slow. Laravel has some performance problems that I described in some previous posts:
- [AggregateServiceProvider affects performance](https://sarvendev.com/2023/03/laravel-aggregateserviceprovider-affects-the-performance/)
- [Uncovering the bottlenecks: An investigation into the poor performance of Laravel’s container](https://sarvendev.com/2023/04/uncovering-the-bottlenecks-an-investigation-into-the-poor-performance-of-laravels-container/)

To register dependencies in the container in Laravel we need to add a ServiceProvider with proper bindings. Large applications contain hundreds of providers.

![Blackfire profile 1](/assets/img/2024-05-28/profile1.png)

I found something like this in the profile of one of the requests. And it was weird that some function is called over 60k times. During bootstrapping the application, every provider needs to be registered. During the register method, there is a check to verify if the provider is already registered and it’s done by using the getProvider method.

```php
public function getProvider($provider)
{
    return array_values($this->getProviders($provider))[0] ?? null;
}
```
```php
public function getProviders($provider)
{
    $name = is_string($provider) ? $provider : get_class($provider);

    return Arr::where($this->serviceProviders, fn ($value) => $value instanceof $name);
}
```

So it means that for every check if the provider is registered we need to iterate over all of the registered providers. It gives us a complexity – ((n – 1) * n) / 2, where n is a number of providers. So for small applications with let’s say 20 providers, it’s 190 executions, but for 300 providers it’s almost 50k executions.

## Solution
The solution to that problem is quite simple, we can just use a data structure – hashtable.

```php
/**
 * All of the registered service providers.
 *
 * @var array<string, \Illuminate\Support\ServiceProvider>
 */
protected $serviceProviders = [];
```

```php
public function getProvider($provider)
{
    $name = is_string($provider) ? $provider : get_class($provider);

    return $this->serviceProviders[$name] ?? null;
}
```

After the change, the complexity of the getProvider method is equal O(1). Of course, the time from Blackfire is not true (profiling overhead), so it isn’t optimized by tens of milliseconds, rather than a few milliseconds.

Merged pull request to Laravel: [https://github.com/laravel/framework/pull/51343/files](https://github.com/laravel/framework/pull/51343/files)

I have a plan to report more optimization changes to Laravel, so if you’re interested in that topic, subscribe to the blog to avoid missing any new posts.
