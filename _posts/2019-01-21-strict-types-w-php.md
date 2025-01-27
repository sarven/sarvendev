---
title: Strict types w php
date: 2019-01-21
categories: [Programming, PHP]
permalink: /pl/2019/01/strict-types-w-php/
redirect_from:
  - /2019/01/strict-types-w-php/
---
Od wersji php 7.0 mamy możliwość używania deklaracji typów w parametrach funkcji, metod, a od 7.1 również możemy określić typ wartości zwracanej. Jednak okazuje się, że nie do końca działa to w sposób jaki moglibyśmy oczekiwać, a często wartości są po prostu w miarę możliwości konwertowane do pożądanego typu. Natomiast konwersja często może być efektem niepożądanym, dlatego warto wiedzieć co można z tym zrobić.

## Type hints
```php
<?php

function add (int $a, int $b): int {
    return $a + $b;
}

add(1, 2); // 3
add(1, true); // 2
add(1, 2.3); // 3
add('1', 2); // 3
```

```php
<?php

declare(strict_types=1);

function add (int $a, int $b): int {
    return $a + $b;
}

add(1, 2); // 3
add(1, true); // PHP Fatal error:  Uncaught TypeError: Argument 2 passed to add() must be of the type integer, boolean given
add(1, 2.3); // PHP Fatal error:  Uncaught TypeError: Argument 2 passed to add() must be of the type integer, float given
add('1', 2); // PHP Fatal error:  Uncaught TypeError: Argument 1 passed to add() must be of the type integer, string given
```
Rozważmy dwa przykłady jak powyżej. W obu używana jest prosta funkcja dodająca dwie liczby całkowite, jednak w drugim deklarujemy używanie strict types. Strict types to jak sama nazwa wskazuje ścisłe trzymanie się określonych typów. Domyślnie php stara się dopasować typ danej wartości do typu wartości wymaganej w danej funkcji. Dopiero po użyciu deklaracji:
```php
declare(strict_types=1);
```
php rygorystycznie przestrzega narzuconych typów. Jeśli miał być integer, a przekazany został float to od razu rzucany jest fatal error.

Używanie strict types pozwala na pisanie bardziej przewidywalnego kodu. Konwersja true na 1 lub float na int nie jest raczej czymś pomocnym, a może prowadzić do wystąpienia problemów. Jeśli te przykłady to mało to warto jeszcze rozważyć następujący:

```php
add('1 asd', 2);
```

W celu zapewnienia poprawności i przewidywalności taka funkcja powinna rzucić fatal error, bo pierwszy argument to kompletna bzdura z punktu widzenia tej funkcji. Natomiast bez deklaracji strict types funkcja zwróci nam po prostu 3, co jest kompletnie bez sensu. Dodatkowo rzucany jest notice:
```php
PHP Notice:  A non well formed numeric value encountered
```
ale o jego przegapienie bardzo łatwo. Lepiej, żeby aplikacja przestała działać niż zwracała niepoprawne wyniki, dlatego zalecałbym stosowanie strict types.

To samo dotyczy deklaracji typu wartości zwracanych, ale rządzą tym te same zasady, więc pominę ich opisywanie.

## Wymuszanie deklaracji
Deklaracje w każdym pliku można wymusić korzystając z codesniffera.
```
composer require squizlabs/php_codesniffer
composer require slevomat/coding-standard
```
Następnie do pliku phpcs.xml dodajemy:
```xml
<?xml version="1.0"?>
<ruleset name="name-of-your-project">
    <config name="installed_paths" value="vendor/slevomat/coding-standard"/>
    <rule ref="vendor/slevomat/coding-standard/SlevomatCodingStandard/Sniffs/TypeHints/DeclareStrictTypesSniff.php" />
</ruleset>
```
Uruchamianie wygląda tak:

```
phpcs --standard=phpcs.xml
```

## Dlaczego nie jest to ustawione domyślnie lub nie ma możliwości globalnego ustawienia?
Obecnie taka deklaracja nie może być ustawiona domyślnie, ani też globalnie dla całego projektu. Ustawianie w pliku pozwala zachować możliwość używania kodu, który nie jest napisany zgodnie ze script types. W przypadku ustawienia tej opcji globalnie, przy instalacji jakiejkolwiek zależności musielibyśmy zastanawiać się czy będzie ona działała prawidłowo  z naszymi ustawieniami.

Dlatego jedyną sensowną opcją na dzień dzisiejszy jest używanie tego w projekcie posiłkując się ustawieniem odpowiednio jakiegoś narzędzia (np. php_codesniffer jak wyżej), które będzie wymuszało na nas używanie tej deklaracji w plikach, w  których powinno się ono znaleźć.


