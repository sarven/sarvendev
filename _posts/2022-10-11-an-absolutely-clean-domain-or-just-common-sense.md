---
title: An absolutely clean domain or just common sense
date: 2022-10-11
categories: [Programming, DDD, PHP]
permalink: /2022/10/an-absolutely-clean-domain-or-just-common-sense/
---
Nowadays, a concept like DDD is widely known and used by many programmers. Curious programmers read a lot about those practices in books written by Evans or Vernon or maybe have knowledge from conferences or blogs. As I saw many times, people are trying to be too much strict with these practices. Trying to make a domain completely clean is of course highly desired, but if you have a not very complex domain, and to have a completely clean domain you need to highly complicate the code around it (probably Infrastructure’s code) something is wrong, don’t you think? So you need to think about the return on investment. Is it worth having more work to do, and complex infrastructure code to just make your domain completely clean? In most cases probably it isn’t at all, only in projects with a highly complicated domain it will be necessary to define a core domain, and then perhaps making this core domain completely clean will make sense. However, not every place in your project will be perfect, you should invest your time only in the most important places. So in this article, I would like to write about using ORMs when approaching DDD which are often hated by many people.

## Over-engineering in most of cases
### Mapping manually

One of many approaches to have a completely separate domain model from the ORMs tools is using an own mapper. So the flow is following:
- DDD Entity
  - Domain Repository
    - Mapper
      - Entity ↔ arrays representing records in the database
    - Save / Get using a plain SQL e.g. DBAL

Pros:
- A completely clean domain model

Cons:
- A lot of work to do
- DDD models must have access methods to data (perhaps method toArray or sth like that)
  - or we need to use the reflection to access the data

### Using ORM, but not use the full potential
I saw also a little bit strange approach by doing the same, but using also ORM. In that solution, we have two types of entities, DDD Entity and ORM Entity.

The flow is following:
- DDD Entity
  - Domain Repository
    - Mapper
      - DDD Entity ↔ ORM Entity
  - Save / Get using ORM Repositories

In this case, ORM Entity is just an anemic model with getters and setters. So we have separate DDD models and then transform them into ORM entities to save them in the database using ORM’s features.

Pros:
- A completely clean domain model

Cons:
- A lot of work to do
- DDD models must have access methods to data (perhaps method toArray or sth like that)
- or we need to use the reflection to access the data
- A lot of additional complexity like ORM models
- we added ORM to our project, but we don’t use the full potential

These two approaches allow us to have a pure domain without any overhead given by ORM tools. However, in my opinion, only the first approach is worth considering. Translating between DDD models and ORM models is ridiculous. Using such an advanced tool as ORM that way isn’t valuable. ORMs were created to map complex models to the database, but in this case, we have complex DDD models and then map them to simple anemic ORM entities, so it would be much simpler to do it using the first approach. Certainly, if you have any reasonable argument to use the second approach please feel free to write a comment 😉

## Pragmatic approach
Nevertheless, in this article, I want to encourage you to just think in a pragmatic way. In most projects/modules we don’t implement a very complex domain. Trying to implement a pure domain without any overhead given by other tools takes more time. So please start with calculating the return on investment. Perhaps it doesn’t worth it.

## How to implement DDD models using popular ORM in PHP – Doctrine?
### Using Collections
Undoubtedly the first objection to using ORM will be Collections, whose in Doctrine must become a part of our domain model.
  
```php
class Order
{
    /**
     * @var Collection<OrderLine>
     */
    private Collection $lines;
    
    public function __construct()
    {
        $this->lines = new ArrayCollection();
    }
    
    public function addOrderLine(OrderLine $line): void
    {
        $this->lines->add($line);
    }
}
```

Naturally, it isn’t desirable to make a domain depending on any external package. However, doctrine-collections is separate from the ORM package https://github.com/doctrine/collections . Furthermore, other less popular ORM e.g. CycleORM also allows the use of this package in models → https://cycle-orm.dev/docs/relation-collections/2.x/en . Interestingly, in CycleORM we can change the implementation of collections.

