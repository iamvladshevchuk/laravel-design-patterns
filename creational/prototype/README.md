# Prototype :robot:

Prototype is a creational design pattern that lets you copy existing objects without making your code dependent on their classes. [Theory](https://refactoring.guru/design-patterns/prototype).

## Implementation
PHP has built-in cloning support. You can `clone` an object without defining any special methods as long as it has fields of primitive types. Fields containing objects retain their references in a cloned object. Therefore, in some cases, you might want to clone those referenced objects as well. You can do this in a special `__clone()` method.

## Example

[Illuminate\Database\Eloquent\Builder::clone()](https://github.com/laravel/framework/blob/1d8e86193fd1740606a836f25043e84fe78c562d/src/Illuminate/Database/Eloquent/Builder.php#L1922) made thanks to keyword `clone`:

```php
/**
 * Clone the Eloquent query builder.
 *
 * @return static
 */
public function clone()
{
    return clone $this;
}
```

It also utilizes [a magic method](https://github.com/laravel/framework/blob/1d8e86193fd1740606a836f25043e84fe78c562d/src/Illuminate/Database/Eloquent/Builder.php#L1932) to clone an underlying query builder:

```php
/**
 * Force a clone of the underlying query builder when cloning.
 *
 * @return void
 */
public function __clone()
{
    $this->query = clone $this->query;
}
```