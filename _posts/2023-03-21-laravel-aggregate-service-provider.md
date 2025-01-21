---
title: Laravel – AggregateServiceProvider affects the performance
date: 2023-03-21
categories: [Programming, PHP, Laravel]
permalink: /2023/03/laravel-aggregateserviceprovider-affects-the-performance/
---
Some time ago I started wondering about the long bootstrap time of an application based on Laravel. I’ve started debugging and have figured out that this problem was related to the lack of deferred providers. It was strange because we’ve used a lot of deferred providers. After more debugging, we’ve found that AggregateProviders doesn’t respect deferred providers.

## Problem
Our application had over 60 AggregateServiceProviders similar to the following one:
```php
use Illuminate\Support\AggregateServiceProvider;

final class SampleProvider extends AggregateServiceProvider
{
    protected $providers = [
        EventsProvider::class,
        ImplementationsProvider::class,
        RoutingProvider::class,
    ];
}
```
In that case, ImplementationsProvider implements \Illuminate\Contracts\Support\DeferrableProvider according to the documentation:

> If your provider is only registering bindings in the service container, you may choose to defer its registration 
> until one of the registered bindings is actually needed. Deferring the loading of such a provider will improve the performance of your application, since it is not loaded from the filesystem on every request.

[https://laravel.com/docs/10.x/providers#deferred-providers](https://laravel.com/docs/10.x/providers#deferred-providers)

If we look at the method register in \Illuminate\Support\AggregateServiceProvider we can see the following implementation:

```php
public function register()
{
    $this->instances = [];

    foreach ($this->providers as $provider) {
        $this->instances[] = $this->app->register($provider);
    }
}
```

It means that every provider added to AggregateServiceProvider will be registered immediately on every request because to register a deferred provider in the container there should be used the method
```php
registerDeferredProvider($provider, $service = null);
```

## Don’t use AggregateServiceProvider
So the conclusion is simple, don’t use AggregateServiceProvider. I recommend using something like that:

```php
final class SampleProvider
{
    /**
     * @return string[]
     */
    public static function providers(): array
    {
        return [
            EventsProvider::class,
            ImplementationsProvider::class,
            RoutingProvider::class,
        ];
    }
}
```

Then in the configuration, we can load providers like that:

```php
'providers' => array_merge(
    // ...
    \App\Modules\Module\SampleProvider::providers(),
    // ...
);
```

## Test to verify the provides method
It’s easy to make a mistake during refactoring those providers, now we must have properly configured the provides method in all deferred providers because they will be loaded only when a particular dependency is needed and the container has to find the proper provider with the definition of this dependency. So it would be good to write a custom PHPStan rule or just a little tricky test in PHPUnit.

## Results after partial refactoring
Doing a partial refactoring of this problem (2/3 AggregateServiceProviders) we were able to decrease the bootstrap time of the application by over 20%. It’s probably just a few milliseconds per request (I estimate a 5 ms profit), of course, not too much, but it makes a difference on every request. Furthermore, in the future when the application will grow it will be a bigger problem, so it’s better to figure out this problem and avoid affecting the performance in a such stupid way.
