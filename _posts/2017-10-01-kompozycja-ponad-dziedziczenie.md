---
title: Kompozycja ponad dziedziczenie
date: 2017-10-01
categories: [Programming, Good practices]
permalink: /pl/2017/10/kompozycja-ponad-dziedziczenie/
image:
  path: /assets/img/2017-10-01/featured.png
---
Jedną z możliwości programowania obiektowego jest dziedziczenie. Daje nam ono możliwość powtórnego wykorzystania kodu poprzez tworzenie podklas. Warto mieć na uwadze, że nie jest ono złotym środkiem, a jednak bywa ono często nadużywane.


## Definicje
**Dziedziczenie** – mechanizm programowania obiektowego, służący do współdzielenia metod oraz składowych pomiędzy 
klasami. Klasa podrzędna dziedziczy po klasie bazowej, co oznacza, że oprócz własnych właściwości oraz zachowań, zawiera również te z klasy bazowej.

**Kompozycja** – tak jak dziedziczenie jest metodą pozwalającą na powtórne wykorzystanie danego rozwiązania. Z tym, że 
w przypadku kompozycji mamy do czynienia ze składaniem obiektów. Klasa zbudowana jest z innych klas, co znaczy, że obiekt takiej klasy agreguje obiekty innych klas i deleguje odpowiednie działania do nich.

## Co jest nie tak z dziedziczeniem?
Dziedziczenie klas definiowane jest statycznie, co znaczy, że nie mamy możliwości zmiany implementacji w czasie wykonywania aplikacji. Implementacja podklas zależna jest od implementacji klasy bazowej, więc zmiany w klasie bazowej często wymuszają również zmiany w podklasach. Rozbudowane hierarchie dziedziczenia wpływają również negatywnie na testowanie kodu oraz analizę danego rozwiązania.

