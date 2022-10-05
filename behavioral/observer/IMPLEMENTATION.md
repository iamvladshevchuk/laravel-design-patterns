# Observer Implementation

Whenever developers call `Illuminate\Support\Facades\Event::listen()` directly or add listeners in [App\Providers\EventServiceProvider](https://github.com/laravel/laravel/blob/5138bc36dbc884098ea68942e805b2267e7a627f/app/Providers/EventServiceProvider.php), they call [listen()](https://github.com/laravel/framework/blob/5cc435df7a99231b1504f100c9f55e44a08bd210/src/Illuminate/Events/Dispatcher.php#L70) method in `Illuminate\Events\Dispatcher`. It simply saves events and listeners in a memory.

> If you're interested in how Facades work, [check it's implementation](/structural/facade/IMPLEMENTATION.md).

```php
/**
 * Register an event listener with the dispatcher.
 *
 * @param  \Closure|string|array  $events
 * @param  \Closure|string|array|null  $listener
 * @return void
 */
public function listen($events, $listener = null)
{
    /* ... CODE ... */

    $this->listeners[$event][] = $listener;

    /* ... CODE ... */
}
```

When events are dispatched using `event(new CustomEvent())`, [dispatch()](https://github.com/laravel/framework/blob/5cc435df7a99231b1504f100c9f55e44a08bd210/src/Illuminate/Events/Dispatcher.php#L225) method in `Illuminate\Events\Dispatcher` is called that fires all the listeners subscribed to the event:

```php
/**
 * Fire an event and call the listeners.
 *
 * @param  string|object  $event
 * @param  mixed  $payload
 * @param  bool  $halt
 * @return array|null
 */
public function dispatch($event, $payload = [], $halt = false)
{
    /* ... CODE ... */

    foreach ($this->getListeners($event) as $listener) {
        $response = $listener($event, $payload);
    }

    /* ... CODE ... */
}
```