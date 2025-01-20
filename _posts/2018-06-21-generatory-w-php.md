---
title: Generatory w php
date: 2018-06-21
categories: [Programming, PHP]
permalink: /pl/2018/06/generatory-w-php/
---
Generatory zostały dodane stosunkowo dawno, bo jeszcze w wersji php 5.5. Jednak wydaje się, że są rzadko spotykane w różnych projektach, a może jednak czasem warto mieć świadomość ich istnienia, gdyż idealnie dopasowują się do niektórych problemów.

## Czym są generatory?
Generatory są funkcjami, które pozwalają w wydajny sposób iterować po dużych zbiorach danych. Różnicą w składni w stosunku do zwykłej funkcji jest słowo kluczowe yield, którego użycie na pierwszy rzut oka przypomina użycie return. Myślę, że najlepiej zobrazować to na konkretnych przykładach.

## Przetwarzanie dużej tablicy
Poniższy kod wygląda pewnie dosyć niepozornie, prosta pętla konstruująca tablicę.  Sprawdzenie kodu w profilerze (blackfire) nie zwraca nic niepokojącego, użycie pamięci 33.6MB da się przeżyć.

```php
<?php

function getData() {
    $data = [];

    for ($i = 0; $i < 1000000; $i++) {
        $data[] = $i;
    }

    return $data;
}

$data = getData() ;

foreach ($data as $item) {

}
```
![Blackfire Profile 1](/assets/images/2018-06-21/profile1.png)
Jednak po zwiększeniu liczby iteracji do 10 000 000 wynik  z blackfire wydaje się już nieco niepokojący. 537 MB to jest stanowczo za dużo jak na tak prostą operację. Gdzie leży problem?
![Blackfire Profile 2](/assets/images/2018-06-21/profile2.png)
Odpowiedź jest bardzo prosta, funkcja getData() musi całą tablicę najpierw skonstruować, a następnie dopiero po zwróconej tablicy iterujemy. Tutaj własnie znajdują zastosowanie Generatory.
```php
<?php

function getData() {
    for ($i = 0; $i < 10000000; $i++) {
        yield $i;
    }
}

$data = getData();

foreach ($data as $item) {

}
```
Przerobienie funkcji, aby korzystała z generatora jest jak widać bardzo proste. Warto zauważyć, że tym razem tablica nie jest od razu konstruowana w pamięci. Funkcja getData zwraca obiekt klasy [Generator](http://php.net/manual/en/class.generator.php), która implementuje interfejs [Iteratora](http://php.net/manual/en/class.iterator.php), co pozwala na proste przetwarzanie w pętli foreach.

Wynik z blackfire prezentuje się następująco:
![Blackfire Profile 3](/assets/images/2018-06-21/profile3.png)

Czas może trochę dłuższy, natomiast różnica zużycia pamięci w stosunku do poprzedniego rozwiązania jest kolosalna.

## Przetwarzanie dużego pliku
Kolejnym dobrym przykładem zastosowania generatora może być przetwarzanie dużego pliku. W tym celu wygenerujmy plik do przetworzenia:
```php
<?php

$file = fopen('file.txt', 'wb');

for ($i = 0; $i < 1000000; $i++) {
    fwrite($file, random_bytes(1024));
}

fclose($file);
```

Powyższy kod tworzy plik o rozmiarze około 1GB.

Teraz próba przetworzenia tego pliku linia po linii:
```php
<?php

function getLines($file) {
    $lines = [];

    while ($line = fgets($file)) {
        $lines[] = $line;
    }

    return $lines;
}
$file = fopen('file.txt', 'rb');
$lines = getLines($file);
fclose($file);

foreach ($lines as $line) {

}
```
![Blackfire Profile 4](/assets/images/2018-06-21/profile4.png)
Wynik z profilera pokazuje problem, przy głupim przetworzeniu pliku potrzebujemy ponad 1GB pamięci.

```php
<?php

function getLines($file) {
    while ($line = fgets($file)) {
        yield $line;
    }
}
$file = fopen('file.txt', 'rb');
$lines = getLines($file);
//fclose($file);

foreach ($lines as $line) {

}

fclose($file);
```

Napisanie kodu z użyciem Generatora jest jak widać równie proste co poprzednio. Z tym, że należy zauważyć jedną rzecz. Uchwyt do pliku zostaje zamknięty dopiero po przetworzeniu generatora, jeśli zamkniemy go wcześniej w tym miejscu co zakomentowałem fclose, dostaniemy Warning:

```php
PHP Warning:  fgets(): supplied resource is not a valid stream resource
```

Wynik z blackfire pokazuje znaczącą różnice w użyciu pamięci:

![Blackfire Profile 5](/assets/images/2018-06-21/profile5.png)

Oczywiście oba przykłady sprowadzają się do tego samego, czyli do iterowania po dużym zbiorze danych bez budowania go w całości w pamięci.

## Szybkość
Wróćmy jeszcze do poprzedniego przykładu i tablicy z milionem elementów. W przypadku normalnej funkcji czas wykonania to ~32ms i ~34MB pamięci, natomiast w przypadku generatora ~7.6s i ~39KB. Myślę, że warto tutaj zwrócić uwagę na fakt, że 34MB to nie jest wcale tak dużo, a ~7.6s to jednak już trochę jest, więc pojawia się pytanie, który sposób wybrać?

Oczywiście dobrą odpowiedzią jak to często bywa w przypadku programowania jest to zależy. W przypadku gdy wiemy, że tych danych do przetworzenia będzie na tyle, że zasobów nam nie braknie to wykorzystujemy pierwszy sposób. Jeśli jednak nie jesteśmy w stanie oszacować ilości danych to lepiej zastosować drugi sposób. Może użytkownik nie musi na to czekać i to przetwarzanie lepiej zrealizować w tle? Wszystko zależy od problemu jaki staramy się rozwiązać.

## Aktualizacja
**Okazuje się, że blackfire podaje niepoprawne czasy. W komentarzu podany jest przykład. W rzeczywistości czas wykonywania kodu z generatorem powinien być tylko minimalnie wyższy, przy dużo mniejszym użyciu pamięci.**
