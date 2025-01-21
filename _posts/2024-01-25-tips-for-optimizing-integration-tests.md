---
title: Tips for optimizing integration tests
date: 2024-01-25
categories: [Programming, PHP, Testing]
permalink: /2024/01/tips-for-optimizing-integration-tests/
---
While unit tests are known for their speed compared to integration tests, the latter offer heightened confidence in the system’s functionality. Thus, avoiding integration tests is not advisable; instead, it’s crucial to strike a balance by writing tests at an appropriate level to ensure high confidence in the codebase. Achieving this equilibrium between time efficiency and confidence is paramount. Rapid feedback is essential for a smooth workflow, and today, I’ll share concise tips to enhance the efficiency of your integration tests. The effort invested is worthwhile, as swift feedback is indispensable for seamless operations, and each minute of improvement is magnified by the frequency of executions and the number of developers in the company.

## Opcache
Ensure you have set up enable_cli and other configs adjusted to your projects.

```
opcache.enable_cli=1
opcache.max_accelerated_files= 130987
opcache.interned_strings_buffer=64
```

You can also add these parameters to PHP executing PHPUnit:

```
php -dopcache.validate_timestamps=0 vendor/bin/phpunit
```

validate_timestamps=0 shouldn’t be set up for the dev environment, but you can add this only to the command executing all tests in the CI pipeline.

## Optimize composer autoloader
```
composer install --optimize-autoloader --classmap-authoritative
```

Add flags to the composer install to optimize the autoloader.

## Use transactions
Use database transactions to clear the database to the initial state after every case. Here you can find more information:

