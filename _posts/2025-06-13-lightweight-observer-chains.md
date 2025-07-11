---
layout: post
title: Lightweight observer chains
tags: [C#, .NET, observable, reactive]
permalink: /lightweight-observer-chains
---

Previously I wrote about another approach making properties observable for reactive programming in .NET using the Kinetic project. This time I would like to talk about a non-less interesting feature there, boxing an observer chain into a single-state machine.

The idea behind that isn't new at all. You might hear about it from projects trying to make LINQ for `IEnumerable<T>` to be zero-allocating. That's not possible for the observable pattern as it involves subscriptions, but what is achievable here is memory pressure reduction.

# Take one: struct it

As I mentioned earlier, the main difference between the interactive and reactive patterns lies in the direction of the control flow. In the first case a program requests data from an iterator that can be sourced by another one, and so on making a chain that executes backward. That's a request-response interaction, and there's just one party asking for data. In reactive programming, data is published into topics that anyone can subscribe to and react to when a notification comes. There a chain of operators is executed exactly in the same order as it was written, and each operator can have multiple subscribers, but in most cases, there's just one listener. That's a real prodigal.

So, how would one fix that? Surely not without structs as we can pack everything into a single state machine and allow the JIT compiler to inline every call between the operators that can be inlined.

```csharp
interface IOperator<TSource, TResult> : IObserver<TSource>, IObservable<TResult>
{ }

struct SelectOperator<TSource, TResult, TContinuation> : IOperator<TSource, TResult>
    where TContinuation : IObserver<TResult>
{ }

struct WhereOperator<TSource, TContinuation> : IOperator<TSource, TSource>
    where TContinuation : IObserver<TSource>
{ }

struct PublishOperator<TSource> : IOperator<TSource, TSource>
{ }
```

And then a simple `x.Where(...).Select(...)` expression can be written as:

```csharp
new SelectOperator<TSource, TResult, WhereOperator<TResult, PublishOperator<TResult>>>(
    new WhereOperator<TResult, PublishOperator<TResult>>(
        new PublishOperator<TResult>())):
```

Definitely, it's not how anyone would like to do it. Hopefully, we have the builder pattern.

```csharp
public interface IOperatorFactory<TSource, TResult>
{
    IDisposable ApplyOperator<TContinuation>(TContinuation continuation)
        where TContinuation : IOperator<TSource, TResult>;
}

public struct SelectOperatorFactory<TSource, TResult, TPrevious>
    where TPrevious : IOperatorFactory<TSource>
{
    public Func<TSource, TResult> Select { get; init; }
    public TPrevious Previous { get; init; }

    public IDisposable ApplyOperator<TSource, TContinuation>(TContinuation continuation)
        where TContinuation : IOperator<TSource, TResult>
    {
        return Previous.ApplyOperator<SelectOperator<TSource, TResult, TContinuation>>(
            new(continuation, Select);
    }
}

public static class ObserverExtensions
{
    public static SelectOperatorFactory<TSource, TResult, TPrevious> Select(this TPrevious source, Func<TSource, TResult> select)
        where TPrevious : IOperatorFactory<TSource>
    {
        return new()
        {
            Previous = previous,
            Select = select,
        };
    }
}
```

It looks good so far, but it won't work due to a Roslyn's limitation. It isn't good at generic argument deduction and that's by design.

```csharp
new List<int>().M(); // error CS0411: The type arguments for method 'C.M<X, Y>(X)' cannot be inferred from the usage. Try specifying the type arguments explicitly.

public static class C {
    public static void M<X, Y>(this X x)
        where X : IEnumerable<Y> {
    }
}
```

One potential workaround is adding operator creation methods to each factory. That's too much even with help from source generators, plus it won't be extensible outside of that library. That really sucks, but there are two more options at least not counting that we can use another language as it's a non-goal.

# Take two: virtual generic dispatch

Let's reimagine what was previously done by dropping complexity that Roslyn cannot handle. First, we need to reduce our operator type to something simple enough, a type with a minimal number of generic arguments.

```csharp
public abstract class Operator<TResult>
{
}
```

Yeah, that's almost `IObservable<T>` and it has no problems with extension method calls. Now let's add a derived type, an operator too, but one that has an idea what its ancestor is.

```csharp
public abstract class Operator<TSource, TResult> : Operator<TResult>
{
    protected Operator<TSource> Source { get; }

    protected Operator(Operator<TSource> source)
    {
        Source = source;
    }
}
```

Last, there should be a method that allows building a single stack machine based on structs. For that we can utilize virtual generic calls, a quite unique dispatch available for us in .NET.


```csharp
public interface IStateMachine<T> : IObserver<T>, IDisposable
{
}

public interface IStateMachineBoxFactory<TBox>
{
    TBox Create<TSource, TStateMachine>(in TStateMachine stateMachine)
        where TStateMachine : struct, IStateMachine<TSource>;
}

public abstract class Operator<TResult>
{
    public abstract TBox Build<TBox, TBoxFactory, TContinuation>(in TBoxFactory boxFactory, in TContinuation continuation)
        where TBoxFactory : struct, IStateMachineBoxFactory<TBox>
        where TContinuation : struct, IStateMachine<TResult>;
}
```

A simple `Select` operator can be expressed like another class derived from `Operator<TSource, TResult>` with the corresponding state machine:

```csharp
internal sealed class SelectOperator<TSource, TResult> : Operator<TSource, TResult>
{
    private readonly Func<TSource, TResult> _selector;

    public SelectOperator(Operator<TSource> source) :
        base(source)
    {
        _selector = selector;
    }

    public override TBox Build<TBox, TBoxFactory, TContinuation>(in TBoxFactory boxFactory, in TContinuation continuation) =>
        Source.Build<TBox, TBoxFactory, StateMachine<TContinuation>>(boxFactory, new(continuation, _selector));

    private struct StateMachine<TContinuation> : IStateMachine<TSource>
        where TContinuation: struct, IStateMachine<TResult>
    {
        private TContinuation _continuation;
        private readonly Func<TSource, TResult> _selector;

        public StateMachine(in TContinuation continuation, Func<TSource, TResult> selector)
        {
            _continuation = continuation;
            _selector = selector;
        }

        public void OnNext(TSource value) =>
            _continuation.OnNext(_selector(value));

        public void OnError(Exception error) =>
            _continuation.OnError(error);

        public void OnCompleted(TSource value) =>
            _continuation.OnCompleted();
    }
}
```

Other operators aren't different, but all of them end up in calling the `Build` method on the `Source` though, and there should be one operator that doesn't do that.

```csharp
internal sealed class BoxOperator<TSource> : Operator<TSource>
{
    private readonly IObservable<TSource> _source;

    public BoxOperator(IObservable<TSource> source) =>
        _source = source;

    public override TBox Build<TBox, TBoxFactory, TContinuation>(in TBoxFactory boxFactory, in TContinuation continuation)
    {
        var box = boxFactory.Create<TSource, Continuation>(boxFactory, new(continuation, _selector));

        // Actually, it's more complicated as not every box
        // can be an observable and we don't restrict that.
        _observable.Subscribe((IObserver<TSource>) box);

        return box;
    }
}
```

A box here is nothing than a class holding some state machine in it with maybe some additional details like some interfaces being implemented like `IObservable<T>` or `IValueTaskSource<T>` and a final state machine that routes handling back to that class. Actually, it's not implemented as simple as I described, but that's the idea and it's simple. Quite enough for now as the focus is on the outer design, on operators.

With just two extension methods the solution is complete. The reason why two methods are needed is that one is for `Operator<TSource>` and another one `IObservable<T>`.

```csharp
public static class OperatorExtensions
{
    public static Operator<TResult> Select<TSource, TResult>(this Operator<TSource> source, Func<TSource, TResult> selector) =>
        new SelectOperator<TSource, TResult>(source, selector);

    public static Operator<TResult> Select<TSource, TResult>(this IObservable<TSource> source, Func<TSource, TResult> selector) =>
        new BoxOperator<TSource>(source).Select(selector);
}
```

At this point, the goal of making a single box for a continuous observer chain construction has been achieved. No more virtual calls in between, but just one virtual call to notify the box and start the handling process. If there's thread safety based on locks, then it needs to be done only for the entry point state machine, so that part shall be dirty cheap also. How much? The numbers of a simple benchmark can show that!

```csharp
[MemoryDiagnoser]
public class HandleChainBenchmarks
{
    private PublishSubject<int> _lightweight = new();
    private PublishSubject<int> _reactive = new();

    [Params(1, 2, 3, 4, 5, 10)]
    public int ChainLength { get; set; }

    [GlobalSetup]
    public void Setup()
    {
        var lightweight = _lightweight.ToOperator();
        var reactive = _reactive as IObservable<int>;

        for (var index = 0; index < ChainLength; index += 1)
        {
            lightweight = lightweight.Select(x => x * 2);
            reactive = System.Reactive.Linq.Observable.Select(reactive, x => x * 2);
        }

        _ = lightweight.ToObservable();
    }

    [Benchmark]
    public void Lightweight() =>
        _lightweight.OnNext(42);

    [Benchmark]
    public void Reactive() =>
        _reactive.OnNext(42);
}
```

| Method      | ChainLength | Mean      | Error     | StdDev    | Allocated |
|------------ |------------ |----------:|----------:|----------:|----------:|
| Lightweight | 1           |  6.513 ns | 1.2197 ns | 0.0669 ns |         - |
| Reactive    | 1           |  1.755 ns | 0.0589 ns | 0.0032 ns |         - |
| Lightweight | 2           |  7.664 ns | 0.3312 ns | 0.0182 ns |         - |
| Reactive    | 2           |  1.749 ns | 0.3682 ns | 0.0202 ns |         - |
| Lightweight | 3           |  9.069 ns | 1.2868 ns | 0.0705 ns |         - |
| Reactive    | 3           |  1.672 ns | 0.0013 ns | 0.0001 ns |         - |
| Lightweight | 4           |  9.309 ns | 0.7826 ns | 0.0429 ns |         - |
| Reactive    | 4           |  1.651 ns | 0.0284 ns | 0.0016 ns |         - |
| Lightweight | 5           | 10.519 ns | 0.0778 ns | 0.0043 ns |         - |
| Reactive    | 5           |  1.670 ns | 0.0429 ns | 0.0024 ns |         - |
| Lightweight | 10          | 17.393 ns | 0.0629 ns | 0.0034 ns |         - |
| Reactive    | 10          |  1.694 ns | 0.3569 ns | 0.0196 ns |         - |

And... It seems that Reactive always outperforms the thing we just made. How? Well, the additional execution time comes from the fact that the demonstrated approach diverges from the canonical implementation in .NET where a returned observable isn't one that actually handles the stream. It only serves as an operator in our case and makes a real handler on subscribe. That's what should be added to the benchmark to make it correct.

```csharp
[GlobalSetup]
public void Setup()
{
    var lightweight = _lightweight.ToOperator();
    var reactive = _reactive as IObservable<int>;

    for (var index = 0; index < ChainLength; index += 1)
    {
        lightweight = lightweight.Select(x => x * 2);
        reactive = System.Reactive.Linq.Observable.Select(reactive, x => x * 2);
    }

    _ = lightweight.ToObservable();
    _ = reactive.Subscribe(NoOp<int>.Instance); // 👈 that's what we need
}
```

Reactive on purpose does lazy creation of handlers known as sinks. Imagine a situation where you have a replay subject or another one that pushes messages on subscribe. If operators would immediately subscribe to it then they will consume the data before a continuation is connected. That's why Reactive uses delayed initialization.

We can support that behavior too by packing an operator chain into a fake observable that will build a new state machine and box it on every subscription, that's not that difficult and makes it compatible with the common implementation. On the other hand instead of calling `ToObservable()` one can do `Subscribe(IObserver<T>)` where the observer is a subject.

What behavior is the right one? Both, though I think that deferred sink creation should be explicit as it gives more control to the developer and lets them decide what code should exactly do without wasting limited resources.

Now let's run the fixed benchmark and observe the actual observer materialization costs:

| Method      | ChainLength | Mean      | Error     | StdDev    | Allocated |
|------------ |------------ |----------:|----------:|----------:|----------:|
| Lightweight | 1           |  6.294 ns | 1.1997 ns | 0.0658 ns |         - |
| Reactive    | 1           |  6.541 ns | 1.9919 ns | 0.1092 ns |         - |
| Lightweight | 2           |  7.255 ns | 0.2327 ns | 0.0128 ns |         - |
| Reactive    | 2           |  9.456 ns | 1.9292 ns | 0.1057 ns |         - |
| Lightweight | 3           |  9.140 ns | 2.7408 ns | 0.1502 ns |         - |
| Reactive    | 3           | 11.835 ns | 0.4100 ns | 0.0225 ns |         - |
| Lightweight | 4           | 10.147 ns | 1.0510 ns | 0.0576 ns |         - |
| Reactive    | 4           | 14.929 ns | 0.6923 ns | 0.0379 ns |         - |
| Lightweight | 5           | 11.141 ns | 1.7854 ns | 0.0979 ns |         - |
| Reactive    | 5           | 16.544 ns | 0.6192 ns | 0.0339 ns |         - |
| Lightweight | 10          | 19.021 ns | 2.9314 ns | 0.1607 ns |         - |
| Reactive    | 10          | 28.396 ns | 1.2660 ns | 0.0694 ns |         - |

It's better now, really. For a single-operator chain, there's no difference, but with each additional operator in the chain, it takes less and less time for a value to be handled as no virtual calls are made. Moreover, the JIT compiler can inline operators' bodies and do more aggressive optimizations if possible. Yet, there's a question. At which cost it was achieved? To answer it we need one more benchmark:

```csharp
[MemoryDiagnoser]
public class CreateChainBenchmarks
{
    private PublishSubject<int> _observable = new();

    [Params(1, 2, 3, 4, 5, 10)]
    public int ChainLength { get; set; }

    [Benchmark]
    public IObservable<int> Lightweight()
    {
        var op = _observable.ToOperator();
        for (int index = 0; index < ChainLength; index += 1)
            op = op.Select(x => x);
        return op.ToObservable();
    }

    [Benchmark]
    public IDisposable Reactive()
    {
        var op = _observable as IObservable<int>;
        for (int index = 0; index < ChainLength; index += 1)
            op = System.Reactive.Linq.Observable.Select(op, x => x);
        return op.Subscribe(NoOp<int>.Instance);  // 👈 remember about it
    }
}
```

Before seeing the results let's remember that the canonical reactive implementation builds real observers only when there's a subscription. That's why there's a no-op observer with empty methods:

```csharp
public sealed class NoOp<T> : IObserver<T>
{
    public static readonly NoOp<T> Instance = new();

    public void OnCompleted() { }

    public void OnError(Exception error) { }

    public void OnNext(T value) { }
}
```

The numbers now:

| Method      | ChainLength | Mean       | Error     | StdDev   | Gen0   | Gen1   | Allocated |
|------------ |------------ |-----------:|----------:|---------:|-------:|-------:|----------:|
| Lightweight | 1           |   404.2 ns | 123.57 ns |  6.77 ns | 0.0253 | 0.0124 |     160 B |
| Reactive    | 1           |   379.8 ns |  84.76 ns |  4.65 ns | 0.0229 | 0.0114 |     144 B |
| Lightweight | 2           |   402.4 ns |  84.99 ns |  4.66 ns | 0.0315 | 0.0153 |     200 B |
| Reactive    | 2           |   554.4 ns |  62.18 ns |  3.41 ns | 0.0343 | 0.0172 |     216 B |
| Lightweight | 3           |   443.0 ns |  55.00 ns |  3.01 ns | 0.0381 | 0.0124 |     240 B |
| Reactive    | 3           |   645.0 ns |  43.42 ns |  2.38 ns | 0.0458 | 0.0229 |     288 B |
| Lightweight | 4           |   492.6 ns |  62.23 ns |  3.41 ns | 0.0439 | 0.0143 |     280 B |
| Reactive    | 4           |   801.4 ns | 172.07 ns |  9.43 ns | 0.0572 | 0.0286 |     360 B |
| Lightweight | 5           |   572.6 ns | 197.65 ns | 10.83 ns | 0.0505 | 0.0172 |     320 B |
| Reactive    | 5           |   922.4 ns | 294.58 ns | 16.15 ns | 0.0687 | 0.0343 |     432 B |
| Lightweight | 10          |   954.3 ns | 311.04 ns | 17.05 ns | 0.0820 | 0.0267 |     520 B |
| Reactive    | 10          | 1,621.0 ns | 310.53 ns | 17.02 ns | 0.1259 | 0.0610 |     792 B |

Honestly, I'm pretty impressed that the lightweight approach is faster as it does expensive generic virtual calls. At the same time, it does fewer allocations that add some pressure even when they are cheap in .NET. Though, while writing that article I wondered if it could be done even better.

# Take three: blend it

Above there was an attempt to make purely using structs and avoid allocation of factories making observers. There I stated that it's possible but has huge drawbacks. I'm taking my words back as it turns out that one can make it both extensible and zero-allocating in almost all cases. The exception here comes from the inability of the Roslyn compiler to deduce generic arguments from the outer context as the Rust compiler does.

As zero allocations mean structs the `Operator<T>` class turns into an interface:

```csharp
public interface IOperator<T>
{
    TBox Box<TBox, TBoxFactory, TStateMachine>(in TBoxFactory boxFactory, in TStateMachine stateMachine)
        where TBoxFactory : struct, IStateMachineBoxFactory<TBox>
        where TStateMachine : struct, IStateMachine<T>;
}
```

The transform aka `Select` operator then turns to be a struct:

```csharp
public readonly struct Select<TObservable, TSource, TResult> : IOperator<TResult>
    where TObservable : IOperator<TSource>
{
    private readonly TObservable _observable;
    private readonly Func<TSource, TResult> _selector;

    public SelectOp(TObservable observable, Func<TSource, TResult> selector)
    {
        _observable = observable;
        _selector = selector;
    }

    public TBox Box<TBox, TBoxFactory, TStateMachine>(in TBoxFactory boxFactory, in TStateMachine stateMachine)
        where TBoxFactory : struct, IStateMachineBoxFactory<TBox>
        where TStateMachine : struct, IStateMachine<TResult>
    {
        return _observable.Box<TBox, TBoxFactory, SelectStateMachine<TSource, TResult, TStateMachine>>(
            boxFactory, new(stateMachine, _selector));
    }
}

internal struct SelectStateMachine<TSource, TResult, TContinuation> : IStateMachine<TSource>
    where TContinuation : struct, IStateMachine<TResult>
{
    // as usual and nothing interesting to show again
}
```

Due to `CS0411` and the compiler limitation extensions cannot accept structs. That's why we need to introduce an intermediate struct that will expose all operators and will hold a chain of them:

```csharp
public readonly struct Observer<TOperator, T>
    where TOperator : IOperator<T>
{
    private readonly TOperator _op;

    public Observer(TOperator op) =>
        _op = op;

    //     👇 return a wrapper instead of the real operator
    public Operator<Select<TOperator, T, TResult>, TResult> Select<TResult>(Func<T, TResult> selector) =>
        new(new(_op, selector));

    // turns the chain into a state machine and boxes it 📦
    public IObservable<T> ToObservable() =>
        ObservableFactory<T>.Create(_op);
}
```

Instances of that type cannot be used in delegates without knowing the exact operator. The same applies for extensions too. Inheriting `IOperator<T>` solves these issues.

```csharp
public readonly struct Observer<TOperator, T> : IOperator<T>
{
    public TBox Box<TBox, TBoxFactory, TStateMachine>(in TBoxFactory boxFactory, in TStateMachine stateMachine)
        where TBoxFactory : struct, IStateMachineBoxFactory<TBox>
        where TStateMachine : struct, IStateMachine<TResult>
    {
        return _op.Box<TBox, TBoxFactory, TStateMachine>(
            boxFactory, stateMachine);
    }
}
```

It leads to duplication of operator-creating methods of the `Operator<TOperator, T>` struct as extension methods for `IOperator<T>`:

```csharp
public static class OperatorExtensions
{
    public Operator<Select<IOperator<T>, T, TResult>, TResult> Select<T, TResult>(this IOperator<T> source, Func<T, TResult> selector) =>
        new(new(source, selector));

    public IObservable<T> ToObservable<T>(this IOperator<T> source) =>
        ObservableFactory<T>.Create(source);
}
```

Extensions for `IObservable<T>` shall be present too, giving us three methods in total per operator. For third-party extensions, the rule of three is relaxed just to two, only IObservable<T>` and `IOperator<T>` targets are required then.

```csharp
public static class OperatorExtensions
{
    public Operator<Select<Observe<T>, T, TResult>, TResult> Select<T, TResult>(this IObservable<T> source, Func<T, TResult> selector) =>
        new(new(new(source), selector));
}
```

Benchmarks?

```csharp
[Benchmark]
public IObservable<int> CreateChain_1()
{
    return _subject
        .Select(x => x)
        .ToObservable();
}

[Benchmark]
public IObservable<int> CreateChain_2()
{
    return _subject
        .Select(x => x)
        .Select(x => x)
        .ToObservable();
}

// and so on for chains of 3, 4, 5, and 10 operators...
```

The table below shows that switching to stack allocations and removal of generic virtual calls improves performance by twice. Neat!

| Method         | Mean     | Error     | StdDev  | Code Size | Gen0   | Gen1   | Allocated |
|--------------- |---------:|----------:|--------:|----------:|-------:|-------:|----------:|
| CreateChain_1  | 195.3 ns |  89.45 ns | 4.90 ns |     134 B | 0.0162 | 0.0081 |     104 B |
| CreateChain_2  | 206.6 ns |  94.17 ns | 5.16 ns |     214 B | 0.0176 | 0.0086 |     112 B |
| CreateChain_3  | 240.0 ns |  12.75 ns | 0.70 ns |     236 B | 0.0191 | 0.0095 |     120 B |
| CreateChain_4  | 251.7 ns | 132.14 ns | 7.24 ns |     269 B | 0.0200 | 0.0100 |     128 B |
| CreateChain_5  | 290.6 ns | 105.83 ns | 5.80 ns |     337 B | 0.0215 | 0.0105 |     136 B |
| CreateChain_10 | 566.4 ns | 168.33 ns | 9.23 ns |     906 B | 0.0277 | 0.0134 |     176 B |


# Outro

Okay, we did that! But know what? We just made [`rxRust`](https://github.com/rxRust/rxRust) in C#, no jokes. I wasn't aware of that crate when started poking with the idea of lightweight observers years ago. Yet, the API design is quite the same. I haven't benchmarked against it and cannot say how close the lightweight observer chains are to what is made in Rust. No doubt they are slower because of delegates and other things that are not supported by .NET and/or C#. It's unavoidable, unfortunately. Anyway, the lightweight chains were crafted for a project where UI was done with Avalonia for quick prototyping and delivering the product faster, so the goal was to make it in C# and make it as fast as possible with minimal resource usage. Sadly, the project was closed before I had a chance to test what I named [Kinetic](https://github.com/YohDeadfall/Kinetic). I cannot say that it's production-ready yet as there are some bugs, but hey, that's open source and you can always open an issue or even contribute by making a pull request! Anything constructive is welcome!
