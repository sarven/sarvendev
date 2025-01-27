---
title: Profilowanie i optymalizacja
date: 2017-07-21
categories: [Programming]
permalink: /pl/2017/07/profilowanie-i-optymalizacja/
redirect_from:
  - /2017/07/profilowanie-i-optymalizacja/
image:
  path: /assets/img/2017-07-21/blackfire.png
---
Wczoraj na blogu opisywałem rozwiązanie zadania „[Misja Gynvaela 008](https://sarvendev.com/2017/07/misja-gynvaela-008/)”, jednak o ile udało się wygenerować mapę i odczytać hasło, to czas przetwarzania plików był stanowczo zbyt długi. Postanowiłem przyjrzeć się temu ponownie i postarać się coś przyśpieszyć.

## Profilowanie
Pierwszym krokiem będzie sprawdzenie za pomocą narzędzia [Blackfire](https://blackfire.io/) co stanowi tzw. wąskie gardło, czyli jaka czynność trwa najdłużej. Ograniczyłem więc główną pętlę klasy Solver, tak aby przetwarzała ona jedynie 10 plików, następnie odpaliłem blackfire.

```
blackfire run php solver.php
```

![Profile](/assets/img/2017-07-21/profile.png)

Okazało się, że przetworzenie 10 plików trwa aż 6.31s, a wąskie gardło stanowi funkcja imagejpeg wywoływana aż 275 razy.

## Optymalizacja
Nie ma potrzeby, aby plik mapy był zapisywany po każdym dodaniu punktu. Przerobiłem więc kod tak, aby przed główną pętlą w klasie Solver utworzyć obraz, a następnie po jej zakończeniu go zapisać. Dokładne zmiany można sprawdzić tutaj. Kolejne sprawdzenie za pomocą narzędzia blackfire przyniosło już zadowalający rezultat.

![Profile](/assets/img/2017-07-21/profile2.png)

Obecnie przetworzenie wszystkich danych i narysowanie mapy zajmuje jakieś 80-90 sekund.

```
Parsed 187812 files in 83 seconds
```
