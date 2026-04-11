# Command :joystick:

Command is a behavioral design pattern that turns a request into a stand-alone object that contains all information about the request, letting you parameterize methods with different requests, delay or queue a request's execution, and support undoable operations. [Theory](https://refactoring.guru/design-patterns/command).

## Table of Contents

1. [Artisan Console Commands](#1-artisan-console-commands)
2. [Queued Jobs and the Bus Dispatcher](#2-queued-jobs-and-the-bus-dispatcher)

## 1. Artisan Console Commands

> File: [src/Illuminate/Console/Command.php](https://github.com/laravel/framework/blob/5cc435df7a99231b1504f100c9f55e44a08bd210/src/Illuminate/Console/Command.php)

Laravel's Artisan CLI is a textbook implementation of the Command pattern. Every console command is a self-contained object that encapsulates all the information needed to perform an action: its name, arguments, options, and the logic itself.

The base `Command` class extends Symfony's `Command` and defines the structure every concrete command must follow. The key method is `execute()`, which acts as the entry point. It delegates to the `handle()` method that each concrete command implements:

```php
class Command extends SymfonyCommand
{
    use Concerns\CallsCommands,
        Concerns\HasParameters,
        Concerns\InteractsWithIO,
        Concerns\InteractsWithSignals,
        Macroable;

    /**
     * The name and signature of the console command.
     *
     * @var string
     */
    protected $signature;

    /**
     * The console command description.
     *
     * @var string
     */
    protected $description;

    /* ... CODE ... */

    /**
     * Execute the console command.
     *
     * @param  \Symfony\Component\Console\Input\InputInterface  $input
     * @param  \Symfony\Component\Console\Output\OutputInterface  $output
     * @return int
     */
    protected function execute(InputInterface $input, OutputInterface $output)
    {
        $method = method_exists($this, 'handle') ? 'handle' : '__invoke';

        return (int) $this->laravel->call([$this, $method]);
    }
}
```

A concrete command — like any `php artisan make:model` or a custom command you write — extends this base class, declares its `$signature` and `$description`, and implements the `handle()` method with the actual business logic. The command object holds everything: what to do, what arguments it needs, and how to describe itself.

The **Invoker** in this setup is the Console Kernel. It bootstraps the application, collects all registered command objects, and runs the one that matches the user's input:

> File: [src/Illuminate/Foundation/Console/Kernel.php](https://github.com/laravel/framework/blob/5cc435df7a99231b1504f100c9f55e44a08bd210/src/Illuminate/Foundation/Console/Kernel.php)

```php
class Kernel implements KernelContract
{
    /**
     * Run the console application.
     *
     * @param  \Symfony\Component\Console\Input\InputInterface  $input
     * @param  \Symfony\Component\Console\Output\OutputInterface|null  $output
     * @return int
     */
    public function handle($input, $output = null)
    {
        $this->commandStartedAt = Carbon::now();

        try {
            $this->bootstrap();

            return $this->getArtisan()->run($input, $output);
        } catch (Throwable $e) {
            $this->reportException($e);

            $this->renderException($output, $e);

            return 1;
        }
    }

    /**
     * Run an Artisan console command by name.
     *
     * @param  string  $command
     * @param  array  $parameters
     * @param  \Symfony\Component\Console\Output\OutputInterface|null  $outputBuffer
     * @return int
     */
    public function call($command, array $parameters = [], $outputBuffer = null)
    {
        $this->bootstrap();

        return $this->getArtisan()->call($command, $parameters, $outputBuffer);
    }
}
```

The Kernel doesn't know what any specific command does — it just finds the right command object and tells it to run. This is the core benefit of the Command pattern: the invoker is completely decoupled from the receiver's business logic.

## 2. Queued Jobs and the Bus Dispatcher

> File: [src/Illuminate/Bus/Dispatcher.php](https://github.com/laravel/framework/blob/5cc435df7a99231b1504f100c9f55e44a08bd210/src/Illuminate/Bus/Dispatcher.php)

Laravel's job system is another strong example of the Command pattern. A job class is a command object — it encapsulates a unit of work along with all the data it needs. The `Dispatcher` is the invoker that decides how to execute each job.

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
