# Chain of Responsibility :chains:

Chain of Responsibility is a behavioral design pattern that lets you pass requests along a chain of handlers. Upon receiving a request, each handler decides either to process the request or to pass it to the next handler in the chain. [Theory](https://refactoring.guru/design-patterns/chain-of-responsibility).

> Before you continue, take a look [how Laravel implemented Chain of Responsibility](./IMPLEMENTATION.md) in the framework.

## Table of Contents

1. [Middleware](#1-middleware)

## 1. Middleware

> File: [app/Http/Kernel.php](https://github.com/laravel/laravel/blob/5138bc36dbc884098ea68942e805b2267e7a627f/app/Http/Kernel.php)

Whenever a user sends a request, Kernel handles it. There's [a list of middleware](https://github.com/laravel/laravel/blob/5138bc36dbc884098ea68942e805b2267e7a627f/app/Http/Kernel.php#L9) that modify or check every request that goes through an app. That's a list of handlers.

```php
class Kernel extends HttpKernel
{
    /**
     * The application's global HTTP middleware stack.
     *
     * These middleware are run during every request to your application.
     *
     * @var array<int, class-string|string>
     */
    protected $middleware = [
        // \App\Http\Middleware\TrustHosts::class,
        \App\Http\Middleware\TrustProxies::class,
        \Illuminate\Http\Middleware\HandleCors::class,
        \App\Http\Middleware\PreventRequestsDuringMaintenance::class,
        \Illuminate\Foundation\Http\Middleware\ValidatePostSize::class,
        \App\Http\Middleware\TrimStrings::class,
        \Illuminate\Foundation\Http\Middleware\ConvertEmptyStringsToNull::class,
    ];

    /* ... CODE ... */
}
```

Middleware classes typically look like [app\Http\Middleware\RedirectIfAuthenticated](https://github.com/laravel/laravel/blob/5138bc36dbc884098ea68942e805b2267e7a627f/app/Http/Middleware/RedirectIfAuthenticated.php) class that's responsible for redirecting authenticated users to a home URL:

```php
class RedirectIfAuthenticated
{
    /**
     * Handle an incoming request.
     *
     * @param  \Illuminate\Http\Request  $request
     * @param  \Closure(\Illuminate\Http\Request): (\Illuminate\Http\Response|\Illuminate\Http\RedirectResponse)  $next
     * @param  string|null  ...$guards
     * @return \Illuminate\Http\Response|\Illuminate\Http\RedirectResponse
     */
    public function handle(Request $request, Closure $next, ...$guards)
    {
        $guards = empty($guards) ? [null] : $guards;

        foreach ($guards as $guard) {
            if (Auth::guard($guard)->check()) {
                return redirect(RouteServiceProvider::HOME);
            }
        }

        return $next($request);
    }
}
```
