# Polly.Contrib.WaitAndRetry

Polly.Contrib.WaitAndRetry contains several helper methods for defining backoff strategies when using [Polly](http://www.thepollyproject.org/)'s wait-and-retry fault handling ability.

[![NuGet version](https://badge.fury.io/nu/Polly.Contrib.WaitAndRetry.svg)](https://badge.fury.io/nu/Polly.Contrib.WaitAndRetry) [![Build status](https://ci.appveyor.com/api/projects/status/5v3bpgjkw4snv3no?svg=true)](https://ci.appveyor.com/project/Polly-Contrib/polly-contrib-waitandretry) [![Slack Status](http://www.pollytalk.org/badge.svg)](http://www.pollytalk.org)

# Installing via NuGet

    Install-Package Polly.Contrib.WaitAndRetry

# Usage

One common approach when calling occasionally unreliable services is to wrap those calls in a retry policy. For example, if we're calling a remote service, we might choose to retry several times with a slight pause in between to account for infrastructure issues.

We can define a policy to do this in Polly.

## Wait and Retry with Constant Back-off

The following defines a policy that will retry five times and pause 200ms between each call.

    Policy
        .Handle<HttpRequestException>()
        .WaitAndRetryAsync(retryCount: 5, retryNumber => TimeSpan.FromMilliseconds(200));

We can simplify this by using the `ConstantBackoff` helper in Polly.Contrib.WaitAndRetry

    var delay = Backoff.ConstantBackoff(TimeSpan.FromMilliseconds(200), retryCount: 5);

    Policy
        .Handle<HttpRequestException>()
        .WaitAndRetryAsync(delay);

Note that `retryCount` must be greater than or equal to zero.

Additionally, when using the `ConstantBackoff` helper, or any other WaitAndRetry helper, we can signal that the first failure should retry immediately rather than waiting the indicated time. To do this, ensure the `fastFirst` parameter is `true`.

    var delay = Backoff.ConstantBackoff(TimeSpan.FromMilliseconds(200), retryCount: 5, fastFirst: true);

This will still retry five times but the first retry will happen immediately.

## Wait and Retry with Linear Back-off

It can be desirable to wait increasingly long times between retries. For example, in cases where a remote service is being impacted by a high workload. In this scenario, we want to give the service some time to stabilize before trying again.

The first tool at our disposal is the `LinearBackoff` helper.

    var delay = Backoff.LinearBackoff(TimeSpan.FromMilliseconds(100), retryCount: 5);

    Policy
        .Handle<HttpRequestException>()
        .WaitAndRetryAsync(delay);

This will create a linearly increasing retry delay of 100, 200, 300, 400, 500ms. 

The default linear factor is 1.0. However, we can provide our own.

    var delay = Backoff.LinearBackoff(TimeSpan.FromMilliseconds(100), retryCount: 5, factor: 2);

This will create an increasing retry delay of 100, 300, 500, 700, 900ms.

Note, the linear factor must be greater than or equal to zero. A factor of zero will return equivalent retry delays to the `ConstantBackoff` helper. 

## Wait and Retry with Exponential Back-off

We can also specify an exponential back-off where the delay duration is `initialDelay x 2^iteration`. Because of the exponential nature, this is best used with a low starting delay or in out-of-band communication, such as a service worker polling for information from a remote endpoint. Due to the potential for rapidly increasing times, care should be taken if an exponential retry is used in the code path for servicing a user request.

    var delay = Backoff.ExponentialBackoff(TimeSpan.FromMilliseconds(100), retryCount: 5);

    Policy
        .Handle<HttpRequestException>()
        .WaitAndRetryAsync(delay);

This will create an exponentially increasing retry delay of 100, 200, 400, 800, 1600ms.

The default exponential growth factor is 2.0. However, can can provide our own.

    var delay = Backoff.ExponentialBackoff(TimeSpan.FromMilliseconds(100), retryCount: 5, factor: 4);

The upper for this retry with a growth factor of four is 25,600ms. Care and a calculator should be used when changing the factor.

Note, the growth factor must be greater than or equal to one. A factor of one will return equivalent retry delays to the `ConstantBackoff` helper. 

## Wait and Retry with Jittered Back-off

In a high-throughput scenario, that is where you have many requests firing at once, where the remote service is non-responsive due to load, the above retry options may not help. This is because the retries are highly correlated. If there are 100 concurrent requests, and all 100 requests enter a wait-and-retry for 10ms, then all 100 requests will hit the service again in 10ms; potentially overwhelming the service again.

One way to address this is to add some randomness to the wait delay. This will cause each request to vary slightly on retry which decorrelates the requests from each other.

The Polly Wiki has a very good write-up on how to randomness, or [jitter](https://github.com/App-vNext/Polly/wiki/Retry-with-jitter), to the retry policy.

However, there are still [drawbacks](https://github.com/App-vNext/Polly/issues/530) with the recommended jitter approach. Worry not! A more robust decorrelated jitter is available a helper method in Polly.Contrib.WaitAndRetry.

    var delay = Backoff.DecorrelatedJitterBackoff(minDelay: TimeSpan.FromMilliseconds(10), maxDelay: TimeSpan.FromMilliseconds(100), retryCount: 5);

    Policy
        .Handle<HttpRequestException>()
        .WaitAndRetryAsync(delay);

This will set up a policy that will retry five times. Each retry will delay for a random amount of time between the minimum of 10ms and the maximum of 100ms. 

Note, unlike the linear and exponent wait-and-retry helpers, each subsequent retry delay in the `DecorrelatedJitterBackoff` helper has no relationship to the previous. That is, the fourth retry delay may be significantly shorter (or longer), than the first or third delay.

Internally, the `DecorrelatedJitterBackoff` uses a shared `Random` to better ensure a random distribution across all calls. You may, optionally, provide your own seed value.

    var delay = Backoff.DecorrelatedJitterBackoff(
        minDelay: TimeSpan.FromMilliseconds(10), 
        maxDelay: TimeSpan.FromMilliseconds(100), 
        retryCount: 5, 
        seed: 100);

The shared `Random` will still be used internally in this case, but it will be seeded with your value versus the default used by .NET in a call to `new Random()`

## Retry first failure fast

All helper methods in Polly.Contrib.WaitAndRetry include an option to retry the first failure immediately. You can trigger this by passing in `fastFirst: true` to any of the helper methods.

    var delay = Backoff.ConstantBackoff(TimeSpan.FromMilliseconds(200), retryCount: 5, fastFirst: true);

Note, the first retry will happen immediately and it will count against your retry count. That is, this will still retry five times but the first retry will happen immediately.

## Further Resources

Be sure to read through the material linked from the [Polly readme](https://github.com/App-vNext/Polly/).