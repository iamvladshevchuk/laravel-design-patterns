# Observer :eyeglasses:

Observer is a behavioral design pattern that lets you define a subscription mechanism to notify multiple objects about any events that happen to the object they’re observing. [Theory](https://refactoring.guru/design-patterns/observer).

## Table of Contents
1. [Registered -> SendEmailVerificationNotification](#1-registered---sendemailverificationnotification)

## 1. Registered -> SendEmailVerificationNotification

> File: [app/Providers/EventServiceProvider.php](https://github.com/laravel/laravel/blob/5138bc36dbc884098ea68942e805b2267e7a627f/app/Providers/EventServiceProvider.php)

A fresh installation of Laravel already has a listener in [App\Providers\EventServiceProvider](https://github.com/laravel/laravel/blob/5138bc36dbc884098ea68942e805b2267e7a627f/app/Providers/EventServiceProvider.php). It listens for [Registered](https://github.com/laravel/framework/blob/5cc435df7a99231b1504f100c9f55e44a08bd210/src/Illuminate/Auth/Events/Registered.php) event and has [SendEmailVerificationNotification]((https://github.com/laravel/framework/blob/5cc435df7a99231b1504f100c9f55e44a08bd210/src/Illuminate/Auth/Listeners/SendEmailVerificationNotification.php)) listener:

```php
class EventServiceProvider extends ServiceProvider
{
    /**
     * The event to listener mappings for the application.
     *
     * @var array<class-string, array<int, class-string>>
     */
    protected $listen = [
        Registered::class => [
            SendEmailVerificationNotification::class,
        ],
    ];

    /* ... CODE ... */
}
```

The structure of [the event](https://github.com/laravel/framework/blob/5cc435df7a99231b1504f100c9f55e44a08bd210/src/Illuminate/Auth/Events/Registered.php) is pretty simple:

```php
class Registered
{
    use SerializesModels;

    /**
     * The authenticated user.
     *
     * @var \Illuminate\Contracts\Auth\Authenticatable
     */
    public $user;

    /**
     * Create a new event instance.
     *
     * @param  \Illuminate\Contracts\Auth\Authenticatable  $user
     * @return void
     */
    public function __construct($user)
    {
        $this->user = $user;
    }
}
```

Where does [Registered](https://github.com/laravel/framework/blob/5cc435df7a99231b1504f100c9f55e44a08bd210/src/Illuminate/Auth/Events/Registered.php) event fire? It depends. If you want to build an authentication on your own, that's your job to fire this event. If you use [Fortify](https://github.com/laravel/fortify), this event is dispatched in [RegisteredUserController](https://github.com/laravel/fortify/blob/07e766a085b749c383333f47f4d8bbf17bac8ac6/src/Http/Controllers/RegisteredUserController.php#L44):

```php
/**
 * Create a new registered user.
 *
 * @param  \Illuminate\Http\Request  $request
 * @param  \Laravel\Fortify\Contracts\CreatesNewUsers  $creator
 * @return \Laravel\Fortify\Contracts\RegisterResponse
 */
public function store(Request $request, CreatesNewUsers $creator): RegisterResponse
{
    event(new Registered($user = $creator->create($request->all())));

    $this->guard->login($user);

    return app(RegisterResponse::class);
}
```

Thus, triggering [the listener](https://github.com/laravel/framework/blob/5cc435df7a99231b1504f100c9f55e44a08bd210/src/Illuminate/Auth/Listeners/SendEmailVerificationNotification.php):

```php
class SendEmailVerificationNotification
{
    /**
     * Handle the event.
     *
     * @param  \Illuminate\Auth\Events\Registered  $event
     * @return void
     */
    public function handle(Registered $event)
    {
        if ($event->user instanceof MustVerifyEmail && ! $event->user->hasVerifiedEmail()) {
            $event->user->sendEmailVerificationNotification();
        }
    }
}
```
