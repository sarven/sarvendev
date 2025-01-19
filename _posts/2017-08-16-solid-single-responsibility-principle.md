---
title: SOLID – Single responsibility principle
date: 2017-08-16
categories: [Programming, Good practices]
permalink: /pl/2017/08/1-solid-single-responsibility-principle/
image:
  path: /assets/img/2017-08-16/featured.png
---

Sama umiejętność rozwiązania problemu nie jest wystarczająca, aby było ono dobrej jakości. W celu zapewnienia sobie spokojnej przyszłości w pracy z łatwym w rozwoju oraz utrzymaniu kodzie należy przestrzegać pewnych zasad.

## SOLID
SOLID jest mnemonikiem ułatwiającym zapamiętanie pięciu podstawowych zasad programowania obiektowego.
- SRP – Single responsibility principle (Zasada pojedynczej odpowiedzialności)
- OCP – Open/closed principle (Zasada otwarte/zamknięte)
- LSP – Liskov substitution principle (Zasada podstawienia Liskov)
- ISP – Interface segregation principle (Zasada segregacji interfejsów)
- DIP – Dependency inversion principle (Zasada odwrócenia zależności)

## SRP – Single responsibility principle
Pierwsza z zasad mówi nam, że klasa powinna mieć tylko jedną odpowiedzialność tj. nie powinien istnieć więcej niż jeden powód do jej zmiany. Jako miniatura tego postu ustawiony jest multitool, który może być przykładem złamania SRP. Zamiast osobnych narzędzi mamy tutaj kilka zamknięte w jedno. Oczywiście jeśli mamy mało wymagające czynności to takie rozwiązanie powinno nam w zupełności wystarczyć, jednak jeśli potrzebujemy czegoś bardziej wymagającego, gdzie wykorzystujemy kilka różnych wielkości śrubokręta to pojawia się problem. Przekładając tę analogię na programowanie, gdy mamy niewielki projekt, który nie będzie rozwijany (np. szybki prototyp, który później leci do kosza) tworzenie takich klas wykonujących kilka czynności nie napędzi nam ogromnych kłopotów. Natomiast przy tworzeniu projektu, który ma być porządnie przetestowany oraz w przyszłości rozwijany powinniśmy mieć tą zasadę na uwadze i starać się pisać kod tak, aby jej nie łamać.

Często w klasach wzorca Active record mamy do czynienia z naruszeniem założeń SRP. Jako przykład możemy tutaj podać Model z używanego w Laravelu ORMa zwanego Eloquent. Odpowiedzialnością modelu powinna być jedynie reprezentacja danych oraz logiki biznesowej. Poniżej mamy przykład z dokumentacji tego frameworka. Widzimy, że oprócz reprezentacji danych model zajmuje się również ich wyszukiwaniem oraz zapisywaniem.

```php
$flight = App\Flight::find(1);
$flight->name = 'New Flight Name';
$flight->save();
```

Przejdźmy jednak do przykładów poprawnych implementacji klas zgodnie z założeniami SRP.

## PHP
```php
<?php

/**
 * Class Point
 */
class Point
{
    /**
     * @var float
     */
    private $x;

    /**
     * @var float
     */
    private $y;

    /**
     * @return float
     */
    public function getX(): float
    {
        return $this->x;
    }

    /**
     * @param float $x
     * @return Point
     */
    public function setX(float $x): Point
    {
        $this->x = $x;
        return $this;
    }

    /**
     * @return float
     */
    public function getY(): float
    {
        return $this->y;
    }

    /**
     * @param float $y
     * @return Point
     */
    public function setY(float $y): Point
    {
        $this->y = $y;
        return $this;
    }
}
```

```php
<?php

/**
 * Class PointDrawer
 */
final class PointDrawer implements PointDrawerInterface
{
    /**
     * {@inheritdoc}
     */
    public function draw(Point $point): void
    {

    }
}
```

## JS

```javascript
class Point
{
    /**
     * @param x
     * @param y
     */
    constructor(x, y)
    {
        this.x = x;
        this.y = y;
    }
}
```

```javascript
class PointDrawer
{
    /**
     * @param Point point
     */
    draw(point)
    {

    }
}
```

Powyżej mamy implementację w php oraz javascript prototypu klasy reprezentującej punkt oraz prototypu klasy odpowiedzialnej za rysowanie punktu. Można zauważyć, że każda z nich posiada tylko jedną odpowiedzialność:
- Point  – reprezentacja punktu
- PointDrawer  – rysowanie punktu
