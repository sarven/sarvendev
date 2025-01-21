---
title: Modeling a future action
date: 2022-12-13
categories: [Programming, DDD]
permalink: /2022/12/modelling-a-future-action/
---
During modeling a business logic we have often a problem with properly highlighting a relevant future action. I mean relevant from a domain point of view. The most popular solution will be using a CLI command which will be executed by cron at a specific time. I think that this solution often hides a lot of business logic in an inappropriate place. Perhaps, there is a better way?

## What is the problem?
Dealing with time is a common thing. In our systems, we encounter a logic that should be executed at a specific time. When thinking about this problem many of us have one solution in the mind – cron, and plan specific actions by defining them in the crontab. Certainly, it works properly, but when the system is quite large cron definitions can become very messy. Also, crontab often hides details that are specific to a certain module. For example:
- sending notifications about expiring accounts
- checking overdue invoices
So crontab contains details on when to send notifications about expiring accounts and when to check overdue invoices, but these details belong to the specific domain.

## Perhaps a better solution to that problem
The better solution will be informing our domain about elapsing time. Then the domain logic can decide what action is required. So we can just prepare a domain event – DayHasPassed which will be thrown by a CLI command executed by cron at 00:00 every day.

```php
<?php

declare(strict_types=1);

namespace App\Shared\Domain\Event;

use DateTimeImmutable;

final class DayHasPassedEvent
{
    public function __construct(private readonly DateTimeImmutable $createdAt)
    {
    }

    public static function getName(): string
    {
        return 'shared.day_has_passed';
    }
}
```
```php
<?php

declare(strict_types=1);

namespace App\UI\Cli\Command;

use App\Shared\Domain\Clock\ClockInterface;
use App\Shared\Domain\Event\DayHasPassedEvent;
use App\Shared\Infrastructure\Bus\EventBus;
use Symfony\Component\Console\Command\Command;
use Symfony\Component\Console\Input\InputInterface;
use Symfony\Component\Console\Output\OutputInterface;

final class DayHasPassedCommand extends Command
{
    protected static $defaultName = 'app:day-has-passed';

    public function __construct(
        private readonly EventBus $eventBus,
        private readonly ClockInterface $clock,
    ) {
        parent::__construct();
    }

    protected function execute(InputInterface $input, OutputInterface $output): int
    {
        $this->eventBus->publish(new DayHasPassedEvent($this->clock->getCurrentDateTime()));

        return Command::SUCCESS;
    }
}
```
Then in the specific module, we can create a subscriber to that event and invoke some logic.
```php
<?php

declare(strict_types=1);

namespace App\SomeDomain\Application\Event;

use App\Shared\Domain\Event\DayHasPassedEvent;
use App\Shared\Domain\Event\DomainEventSubscriberInterface;

final class SomeSubscriber implements DomainEventSubscriberInterface
{
    public function __invoke(DayHasPassedEvent $event): void
    {
        // do something
    }
}
```
This solution simplifies our cron which now only informs our domain about elapsing time and doesn’t control the logic. As a result, we have a good separation of concerns, cron is just an event emitter and business details are properly encapsulated in the right module.

