# Command :joystick:

Command is a behavioral design pattern that turns a request into a stand-alone object that contains all information about the request, letting you parameterize methods with different requests, delay or queue a request's execution, and support undoable operations. [Theory](https://refactoring.guru/design-patterns/command).

## Table of Contents

1. [Queued Jobs and the Bus Dispatcher](#1-queued-jobs-and-the-bus-dispatcher)

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

The `Dispatcher` — the **Invoker** — is completely decoupled from what any job does. It only cares about *how* to dispatch: synchronously or to a queue.

The `dispatchNow()` method reveals a key detail about the pattern's structure. It supports two modes of execution. If a separate handler is mapped for the command (via `map()`), the handler acts as the **Receiver** — its `handle()` method is called with the command object as an argument. This is a classic Command-to-Receiver delegation. If no handler is mapped, the command itself acts as both the command and the receiver, executing its own `handle()` method. Laravel's code even names the parameter `$command` throughout the `Dispatcher`, reflecting this intent.

The contract that the `Dispatcher` implements makes the invoker role explicit:

> File: [src/Illuminate/Contracts/Bus/Dispatcher.php](https://github.com/laravel/framework/blob/5cc435df7a99231b1504f100c9f55e44a08bd210/src/Illuminate/Contracts/Bus/Dispatcher.php)

```php
interface Dispatcher
{
    /**
     * Dispatch a command to its appropriate handler.
     *
     * @param  mixed  $command
     * @return mixed
     */
    public function dispatch($command);

    /**
     * Dispatch a command to its appropriate handler in the current process.
     *
     * @param  mixed  $command
     * @param  mixed  $handler
     * @return mixed
     */
    public function dispatchSync($command, $handler = null);

    /**
     * Dispatch a command to its appropriate handler in the current process.
     *
     * @param  mixed  $command
     * @param  mixed  $handler
     * @return mixed
     */
    public function dispatchNow($command, $handler = null);

    /* ... CODE ... */
}
```

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
