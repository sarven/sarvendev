---
title: Rethinking Mocking - DIY Approach vs. Frameworks on examples in PHP and Typescript
date: 2024-04-23
categories: [Programming, Testing, PHP, Typescript]
permalink: /2024/04/rethinking-mocking-diy-approach-vs-frameworks-on-examples-in-php-and-typescript/
image:
  path: /assets/img/2024-04-23/featured.png
---
In the landscape of software testing, the choice between a do-it-yourself (DIY) approach to mocking or utilizing mocking frameworks is a pivotal decision for programmers. While mocking is indispensable for code reliability, its overuse or incorrect implementation can introduce complexities and fragilities within test suites. This article navigates the balance between leveraging mocks effectively and avoiding common pitfalls, including the dangers of writing mocks in a suboptimal manner.

## Test doubles
![Test doubles](/assets/img/2024-04-23/test-doubles.png)

Test doubles are stand-in objects used during software testing to mimic the behavior of real objects. They help isolate the code being tested by replacing dependencies such as collaborators, external services, or complex components. Mocking is used for:
- Isolation – allows testing components in isolation
- Speed – replacing slow or external dependencies allows running tests faster and getting feedback faster

There are two methods of mocking:
- DIY mocks  – written by ourselves using standard language code
- mocks created by frameworks – e.g. Mockery, or those delivered by testing frameworks like Jest or Phpunit

## Dummy
A dummy is a just simple implementation that does nothing.

```php
final readonly class Mailer implements MailerInterface
{
    public function send(Message $message): void
    {
    }
}
```

## Fake
A fake is a simplified implementation to simulate the original behavior.

```php
final class InMemorySubscriptionRepository implements SubscriptionRepository
{
    private array $subscriptions = [];

    public function get(SubscriptionId $id): Subscription
    {
        if (!isset($this->subscriptions[$id->toString()])) {
            throw new SubscriptionNotFound();
        }

        return $this->subscriptions[$id->toString()];
    }

    public function save(Subscription $subscription): void
    {
        $this->subscriptions[$subscription->id->toString()] = $subscription;
    }
}
```
```typescript
export class InMemorySubscriptionRepository implements SubscriptionRepository {
  private subscriptions: Array<Subscription> = [];

  async save(subscription: Subscription): Promise<void {
    this.subscriptions.push(subscription);
  }

  async get(id: SubscriptionId): Promise<Subscription> {
    const subscription = this.subscriptions.find(
      (subscription) => subscription.id === id,
    );

    if (!subscription) {
      throw new Error('Subscription not found');
    }

    return subscription;
  }
}
```

## Stub
A stub is the simplest implementation with a hardcoded behavior.

```php
final readonly class UniqueEmailSpecificationStub implements UniqueEmailSpecificationInterface
{
    public function isUnique(Email $email): bool
    {
        return true;
    }
}
```
```php
$specificationStub = $this->createStub(UniqueEmailSpecificationInterface::class);
$specificationStub->method('isUnique')->willReturn(true);
```

## Spy
A spy is an implementation to verify a specific behavior.

```php
final class SpyEventBus implements EventBus
{
    /**
     * @var DomainEvent[]
     */
    private array $events = [];

    public function dispatch(array $events): void
    {
        foreach ($events as $event) {
            $this->events[] = $event;
        }
    }

    /**
     * @param class-string $event
     */
    public function wasDispatched(string $event): bool
    {
        return false === empty(array_filter($this->events, fn (DomainEvent $e) => $e instanceof  $event));
    }
}
```
```typescript
export class SpyEventBus implements EventBus {
  private events: Array<DomainEvent> = [];

  async dispatch(events: Array<DomainEvent>): Promise<void> {
    this.events.push(...events);
  }

  wasDispatched(event: DomainEvent): boolean {
    return this.events.some((e) => JSON.stringify(e) === JSON.stringify(event));
  }
}
```

## Mock
A mock is a configured imitation to verify calls on a collaborator.

```php
$eventBus = $this->createMock(EventBus::class);
$eventBus->expects(self::once())->method('dispatch');
```
```php
const eventBus = { dispatch: jest.fn() };
expect(eventBus.dispatch).toHaveBeenCalledTimes(1);
```

## Mock vs Stub
![Mock vs Stub](/assets/img/2024-04-23/mock-vs-stub.png)
It’s important to know the difference between a stub and a mock. A stub is used to replace the incoming communication “Get data from a database” and we shouldn’t verify which method was used to do that. It isn’t important, and testing that, couples tests to the implementation details. A mock is used to replace the outgoing communication, and that communication is important, we need to verify that a particular event was dispatched or an email was sent.

