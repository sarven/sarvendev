---
title: Draft - Repository Testing Done Right
date: 2025-02-24
categories: [Programming, Testing]
---
In most projects, there is a need to interact with some kind of database. There is an approach that we have a layer called 
repository which is responsible for saving and retrieving data from the database. Usually, we write an integration test 
for the repository, and to make the rest of the tests faster we use an in-memory implementation of that repository. 
It’s a basic concept, and almost everyone is familiar with it, so everything is clear and simple, right? 
Not really. In this article, I will show you how to test repositories in a way that will make your tests closer 
to the real behavior, which means that you will be more confident that your code works as expected.

## Example 
Let's start with a simple example. We have a repository that is responsible for saving and retrieving entities.

```php
interface EntityRepository
{
    public function save(Entity $entity): void;

    /**
     * @throws EntityNotFoundException
     */
    public function get(int $id): Entity;
}
```

## Integration test
Let's start with an integration test. 

```php
final class DoctrineEntityRepositoryTest extends KernelTestCase
{
    private EntityManagerInterface $entityManager;
    private EntityRepository $sut;

    protected function setUp(): void
    {
        self::bootKernel();
        $this->entityManager = self::getContainer()->get('doctrine')->getManager();
        $this->sut = self::getContainer()->get(EntityRepository::class);
    }

    public function testSave(): void
    {
        $entity = new Entity('Name');
        $this->sut->save($entity);
        
        $savedEntity = $this->sut->get($entity->getId());
        
        self::assertSame($entity->getId(), $savedEntity->getId());
        self::assertSame($entity->getName(), $savedEntity->getName());
    }
}
```

Does this test look ok? Yes, it does. But there is a problem with it. Now let's write the implementation of the repository.

```php
final readonly class DoctrineEntityRepository implements EntityRepository
{
    public function __construct(private EntityManagerInterface $entityManager)
    {
    }

    public function get(int $id): Entity
    {
        return $this->entityManager->find(Entity::class, $id)
            ?? throw new RuntimeException('Entity not found');
    }

    public function save(Entity $entity): void
    {
        $this->entityManager->persist($entity);
    }
}
```

Notice that there is no flush method in the save method. However, the test is passing, but no changes will be saved in the database.
How to fix that problem? We need to start from a failing test, so let's write a test that will fail. Now our test is working only
in memory, we use the persist method, the entity is saved in Unit of Work inside Entity Manager, and then during getting the entity
we can retrieve it from the memory, but in the real implementation, we need to save the entity in the database. To make the test
more realistic we need to clear the Unit of Work after saving the entity.

```php
public function testSave(): void
{
    $entity = new Entity('Name');
    $this->sut->save($entity);
    
    // Clear the entity manager to make sure we are getting the entity from the database
    $this->entityManager->clear();

    $savedEntity = $this->sut->get($entity->getId());
    self::assertSame($entity->getId(), $savedEntity->getId());
    self::assertSame($entity->getName(), $savedEntity->getName());
}
```

Now the test is failing, so we can fix the implementation of the save method and add flush there:
```php
public function save(Entity $entity): void
{
    $this->entityManager->persist($entity);
    $this->entityManager->flush();
}
```

Now the test is passing. It isn't only a case in testing repositories. The same thing can happen when we are testing
a full REST endpoint, which updates some data in the database, if we check assertions without clearing data
in the Entity Manager, we can get an entity from the repository, edit it, forget about saving it by the repository and
then checking in assertions will cause the same problem, we'll check data in memory instead of the database, so the test
will pass, but on the production it will be a bug.

## In-memory test double and unit tests

Let's consider a simple use case with changing the name of the entity. The test will look like this:
```php
final class UpdateEntityTest extends TestCase
{
    private InMemoryEntityRepository $inMemoryEntityRepository;
    private UpdateEntity $sut;

    protected function setUp(): void
    {
        parent::setUp();

        $this->inMemoryEntityRepository = new InMemoryEntityRepository();
        $this->sut = new UpdateEntity($this->inMemoryEntityRepository);
    }

    public function testUpdate(): void
    {
        $entity = $this->givenEntity('Name');

        $this->sut->__invoke($entity->getId(), 'New Name');

        $updatedEntity = $this->inMemoryEntityRepository->get($entity->getId());
        self::assertSame('New Name', $updatedEntity->getName());
    }

    private function givenEntity(string $name): Entity
    {
        $entity = new Entity($name);
        $this->inMemoryEntityRepository->save($entity);

        return $entity;
    }
}
```
We can use in-memory implementation of the repository to make the test faster because the real implementation
of the repository was tested in the integration test.
```php
final class InMemoryEntityRepository implements EntityRepository
{
    /**
     * @var array<int, Entity>
     */
    private array $entities = [];

    private int $nextId = 1;

    public function save(Entity $entity): void
    {
        if (null !== $entity->getId()) {
            $this->entities[$entity->getId()] = $entity;
            return;
        }

        $reflection = new \ReflectionClass($entity);
        $property = $reflection->getProperty('id');
        $property->setAccessible(true);
        $property->setValue($entity, $this->nextId);
        $this->entities[$this->nextId] = $entity;
        $this->nextId++;
    }

    public function get(int $id): Entity
    {
        return $this->entities[$id] ?? throw new EntityNotFoundException();
    }
}
```

```php
final readonly class UpdateEntity
{
    public function __construct(
        private EntityRepository $entityRepository,
    ) {
    }

    public function __invoke(int $id, string $name): void
    {
        $entity = $this->entityRepository->get($id);
        $entity->changeName($name);
    }
}
```
Notice that there is no save method in the UpdateEntity class. We are not saving the entity in the repository, but
the test is passing because we are checking the same instance of the entity, which is in the memory. So it works in the test,
but with the real implementation of the repository on the production, it will be a bug. How to make the test fail? We need to
make the behavior of the test closer to the real behavior. We need to make sure that the get method will return
a new instance of the entity, not the same instance that we have in the memory. We can do it by cloning
the entity in the get method, but to make the full clone, it's better to use a package like DeepCopy.
```
composer require myclabs/deep-copy
```

```php
public function get(int $id): Entity
{
    $entity = $this->entities[$id] ?? throw new EntityNotFoundException();
    /** @var Entity $entity */
    $entity = deep_copy($entity);
    return $entity;
}
```
Now the test is failing, so we can fix the code by adding the missing save method in the UpdateEntity class:
```php
public function __invoke(int $id, string $name): void
{
    $entity = $this->entityRepository->get($id);
    $entity->changeName($name);
    $this->entityRepository->save($entity);
}
```

## Summary
Test fidelity is very important. We need to make sure that our tests are as close to the real behavior as possible.
It helps us find bugs earlier and makes our code more reliable, so we can be more confident
that our code works as expected.
