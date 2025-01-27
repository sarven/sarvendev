---
title: Encja powinna być zawsze poprawnym obiektem
date: 2018-01-29
categories: [Programming, Good practices]
permalink: /pl/2018/01/encja-byc-zawsze-poprawnym-obiektem/
redirect_from:
  - /2018/01/encja-byc-zawsze-poprawnym-obiektem/
---
Bardzo często w projektach z użyciem Doctrine, encja wygląda w ten sposób, że zrobione jest mapowanie odpowiednich pól, oraz do każdego pola utworzone są gettery oraz settery. Dodatkowo do każdego pola mamy odpowiednie adnotacje walidacji, a formularze walidowane są na encji. Czy to na pewno jest dobre podejście?
```php
<?php

namespace App\Entity;

use Doctrine\ORM\Mapping as ORM;
use Symfony\Component\Validator\Constraints as Assert;
use Symfony\Bridge\Doctrine\Validator\Constraints\UniqueEntity;

/**
 * Class User
 * @package App\Entity
 *
 * @ORM\Table(name="users")
 * @ORM\Entity(repositoryClass="App\Repository\User\UserRepository")
 * @UniqueEntity(fields={"email"})
 */
class User
{
    /**
     * @var int
     * @ORM\Column(type="integer")
     * @ORM\Id
     */
    private $id;

    /**
     * @var string
     * @ORM\Column(type="string", length=64)
     * @Assert\Length(min="8", max="4096")
     */
    private $password;

    /**
     * @var string
     * @ORM\Column(type="string", length=80, unique=true)
     * @Assert\Email()
     */
    private $email;

    /**
     * User constructor.
     */
    public function __construct()
    {
    }

    /**
     * @return int
     */
    public function getId(): int
    {
        return $this->id;
    }

    /**
     * @return string
     */
    public function getPassword(): string
    {
        return $this->password;
    }

    /**
     * @param string $password
     */
    public function setPassword(string $password): void
    {
        $this->password = $password;
    }

    /**
     * @return string
     */
    public function getEmail(): string
    {
        return $this->email;
    }

    /**
     * @param string $email
     */
    public function setEmail(string $email): void
    {
        $this->email = $email;
    }
}
```
```php
<?php

namespace App\Form;

use App\Entity\User;
use Symfony\Component\Form\AbstractType;
use Symfony\Component\Form\Extension\Core\Type\EmailType;
use Symfony\Component\Form\Extension\Core\Type\PasswordType;
use Symfony\Component\Form\Extension\Core\Type\RepeatedType;
use Symfony\Component\Form\FormBuilderInterface;
use Symfony\Component\OptionsResolver\OptionsResolver;

/**
 * Class UserType
 * @package App
 */
final class UserType extends AbstractType
{
    /**
     * {@inheritdoc}
     */
    public function buildForm(FormBuilderInterface $builder, array $options)
    {
        $builder
            ->add('email', EmailType::class)
            ->add('password', RepeatedType::class, [
                'type' => PasswordType::class
            ])
        ;
    }

    /**
     * {@inheritdoc}
     */
    public function configureOptions(OptionsResolver $resolver)
    {
        $resolver->setDefaults([
            'data_class' => User::class
        ]);
    }

}
```
Powyższy kod pokazuje  jak w większości projektów wygląda implementacja encji doctrinowych.

## Encja to nie struktura danych
W podanym przykładzie zaprezentowany obiekt w zasadzie pełni podobną funkcję jak tablica. Nie posiada żadnych zachowań, tylko zwykłe przechowywanie, ustawianie i pobieranie wartości. Do zachowań można byłoby tutaj zaliczyć zmianę hasła, więc w zasadzie setPassword jest w porządku, jedynie może lepsza byłaby nazwa changePassword. Zakładamy, że email jest ustawiany raz podczas rejestracji i nie można go później zmienić. Więc setter dla pola email jest zbędny.

## Walidacja
Encja powinna być zawsze poprawnym obiektem, aby nie było sytuacji, gdzie nieprawidłowe dane zostaną zapisane w bazie danych. W związku z tym walidacja przesłanych danych powinna się zawierać w pomocniczym obiekcie np. DTO (Data Transfer Object).

Dodatkowo jeśli chcemy używać formularzy symfonowych z encjami, musimy modyfikować type hinty, tak aby były wstanie przyjąć nieprawidłowe dane w celu ich sprawdzenia i wyrzucenia odpowiedniego błędu. Co stanowi kolejny powód, aby walidacja odbywała się na innym obiekcie.

Podsumowując, adnotacje walidacji trzymamy w innym obiekcie, z formularzy korzystamy przypisując jako data_class ten obiekt, a nie encję. Encja jest tworzona z tego obiektu dopiero po jego zwalidowaniu, dzięki temu mamy pewność, że w każdym momencie działania aplikacji jest ona poprawna.

