---
title: Array destructuring
date: 2018-06-07
categories: [Programming, PHP]
permalink: /pl/2018/06/array-destructing-php-7-1/
redirect_from:
  - /2018/06/array-destructing-php-7-1/
---
Kiedyś pisałem dosyć dużo kodu w javascript (es6). Wykorzystywałem różne możliwość języka, których później brakowało mi w php. Jedną z nich było tzw. destructuring assignment.

## Przykład w ES6
```javascript
const user = [1, 'name'];
const [id, name] = user;
console.log(id); // 1
console.log(name); // name
```

W php 5.6 też dało się coś takiego zrobić, ale tylko z użyciem funkcji list(). Natomiast w 7.1 już to według mnie wygląda bardziej przyjemnie.

## PHP 5.6
```php
$user = [1, 'name'];
list($id, $name) = $user;
var_dump($id); // 1
var_dump($name); // name
```

## PHP 7.1
```php
$user = [1, 'name'];
[$id, $name] = $user;
var_dump($id); // 1
var_dump($name); // name
```

## Pętle

```php
$user = ['id' => 1, 'name' => 'user'];
['id' => $id, 'name' => $name] = $user;
var_dump($id); // 1
var_dump($name); // name
```

Teraz rozważmy trochę szerszy przykład. Załóżmy, że mamy taką tablicę z danymi użytkowników.

```php
$users = [
    ['id' => 1, 'name' => 'user', 'email' => 'user@test.com'],
    ['id' => 2, 'name' => 'user2', 'email' => 'user2@test.com'],
    ['id' => 3, 'name' => 'user3', 'email' => 'user3@test.com']
];
```

Chcemy teraz przetworzyć w pętli nazwę i email użytkownika. Można to zrobić w następujący sposób.
```php
$users = [
    ['id' => 1, 'name' => 'user', 'email' => 'user@test.com'],
    ['id' => 2, 'name' => 'user2', 'email' => 'user2@test.com'],
    ['id' => 3, 'name' => 'user3', 'email' => 'user3@test.com']
];
foreach ($users as ['name' => $name, 'email' => $email]) {
    var_dump($name); // user, user2, user3
    var_dump($email); // user@test.com, user2@test.com, user3@test.com
}
```

## Zagnieżdżenia

Dodatkowo możliwe jest również wydobywanie zagnieżdżonych elementów.

```php
$user = [
    'id' => 1,
    'name' => 2,
    'address' => [
        'city' => 'Krakow',
        'street' => 'Wielicka'
    ]
];

['address' => ['city' => $city]] = $user;

var_dump($city); // Krakow
```
