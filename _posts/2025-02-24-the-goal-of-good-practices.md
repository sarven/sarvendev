---
title: The goal of good practices
date: 2025-02-15
categories: [Programming, Good practices]
---
Recently I had a discussion about violating of some SOLID principles, which made me think about the goal of these practices, and other good practices in general.
It forced me to think about it, because we weren't talking about consequences of these violations, but about the rules themselves.
The goal of good practices isn’t just to follow steps or avoid certain patterns;
it's to create code that is clear, maintainable, and adaptable to change. If we focus solely on whether something "breaks a rule,"
we might miss opportunities to consider whether it’s serving the actual goals that matter.

# Goal
We have a lot of good practices, like SOLID principles, GRASP, CUPID, DRY, KISS, YAGNI, and other approaches like DDD etc. We often hear that we should follow them,
and for example that ActiveRecord pattern is bad, because it violates SRP, or that we should avoid using `if` statements in some places, because it violates OCP.
However, while these principles can be helpful, they are not the end goal in themselves. The purpose is different, we need to understand the problem,
we need to gather all requirements, and then we can choose appropriate solutions to meet the requirements. We should use these principles as a tool
to achieve our goals, not as a goal itself. Maybe we can achieve the same goal differently, and it will be better for our case. It will be simpler, faster and easier to do. Maybe we can even achieve
the same goal without even writing a line of code, if we can use some external service, or some other tool.

Principles are tools that guide us toward these goals. However following them without understanding the context
can sometimes lead to overengineering or unnecessary complexity, especially if a project’s scope or constraints
don’t justify their full application. I think that **KISS - Keep it simple stupid**, **YAGNI - You
aren't gonna need it**, are the most important principles.

For example, while SRP violations in the ActiveRecord pattern can lead to tighter coupling, the simplicity and rapid
development it offers can be perfectly appropriate in smaller projects where speed is a higher priority than flexibility. Good practice in programming is ultimately about
making informed, pragmatic choices that balance short- and long-term needs without being constrained by strict adherence to rules at the expense of the solution.

# Let's consider some examples

## SRP - Single Responsibility Principle

> A class should have only one reason to change.

In this principle, we want to achieve separation of concerns, by dividing the code into smaller parts, which are
responsible for only one thing. For example, what if I have a request that accepts only one enum, and I want to
validate it?

```php
#[Route('/taxes')]
public function getTaxes(Request $request): Response
{
    $country = $request->get('country');
    $country = Country::tryFrom($country);

    if (null === $country) {
        return new JsonResponse(['error' => 'Invalid country'], Response::HTTP_BAD_REQUEST);
    }
    
    try {
        $taxes = $this->taxProvider->getTaxes($country);
    } catch (CouldNotRetrieveTaxException $e) {
        $this->logger->error('Could not retrieve taxes', ['exception' => $e]);

        return new JsonResponse(['error' => 'Could not retrieve taxes'], Response::HTTP_UNPROCESSABLE_ENTITY);
    }
    
    return new JsonResponse($taxes);
}
```

Is it a violation of SRP? I don't think so, but I heard voices that it is. I think that it is a good place to
validate the request, especially if it is a simple validation. If it is more complex, then we can extract it to
a separate class.

What about exception handling? Should we extract it to a separate class? Maybe yes, maybe not, we could have some
abstraction to handle all exceptions and map them to proper http codes.

There is one advantage of having it in the controller, we can see all the logic in one place, and we can easily see what is happening. If we extract it to a
separate place we need to jump between files to see what is happening, but someone could say that it is only
the right solution to not violate SRP. For me, it is a trade-off, and we need to consider what is more important in our
case, do we have a complex validation, complex error handling, or do we want to test validation thoroughly by unit tests
then of course it's worth putting it in a separate place.

## OCP - Open/Closed Principle

> Open for extension, closed for modification.

In this principle, we want to achieve that we can extend our code without changing the existing code. Let's consider
another example:

