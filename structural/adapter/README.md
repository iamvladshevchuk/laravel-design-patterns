# Adapter :hourglass: :watch:

Adapter is a structural design pattern that allows objects with incompatible interfaces to collaborate. [Theory](https://refactoring.guru/design-patterns/adapter).

## Table of Contents

1. [Filesystem Adapter](#1-filesystem-adapter)

## 1. Filesystem Adapter

> Adapter: [src/Illuminate/Filesystem/FilesystemAdapter.php](https://github.com/laravel/framework/blob/5cc435df7a99231b1504f100c9f55e44a08bd210/src/Illuminate/Filesystem/FilesystemAdapter.php)

> Service: [League\Flysystem\Filesystem](https://github.com/thephpleague/flysystem/blob/c73c4eb31f2e883b3897ab5591aa2dbc48112433/src/Filesystem.php)

Laravel uses a 3d-party package [FlySystem](https://github.com/thephpleague/flysystem) to deal with a filesystem. In the adapter Laravel rewrites functions and adds new ones to create a class that's compatible with the framework. For example:

* Adds [exists()](https://github.com/laravel/framework/blob/5cc435df7a99231b1504f100c9f55e44a08bd210/src/Illuminate/Filesystem/FilesystemAdapter.php#L172) method instead of `has()`:

```php
/**
 * Determine if a file or directory exists.
 *
 * @param  string  $path
 * @return bool
 */
public function exists($path)
{
    return $this->driver->has($path);
}
```

* Adds [get()](https://github.com/laravel/framework/blob/5cc435df7a99231b1504f100c9f55e44a08bd210/src/Illuminate/Filesystem/FilesystemAdapter.php#L249) method instead of `read()`:

```php
/**
 * Get the contents of a file.
 *
 * @param  string  $path
 * @return string|null
 */
public function get($path)
{
    try {
        return $this->driver->read($path);
    } catch (UnableToReadFile $e) {
        throw_if($this->throwsExceptions(), $e);
    }
}
```

* Adds additional logic to [move()](https://github.com/laravel/framework/blob/5cc435df7a99231b1504f100c9f55e44a08bd210/src/Illuminate/Filesystem/FilesystemAdapter.php#L526) method:

```php
/**
 * Move a file to a new location.
 *
 * @param  string  $from
 * @param  string  $to
 * @return bool
 */
public function move($from, $to)
{
    try {
        $this->driver->move($from, $to);
    } catch (UnableToMoveFile $e) {
        throw_if($this->throwsExceptions(), $e);

        return false;
    }

    return true;
}
```
