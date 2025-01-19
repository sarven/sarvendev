---
title: #5 SOLID – Dependency inversion principle
date: 2017-08-31
categories: [Programming, Good practices]
permalink: /pl/2017/08/5-solid-dependency-inversion-principle/
image:
  path: /assets/img/2017-08-31/featured.png
---
Opierając się na szczegółowej implementacji klasy podczas wstrzykiwania zależności tworzymy sprzężenie pomiędzy klasą a zależnością. Kod staje się sztywny, a ewentualna podmiana zależności stanowi problem. W celu stworzenia elastycznego rozwiązania powinniśmy przestrzegać zasady Dependency inversion principle, która prezentuje się następująco:

> Moduły wysokopoziomowe nie powinny zależeć od modułów niskopoziomowych. Jedne i drugie powinny zależeć od abstrakcji.
> Abstrakcje nie powinny zależeć od szczegółów. To szczegóły powinny zależeć od abstrakcji.

## Przykład złamania
```php
<?php

/**
 * Class EntityManager
 */
final class EntityManager
{
    /**
     * @var MysqlConnection
     */
    private $connection;

    /**
     * EntityManager constructor.
     * @param MysqlConnection $connection
     */
    public function __construct(MysqlConnection $connection)
    {
        $this->connection = $connection;
    }
}
```
```php
<?php

/**
 * Class MysqlConnection
 */
final class MysqlConnection
{
    public function connect(): void
    {
    }
}
```

W powyższym przykładzie mamy dwie klasy:
- EntityManager – utrwalanie obiektów w bazie danych
- MysqlConnection – udostępnianie operacji na bazie danych MySQL
Baz danych możemy używać jednak różnych, więc kod napisany w ten sposób nie jest elastyczny i łamie zasadę DIP. EntityManager operuje na szczegółowej implementacji MysqlConnection zamiast na abstrakcji.

## Przykład poprawnej implementacji
```php
<?php

/**
 * Interface DatabaseConnectionInterface
 */
interface DatabaseConnectionInterface
{
    public function connect(): void;
}
```
```php
<?php

/**
 * Class MysqlConnection
 */
final class MysqlConnection implements DatabaseConnectionInterface
{
    public function connect(): void
    {
    }
}
```
```php
<?php

/**
 * Interface EntityManagerInterface
 */
interface EntityManagerInterface
{
}
```
```php
<?php

/**
 * Class EntityManager
 */
final class EntityManager implements EntityManagerInterface
{
    /**
     * @var DatabaseConnectionInterface
     */
    private $connection;

    /**
     * EntityManager constructor.
     * @param DatabaseConnectionInterface $connection
     */
    public function __construct(DatabaseConnectionInterface $connection)
    {
        $this->connection = $connection;
    }
}
```

Obecnie EntityManager operuje na interfejsie DatabaseConnectionInterface, więc dodanie innej szczegółowej implementacji tego interfejsu nie stanowi problemu. Kod został tak zrefaktoryzowany, że spełnia zasadę DIP.

Należy pamiętać, że ta, jak i pozostałe zasady SOLID dotyczą głównie kodu, który będzie w przyszłości zmieniany, więc jeśli w niektórych przypadkach posłużymy się szczegółową implementacją zamiast abstrakcją, to nie powinno to wpłynąć na trudność utrzymania projektu. Powinniśmy umieć wyważyć w jakich sytuacjach pisanie kodu w stu procentach zgodnego z SOLID jest konieczne, a w jakich nie.
