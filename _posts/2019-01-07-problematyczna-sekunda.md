---
title: Problematyczna sekunda
date: 2019-01-07
categories: [Interesting]
permalink: /pl/2019/01/problematyczna-sekunda/
image:
  path: /assets/img/2019-01-07/featured.png
---
Obecnie każdy z nas posiada kilka możliwości sprawdzenia aktualnego czasu. Mamy telefony, zegarki, komputery, telewizory, lodówki, kuchenki itd. Każdy z nas wie ile to jest rok i skąd to się wzięło. Wiemy o strefach czasowych, czasie UTC. Wszystko wydaje się proste, jednak z punktu widzenia systemów informatycznych jest wiele niespodzianek, na które możemy się natknąć, co niestety powoduje mierzenie się z problemami trudnymi do zanalizowania, bo występują one bardzo rzadko.


Problemów z czasem, które występują w różnych systemach informatycznych jest bardzo wiele w zależności od przeznaczenia takiego systemu. Głównie dotyczą one dokładności pomiaru czasu oraz synchronizacji zegarów pomiędzy węzłami w systemach rozproszonych.  W tym artykule skupię się jednak na problematycznej sekundzie przestępnej.

## Sekunda przestępna
Początkowo sekunda została zdefiniowana jako 1/86400 część doby. Następnie w roku 1960 definicja została zmieniona na 1/31 556 925,9747 część roku zwrotnikowego. Jednak okazało się, że obieg Ziemi wokół Słońca nie zawsze trwa tyle samo, a często występują drobne różnice spowodowane pływami, ruchem mas wewnątrz planety jak i również trzęsieniami ziemi. Dlatego wartość sekundy w roku 1967 została zdefiniowana w całkowicie inny sposób:

> Jest to czas równy 9 192 631 770 okresom promieniowania odpowiadającego przejściu między dwoma poziomami F = 3 i F = 4 struktury nadsubtelnej stanu podstawowego S1/2 atomu cezu 133Cs (powyższa definicja odnosi się do atomu cezu w spoczynku w temperaturze 0 K) .

Najdokładniejszymi zegarami są zegary atomowe, które właśnie mierzą czas na podstawie szybkości przejścia elektronów pomiędzy dwiema powłokami atomu. Pozostała jednak kwestia wyrównania różnicy pomiędzy wartościami zmierzonymi przez zegary atomowe, a faktycznym czasem obiegu Ziemi wokół Słońca.

W tym celu powstało pojęcie sekundy przestępnej. Międzynarodowa Służba Ruchu Obrotowego Ziemi i Systemów Odniesienia (IERS (International Earth Rotation and Reference Systems Service)) decyduje kiedy taka sekunda powinna zostać dodana. Dodawana jest zwykle ostatniego dnia czerwca lub grudnia. Możliwe jest również odjęcie jednej sekundy, jednak na dzień dzisiejszy taka sytuacja jeszcze nie wystąpiła.

Lista dodanych sekund:
> (+11s) 30 czerwca 1972  
> (+12s) 31 grudnia 1972  
> (+13s) 31 grudnia 1973   
> (+14s) 31 grudnia 1974  
> (+15s) 31 grudnia 1975  
> (+16s) 31 grudnia 1976  
> (+17s) 31 grudnia 1977  
> (+18s) 31 grudnia 1978  
> (+19s) 31 grudnia 1979  
> (+20s) 30 czerwca 1981  
> (+21s) 30 czerwca 1982  
> (+22s) 30 czerwca 1983  
> (+23s) 30 czerwca 1985  
> (+24s) 31 grudnia 1987  
> (+25s) 31 grudnia 1989  
> (+26s) 31 grudnia 1990  
> (+27s) 30 czerwca 1992  
> (+28s) 30 czerwca 1993  
> (+29s) 30 czerwca 1994  
> (+30s) 31 grudnia 1995  
> (+31s) 30 czerwca 1997  
> (+32s) 31 grudnia 1998  
> (+33s) 31 grudnia 2005  
> (+34s) 31 grudnia 2008  
> (+35s) 30 czerwca 2012  
> (+36s) 30 czerwca 2015  
> (+37s) 31 grudnia 2016  

