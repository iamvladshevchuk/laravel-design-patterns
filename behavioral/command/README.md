# Command :joystick:

Command is a behavioral design pattern that turns a request into a stand-alone object that contains all information about the request, letting you parameterize methods with different requests, delay or queue a request's execution, and support undoable operations. [Theory](https://refactoring.guru/design-patterns/command).

## Table of Contents

1. [Queued Jobs and the Bus Dispatcher](#1-queued-jobs-and-the-bus-dispatcher)
2. [Database Migrations](#2-database-migrations)

## 1. Queued Jobs and the Bus Dispatcher

> File: [src/Illuminate/Bus/Dispatcher.php](https://github.com/laravel/framework/blob/5cc435df7a99231b1504f100c9f55e44a08bd210/src/Illuminate/Bus/Dispatcher.php)

Laravel's job system is the clearest example of the Command pattern in the framework. A job class is a command object — it encapsulates a unit of work along with all the data it needs. The `Dispatcher` is the invoker that decides how to execute each job.

The `Dispatcher` accepts any job object through its `dispatch()` method. If the job implements `ShouldQueue`, it gets serialized and sent to a queue for later execution. Otherwise, it runs immediately:

```php
class Dispatcher implements QueueingDispatcher
{
    public function dispatch($command)
    {
        return $this->queueResolver && $this->commandShouldBeQueued($command)
                        ? $this->dispatchToQueue($command)
                        : $this->dispatchNow($command);
    }

    public function dispatchNow($command, $handler = null)
    {
        /* ... CODE ... */

        if ($handler || $handler = $this->getCommandHandler($command)) {
            $callback = function ($command) use ($handler) {
                $method = method_exists($handler, 'handle') ? 'handle' : '__invoke';

                return $handler->{$method}($command);
            };
        } else {
            $callback = function ($command) {
                $method = method_exists($command, 'handle') ? 'handle' : '__invoke';

                return $this->container->call([$command, $method]);
            };
        }

        return $this->pipeline->send($command)->through($this->pipes)->then($callback);
    }

    protected function commandShouldBeQueued($command)
    {
        return $command instanceof ShouldQueue;
    }
}
```

Notice how `dispatchNow()` supports two modes. If a separate handler is mapped for the command (via `map()`), the handler's `handle()` method is called with the command as an argument — a classic Command-to-Receiver delegation. If no handler is mapped, the command itself acts as both the command and the receiver, executing its own `handle()` method.

Jobs gain dispatching capabilities through the `Dispatchable` trait, which provides a clean static API for creating and sending command objects:

> File: [src/Illuminate/Foundation/Bus/Dispatchable.php](https://github.com/laravel/framework/blob/5cc435df7a99231b1504f100c9f55e44a08bd210/src/Illuminate/Foundation/Bus/Dispatchable.php)

```php
trait Dispatchable
{
    /**
     * Dispatch the job with the given arguments.
     *
     * @return \Illuminate\Foundation\Bus\PendingDispatch
     */
    public static function dispatch(...$arguments)
    {
        return new PendingDispatch(new static(...$arguments));
    }

    /**
     * Dispatch a command to its appropriate handler in the current process.
     *
     * Queueable jobs will be dispatched to the "sync" queue.
     *
     * @return mixed
     */
    public static function dispatchSync(...$arguments)
    {
        return app(Dispatcher::class)->dispatchSync(new static(...$arguments));
    }
}
```

When you call `ProcessPodcast::dispatch($podcast)`, the trait creates a new `ProcessPodcast` instance (the command object) with all its data, wraps it in a `PendingDispatch`, and eventually sends it to the `Dispatcher`. The command object carries everything needed to do the work later — the podcast data, queue preferences, delay settings — which is exactly what the Command pattern is about: turning a request into an object that can be parameterized, queued, and executed independently.

This design also enables powerful features like job chaining and batching, where multiple command objects are grouped and executed in sequence or in parallel — all possible because each job is a self-contained, serializable command.

## 2. Database Migrations

> File: [src/Illuminate/Database/Migrations/Migrator.php](https://github.com/laravel/framework/blob/5cc435df7a99231b1504f100c9f55e44a08bd210/src/Illuminate/Database/Migrations/Migrator.php)

Database migrations are a textbook Command pattern with a feature that the job system lacks: **undo**. Each migration is a concrete command with `up()` to execute and `down()` to reverse. The `Migrator` is the invoker, and the database connection (via the Schema Builder) is the receiver.

The abstract `Migration` class defines the contract. It's intentionally minimal — every concrete migration provides its own `up()` and `down()` methods:

> File: [src/Illuminate/Database/Migrations/Migration.php](https://github.com/laravel/framework/blob/5cc435df7a99231b1504f100c9f55e44a08bd210/src/Illuminate/Database/Migrations/Migration.php)

```php
abstract class Migration
{
    /**
     * The name of the database connection to use.
     *
     * @var string|null
     */
    protected $connection;

    /**
     * Enables, if supported, wrapping the migration within a transaction.
     *
     * @var bool
     */
    public $withinTransaction = true;

    /**
     * Get the migration connection name.
     *
     * @return string|null
     */
    public function getConnection()
    {
        return $this->connection;
    }
}
```

The `Migrator` is the invoker. It resolves migration objects and calls `up()` or `down()` on them without knowing what schema changes they perform. The `runUp()` and `runDown()` methods mirror each other — execute and undo:

```php
/**
 * Run "up" a migration instance.
 *
 * @param  string  $file
 * @param  int  $batch
 * @param  bool  $pretend
 * @return void
 */
protected function runUp($file, $batch, $pretend)
{
    $migration = $this->resolvePath($file);

    $name = $this->getMigrationName($file);

    if ($pretend) {
        return $this->pretendToRun($migration, 'up');
    }

    $this->write(Task::class, $name, fn () => $this->runMigration($migration, 'up'));

    $this->repository->log($name, $batch);
}

/**
 * Run "down" a migration instance.
 *
 * @param  string  $file
 * @param  object  $migration
 * @param  bool  $pretend
 * @return void
 */
protected function runDown($file, $migration, $pretend)
{
    $instance = $this->resolvePath($file);

    $name = $this->getMigrationName($file);

    if ($pretend) {
        return $this->pretendToRun($instance, 'down');
    }

    $this->write(Task::class, $name, fn () => $this->runMigration($instance, 'down'));

    $this->repository->delete($migration);
}
```

Both methods delegate to `runMigration()`, which calls the migration's method generically — it doesn't care whether it's running `up` or `down`. If the database supports it and the migration opts in, the entire operation is wrapped in a transaction:

```php
/**
 * Run a migration inside a transaction if the database supports it.
 *
 * @param  object  $migration
 * @param  string  $method
 * @return void
 */
protected function runMigration($migration, $method)
{
    $connection = $this->resolveConnection(
        $migration->getConnection()
    );

    $callback = function () use ($connection, $migration, $method) {
        if (method_exists($migration, $method)) {
            $this->fireMigrationEvent(new MigrationStarted($migration, $method));

            $this->runMethod($connection, $migration, $method);

            $this->fireMigrationEvent(new MigrationEnded($migration, $method));
        }
    };

    $this->getSchemaGrammar($connection)->supportsSchemaTransactions()
        && $migration->withinTransaction
            ? $connection->transaction($callback)
            : $callback();
}
```

This maps cleanly to every aspect of the Command pattern. Each migration encapsulates a unique schema change (the request). The `Migrator` runs them without knowing what any migration does (invoker decoupled from receiver). Migrations are collected, sorted by timestamp, and executed in batches (queuing and ordering). The migration repository tracks which commands have been executed (history). And `down()` provides built-in undo, which is one of the pattern's most important motivations — when you run `php artisan migrate:rollback`, the `Migrator` iterates the last batch in reverse order, calling `down()` on each.
