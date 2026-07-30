# Simple Approach to Complex Query Filtering in Laravel

Filtering data in Laravel often starts simple but quickly devolves into a mess of `if ($request->has(...))` statements scattered across controllers. As your application grows, these queries become harder to maintain and test.

In this article, I'll show you how I tackled complex query filtering using a simple, composable, and type-safe approach.

---

## The Goal: Clean Controllers

The objective was to transform a potentially bloated controller into something declarative and readable:

```php
public function __invoke(IndexProductsRequest $request): Response
{
    $filter = $request->getFilter();

    $products = Product::search($filter->searchFilter->value())
        ->query(fn (ProductBuilder $query) => $query
            ->tap(fn ($q) => $filter->applyEloquent($q))
        )
        ->tap(fn ($scout) => $filter->applyScout($scout))
        ->paginate($filter->perPageFilter->value());

    return Inertia::render('admin/products/index', [
        'products' => $products,
        'filters' => ProductFilterData::fromProductFilter($filter),
    ]);
}
```

## The Architecture

The system relies on four key components:

1.  **Custom Form Requests**: For encapsulating filter initialization.
2.  **Composite Filters**: To aggregate multiple filtering rules.
3.  **Leaf Filters**: Single-responsibility classes for specific fields.
4.  **Data Transfer Objects (DTOs)**: To bridge the state to the frontend with type safety.

### 1. The Request as a Factory

Instead of building filters in the controller, we let the `FormRequest` handle it. This keeps the controller focused on the high-level flow.

```php
class IndexProductsRequest extends IndexRequest
{
    public function getFilter(): ProductFilter {
        return new ProductFilter(request: $this);
    }
}
```

### 2. Composition Over Inheritance

A `ProductFilter` isn't just one big query; it’s a collection of smaller filters. It inherits base filters (like Search, Sort, Pagination) and adds domain-specific ones.

```php
class ProductFilter extends IndexFilter
{
    public ProductStockStatusFilter $productStockStatusFilter;

    public function __construct(Request $request) {
        parent::__construct($request);
        $this->productStockStatusFilter = new ProductStockStatusFilter($request);
    }

    public function applyEloquent(Builder $builder): Builder {
        $builder = parent::applyEloquent($builder);
        return $this->productStockStatusFilter->apply($builder);
    }
}
```

### 3. The "Leaf" Filter: Single Responsibility

Each individual filter class is responsible for exactly one thing: reading its value from the request and applying it to the query builder.

```php
class ProductStockStatusFilter implements EloquentQueryFilter
{
    protected string $key = 'stockStatus';

    public function apply(ProductBuilder|Builder $builder): ProductBuilder {
        $value = $this->value();
        return $value ? $builder->whereStockStatus($value) : $builder;
    }

    public function value(): ?ProductStockStatus {
        return ProductStockStatus::tryFrom($this->request->input($this->key));
    }
}
```

### 4. Closing the Loop with DTOs

When using Inertia.js or any SPA, you need to send the current filter state back to the frontend. We use a DTO to ensure the frontend knows exactly what filters are active, complete with TypeScript support.

```php
#[TypeScript]
class ProductFilterData extends IndexFilterData
{
    public function __construct(
        public ?string $search,
        public ?ProductStockStatus $stockStatus,
        // ...
    ) {}

    public static function fromProductFilter(ProductFilter $filter): static {
        return new ProductFilterData(
            search: $filter->searchFilter->input(),
            stockStatus: $filter->productStockStatusFilter->value(),
        );
    }
}
```

## Why This Works

1.  **Reusability**: `SearchFilter` or `PerPageFilter` can be dropped into any new module.
2.  **Scout & Eloquent Integration**: The same filter object can handle both full-text search (Scout) and standard database queries (Eloquent).
3.  **Testability**: You can test individual filter classes in isolation.
4.  **Type Safety**: Enums and DTOs prevent "magic strings" from causing bugs.

By separating the **extraction** of filter values from the **application** of query logic, we achieve a clean, maintainable system that stays simple regardless of how many filters you add.

---

*What do you think? How do you handle complex filtering in your Laravel apps?*

