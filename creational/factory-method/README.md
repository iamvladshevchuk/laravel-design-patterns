# Factory Method :factory:

Factory Method is a creational design pattern that provides an interface for creating objects in a superclass, but allows subclasses to alter the type of objects that will be created. [Theory](https://refactoring.guru/design-patterns/factory-method).

## Table of Contents

1. [Manager](#1-manager)
2. [CacheManager](#2-cachemanager)

## 1. Manager

> File: [src/Illuminate/Support/Manager.php](https://github.com/laravel/framework/blob/5cc435df7a99231b1504f100c9f55e44a08bd210/src/Illuminate/Support/Manager.php)

Laravel's `Manager` abstract class is the backbone of the Factory Method pattern across the framework. It defines `createDriver()` — the factory method — which uses PHP's dynamic method dispatch to delegate driver creation to concrete subclasses:

```php
abstract class Manager
{
    /* ... CODE ... */

    /**
     * Get the default driver name.
     *
     * @return string
     */
    abstract public function getDefaultDriver();

    /**
     * Get a driver instance.
     *
     * @param  string|null  $driver
     * @return mixed
     *
     * @throws \InvalidArgumentException
     */
    public function driver($driver = null)
    {
        $driver = $driver ?: $this->getDefaultDriver();

        /* ... CODE ... */

        if (! isset($this->drivers[$driver])) {
            $this->drivers[$driver] = $this->createDriver($driver);
        }

        return $this->drivers[$driver];
    }

    /**
     * Create a new driver instance.
     *
     * @param  string  $driver
     * @return mixed
     *
     * @throws \InvalidArgumentException
     */
    protected function createDriver($driver)
    {
        // First, we will determine if a custom driver creator exists for the given driver and
        // if it does not we will check for a creator method for the driver. Custom creator
        // callbacks allow developers to build their own "drivers" easily using Closures.
        if (isset($this->customCreators[$driver])) {
            return $this->callCustomCreator($driver);
        } else {
            $method = 'create'.Str::studly($driver).'Driver';

            if (method_exists($this, $method)) {
                return $this->$method();
            }
        }

        throw new InvalidArgumentException("Driver [$driver] not supported.");
    }
}
```

Two abstract ideas are baked in here. First, `getDefaultDriver()` is abstract, forcing every subclass to declare which driver to use when none is specified. Second, `createDriver()` translates a driver name like `"file"` or `"redis"` into a method call like `createFileDriver()` or `createRedisDriver()` — methods that each subclass implements.

The `Manager` class itself never creates any concrete driver. It defines the protocol for when and how the factory method is called, while the actual object creation is delegated to subclasses. This is exactly the Creator/Concrete Creator split in the Factory Method pattern.

## 2. CacheManager

> File: [src/Illuminate/Cache/CacheManager.php](https://github.com/laravel/framework/blob/5cc435df7a99231b1504f100c9f55e44a08bd210/src/Illuminate/Cache/CacheManager.php)

`CacheManager` is one of many concrete creators in Laravel that follow the Factory Method convention. It defines a family of `create{X}Driver()` methods, each producing a `Repository` wrapping a different cache backend:

```php
class CacheManager implements FactoryContract
{
    /* ... CODE ... */

    protected function resolve($name)
    {
        $config = $this->getConfig($name);

        /* ... CODE ... */

        $driverMethod = 'create'.ucfirst($config['driver']).'Driver';

        if (method_exists($this, $driverMethod)) {
            return $this->{$driverMethod}($config);
        } else {
            throw new InvalidArgumentException("Driver [{$config['driver']}] is not supported.");
        }
    }

    protected function createFileDriver(array $config)
    {
        return $this->repository(new FileStore($this->app['files'], $config['path'], $config['permission'] ?? null));
    }

    protected function createRedisDriver(array $config)
    {
        $redis = $this->app['redis'];

        $connection = $config['connection'] ?? 'default';

        $store = new RedisStore($redis, $this->getPrefix($config), $connection);

        return $this->repository(
            $store->setLockConnection($config['lock_connection'] ?? $connection)
        );
    }

    protected function createMemcachedDriver(array $config)
    {
        $prefix = $this->getPrefix($config);

        $memcached = $this->app['memcached.connector']->connect(
            $config['servers'],
            $config['persistent_id'] ?? null,
            $config['options'] ?? [],
            array_filter($config['sasl'] ?? [])
        );

        return $this->repository(new MemcachedStore($memcached, $prefix));
    }

    protected function createNullDriver()
    {
        return $this->repository(new NullStore);
    }

    /* ... CODE ... */
}
```

When you call `Cache::store('redis')`, Laravel resolves `createRedisDriver()` and returns a `Repository` backed by Redis. Changing the driver to `"file"` calls `createFileDriver()` instead. The calling code never changes — it always receives a `Repository` with the same interface.

The same convention applies throughout Laravel. `LogManager` has `createSingleDriver()`, `createDailyDriver()`, and `createSyslogDriver()`. `SessionManager` has `createFileDriver()`, `createCookieDriver()`, and `createDatabaseDriver()`. Each manager is a concrete creator; each `create{X}Driver()` method is a factory method that produces a product conforming to a common interface.
