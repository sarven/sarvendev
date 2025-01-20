---
title: Active record (Eloquent) vs Data mapper (Doctrine)
date: 2018-01-15
categories: [Programming, Patterns]
permalink: /pl/2018/01/active-record-eloquent-vs-data-mapper-doctrine/
image:
  path: /assets/img/2018-01-15/featured.png
---
W większości tworzonych systemów trzeba gdzieś i w jakiś sposób zapisywać dane. ORM (Object-Relational Mapping), czyli mapowanie obiektowo-relacyjne jest sposobem odwzorowania systemu na bazę danych. ORM jest warstwą pomiędzy bazą danych, a aplikacją. Zajmuje się tworzeniem, aktualizowaniem, odczytywaniem oraz usuwaniem danych.

![ORM](/assets/img/2018-01-15/orm.png)

Jak widać na tej ilustracji wzorce DataMapper oraz ActiveRecord należą do warstwy Data access, która przykładowo podczas pobierania danych z bazy ma za zadanie zwrócić odpowiednio uzupełniony obiekt zamiast zwracania „czystego” rekordu.

## Active record
Instancja modelu we wzorcu Active record jest ściśle powiązana z pojedynczym rekordem w bazie danych. Konkretna implementacja modelu dziedziczy po modelu bazowym. Nie trzeba ustawiać właściwości obiektu, bo te odpowiadają kolumnom z bazy danych. Przykładowo w Eloquent (ORM od Laravela) w modelu bazowym Illuminate\Database\Eloquent\Model wykorzystywana jest magiczna metoda __get() i jeśli chcemy z modelu Usera wyciągnąć email to pisząc w kodzie coś takiego:

```php
echo $user->email;
```
Pod spodem wykonuje się odpowiednio:
```php
public function __get($key)
{
    return $this->getAttribute($key);
}
```
oraz:

```php
public function getAttribute($key)
{
    if (! $key) {
        return;
    }
    // If the attribute exists in the attribute array or has a "get" mutator we will
    // get the attribute's value. Otherwise, we will proceed as if the developers
    // are asking for a relationship's value. This covers both types of values.
    if (array_key_exists($key, $this->attributes) ||
        $this->hasGetMutator($key)) {
        return $this->getAttributeValue($key);
    }
    // Here we will determine if the model base class itself contains this given key
    // since we don't want to treat any of those methods as relationships because
    // they are all intended as helper methods and none of these are relations.
    if (method_exists(self::class, $key)) {
        return;
    }
    return $this->getRelationValue($key);
}
```

Więc model trzyma wczytany do tablicy rekord z bazy danych i zwraca odpowiednie wartości.

Jeśli chcemy zapisać model to wywołujemy przykładowo:

```php
$user->save();
```

Tutaj na pewno można zauważyć, że zostaje złamana [zasada pojedynczej odpowiedzialności](https://sarvendev.com/2017/08/1-solid-single-responsibility-principle/). Obiekt zapisuje sam siebie. 
Jednak to nie jest tak, że Active record jest zły. Wszystko zależy od tego jaką aplikację projektujemy. Jeśli większość to proste CRUDy, brak logiki biznesowej to jak najbardziej można z niego korzystać. Przy tym wzorcu sprawdzi się również podejście Database first, czyli najpierw projekt bazy danych, później kod. Kolejną zaletą jest na pewno szybkość pisania, przy wzorcu Data mapper musimy zadbać o więcej rzeczy, co przekłada się na dłuższy czas realizacji. Więc naturalnie Active Record jest lepszym wyborem przy tworzeniu na przykład prototypów.

## Data mapper
Model we wzorcu Data mapper jest kompletnie odseparowanym bytem,  który nie ma świadomości w jaki sposób, ani gdzie jest zapisywany. W przypadku złożonych modeli jest to na pewno sporą zaletą. Dzięki temu nie jest zaciemnianie to co najważniejsze, czyli logika biznesowa.

Wypełniane są również rzeczywiste właściwości obiektu, więc model automatycznie staje się bardziej rozbudowany, bo musimy wszystkie zaimplementować i odpowiednio oznaczyć.

```php
<?php
 
use Doctrine\ORM\Mapping AS ORM;
 
/**
 * @ORM\Entity
 * @ORM\Table(name="users")
 */
class User
{
    /**
    * @ORM\Id
    * @ORM\GeneratedValue
    * @ORM\Column(type="integer")
    */
    private $id;

    /**
    * @ORM\Column(type="string")
    */
    private $email;

    ...
}
```
Zapisywanie modelu wygląda mniej więcej w ten sposób:

```php
$entityManager->persist($user);
$entityManager->flush();
```

W przypadku jeśli model jest anemiczny tzn. zawiera jedynie settery i gettery dla odpowiednich elementów, a nie posiada żadnych zachowań to lepszym wyborem jest Active Record. Oczywiście tutaj należy dobrze podejść do projektowania, gdyż nawet z najbardziej rozbudowanego modelu można zrobić anemiczny przenosząc odpowiedzialności do serwisów, kontrolerów czy w jakiekolwiek inne miejsce.

Projektowanie w przypadku tego wzorca powinno być zaczynane od kodu, a nie od bazy danych.


