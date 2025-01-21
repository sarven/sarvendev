---
title: Laravel – variadic parameter trap
date: 2023-03-14
categories: [Programming, PHP, Laravel]
permalink: /2023/03/laravel-variadic-parameter-trap/
---
What do you expect from the framework if the dependency cannot be created? Sure, it should be an exception, but I had an unpleasant surprise.

## Problem
I had code similar to the following:
```php
final class Service
{
    /**
    * @var HandlerInterface[]
    */
    private readonly array $handlers;

    public function __construct(
        HandlerInterface ...$handlers,
    ) {
        $this->handlers = $handlers;
    }

    public function doSomething(string $type): void
    {
        foreach ($this->handlers as $handler) {
            if ($handler->supports($type)) {
                $handler->handle();
            }
        }
    }
}
```
And the proper configuration in the ServiceProvider:
```php
$this
    ->app
    ->when(Service::class)
    ->needs(HandlerInterface::class)
    ->giveTagged(HandlerInterface::class)
;
```
Then I decided to refactor ServiceProvider, where the handler was defined, to DeferrableProvider and unfortunately made a mistake in the provides method. It caused Laravel couldn’t find the definition of HandlerInterface. In that case, I would expect an exception, and there was the exception under the hood – BindingResolutionException, but the container of Laravel has a catch as follows:

\Illuminate\Container\Container::resolveClass

```php
catch (BindingResolutionException $e) {
    if ($parameter->isDefaultValueAvailable()) {
        array_pop($this->with);

        return $parameter->getDefaultValue();
    }

    if ($parameter->isVariadic()) {
        array_pop($this->with);

        return [];
    }

    throw $e;
}
```

This means that even if the dependency cannot be created and a variadic parameter is used it will bind an empty array by default. This is quite odd and rather doesn’t seem to be in line with the Fail Fast principle. Also, my integration tests didn’t catch this bug, because it was related to communication between two modules, and according to the Modular Monolith architecture, I just bind a fake object to my tests.

## Improvement
```php
use Webmozart\Assert\Assert;

final class Service
{
    /**
     * @var HandlerInterface[]
     */
    private readonly array $handlers;

    /**
     * @param HandlerInterface[] $handlers
     */
    public function __construct(
        array $handlers,
    ) {
        Assert::allIsInstanceOf($handlers, HandlerInterface::class);
        Assert::notEmpty($handlers);
        $this->handlers = $handlers;
    }

    public function doSomething(string $type): void
    {
        foreach ($this->handlers as $handler) {
            if ($handler->supports($type)) {
                $handler->handle();
            }
        }
    }
}
```
```php
$this
    ->app
    ->when(Service::class)
    ->needs('$handlers')
    ->giveTagged(HandlerInterface::class)
;
```
The improvement is quite simple, we can just throw an exception as fast as possible when something unexpected happens e.g. $handlers array is empty.
