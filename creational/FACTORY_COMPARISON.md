# Factory Comparison

You can find references to these terms [factories] all around the web. Though they may look similar, they all have different meanings. A lot of people don’t realize that, which leads to confusion and misunderstanding. [Theory](https://refactoring.guru/design-patterns/factory-comparison).

## Table of Contents

1. [Factory](#1-factory)
2. [Creation method](#2-creation-method)
3. [Static creation (or factory) method](#3-static-creation-or-factory-method)
4. [Simple factory](#4-simple-factory-pattern)
5. [Factory Method pattern](#5-factory-method-pattern)
6. [Abstract Factory pattern](#6-abstract-factory-pattern)

## 1. Factory

> Factory is an ambiguous term that stands for a function, method or class that supposed to be producing something. Most commonly, factories produce objects. But they may also produce files, records in databases, etc.

*Work in progress.*

## 2. Creation method

> Creation method defined in the Refactoring To Patterns book as “a method that creates objects.” This means that every result of a factory method pattern is a “creation method” but not necessarily the reverse. 

*Work in progress.*

## 3. Static creation (or factory) method

> Static creation method is a creation method declared as static. In other words, it can be called on a class and doesn’t require an object to be created.

### Example

`Model::create()` method is a static creation method that creates your models. Under the hood it uses magic methods [__call()](https://github.com/laravel/framework/blob/5cc435df7a99231b1504f100c9f55e44a08bd210/src/Illuminate/Database/Eloquent/Model.php#L2213) and [__callStatic()](https://github.com/laravel/framework/blob/5cc435df7a99231b1504f100c9f55e44a08bd210/src/Illuminate/Database/Eloquent/Model.php#L2233) to forward the call:

```php
 /**
 * Handle dynamic method calls into the model.
 *
 * @param  string  $method
 * @param  array  $parameters
 * @return mixed
 */
public function __call($method, $parameters)
{
    /* ... CODE ... */

    return $this->forwardCallTo($this->newQuery(), $method, $parameters);
}
```

It forwards the call to  [Illuminate\Database\Eloquent\Builder::create()](https://github.com/laravel/framework/blob/5cc435df7a99231b1504f100c9f55e44a08bd210/src/Illuminate/Database/Eloquent/Builder.php#L966) that utilizes [Illuminate\Database\Eloquent\Model::newInstance()](https://github.com/laravel/framework/blob/5cc435df7a99231b1504f100c9f55e44a08bd210/src/Illuminate/Database/Eloquent/Model.php#L511) function:

```php
/**
 * Create a new instance of the given model.
 *
 * @param  array  $attributes
 * @param  bool  $exists
 * @return static
 */
public function newInstance($attributes = [], $exists = false)
{
    // This method just provides a convenient way for us to generate fresh model
    // instances of this current model. It is particularly useful during the
    // hydration of new objects via the Eloquent query builder instances.
    $model = new static;

    $model->exists = $exists;

    $model->setConnection(
        $this->getConnectionName()
    );

    $model->setTable($this->getTable());

    $model->mergeCasts($this->casts);

    $model->fill((array) $attributes);

    return $model;
}
```

## 4. Simple factory pattern

> The Simple factory pattern  describes a class that has one creation method with a large conditional that based on method parameters chooses which product class to instantiate and then return.

*Work in progress.*

## 5. Factory Method pattern

> The Factory Method  is a creational design pattern that provides an interface for creating objects but allows subclasses to alter the type of an object that will be created.

*Work in progress.*

## 6. Abstract Factory pattern

> The Abstract Factory  is a creational design pattern that allows producing families of related or dependent objects without specifying their concrete classes.

*Work in progress.*