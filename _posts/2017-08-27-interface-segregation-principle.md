---
title: SOLID – Interface segregation principle
date: 2017-08-27
categories: [Programming, Good practices]
permalink: /pl/2017/08/4-solid-interface-segregation-principle/
redirect_from:
  - /2017/08/4-solid-interface-segregation-principle/
image:
  path: /assets/img/2017-08-27/featured.png
---
W klasie implementującej interfejs znaleźć muszą się implementacje wszystkich metod zawartych w tym interfejsie. W przypadku gdy w danej klasie nie potrzebujemy wszystkich metod, pojawia się problem. Przestrzeganie kolejnej z zasad programowania obiektowego SOLID pozwoli nam uniknąć takich kłopotów.


Zasada Interface segregation principle mówi nam, że klasa nigdy nie powinna być zmuszana do implementacji metod, których nie używa. W celu osiągnięcia takiego stanu rzeczy należy tworzyć zwięzłe interfejsy zamiast tzw. grubych interfejsów. Powinny one zawierać metody ściśle ze sobą powiązane.

## Przykład złamania
```php
<?php

/**
 * Interface ShapeInterface
 */
interface ShapeInterface
{
    /**
     * @return float
     */
    public function calcArea(): float;

    /**
     * @return float
     */
    public function calcVolume(): float;
}
```

```php
<?php

/**
 * Class Rectangle
 */
final class Rectangle implements ShapeInterface
{
    /**
     * @var float
     */
    private $width;

    /**
     * @var float
     */
    private $height;

    /**
     * Rectangle constructor.
     * @param float $width
     * @param float $height
     */
    public function __construct(float $width, float $height)
    {
        $this->width = $width;
        $this->height = $height;
    }

    /**
     * @return float
     */
    public function getWidth(): float
    {
        return $this->width;
    }

    /**
     * @return float
     */
    public function getHeight(): float
    {
        return $this->height;
    }

    /**
     * @return float
     */
    public function calcArea(): float
    {
        return $this->width * $this->height;
    }

    /**
     * @return float
     */
    public function calcVolume(): float
    {
        return 0;
    }
}
```

```php
<?php

/**
 * Class Cuboid
 */
final class Cuboid implements ShapeInterface
{
    /**
     * @var float
     */
    private $a;

    /**
     * @var float
     */
    private $b;

    /**
     * @var float
     */
    private $c;

    /**
     * Cuboid constructor.
     * @param float $a
     * @param float $b
     * @param float $c
     */
    public function __construct(float $a, float $b, float $c)
    {
        $this->a = $a;
        $this->b = $b;
        $this->c = $c;
    }

    /**
     * @return float
     */
    public function getA(): float
    {
        return $this->a;
    }

    /**
     * @return float
     */
    public function getB(): float
    {
        return $this->b;
    }

    /**
     * @return float
     */
    public function getC(): float
    {
        return $this->c;
    }

    /**
     * @return float
     */
    public function calcArea(): float
    {
        return 2 * ($this->a * $this->b + $this->c * $this->b + $this->a * $this->c);
    }

    /**
     * @return float
     */
    public function calcVolume(): float
    {
        return $this->a * $this->b * $this->c;
    }
}
```

W powyższym przykładzie mamy interfejs ShapeInterface oraz dwie klasy(Rectangle, Cuboid), które go implementują. Z racji, że w figurach płaskich obliczamy jedynie pole, natomiast objętość jest przeznaczona dla figur przestrzennych to metody calcArea() oraz calcVolume() nie powinny się znaleźć w jednym interfejsie.

## Poprawny przykład

```php
<?php

/**
 * Interface AreaCalculableInterface
 */
interface AreaCalculableInterface
{
    /**
     * @return float
     */
    public function calcArea(): float;
}
```

```php
<?php

/**
 * Interface VolumeCalculableInterface
 */
interface VolumeCalculableInterface
{
    /**
     * @return float
     */
    public function calcVolume(): float;
}
```

```php
<?php

/**
 * Class Cuboid
 */
final class Cuboid implements VolumeCalculableInterface, AreaCalculableInterface
{
    /**
     * @var float
     */
    private $a;

    /**
     * @var float
     */
    private $b;

    /**
     * @var float
     */
    private $c;

    /**
     * Cuboid constructor.
     * @param float $a
     * @param float $b
     * @param float $c
     */
    public function __construct(float $a, float $b, float $c)
    {
        $this->a = $a;
        $this->b = $b;
        $this->c = $c;
    }

    /**
     * @return float
     */
    public function getA(): float
    {
        return $this->a;
    }

    /**
     * @return float
     */
    public function getB(): float
    {
        return $this->b;
    }

    /**
     * @return float
     */
    public function getC(): float
    {
        return $this->c;
    }

    /**
     * @return float
     */
    public function calcArea(): float
    {
        return 2 * ($this->a * $this->b + $this->c * $this->b + $this->a * $this->c);
    }

    /**
     * @return float
     */
    public function calcVolume(): float
    {
        return $this->a * $this->b * $this->c;
    }
}
```

```php
<?php

/**
 * Class Rectangle
 */
final class Rectangle implements AreaCalculableInterface
{
    /**
     * @var float
     */
    private $width;

    /**
     * @var float
     */
    private $height;

    /**
     * Rectangle constructor.
     * @param float $width
     * @param float $height
     */
    public function __construct(float $width, float $height)
    {
        $this->width = $width;
        $this->height = $height;
    }

    /**
     * @return float
     */
    public function getWidth(): float
    {
        return $this->width;
    }

    /**
     * @return float
     */
    public function getHeight(): float
    {
        return $this->height;
    }

    /**
     * @return float
     */
    public function calcArea(): float
    {
        return $this->width * $this->height;
    }
}
```

W tym przykładzie interfejs został rozbity, przez co w klasie Rectangle implementujemy tylko AreaCalculableInterface i  możemy usunąć zbędną metodę calcVolume(), dzięki czemu kod jest zgodny z zasadą ISP.
