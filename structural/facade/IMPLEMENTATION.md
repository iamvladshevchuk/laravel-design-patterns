# Facade Implementation

> File: [src/Illuminate/Support/Facades/Facade.php](https://github.com/laravel/framework/blob/5cc435df7a99231b1504f100c9f55e44a08bd210/src/Illuminate/Support/Facades/Facade.php)

Laravel has [a set of Facades](https://github.com/laravel/framework/tree/9.x/src/Illuminate/Support/Facades) that developers use to access the most crucial parts of the framework. When a developer uses a facade through a static method (e.g. `Illuminate\Support\Facades\Log::info()`), it calls a magic method [Facade::__callStatic()](https://github.com/laravel/framework/blob/5cc435df7a99231b1504f100c9f55e44a08bd210/src/Illuminate/Support/Facades/Facade.php#L321).

```php
/**
 * Handle dynamic, static calls to the object.
 *
 * @param  string  $method
 * @param  array  $args
 * @return mixed
 *
 * @throws \RuntimeException
 */
public static function __callStatic($method, $args)
{
    $instance = static::getFacadeRoot();

    if (! $instance) {
        throw new RuntimeException('A facade root has not been set.');
    }

    return $instance->$method(...$args);
}
```

A magic method resolves a facade using an accessor in [Facade::getFacadeRoot()](https://github.com/laravel/framework/blob/5cc435df7a99231b1504f100c9f55e44a08bd210/src/Illuminate/Support/Facades/Facade.php#L186) function:

```php
/**
 * Get the root object behind the facade.
 *
 * @return mixed
 */
public static function getFacadeRoot()
{
    return static::resolveFacadeInstance(static::getFacadeAccessor());
}
```

A facade accessor is a unique name for a facade that Laravel binds to a concrete implementation. To understand this you should have a basic understanding of how [Service Container](https://laravel.com/docs/9.x/container) works. 

That's how [Illuminate\Support\Facades\Log](https://github.com/laravel/framework/blob/5cc435df7a99231b1504f100c9f55e44a08bd210/src/Illuminate/Support/Facades/Log.php) class looks, for example. It has a single function `getFacadeAccessor()` that provides a name of the facade. This name is used to bind this facade to a concrete implementation.

```php
class Log extends Facade
{
    /**
     * Get the registered name of the component.
     *
     * @return string
     */
    protected static function getFacadeAccessor()
    {
        return 'log';
    }
}
```

What class a developer uses when he/she calls `Illuminate\Support\Facades\Log::info()`? We can see clearly that there's no `info()` method in the class. The Facade responsible only for resolving the class. To put it simply, when you call `Log::info()` method, the facade above does someting like this:

```php
$log = app('log'); // that's what __callStatic() method does under the hood
$log->info(); // then a developer can use any method available from a resolved class
```

What class a developer actually uses is defined in [Service Providers](https://laravel.com/docs/9.x/providers). Let's roll with the `Log` example. Here's a [Service Provider](https://github.com/laravel/framework/blob/5cc435df7a99231b1504f100c9f55e44a08bd210/src/Illuminate/Log/LogServiceProvider.php) for it:

```php
class LogServiceProvider extends ServiceProvider
{
    /**
     * Register the service provider.
     *
     * @return void
     */
    public function register()
    {
        $this->app->singleton('log', function ($app) {
            return new LogManager($app);
        });
    }
}
```

The class binds a string `log` to [Illuminate\Log\LogManager](https://github.com/laravel/framework/blob/5cc435df7a99231b1504f100c9f55e44a08bd210/src/Illuminate/Log/LogManager.php) class. That's the class developer actually use when they call `Illuminate\Support\Facades\Log`. The [info()](https://github.com/laravel/framework/blob/5cc435df7a99231b1504f100c9f55e44a08bd210/src/Illuminate/Log/LogManager.php#L672) method mentioned earlier is here:

```php
/**
 * Interesting events.
 *
 * Example: User logs in, SQL logs.
 *
 * @param  string  $message
 * @param  array  $context
 * @return void
 */
public function info($message, array $context = []): void
{
    $this->driver()->info($message, $context);
}
```

**Summary**: Laravel's Facade provides a simple way to access a class inside the framework. It uses `Service Container` to bind a string identifier to an actual implementation. When a developer calls a static method in a facade, `__callStatic` method resolves a class based on an indentifier, then runs the method.
