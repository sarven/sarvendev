---
title: Unlocking ORM Performance - The Essential Role of Read Models
date: 2024-10-01
categories: [Programming, Good practices, PHP]
permalink: /2024/10/unlocking-orm-performance-the-essential-role-of-read-models/
image:
  path: /assets/img/2024-10-01/featured.png
---
ORMs are useful tools that help us save our objects to the database. However, there are some pitfalls, so it is important to know the tools we use. In this article, I want to focus on the poor performance of reading a lot of data using ORMs.

## Example
I generated 1000 products, 50 categories, 50 warehouses, 50 suppliers, and 50 manufacturers. And I’m going to compare three solutions to get the list of 1000 products from the database:
- Doctrine ORM
- Doctrine DBAL
- Eloquent
- Eloquent with read-model
Of course, we can say we don’t get 1000 rows from the database, but it depends on the requirements. I’ve seen some projects where the number of elements was much bigger, and the relations between entities were way more complicated.

```php
#[ORM\Entity]
#[ORM\Table(name: 'products')]
class Product
{
    #[ORM\Id]
    #[ORM\GeneratedValue]
    #[ORM\Column(type: 'integer')]
    private int $id;

    #[ORM\Column(type: 'string', length: 255)]
    private string $name;

    #[ORM\Column(type: 'decimal', precision: 10, scale: 2)]
    private float $price;

    #[ORM\ManyToOne(targetEntity: Category::class)]
    #[ORM\JoinColumn(nullable: false)]
    private Category $category;

    #[ORM\ManyToOne(targetEntity: Supplier::class)]
    #[ORM\JoinColumn(nullable: false)]
    private Supplier $supplier;

    #[ORM\ManyToOne(targetEntity: Manufacturer::class)]
    #[ORM\JoinColumn(nullable: false)]
    private Manufacturer $manufacturer;

    #[ORM\ManyToOne(targetEntity: Warehouse::class)]
    #[ORM\JoinColumn(nullable: false)]
    private Warehouse $warehouse;
```
## Doctrine ORM
```php
use App\Entity\Product;
use Doctrine\Bundle\DoctrineBundle\Repository\ServiceEntityRepository;
use Doctrine\Persistence\ManagerRegistry;

final class OrmProductRepository extends ServiceEntityRepository
{
    public function __construct(ManagerRegistry $registry)
    {
        parent::__construct($registry, Product::class);
    }

    /**
     * @return Product[]
     */
    public function findAll(): array
    {
        return $this->createQueryBuilder('p')
            ->select('p', 'c', 's', 'm', 'w')
            ->leftJoin('p.category', 'c')
            ->leftJoin('p.supplier', 's')
            ->leftJoin('p.manufacturer', 'm')
            ->leftJoin('p.warehouse', 'w')
            ->setMaxResults(1000)
            ->getQuery()
            ->getResult();
    }
}
```
![Profile 1](/assets/img/2024-10-01/profile1.png)

**Query**
```sql
select ... from products p0_ left join categories c1_ on p0_.category_id = c1_.id left join suppliers s2_ on p0_.supplier_id = s2_.id left join manufacturers m3_ on p0_.manufacturer_id = m3_.id left join warehouses w4_ on p0_.warehouse_id = w4_.id limit ?
```

## Doctrine DBAL
```php
final readonly class ProductItem
{
    public function __construct(
        private string $name,
        private float $price,
        private string $category,
        private string $manufacturer,
        private string $supplier,
        private string $warehouse
    ) {
    }
}
```

```php
use App\ReadModel\ProductItem;
use Doctrine\DBAL\Connection;
use function array_map;

final readonly class DbalProductRepository
{
    private Connection $connection;

    public function __construct(Connection $connection)
    {
        $this->connection = $connection;
    }

    /**
     * @return ProductItem[]
     */
    public function findAll(): array
    {
        $data = $this->connection
            ->createQueryBuilder()
            ->select('p.*, c.name AS category_name, s.name AS supplier_name, 
                     m.name AS manufacturer_name, w.location AS warehouse_location')
            ->from('products', 'p')
            ->leftJoin('p', 'categories', 'c', 'p.category_id = c.id')
            ->leftJoin('p', 'suppliers', 's', 'p.supplier_id = s.id')
            ->leftJoin('p', 'manufacturers', 'm', 'p.manufacturer_id = m.id')
            ->leftJoin('p', 'warehouses', 'w', 'p.warehouse_id = w.id')
            ->setMaxResults(1000)
            ->fetchAllAssociative();

        return array_map(
            fn (array $row) => new ProductItem(
                $row['name'],
                (float) $row['price'],
                $row['category_name'],
                $row['manufacturer_name'],
                $row['supplier_name'],
                $row['warehouse_location']
            ),
            $data
        );
    }
}
```
![Profile 2](/assets/img/2024-10-01/profile2.png)
```sql
select p.*, ... from products p left join categories c on p.category_id = c.id left join suppliers s on p.supplier_id = s.id left join manufacturers m on p.manufacturer_id = m.id left join warehouses w on p.warehouse_id = w.id limit ?
```

