---
title: #3 SOLID – Liskov substitution principle
date: 2017-08-23
categories: [Programming, Good practices]
permalink: /pl/2017/08/3-solid-liskov-substitution-principle/
image:
  path: /assets/img/2017-08-23/featured.png
---
Kolejną z zasad SOLID pozwalających na tworzenie dobrej jakości rozwiązań jest zasada Liskov substitution principle(Zasada podstawiania Liskov). Sformułowana ona została przez Barbarę Liskov w książce Data Abstraction and Hierarch.

Definicja prezentuje się w następujący sposób.
> Let f(x) be a property provable about objects x of type T. Then f(y) should be true for objects y of type S where S is a subtype of T.

Przekładając to na prosty język, po prostu musi istnieć możliwość podstawienia typów pochodnych za ich typy bazowe. Jeśli jakaś funkcja przyjmuje jako argument obiekt klasy A, to musi również działać prawidłowo dla obiektów pochodnych klasy A, czyli dziedziczących po niej.

## Przykład złamania

```php
<?php

/**
 * Class Rectangle
 */
class Rectangle
{
    /**
     * @var float
     */
    protected $width;

    /**
     * @var float
     */
    protected $height;

    /**
     * @return float
     */
    public function getWidth(): float
    {
        return $this->width;
    }

    /**
     * @param float $width
     * @return Rectangle
     */
    public function setWidth(float $width): Rectangle
    {
        $this->width = $width;
        return $this;
    }

    /**
     * @return float
     */
    public function getHeight(): float
    {
        return $this->height;
    }

    /**
     * @param float $height
     * @return Rectangle
     */
    public function setHeight(float $height): Rectangle
    {
        $this->height = $height;
        return $this;
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
```php
<?php

/**
 * Class Square
 */
class Square extends Rectangle
{
    /**
     * @param float $height
     * @return Rectangle
     */
    public function setHeight(float $height): Rectangle
    {
        $this->height = $height;
        $this->width = $height;

        return $this;
    }

    /**
     * @param float $width
     * @return Rectangle
     */
    public function setWidth(float $width): Rectangle
    {
        $this->width = $width;
        $this->height = $width;

        return $this;
    }
}
```
```php
<?php

function getArea(Rectangle $rectangle): float {
    $rectangle->setHeight(2);
    $rectangle->setWidth(3);

    return $rectangle->calcArea();
}

$square = new Square();

$area = getArea($square); // 9
```
W powyższym przykładzie mamy klasę Rectangle oraz klasę pochodną Square. Oczywiście wszystko wydaje się być w porządku, przecież kwadrat jest prostokątem, więc dziedziczenie wydaje się uzasadnione. Na ostatnim listingu znajduje się funkcja getArea() przyjmująca jako argument obiekt typu Rectangle, co znaczy, że w myśl zasady LSP powinna ona działać poprawnie dla obiektów klasy Rectangle oraz obiektów pochodnych klasy Rectangle, czyli w tym wypadku Square. Jednak po ustawieniu odpowiednich wartości boków, które wskazują, że operujemy na prostokącie okazuje się, że zwracana jest niepoprawna wartość pola, dlatego iż parametrem przekazanym do funkcji był obiekt klasy Square.

## Jak to poprawnie zaimplementować?
Rozwiązanie tego problemu wcale nie jest prostym zadaniem. Również do takiego wniosku doszedłem przeglądając niektóre rozwiązania znalezione w Google. [Jedno](https://github.com/jupeter/clean-code-php) z nich prezentuje się następująco:
```php
abstract class Shape {
    private $width, $height;
    
    abstract public function getArea();
    
    public function setColor($color) {
        // ...
    }
    
    public function render($area) {
        // ...
    }
}

class Rectangle extends Shape {
    public function __construct {
    parent::__construct();
        $this->width = 0;
        $this->height = 0;
    }
    
    public function setWidth($width) {
        $this->width = $width;
    }
    
    public function setHeight($height) {
        $this->height = $height;
    }
    
    public function getArea() {
        return $this->width * $this->height;
    }
}

class Square extends Shape {
    public function __construct {
        parent::__construct();
        $this->length = 0;
    }
    
    public function setLength($length) {
        $this->length = $length;
    }
    
    public function getArea() {
        return $this->length * $this->length;
    }
}

function renderLargeRectangles($rectangles) {
    foreach($rectangle in $rectangles) {
        if ($rectangle instanceof Square) {
            $rectangle->setLength(5);
        } else if ($rectangle instanceof Rectangle) {
            $rectangle->setWidth(4);
            $rectangle->setHeight(5);
        }
        
        $area = $rectangle->getArea(); 
        $rectangle->render($area);
    });
}

$shapes = [new Rectangle(), new Rectangle(), new Square()];
renderLargeRectangles($shapes);
```
Ten kod zawiera kilka błędów, wygląda jakby autor przepisywał rozwiązanie z innego języka, bo znajdują się tam wstawki składni niepoprawnej w php. Problem naruszenia LSP został niby zlikwidowany, jednak pojawia się kilka wątpliwości co do takiej implementacji. W abstrakcyjnej klasie mamy właściwości width oraz height, jednak nie wszystkie figury posiadają takie atrybuty, w okręgu mamy na przykład promień. Nawet w samym kwadracie używana jest inna właściwość – length, więc jaki jest sens pakowania tego do Shape?

W funkcji renderLargeRectangles() mamy sprawdzanie typu obiektu. W ten sposób moglibyśmy również załatwić problem w sytuacji, gdy Square dziedziczy po Rectangle, wystarczyłoby sprawdzanie czy obiekt jest typu Square i wiadomo jakiej wartości pola się spodziewać. Ten fragment również narusza zasadę [OCP](https://sarvendev.com/2017/08/2-solid-openclosed-principle/), w przypadku gdy dodamy kolejną figurę trzeba będzie dodać kolejnego ifa. Wiem, że ta funkcja jest przykładowa i ma prezentować działanie, jednak zdaję sobie sprawę, że mogłaby ona znaleźć się gdzieś w systemie, więc warto o tym wspomnieć.

## Przykład poprawnego kodu
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
```php
<?php

/**
 * Class Square
 */
final class Square implements AreaCalculableInterface
{
    /**
     * @var float
     */
    private $width;

    /**
     * Square constructor.
     * @param float $width
     */
    public function __construct(float $width)
    {
        $this->width = $width;
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
    public function calcArea(): float
    {
        return $this->width * $this->width;
    }
}
```
Kluczem do rozwiązania tego problemu jest immutability, czyli niezmienność. Odpowiednie wartości boków ustawiane są przy tworzeniu obiektu, późniejsza ich modyfikacja nie jest możliwa, gdyż nie mamy odpowiednich setterów. Dodatkowo te klasy(Rectangle, Square) raczej nie powinny być dziedziczone, więc ważne jest dodanie słowa kluczowego final, tak aby dać sygnał innym deweloperom, że nie powinni dziedziczyć z tych klas.

W javascripcie taka implementacja będzie stanowiła kłopot, bo tam nadal nie ma modyfikatorów dostępu, więc brak setterów nie rozwiązuje sprawy.  Może masz na to jakiś sposób? Zachęcam do podzielenia się 😉
