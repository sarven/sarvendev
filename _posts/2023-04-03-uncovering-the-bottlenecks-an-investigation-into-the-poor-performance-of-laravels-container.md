---
title: Uncovering the bottlenecks - An investigation into the poor performance of Laravel’s container
date: 2023-04-03
categories: [Programming, PHP, Laravel]
permalink: /2023/04/uncovering-the-bottlenecks-an-investigation-into-the-poor-performance-of-laravels-container/
image:
  path: /assets/img/2023-04-03/featured.png
---
Last time I’ve been analyzing the performance of Laravel’s container. I’ve encountered that the application spends a lot of time building dependencies, especially for heavy endpoints. That was strange because I would rather expect the relevant logic should be the heaviest part of the request.


Problem
Turns out, that by default every dependency in Laravel is non-shared. So when an application needs a specific dependency, the container will create and inject a new instance of this dependency. For example, in Symfony as default every dependency is shared:

> In the service container, all services are shared by default. This means that each time you retrieve the service, 
> you’ll get the same instance.
> https://symfony.com/doc/current/service_container/shared.html

So the implementation of the container in Laravel is weird because it means that every large application using Laravel will have a problem with injecting a lot of instances of dependencies. It wastes a lot of resources because an application has to recognize the parameters of a specific class using Reflection (auto-wiring magic 😄) and then build all of the dependencies and do the same things during building dependencies of this particular class and so on. 🤯

\Illuminate\Container\Container::resolve

```php
// If an instance of the type is currently being managed as a singleton we'll
// just return an existing instance instead of instantiating new instances
// so the developer can keep using the same objects instance every time.
if (isset($this->instances[$abstract]) && ! $needsContextualBuild) {
    return $this->instances[$abstract];
}
```

## How to solve that?
Use scoped or singleton instead of bind in providers:
```php
$this->app->scoped(ServiceInterface::class, Service::class);
$this->app->singleton(ServiceInterface::class, Service::class);
```

There is one difference between scoped and singleton.

> The scoped method binds a class or interface into the container that should only be resolved one time within a 
> given Laravel request / job lifecycle. While this method is similar to the singleton method, instances registered using the scoped method will be flushed whenever the Laravel application starts a new “lifecycle”, such as when a Laravel Octane worker processes a new request or when a Laravel queue worker processes a new job:
> https://laravel.com/docs/10.x/container#binding-scoped

You can also manually clear scoped dependencies:

```php
$this->container->forgetScopedInstances();
```

In most cases, that change should be safe, but be careful and check that your services are stateless.

## Auto-wired services
Unfortunately, non-shared dependencies are used by default. So it means that when we have auto-wired services without any interface, we don’t need to declare them in any provider, then we can’t make them shared.

To solve that problem, I figured out only the following nasty solution for that:

```php
$this->app->scoped(Service:class, Service:class)
```
So we can just create a provider and define the service using the scoped method to make that service shared.

## Comparison

**Before** 

![Blackfire profile 1](/assets/img/2023-04-03/profile1.png)

**After**

![Blackfire profile 2](/assets/img/2023-04-03/profile2.png)

These results are after some changes from bind to scoped, but still not all of them, so there is still room for improvement. The time in Blackfire is irrelevant, but we still see the performance boost by ~60% during building dependencies.

## Summary
I’m curious what was the root cause to implement the container in that way. It means that Laravel may not be the best option for large monolithic applications. A few years ago I worked with a huge monolith based on Symfony framework developed by around 50 backend developers, and I can’t even imagine using Laravel in such a project with this construction of the container. It probably would be a nightmare.
