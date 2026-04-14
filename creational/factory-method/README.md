# Factory Method :factory:

Factory Method is a creational design pattern that provides an interface for creating objects in a superclass, but allows subclasses to alter the type of objects that will be created. [Theory](https://refactoring.guru/design-patterns/factory-method).

## Table of Contents

1. [Manager](#1-manager)
2. [SessionManager](#2-sessionmanager)

## 1. Manager

> File: [src/Illuminate/Support/Manager.php](https://github.com/laravel/framework/blob/5cc435df7a99231b1504f100c9f55e44a08bd210/src/Illuminate/Support/Manager.php)

Laravel's `Manager` abstract class is the **Creator** in the Factory Method pattern. It defines `createDriver()` — the factory method — which translates a driver name into a convention-named method call and delegates the actual object creation to whichever subclass is in use:

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

`Manager` is declared `abstract` and never instantiated directly. It defines two things: the abstract `getDefaultDriver()` that each subclass must implement to declare its default, and `createDriver()` which is the fixed factory method algorithm — check custom creators first, then call `create{X}Driver()` on the current object. The `Manager` class itself creates nothing; it only defines when and how creation is triggered. The actual **ConcreteProduct** objects are produced by the `create{X}Driver()` methods that concrete subclasses provide.

## 2. SessionManager

> File: [src/Illuminate/Session/SessionManager.php](https://github.com/laravel/framework/blob/5cc435df7a99231b1504f100c9f55e44a08bd210/src/Illuminate/Session/SessionManager.php)

`SessionManager` extends `Manager` directly and acts as the **ConcreteCreator** for session drivers. By inheriting from `Manager`, it participates in the Factory Method hierarchy: it inherits `createDriver()` unchanged and only supplies the concrete factory methods — one per supported session backend:

```php
class SessionManager extends Manager
{
    /* ... CODE ... */

    protected function createNullDriver()
    {
        return $this->buildSession(new NullSessionHandler);
    }

    protected function createArrayDriver()
    {
        return $this->buildSession(new ArraySessionHandler(
            $this->config->get('session.lifetime')
        ));
    }

    protected function createCookieDriver()
    {
        return $this->buildSession(new CookieSessionHandler(
            $this->container->make('cookie'), $this->config->get('session.lifetime')
        ));
    }

    protected function createNativeDriver()
    {
        $lifetime = $this->config->get('session.lifetime');

        return $this->buildSession(new FileSessionHandler(
            $this->container->make('files'), $this->config->get('session.files'), $lifetime
        ));
    }

    protected function createDatabaseDriver()
    {
        $table = $this->config->get('session.table');

        $lifetime = $this->config->get('session.lifetime');

        return $this->buildSession(new DatabaseSessionHandler(
            $this->getDatabaseConnection(), $table, $lifetime, $this->container
        ));
    }

    /* ... CODE ... */

    public function getDefaultDriver()
    {
        return $this->config->get('session.driver');
    }
}
```

Each `create{X}Driver()` method is a **ConcreteProduct** factory: it constructs the appropriate session handler and wraps it in a `Store` (the **AbstractProduct** — all session backends conform to `SessionHandlerInterface`). When a developer calls `Session::driver('database')`, the inherited `createDriver()` translates that to `createDatabaseDriver()` and returns a `Store` backed by the database. The calling code only sees a `Store`; it never knows which backend was created.

Other direct subclasses of `Manager` — including `HashManager`, `BroadcastManager`, and `AuthManager` — follow the same structure: each extends `Manager`, implements `getDefaultDriver()`, and defines its own family of `create{X}Driver()` factory methods. Laravel's Factory Method pattern is the repeated application of this one inheritance relationship across many subsystems.