## Czas, a systemy informatyczne
Czas występuje w zasadzie w większości projektów. Musimy mieć możliwość określenia kiedy wystąpiła dana akcja użytkownika, kiedy ma zostać wysłany dany email. Ważne może być również określenie czy limit czasu żądania już upłynął. W przypadku jednej maszyny jeszcze nie jest to jakoś bardzo skomplikowane. Jednak w przypadku systemów rozproszonych, gdzie mamy do czynienia z wieloma węzłami, replikacją itd. sytuacja jest bardziej skomplikowana. Czasy sprzętowe podawane przez zwykle oscylator kwarcowy nie są do końca dokładne i mogą być różnice pomiędzy węzłami. Gdy kolejność zdarzeń jest istotna, rozbieżność wskazań zegarów w węzłach jest nie do przyjęcia. Dlatego stosowany jest NTP (Network Time Protocol – protokół synchronizacji czasu), który dopasowuje wskazywany przez daną maszynę czas do czasu podawanego przez grupę serwerów.

W komputerach wyróżnia się przynajmniej dwa zegary: zegary czasu rzeczywistego oraz zegary monotoniczne.

## Zegary czasu rzeczywistego
Zwracają po prostu aktualną datę i czas. W systemach opartych na Uniksie zegar bazuje na dacie 1 stycznia 1970 00:00:00 i od tego momentu zliczana jest liczba sekund. Z tymi zegarami jest taki problem, że są synchronizowane przez NTP, więc mogą wystąpić jakieś przeskoki w przód lub w tył. Dlatego też kompletnie nie nadają się do mierzenia upływu czasu np. sprawdzania czy upłynął już limit czasu żądania.

## Zegary monotoniczne
Zegary monotoniczne są zwykle bardzo precyzyjne, podają wartości co do mikrosekundy lub też nawet dokładniej. Wykorzystywane jest tutaj po prostu zwiększanie danego licznika, który może być uruchomiony po starcie systemu. NTP w ich przypadku może jedynie dostrajać szybkość działania, przyśpieszyć lub spowolnić dany zegar. Nie ma więc tutaj żadnych przeskoków, dlatego też mierzenie za ich pomocą upływu czasu jest raczej akceptowalne.

## Konsekwencje złej synchronizacji
Skutkiem złej synchronizacji lub brakiem synchronizacji zegarów może być utrata danych. Lepszy jest system, który wyłoży się całkowicie niż działa niezauważony nieprawidłowo i psuje dane. Dlatego w sytuacji, gdy polegamy na zegarach, muszą one być odpowiednio synchronizowane i monitorowane, jeśli któryś z nich działa nieprawidłowo powinien zostać uznany za całkowicie niesprawny.

## Porządkowanie zdarzeń
Przykładowo w replikacji z wieloma liderami stosowana jest strategia ostatni zapis wygrywa. Jeśli więc dwa węzły mają niezsynchronizowane zegary, różnica może wynosić nawet 2-3ms, to może dojść do sytuacji, gdzie kolejność zapisów zostanie przestawiona, co doprowadzi do niepoprawnych danych. Okazuje się jednak, że nawet synchronizacja zegarów przez NTP może być niewystarczająca, bowiem duży wpływ na nią mają także opóźnienia w sieci. Dlatego do porządkowania zdarzeń powinny być raczej stosowane zegary logiczne oparte na zwiększanych licznikach. Nie mierzą one czasu, a jedynie określają względne uporządkowanie zdarzeń.

W niektórych bazach jak np. Spanner od Google używany jest interfejs API TrueTime, który zwraca odpowiednio zakres czasu jaki może obecnie być (czyli uwzględnia błąd jaki może występować) [najwcześniej, najpóźniej]. Taka baza odpowiednio zarządza transakcjami odnosząc się do tego błędu, tak aby zapewnić odpowiednią kolejność. Oczywiście, aby wydajność była odpowiednia przedział czasu nie powinien duży, dlatego Google w każdym ze swoich centrów danych korzysta z odbiornika GPS lub zegara atomowego do określania aktualnego czasu, co pozwala na synchronizację zegarów z dokładnością do około 7ms.

Jak widać czas w projektach informatycznych nie jest rzeczą trywialną. Skupmy się jednak na sekundzie przestępnej i problemach z nią związanych.

