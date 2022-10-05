# Builder :bricks:

Builder is a creational design pattern that lets you construct complex objects step by step. The pattern allows you to produce different types and representations of an object using the same construction code. [Theory](https://refactoring.guru/design-patterns/builder).

## Table of Contents

1. [Eloquent Builder](#1-eloquent-builder)

## 1. Eloquent Builder

> File: [src/Illuminate/Database/Eloquent/Builder.php](https://github.com/laravel/framework/blob/5cc435df7a99231b1504f100c9f55e44a08bd210/src/Illuminate/Database/Eloquent/Builder.php)

`Eloquent/Builder` allows a developer to create your database query step by step. Consider the following code in an app:

```php
User::where('active', true)
    ->with('payments')
    ->get();
```

This chain of conditions creates a query. First, let's add a [where()](https://github.com/laravel/framework/blob/5cc435df7a99231b1504f100c9f55e44a08bd210/src/Illuminate/Database/Eloquent/Builder.php#L269) condition:

```php
/**
 * Add a basic where clause to the query.
 *
 * @param  \Closure|string|array|\Illuminate\Database\Query\Expression  $column
 * @param  mixed  $operator
 * @param  mixed  $value
 * @param  string  $boolean
 * @return $this
 */
public function where($column, $operator = null, $value = null, $boolean = 'and')
{
    if ($column instanceof Closure && is_null($operator)) {
        $column($query = $this->model->newQueryWithoutRelationships());

        $this->query->addNestedWhereQuery($query->getQuery(), $boolean);
    } else {
        $this->query->where(...func_get_args());
    }

    return $this;
}
```

Second, let's instruct the builder [to eager load](https://github.com/laravel/framework/blob/5cc435df7a99231b1504f100c9f55e44a08bd210/src/Illuminate/Database/Eloquent/Builder.php#L1388) relations:

```php
/**
 * Set the relationships that should be eager loaded.
 *
 * @param  string|array  $relations
 * @param  string|\Closure|null  $callback
 * @return $this
 */
public function with($relations, $callback = null)
{
    if ($callback instanceof Closure) {
        $eagerLoad = $this->parseWithRelations([$relations => $callback]);
    } else {
        $eagerLoad = $this->parseWithRelations(is_string($relations) ? func_get_args() : $relations);
    }

    $this->eagerLoad = array_merge($this->eagerLoad, $eagerLoad);

    return $this;
}
```

At this point the builder is done. [get()](https://github.com/laravel/framework/blob/5cc435df7a99231b1504f100c9f55e44a08bd210/src/Illuminate/Database/Eloquent/Builder.php#L670) function is responsible for executing a query and returning a collection.

```php
/**
 * Execute the query as a "select" statement.
 *
 * @param  array|string  $columns
 * @return \Illuminate\Database\Eloquent\Collection|static[]
 */
public function get($columns = ['*'])
{
    $builder = $this->applyScopes();

    // If we actually found models we will also eager load any relationships that
    // have been specified as needing to be eager loaded, which will solve the
    // n+1 query issue for the developers to avoid running a lot of queries.
    if (count($models = $builder->getModels($columns)) > 0) {
        $models = $builder->eagerLoadRelations($models);
    }

    return $builder->getModel()->newCollection($models);
}
```