## Konstruktory
Z racji, że encja powinna być zawsze poprawnym obiektem, nie powinna być ona tworzona przez utworzenie obiektu korzystając z konstuktora, a następnie uzupełnienie odpowiednich pól korzystając z setterów. Konstruktor powinien przyjmować takie parametry, aby można było stworzyć poprawny obiekt tj. zgodny z narzuconymi ograniczeniami. W php nie mamy możliwości przeciążania metod, więc dla tworzenia obiektów w różny sposób, na podstawie różnych parametrów dobrze jest utworzyć tak zwane named constructors tj. statyczne metody zwracające obiekt klasy w jakiej się zawierają.

## Jak to powinno wyglądać?
```php
<?php

namespace App\Entity;

use Doctrine\ORM\Mapping as ORM;
use Ramsey\Uuid\Uuid;
use App\User\Command\RegisterUserCommand;

/**
 * Class User
 * @package App\Entity
 *
 * @ORM\Table(name="users")
 * @ORM\Entity(repositoryClass="App\Repository\User\UserRepository")
 */
class User
{
    /**
     * @var string
     * @ORM\Column(type="string")
     * @ORM\Id
     */
    private $id;

    /**
     * @var string
     * @ORM\Column(type="string", length=64)
     */
    private $password;

    /**
     * @var string
     * @ORM\Column(type="string", length=80, unique=true)
     */
    private $email;

    /**
     * User constructor.
     * @param string $password
     * @param string $email
     */
    public function __construct(string $password, string $email)
    {
        $this->id = Uuid::uuid4();
        $this->password = $password;
        $this->email = $email;
    }

    /**
     * @param RegisterUserCommand $registerUserCommand
     * @return User
     */
    public static function fromRegisterUserCommand(RegisterUserCommand $registerUserCommand): User
    {
        return new self($registerUserCommand->email, $registerUserCommand->password);
    }

    /**
     * @return int
     */
    public function getId(): int
    {
        return $this->id;
    }

    /**
     * @return string
     */
    public function getPassword(): string
    {
        return $this->password;
    }

    /**
     * @param string $password
     */
    public function changePassword(string $password): void
    {
        $this->password = $password;
    }

    /**
     * @return string
     */
    public function getEmail(): string
    {
        return $this->email;
    }
}
```
```php
<?php

namespace App\Form;

use App\Entity\User;
use App\User\Command\RegisterUserCommand;
use Symfony\Component\Form\AbstractType;
use Symfony\Component\Form\Extension\Core\Type\EmailType;
use Symfony\Component\Form\Extension\Core\Type\PasswordType;
use Symfony\Component\Form\Extension\Core\Type\RepeatedType;
use Symfony\Component\Form\FormBuilderInterface;
use Symfony\Component\OptionsResolver\OptionsResolver;

/**
 * Class UserType
 * @package App
 */
final class RegisterUserType extends AbstractType
{
    /**
     * {@inheritdoc}
     */
    public function buildForm(FormBuilderInterface $builder, array $options)
    {
        $builder
            ->add('email', EmailType::class)
            ->add('password', RepeatedType::class, [
                'type' => PasswordType::class
            ])
        ;
    }

    /**
     * {@inheritdoc}
     */
    public function configureOptions(OptionsResolver $resolver)
    {
        $resolver->setDefaults([
            'data_class' => RegisterUserCommand::class
        ]);
    }

}
```
```php
<?php

namespace App\User\Command;

use Symfony\Component\Validator\Constraints as Assert;
use App\Common\Validator\Constraint\UniqueField\UniqueField;

/**
 * Class RegisterUserCommand
 * @package App\User\Command
 */
final class RegisterUserCommand
{
    /**
     * @var string
     *
     * @Assert\Email()
     * @UniqueField(entityClass="App\Entity\User\User", field="email")
     */
    public $email;

    /**
     * @var string
     *
     * @Assert\Length(min="8", max="4096")
     */
    public $password;
}
```
```php
use Doctrine\ORM\EntityManagerInterface;
use Symfony\Component\Validator\Constraint;
use Symfony\Component\Validator\ConstraintValidator;

/**
 * Class UniqueFieldValidator
 * @package Gdata\CoreBundle\Validator\Constraint\UniqueField
 */
class UniqueFieldValidator extends ConstraintValidator
{
    /**
     * @var EntityManagerInterface
     */
    private $entityManager;

    /**
     * UniqueFieldValidator constructor.
     * @param EntityManagerInterface $entityManager
     */
    public function __construct(EntityManagerInterface $entityManager)
    {
        $this->entityManager = $entityManager;
    }

    /**
     * @param mixed $value
     * @param Constraint $constraint
     */
    public function validate($value, Constraint $constraint): void
    {
        $entityRepository = $this->entityManager->getRepository($constraint->entityClass);

        if (!is_scalar($constraint->field)) {
            throw new \InvalidArgumentException('"field" parameter should be any scalar type');
        }

        $searchResults = $entityRepository->findBy([
            $constraint->field => $value
        ]);

        if (count($searchResults) > 0) {
            $this->context->buildViolation($constraint->message)
                ->addViolation();
        }
    }
}
```