## Problemy i porażki
Programiści doskonale wiedzą, że jeśli coś występuje niezwykle rzadko, możemy spodziewać się, że coś nie działa tak jak zakładaliśmy, a wcześniej nikt tego nie doświadczył, bo po prostu nie miał okazji. W przypadku sekund przestępnych taka sytuacja miała miejsce 27 razy w przeciągu 52 lat, czyli średnio raz na dwa lata. Jest to niezwykle rzadko, więc różnego rodzaju problemy i błędy w oprogramowaniu były nieuniknione. Najgorszy był okres między 1999, a 2005 rokiem, gdzie przez 6 lat w zasadzie można było zapomnieć o istnieniu sekundy przestępnej.

30 czerwca 2012 roku poważny spadek wydajności zanotował serwis Reddit,  początkowo administratorzy obwiniali spowolnienie sieci, które kilka dni wcześniej również miało miejsce. Dopiero po głębszych poszukiwaniach okazało się, że maszyny z linuxem mają bardzo wysokie użycie CPU. Problemem był bug w kernelu linuxa, moduł „hrtimer” zwariował po dodaniu dodatkowej sekundy. Reddit nie był jedynym poszkodowanym, bo tego typu problemy zanotowały też inne serwisy. Linus Torvalds skomentował sytuację w następujący sposób:

> Almost every time we have a leap second, we find something. It’s really annoying, because it’s a classic case of code that is basically never run, and thus not tested by users under their normal conditions.

## Wyjaśnienie problemu z hrtimer
Hrtimer jest to moduł odpowiedzialny za wybudzanie aplikacji ze stanu bezczynności. Aplikacji, które na przykład czekają na wykonanie jakiegoś zadania przez system operacyjny, a dopiero później będą mogły wykonać swoje czynności. Po dodaniu sekundy przestępnej moduł zwariował i wybudził wszystkie aplikacje co spowodowało wysokie użycie CPU i gigantyczny spadek wydajności, a nawet w niektórych przypadkach całkowite zawieszenie systemu.

W przypadku Reddita problem dotyczył głownie bazy danych Cassandra i Javy, natomiast podobne problemy powiązane z tym samym bugiem w hrtimer napotkane zostały w bazach MySQL, serwerach Tomcat czy Hadoopie.

Co ciekawe serwery Opery zawiesiły się dzień wcześniej po samym ogłoszeniu przez serwery NTP sekundy przestępnej.

## Zużycie prądu
Centrum danych Hetzner Online podało, że w nocy z 30 czerwca na 1 lipca 2012 roku zużycie energii znacząco skoczyło do wartości powyżej 1 megawata. Spowodowane było to bugiem opisanym powyżej. Procesory nagle zaczęły pracować z obciążeniem 100%, co przełożyło się na wzrost zużycia energii. Na wykresie pokazana jest 2:00 w nocy, prawdopodobnie dlatego, że uwzględniona jest tutaj strefa czasowa. Podobny skok zużycia energii wystąpił również w OVH. Prawdopodobnie również w innych centrach, aczkolwiek tylko te dwa pochwaliły się takimi obserwacjami.

![Zużycie energii w Hetzner Online](/assets/img/2019-01-07/power.png)

## Problem roku 2000
Kilkadziesiąt lat temu nie mieliśmy takiego komfortu, aby pozwolić sobie na szastanie pamięcią na prawo i lewo.

![1969 RAM vs 2017 RAM](/assets/img/2019-01-07/tweet.png)

Dlatego też rok zapisywano podając tylko ostatnie dwie cyfry. Pierwsze problemy zaczęły pojawiać się w latach 90, przy określaniu np. ważności karty kredytowej, która wykraczała poza rok 2000. Wiele osób mówiło o katastrofie, która przez to nastąpi, zapowiadano prawdziwą apokalipsę. Końca świata co prawda nie było, jednak nadal jest to jeden z najgłupszych popełnionych błędów przez programistów. Myślenie tylko o niedalekiej przyszłości nigdy nie jest dobre. Finalnie doprowadzenie oprogramowania do stanu używalności wymagało sporo pracy, co oczywiście wiązało się z kosztami.

## Implementacje sekundy przestępnej
Najprostszą implementacją jest po prostu ogłaszanie przez serwery NTP sekundy przestępnej. Następnie po prostu na koniec minuty, kiedy ta sekunda ma zostać dodana wartość 59 pojawia się dwa razy.

