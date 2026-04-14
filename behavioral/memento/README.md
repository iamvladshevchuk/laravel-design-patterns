# Memento :floppy_disk:

Memento is a behavioral design pattern that lets you save and restore the previous state of an object without revealing the details of its implementation. [Theory](https://refactoring.guru/design-patterns/memento).

## Table of Contents

1. [Eloquent Model Dirty Tracking](#1-eloquent-model-dirty-tracking)

## 1. Eloquent Model Dirty Tracking

> File: [src/Illuminate/Database/Eloquent/Concerns/HasAttributes.php](https://github.com/laravel/framework/blob/5cc435df7a99231b1504f100c9f55e44a08bd210/src/Illuminate/Database/Eloquent/Concerns/HasAttributes.php)

Every Eloquent model implements a Memento-like snapshot mechanism through its `$original` array. The model acts as the **Originator** — it creates and restores from a snapshot of its own attribute values. The `$original` array is the **Memento** — an internal store of the last known persisted state. In the classical GoF formulation, the **Caretaker** is a *separate* entity that holds the Memento externally without examining its contents. Laravel's implementation collapses the Caretaker role into the model itself: `$original` is a private field that never leaves the model. This simplification trades the cross-boundary encapsulation guarantee for a more practical in-object checkpoint mechanism, but the core behavioral contract — snapshot at load, compare against current state, restore on demand — maps faithfully to what the Memento pattern accomplishes.

```php
/**
 * The model attribute's original state.
 *
 * @var array
 */
protected $original = [];
```

The snapshot is created inside `newFromBuilder()`, the factory method Eloquent calls every time it hydrates a row from the database:

> File: [src/Illuminate/Database/Eloquent/Model.php](https://github.com/laravel/framework/blob/5cc435df7a99231b1504f100c9f55e44a08bd210/src/Illuminate/Database/Eloquent/Model.php#L547)

```php
public function newFromBuilder($attributes = [], $connection = null)
{
    $model = $this->newInstance([], true);

    $model->setRawAttributes((array) $attributes, true);

    $model->setConnection($connection ?: $this->getConnectionName());

    $model->fireModelEvent('retrieved', false);

    return $model;
}
```

The second argument `true` passed to `setRawAttributes()` triggers `syncOriginal()` — the method that takes the snapshot:

```php
public function setRawAttributes(array $attributes, $sync = false)
{
    $this->attributes = $attributes;

    if ($sync) {
        $this->syncOriginal();
    }

    /* ... CODE ... */

    return $this;
}

/**
 * Sync the original attributes with the current.
 *
 * @return $this
 */
public function syncOriginal()
{
    $this->original = $this->getAttributes();

    return $this;
}
```

Once the snapshot is captured, the model can be mutated freely. The `isDirty()` and `getDirty()` methods compare `$attributes` against the memento to determine what has changed:

```php
/**
 * Get the attributes that have been changed since the last sync.
 *
 * @return array
 */
public function getDirty()
{
    $dirty = [];

    foreach ($this->getAttributes() as $key => $value) {
        if (! $this->originalIsEquivalent($key)) {
            $dirty[$key] = $value;
        }
    }

    return $dirty;
}

/**
 * Determine if the model or any of the given attribute(s) have been modified.
 *
 * @param  array|string|null  $attributes
 * @return bool
 */
public function isDirty($attributes = null)
{
    return $this->hasChanges(
        $this->getDirty(), is_array($attributes) ? $attributes : func_get_args()
    );
}
```

When you need to discard in-memory changes and restore the model to its database state, `discardChanges()` replaces `$attributes` with the captured `$original` — a direct memento restoration:

```php
/**
 * Discard attribute changes and reset the attributes to their original state.
 *
 * @return $this
 */
public function discardChanges()
{
    [$this->attributes, $this->changes] = [$this->original, []];

    return $this;
}
```

A developer can call `$user->discardChanges()` at any point before saving to roll back any in-memory mutations without touching the database, relying entirely on the stored memento.

The memento is not permanent — it advances whenever the model is successfully persisted. After every `save()`, Eloquent calls `finishSave()`, which invokes `syncOriginal()` to bring the stored snapshot up to date with the just-persisted state:

> File: [src/Illuminate/Database/Eloquent/Model.php](https://github.com/laravel/framework/blob/5cc435df7a99231b1504f100c9f55e44a08bd210/src/Illuminate/Database/Eloquent/Model.php#L1090)

```php
protected function finishSave(array $options)
{
    $this->fireModelEvent('saved', false);

    if ($this->isDirty() && ($options['touch'] ?? true)) {
        $this->touchOwners();
    }

    $this->syncOriginal();
}
```

After `save()` returns, `isDirty()` returns `false` because `$original` now matches `$attributes`. Any subsequent call to `discardChanges()` would restore to the post-save state, not the state from when the model was first loaded. This mirrors a real-world undo history: once a change is committed, the snapshot advances and the previous baseline is gone.