In the following example, NotificationService is tested using a spy SpyMailer and a fake InMemoryRepository. Getters are unimportant, so only messages were saved by the repository, and we know nothing about the internal implementation of the system under test.
  
```php
final class NotificationService
{
    public function __construct(
        private readonly MailerInterface $mailer,
        private readonly MessageRepositoryInterface $messageRepository
    ) {}

    public function send(): void
    {
        $messages = $this->messageRepository->getAll();
        foreach ($messages as $message) {
            $this->mailer->send($message);
        }
    }
}
```
```php
final class TestExample extends TestCase
{
    /**
     * @test
     */
    public function sends_all_notifications(): void
    {
        $message1 = new Message();
        $message2 = new Message();
        $messageRepository = new InMemoryMessageRepository();
        $messageRepository->save($message1);
        $messageRepository->save($message2);
        $mailer = new SpyMailer();
        $sut = new NotificationService($mailer, $messageRepository);

        $sut->send();
        
        $mailer->assertThatMessagesHaveBeenSent([$message1, $message2]);
    }
}
```

## Two essential considerations for comparing mocking techniques
First, we need to understand two important definitions related to tests to debate mocking techniques.

## Resistance to refactoring
Refactoring is the process of restructuring existing code without changing its external behavior. It involves making improvements to the design, structure, and readability of code to enhance its maintainability, extensibility, and performance. Tests should not be broken after the refactoring process because they serve as a safety net for the codebase, ensuring that changes do not inadvertently introduce regressions or defects. However, the structure of code should be able to change, only the behavior of that code should be the same. The conclusion is simple, tests shouldn’t be coupled to the implementation details and should only verify the external behavior.

## Maintainability
Maintainability refers to the ease with which test code can be understood, modified, and extended over time. A maintainable test suite is well-organized, clear, and concise, making it efficient for developers to update or add new tests as the software evolves. This includes aspects such as readability, modularity, and the minimization of duplicated code, all of which contribute to the test suite’s ability to remain effective and adaptable in the face of changes to the underlying codebase. Tests should be treated in the same way as the production code.

## Usage of test doubles
Use test doubles for:
- in unit tests
  - out-of-process dependencies
    - a database
    - an event bus
    - a call to external API
    - a filesystem
    - volatile (non-deterministic) dependencies
      - a time
      - a random generator
- in integration tests
  - out-of-process dependencies
    - an event bus
    - a call to external API
  - volatile (non-deterministic) dependencies
    - a time
    - a random generator

Do not mock:
- Domain logic
- DTOs
- pure logic without infrastructure code

## Comparison of mocking techniques
The best way to talk about the pros and cons is by presenting a simple example of two approaches to using mocks in tests:
- configured by frameworks
- DIY / own mocks

