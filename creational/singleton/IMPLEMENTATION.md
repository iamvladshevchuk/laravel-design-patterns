## Singleton Implementation

> File: [src/Illuminate/Container/Container.php](https://github.com/laravel/framework/blob/5cc435df7a99231b1504f100c9f55e44a08bd210/src/Illuminate/Container/Container.php#L379)

Although the [usual approach](https://refactoring.guru/design-patterns/singleton/php/example) to implement a singleton in PHP is to make `__construct()` and `__clone()` functions protected (thus making it impossible to create an instance without static creational method), Laravel implements it differently. Laravel relies on you to use `app()->singleton()` function when you want to create a singleton. It uses [Service Container](https://laravel.com/docs/9.x/container) to control whether user gets a new instance or an old one. Here's a chain of functions called when user uses [app()->singleton()](https://github.com/laravel/framework/blob/5cc435df7a99231b1504f100c9f55e44a08bd210/src/Illuminate/Container/Container.php#L379):

```php
/**
 * Register a shared binding in the container.
 *
 * @param  string  $abstract
 * @param  \Closure|string|null  $concrete
 * @return void
 */
public function singleton($abstract, $concrete = null)
{
    $this->bind($abstract, $concrete, true);
}
```

When developer calls `singleton`, it binds the abstract class to the to the concrete one. And tags it `shared` (that's what the last `true` means in the function above). I'll omit some code to show you [how it's implemented](https://github.com/laravel/framework/blob/5cc435df7a99231b1504f100c9f55e44a08bd210/src/Illuminate/Container/Container.php#L239).

```php
/**
 * Register a binding with the container.
 *
 * @param  string  $abstract
 * @param  \Closure|string|null  $concrete
 * @param  bool  $shared
 * @return void
 *
 * @throws \TypeError
 */
public function bind($abstract, $concrete = null, $shared = false)
{
    /* ... CODE ... */

    $this->bindings[$abstract] = compact('concrete', 'shared');

    /* ... CODE ... */
}
```

When developer tries to resolve the abstract class using [app()->make()](https://github.com/laravel/framework/blob/5cc435df7a99231b1504f100c9f55e44a08bd210/src/Illuminate/Container/Container.php#L681) function, every shared binding will be memorised and returned to developer. That's how it looks [in the code](https://github.com/laravel/framework/blob/5cc435df7a99231b1504f100c9f55e44a08bd210/src/Illuminate/Container/Container.php#L713):

```php
/**
 * Resolve the given type from the container.
 *
 * @param  string|callable  $abstract
 * @param  array  $parameters
 * @param  bool  $raiseEvents
 * @return mixed
 *
 * @throws \Illuminate\Contracts\Container\BindingResolutionException
 * @throws \Illuminate\Contracts\Container\CircularDependencyException
 */
protected function resolve($abstract, $parameters = [], $raiseEvents = true)
{    
    /* ... CODE ... */

    // If an instance of the type is currently being managed as a singleton we'll
    // just return an existing instance instead of instantiating new instances
    // so the developer can keep using the same objects instance every time.
    if (isset($this->instances[$abstract]) && ! $needsContextualBuild) {
        return $this->instances[$abstract];
    }

    /* ... CODE ... */

    // If the requested type is registered as a singleton we'll want to cache off
    // the instances in "memory" so we can return it later without creating an
    // entirely new instance of an object on each subsequent request for it.
    if ($this->isShared($abstract) && ! $needsContextualBuild) {
        $this->instances[$abstract] = $object;
    }

    /* ... CODE ... */
}
```

**Summary:** Laravel singleton pattern is implemented using a [Service Container](https://laravel.com/docs/9.x/container). It allows to resolve abstract classes to return the same instance every time it's called.
