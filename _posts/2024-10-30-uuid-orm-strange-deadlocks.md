---
title: UUID, ORM and strange deadlocks
date: 2024-10-30
categories: [Programming, PHP]
permalink: /2024/10/uuid-orm-and-strange-deadlocks/
image:
  path: /assets/img/2024-10-30/featured.png
---
Some time ago I took over the development of one module in a bigger application. The volume of requests in this module was quite small. However, I’ve noticed some strange deadlocks in logs. At first glance, the introduced models were too large, which I suspected was the source of the problems. In the case of designing models, it’s important to determine the consistency boundary at the lowest possible level to achieve business requirements. It’s needed to define where is the boundary and which data should be saved in the transaction and keep a proper balance between consistency and scalability. However, it turned out that it wasn’t the cause of deadlocks.

## Problem
In the module, there was a base entity, and the rest of the entities had a relation to the base one. There was no big volume of requests, but there was parallel processing on a queue, and many consumers, could process data belonging to the same base entity at the same moment. The base entity was designed to be a central place of every save, and on every save, there was a version number changed. Sometimes, we got alerts about deadlocks, so I started investigating what’s the real cause.

## Deadlock
A deadlock is a situation where two or more transactions are stuck, each waiting for the other to release a resource, making it impossible for any of them to proceed. It’s a standard definition of deadlock. However, in MySQL in InnoDB, it’s possible to get a deadlock in a simple transaction with one record:

> InnoDB uses automatic row-level locking. You can get deadlocks even in the case of transactions that just insert or delete a single row. That is because these operations are not really “atomic”; they automatically set locks on the (possibly several) index records of the row inserted or deleted.

## Cause
So from the code, we see that it could be a problem with a big model and updating the version column. Let’s check what is in the database. To check that we use:
  
```
SHOW ENGINE INNODB STATUS;
```

```
LATEST DETECTED DEADLOCK
------------------------
2024-05-11 07:16:57 140667723896384
*** (1) TRANSACTION:
TRANSACTION 8474454, ACTIVE 0 sec starting index read
mysql tables in use 1, locked 1
LOCK WAIT 7 lock struct(s), heap size 1128, 3 row lock(s), undo log entries 9
MySQL thread id 18158847, OS thread handle 140666811168320, query id 237833085 x.x.x.x xxx updating
UPDATE table SET version = '22e4664f-06fe-4952-8034-57f013421876' WHERE id = '5be8e6b6-2e8a-4dcf-9f13-859dc10ce0c9'
```

Ok, so it’s clear that this version update is problematic, but requirements have changed and now we don’t need this.

We removed that, but still sometimes we got errors with deadlocks. So one more check, and we had another query that caused a deadlock:

```sql
UPDATE table SET id = '2082c2e5-9eb5-43c1-809e-82265da220f0' WHERE id = '2082c2e5-9eb5-43c1-809e-82265da220f0'
```

WTF? 🤯

We use Doctrine ORM, so firstly I thought that maybe it’s a feature of ORM to lock some parent rows, when updating children entities. But there are better ways, to explicitly set some lock instead of doing some weird update.

So I started debugging the saving process, what’s going on in the Unit of Work, under the hood of ORM.

I found that in the Unit of Work, which computes the changes and transforms them to updates that need to be executed,  there is a comparison string uuid vs object uuid, which is always treated as a change, so then the update is done.

```php
class BaseEntity
{
    private function __construct(
        public readonly string $id,
```

```xml
<id name="id" type="uuid" length="36">
    <generator strategy="NONE"/>
</id>
```

uuid type – [https://github.com/ramsey/uuid-doctrine/blob/main/src/UuidType.php](https://github.com/ramsey/uuid-doctrine/blob/main/src/UuidType.php)

## Fix
So the fix was really simple, I only added a type to the id and changed some related places.

```php
class BaseEntity
{
    private function __construct(
        public readonly UuidInterface $id,
```

Additionally, we refactored a big model to smaller ones, to improve the overall performance. I decided to write about this problem, because it was really interesting, how such a small mistake can cause some weird problems with the database.

**Read more:** 
- [MySQL – How to Minimize and Handle Deadlocks](https://dev.mysql.com/doc/refman/8.4/en/innodb-deadlocks-handling.html)
- [InnoDB – Locking](https://dev.mysql.com/doc/refman/8.0/en/innodb-locking.html)

