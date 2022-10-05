## Chain of Responsibility Implementation

> File: [src/Illuminate/Pipeline/Pipeline.php](https://github.com/laravel/framework/blob/5cc435df7a99231b1504f100c9f55e44a08bd210/src/Illuminate/Pipeline/Pipeline.php)

How exactly does Laravel implement Chain of Responsibility pattern? Whenever [Kernel](https://github.com/laravel/framework/blob/5cc435df7a99231b1504f100c9f55e44a08bd210/src/Illuminate/Foundation/Http/Kernel.php) gets a request to handle, it forwards a request to [sendRequestThroughRouter()](https://github.com/laravel/framework/blob/5cc435df7a99231b1504f100c9f55e44a08bd210/src/Illuminate/Foundation/Http/Kernel.php#L148) function:

```php
/**
 * Send the given request through the middleware / router.
 *
 * @param  \Illuminate\Http\Request  $request
 * @return \Illuminate\Http\Response
 */
protected function sendRequestThroughRouter($request)
{
    /* ... CODE ... */

    return (new Pipeline($this->app))
                ->send($request)
                ->through($this->app->shouldSkipMiddleware() ? [] : $this->middleware)
                ->then($this->dispatchToRouter());
}
```

The main class responsible for this pattern is [Illuminate\Pipeline\Pipeline](https://github.com/laravel/framework/blob/5cc435df7a99231b1504f100c9f55e44a08bd210/src/Illuminate/Pipeline/Pipeline.php). That's [a builder](/creational/builder/) to create a complex pipeline. [send()](https://github.com/laravel/framework/blob/5cc435df7a99231b1504f100c9f55e44a08bd210/src/Illuminate/Pipeline/Pipeline.php#L52) method defines the object a developer wants to change in the pipeline. [through()](https://github.com/laravel/framework/blob/5cc435df7a99231b1504f100c9f55e44a08bd210/src/Illuminate/Pipeline/Pipeline.php#L65) defines an array of handlers the request will go through. And, finally, it runs when [then()](https://github.com/laravel/framework/blob/5cc435df7a99231b1504f100c9f55e44a08bd210/src/Illuminate/Pipeline/Pipeline.php#L104) function is called. It also defines what's the last handler will do after every other handler did their job. 

If you wish to go deeper, here's the function that actually creates a pipeline:

```php
/**
 * Run the pipeline with a final destination callback.
 *
 * @param  \Closure  $destination
 * @return mixed
 */
public function then(Closure $destination)
{
    $pipeline = array_reduce(
        array_reverse($this->pipes()), $this->carry(), $this->prepareDestination($destination)
    );

    return $pipeline($this->passable);
}
```

To put it simply, the method above is "merging" every handler a developer set into a single function to handle `$this->passable` property. In our case it's a `$request` variable. If you want to go even deeper than that, check [a source file](https://github.com/laravel/framework/blob/5cc435df7a99231b1504f100c9f55e44a08bd210/src/Illuminate/Pipeline/Pipeline.php).

**Summary:** Chain of Responsibility pattern in Laravel is implemented using a `Pipeline` class.
