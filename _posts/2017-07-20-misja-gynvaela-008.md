---
title: Misja Gynvaela 008
date: 2017-07-20
categories: [Programming, Ctf]
permalink: /pl/2017/07/misja-gynvaela-008/
---
Ostatnio wpadło mi w ręce zadanie podane przez Gynvaela na jednym z ostatnich streamów, które wydało mi się na tyle ciekawe, że postanowiłem je rozwiązać oraz opisać na blogu. Zapraszam więc do dalszego czytania!

```
MISJA 008 goo.gl/gg4QcA DIFFICULTY: █████████░ [9/10]
┅┅┅┅┅┅┅┅┅┅┅┅┅┅┅┅┅┅┅┅┅┅┅┅┅┅┅┅┅┅┅┅┅┅┅┅┅┅┅┅┅┅┅┅┅┅┅┅┅┅┅┅┅┅┅┅┅┅┅┅┅┅┅┅┅┅┅┅┅┅┅┅┅┅┅┅┅┅┅┅┅

Otrzymaliśmy dość nietypową prośbę o pomoc od lokalnego Instytutu Archeologii.
Okazało się, iż podczas prac remontowych studni w pobliskim zamku odkryto
niewielki tunel. Poproszono nas abyśmy skorzystali z naszego autonomicznego
drona wyposażonego w LIDAR (laserowy skaner odległości zamontowany na obracającej
się platformie) do stworzenia mapy tunelu.

Przed chwilą dotarliśmy na miejsce i opuściliśmy drona do studni. Interfejs I/O
drona znajduje się pod poniższym adresem:

http://gynvael.coldwind.pl/misja008_drone_output/

Powodzenia!

--

Korzystając z powyższych danych stwórz mapę tunelu (i, jak zwykle, znajdź tajne
hasło). Wszelkie dołączone do odpowiedzi animacje są bardzo mile widziane.

Odzyskaną wiadomość (oraz mapę) umieśc w komentarzu pod tym video :)
Linki do kodu/wpisów na blogu/etc z opisem rozwiązania są również mile widziane!

HINT 1: Serwer może wolno odpowiadać a grota jest dość duża. Zachęcam więc do
cache'owania danych na dysku (adresy skanów są stałe dla danej pozycji i nigdy
nie ulegają zmianie).

HINT 2: Hasło będzie można odczytać z mapy po odnalezieniu i zeskanowaniu
centralnej komnaty.

P.S. Rozwiązanie zadania przedstawię na początku kolejnego livestreama.
```

## Pobieranie wszystkich plików z danymi
Postanowiłem, że zabiorę się do tego zaczynając od pobrania wszystkich plików z danymi do siebie, tak aby móc na nich pracować lokalnie. Założyłem sobie również, że napiszę kod wydzielając odpowiedzialności na poszczególne klasy, czyli postępując jednak w trochę inny sposób niż takie zadania zwykle się rozwiązuje tj. pisząc na szybko jakiś „jednoplikowiec”, który zassie odpowiednie dane oraz później ewentualnie drugi na przetworzenie punktów i narysowanie mapy.

Klasą odpowiedzialną za pobranie plików jest Crawler, który przyjmuje następujące zależności:
- FileHandler – obsługa plików (odczyt, zapis etc.)
- Output – obsługa wyjścia (czyli to co wyrzuca konsola – komunikaty etc.)
- DataParser – parsowanie danych, czyli przekształcanie treści pliku na użyteczny obiekt

