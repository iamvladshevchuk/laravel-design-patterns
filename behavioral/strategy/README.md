# Strategy :military_helmet:

Strategy is a behavioral design pattern that lets you define a family of algorithms, put each of them into a separate class, and make their objects interchangeable. [Theory](https://refactoring.guru/design-patterns/strategy).

## Table of Contents

1. [Cache](#1-cache)

## 1. Cache

> Strategy (Interface): [src/Illuminate/Contracts/Cache/Store.php](https://github.com/laravel/framework/blob/5cc435df7a99231b1504f100c9f55e44a08bd210/src/Illuminate/Contracts/Cache/Store.php)

> Concrete Strategies: 
> 1. [scr/Illuminate/Cache/ApcStore.php](https://github.com/laravel/framework/blob/5cc435df7a99231b1504f100c9f55e44a08bd210/src/Illuminate/Cache/ApcStore.php)
> 2. [src/Illuminate/Cache/ArrayStore.ph](https://github.com/laravel/framework/blob/5cc435df7a99231b1504f100c9f55e44a08bd210/src/Illuminate/Cache/ArrayStore.php)
> 3. [src/Illuminate/Cache/DatabaseStore.php](https://github.com/laravel/framework/blob/5cc435df7a99231b1504f100c9f55e44a08bd210/src/Illuminate/Cache/DatabaseStore.php)
> 4. [src/Illuminate/Cache/DynamoDbStore.phpp](https://github.com/laravel/framework/blob/5cc435df7a99231b1504f100c9f55e44a08bd210/src/Illuminate/Cache/DynamoDbStore.php)
> 5. [src/Illuminate/Cache/FileStore.php](https://github.com/laravel/framework/blob/5cc435df7a99231b1504f100c9f55e44a08bd210/src/Illuminate/Cache/FileStore.php)
> 6. [src/Illuminate/Cache/MemcachedStore.php](https://github.com/laravel/framework/blob/5cc435df7a99231b1504f100c9f55e44a08bd210/src/Illuminate/Cache/MemcachedStore.php)
> 7. [src/Illuminate/Cache/NullStore.php](https://github.com/laravel/framework/blob/5cc435df7a99231b1504f100c9f55e44a08bd210/src/Illuminate/Cache/NullStore.php)
> 8. [src/Illuminate/Cache/RedisStore.php](https://github.com/laravel/framework/blob/5cc435df7a99231b1504f100c9f55e44a08bd210/src/Illuminate/Cache/RedisStore.php)

Laravel deals with many drivers that manage cache. You can store your cache in file, Redis, database, etc. Laravel uses single [interface](https://github.com/laravel/framework/blob/5cc435df7a99231b1504f100c9f55e44a08bd210/src/Illuminate/Contracts/Cache/Store.php) for them:

```php
interface Store
{
    /**
     * Retrieve an item from the cache by key.
     *
     * @param  string|array  $key
     * @return mixed
     */
    public function get($key);

    /* ... CODE ... */

    /**
     * Store an item in the cache for a given number of seconds.
     *
     * @param  string  $key
     * @param  mixed  $value
     * @param  int  $seconds
     * @return bool
     */
    public function put($key, $value, $seconds);

    /* ... CODE ... */
}
```

Laravel has many implementation of the interface. [One](https://github.com/laravel/framework/blob/5cc435df7a99231b1504f100c9f55e44a08bd210/src/Illuminate/Cache/FileStore.php) of them:

```php
class FileStore implements Store, LockProvider
{
    /* ... CODE ... */

    /**
     * Retrieve an item from the cache by key.
     *
     * @param  string|array  $key
     * @return mixed
     */
    public function get($key)
    {
        return $this->getPayload($key)['data'] ?? null;
    }

    /**
     * Store an item in the cache for a given number of seconds.
     *
     * @param  string  $key
     * @param  mixed  $value
     * @param  int  $seconds
     * @return bool
     */
    public function put($key, $value, $seconds)
    {
        $this->ensureCacheDirectoryExists($path = $this->path($key));

        $result = $this->files->put(
            $path, $this->expiration($seconds).serialize($value), true
        );

        if ($result !== false && $result > 0) {
            $this->ensurePermissionsAreCorrect($path);

            return true;
        }

        return false;
    }

    /* ... CODE ... */
}
```

These stores are interchangeable. Laravel chooses a store based on config and whatever driver you choose, but it provides a developer with the same interface to work with.
