# Template Method :clipboard:

Template Method is a behavioral design pattern that defines the skeleton of an algorithm in the superclass but lets subclasses override specific steps of the algorithm without changing its structure. [Theory](https://refactoring.guru/design-patterns/template-method).

## Table of Contents

1. [Artisan Console Commands](#1-artisan-console-commands)
2. [Mailable](#2-mailable)

## 1. Artisan Console Commands

> File: [src/Illuminate/Console/Command.php](https://github.com/laravel/framework/blob/5cc435df7a99231b1504f100c9f55e44a08bd210/src/Illuminate/Console/Command.php)

Laravel's `Command` class extends Symfony's console command and uses Template Method to define the execution flow for all Artisan commands. The `execute()` method is the template method — it defines the algorithm skeleton for running a command and delegates the actual work to the `handle()` step:

```php
class Command extends SymfonyCommand
{
    use Concerns\CallsCommands,
        Concerns\HasParameters,
        Concerns\InteractsWithIO,
        Concerns\InteractsWithSignals,
        Macroable;

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

Every Artisan command you create overrides only the `handle()` method. The `execute()` method is never touched — it resolves and invokes `handle()` through the service container, ensuring automatic dependency injection for all parameters. The developer defines *what* the command does in `handle()`; the framework controls *how* and *when* it runs.

## 2. Mailable

> File: [src/Illuminate/Mail/Mailable.php](https://github.com/laravel/framework/blob/5cc435df7a99231b1504f100c9f55e44a08bd210/src/Illuminate/Mail/Mailable.php)

Laravel's `Mailable` class provides a richer example of Template Method. The `send()` method is the template method: it orchestrates the entire email-sending pipeline — applying locale, invoking `build()` on the subclass, constructing the view, and running all message-building steps in sequence:

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

When you define a mailable, you implement only the `build()` method. Inside it, you call fluent methods like `->to()`, `->subject()`, `->view()`, and `->attach()` to configure the email. The `send()` method picks up whatever `build()` configured and executes the full delivery sequence. The entire pipeline — locale handling, view construction, recipient routing, tag metadata, and attachments — is owned and sequenced by the base class.
