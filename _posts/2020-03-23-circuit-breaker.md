---
title: Circuit Breaker
date: 2020-03-23
categories: [Programming, Patterns]
permalink: /2020/03/circuit-breaker/
image:
  path: /assets/img/2020-03-23/featured.png
---
In most systems, we use remote calls. Many factors may have an impact on these remote calls e.g. network latency, server availability and so on. So we should assume that something can go wrong. These calls can be potential bottlenecks, we don’t want user waiting for the response from the server very long, because external API is very slow or not available. Also if we have a few services which communicate with each other we shouldn’t aggravate the situation when one of them has too much traffic and slow down significantly. So how to do it correctly?


## Example problem
Let’s check a very simple example with two applications, where one is calling another. The first app has two endpoints:
- endpoint1 which is calling the second application
- endpoint2 which returns only a string.

```php
<?php

require 'vendor/autoload.php';

$dispatcher = FastRoute\simpleDispatcher(static function(FastRoute\RouteCollector $r) {
    $r->addRoute('GET', '/endpoint1', static function () {
         echo file_get_contents('http://app2');
    });
    $r->addRoute('GET', '/endpoint2', static function () {
        echo 'endpoint2';
    });
});

$httpMethod = $_SERVER['REQUEST_METHOD'];
$uri = $_SERVER['REQUEST_URI'];

if (false !== $pos = strpos($uri, '?')) {
    $uri = substr($uri, 0, $pos);
}
$uri = rawurldecode($uri);

$routeInfo = $dispatcher->dispatch($httpMethod, $uri);
switch ($routeInfo[0]) {
    case FastRoute\Dispatcher::NOT_FOUND:
        break;
    case FastRoute\Dispatcher::METHOD_NOT_ALLOWED:
        $allowedMethods = $routeInfo[1];
        break;
    case FastRoute\Dispatcher::FOUND:
        $handler = $routeInfo[1];
        $handler();
        break;
}
```

The second application has only one endpoint1 which returns only a string.

```php
<?php

require 'vendor/autoload.php';

$dispatcher = FastRoute\simpleDispatcher(static function(FastRoute\RouteCollector $r) {
    $r->addRoute('GET', '/', static function () {
        echo 'service2';
    });
});

$httpMethod = $_SERVER['REQUEST_METHOD'];
$uri = $_SERVER['REQUEST_URI'];

if (false !== $pos = strpos($uri, '?')) {
    $uri = substr($uri, 0, $pos);
}
$uri = rawurldecode($uri);

$routeInfo = $dispatcher->dispatch($httpMethod, $uri);
switch ($routeInfo[0]) {
    case FastRoute\Dispatcher::NOT_FOUND:
        break;
    case FastRoute\Dispatcher::METHOD_NOT_ALLOWED:
        $allowedMethods = $routeInfo[1];
        break;
    case FastRoute\Dispatcher::FOUND:
        $handler = $routeInfo[1];
        $handler();
        break;
}
```

Now we can check the performance of the first application using JMeter. On the following screen, we can see a few parameters describing the performance of the application:
- Samples – the number of requests
- Average – the sum of all response times / count of responses
- Min – minimal response time in milliseconds
- Max – maximum response time in milliseconds
- Std. Dev. – standard deviation
- Error % – the percentage of failed tests
- Throughput – the number of requests per second which a server can handle

![JMeter](/assets/img/2020-03-23/jmeter1.png)

Ok now, let’s simulate that service2 works very slowly. We can do it by just adding a sleep method.

```php
$r->addRoute('GET', '/', static function () {
    sleep(5);
    echo 'service2';
});
```
Now we can see that Jmeter’s report looks completely different. There are so many errors, but what is most interesting for us that despite endpoint2 from the first service isn’t using any remote calls also has a very high error rate. Only one service has problems, but it has an impact on all other services which are using this one. It’s an undesirable effect. We should design our code in the way where one service will be still available and functionalities unrelated to problematic service will be available.

![JMeter](/assets/img/2020-03-23/jmeter2.png)

## Circuit breaker
The pattern “Circuit breaker” was popularized by Michael Nygard in his book “Release It”. Most likely the name came from Electrical Engineering where it is defined by:

> A circuit breaker is an automatically operated electrical switch designed to protect an electrical circuit from damage caused by excess current from an overload or short circuit. Its basic function is to interrupt current flow after a fault is detected. https://en.wikipedia.org/wiki/Circuit_breaker

In the programming world, we use this pattern due to very similar reasons. The idea of Circuit Breaker is very simple. We wrap every calls by a special object which besides makes the call, monitor also the status of the target service.

![Circuit Breaker](/assets/img/2020-03-23/circuit-breaker.png)

In comparison to the electrical circuit breaker, the pattern from our world is even better designed, because the electrical circuit breaker can’t do a self-repair. If something is wrong the switch will turn off automatically, but if you want to turn on it again you have to do it manually. In the programming, it will be repaired automatically.

