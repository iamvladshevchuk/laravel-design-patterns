# Memento :floppy_disk:

Memento is a behavioral design pattern that lets you save and restore the previous state of an object without revealing the details of its implementation. [Theory](https://refactoring.guru/design-patterns/memento).

## Table of Contents

1. [Eloquent Model Dirty Tracking](#1-eloquent-model-dirty-tracking)
2. [Post-Save Snapshot Rollforward](#2-post-save-snapshot-rollforward)

## 1. Eloquent Model Dirty Tracking

> File: [src/Illuminate/Database/Eloquent/Concerns/HasAttributes.php](https://github.com/laravel/framework/blob/5cc435df7a99231b1504f100c9f55e44a08bd210/src/Illuminate/Database/Eloquent/Concerns/HasAttributes.php)

Every Eloquent model acts as an **Originator** in the Memento pattern. When a model is loaded from the database, it captures a snapshot of its attribute values in the protected `$original` array — this array is the **Memento**. The model itself also acts as the **Caretaker**, holding onto that snapshot for the rest of the request lifecycle.

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

The second argument `true` passed to `setRawAttributes()` triggers `syncOriginal()` — the method that actually takes the snapshot:

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

## 2. Post-Save Snapshot Rollforward

> File: [src/Illuminate/Database/Eloquent/Model.php](https://github.com/laravel/framework/blob/5cc435df7a99231b1504f100c9f55e44a08bd210/src/Illuminate/Database/Eloquent/Model.php#L1091)

The Memento is not permanent — it is updated whenever the model is successfully persisted. After every `save()`, Eloquent calls `finishSave()`, which invokes `syncOriginal()` to bring the stored snapshot up to date with the just-persisted state:

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

This rolling-forward behavior is important: after `save()` returns, `isDirty()` returns `false` because `$original` now matches `$attributes`. Any subsequent call to `discardChanges()` would restore to the post-save state, not the state from when the model was first loaded.

This mirrors how a real-world undo history works — once you commit a change, the previous snapshot is discarded and the new state becomes the baseline.
