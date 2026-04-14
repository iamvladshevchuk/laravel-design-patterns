# Template Method :clipboard:

Template Method is a behavioral design pattern that defines the skeleton of an algorithm in the superclass but lets subclasses override specific steps of the algorithm without changing its structure. [Theory](https://refactoring.guru/design-patterns/template-method).

## Table of Contents

1. [Generator Commands](#1-generator-commands)
2. [Mailable](#2-mailable)

## 1. Generator Commands

> File: [src/Illuminate/Console/GeneratorCommand.php](https://github.com/laravel/framework/blob/5cc435df7a99231b1504f100c9f55e44a08bd210/src/Illuminate/Console/GeneratorCommand.php)

`GeneratorCommand` is the clearest Template Method in Laravel. It is the abstract base class for all `make:*` Artisan commands (`make:controller`, `make:model`, `make:middleware`, etc.). The class is declared `abstract` and requires every subclass to supply one thing: the stub file path.

```php
abstract class GeneratorCommand extends Command
{
    /* ... CODE ... */

    /**
     * Get the stub file for the generator.
     *
     * @return string
     */
    abstract protected function getStub();

    /* ... CODE ... */
}
```

The `handle()` method is the **template method** — the invariant algorithm that every `make:*` command follows, regardless of what class type it generates:

```php
public function handle()
{
    if ($this->isReservedName($this->getNameInput())) {
        $this->components->error('The name "'.$this->getNameInput().'" is reserved by PHP.');

        return false;
    }

    $name = $this->qualifyClass($this->getNameInput());

    $path = $this->getPath($name);

    if ((! $this->hasOption('force') ||
         ! $this->option('force')) &&
         $this->alreadyExists($this->getNameInput())) {
        $this->components->error($this->type.' already exists.');

        return false;
    }

    $this->makeDirectory($path);

    $this->files->put($path, $this->sortImports($this->buildClass($name)));

    $info = $this->type;

    if (in_array(CreatesMatchingTest::class, class_uses_recursive($this))) {
        if ($this->handleTestCreation($path)) {
            $info .= ' and test';
        }
    }

    $this->components->info(sprintf('%s [%s] created successfully.', $info, $path));
}
```

The sequence is fixed: validate the name → `qualifyClass()` → `getPath()` → check for existing file → `buildClass()` → write file → report success. None of these steps can be reordered or skipped by a subclass.

The abstract step is invoked inside `buildClass()`, which loads the stub file and performs namespace and class-name replacements:

```php
protected function buildClass($name)
{
    $stub = $this->files->get($this->getStub());

    return $this->replaceNamespace($stub, $name)->replaceClass($stub, $name);
}
```

`$this->getStub()` is the single point of variability. Every **ConcreteClass** — `ControllerMakeCommand`, `ModelMakeCommand`, `MiddlewareMakeCommand`, and others — extends `GeneratorCommand` and overrides only `getStub()` to return the path to its own stub file:

> File: [src/Illuminate/Routing/Console/ControllerMakeCommand.php](https://github.com/laravel/framework/blob/5cc435df7a99231b1504f100c9f55e44a08bd210/src/Illuminate/Routing/Console/ControllerMakeCommand.php)

```php
class ControllerMakeCommand extends GeneratorCommand
{
    /* ... CODE ... */

    /**
     * Get the stub file for the generator.
     *
     * @return string
     */
    protected function getStub()
    {
        $stub = null;

        if ($type = $this->option('type')) {
            $stub = "/stubs/controller.{$type}.stub";
        } elseif ($this->option('parent')) {
            $stub = '/stubs/controller.nested.stub';
        } elseif ($this->option('model')) {
            $stub = '/stubs/controller.model.stub';

        /* ... CODE ... */

        $stub ??= '/stubs/controller.plain.stub';

        return $this->resolveStubPath($stub);
    }
}
```

`GeneratorCommand` acts as the **AbstractClass**: it defines `handle()` as the template method and `getStub()` as the abstract step. `ControllerMakeCommand` (and every other `make:*` command) acts as the **ConcreteClass**: it inherits the full generation algorithm from `handle()` and overrides only `getStub()` to specify which template to use. The entire file-generation pipeline is owned by the base class; subclasses only answer the question "which stub?"

## 2. Mailable

> File: [src/Illuminate/Mail/Mailable.php](https://github.com/laravel/framework/blob/5cc435df7a99231b1504f100c9f55e44a08bd210/src/Illuminate/Mail/Mailable.php)

Laravel's `Mailable` class provides a second example where Template Method governs the email-sending pipeline. The `send()` method is the template method — it defines an invariant sequence of steps: apply locale, invoke the subclass hook, construct the view, then chain all message-building operations:

```php
class Mailable implements MailableContract, Renderable
{
    /* ... CODE ... */

    /**
     * Send the message using the given mailer.
     *
     * @param  \Illuminate\Contracts\Mail\Factory|\Illuminate\Contracts\Mail\Mailer  $mailer
     * @return \Illuminate\Mail\SentMessage|null
     */
    public function send($mailer)
    {
        return $this->withLocale($this->locale, function () use ($mailer) {
            Container::getInstance()->call([$this, 'build']);

            $mailer = $mailer instanceof MailFactory
                            ? $mailer->mailer($this->mailer)
                            : $mailer;

            return $mailer->send($this->buildView(), $this->buildViewData(), function ($message) {
                $this->buildFrom($message)
                     ->buildRecipients($message)
                     ->buildSubject($message)
                     ->buildTags($message)
                     ->buildMetadata($message)
                     ->runCallbacks($message)
                     ->buildAttachments($message);
            });
        });
    }
}
```

The `build()` call is the hook step — the single point of subclass customization. Unlike `GeneratorCommand::getStub()`, `build()` is not declared `abstract`; the base class invokes it via `Container::getInstance()->call([$this, 'build'])` and relies on the convention that every concrete mailable defines it. The helper methods `buildFrom()`, `buildRecipients()`, `buildSubject()`, and the rest are concrete steps that run unconditionally in the base class — subclasses never override them.

When you define a mailable, you implement `build()` and call fluent methods like `->to()`, `->subject()`, `->view()`, and `->attach()` to configure the message. The `send()` template method picks up that configuration and drives the full delivery sequence. The entire pipeline — locale handling, view construction, recipient routing, tag metadata, and attachments — is owned and sequenced by the base class. A developer provides only the content of `build()`; the framework controls the rest.
