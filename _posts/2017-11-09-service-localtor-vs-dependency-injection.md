---
title: Service locator vs Dependency injection
date: 2017-11-09
categories: [Programming, Patterns]
permalink: /pl/2017/08/5-solid-dependency-inversion-principle/
image:
  path: /assets/img/2017-11-09/featured.png
---
Projektując aplikację w obiektowym języku programowania tworzymy klasy. Klasy mają własne zależności. Wyróżniamy dwa wzorce odpowiadające za zarządzanie zależnościami klasy:
- Dependency injection
- Service locator

## Dependency injection
W poniższym przykładzie klasa Service doskonale pokazuje swoje zależności. Wiemy, że aby utworzyć instancję tej klasy musimy wstrzyknąć Logger do konstruktora.

```php
<?php

use Psr\Log\LoggerInterface;

class Service
{
    private $logger;

    public function __construct(LoggerInterface $em)
    {
        $this->logger = $logger;
    }

    public function doSomething()
    {
        $this->logger->info('...');
    }
}
```

## Service locator
Poniżej mamy prosty przykład wzorca service locator:
```php
<?php

use Psr\Container\ContainerInterface;

class Service
{
    private $container;

    public function __construct(ContainerInterface $container)
    {
        $this->container = $container;
    }
    
    public function doSomething()
    {
        $this->container->get('logger')->info('...');
    }
}
```
Powiedz mi teraz jakie są zależności klasy Service? Oczywiście nie wiesz. Nie jesteśmy w stanie tego określić. Wiemy, że zależnością klasy Service jest klasa implementująca ContainerInterface, jednak z kontenera jesteśmy w stanie wyciągnąć pozostałe zależności, które bez sprawdzania kodu całej klasy pozostają dla nas ukryte.

## Service locator jako antywzorzec
Service locator przez ukrywanie zależności jest uznawany przez wielu ludzi za antywzorzec. Ukryte zależności utrudniają pisanie testów jednostkowych. Nie możemy w prosty sposób mockować zależności i tworzyć rzeczywistych testów jednostkowych.

Mockowanie oznacza tworzenie atrapy obiektu, tworząc test jednostkowy skupiamy się tylko na jednostce tj. testowanej klasie, dlatego za wszelkie zależności podstawiamy atrapy, które tylko naśladują konkretne obiekty.

## Magia laravela
W laravelu mamy pewne twory zwane [Fasadami](https://laravel.com/docs/5.5/facades). Wiele osób hejtuje laravela własnie za nie. Dlaczego?

Z fasadami jest taki sam problem jak z wzorcem Service locator, ukrywają one zależności. Jeśli spojrzysz na konstruktor klasy korzystającej z fasad, nie wiesz jakie są jej zależności. Jeśli tego nie wiesz, nie możesz ich zmockować. Można powiedzieć, że fasady są nawet gorsze, ponieważ możemy je wywołać wszędzie. Fasada w szablonie? Żaden problem.

## Jak to działa?
```php
Log::info('...');
```

Widzimy tutaj operator :: co wskazywałoby na wywołanie statycznej metody info(), nic bardziej mylnego! W rzeczywistości każda fasada dziedziczy po klasie bazowej magiczną metodę:

```php
public static function __callStatic($method, $args)
{
    $instance = static::getFacadeRoot();
    if (! $instance) {
        throw new RuntimeException('A facade root has not been set.');
    }
    return $instance->$method(...$args);
}
```
Z tego wynika, że nie jest to faktycznie wywołanie statycznej metody, lecz metody instancyjnej.

## Wiązanie z frameworkiem
Porównajmy dwa przykład kodu. Pierwszy z wykorzystaniem dependency injection:
```php
<?php
final class Service implements ServiceInterface
{
    private $logger;
    public function __construct(LoggerInterface $logger)
    {
        $this->logger = $logger;
    }
    public function doSomething(): void
    {
        $this->logger->info('...');
    }
}
```

Drugi z fasadami laravela:

```php
final class Service implements ServiceInterface
{
    public function doSomething(): void
    {
        Log::info('...');
    }
}
```
Kod z pierwszego przykładu możemy łatwo wykorzystać w innym frameworku. Nasza klasa nie wie jakiego frameworka używa, więc jego zmiana nie stanowi problemu. Natomiast w drugim przykładzie klasa ma pełną świadomość korzystania z konkretnego frameworka, więc użycie jej w innym projekcie z innym frameworkiem bez refaktoryzacji jest niemożliwe. Wnioski nasuwają się same, używając fasad wiążemy nasz kod z laravelem, co do dobrych praktyk nie należy.

## Podsumowanie
Jeśli chcemy pisać kod łatwy w testowaniu oraz zarządzaniu powinniśmy unikać używania wzorca service locator oraz fasad. Zależności klasy powinny być jasno określone, a nie ukryte. Po prostu należy wykorzystywać Dependency injection zgodnie z zasadą [Dependency inversion principle](https://sarvendev.com/2017/08/5-solid-dependency-inversion-principle/).