```php
final readonly class TaxProvider implements TaxProviderInterface
{
    public function __construct(
        private TaxProviderInterface $aProvider,
        private TaxProviderInterface $bProvider,
    ) {}

    public function getTaxes(Country $country): Taxes
    {
        return match ($location->country) {
            Country::POLAND => $this->bProvider->getTaxes($country),
            default => $this->aProvider->getTaxes($country),
        };
    }
}
```

So we have two providers, and we need to use one of them for Poland, and the other for the rest of the countries. I
chose Poland here, because once we had a change of taxes three times in year, so it is a good example of a place
where we need a different provider.

Ok, now we have a new requirement, we need to add a new provider for Germany. What should we do? We need to change
the main TaxProvider class, and add a new case in the match statement. Is it a violation of OCP? I don't think so,
because we can add a new provider, and each provider is separated from the others, so we don't need to change them,
we only change one central place. And the same is true for example with using abstractions hidden behind interfaces,
in some places you need to create a concrete implementation (e.g. Factory pattern).

Ok, someone would say that we could use tagged services, and then we don't need to change the main class. It is
partially true, we don't change the main class, but we also need to change the central place - the configuration.

```yaml
services:
    App\Infrastructure\TaxProvider\AProvider:
        tags:
            - { name: 'app.tax_provider', country: 'default' }

    App\Infrastructure\TaxProvider\BProvider:
        tags:
            - { name: 'app.tax_provider', country: 'poland' }
```

```php
    /** @var iterable<TaxProviderInterface> */
    private iterable $providers;

    public function __construct(
        #[TaggedIterator('app.tax_provider')]
        iterable $providers
    ) {
        $this->providers = $providers;
    }
```

What if we complicate the requirements a little bit, and we will need to add a new provider for Germany, but used
only at the beginning of the next year? Let's say that the previously used provider supports only taxes until 2024
for Germany, but at the beginning of 2025 we need to start smoothly using the new provider. We need to change the
central place again, and we need to add some logic to check the current date, and then choose the proper provider.

So in this case, it is better to have it in the main class, because we can see all the logic in one
place, and be able to easily adapt it to new requirements.

I remember a situation when I used tagged services, because I had two handlers and expected to have more in the future. And I needed to refactor
the configuration file, and made a mistake, which caused a strange bug. It's described [here](https://sarvendev.com/2023/03/laravel-variadic-parameter-trap/).
So my code was ready for easily adding new handlers, and guess, how many providers I added in the next two years? Just one, so a simpler approach would be totally
fine.

## DIP - Dependency Inversion Principle

> High-level modules should not import anything from low-level modules.
> Abstractions should not depend on details. Details (concrete implementations) should depend on abstractions.

```php

use App\Core\Port\ReadRepositoryInterface;
use App\Infrastructure\DbalReadRepository;
use App\Infrastructure\Cache\CacheInterface;

final class CachedProductReadRepository implements ProductReadRepository
{
    public function __construct(
        private DbalProductReadRepository $repository,
        private CacheInterface $cache
    ) {}

    public function get(int $id): Product
    {
        // ...
    }
}
```

One can say that here we have a violation of DIP, because we are using a concrete implementation of the repository. However, is that really a problem?
The cached repository is in the same layer as the dbal repository, so it is not a violation of DIP. Our Core layer is on a higher level,
and it is not dependent on the Infrastructure layer, because the implementation details are hidden behind the interface (ProductReadRepository). This
is the most important thing, because then we have a proper level of encapsulation, maintainability, and testability. In the case of the cached repository, it could
be a problem if we would like to test it by unit tests. Usually, I don't see this as a problem, because it's covered by integration tests.

## What's really important?

I know that it is a very controversial topic, and everyone has their own opinion, and maybe you are just thinking what is the point of this article?
I decided to write about it, because I often saw discussions about such rules during code review, and other discussions about basic things.
The point is that we should focus on the goal, and not on the rules. We should use the rules as a tool to achieve the goal, and not as a goal itself.