Kolejnym podejściem jest to stosowane w Windowsach głównie, system ignoruje sekundę przestępną. Dopiero przy następnej synchronizacji z NTP czas przeskakuje do właściwej wartości.

Powyższe dwa rozwiązania mają swoje wady, bo w pierwszym mamy dwa razy ten sam czas, a w drugim dziwny przeskok, więc Google postanowiło podejść do tematu inaczej. Zaproponowali rozwiązanie zwane „leap smear”, które stopniowo dodaje milisekundy do czasu przez cały dzień, zamiast w jednym momencie dodawać całą sekundę. Dokładnie to w każdej sekundzie dnia kiedy ma zostać dodana sekunda przestępna dodawana jest wartość 1/86400, co odpowiada po prostu rozłożeniu tej jednej dodatkowej sekundy na sekundy doby (24*60*60 = 86400).

## Przyszłe problemy z czasem
Kolejny problem z czasem może nastąpić 19 stycznia 2038. W systemach opartych na Uniksie czas zapisywany jest liczbą sekund od 1 stycznia 1970 00:00:00. Liczba sekund przechowywana jest w zmiennej typu signed integer (liczba całkowita ze znakiem) o maksymalnej wartości dodatniej 2 147 483 647 (32bity), co odpowiada godzinie 03:14:07 UTC, 19 stycznia 2038. Rozwiązaniem problemu będzie przejście na zapis z użyciem 64 bitów, co znacząco zwiększy zakres wartości (2^63-1). Przy użyciu 64 bitów kolejny taki problem pojawi się dopiero w roku 292 277 026 596 czyli za 292 miliardy lat. Systemy operacyjne raczej zostaną dostosowane w miarę możliwości bezproblemowo. Jednak wiadomo zawsze znajdzie się ktoś, kto korzysta jeszcze z jakiegoś starego nierozwijanego już oprogramowania i ma wtedy poważny problem.

## Koniec sekundy przestępnej
Po latach dyskusji prowadzonych przez różne organy normalizacyjne, decyzja o zniesieniu sekundy przestępnej do lub przed 2035 r. została podjęta podczas 27. Generalnej Konferencji Miar i Wag w listopadzie 2022 r. Począwszy od 2035 r. sekundy przestępne nie będą już stanowić problemu. Międzynarodowe Biuro Miar i Wag pozwoli UTC i UT1 oddalać się od siebie do co najmniej 2135 roku, w nadziei, że naukowcy opracują lepszą metodę rozliczania straconego czasu – lub że komputery staną się bardziej biegłe w zarządzaniu zmianami zegara.

## Podsumowanie
Jak widać obsługa czasu wcale nie jest rzeczą łatwą, a tym bardziej w przypadku sekundy przestępnej, gdzie faktyczne działanie rozwiązania na produkcji jest sprawdzane bardzo rzadko. Takie problemy na pewno nadal się będą zdarzać. Z resztą nie ma co się dziwić, bo są to raczej rozwiązania stosunkowo niskopoziomowe, z których korzysta cała masa innych rzeczy. Trzeba jednak przyznać, że najbardziej przewidywalnym i bezpiecznym rozwiązaniem wydaje się to od Google zwane „Leap smear”, gdzie nie ma jakiegoś nienaturalnego przeskoku raz na x lat, tylko wszystko odbywa się płynnie nie naruszając normalnego działania.

PS. W dniu dodawania sekundy przestępnej w miarę możliwości lepiej nie wsiadajcie do samolotu,  a już najlepiej to zostańcie w domu. Na pewno coś znowu nie zadziała, przekonamy się co tym razem  😀

Źródła:

https://pl.wikipedia.org/wiki/Sekunda

https://pl.wikipedia.org/wiki/Sekunda_przest%C4%99pna

„Przetwarzanie danych w dużej skali” – Martin Kleppmann

https://www.wired.com/2012/07/leap-second-glitch-explained/

http://www.h-online.com/open/news/item/Leap-second-Linux-can-freeze-1629805.html

https://pl.wikipedia.org/wiki/Problem_roku_2038

https://access.redhat.com/articles/15145

https://googleblog.blogspot.com/2011/09/time-technology-and-leaping-seconds.html