### Mapping
To use ORM, we need to prepare mapping objects onto the database. The recommended way is to keep the mapping separate from our model. So in Doctrine, we can use XML or PHP. In my opinion, this point isn’t a big deal and if you right now have a code with mapping using annotations changing that won’t be worthwhile. Changing a framework to another one is a very rare situation, and if you have annotations right now it’s ok, probably that code has bigger problems to solve than something so trivial. Perhaps, preparing the possibility of using separate mapping with the emergence of new code would be a good idea.

### Implementing ValueObjects
#### Custom types
Doctrine gives us the possibility to define custom types.

https://www.doctrine-project.org/projects/doctrine-orm/en/2.13/cookbook/custom-mapping-types.html

However, it’s required to implement a bidirectional conversion between our model and the database.

```php
<?php

final class EndDate implements Stringable
{
    private const FORMAT = 'Y-m-d';
    private readonly DateTimeImmutable $dateTime;

    final private function __construct(DateTimeImmutable $dateTime)
    {
        $this->dateTime = $dateTime->setTime(0, 0, 0, 0);
    }

    public function __toString(): string
    {
        return $this->dateTime->format(self::FORMAT);
    }

    public static function fromDateTimeImmutable(DateTimeImmutable $dateTimeImmutable): self
    {
        return new self($dateTimeImmutable);
    }

    public function toNative(): DateTimeImmutable
    {
        return $this->dateTime;
    }
}
```
```php
<?php

use Doctrine\DBAL\Platforms\AbstractPlatform;
use Doctrine\DBAL\Types\ConversionException;
use Doctrine\DBAL\Types\DateImmutableType;

final class EndDateType extends DateImmutableType
{
    public function convertToDatabaseValue($value, AbstractPlatform $platform): string|null
    {
        if (null === $value) {
            return null;
        }

        if ($value instanceof EndDate) {
            return $value->toNative()->format($platform->getDateFormatString());
        }

        throw ConversionException::conversionFailedInvalidType($value, $this->getName(), ['null', EndDate::class]);
    }

    public function convertToPHPValue($value, AbstractPlatform $platform): EndDate|null
    {
        if (null === $value || $value instanceof EndDate) {
            return $value;
        }

        try {
            if (false === is_string($value)) {
                throw new Exception(sprintf('A value should be string, %s given', gettype($value)));
            }

            return EndDate::fromString($value);
        } catch (Exception $e) {
            throw ConversionException::conversionFailedFormat(strval($value), $this->getName(), $platform->getDateTimeFormatString(), $e);
        }
    }
}
```

```yaml
doctrine:
    dbal:
        url: '%env(resolve:DATABASE_URL)%'
        types:
            end_date: App\SomeModule\Infrastructure\Doctrine\EndDateType
```

That will be a good solution unless we have a value object with more than one property. Of course, it’s possible to flat those properties and write them into one column, but it doesn’t seem to be a proper way.

#### Embeddables
Nevertheless, Doctrine gives us a second option to handle VOs.

https://www.doctrine-project.org/projects/doctrine-orm/en/2.13/tutorials/embeddables.html

It will be a good fit for VOs with more than one property, but also one property it’s not a problem in this case.

```php
<?php

final class Price
{
    public function __construct(
        private readonly int $amount,
        private readonly string $currency,
    ) {
    }
}
```
```xml
<entity name="App\SomeModule\Domain\Entity">
    <embedded name="price" class="App\Shared\Domain\ValueObject\Price" />
</entity>
```
It’s a lot simpler than custom types. We don’t have to implement a whole conversion process. However, there’s one drawback we need to remember – it isn’t possible to create nullable Embeddable.

```php
<?php

final class Order
{
    private Price|null $price; // nullable embeddables currently are not supported in Doctrine
}
```

Perhaps, instead of a null, a default value will be good enough or even better, but it highly depends on the context we have, sometimes a default value doesn’t make any sense, and an old good null seems the best idea.As default value, I mean something like the following:

```php
<?php

final class Price
{
    private const DEFAULT_CURRENCY = 'USD';
    
    public function __construct(
        private readonly int $amount,
        private readonly string $currency,
    ) {
    }
    
    public static function zero(): self
    {
        return new self(0, self::DEFAULT_CURRENCY);
    }
}
```

#### Collections of ValueObjects
Value objects have no identity, so when we have a model where a collection of value objects is required then will be a problem to store them in the database. Although, to do that we have two solutions:
- save as JSON when the size of the collection will be small
- add an id to the Value Object and define relations as between usual ORM entities

