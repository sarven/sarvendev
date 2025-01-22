---
title: Container Efficiency in Modular Monoliths - Symfony vs. Laravel
date: 2024-07-10
categories: [Programming, PHP, Symfony, Laravel]
permalink: /2024/07/container-efficiency-in-modular-monoliths-symfony-vs-laravel/
image:
  path: /assets/img/2024-07-10/featured.png
---
In the evolving landscape of software development, modular monolith architectures have gained significant traction. This approach offers a balanced middle ground between traditional monolithic applications and microservices. However, choosing the right PHP framework for building modular monoliths is crucial, as it can have a profound impact on the application’s performance.  The container performance, which is critical for dependency injection and managing service lifecycles, varies significantly between frameworks such as Symfony and Laravel. Despite the rise of framework-agnostic practices, the reality is that switching frameworks can be prohibitively expensive. It’s crucial to make an informed decision when choosing a framework. This article explores a comparative analysis of container performance in Symfony and Laravel within modular monolith architectures, offering engineers valuable insights to guide their framework selection.

## Motivation for this article
A poor performance of Laravel’s container isn’t a new thing. I’ve written about it an article: [Uncovering the 
bottlenecks: An investigation into the poor performance of Laravel’s container](https://sarvendev.com/2023/04/uncovering-the-bottlenecks-an-investigation-into-the-poor-performance-of-laravels-container/)

I had a high motivation to create some pull requests to Laravel to improve this performance.

The first and simplest one:  [New option – Shared dependencies by default](https://github.com/laravel/framework/pull/51209)

It was closed after a few minutes with some generic message, and I even started some discussion on X.

![Tweet PR Laravel](/assets/img/2024-07-10/tweet.png)

[https://x.com/Sarvendev/status/1783773524944961741](https://x.com/Sarvendev/status/1783773524944961741)
After a long discussion, I got some information, that maintainers are afraid of how this change could impact the whole Laravel ecosystem.

Then I found some small performance issue in the bootstrapping process, which I described here: Laravel: [Bootstrap 
time optimization by using a hashtable to store providers](https://sarvendev.com/2024/05/laravel-bootstrap-time-optimization-by-using-a-hashtable-to-store-providers/)

And it was merged. So it gave me hope that maybe with some smaller changes, it will be possible to fix these 
performance issues in the container. So I created another pull request with caching reflection data: [https://github.com/laravel/framework/pull/51794]
(https://github.com/laravel/framework/pull/51794)

But it was closed:
![PR comment](/assets/img/2024-07-10/pr-comment.png)

So for now, there is no option to improve the performance of the container due to the following:
- every longer change  to the core will be probably declined
- the container in Laravel is highly coupled with the rest of the framework and there is no possibility without any 
  hacky solution to use some different container

## Laravel vs Symfony
To show what can be improved in Laravel it would be best to compare it to the main competitor – Symfony. I’ve 
created a simple repository: [https://github.com/sarven/laravel-optimization-test/tree/laravel-vs-symfony](https://github.com/sarven/laravel-optimization-test/tree/laravel-vs-symfony)

I’ve added a few services but shaped them as a complicated dependency tree. Of course, it’s quite exaggerated, but in large monolithic applications is possible to have such a complex dependency tree.
![Dependency tree](/assets/img/2024-07-10/dependency-tree.png)

And I’ve written a test like this to compare these two frameworks:
```php
<?php

declare(strict_types=1);

use App\Services\RootService;
use App\Services\RootService2;
use Illuminate\Contracts\Http\Kernel;

require_once __DIR__ . '/laravel/vendor/autoload.php';

$count = 200;
$data = [];

for ($i = 0; $i < $count; $i++) {
    $app = require __DIR__ . '/laravel/bootstrap/app.php';
    $app->make(Kernel::class)->bootstrap();

    $start = microtime(true);
    $startMemory = memory_get_usage(true);
    $service = $app->make(RootService::class);
    $service = $app->make(RootService::class);
    $service = $app->make(RootService2::class);
    $service = $app->make(RootService2::class);
    $end = microtime(true);
    $endMemory = memory_get_usage(true);

    $data[] = [
        'time' => ($end - $start) * 1000,
        'memory' => $endMemory - $startMemory,
    ];

}

$averageTime = array_sum(array_column($data, 'time')) / $count;
$averageMemory = array_sum(array_column($data, 'memory')) / $count;

dump($averageTime . ' ms');
dump(($averageMemory / 1024) . ' KB');
```
```php
<?php

declare(strict_types=1);

use App\Kernel;
use App\Services\RootService;
use App\Services\RootService2;

require_once __DIR__ . '/symfony/vendor/autoload.php';

$count = 200;
$data = [];

for ($i = 0; $i < $count; $i++) {
    $kernel = new Kernel('dev', false);
    $kernel->boot();
    $container = $kernel->getContainer();

    $start = microtime(true);
    $startMemory = memory_get_usage(true);
    $service = $container->get(RootService::class);
    $service = $container->get(RootService::class);
    $service = $container->get(RootService2::class);
    $service = $container->get(RootService2::class);
    $end = microtime(true);
    $endMemory = memory_get_usage(true);

    $data[] = [
        'time' => ($end - $start) * 1000,
        'memory' => $endMemory - $startMemory,
    ];
}

$averageTime = array_sum(array_column($data, 'time')) / $count;
$averageMemory = array_sum(array_column($data, 'memory')) / $count;

dump($averageTime . ' ms');
dump(($averageMemory / 1024) . ' KB');
```

This test creates a new Kernel / Application and creates four instances of these two services. To make this more predictable, the test is repeated 200 times, and only the average values are presented.

**Results:**
**Laravel**
```
"19.366332292557 ms" // test.php:38
"20.48 KB" // test.php:39
```

**Symfony**
```
"0.017472505569458 ms"
"10.24 KB"
```

Laravel resolves all services at runtime, and there is no such thing in this case as a shared instance. So the 
container needs to create many instances of those objects. More about shared instances you can read here:
[https://sarvendev.com/2023/04/uncovering-the-bottlenecks-an-investigation-into-the-poor-performance-of-laravels-container/](https://sarvendev.com/2023/04/uncovering-the-bottlenecks-an-investigation-into-the-poor-performance-of-laravels
-container/)
So every time a service is needed, the container has to parse reflection data, create dependencies, create dependencies of dependencies, and so on. The more complicated the dependency tree, the more processing time it takes. Here is the sample screenshot from the profiler (Blackfire) which shows how it looks under the hood (it’s from the real application, not the test one):

![Profile](/assets/img/2024-07-10/profile.png)

On the other hand, Symfony has a precompiled configuration, which speeds up resolving dependencies. All instances are shared by default.
> In the service container, all services are shared by default. This means that each time you retrieve the service, you’ll get the same instance.
> [https://symfony.com/doc/current/service_container/shared.html](https://symfony.com/doc/current/service_container/shared.html)

The container is compiled before going to the production, and it looks like this:
```php
class getRootServiceService extends App_KernelDevDebugContainer
{
    /**
     * Gets the public 'App\Services\RootService' shared autowired service.
     *
     * @return \App\Services\RootService
     */
    public static function do($container, $lazyLoad = true)
    {
        include_once \dirname(__DIR__, 4).'/src/Services/RootService.php';

        return $container->services['App\\Services\\RootService'] = new \App\Services\RootService(
            ($container->privates['App\\Services\\GlobalService'] ?? $container->load('getGlobalServiceService')),
            ($container->privates['App\\Services\\GlobalService1'] ?? $container->load('getGlobalService1Service')),
            ($container->privates['App\\Services\\GlobalService2'] ?? $container->load('getGlobalService2Service')),
            ($container->privates['App\\Services\\GlobalService3'] ?? $container->load('getGlobalService3Service')),
            ($container->privates['App\\Services\\GlobalService4'] ?? $container->load('getGlobalService4Service')),
            ($container->privates['App\\Services\\GlobalService5'] ?? $container->load('getGlobalService5Service'))
        );
    }
}
```

## Laravel possible improvement
Laravel could be improved in a few smaller steps:
- caching reflection data – it should speed up a little resolving the same dependency many times
- cached reflection data generated during the deployment of the application – it could be saved to the file as other cache files: config, routes, etc.
- precompilation of the full container during the deployment – similar to the Symfony approach
- possibility to define auto-wired services as shared by default – now, it is possible for services that need binding 
(scoped, singleton), but currently everything without an interface doesn’t need to be declared in providers and 
there is no possibility to define those services as shared (exactly there is a possibility, but then every service 
needs to be defined in providers, [Read more here](https://sarvendev.com/2023/04/uncovering-the-bottlenecks-an-investigation-into-the-poor-performance-of-laravels-container/) – Auto-wired services)

## Summary
Many factors impact our decision in choosing the right framework for our application. This time, I compared the performance of containers. Bearing in mind the performance of these two container implementations, I think that Symfony is a much better choice for a modular monolith architecture. For smaller applications, the difference won’t be as noticeable as it is for larger ones, but if you strive for better performance, you should thoroughly consider your decision.