## Frameworks – example
**PHP:**
```php
final readonly class CreateSubscription
{
    public function __construct(
        private SubscriptionRepository $repository,
        private EventBus $eventBus,
    ) {
    }

    public function execute(SubscriptionId $id): bool
    {
        try {
            $subscription = $this->repository->get($id);
        } catch (SubscriptionNotFound) {
            $subscription = null;
        }

        if ($subscription) {
            return false;
        }

        $subscription = new Subscription($id);
        $this->repository->save($subscription);
        $this->eventBus->dispatch($subscription->pullEvents());
        return true;
    }
}
```
```php
final class CreateSubscriptionTest extends TestCase
{
    public function test_creates_a_new_subscription(): void
    {
        // Arrange
        $repository = $this->createMock(SubscriptionRepository::class);
        $repository->method('get')->willThrowException(new SubscriptionNotFound());
        $eventBus = $this->createMock(EventBus::class);
        $subscriptionId = SubscriptionId::generate();
        $sut = new CreateSubscription($repository, $eventBus);

        // Assert
        $repository->expects(self::once())->method('save');
        $eventBus->expects(self::once())->method('dispatch');

        // Act
        $result = $sut->execute($subscriptionId);

        // Assert
        $this->assertTrue($result);
    }

    public function test_creating_is_not_possible_with_an_existing_subscription(): void
    {
        // Arrange
        $subscriptionId = SubscriptionId::generate();
        $subscription = new Subscription($subscriptionId);
        $repository = $this->createMock(SubscriptionRepository::class);
        $repository->method('get')->willReturn($subscription);
        $eventBus = $this->createMock(EventBus::class);
        $sut = new CreateSubscription($repository, $eventBus);

        // Assert
        $eventBus->expects(self::never())->method('dispatch');

        // Act
        $result = $sut->execute($subscriptionId);

        // Assert
        $this->assertFalse($result);
    }
}
```
```typescript
export class CreateSubscription {
  constructor(
    private readonly repository: SubscriptionRepository,
    private readonly eventBus: EventBus,
  ) {}
  async execute(subscriptionId: SubscriptionId): Promise {
    let subscription = null;

    try {
      subscription = await this.repository.get(subscriptionId);
    } catch (SubscriptionNotFound) {
      subscription = null;
    }

    if (subscription) {
      return false;
    }

    subscription = new Subscription(subscriptionId);
    await this.repository.save(subscription);
    await this.eventBus.dispatch(subscription.pullEvents());
    return true;
  }
}
```
```typescript
describe('CreateSubscription', () => {
  it('creates a new subscription', async () => {
    // Arrange
    const repository = {
      get: jest.fn(),
      find: jest.fn(),
      save: jest.fn(),
    };
    const eventBus = {
      dispatch: jest.fn(),
    };
    const sut = new CreateSubscription(repository, eventBus);

    // Act
    const result = await sut.execute(SubscriptionId.generate());

    // Assert
    expect(result).toBe(true);
    expect(repository.save).toHaveBeenCalledTimes(1);
    expect(eventBus.dispatch).toHaveBeenCalledTimes(1);
  });

  it('creating is not possible with an existing subscription', async () => {
    // Arrange
    const subscriptionId = SubscriptionId.generate();
    const repository = {
      get: jest.fn().mockResolvedValue({}),
      find: jest.fn(),
      save: jest.fn(),
    };
    const eventBus = {
      dispatch: jest.fn(),
    };
    const sut = new CreateSubscription(repository, eventBus);

    // Act
    const result = await sut.execute(subscriptionId);

    // Assert
    expect(result).toBe(false);
    expect(eventBus.dispatch).not.toHaveBeenCalledTimes(1);
    expect(repository.save).not.toHaveBeenCalledTimes(1);
  });
});
```

## DIY mocks – example
**PHP:**
```php
final class CreateSubscriptionTest extends TestCase
{
    public function test_creates_a_new_subscription(): void
    {
        // Arrange
        $subscriptionId = SubscriptionId::generate();
        $repository = new InMemorySubscriptionRepository();
        $eventBus = new SpyEventBus();
        $sut = new CreateSubscription($repository, $eventBus);

        // Act
        $result = $sut->execute($subscriptionId);

        // Assert
        $this->assertTrue($result);
        $this->assertNotNull($repository->get($subscriptionId));
        $this->assertTrue($eventBus->wasDispatched(SubscriptionCreated::class));
    }

    public function test_creating_is_not_possible_with_an_existing_subscription(): void
    {
        // Arrange
        $subscriptionId = SubscriptionId::generate();
        $repository = new InMemorySubscriptionRepository();
        $repository->save(new Subscription($subscriptionId));
        $eventBus = new SpyEventBus();
        $sut = new CreateSubscription($repository, $eventBus);

        // Act
        $result = $sut->execute($subscriptionId);

        // Assert
        $this->assertFalse($result);
        $this->assertFalse($eventBus->wasDispatched(SubscriptionCreated::class));
    }
}
```
```typescript
describe('CreateSubscription', () => {
  it('creates a new subscription', async () => {
    // Arrange
    const repository = new InMemorySubscriptionRepository();
    const eventBus = new SpyEventBus();
    const sut = new CreateSubscription(repository, eventBus);
    const subscriptionId = SubscriptionId.generate();

    // Act
    const result = await sut.execute(subscriptionId);

    // Assert
    expect(result).toBe(true);
    expect(await repository.get(subscriptionId)).not.toBeNull();
    expect(
      eventBus.wasDispatched(
        new SubscriptionCreated(subscriptionId.toString()),
      ),
    ).toBe(true);
  });

  it('creating is not possible with an existing subscription', async () => {
    // Arrange
    const repository = new InMemorySubscriptionRepository();
    const eventBus = new SpyEventBus();
    const sut = new CreateSubscription(repository, eventBus);
    const subscriptionId = SubscriptionId.generate();
    await repository.save(new Subscription(subscriptionId));

    // Act
    const result = await sut.execute(subscriptionId);

    // Assert
    expect(result).toBe(false);
    expect(await repository.find(subscriptionId)).not.toBeNull();
    expect(
      eventBus.wasDispatched(
        new SubscriptionCreated(subscriptionId.toString()),
      ),
    ).toBe(false);
  });
});
```

