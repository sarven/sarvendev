---
title: Poor performance of Eloquent ORM in comparison to Doctrine
date: 2024-10-14
categories: [Programming, PHP, Laravel, Symfony]
permalink: /2024/10/poor-performance-of-eloquent-orm-in-comparison-to-doctrine/
image:
  path: /assets/img/2024-10-14/featured.png
---
In the last article, I compared two ORMs: Eloquent mostly related to Laravel, and Doctrine associated with Symfony. I presented a comparison on why reading data from the database would be much more performant using read models instead of slow ORM models with slow hydration. Read models, of course, are the best way to read data from the database, but I didn’t properly present the comparison of ORMs, which was pointed out by one user on Reddit.

Here is the link to the related article about performance and read models: [https://sarvendev.com/2024/10/unlocking-orm-performance-the-essential-role-of-read-models/](https://sarvendev.com/2024/10/unlocking-orm-performance-the-essential-role-of-read-models/)

## Wrong assumption
![Reddit comment](/assets/img/2024-10-14/reddit-comment.png)

Thx @walden42 😉

I’m from the Doctrine world, so I assumed that when I have the following code, the most complex part is already done, and reading data from the collection is simple. It’s true for Doctrine, but not for Eloquent.

**Eloquent:**
```php
Product::with(['category', 'supplier', 'manufacturer', 'warehouse'])->get();
```

**Doctrine:**
```php
$products = $this->ormProductRepository->findAll();
```

So I mapped models to array and returned data as JSON on some endpoint:

**Eloquent:**
```php
public function index(): JsonResponse
{
    $stopWatch = new Stopwatch();

    $stopWatch->start('eloquent');
    $products = Product::with(['category', 'supplier', 'manufacturer', 'warehouse'])->get();

    $products = $products->map(static function (Product $product): array {
        return [
            'name' => $product->name,
            'price' => $product->price,
            'category_name' => $product->category->name,
            'manufacturer_name' => $product->manufacturer->name,
            'supplier_name' => $product->supplier->name,
            'warehouse_location' => $product->warehouse->location,
        ];
    });
    $event = $stopWatch->stop('eloquent');

    return new JsonResponse([
        'time' => (string) $event,
        'products' => $products,
    ]);
}
```

**Doctrine:**
```php
public function index(): JsonResponse
{
    $stopWatch = new Stopwatch();

    $stopWatch->start('orm');
    $products = $this->ormProductRepository->findAll();
    $products = array_map(static function (Product $product): array {
        return [
            'name' => $product->name,
            'price' => $product->price,
            'category_name' => $product->category->name,
            'manufacturer_name' => $product->manufacturer->name,
            'supplier_name' => $product->supplier->name,
            'warehouse_location' => $product->warehouse->location,
        ];
    }, $products);
    $event = $stopWatch->stop('orm');

    return new JsonResponse([
        'time' => (string) $event,
        'products' => $products,
    ]);
}
```

### Blackfire profiles:

**Eloquent:**
![Eloquent profile](/assets/img/2024-10-14/eloquent-profile.png)

**Doctrine:**
![Doctrine profile](/assets/img/2024-10-14/doctrine-profile.png)

As we can see in the case of Doctrine, after the slow hydration process, the data are already in the objects, so we can easily map them to the array. After hydration in Eloquent, there are still some steps required to get the data 🤯

![Eloquent hydration](/assets/img/2024-10-14/eloquent-profile-2.png)

I thought that Eloquent would be faster because it’s based on the simpler pattern Active Record. Doctrine is more complicated because it’s based on the Data mapper pattern. And here is another comparison in the tool AB:

**Eloquent:**
```
Time taken for tests:   50.842 seconds
Complete requests:      1000
Requests per second:    19.67 [#/sec] (mean)
Time per request:       50.842 [ms] (mean)
```

**Doctrine:**
```
Time taken for tests:   39.943 seconds
Complete requests:      1000
Requests per second:    25.04 [#/sec] (mean)
Time per request:       39.943 [ms] (mean)
```

On average Doctrine is 10 ms faster per request.

## Conclusion
- the best way is to use read models especially if you read a lot of data from the database: [Read more](https://sarvendev.com/2024/10/unlocking-orm-performance-the-essential-role-of-read-models/)
- keep in mind that reading data will be slower in Eloquent than in Doctrine if you use heavy ORM models to read data
