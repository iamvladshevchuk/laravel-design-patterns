# Singleton :standing_man:

Singleton is a creational design pattern that lets you ensure that a class has only one instance, while providing a global access point to this instance. [Theory](https://refactoring.guru/design-patterns/singleton).

> Before you continue, take a look [how Laravel implemented Singleton](./IMPLEMENTATION.md) in the framework.

## Table of Contents

1. [Kernel as Singleton](#1-kernel-as-singleton)
2. [Facade as Singleton](#2-facade-as-singleton)
    * [How it's implemented](#how-its-implemented)
    1. [Cache](#21-cache)
    2. [Session](#22-session)

## 1. Kernel as Singleton

> File: [bootstrap/app.php](https://github.com/laravel/laravel/blob/5138bc36dbc884098ea68942e805b2267e7a627f/bootstrap/app.php#L18)

[bootstrap/app.php](https://github.com/laravel/laravel/blob/5138bc36dbc884098ea68942e805b2267e7a627f/bootstrap/app.php) runs on every request in your app. It binds the kernel of the app to the interface (contract). Every time it's accessed down the line, it returns the same instance. Thus, it's a singleton.

```php
/*
|--------------------------------------------------------------------------
| Bind Important Interfaces
|--------------------------------------------------------------------------
|
| Next, we need to bind some important interfaces into the container so
| we will be able to resolve them when needed. The kernels serve the
| incoming requests to this application from both the web and CLI.
|
*/

$app->singleton(
    Illuminate\Contracts\Http\Kernel::class,
    App\Http\Kernel::class
);

$app->singleton(
    Illuminate\Contracts\Console\Kernel::class,
    App\Console\Kernel::class
);

$app->singleton(
    Illuminate\Contracts\Debug\ExceptionHandler::class,
    App\Exceptions\Handler::class
);
```

## 2. Facade as Singleton

Facades are always registered as singletons in [Service Providers](https://laravel.com/docs/9.x/providers).

> Want to know how Facade works in Laravel? Check [Facade Design Pattern](/structural/facade/) to understand how it's implemented.

### 2.1. Cache

> File: [src/Illuminate/Cache/CacheServiceProvider.php](https://github.com/laravel/framework/blob/5cc435df7a99231b1504f100c9f55e44a08bd210/src/Illuminate/Cache/CacheServiceProvider.php)

```php
public function register()
{
    $this->app->singleton('cache', function ($app) {
        return new CacheManager($app);
    });

    /* ... CODE ... */
}
```

### 2.2. Session

> File: [src/Illuminate/Session/SessionServiceProvider.php](https://github.com/laravel/framework/blob/5cc435df7a99231b1504f100c9f55e44a08bd210/src/Illuminate/Session/SessionServiceProvider.php)

```php
protected function registerSessionManager()
{
    $this->app->singleton('session', function ($app) {
        return new SessionManager($app);
    });
}
```