## Eloquent
```php
use Illuminate\Database\Eloquent\Model;

class Product extends Model
{
    protected $fillable = ['name', 'price', 'category_id', 'supplier_id', 'manufacturer_id', 'warehouse_id'];

    public function category()
    {
        return $this->belongsTo(Category::class);
    }

    public function supplier()
    {
        return $this->belongsTo(Supplier::class);
    }

    public function manufacturer()
    {
        return $this->belongsTo(Manufacturer::class);
    }

    public function warehouse()
    {
        return $this->belongsTo(Warehouse::class);
    }
}
```
```php
Product::with(['category', 'supplier', 'manufacturer', 'warehouse'])->get();
```
![Profile 3](/assets/img/2024-10-01/profile3.png)
```sql
select * from products
select * from categories where categories.id in (...)
select * from suppliers where suppliers.id in (...)
select * from warehouses where warehouses.id in (...)
select * from manufacturers where manufacturers.id in (...)
```

It’s implemented differently than in Doctrine, because we have here 5 queries instead of joins.

## Eloquent with read-model
```php
use App\Model\Product;
use App\ReadModel\ProductItem;
use stdClass;
use function array_map;

final readonly class EloquentProductRepository
{
    /**
     * @return ProductItem[]
     */
    public function findAll(): array
    {
        $data = Product::select(
            'products.*',
            'categories.name as category_name',
            'suppliers.name as supplier_name',
            'manufacturers.name as manufacturer_name',
            'warehouses.location as warehouse_location'
        )
            ->leftJoin('categories', 'products.category_id', '=', 'categories.id')
            ->leftJoin('suppliers', 'products.supplier_id', '=', 'suppliers.id')
            ->leftJoin('manufacturers', 'products.manufacturer_id', '=', 'manufacturers.id')
            ->leftJoin('warehouses', 'products.warehouse_id', '=', 'warehouses.id')
            ->limit(1000)
            ->toBase()->get();

        return array_map(
            fn (stdClass $row) => new ProductItem(
                $row->name,
                (float) $row->price,
                $row->category_name,
                $row->manufacturer_name,
                $row->supplier_name,
                $row->warehouse_location,
            ),
            $data->toArray(),
        );
    }
}
```
![Profile 4](/assets/img/2024-10-01/profile4.png)
```sql
select ?.*, ?.? as ?.? as ?.? as ?.? as ? from ? left join ? on ?.? = ?.? left join ? on ?.? = ?.? left join ? on ?.? = ?.? left join ? on ?.? = ?.? limit ?
```

## Summary
**Doctrine ORM** – 235 ms

**Doctrine DBAL with read-model** – 94 ms

**Eloquent** – 285 ms

**Eloquent with read-model** – 102ms

## Why
As we can see in the following profile the most time-consuming part is hydrating data. The hydration is a process of populating data into entities. Doctrine creates 5000 entities, 1000 x 5 types of entities in this case.

![Profile 5](/assets/img/2024-10-01/profile5.png)

![Profile Timeline](/assets/img/2024-10-01/profile-timeline.png)

Eloquent uses a different strategy, and loads data in 5 queries and then joins them together, which creates fewer objects – 1200, and the hydration is faster, but the additional logic is required to join all data which consumes some time.

![Profile 6](/assets/img/2024-10-01/profile6.png)

![Profile Timeline 2](/assets/img/2024-10-01/profile-timeline2.png)

The hydration is slow because it’s done by reflection.

## Use separate read models
As we can see a separate model will be much faster than using ORM entities to read data. Another advantage of using this approach is the separation between writing and reading. The intent of the read model is clear, there is no way of changing the state. It also helps us avoid another typical ORM problem – the N+1 problem.

## Know the tools we use
When choosing a tool, we must consider the pros and cons because there is no ideal solution. I’ve compared performance of a few approaches, but we still need to take into account other factors like speed of development, flexibility, complexity, etc. Doctrine ORM uses the Data mapper pattern which gives us great flexibility in modeling our business logic and then saving it to the database in different forms, it doesn’t need to be mapped 1:1, object : row in the database. However, it’s way more complex than the Active Record pattern used by Eloquent, where the model is coupled with the whole framework and mapped usually 1:1 with the database row.

Using a basic abstraction layer (Doctrine DBAL or simple Eloquent) gives us simplicity and better performance, but it requires more work to map objects to the rows in the database. However, using them to read data should be easy and performant, so I would usually go with this solution, full ORM for write models and a basic layer for reads to have separation and better performance.

Bear in mind that the hydration based on reflection will always negatively affect performance. It isn’t only the problem of ORMs, we can also find similar problems in serializers using the same approach.

## EDIT: UPDATE
Here is the continuation of the article: [https://sarvendev.com/2024/10/poor-performance-of-eloquent-orm-in-comparison-to-doctrine/](https://sarvendev.com/2024/10/poor-performance-of-eloquent-orm-in-comparison-to-doctrine/)

Read more:

There is a good set of best practices for Doctrine: [https://www.doctrine-project.org/projects/doctrine-orm/en/3.2/reference/best-practices.html](https://www.doctrine-project.org/projects/doctrine-orm/en/3.
2/reference/best-practices.html)

My article about DDD and ORMs: [https://sarvendev.com/2022/10/an-absolutely-clean-domain-or-just-common-sense/](https://sarvendev.com/2022/10/an-absolutely-clean-domain-or-just-common-sense/)