Firstly, we should automate what we can, and focus the discussion on what is the hard part, and can't be automated. So things like code formatting, types,
linting, and other things that can be automated should be automated. Then we can focus on more important things.
So if those basic things are automated, there are discussions like the above e.g. SOLID. I presented those examples to emphasize the fact that
we have also some hierarchy of things that we should focus on. It is easy to talk a lot about what is single responsibility, and what is not, but it is not so important,
and sometimes it is treated as a goal itself, that we need to have this code perfect, but it is hard to define what is perfect, so we like to debate about it.

Just stop doing it, it is not an art. Just focus on the goal. The goal is to solve a problem, maybe it is even possible to solve it without writing a line of code,
if so, just do it.

If the problem is not clear yet, maybe you need to prepare some proof of concept to validate an idea, don't focus on the rules, just do it as fast as you can, and then you can refactor it.
The best software is working software, and not the one that is perfect, but not working.

Of course, if you have a clear problem, gathered requirements, then you can choose "the best" solution. Ok, so now we can focus on SOLID and other low-level principles?
Of course not, we should keep in mind the hierarchy of rules.

#### KISS & YAGNI

Keep it simple stupid, and you aren't gonna need it. I think that is the most important. When you have requirements, you can choose what is the best solution,
and if you can achieve the goal in a simple way, just do it.

#### Global complexity

> The art of programming is the art of organizing complexity.   
> Edsger W. Dijkstra

When we talk about complexity in software, it’s easy to get bogged down in the details - 
perfecting a single line of code or debating the finer points of style guides. 
But the real challenge, and opportunity, lies in managing global complexity.

Global complexity refers to the overall structure of your system:

- How well are your modules organized?
- Are the boundaries between them clear and effective?
- How cleanly do they communicate with each other? 

The better we handle global complexity, the easier it becomes to scale, 
maintain, and evolve our systems. Local complexity, like a single function or algorithm, matters, 
but only after the big picture is sound. Without a clear discussion about the overall architecture, 
diving into local details is like arguing over furniture placement in a house without a solid foundation.

At the level of global complexity, usually, we have a lot of one-way door decisions, 
which are hard to revert, and we should mostly focus on them. 

#### Local complexity

When we have a good architecture, we can focus a bit on local complexity, and discuss principles on a lower level. We should focus mostly on the core domain,
on places that change often, because complexity in these places can grow rapidly. Other places are less important, and we can accept some imperfections.
We should have automated as many rules as we can (static analysis, formatting, architecture tests etc.) to achieve some level of consistency in the project. However,
it is also important to accept some personal choices, and imperfection, keeping in mind that global complexity is more important.

To achieve flexibility to change the code easily, we need to have good quality of tests, tests that are decoupled from the implementation details, 
and tests that give us confidence that we can change the code without breaking anything. 

#### Imperfection
Focus on the goal, deliver value to the users, and accept the fact that not everything will be perfect. We can't achieve perfection, and we shouldn't focus on it.
Focus more on the global complexity, and your core domain, and accept more trade-offs in other places. Keep in mind that on the market there are many solutions
using different approaches, and they are successful. There are for sure projects using ActiveRecord pattern violating SRP principle,
and they are earning a lot of money.

# Summary
Adhering to good practices we should focus on achieving goals rather than treating the process as an art form.
The purpose of rules matters more than strict adherence to them, with a clear hierarchy of principles - e.g.
KISS outweighs SOLID. Know when to make conscious shortcuts, document critical decisions, and iterate for improvement when needed.

In teams, especially among seniors, the focus should be on the overall solution rather than minor details.
Automate wherever possible, accept some imperfections and personal coding styles, and avoid over-debating trivial
matters like naming conventions. Not every part of the codebase will be perfect, and that’s okay. Instead, prioritize solving
the problem according to the collected business requirements and start by building a solid foundation to track global complexity,
then move on to local complexity.