For Laravel: [https://laravel.com/docs/10.x/database-testing#resetting-the-database-after-each-test](https://laravel.
com/docs/10.x/database-testing#resetting-the-database-after-each-test)

For Symfony: [https://github.com/dmaicher/doctrine-test-bundle](https://github.com/dmaicher/doctrine-test-bundle)

Make sure that you don’t use TRUNCATE, it’s very slow. Especially it’s observable after switching from MySQL 5.7 to 8, TRUNCATE is much slower on MySQL 8.

If you use Doctrine with other frameworks than Symfony, make sure that you also use static cache implemented like the following:

```php
<?php

namespace DAMA\DoctrineTestBundle\Doctrine\Cache;

use Psr\Cache\CacheItemInterface;
use Psr\Cache\CacheItemPoolInterface;
use Symfony\Component\Cache\Adapter\ArrayAdapter;

final class Psr6StaticArrayCache implements CacheItemPoolInterface
{
    /**
     * @var array<string, ArrayAdapter>
     */
    private static $adaptersByNamespace;

    /**
     * @var ArrayAdapter
     */
    private $adapter;

    public function __construct(string $namespace)
    {
        if (!isset(self::$adaptersByNamespace[$namespace])) {
            self::$adaptersByNamespace[$namespace] = new ArrayAdapter(0, false);
        }
        $this->adapter = self::$adaptersByNamespace[$namespace];
    }

    /**
     * @internal
     */
    public static function reset(): void
    {
        self::$adaptersByNamespace = [];
    }

    public function getItem($key): CacheItemInterface
    {
        return $this->adapter->getItem($key);
    }

    public function getItems(array $keys = []): iterable
    {
        return $this->adapter->getItems($keys);
    }

    public function hasItem($key): bool
    {
        return $this->adapter->hasItem($key);
    }

    public function clear(): bool
    {
        return $this->adapter->clear();
    }

    public function deleteItem($key): bool
    {
        return $this->adapter->deleteItem($key);
    }

    public function deleteItems(array $keys): bool
    {
        return $this->adapter->deleteItems($keys);
    }

    public function save(CacheItemInterface $item): bool
    {
        return $this->adapter->save($item);
    }

    public function saveDeferred(CacheItemInterface $item): bool
    {
        return $this->adapter->saveDeferred($item);
    }

    public function commit(): bool
    {
        return $this->adapter->commit();
    }
}
```
[https://github.com/dmaicher/doctrine-test-bundle/blob/master/src/DAMA/DoctrineTestBundle/Doctrine/Cache
/Psr6StaticArrayCache.php](https://github.com/dmaicher/doctrine-test-bundle/blob/master/src/DAMA/DoctrineTestBundle/Doctrine/Cache/Psr6StaticArrayCache.php)

```php
$em->getConfiguration()->setMetadataCache($staticCache);
$em->getConfiguration()->setQueryCache($staticCache);
$em->getMetadataFactory()->setCache($staticCache);
```
It also has a significant impact on overall performance.

## Use tmpfs for the database

Define in your docker-compose.yml for the database:

MySQL
```
tmpfs:
  - /var/lib/mysql
```

PostgreSQL
```
tmpfs:
  - /var/lib/postgresql/data
```
tmpfs mount is persisted in the memory. When the container stops, the tmpfs mount is removed, and files written there won’t be persisted.

## Use a dump of the database instead of using migrations every time
If you have a lot of migrations and execute them before tests, you can generate a dump and just import this dump to a database. It’ll be much faster than recreating a database from migrations.

## Clear memory used by properties
If you use PHPUnit you probably can observe that after every case more and more memory is used. It’s because there is some reference from PHPUnit to TestCase objects and the garbage collector has a problem cleaning it up. To fix that issue you can use the following code in the tearDown method.

```php
// Remove properties defined during the test
$refl = new \ReflectionObject($this);
foreach ($refl->getProperties() as $prop) {
   if (!$prop->isStatic() && 0 !== strpos($prop->getDeclaringClass()->getName(), 'PHPUnit_')) {
       $prop->setAccessible(true);
       $prop->setValue($this, null);
   }
}
```
[https://stackoverflow.com/a/37864440](https://stackoverflow.com/a/37864440)

## Build up an application using production settings
You should check which configs of a framework should be turned on in tests. For example, in Laravel you can before tests call these commands to generate a cache:

```
config:cache
routing:cache
```

And set up ENV APP_DEBUG to false. It’s important to have opcache turned on.

## Don’t use bcrypt in tests
Don’t generate hashes dynamically before every test case. Just generate the hash once and use a plain string. Hashing algorithms like bcrypt are pretty slow, so you probably don’t need and for sure don’t want to do this before every test to e.g. create a new user.

## Don’t use Doctrine logger and similar loggers
```
doctrine:
  dbal:
    logging: false
```

Think about the loggers that you use, maybe there is some room for improvement by disabling them.

## Distribute executing tests over many jobs
Tests are so varied, that you rather cannot divide them into many sets based on the count. It’s better to divide 
them based on execution time. Some CI systems allow distributing tests between many jobs based on execution time e.g.
: [https://circleci.com/docs/parallelism-faster-jobs/](https://circleci.com/docs/parallelism-faster-jobs/)

However, if your CI system doesn’t support that, you can implement this easily:

1 step:
- generate reports from tests -> in PHPUnit you can do it by adding –log-junit log.xml
- save those reports as artifacts
2 step:
- get reports saved as artifacts
- generate X equal batches based on execution time
  - it could be saved to the file as a regex or as a separate phpunit config with specific files (it is more 
    performant):

```
Tests\ModuleA\DoSomethingTestCase|Tests\ModuleA\DoSomethingTestCase2
```
- run tests with option
```
phpunit --filter="Tests\ModuleA\DoSomethingTestCase|Tests\ModuleA\DoSomethingTestCase2"
```
- where the filter parameter is loaded from the generated file Y

So you need to define two env variables
- how many jobs do you want to have – X
- what batch should be executed in the current job – Y

## Use pcov instead of xdebug to generate code coverage
Turn off xdebug in tests and use pcov instead which is much faster.

```
php vendor/bin/phpunit -dpcov.enabled=1 -dxdebug.mode=off
```
Of course, you need to install pcov extension.

## Use Paratest
If your tests are independent, you can use parallel testing tool – Paratest. There is also the possibility to create many databases with some token and use them during the parallel execution.

[https://github.com/paratestphp/paratest](https://github.com/paratestphp/paratest)

## Speed up building docker image
If you use docker, you should focus on decreasing the size of an image as much as possible. It’s important because once built image is downloaded many times, so it’s helpful to speed up this process by creating lightweight images.

Use the newest version of docker with BuildKit [https://docs.docker.com/build/buildkit/](https://docs.docker.com/build/buildkit/), which is faster than 
previous builders.

Use overlay2 storage driver which is more efficient than other drivers.

## Use up-to-date PHP
Some PHP versions have bugs causing memory leaks. Also, the performance of PHP versions is getting better and better. So it’s quite important to have the highest possible version.

[https://bugs.php.net/bug.php?id=79519](https://bugs.php.net/bug.php?id=79519)

## Set up timeouts
Use a hard timeout to fail the test it takes more than X seconds. In PHPUnit you can define that:

```
--default-time-limit=5 --enforce-time-limit
```

## Monitoring
Monitoring the memory usage and execution time of each test is valuable, helping to identify and address any potential issues.

In PHPUnit you can use the flag:

```
--log-junit log.xml
```

to generate reports containing time and after that, you can parse those files and present these times in your CI tool.