In this pattern we can distinguish three states:
- open
- closed
- half-open
The default state is closed, which means that a resource probably is fine or the count of failures is still under the defined threshold. In this state, every request is going to the target service.

When the count of failures is over the threshold, the pattern is marked as open and requests aren’t going to the target service. It is immediately returned cached data or just timeout error.

But if we want Circuit Breaker to be able for self-repair, sometimes a new request has to go to the target service to check that there is still a problem or maybe now is everything fine. For this reason is the third state half-open, which means that periodically a new request is going to the target service. If the request is successful the resource will be marked as closed or as open if the request is unsuccessful.

## Ganesha – PHP implementation
The most popular implementation of Circuit Breaker in PHP is [Ganesha](https://github.com/ackintosh/ganesha). The implementation is quite simple. In Circuit Breaker is a need to collect service statistics. Ganesha provides three ways to do it: Redis, Memcached and MongoDB. There are two strategies which work a little different:

### Rate (default)
```php
/**
 * @param  string $service
 * @return int
 */
public function recordFailure($service)
{
    $this->storage->setLastFailureTime($service, time());
    $this->storage->incrementFailureCount($service);
    if (
        $this->storage->getStatus($service) === Ganesha::STATUS_CALMED_DOWN
        && $this->isClosedInCurrentTimeWindow($service) === false
    ) {
        $this->storage->setStatus($service, Ganesha::STATUS_TRIPPED);
        return Ganesha::STATUS_TRIPPED;
    }
    return Ganesha::STATUS_CALMED_DOWN;
}

/**
 * @param  string $service
 * @return null | int
 */
public function recordSuccess($service)
{
    $this->storage->incrementSuccessCount($service);
    if (
        $this->storage->getStatus($service) === Ganesha::STATUS_TRIPPED
        && $this->isClosedInPreviousTimeWindow($service)
    ) {
        $this->storage->setStatus($service, Ganesha::STATUS_CALMED_DOWN);
        return Ganesha::STATUS_CALMED_DOWN;
    }
}

/**
 * @param  string $service
 * @return bool
 */
public function isAvailable($service)
{
    if ($this->isClosed($service) || $this->isHalfOpen($service)) {
        return true;
    }
    $this->storage->incrementRejectionCount($service);
    return false;
}
```

Above we can see three methods from Rate strategy. Based on this it’s clear what statistics are collecting:
- the last failure time
- the count of failures
- the count of successes
- the circuit breaker state
- the count of rejections, rejection is when the pattern has open state and request aren’t going outside

In this strategy we have to define five parameters:
- timeWindow – the interval in time (seconds) that evaluate the thresholds
- failureRateThreshold – the failure rate threshold in percentage that changes CircuitBreaker’s state to open
- minimumRequests – the minimum number of requests to detect failures. Even if failureRateTreshold exceeds the threshold, CircuitBreaker remains in closed if minimumRequests is below this threshold
- intervalToHalfOpen – seconds to change CircuitBreaker’s state from open to half open
- adapter – just a storage adapter

According to the constraints of the storage functionalities, there are two ways of selecting statistics:
- SlidingTimeWindow (Redis and MongoDB)
- TumblingTimeWindow (Memcached)

You can check the details [in the documentation](https://github.com/ackintosh/ganesha).

### Count
```php
/**
 * @return int
 */
public function recordFailure($service)
{
    $this->storage->setLastFailureTime($service, time());
    $this->storage->incrementFailureCount($service);
    if ($this->storage->getFailureCount($service) >= $this->configuration['failureCountThreshold']
        && $this->storage->getStatus($service) === Ganesha::STATUS_CALMED_DOWN
    ) {
        $this->storage->setStatus($service, Ganesha::STATUS_TRIPPED);
        return Ganesha::STATUS_TRIPPED;
    }
    return Ganesha::STATUS_CALMED_DOWN;
}

/**
 * @return void
 */
public function recordSuccess($service)
{
    $this->storage->decrementFailureCount($service);
    if ($this->storage->getFailureCount($service) === 0
        && $this->storage->getStatus($service) === Ganesha::STATUS_TRIPPED
    ) {
        $this->storage->setStatus($service, Ganesha::STATUS_CALMED_DOWN);
    }
}

/**
 * @return bool
 */
public function isAvailable($service)
{
    return $this->isClosed($service) || $this->isHalfOpen($service);
}
```

In the Count strategy there are collecting fewer statistics. There hasn’t a count of successes and the count of rejections. It is because Count strategy works in a different way and doesn’t need this data. In this strategy we have to define three parameters:
- failureCountThreshold – the failure count threshold that changes CircuitBreaker’s state to open
- intervalToHalfOpen – seconds to change CircuitBreaker’s state from open to half open
- adapter – just a storage adapter

## Example usage
Now it’s time for an example of how it works. Ganesha can be integrated with Guzzle. So we start with Composer and installation of these tools.
```
composer require ackintosh/ganesha guzzlehttp/guzzle
```
Let’s check performance with just Guzzle implementation.
```php
<?php

require 'vendor/autoload.php';

use Ackintosh\Ganesha\Exception\RejectedException;
use GuzzleHttp\Client;
use GuzzleHttp\Exception\ConnectException;

$client = new Client(['timeout' => 4]);

$dispatcher = FastRoute\simpleDispatcher(static function(FastRoute\RouteCollector $r) use ($client) {
    $r->addRoute('GET', '/endpoint1', static function () use ($client) {
        try {
            echo $client->get('http://app2p')->getBody();
        } catch (ConnectException $e) {
            echo $e->getMessage();
        } catch (RejectedException $e) {
            echo $e->getMessage();
        }
    });
    $r->addRoute('GET', '/endpoint2', static function () {
        echo 'endpoint2';
    });
});

$httpMethod = $_SERVER['REQUEST_METHOD'];
$uri = $_SERVER['REQUEST_URI'];

if (false !== $pos = strpos($uri, '?')) {
    $uri = substr($uri, 0, $pos);
}
$uri = rawurldecode($uri);

$routeInfo = $dispatcher->dispatch($httpMethod, $uri);
switch ($routeInfo[0]) {
    case FastRoute\Dispatcher::NOT_FOUND:
        break;
    case FastRoute\Dispatcher::METHOD_NOT_ALLOWED:
        $allowedMethods = $routeInfo[1];
        break;
    case FastRoute\Dispatcher::FOUND:
        $handler = $routeInfo[1];
        $handler();
        break;
}
```
As we can see the results from Jmeter are better than previously. It’s because we are able to define a timeout in Guzzle. But there are also still over 20% errors. Timeouts aren’t a good solution, because when we define 4 seconds timeout, every request must wait 4 seconds. It’s the reason why we need something better. Also, we need to know what a timeout will be suitable for our case.

![JMeter](/assets/img/2020-03-23/jmeter3.png)

Let’s check Ganesha with Guzzle and its performance. The implementation is very simple. We use GuzzleMiddleware.

```php
<?php

require 'vendor/autoload.php';

use Ackintosh\Ganesha\Builder;
use Ackintosh\Ganesha\GuzzleMiddleware;
use Ackintosh\Ganesha\Exception\RejectedException;
use GuzzleHttp\Client;
use GuzzleHttp\HandlerStack;
use GuzzleHttp\Exception\ConnectException;

$redis = new \Redis();
$redis->connect('redis_1');
$adapter = new Ackintosh\Ganesha\Storage\Adapter\Redis($redis);

$ganesha = Builder::build([
    'timeWindow'           => 30,
    'failureRateThreshold' => 50,
    'minimumRequests'      => 10,
    'intervalToHalfOpen'   => 5,
    'adapter'              => $adapter,
]);
$middleware = new GuzzleMiddleware($ganesha);

$handlers = HandlerStack::create();
$handlers->push($middleware);

$client = new Client(['handler' => $handlers, 'timeout' => 4]);

$dispatcher = FastRoute\simpleDispatcher(static function(FastRoute\RouteCollector $r) use ($client) {
    $r->addRoute('GET', '/endpoint1', static function () use ($client) {
        try {
            echo $client->get('http://app2p')->getBody();
        } catch (ConnectException $e) {
            echo $e->getMessage();
        } catch (RejectedException $e) {
            echo $e->getMessage();
        }
    });
    $r->addRoute('GET', '/endpoint2', static function () {
        echo 'endpoint2';
    });
});

$httpMethod = $_SERVER['REQUEST_METHOD'];
$uri = $_SERVER['REQUEST_URI'];

if (false !== $pos = strpos($uri, '?')) {
    $uri = substr($uri, 0, $pos);
}
$uri = rawurldecode($uri);

$routeInfo = $dispatcher->dispatch($httpMethod, $uri);
switch ($routeInfo[0]) {
    case FastRoute\Dispatcher::NOT_FOUND:
        break;
    case FastRoute\Dispatcher::METHOD_NOT_ALLOWED:
        $allowedMethods = $routeInfo[1];
        break;
    case FastRoute\Dispatcher::FOUND:
        $handler = $routeInfo[1];
        $handler();
        break;
}
```

![JMeter](/assets/img/2020-03-23/jmeter4.png)

Now we have no errors. How it is possible? Though the second app is “not working” (of course we are just simulating such a behavior). In this situation circuit breaker is working and after a few failed requests the state is set to open and next requests are not going to the second service. The response is returned immediately, so we can inform the user about the problem. In the strategy called “Design for Failure” it is better to return error than force users for waiting a long time.

## Summary
Circuit Breaker is a very helpful pattern. It should be used in most cases when an application uses some kind of APIs. This pattern helps an application to know when its dependent services are down. When it’s known, the application can returns data from cache or just returns suitable feedback. However, sometimes the pattern may have no sense. For example when between services requests are sending very rarely then we don’t have time to collect the right number of statistics and consequently changing the state of Circuit Breaker is sporadic.