From the perspective of our domain, it will be still a Value object, but we have to assure to have a proper encapsulation and access them only through a proper aggregate.

## Be careful with lazy loading and aggregates, because it loads at the different point in time
An aggregate should be a consistent unit that is protecting its invariants. Invariants are just business rules. Therefore, it’s crucial to get all data contained by the aggregate from the database at once. In Doctrine as a default, all relations will be fetched lazy, which means that we operate on a proxy, and when we call some properties, ORM will fetch the required data from the database. So to achieve data integrity we should change fetch mode to eager to load all data at once, and then we will have a certainty that data in the aggregate are from one point in time.

## References to other aggregates
In DDD we should make references from one aggregate to another aggregate only by using unique identifiers. I heard many times opinions that Doctrine requires a whole object, but it’s not true. You can just use a value object which represents a unique identifier of an aggregate e.g. CustomerId. Therefore, there is a little issue with that. When we will want to generate automatically DDL migrations by Doctrine, there won’t be generated foreign key constraints, because Doctrine doesn’t have any knowledge about this relation. It’s just a simple VO. Of course, it’s possible to add them by writing manually the right definition.

However, most likely we don’t even need foreign keys in that case. An aggregate is a unit of consistency, so everything outside is irrelevant from its point of view. Other aggregates can be even in a separate application – e.g. microservices architecture, also when we have a modular monolith architecture we should design that system with the thought that an aggregate from other modules can be placed in the future in another application.

## Final as a comment, but verify whether it is always required
Declaring classes as final is a good practice, but when using Doctrine is impossible to declare classes as final if you want to use lazy loading, because Doctrine generates proxies whose extend entities. Perhaps you even shouldn’t use lazy loading in most of the places (see a few paragraphs above – lazy loading and aggregates). However, if you know the drawbacks and still want to use lazy loading, it’s possible to just declare classes as final using a comment, and just rely on static code analysis.
```php
<?php

/**
 * @final
 */
class Order
{
}
```

## Automate architecture checks with Deptrac
Let’s assume that we’re using Doctrine to save our models, so we allow to use of external dependencies in our domain. Certainly, it can be a difficult problem to manually check what our domain has dependencies, so that new things are not easily added to it. It always should be a well-considered decision. The best we can do is define rules with the structure of the whole application – layers and modules, and then just add some exceptions like doctrine/collections. Then it will be much easier to keep an eye on the situation. The following example presents that concept using Deptrac.

```yaml
parameters:
  paths:
    - ./src
  exclude_files:
  layers:
    - name: Domain
      collectors:
        - type: className
          regex: ^App\\.*\\Domain\\.*

    - name: Application
      collectors:
        - type: className
          regex: ^App\\.*\\Application\\.*

    - name: Infrastructure
      collectors:
        - type: className
          regex: ^App\\.*\\Infrastructure\\.*

    - name: UI
      collectors:
        - type: className
          regex: ^App\\UI\\.*

    - name: Vendor
      collectors:
        - type: bool
          must:
            - type: className
              regex: .+\\.*
          must_not:
            - type: className
              regex: ^App\\.*
            - type: className
              regex: Doctrine\\Common\\Collections\\.*

  ruleset:
    Domain:
    Application:
      - Domain
    Infrastructure:
      - Domain
      - Application
      - Vendor
    UI:
      - Application
      - Infrastructure
      - Domain
      - Vendor
```

## Summary
There is no silver bullet. Unfortunately, the best solution doesn’t exist. It always depends on the context. So we need to decide carefully, try to think about pros and cons, what our decision gives us and what an additional problem we introduce. A good rule of thumb will be evaluating the level of complexity in our subdomains and also what is our core, supporting, and generic domain. If it is a highly complex subdomain, maybe the overhead of ORMs tools will have a negative impact on this solution and it will be worth not using ORM at all, but perhaps only in this sensitive part of the system. Then we will have more work to do during the implementation but we gain the flexibility to implement models as we want, but maybe in quite simple domains, maybe also supporting or generic it will be just good enough to use ORM and don’t think too much about flexibility, because it’s a less significant part of the system.