## Dlaczego kompozycja jest lepsza niż dziedziczenie?
Poprzez zastosowanie kompozycji zyskujemy pewną elastyczność.  W razie potrzeby możemy dynamicznie w czasie 
wykonywania aplikacji zmienić implementację jakiej używamy. Kolejnym plusem jest rozbicie klas na mniejsze, co 
pozwala nam na dostarczanie rozwiązań zgodnych z zasadami programowania obiektowego [SOLID](https://sarvendev.com/2017/08/1-solid-single-responsibility-principle/).

Załóżmy, że musimy zaprojektować system, w którym do czynienia będziemy mieli z pracownikami podzielonymi na poszczególne stanowiska np. Developer, Project Manager itp.  Każdy pracownik będzie miał imię, nazwisko, jednak różnica będzie taka, że do developera będziemy mogli przypisać jego główny język programowania, natomiast do Project Managera będzie możliwość dodania projektów jakimi zarządza. Do tego momentu oczywiście myślimy o dziedziczeniu.

Jednak dodatkowo każdy pracownik może rozliczać swoje wynagrodzenie na różne sposoby np. umowa o pracę lub własna działalność(b2b). Czy widzisz tutaj jakiś dobry sposób dziedziczenia? Z racji, że zarówno Developer jak i Project Manager może wybrać dowolny sposób rozliczania, to najwygodniej jest tutaj zastosować kompozycję. Chcąc załatwić sprawę za pomocą dziedziczenia, nie jesteśmy w stanie dostarczyć rozwiązania łatwego w późniejszym utrzymaniu.

Przejdźmy zatem do implementacji. Zaczniemy najpierw od abstrakcyjnej klasy bazowej pracownika oraz odpowiednich interfejsów. Abstrakcyjnej dlatego, że zakładamy iż nie będzie „zwykłego” pracownika, a jedynie pracownicy z odpowiednimi tytułami: Developer etc., mającymi własne cechy charakterystyczne.

```php
<?php

namespace App\Employee;

use App\Salary\SalaryCalculatorInterface;

abstract class Employee implements SalaryCalculable
{
    /**
     * @var string
     */
    protected $firstName;

    /**
     * @var string
     */
    protected $lastName;

    /**
     * @var float
     */
    protected $netSalaryPerHour;

    /**
     * @var SalaryCalculatorInterface
     */
    protected $salaryCalculator;

    public function __construct(SalaryCalculatorInterface $salaryCalculator)
    {
        $this->salaryCalculator = $salaryCalculator;
        $this->netSalaryPerHour = 0;
    }

    public function getFirstName(): string
    {
        return $this->firstName;
    }

    public function setFirstName(string $firstName): Employee
    {
        $this->firstName = $firstName;
        return $this;
    }

    public function getLastName(): string
    {
        return $this->lastName;
    }

    public function setLastName(string $lastName): Employee
    {
        $this->lastName = $lastName;
        return $this;
    }

    public function getNetSalaryPerHour(): float
    {
        return $this->netSalaryPerHour;
    }

    public function setNetSalaryPerHour(float $netSalaryPerHour): Employee
    {
        $this->netSalaryPerHour = $netSalaryPerHour;
        return $this;
    }

    public function getSalary(): float
    {
        $this->salaryCalculator->calcSalary($this->netSalaryPerHour);
    }
}
```
```php
<?php

namespace App\Employee;

interface SalaryCalculable
{
    public function getSalary(): float;
}
```
```php
<?php

namespace App\Salary;

interface SalaryCalculatorInterface
{
    public function calcSalary(float $netPerHour): float;
}
```
Interfejs SalaryCalculatorInterface implementowany będzie przez klasy odpowiedzialne za różne sposoby rozliczania wynagrodzenia. Następnie implementujemy klasy reprezentujące konkretnych pracowników według założeń opisanych wyżej.

```php
<?php

namespace App\Employee;

class Developer extends Employee
{
    /**
     * @var string
     */
    protected $programmingLanguage;

    /**
     * @return string
     */
    public function getProgrammingLanguage(): string
    {
        return $this->programmingLanguage;
    }

    /**
     * @param string $programmingLanguage
     * @return Developer
     */
    public function setProgrammingLanguage($programmingLanguage): Developer
    {
        $this->programmingLanguage = $programmingLanguage;
        return $this;
    }
}
```
```php
<?php

namespace App\Employee;

class ProjectManager extends Employee
{
    /**
     * @var string[]
     */
    protected $projects;

    /**
     * @return string[]
     */
    public function getProjects(): array
    {
        return $this->projects;
    }

    public function addProject(string $project): ProjectManager
    {
        if (!in_array($project, $this->projects)) {
            $this->projects[] = $project;
        }

        return $this;
    }

    public function removeProject(string $project): ProjectManager
    {
        if ($index = array_search($project, $this->projects)) {
            unset($this->projects[$index]);
        }

        return $this;
    }
}
```
Pozostała nam jedynie szczegółowa implementacja klas odpowiedzialnych za metody rozliczania.

```php
<?php

namespace App\Salary;

final class EmploymentContractSalaryCalculator implements SalaryCalculatorInterface
{
    public function calcSalary(float $netPerHour): float
    {
        return 15000;
    }
}
```

```php
<?php

namespace App\Salary;

final class B2bSalaryCalculator implements SalaryCalculatorInterface
{
    public function calcSalary(float $netPerHour): float
    {
        return 30000;
    }
}
```

Dla uproszczenia przykłady nie zagłębiamy się w szczegóły, tylko podajemy stałe kwoty. W prawdziwym projekcie warto byłoby jednak nazwać dokładniej czy zwracana kwota to brutto czy netto. Przedstawione klasy użyć możemy w następujący sposób:

```php
<?php

use App\Employee\Developer;
use App\Employee\ProjectManager;
use App\Salary\B2bSalaryCalculator;
use App\Salary\EmploymentContractSalaryCalculator;

$firstDeveloper = new Developer(new B2bSalaryCalculator());
$secondDeveloper = new Developer(new EmploymentContractSalaryCalculator());

$projectManager = new ProjectManager(new EmploymentContractSalaryCalculator());
```

## Podsumowanie
Stosując kompozycję aplikacja przez większą liczbę klas będzie sprawiała wrażenie bardziej złożonej. Po głębszej analizie powinno okazać się jednak, że łatwiej jest przewidzieć zachowanie danego kodu niż ma to miejsce przy rozbudowanym dziedziczeniu.

Oczywiście kompozycja oraz dziedziczenie powinny ze sobą współgrać, a dostarczone rozwiązanie powinno zostać przemyślane pod kątem przyszłych zmian. Warto jednak pamiętać, że idealnych rozwiązań nie ma i często jest tak, że nie jesteśmy w stanie przewidzieć, że coś się zmieni. Ciężko jest również napisać kod przygotowany na wszystkie zmiany. Dążenie do perfekcji nie jest niczym dobrym, należy po prostu w razie wystąpienia zmian, których nie przewidzieliśmy dostosować odpowiednio kod i zostawić go lepszym niż zastaliśmy.