## Jak działa pobieranie plików?
Początkowo wczytywany jest [plik startowy](http://gynvael.coldwind.pl/misja008_drone_io/scans/68eb1a7625837e38d55c54dc99257a17.txt):
```
SCAN DRONE v0.17.3
32 34
10.015625
10.156250
9.578125
10.000000
9.343750
9.343750
10.015625
9.578125
10.156250
10.000000
10.156250
10.656250
12.703125
12.453125
10.453125
10.406250
10.656250
11.171875
11.000000
11.171875
11.703125
11.562500
10.890625
10.890625
11.562500
10.656250
11.171875
11.015625
11.171875
10.656250
10.406250
10.453125
10.453125
10.406250
9.578125
10.156250
MOVE_EAST: 8deb0b192017a1f44a39a4a93501c985.txt
MOVE_WEST: 92906bed3a133722a3c3caf62afd9cff.txt
MOVE_SOUTH: db743fbf7f49f9bfa2d48edd613af803.txt
MOVE_NORTH: 1ae0fc9499130c246641965299adabaf.txt
```
Następnie treść zostaje przetworzona, tak aby wydostać z niej nazwy kolejnych plików, które to jeśli nie zawierają nazwy „not_possible”, nie zawierają się w przetworzonych już plikach oraz nie są już zakolejkowane, dodawane są do kolejki. Na potrzeby odpowiedniej organizacji pobierania zaimplementowałem dwie proste struktury danych: kolejkę oraz stos.

**Kolejka**
```php
<?php

namespace LIDAR\Data;

/**
 * Class Queue
 * @package LIDAR\Data
 */
final class Queue implements QueueInterface
{
    /**
     * @var string[]
     */
    private $items;

    /**
     * Queue constructor.
     */
    public function __construct()
    {
        $this->items = [];
    }

    /**
     * @return string
     */
    public function get(): string
    {
        if ($this->isEmpty()) {
            throw new \RuntimeException('Queue is empty.');
        }

        return array_shift($this->items);
    }

    /**
     * @param string $fileName
     */
    public function enqueue(string $fileName): void
    {
        $this->items[] = $fileName;
    }

    /**
     * @param string $fileName
     */
    public function dequeue(string $fileName): void
    {
        if ($key = array_search($fileName, $this->items)) {
            unset($this->items[$key]);
        }
    }

    /**
     * @return bool
     */
    public function isEmpty(): bool
    {
        return empty($this->items);
    }

    /**
     * @param string $fileName
     * @return bool
     */
    public function contain(string $fileName): bool
    {
        return in_array($fileName, $this->items);
    }
}
```

**Stos**
```php
<?php

namespace LIDAR\Data;

/**
 * Class Stack
 * @package LIDAR\Data
 */
final class Stack implements StackInterface
{
    /**
     * @var array
     */
    private $items;

    /**
     * Stack constructor.
     */
    public function __construct()
    {
        $this->items = [];
    }

    /**
     * @param string $fileName
     */
    public function push(string $fileName): void
    {
        $this->items[] = $fileName;
    }

    /**
     * @param string $fileName
     * @return bool
     */
    public function contain(string $fileName): bool
    {
        return in_array($fileName, $this->items);
    }
}
```
Z racji iż mój internet do demonów prędkości nie należy, postanowiłem odpalić crawlera na serwerze, co pozwoliło przyśpieszyć trochę całą operację.

![Terminal](/assets/img/2017-07-20/terminal.png)

Plików jak się okazało jest aż 187 812, które zajmują około 90MB.

## Przetwarzanie plików oraz rysowanie mapy
Za rozwiązanie zadania odpowiada klasa Solver, która przyjmuje następujące zależności:
- FileHandler – obsługa plików (odczyt, przeglądanie katalogu etc.)
- Output – obsługa wyjścia (czyli to co wyrzuca konsola – komunikaty etc.)
- DataParser – parsowanie danych, czyli przekształcanie treści pliku na użyteczny obiekt
- Drawer – rysowanie mapy
W tym wypadku doszła jedna dodatkowa zależność – Drawer. Postanowiłem jednak nie przeglądać plików jak poprzednio, lecz po prostu mając je wszystkie na dysku, odczytywać je po kolei z katalogu. Na podstawie danych z każdego pliku, czyli skanów z systemu LIDAR jak mówi nam to treść zadania, mając odległość od drona przy danym kącie liczymy punkty ścian jaskini korzystając z trygonometrii. Odpowiada za to klasa PointCalculator:

```php
<?php

namespace LIDAR\Point;

use LIDAR\Entity\Data;
use LIDAR\Entity\Distance;
use LIDAR\Entity\Point;

/**
 * Class PointCalculator
 * @package LIDAR\Point
 */
final class PointCalculator implements PointCalculatorInterface
{
    /**
     * @param Data $data
     * @return Point[]
     */
    public function calc(Data $data): array
    {
        $points = [];

        foreach ($data->getDistances() as $distance) {
            $point = $this->calcPoint($distance, $data->getPoint());

            if ($point) {
                $points[] = $point;
            }
        }

        return $points;
    }

    /**
     * @param Distance $distance
     * @param Point $point
     * @return Point|null
     */
    private function calcPoint(Distance $distance, Point $point): ?Point
    {
        if ($distance->getDistance() === -1.0) {
            return null;
        }

        $angle = $distance->getAngle() * M_PI / 180;
        $x = round(sin($angle) * $distance->getDistance() + $point->getX());
        $y = round(-cos($angle) * $distance->getDistance() + $point->getY());

        return (new Point())
            ->setX($x)
            ->setY($y)
        ;
    }
}
```
## Problemy
Podany kod działa, jednak działa dosyć wolno, co znacząco utrudnia przetworzenie wszystkich plików. Jednak po sprawdzeniu jak będzie wyglądała mapa po przetworzeniu jedynie 10 000 plików, okazało się, że rozwiązanie jest widoczne.

**Cała mapa** 

![Map](/assets/img/2017-07-20/map.png)

**Hasło**

![Password](/assets/img/2017-07-20/pass.png)

Całość rozwiązania znaleźć można [tutaj](https://github.com/sarven/misja-gynvaela-008).

Postaram się jeszcze pobawić programowaniem współbieżnym, co powinno przyśpieszyć działanie kodu, tak aby dało się przetworzyć wszystko w sensownym czasie. Jeśli masz inny sposób lub wiesz co go tak zwalnia to czekam na komentarz 😉
