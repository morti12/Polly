# Circuit breaker resilience strategy

## About

- **Option(s)**:
  - [`CircuitBreakerStrategyOptions`](xref:Polly.CircuitBreaker.CircuitBreakerStrategyOptions)
  - [`CircuitBreakerStrategyOptions<T>`](xref:Polly.CircuitBreaker.CircuitBreakerStrategyOptions`1)
- **Extension(s)**:
  - `AddCircuitBreaker`
- **Strategy Type**: Reactive
- **Exception(s)**:
  - [`BrokenCircuitException`](xref:Polly.CircuitBreaker.BrokenCircuitException): Thrown when a circuit is broken and the action was not executed.
  - [`IsolatedCircuitException`](xref:Polly.CircuitBreaker.IsolatedCircuitException): Thrown when a circuit is isolated (held open) by manual override.

---

The circuit breaker **reactive** resilience strategy shortcuts the execution if the underlying resource is detected as unhealthy. The detection process is done via sampling. If the sampled executions' failure-success ratio exceeds a predefined threshold then a circuit breaker will prevent any new executions by throwing a `BrokenCircuitException`. After a preset duration the circuit breaker performs a probe, because the assumption is that this period was enough for the resource to self-heal. Depending on the outcome of the probe, the circuit will either allow new executions or continue to block them. If an execution is blocked by the circuit breaker, the thrown exception may indicate the amount of time executions will continue to be blocked through its `RetryAfter` property.

> [!NOTE]
> Be aware that the Circuit Breaker strategy [rethrows all exceptions](https://github.com/App-vNext/Polly/wiki/Circuit-Breaker#exception-handling), including those that are handled. A Circuit Breaker's role is to monitor faults and break the circuit when a certain threshold is reached; it does not manage retries. Combine the Circuit Breaker with a Retry strategy if needed.

## Usage

<!-- snippet: circuit-breaker -->
```cs
// Circuit breaker with default options.
// See https://www.pollydocs.org/strategies/circuit-breaker#defaults for defaults.
var optionsDefaults = new CircuitBreakerStrategyOptions();

// Circuit breaker with customized options:
// The circuit will break if more than 50% of actions result in handled exceptions,
// within any 10-second sampling duration, and at least 8 actions are processed.
var optionsComplex = new CircuitBreakerStrategyOptions
{
    FailureRatio = 0.5,
    SamplingDuration = TimeSpan.FromSeconds(10),
    MinimumThroughput = 8,
    BreakDuration = TimeSpan.FromSeconds(30),
    ShouldHandle = new PredicateBuilder().Handle<SomeExceptionType>()
};

// Circuit breaker using BreakDurationGenerator:
// The break duration is dynamically determined based on the properties of BreakDurationGeneratorArguments.
var optionsBreakDurationGenerator = new CircuitBreakerStrategyOptions
{
    FailureRatio = 0.5,
    SamplingDuration = TimeSpan.FromSeconds(10),
    MinimumThroughput = 8,
    BreakDurationGenerator = static args => new ValueTask<TimeSpan>(TimeSpan.FromMinutes(args.FailureCount)),
};

// Handle specific failed results for HttpResponseMessage:
var optionsShouldHandle = new CircuitBreakerStrategyOptions<HttpResponseMessage>
{
    ShouldHandle = new PredicateBuilder<HttpResponseMessage>()
        .Handle<SomeExceptionType>()
        .HandleResult(response => response.StatusCode == HttpStatusCode.InternalServerError)
};

// Monitor the circuit state, useful for health reporting:
var stateProvider = new CircuitBreakerStateProvider();
var optionsStateProvider = new CircuitBreakerStrategyOptions<HttpResponseMessage>
{
    StateProvider = stateProvider
};

var circuitState = stateProvider.CircuitState;

/*
CircuitState.Closed - Normal operation; actions are executed.
CircuitState.Open - Circuit is open; actions are blocked.
CircuitState.HalfOpen - Recovery state after break duration expires; actions are permitted.
CircuitState.Isolated - Circuit is manually held open; actions are blocked.
*/

// Manually control the Circuit Breaker state:
var manualControl = new CircuitBreakerManualControl();
var optionsManualControl = new CircuitBreakerStrategyOptions
{
    ManualControl = manualControl
};

// Manually isolate a circuit, e.g., to isolate a downstream service.
await manualControl.IsolateAsync();

// Manually close the circuit to allow actions to be executed again.
await manualControl.CloseAsync();

// Add a circuit breaker strategy with a CircuitBreakerStrategyOptions{<TResult>} instance to the pipeline
new ResiliencePipelineBuilder().AddCircuitBreaker(optionsDefaults);
new ResiliencePipelineBuilder<HttpResponseMessage>().AddCircuitBreaker(optionsStateProvider);
```
<!-- endSnippet -->

### Failure handling

The circuit breaker returns the result / exception during the sampling period. Once the strategy opens the circuit, every subsequent call will be shortcut with a `BrokenCircuitException`.

<!-- snippet: circuit-breaker-failure-handling -->
```cs
var pipeline = new ResiliencePipelineBuilder()
    .AddCircuitBreaker(new CircuitBreakerStrategyOptions
    {
        FailureRatio = 0.1,
        SamplingDuration = TimeSpan.FromSeconds(1),
        MinimumThroughput = 3,
        BreakDuration = TimeSpan.FromSeconds(30),
        ShouldHandle = new PredicateBuilder().Handle<SomeExceptionType>()
    })
    .Build();

for (int i = 0; i < 10; i++)
{
    try
    {
        pipeline.Execute(() => throw new SomeExceptionType());
    }
    catch (SomeExceptionType)
    {
        Console.WriteLine("Operation failed please try again.");
    }
    catch (BrokenCircuitException)
    {
        Console.WriteLine("Operation failed too many times please try again later.");
    }
}
```
<!-- endSnippet -->

The output would look like this:

```none
Operation failed please try again.
Operation failed please try again.
Operation failed please try again.
Operation failed too many times please try again later.
Operation failed too many times please try again later.
Operation failed too many times please try again later.
Operation failed too many times please try again later.
Operation failed too many times please try again later.
Operation failed too many times please try again later.
Operation failed too many times please try again later.
```

The `BrokenCircuitException` and the `IsolatedCircuitException` provide access to the following properties:

- `RetryAfter`;
- `TelemetrySource`.

If a `TimeSpan` value is provided to the optional `RetryAfter` property then this indicates that circuit is open for at least this time period and you should retry your operation no sooner than the value given.

The `TelemetrySource` property is a [`ResilienceTelemetrySource`](xref:Polly.Telemetry.ResilienceTelemetrySource) which allows you retrieve information such as the executed pipeline and strategy. These can be useful if you have multiple circuit breaker strategies in your pipeline and you need to know which strategy caused the `BrokenCircuitException` to be thrown.

## Defaults

| Property                 | Default Value                                           | Description                                                                                                                                               |
|--------------------------|---------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------|
| `ShouldHandle`           | Any exceptions other than `OperationCanceledException`. | Defines a predicate to determine what results and/or exceptions are handled by the circuit breaker strategy.                                              |
| `FailureRatio`           | 0.1                                                     | The failure-success ratio that will cause the circuit to break/open. `0.1` means 10% failed of all sampled executions.                                    |
| `MinimumThroughput`      | 100                                                     | The minimum number of executions that must occur within the specified sampling duration.                                                                  |
| `SamplingDuration`       | 30 seconds                                              | The time period over which the failure-success ratio is calculated.                                                                                       |
| `BreakDuration`          | 5 seconds                                               | Defines a **fixed** time period for which the circuit will remain broken/open before attempting to reset.                                                 |
| `BreakDurationGenerator` | `null`                                                  | This delegate allows you to **dynamically** calculate the break duration by utilizing information that is only available at runtime (like failure count). |
| `ManualControl`          | `null`                                                  | If provided then the circuit's state can be manually controlled via a `CircuitBreakerManualControl` object.                                               |
| `StateProvider`          | `null`                                                  | If provided then the circuit's current state can be retrieved via a `CircuitBreakerStateProvider` object.                                                 |
| `OnClosed`               | `null`                                                  | If provided then it will be invoked after the circuit transitions to either the `Closed` or `Isolated` states.                                            |
| `OnOpened`               | `null`                                                  | If provided then it will be invoked after the circuit transitions to the `Opened` state.                                                                  |
| `OnHalfOpened`           | `null`                                                  | If provided then it will be invoked after the circuit transitions to the `HalfOpened` state.                                                              |

> [!NOTE]
> If both `BreakDuration` and `BreakDurationGenerator` are specified then `BreakDuration` will be ignored.

---

> [!IMPORTANT]
> If the `MinimumThroughput` is not reached during the `SamplingDuration` then the `FailureRatio` is ignored.
> In other words, the circuit will not break even if all of the executions failed when their quantity is below the minimum throughput.

## Telemetry

The circuit breaker strategy reports the following telemetry events:

| Event Name            | Event Severity | When?                                                      |
|-----------------------|----------------|------------------------------------------------------------|
| `OnCircuitClosed`     | `Information`  | Just before the strategy calls the `OnClosed` delegate     |
| `OnCircuitOpened`     | `Error`        | Just before the strategy calls the `OnOpened` delegate     |
| `OnCircuitHalfOpened` | `Warning`      | Just before the strategy calls the `OnHalfOpened` delegate |

Here are some sample events:

```none
Resilience event occurred. EventName: 'OnCircuitOpened', Source: 'MyPipeline/MyPipelineInstance/MyCircuitBreakerStrategy', Operation Key: 'MyCircuitedOperation', Result: 'Exception of type 'CustomException' was thrown.'
    CustomException: Exception of type 'CustomException' was thrown.
        at Program.<>c.<<Main>b__0_1>d.MoveNext()
        ...
        at Polly.ResiliencePipeline.<>c__8`1.<<ExecuteAsync>b__8_0>d.MoveNext() in /_/src/Polly.Core/ResiliencePipeline.AsyncT.cs:line 95

Resilience event occurred. EventName: 'OnCircuitHalfOpened', Source: 'MyPipeline/MyPipelineInstance/MyCircuitBreakerStrategy', Operation Key: 'MyCircuitedOperation', Result: ''

Resilience event occurred. EventName: 'OnCircuitClosed', Source: 'MyPipeline/MyPipelineInstance/MyCircuitBreakerStrategy', Operation Key: 'MyCircuitedOperation', Result: '42'
```

> [!NOTE]
> Please note that the `OnCircuitXYZ` telemetry events will be reported **only if** the circuit breaker strategy transitions from one state into another.
>
> Remember in case of `ManualControl` the `OnCircuitHalfOpened` telemetry event will not be emitted.
>
> Also the `Result` will be **always empty** for the `OnCircuitHalfOpened` telemetry event.

For further information please check out the [telemetry page](../advanced/telemetry.md).

## Diagrams

### State diagram

```mermaid
stateDiagram-v2
direction LR
    [*] --> Closed
    Closed --> Open: Exceeds threshold
    Open --> HalfOpen: Elapses break duration
    HalfOpen --> Closed: Passes the probe
    HalfOpen --> Open: Fails the probe
```

Whenever someone says _the circuit breaks_ that means the Circuit Breaker transitions from the `Closed` state to the `Open` state.

### Simple

Let's suppose we have a circuit breaker strategy with the following configuration:

- `SamplingDuration`: `2 seconds`;
- `MinimumThroughput`: `2`;
- `FailureRatio` : `0.5`.

#### Simple: happy path sequence diagram

The circuit will not break because the actual failure ratio (0.33) will be below the threshold (0.5) after the 3rd call.

```mermaid
sequenceDiagram
    autonumber
    actor C as Caller
    participant P as Pipeline
    participant CB as CircuitBreaker
    participant D as DecoratedUserCallback

    C->>P: Calls ExecuteAsync
    P->>CB: Calls ExecuteCore
    Note over CB: Closed state
    Note over CB, D: Sampling start
    activate CB
    CB->>+D: Invokes
    D->>-CB: Returns result
    CB->>P: Returns result
    P->>C: Returns result

    C->>P: Calls ExecuteAsync
    P->>CB: Calls ExecuteCore
    CB->>+D: Invokes
    D->>-CB: Returns result
    CB->>P: Returns result
    P->>C: Returns result

    C->>P: Calls ExecuteAsync
    P->>CB: Calls ExecuteCore
    CB->>+D: Invokes
    D->>-CB: Fails
    deactivate CB
    Note over CB, D: Sampling end
    CB->>P: Propagates failure
    P->>C: Propagates failure
```

#### Simple: unhappy path sequence diagram

The circuit will break because the actual failure ratio meets the threshold (0.5) after the 2nd call.

```mermaid
sequenceDiagram
    autonumber
    actor C as Caller
    participant P as Pipeline
    participant CB as CircuitBreaker
    participant D as DecoratedUserCallback

    C->>P: Calls ExecuteAsync
    P->>CB: Calls ExecuteCore
    Note over CB: Closed state
    Note over CB, D: Sampling start
    activate CB
    CB->>+D: Invokes
    D->>-CB: Returns result
    CB->>P: Returns result
    P->>C: Returns result

    C->>P: Calls ExecuteAsync
    P->>CB: Calls ExecuteCore
    CB->>+D: Invokes
    D->>-CB: Fails
    Note over CB: Moves to Open state
    CB->>P: Propagates failure
    P->>C: Propagates failure

    C->>P: Calls ExecuteAsync
    P->>CB: Calls ExecuteCore
    CB-->>CB: Rejects request
    CB->>P: Throws <br/>BrokenCircuitException
    P->>C: Propagates exception
    deactivate CB
    Note over CB, D: Sampling end
```

### Complex

Let's suppose we have a circuit breaker strategy with the following configuration:

- `SamplingDuration`: `2 seconds`;
- `MinimumThroughput`: `2`;
- `FailureRatio`: `0.5`;
- `BreakDuration`:`1 second`.

#### Complex: happy path sequence diagram

The circuit will break and later it will transition into the `HalfOpen` state. The probe will then succeed, so the circuit breaker will go back to the normal (`Closed`) state.

```mermaid
sequenceDiagram
    autonumber
    actor C as Caller
    participant P as Pipeline
    participant CB as CircuitBreaker
    participant D as DecoratedUserCallback

    C->>P: Calls ExecuteAsync
    P->>CB: Calls ExecuteCore
    Note over CB: Closed state
    CB->>+D: Invokes
    D->>-CB: Fails
    CB->>P: Propagates failure
    P->>C: Propagates failure

    C->>P: Calls ExecuteAsync
    P->>CB: Calls ExecuteCore
    CB->>+D: Invokes
    D->>-CB: Fails
    Note over CB: Moves to Open state
    Note over CB: Break duration start
    CB->>P: Propagates failure
    P->>C: Propagates failure

    C->>P: Calls ExecuteAsync
    P->>CB: Calls ExecuteCore
    CB-->>CB: Rejects request
    CB->>P: Throws <br/>BrokenCircuitException
    P->>C: Propagates exception

    C->>P: Calls ExecuteAsync
    P->>CB: Calls ExecuteCore
    Note over CB: Break duration end
    Note over CB: Moves to HalfOpen state
    CB->>+D: Invokes
    D->>-CB: Returns result
    Note over CB: Moves to Closed state
    CB->>P: Returns result
    P->>C: Returns result
```

#### Complex: unhappy path sequence diagram

The circuit will break and later it will transition into the `HalfOpen` state. The probe will then fail, so the circuit breaker will become broken again (the `Open` state).

```mermaid
sequenceDiagram
    autonumber
    actor C as Caller
    participant P as Pipeline
    participant CB as CircuitBreaker
    participant D as DecoratedUserCallback

    C->>P: Calls ExecuteAsync
    P->>CB: Calls ExecuteCore
    Note over CB: Closed state
    CB->>+D: Invokes
    D->>-CB: Fails
    CB->>P: Propagates failure
    P->>C: Propagates failure

    C->>P: Calls ExecuteAsync
    P->>CB: Calls ExecuteCore
    CB->>+D: Invokes
    D->>-CB: Fails
    Note over CB: Moves to Open state
    Note over CB: Break duration start
    CB->>P: Propagates failure
    P->>C: Propagates failure

    C->>P: Calls ExecuteAsync
    P->>CB: Calls ExecuteCore
    CB-->>CB: Rejects request
    CB->>P: Throws <br/>BrokenCircuitException
    P->>C: Propagates exception

    C->>P: Calls ExecuteAsync
    P->>CB: Calls ExecuteCore
    Note over CB: Break duration end
    Note over CB: Moves to HalfOpen state
    CB->>+D: Invokes
    D->>-CB: Fails
    Note over CB: Moves to Open state
    CB->>P: Propagates failure
    P->>C: Propagates failure
```

#### Complex: dynamic break duration sequence diagram

This sequence diagram illustrates the behavior of a circuit breaker using a `BreakDurationGenerator`. The generator dynamically calculates the break duration based on specific criteria, such as the number of failures:

```mermaid
sequenceDiagram
    autonumber
    actor C as Caller
    participant P as Pipeline
    participant CB as CircuitBreaker
    participant BDG as BreakDurationGenerator
    participant D as DecoratedUserCallback

    C->>P: Calls ExecuteAsync
    P->>CB: Calls ExecuteCore
    Note over CB: Closed state
    CB->>+D: Invokes
    D->>-CB: Fails
    Note over CB: Moves to Open state
    CB->>+BDG: Calls Generator
    BDG->>-CB: Returns calculated <br/> duration
    Note over CB: Break duration start
    CB->>P: Propagates failure
    P->>C: Propagates failure

    C->>P: Calls ExecuteAsync
    P->>CB: Calls ExecuteCore
    CB-->>CB: Rejects request
    CB->>P: Throws <br/>BrokenCircuitException
    P->>C: Propagates exception

    C->>P: Calls ExecuteAsync
    P->>CB: Calls ExecuteCore
    Note over CB: Break duration end
    Note over CB: Moves to HalfOpen state
    CB->>+D: Invokes
    D->>-CB: Returns result
    Note over CB: Moves to Closed state
    CB->>P: Returns result
    P->>C: Returns result
```

## Resources

- [Making the Netflix API More Resilient](https://techblog.netflix.com/2011/12/making-netflix-api-more-resilient.html)
- [Circuit Breaker by Martin Fowler](https://martinfowler.com/bliki/CircuitBreaker.html)
- [Circuit Breaker Pattern by Microsoft](https://msdn.microsoft.com/en-us/library/dn589784.aspx)
- [Original Circuit Breaking Article](https://web.archive.org/web/20160106203951/http://thatextramile.be/blog/2008/05/the-circuit-breaker)

## Anti-patterns

Over the years, many developers have used Polly in various ways. Some of these
recurring patterns may not be ideal. The sections below highlight anti-patterns to avoid.

### Using different sleep duration between retry attempts based on Circuit Breaker state

Imagine that we have an inner Circuit Breaker and an outer Retry strategies.

We would like to define the retry in a way that the sleep duration calculation is taking into account the Circuit Breaker's state.

❌ DON'T

Use a closure to branch based on circuit breaker state:

<!-- snippet: circuit-breaker-anti-pattern-circuit-aware-retry -->
```cs
var stateProvider = new CircuitBreakerStateProvider();
var circuitBreaker = new ResiliencePipelineBuilder()
    .AddCircuitBreaker(new()
    {
        ShouldHandle = new PredicateBuilder().Handle<HttpRequestException>(),
        BreakDuration = TimeSpan.FromSeconds(5),
        StateProvider = stateProvider
    })
    .Build();

var retry = new ResiliencePipelineBuilder()
    .AddRetry(new()
    {
        ShouldHandle = new PredicateBuilder()
            .Handle<HttpRequestException>()
            .Handle<BrokenCircuitException>(),
        DelayGenerator = args =>
        {
            TimeSpan? delay = TimeSpan.FromSeconds(1);
            if (stateProvider.CircuitState == CircuitState.Open)
            {
                delay = TimeSpan.FromSeconds(5);
            }

            return ValueTask.FromResult(delay);
        }
    })
    .Build();
```
<!-- endSnippet -->

**Reasoning**:

- By default, each strategy is independent and has no reference to any other strategies.
- We use the (`stateProvider`) to access the Circuit Breaker's state. However, this approach is not optimal as the retry strategy's `DelayGenerator` varies based on state.
- This solution is delicate because the break duration and the sleep duration aren't linked.
- If a future code maintainer modifies the `circuitBreaker`'s `BreakDuration`, they might overlook adjusting the sleep duration.

✅ DO

Use `Context` to pass information between strategies:

<!-- snippet: circuit-breaker-pattern-circuit-aware-retry -->
```cs
var circuitBreaker = new ResiliencePipelineBuilder()
    .AddCircuitBreaker(new()
    {
        ShouldHandle = new PredicateBuilder().Handle<HttpRequestException>(),
        BreakDuration = TimeSpan.FromSeconds(5),
        OnOpened = static args =>
        {
            args.Context.Properties.Set(SleepDurationKey, args.BreakDuration);
            return ValueTask.CompletedTask;
        },
        OnClosed = args =>
        {
            args.Context.Properties.Set(SleepDurationKey, null);
            return ValueTask.CompletedTask;
        }
    })
    .Build();

var retry = new ResiliencePipelineBuilder()
    .AddRetry(new()
    {
        ShouldHandle = new PredicateBuilder()
            .Handle<HttpRequestException>()
            .Handle<BrokenCircuitException>(),
        DelayGenerator = static args =>
        {
            _ = args.Context.Properties.TryGetValue(SleepDurationKey, out var delay);
            delay ??= TimeSpan.FromSeconds(1);
            return ValueTask.FromResult(delay);
        }
    })
    .Build();
```
<!-- endSnippet -->

**Reasoning**:

- Both strategies are less coupled in this approach since they rely on the context and the `sleepDurationKey` components.
- The Circuit Breaker shares the `BreakDuration` through the context when it breaks. When it transitions back to Closed, the sharing is revoked.
- The Retry strategy fetches the sleep duration dynamically without knowing any specific knowledge about the Circuit Breaker.
- If adjustments are needed for the `BreakDuration`, they can be made in one place.

### Wrapping each endpoint with a circuit breaker

Imagine that you have to call N number of services via `HttpClient`s.
You want to decorate all downstream calls with the service-aware Circuit Breaker.

❌ DON'T

Use a collection of Circuit Breakers and explicitly call `ExecuteAsync()`:

<!-- snippet: circuit-breaker-anti-pattern-cb-per-endpoint -->
```cs
// Defined in a common place
var uriToCbMappings = new Dictionary<Uri, ResiliencePipeline>
{
    [new Uri("https://downstream1.com")] = GetCircuitBreaker(),
    // ...
    [new Uri("https://downstreamN.com")] = GetCircuitBreaker()
};

// Used in the downstream 1 client
var downstream1Uri = new Uri("https://downstream1.com");
await uriToCbMappings[downstream1Uri].ExecuteAsync(CallXYZOnDownstream1, CancellationToken.None);
```
<!-- endSnippet -->

**Reasoning**:

- Whenever you use an `HttpClient`, you must have a reference to the `uriToCbMappings` dictionary.
- It's your responsibility to decorate each network call with the corresponding circuit breaker.

✅ DO

Use named `HttpClient`s and then call `AddResilienceHandler` extension that is included in the [`Microsoft.Extensions.Http.Resilience`](https://www.nuget.org/packages/Microsoft.Extensions.Http.Resilience) package.

<!-- snippet: circuit-breaker-pattern-cb-per-endpoint -->
```cs
services
  .AddHttpClient("my-client")
  .AddResilienceHandler("circuit-breaker", builder =>
  {
      builder.AddCircuitBreaker(new());
  })
  .SelectPipelineByAuthority(); // This call ensures that circuit breaker is cached by each URL authority
```
<!-- endSnippet -->

And then use it:

<!-- snippet: circuit-breaker-pattern-cb-per-endpoint-usage -->
```cs
HttpClient client = httpClientFactory.CreateClient("my-client");

await client.GetAsync(new Uri("https://downstream1.com/some-path"));
```
<!-- endSnippet -->

**Reasoning**:

- The `HttpClient` integrates with Circuit Breaker during startup by using the `AddResilienceHandler` extension.
- There's no need to call `ExecuteAsync()` directly. The `DelegatingHandler` handles it automatically.
- By using the `SelectPipelineByAuthority` extension method, the resilience handler caches and assigns resilience pipeline to each authority (scheme + host + port) that is extracted from HTTP request message.

### Reducing thrown exceptions

In case of Circuit Breaker when it is either in the `Open` or `Isolated` state new requests are rejected immediately.

That means the strategy will throw either a `BrokenCircuitException` or an `IsolatedCircuitException` respectively.

❌ DON'T

Use guard expression to call `Execute{Async}` only if the circuit is not broken:

<!-- snippet: circuit-breaker-anti-pattern-reduce-thrown-exceptions -->
```cs
var stateProvider = new CircuitBreakerStateProvider();
var circuitBreaker = new ResiliencePipelineBuilder()
    .AddCircuitBreaker(new()
    {
        ShouldHandle = new PredicateBuilder().Handle<HttpRequestException>(),
        BreakDuration = TimeSpan.FromSeconds(0.5),
        StateProvider = stateProvider
    })
    .Build();

if (stateProvider.CircuitState
    is not CircuitState.Open
    and not CircuitState.Isolated)
{
    var response = await circuitBreaker.ExecuteAsync(static async ct =>
    {
        return await IssueRequest();
    }, CancellationToken.None);

    // Your code goes here to process response
}
```
<!-- endSnippet -->

**Reasoning**:

- The problem with this approach is that the circuit breaker will never transition into the `HalfOpen` state.
- The circuit breaker does not act as an active object. In other words the state transition does not happen automatically in the background.
- The circuit transition into the `HalfOpen` state when the `Execute{Async}` method is called and the `BreakDuration` elapsed.

✅ DO

Use `ExecuteOutcomeAsync` to avoid throwing exception:

<!-- snippet: circuit-breaker-pattern-reduce-thrown-exceptions -->
```cs
var context = ResilienceContextPool.Shared.Get();
var circuitBreaker = new ResiliencePipelineBuilder()
    .AddCircuitBreaker(new()
    {
        ShouldHandle = new PredicateBuilder().Handle<HttpRequestException>(),
        BreakDuration = TimeSpan.FromSeconds(0.5),
    })
    .Build();

Outcome<HttpResponseMessage> outcome = await circuitBreaker.ExecuteOutcomeAsync(static async (ctx, state) =>
{
    var response = await IssueRequest();
    return Outcome.FromResult(response);
}, context, "state");

ResilienceContextPool.Shared.Return(context);

if (outcome.Exception is BrokenCircuitException)
{
    // The execution was stopped by the circuit breaker
}
else
{
    HttpResponseMessage response = outcome.Result!;
    // Your code goes here to process the response
}
```
<!-- endSnippet -->

**Reasoning**:

- The `ExecuteOutcomeAsync` is a low-allocation API which does not throw exceptions; rather it captures them inside an `Outcome` data structure.
- Since you are calling one of the `Execute` methods, that's why the circuit breaker can transition into the `HalfOpen` state.
