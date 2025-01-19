---
title: SOLID – Open/closed principle
date: 2017-08-20
categories: [Programming, Good practices]
permalink: /pl/2017/08/2-solid-openclosed-principle/
image:
  path: /assets/img/2017-08-20/featured.png
---
Projektując poważny system musimy mieć na uwadze jego przyszłą ewolucję. Kolejną z zasad SOLID, która pozwoli nam w spokoju rozwijać nasz kod jest Open/closed principle. Mówimy nam ona o tym, że kod powinien być otwarty na rozbudowę oraz zamknięty na modyfikację.


## Co to znaczy?
Najprostszym sposobem na ocenienie czy nasz kod jest zgodny z tą zasadą jest odpowiedzenie sobie na następujące pytanie:
- Czy jesteśmy wstanie dodać nową funkcjonalność poprzez dodanie nowych klas nie zmieniając istniejącego już kodu?

## Przykład złamania zasady
```php
<?php

/**
 * Class Triangle
 */
class Triangle
{
    /**
     * @var float
     */
    private $a;

    /**
     * @var float
     */
    private $h;

    /**
     * @return float
     */
    public function getA(): float
    {
        return $this->a;
    }

    /**
     * @param float $a
     * @return Triangle
     */
    public function setA(float $a): Triangle
    {
        $this->a = $a;
        return $this;
    }

    /**
     * @return float
     */
    public function getH(): float
    {
        return $this->h;
    }

    /**
     * @param float $h
     * @return Triangle
     */
    public function setH(float $h): Triangle
    {
        $this->h = $h;
        return $this;
    }
}
```
```php
<?php

/**
 * Class Circle
 */
class Circle
{
    /**
     * @var float
     */
    private $radius;

    /**
     * @return float
     */
    public function getRadius(): float
    {
        return $this->radius;
    }

    /**
     * @param float $radius
     * @return Circle
     */
    public function setRadius(float $radius): Circle
    {
        $this->radius = $radius;
        return $this;
    }
}
```

```php
<?php

/**
 * Class AreaCalculator
 */
class AreaCalculator
{
    /**
     * @param array ...$shapes
     * @return float
     */
    public function sum(...$shapes): float
    {
        $sum = 0;

        foreach ($shapes as $shape) {
            $sum += $this->calcArea($shape);
        }

        return $sum;
    }

    /**
     * @param $shape
     * @return float
     */
    private function calcArea($shape): float
    {
        switch(get_class($shape)) {
            case 'Triangle':
                return 0.5 * $shape->getA() * $shape->getH();
            case 'Circle':
                return M_PI * $shape->getRadius() * $shape->getRadius();
        }
    }
}
```
Powyżej znajduje się napisany na szybko przykład złamania zasady Open/closed. Mamy tutaj trzy klasy: dwie reprezentujące odpowiednio trójkąt oraz okrąg, natomiast trzecia z nich odpowiada za wykonywanie operacji na polach figur. Wszystko wydaje się tutaj ok, jednak zastanówmy się, co jeśli będziemy chcieli rozszerzyć funkcjonalność i dodać prostokąt. Oprócz napisania odpowiedniej klasy reprezentującej nową figurę, będziemy również musieli zmodyfikować metodę calcArea w klasie AreaCalculator, czyli nasz kod nie jest zgodny z zasadą OCP.

### Przykład poprawnego kodu
```php
<?php

/**
 * Interface AreaCalculableInterface
 */
interface AreaCalculableInterface
{
    public function calcArea(): float;
}
```

```php
<?php

/**
 * Class Circle
 */
class Circle implements AreaCalculableInterface
{
    /**
     * @var float
     */
    private $radius;

    /**
     * @return float
     */
    public function getRadius(): float
    {
        return $this->radius;
    }

    /**
     * @param float $radius
     * @return Circle
     */
    public function setRadius(float $radius): Circle
    {
        $this->radius = $radius;
        return $this;
    }

    /**
     * {@inheritdoc}
     */
    public function calcArea(): float
    {
        return M_PI * $this->radius * $this->radius;
    }
}
```

```php
<?php

/**
 * Class Triangle
 */
class Triangle implements AreaCalculableInterface
{
    /**
     * @var float
     */
    private $a;

    /**
     * @var float
     */
    private $h;

    /**
     * @return float
     */
    public function getA(): float
    {
        return $this->a;
    }

    /**
     * @param float $a
     * @return Triangle
     */
    public function setA(float $a): Triangle
    {
        $this->a = $a;
        return $this;
    }

    /**
     * @return float
     */
    public function getH(): float
    {
        return $this->h;
    }

    /**
     * @param float $h
     * @return Triangle
     */
    public function setH(float $h): Triangle
    {
        $this->h = $h;
        return $this;
    }

    /**
     * {@inheritdoc}
     */
    public function calcArea(): float
    {
        return 0.5 * $this->a * $this->h;
    }
}
```

```php
<?php

/**
 * Class AreaCalculator
 */
class AreaCalculator
{
    /**
     * @param AreaCalculableInterface[] ...$shapes
     * @return float
     */
    public function sum(AreaCalculabeInterface ...$shapes): float
    {
        $sum = 0;

        foreach ($shapes as $shape) {
            if (!$shape instanceof AreaCalculableInterface) {
                throw new \RuntimeException('Shape have to be instance of AreaCalculabeInterface.');
            }

            $sum += $shape->calcArea();
        }

        return $sum;
    }
}
```

W powyższym przykładzie dodany został interfejs, który następnie jest implementowany w klasach reprezentujących figury, co wymusza zaimplementowanie w nich metody calcArea(). Dzięki takiej zmianie dodanie nowej figury wymaga od nas utworzenia jedynie odpowiedniej klasy reprezentującej tę figurę.

## Podsumowanie
Oczywiście powyższy przykład jest trywialny i wiadomo jakie elementy mogą się zmieniać. Jednak jeśli projektujemy złożony system to często spotykamy się z problemem ustalenia, które elementy w przyszłości mogą ulec zmianie, tak aby je odpowiednio zaprojektować i przygotować się na wdrażanie nowych funkcjonalności w prosty sposób.

Nie powinniśmy też popadać w paranoje próbując idealnie zaprojektować każdą część aplikacji. Ważną kwestią jest odpowiednia refaktoryzacja kodu, do którego nie przewidzieliśmy przyszłych zmian, wtedy gdy jakieś zmiany w tym miejscu wystąpią. Wszystkiego idealnie nie przewidzimy, więc trzeba czasem reagować na bieżąco i pamiętać o zostawianiu kodu w lepszym stanie niż go zastaliśmy.
