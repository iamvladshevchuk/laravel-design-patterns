# Bridge :bridge_at_night:

Bridge is a structural design pattern that lets you split a large class or a set of closely related classes into two separate hierarchies—abstraction and implementation—which can be developed independently of each other. [Theory](https://refactoring.guru/design-patterns/bridge).

## Table of Contents

1. [UploadedFile -> Filesystem](#1-uploadedfile---filesystem)

## 1. UploadedFile -> Filesystem

> Abstraction: [src/Illuminate/Http/UploadedFile.php](https://github.com/laravel/framework/blob/5cc435df7a99231b1504f100c9f55e44a08bd210/src/Illuminate/Http/UploadedFile.php)

> Implementation: [src/Illuminate/Filesystem/FilesystemAdapter.php](https://github.com/laravel/framework/blob/5cc435df7a99231b1504f100c9f55e44a08bd210/src/Illuminate/Filesystem/FilesystemAdapter.php)

When a developer uses [storeAs()](https://github.com/laravel/framework/blob/5cc435df7a99231b1504f100c9f55e44a08bd210/src/Illuminate/Http/UploadedFile.php#L72) function in `Illuminate/Http/UploadedFile` class, Laravel gives a bridge to a developer that allows to use `Illuminate/Filesystem/FilesystemAdapter` without even knowing it. It provides `high-level control logic` over the filesystem.

```php
/**
 * Store the uploaded file on a filesystem disk.
 *
 * @param  string  $path
 * @param  string  $name
 * @param  array|string  $options
 * @return string|false
 */
public function storeAs($path, $name, $options = [])
{
    $options = $this->parseOptions($options);

    $disk = Arr::pull($options, 'disk');

    return Container::getInstance()->make(FilesystemFactory::class)->disk($disk)->putFileAs(
        $path, $this, $name, $options
    );
}
```
