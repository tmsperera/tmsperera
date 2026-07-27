# Mastering Complex Query Filtering in Laravel: The Simple Way

When building data-rich applications, we often end up with controllers that are drowning in conditional logic. `if ($request->has('search'))`, `if ($request->has('order_by'))`, `if ($request->has('status'))` – sound familiar?

This "Controller Bloat" makes code hard to read, harder to test, and nearly impossible to reuse.

In this article, I’ll show you a simple, object-oriented approach to tackling complex query filtering using **Composition**.

---

## 1. The Core Interface: `EloquentQueryFilter`

Everything starts with a simple contract. We want every filter to do two things:
1. **Apply** itself to an Eloquent builder.
2. **Expose** its current value (useful for UI or Data Objects).

```php
interface EloquentQueryFilter
{
    public function apply(Builder $builder): Builder;
    public function value(): mixed;
}
```

## 2. Granular, Reusable "Leaf" Filters

Instead of one giant filter class, we create tiny, focused ones. For example, an `OrderByFilter` only cares about two things: the allowed columns and the sort direction.

```php
class OrderByFilter implements EloquentQueryFilter
{
    public function apply(Builder $builder): Builder
    {
        $value = $this->value();
        if (!$value) return $builder;

        return $builder->orderBy($value->column, $value->ascending ? 'asc' : 'desc');
    }

    public function value(): ?OrderByFilterData
    {
        // ... extract and validate from Request
    }
}
```

## 3. The Power of Composition: `IndexQueryFilter`

Most of our index pages share common filtering needs: search, trash status, ordering, and pagination. We can group these into a base `IndexQueryFilter`.

```php
class IndexQueryFilter implements EloquentQueryFilter
{
    public SearchFilter $search;
    public OrderByFilter $orderBy;
    // ...

    public function __construct(Request $request) {
        $this->search = new SearchFilter($request);
        $this->orderBy = new OrderByFilter($request, ['created_at']);
        // ...
    }

    public function apply(Builder $builder): Builder
    {
        $this->orderBy->apply($builder);
        $this->trashStatus->apply($builder);
        return $builder;
    }
}
```

## 4. Extending for Domain Specifics

When we need product-specific filters (like stock status), we simply extend the base filter. No need to rewrite the search or ordering logic!

```php
class ProductFilter extends IndexQueryFilter
{
    public ProductStockStatusFilter $stockStatus;

    public function __construct(Request $request, array $allowedOrderByColumns) {
        parent::__construct($request, $allowedOrderByColumns);
        $this->stockStatus = new ProductStockStatusFilter($request);
    }

    public function apply(Builder $builder): Builder
    {
        parent::apply($builder); // Apply common filters
        $this->stockStatus->apply($builder); // Apply product-specific filter logic
        return $builder;
    }
}
```

## 5. The Result: Clean, Declarative Controllers (including Scout!)

Now, look at how clean the controller becomes. The filtering logic is entirely decoupled from the HTTP layer. This pattern also works beautifully when you're using **Laravel Scout** for searching.

```php
public function __invoke(Request $request)
{
    $queryFilter = new ProductFilter($request);

    return Product::search($queryFilter->search->value())
        ->query(fn (ProductBuilder $query) => $query
            ->tap(fn (ProductBuilder $query) => $queryFilter->apply($query))
        )
        ->paginate($queryFilter->perPage->value());
}
```

---

## Bonus: Interface for Different Builders

The same pattern can be applied to other builders, like **Laravel Scout**. By defining a `ScoutQueryFilter` interface, you can handle search engine specific logic (like `withTrashed()` on a Scout builder) with the same clean, composable approach.

```php
interface ScoutQueryFilter
{
    public function apply(ScoutBuilder $builder): ScoutBuilder;
}
```

Controllers (including Scout!)

```php
public function __invoke(Request $request)
{
    $queryFilter = new ProductFilter($request);
    $scoutFilter = new ScoutTrashStatusFilter($request);

    return Product::search($queryFilter->search->value())
        ->query(fn (ProductBuilder $query) => $query
            ->tap(fn (ProductBuilder $query) => $queryFilter->apply($query))
        )
        ->tap(fn (ScoutBuilder $scoutBuilder) => $scoutFilter->apply($scoutBuilder))
        ->paginate($queryFilter->perPage->value());
}
```

---

## Why This Works

1.  **Readability**: The controller tells you *what* it's doing, not *how* to filter the database.
2.  **Reusability**: `OrderByFilter` or `SearchFilter` can be used across any model.
3.  **Testability**: You can unit test individual filters without spinning up a full HTTP request or a database.
4.  **Type Safety**: By using DTOs in the `value()` method, you get IDE autocompletion and prevent "stringly-typed" bugs.

Tackling complexity doesn't require complex tools—sometimes, a little bit of object-oriented composition is all you need.

