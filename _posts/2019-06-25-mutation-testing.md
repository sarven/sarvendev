---
title: Mutation testing – we are testing tests
date: 2019-06-25
categories: [Programming, PHP, Testing]
permalink: /2019/06/mutation-testing-we-are-testing-tests/
image:
  path: /assets/img/2019-06-25/featured.png
---
Writing tests should assure us that the code created by us is working correctly. Often we point out the code coverage factor and if we have 100% we can say that implemented solutions are correct. Are you sure? Maybe there is a tool that can give us more accurate feedback?

## Mutation testing
Mutation testing is modifying small pieces of code and checking what impact it has on tests. If after the modification, tests still passed correctly then we have the signal that for this excerpt of code, tests are insufficient.  Of course, everything depends on what the changes are because we don’t want to test every petite modification e.g. indents or variable’s names, because after these changes tests also should pass. Therefore in mutation tests, we are using so-called mutators, which are responsible for change some fragments of code with different ones, but only in the way in which it has sense, but more about this in the further part of the article. These tests sometimes we do ourselves, checking if when we change something in the code it will break tests. If we did refactoring “half of the system” and tests still are green, then we can at once tell that these tests are poor. If someone did something like that and tests were good then congratulations! 😀

## Infection framework
In PHP currently, the most popular framework for mutation testing is [Infection](https://github.com/infection/infection). Currently, it supports PHPUnit and PHPSpec, and to work requires PHP 7.1+ and Xdebug or phpdbg.

## The first execution and configuration
The first execution shows us an interactive configurator of this tool, which ends preparing the specific file with settings infection.json.dist. In an example, which I will present below it looks like this:
```php
{
    "timeout": 10,
    "source": {
        "directories": [
            "src"
        ]
    },
    "logs": {
        "text": "infection.log",
        "perMutator": "per-mutator.md"
    },
    "mutators": {
        "@default": true
    }
}
```

Timeout is an option, which should be an equal maximal time in which a single test should be executed. In source we set directories, from which the code should be mutated, it is possible to add specific excludes. In logs, we have the option text, where we are setting collecting statistics, which are the most interesting for us, just our inaccurate tests. The option perMutator allows us to save mutators which were used. More about this topic you can find in the documentation.

## The practical example
```php
final class Calculator
{
    public function add(int $a, int $b): int
    {
        return $a + $b;
    }
}
```
Let’s say that we have a class like above. We write test in PHPUnit:
```php
final class CalculatorTest extends TestCase
{
    /**
     * @var Calculator
     */
    private $calculator;

    public function setUp(): void
    {
        $this->calculator = new Calculator();
    }

    /**
     * @dataProvider additionProvider
     */
    public function testAdd(int $a, int $b, int $expected): void
    {
        $this->assertEquals($expected, $this->calculator->add($a, $b));
    }

    public function additionProvider(): array
    {
        return [
            [0, 0, 0],
            [6, 4, 10],
            [-1, -2, -3],
            [-2, 2, 0]
        ];
    }
}
```

Of course, this test should be written before the implementation of method add(). Executing ./vendor/bin/phpunit returns:
```
PHPUnit 8.2.2 by Sebastian Bergmann and contributors.
....                                                                4 / 4 (100%)
Time: 39 ms, Memory: 4.00 MB
OK (4 tests, 4 assertions)
```

Now we are executing ./vendor/bin/infection:

```
You are running Infection with Xdebug enabled.
     ____      ____          __  _
    /  _/___  / __/__  _____/ /_(_)___  ____
    / // __ \/ /_/ _ \/ ___/ __/ / __ \/ __ \
  _/ // / / / __/  __/ /__/ /_/ / /_/ / / / /
 /___/_/ /_/_/  \___/\___/\__/_/\____/_/ /_/

Running initial test suite...

PHPUnit version: 8.2.2

    9 [============================]  1 sec

Generate mutants...

Processing source code files: 1/1Creating mutated files and processes: 0/2
Creating mutated files and processes: 2/2
.: killed, M: escaped, S: uncovered, E: fatal error, T: timed out

..                                                   (2 / 2)

2 mutations were generated:
       2 mutants were killed
       0 mutants were not covered by tests
       0 covered mutants were not detected
       0 errors were encountered
       0 time outs were encountered

Metrics:
         Mutation Score Indicator (MSI): 100%
         Mutation Code Coverage: 100%
         Covered Code MSI: 100%

Please note that some mutants will inevitably be harmless (i.e. false positives).

Time: 1s. Memory: 10.00MB
```

So our tests according to Infection are accurate. In the file per-mutator.md we can check what mutations were used:

```
# Effects per Mutator

| Mutator | Mutations | Killed | Escaped | Errors | Timed Out | MSI | Covered MSI |
| ------- | --------- | ------ | ------- |------- | --------- | --- | ----------- |
| Plus | 1 | 1 | 0 | 0 | 0 | 100| 100|
| PublicVisibility | 1 | 1 | 0 | 0 | 0 | 100| 100|
```
Mutator Plus is just change sign plus to minus, it should break tests, and mutator PublicVisibility is changing access modifier of the method, what also should break tests and it works in this case.

So now we add  a little more complicated method.

```php
/**
 * @param int[] $numbers
 */
public function findGreaterThan(array $numbers, int $threshold): array
{
    return \array_values(\array_filter($numbers, static function (int $number) use ($threshold) {
        return $number > $threshold;
    }));
}
```

```php
/**
 * @dataProvider findGreaterThanProvider
 */
public function testFindGreaterThan(array $numbers, int $threshold, array $expected): void
{
    $this->assertEquals($expected, $this->calculator->findGreaterThan($numbers, $threshold));
}

public function findGreaterThanProvider(): array
{
    return [
        [[1, 2, 3], -1, [1, 2, 3]],
        [[-2, -3, -4], 0, []]
    ];
}
```

At this time after execution we see:
```
You are running Infection with Xdebug enabled.
     ____      ____          __  _
    /  _/___  / __/__  _____/ /_(_)___  ____
    / // __ \/ /_/ _ \/ ___/ __/ / __ \/ __ \
  _/ // / / / __/  __/ /__/ /_/ / /_/ / / / /
 /___/_/ /_/_/  \___/\___/\__/_/\____/_/ /_/

Running initial test suite...

PHPUnit version: 8.2.2

   11 [============================] < 1 sec

Generate mutants...

Processing source code files: 1/1Creating mutated files and processes: 0/7
Creating mutated files and processes: 7/7
.: killed, M: escaped, S: uncovered, E: fatal error, T: timed out

..M..M.                                              (7 / 7)

7 mutations were generated:
       5 mutants were killed
       0 mutants were not covered by tests
       2 covered mutants were not detected
       0 errors were encountered
       0 time outs were encountered

Metrics:
         Mutation Score Indicator (MSI): 71%
         Mutation Code Coverage: 100%
         Covered Code MSI: 71%

Please note that some mutants will inevitably be harmless (i.e. false positives).

Time: 1s. Memory: 10.00MB
```
So now is something bad with our tests, firstly we check the file infection.log:

```
Escaped mutants:
================


1) /home/sarven/projects/infection-playground/infection-playground/src/Calculator.php:19    [M] UnwrapArrayValues

--- Original
+++ New
@@ @@
      */
     public function findGreaterThan(array $numbers, int $threshold) : array
     {
-        return \array_values(\array_filter($numbers, static function (int $number) use($threshold) {
+        return \array_filter($numbers, static function (int $number) use($threshold) {
             return $number > $threshold;
-        }));
+        });
     }

2) /home/sarven/projects/infection-playground/infection-playground/src/Calculator.php:20    [M] GreaterThan

--- Original
+++ New
@@ @@
     public function findGreaterThan(array $numbers, int $threshold) : array
     {
         return \array_values(\array_filter($numbers, static function (int $number) use($threshold) {
-            return $number > $threshold;
+            return $number >= $threshold;
         }));
     }

Timed Out mutants:
==================

Not Covered mutants:
====================
```

The first problem, which wasn’t caught is the usage of function array_values, it’s used in this place to reset keys because of array_filter returns values with keys from the previous array. In these tests also we don’t have the case, which requires the usage of array_values because in another way returns an array with the same values, but other keys.

The second problem instead concerns the edge cases. We used the sign > in comparison, but we don’t test any edge case, so changing the sign to >= doesn’t cause breaking the tests. We need to add just one test:
```php
public function findGreaterThanProvider(): array
{
    return [
        [[1, 2, 3], -1, [1, 2, 3]],
        [[-2, -3, -4], 0, []],
        [[4, 5, 6], 4, [5, 6]]
    ];
}
```

And now infection doesn’t show any objections:
```
You are running Infection with Xdebug enabled.
     ____      ____          __  _
    /  _/___  / __/__  _____/ /_(_)___  ____
    / // __ \/ /_/ _ \/ ___/ __/ / __ \/ __ \
  _/ // / / / __/  __/ /__/ /_/ / /_/ / / / /
 /___/_/ /_/_/  \___/\___/\__/_/\____/_/ /_/

Running initial test suite...

PHPUnit version: 8.2.2

   12 [============================] < 1 sec

Generate mutants...

Processing source code files: 1/1Creating mutated files and processes: 0/7
Creating mutated files and processes: 7/7
.: killed, M: escaped, S: uncovered, E: fatal error, T: timed out

.......                                              (7 / 7)

7 mutations were generated:
       7 mutants were killed
       0 mutants were not covered by tests
       0 covered mutants were not detected
       0 errors were encountered
       0 time outs were encountered

Metrics:
         Mutation Score Indicator (MSI): 100%
         Mutation Code Coverage: 100%
         Covered Code MSI: 100%

Please note that some mutants will inevitably be harmless (i.e. false positives).

Time: 1s. Memory: 10.00MB
```
Let’s add one method subtract to class Calculator, but at this time without a specific test in PHPUnit:
```php
public function subtract(int $a, int $b): int
{
    return $a - $b;
}
```
And after infection execution we can see:
```
You are running Infection with Xdebug enabled.
     ____      ____          __  _
    /  _/___  / __/__  _____/ /_(_)___  ____
    / // __ \/ /_/ _ \/ ___/ __/ / __ \/ __ \
  _/ // / / / __/  __/ /__/ /_/ / /_/ / / / /
 /___/_/ /_/_/  \___/\___/\__/_/\____/_/ /_/

Running initial test suite...

PHPUnit version: 8.2.2

   11 [============================] < 1 sec

Generate mutants...

Processing source code files: 1/1Creating mutated files and processes: 0/9
Creating mutated files and processes: 9/9
.: killed, M: escaped, S: uncovered, E: fatal error, T: timed out

.......SS                                            (9 / 9)

9 mutations were generated:
       7 mutants were killed
       2 mutants were not covered by tests
       0 covered mutants were not detected
       0 errors were encountered
       0 time outs were encountered

Metrics:
         Mutation Score Indicator (MSI): 77%
         Mutation Code Coverage: 77%
         Covered Code MSI: 100%

Please note that some mutants will inevitably be harmless (i.e. false positives).

Time: 1s. Memory: 10.00MB
```
So at this time, the tool returns two uncovered mutations.
```
Escaped mutants:
================

Timed Out mutants:
==================

Not Covered mutants:
====================


1) /home/sarven/projects/infection-playground/infection-playground/src/Calculator.php:24    [M] PublicVisibility

--- Original
+++ New
@@ @@
             return $number > $threshold;
         }));
     }
-    public function subtract(int $a, int $b) : int
+    protected function subtract(int $a, int $b) : int
     {
         return $a - $b;
     }
 }

2) /home/sarven/projects/infection-playground/infection-playground/src/Calculator.php:26    [M] Minus

--- Original
+++ New
@@ @@
     }
     public function subtract(int $a, int $b) : int
     {
-        return $a - $b;
+        return $a + $b;
     }
 }
```

## Metrics
The tool after every execution returns three metrics:
```
Metrics:
    Mutation Score Indicator (MSI): 47%
    Mutation Code Coverage: 67%
    Covered Code MSI: 70%
```
**Mutation Score Indicator** – the percentage value of mutations detected by tests

It’s calculated in the following way:
```
TotalDefeatedMutants = KilledCount + TimedOutCount + ErrorCount;

MSI = (TotalDefeatedMutants / TotalMutantsCount) * 100;
```

**Mutation Code Coverage** – the percentage value of the code covered by mutations

It’s calculated in the following way:
```
TotalCoveredByTestsMutants = TotalMutantsCount - NotCoveredByTestsCount;

CoveredRate = (TotalCoveredByTestsMutants / TotalMutantsCount) * 100;
```

**Covered Code Mutation Score Indicator** – it determines the efficiency of tests only for the code, which is covered by tests

It’s calculated in the following way:

```
TotalCoveredByTestsMutants = TotalMutantsCount - NotCoveredByTestsCount;
TotalDefeatedMutants = KilledCount + TimedOutCount + ErrorCount;

CoveredCodeMSI = (TotalDefeatedMutants / TotalCoveredByTestsMutants) * 100; 
```

## Usage in a more complicated project
In the above example, we have only one class, so we could execute infection without any parameters. However in daily work in a normal project, useful option will be parameter –filter, which allows set a file, for which we want to check mutations.

```
./vendor/bin/infection --filter=Calculator.php
```

## False positives
Some mutations don’t change the working of the code and in fact, infection returns a lower MSI than 100%, but not always we will be able to do something with it, so we have to accept those situations. We can see something like that in the following example:

```php
public function calcNumber(int $a): int
{
    return $a / $this->getRatio();
}

private function getRatio(): int
{
    return 1;
}
```

Of course in this case the method getRatio is meaningless, in the usual project there would probably be some calculation here, but it might as well have result also 1. Infection returns:

```
Escaped mutants:
================


1) /home/sarven/projects/infection-playground/infection-playground/src/Calculator.php:26    [M] Division

--- Original
+++ New
@@ @@
     }
     public function calcNumber(int $a) : int
     {
-        return $a / $this->getRatio();
+        return $a * $this->getRatio();
     }
     private function getRatio() : int
     {
```
As we know, multiplying and dividing by 1 returns the same result equal 1. So this mutations shouldn’t break tests, so despite that infection has objections about the accuracy of our tests, it’s all right.

## Optimizations for big projects
In big projects execution of infection can be very time-consuming. It is possible to optimize execution during CI only for changed files. More about this topic you can find in the documentation: https://infection.github.io/guide/how-to.html

Additionally, it is possible to execute tests for the mutated code in parallel. However, this option is possible only when every single test is independent. Good tests should just be. To enable this option you should use the parameter –threads:
```
./vendor/bin/infection --threads=4
```

## How it works?

Framework Infection uses AST (Abstract Syntax Tree), which is the representation of the code using an abstract data structure. There is using the parser written by one person developing PHP ([php-parser](https://github.com/nikic/PHP-Parser)).

The working of this tool we can present in simplify:
1. Generate AST from the code
2. Apply appropriate mutators (list of all you can find here)
3. Create a mutated code from AST 
4. 4.Execute tests for a mutated code

For example, we can check the mutator changing plus to minus:
```php
<?php

declare(strict_types=1);

namespace Infection\Mutator\Arithmetic;

use Infection\Mutator\Util\Mutator;
use PhpParser\Node;
use PhpParser\Node\Expr\Array_;

/**
 * @internal
 */
final class Plus extends Mutator
{
    /**
     * Replaces "+" with "-"
     *
     * @param Node&Node\Expr\BinaryOp\Plus $node
     *
     * @return Node\Expr\BinaryOp\Minus
     */
    public function mutate(Node $node)
    {
        return new Node\Expr\BinaryOp\Minus($node->left, $node->right, $node->getAttributes());
    }

    protected function mutatesNode(Node $node): bool
    {
        if (!($node instanceof Node\Expr\BinaryOp\Plus)) {
            return false;
        }

        if ($node->left instanceof Array_ || $node->right instanceof Array_) {
            return false;
        }

        return true;
    }
}
```
The method mutate() creates a new element, which should be replaced with a plus. The class Node comes from the package php-parser, which is using to operations on AST and modifying the PHP code. However that change can’t be executed in every place, therefore the method mutatesNode() contains additional conditions. If on the left side of plus is an array or on the right side of minus is an array, then the change is not possible. This condition is required because of the code:

```php
$tab = [0] + [1];
```
is correct, but the following one isn’t correct.
```php
$tab = [0] - [1];
```

## Summary

Mutation testing is a very good tool complementary CI process giving information about tests quality. The green bar in the tests don’t make us sure that everything is well written, testing tests or mutation testing allows increase the accuracy of tests, so in the effect should increase confidence, that we provide working solutions. Of course, as we know aspiration to have 100% in metrics isn’t required, because it isn’t possible always. We should analyze logs and adjust tests accordingly.