## Resistance to refactoring
### Renaming
The IDE handles simple renaming of the get method properly in the mocks configured by frameworks in PHP. I tested this using JetBrains IDE. However, in the case of TypeScript, my IDE doesn’t even automatically handle renaming the name of mocks

### Changing the used method
The most important aspect here is that when using frameworks, the test needs to identify which method is employed in the system under test. It’s not just a matter of injecting dependencies; there’s also a specification of which method should return the appropriate value. Let’s consider a scenario where a refactor is necessary, such as changing the method from “get” to “find” to eliminate an exception.

This change doesn’t disrupt the test with the in-memory repository because there’s no tight coupling between the test and the implementation. However, in the case of a mocking framework, only the “get” method is defined. Therefore, to adjust the code, we also need to modify our test and adapt the test double accordingly.

## Learning framework
Learning a new mock framework can be overhead because it requires time and effort. Additionally, in many projects are multiple mocking frameworks and there is no rule to use a particular one, which also adds some complexity to properly use them, and understand the code implemented by them.

## Reusability
Using DIY mocks provides us with the possibility of having fakes created by the owner of the real implementation and reusing them in every test case, which is convenient and eliminates duplication. Creating a separate test double for every test leads to a duplication of knowledge. The situation becomes more complicated when, for example, we make an assumption that a particular dependency returns specific values. For instance, when creating a mock, we might assume that the mocked dependency returns only positive numbers. However, if the requirements change to include returning negative numbers as well, we find ourselves with tests relying solely on positive numbers. We could opt to meticulously adjust all tests with mocks of that dependency, but this can be challenging in larger systems.

## Static analysis
Static analysis works well with DIY mocks because they adhere to standard language code. However, mocking frameworks are usually based on a hacky syntax, often diverging from the original dependency types. Frameworks like Mockery in PHP and Jest in TypeScript allow breaking the contract, for example, by returning null when there is no null defined in the interface. While this allows for faster creation of test doubles, it poses challenges for static analysis, resulting in the loss of useful checks. Consequently, the defined dependency may differ from the actual one.

## Easy to read
DIY mocks are easier to read because the test is shorter, the “arrange” section is simpler, and the actions are more meaningful:
- the event was dispatched vs the method has been called
- saving a subscription vs a configuration of the get method

## Easy to write
Mocks created by frameworks are often easier to write. However, it is not always true, because I would say that when we make an effort and prepare fakes, reusing them in other test cases should be even easier than using frameworks.

|                           | DIY mocks	 | Frameworks |
|---------------------------|------------|------------|
| Resistance to refactoring | 	✅         |	❌ |
| Learning framework | 	✅         | 	❌         |
| Reusability	| ✅	| ❌ |          
| Static analysis |	✅ |	❌ |      
| Easy to read |	✅ |	❌ |          
| Easy to write |	❌ |	✅ |        

## Summary
In light of the challenges associated with mocking frameworks, an increasing number of engineers, including those at Google (I highly recommend book – “Google Software Engineering”), are recognizing the benefits of adopting a custom test double approach. This shift reflects a broader understanding of the importance of selecting the appropriate testing methodologies and acknowledges the limitations of heavy reliance on mocking frameworks. By opting for custom test doubles, engineers can mitigate scalability issues inherent in mocking frameworks. Unlike mocks with hardcoded behavior, which often lead to violations of the DRY principle and duplication of knowledge, custom test doubles offer a more sustainable solution. They provide resistance to refactoring, allowing tests to remain stable in the face of changes to the codebase. Additionally, engineers are freed from the need to learn and navigate the complexities of mocking frameworks, reducing the barrier to entry for testing.

Furthermore, the reusability of custom mocks fosters efficiency and consistency in testing practices. These mocks can be easily shared and adapted across different tests. Additionally, custom test doubles align more seamlessly with static analysis tools, enabling early detection of potential issues and ensuring code quality. Despite the challenges associated with writing custom test doubles, including the higher initial investment of development effort and the increased complexity of implementation, the long-term benefits of improved resilience, simplicity, and maintainability justify this approach.

In conclusion, while mocking frameworks may offer convenience in the short term, the drawbacks of scalability, complexity, and maintenance overhead necessitate a reevaluation of testing strategies. By embracing custom test doubles, engineers can achieve more realistic testing scenarios, enhance the reliability of their tests, and ultimately contribute to a more robust and sustainable testing ecosystem.
