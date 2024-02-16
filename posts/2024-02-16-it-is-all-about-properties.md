# It's all about properties

A few years ago I was working in the entertainment industry and participated in the creation of a brand-new graphical engine and its editor with a primary focus on cinematics. As one might expect the core was made using C++, but for the editor [Avalonia UI](https://avaloniaui.net/) was chosen as an easy-to-use cross-platform UI framework. Sure, it wasn't without the MVVM approach with the support of [Reactive UI](https://www.reactiveui.net/). All of that worked well, but only in the beginning while the editor wasn't very complex and wasn't killing the performance by eating a lot of RAM and consuming too much time on event handling. One might say that the toolset was completely wrong and we should have used Y instead of X, but the story isn't about that. It's about the investigation being made and an attempt to solve the performance issue.

One day a colleague announced a drop-in replacement for subscriptions and change notifications provided by Reactive UI. The approach provided two options for subscriptions. The first one was about specifying the interested property name, while another one used a LINQ expression from which the property to be listened to was determined. Internally there was a dictionary-based cache to keep observables for requested properties and reuse them. While it worked well, it had a few drawbacks, but before going deeper into it let's talk about how ReactiveUI works.

## Reactive UI's approach

The core of Reactive UI starts from the `IReactiveObject` interface:

```csharp
public interface IReactiveObject : INotifyPropertyChanged, INotifyPropertyChanging, IEnableLogger
{
    void RaisePropertyChanging(PropertyChangingEventArgs args);

    void RaisePropertyChanged(PropertyChangedEventArgs args);
}
```

As you can easily spot, it's all about the `System.ComponentModel` notification model, `INotifyPropertyChanged` and `INotifyPropertyChanging` interfaces which were introduced in .NET Framework 2 and used extensively in WinForms applications for binding data in UI. Later the same model was reused in WPF, an MVVM framework where properties were used for data transfer and events for change notifications.

In the code below we implemented a simple class supporting property change notifications:

```csharp
class Person : INotifyPropertyChanged
{
    public event PropertyChangedEventHandler? PropertyChanged;

    private string? _firstName;
    public string? FirstName
    {
        get => _firstName;
        set => Set(ref _firstName, value);
    }

    private string? _lastName;
    public string? LastName
    {
        get => _lastName;
        set => Set(ref _lastName, value);
    }

    private void Set<T>(ref T? field, T? value, [CallerMemberName] string propertyName = "")
    {
        if (EqualityComparer<T>.Default.Equals(field, value))
        {
            // No event should be fired due to the value being the same
        }
        else
        {
            field = value;

            // Call handlers only if they are present
            PropertyChanged?.Invoke(this, new PropertyChangedEventArgs(propertyName));
        }
    }
}
```

ReactiveUI provides us the same capabilities through the abstract `ReactiveObject` to be used as a base class and extension methods where `RaiseAndSetIfChanged` is for setting a property, while  `WhenAnyValue` and friends shall be used for subscriptions. Looks quite simple and user-friendly, but how is it implemented?

The base class has almost nothing interesting inside it. There are just a few fields with lazy initializers. You can see all of them in [the original code](https://github.com/reactiveui/ReactiveUI/blob/d75f999be903001bbf3988d8172f93b081ae13af/src/ReactiveUI/ReactiveObject/ReactiveObject.cs), and here's a snipped of the most interesting part of it:

```csharp
ublic class ReactiveObject : IReactiveNotifyPropertyChanged<IReactiveObject>, IHandleObservableErrors, IReactiveObject
{
    private readonly Lazy<Unit> _propertyChangedEventsSubscribed;
    
    public ReactiveObject()
    {
        _propertyChangingEventsSubscribed = new Lazy<Unit>(
                                                           () =>
                                                           {
                                                               this.SubscribePropertyChangingEvents();
                                                               return Unit.Default;
                                                           },
                                                           LazyThreadSafetyMode.PublicationOnly);
    }

    public event PropertyChangedEventHandler? PropertyChanged
    {
        add
        {
            _ = _propertyChangedEventsSubscribed.Value;
            PropertyChangedHandler += value;
        }
        remove => PropertyChangedHandler -= value;
    }

    private event PropertyChangedEventHandler? PropertyChangedHandler;

    void IReactiveObject.RaisePropertyChanged(PropertyChangedEventArgs args) =>
        PropertyChangedHandler?.Invoke(this, args);
}
```

It delegates the actual implementation of property subscription to the `SubscribePropertyChangingEvents` extension method which belongs to the `IReactiveObjectExtensions` static class. The idea behind such implementation is to delay allocations till the moment when someone or something subscribes to `PropertyChanged`.

Meanwhile, the real implementation is done in `IReactiveObjectExtensions` and it allocates a state object for each reactive object keeping them linked together using a `ConditionalWeakTable<TKey, TValue>`. That special type allows linking some data to an object, like `Dictionary<TKey, TValue>`, but without storing a strong reference to the key and therefore preventing it from being garbage collected. And lazy initializers are here too for the same reason as in `ReactiveObject`, to prevent needless and expensive allocations.

```csharp
public static class IReactiveObjectExtensions
{
    private static readonly ConditionalWeakTable<IReactiveObject, IExtensionState<IReactiveObject>> state = new();

    private interface IExtensionState<out TSender>
        where TSender : IReactiveObject
    {
        IObservable<IReactivePropertyChangedEventArgs<TSender>> Changed { get; }

        void SubscribePropertyChangedEvents();

        void RaisePropertyChanged(string propertyName);
    }

    public static void SubscribePropertyChangedEvents<TSender>(this TSender reactiveObject)
        where TSender : IReactiveObject
    {
        var s = state.GetValue(reactiveObject, _ => (IExtensionState<IReactiveObject>)new ExtensionState<TSender>(reactiveObject));

        s.SubscribePropertyChangedEvents();
    }

    private class ExtensionState<TSender> : IExtensionState<TSender>
        where TSender : IReactiveObject
    {
        private readonly TSender _sender;
        private readonly Lazy<(ISubject<IReactivePropertyChangedEventArgs<TSender>> subject, IObservable<IReactivePropertyChangedEventArgs<TSender>> observable)> _changed;
        private readonly Lazy<ISubject<ReactivePropertyChangedEventArgs<TSender>>> _propertyChanged;

        public ExtensionState(TSender sender)
        {
            _sender = sender;
            _changed = CreateLazyDelayableSubjectAndObservable();
            _propertyChanged = CreateLazyDelayableEventSubject<ReactivePropertyChangedEventArgs<TSender>>(_sender.RaisePropertyChanged);
        }

        public IObservable<IReactivePropertyChangedEventArgs<TSender>> Changed => _changed.Value.observable;

        public void SubscribePropertyChangedEvents() => _ = _propertyChanged.Value;

        public void RaisePropertyChanged(string propertyName)
        {
            if (!AreChangeNotificationsEnabled())
            {
                return;
            }

            var changed = new ReactivePropertyChangedEventArgs<TSender>(_sender, propertyName);
            if (_propertyChanged.IsValueCreated)
            {
                // Do not use NotifyObservable because event exceptions shouldn't be put in ThrownExceptions
                _propertyChanged.Value.OnNext(changed);
            }

            if (_changed.IsValueCreated)
            {
                NotifyObservable(_sender, changed, _changed.Value.subject);
            }
        }

        private void NotifyObservable<T>(TSender rxObj, T item, ISubject<T>? subject)
        {
            try
            {
                subject?.OnNext(item);
            }
            catch (Exception ex)
            {
                rxObj.Log().Error(ex, "ReactiveObject Subscriber threw exception");
                if (!_thrownExceptions.IsValueCreated)
                {
                    throw;
                }

                _thrownExceptions.Value.OnNext(ex);
            }
        }
    }
}
```

There, in `IReactiveObjectExtensions`, exists another method, `RaiseAndSetIfChanged. It is used each time when one needs to assign a new value to some property and raise an event if it's changed. 

```csharp
public static class IReactiveObjectExtensions
{
    public static TRet RaiseAndSetIfChanged<TObj, TRet>(
        this TObj reactiveObject,
        ref TRet backingField,
        TRet newValue,
        [CallerMemberName] string? propertyName = null)
        where TObj : IReactiveObject
    {
        if (propertyName is null)
        {
            throw new ArgumentNullException(nameof(propertyName));
        }

        if (EqualityComparer<TRet>.Default.Equals(backingField, newValue))
        {
            return newValue;
        }

        reactiveObject.RaisingPropertyChanging(propertyName);
        backingField = newValue;
        reactiveObject.RaisingPropertyChanged(propertyName);
        return newValue;
    }

    internal static void RaisingPropertyChanged<TSender>(this TSender reactiveObject, string propertyName)
        where TSender : IReactiveObject
    {
        if (propertyName is null)
        {
            throw new ArgumentNullException(nameof(propertyName));
        }

        var s = state.GetValue(reactiveObject, _ => (IExtensionState<IReactiveObject>)new ExtensionState<TSender>(reactiveObject));

        s.RaisePropertyChanged(propertyName);
    }
}
```

The method looks exactly as our naive implemntation of `INotifyPropertyChanged` except that it has no access to the backing storage of the `PropertyChanged` event, and therefore, the extension method invokes `RaisingPropertyChanged` on the passed reactive object. `RaisingPropertyChanged` in turn fetches the corresponding state from the shared global table and notifies all the subscribers of the object.

That's non-obvious, but there's a strong reason why it's so. It reduces the amount of memory occupied by features that won't be used. It still takes some space because `Lazy<T>` instances and delegates have to be allocated anyway, but that takes less than the whole feature state. Then one can subscribe to an event or a property observable to listen for changes that are going to happen or already happened, four features in total.

### Avalonia UI's approach

Avalonia UI. It started as an attempt to make a cross-platform implementation of WPF with similar APIs but grew into something more powerful with its XPF SDK allowing running WPF applications anywhere Avalonia can go. 

The first difference lies in the base classes, `AvaloniaObject` and `AvaloniaProperty`. While both of them resemble `DependencyObject` and `DependencyProperty` Avalonia implements it slightly differently by allowing property values to be stored directly in fields in addition to property bags used by styled and attached properties. That's done for performance-sensitive properties using the `DirectProperty<T>` type and not for styling at all. As an example, `Bounds` in the `Visual` type is a direct property:

```csharp
public partial class Visual : StyledElement
{
    public static readonly DirectProperty<Visual, Rect> BoundsProperty =
        AvaloniaProperty.RegisterDirect<Visual, Rect>(nameof(Bounds), o => o.Bounds);

    public Rect Bounds
    {
        get { return _bounds; }
        protected set { SetAndRaise(BoundsProperty, ref _bounds, value); }
    }
}
```

Yes, that's exactly what you have seen before! The same set and raise pattern is also used in Avalonia for direct properties. Moreover, even notification changes happen through the `PropertyChanged` event if someone subscribes to it. Otherwise, a better notification change source should be used instead via subscribing to the `Change` observer of the interesting property. Note, that this observer should be used for event listening for all living objects. In cases when a specific instance is required, the `GetPropertyChangedObservable` extension method should be used instead. A change notification contains such information as the sender, original and new values, priority, and the kind of change, effective or not. When only values are interesting `GetObservable` is here with its overloads.

```csharp
public class AvaloniaObject : IAvaloniaObjectDebug, INotifyPropertyChanged
{
    private PropertyChangedEventHandler? _inpcChanged;
    private EventHandler<AvaloniaPropertyChangedEventArgs>? _propertyChanged;

    public event EventHandler<AvaloniaPropertyChangedEventArgs>? PropertyChanged
    {
        add { _propertyChanged += value; }
        remove { _propertyChanged -= value; }
    }

    event PropertyChangedEventHandler? INotifyPropertyChanged.PropertyChanged
    {
        add { _inpcChanged += value; }
        remove { _inpcChanged -= value; }
    }

    internal void RaisePropertyChanged<T>(
        AvaloniaProperty<T> property,
        Optional<T> oldValue,
        BindingValue<T> newValue,
        BindingPriority priority,
        bool isEffectiveValue)
    {
        var e = new AvaloniaPropertyChangedEventArgs<T>(
            this,
            property,
            oldValue,
            newValue,
            priority,
            isEffectiveValue);

        OnPropertyChangedCore(e);

        if (isEffectiveValue)
        {
            property.NotifyChanged(e);
            _propertyChanged?.Invoke(this, e);
            _inpcChanged?.Invoke(this, new PropertyChangedEventArgs(property.Name));
        }
    }
}

public static class AvaloniaObjectExtensions
{
    public static IObservable<T> GetObservable<T>(this AvaloniaObject o, AvaloniaProperty<T> property);

    ublic static IObservable<TResult> GetObservable<TSource, TResult>(this AvaloniaObject o, AvaloniaProperty<TSource> property, Func<TSource, TResult> converter);

    public static IObservable<TResult> GetObservable<TResult>(this AvaloniaObject o, AvaloniaProperty property, Func<object?, TResult> converter);

    public static IObservable<AvaloniaPropertyChangedEventArgs> GetPropertyChangedObservable(
            this AvaloniaObject o,
            AvaloniaProperty property)
}

public abstract class AvaloniaProperty<TValue> : AvaloniaProperty
{
    public new IObservable<AvaloniaPropertyChangedEventArgs<TValue>> Changed { get; }
}
```

Worth mentioning that `AvaloniaObject` hides `INotifyPropertyChanged.PropertyChanged` and exposes its event providing the same information as a property change observable obtained via `GetPropertyChangedObservable` or `AvaloniaProperty<T>.Change`. In contradiction to observables, the event allows receiving all property changes of an instance.

## One more, please

Looking back at all the ways we have seen so far there's a common point. None of them fits well into the .NET model of a property consisting of a getter and a setter method, or at least one of them. There's no place for events at all! One can write them outside of properties with names consisting of the corresponding property name and the `Changed` suffix. If my memory serves me right, WPF can handle that pattern well, but not other frameworks.

Could that be changed? Perhaps, let's give it a try and write our property using the observable pattern.

Previously we have observed a few different approaches to solve the problem and seen the common point between them. Not it's time to turn on our imagination and think about how a property should look taking the notification requirement into account.

```csharp
public struct Property<T>
{
    public T Get();
    public void Set(T value);

    public IObserveable<T> Changed { get; }
}
```

Not yet a complete implementation, but good enough for the start and to evaluate usability.

```csharp
var person = new Person();

person.FirstName.Set("Alan");
person.LastName.Set("Turing");

var first = person.FirstName.Get();
var last = person.LastName.Get();

public class Person
{
    public Property<string?> FirstName { get; }
    public Property<string?> LastName { get; }
}
```

Not so short as .NET properties, but hey, not every language has a concept of properties like, for example, C++ or Rust. Even in .NET properties are made of methods which then invoked when needed. Anything on the top is just syntax sugar. The following snippet demonstrates how `Person` with pure .NET properties is compiled into an IL code.

```csharp
public class Person
{
    public string? FirstName { get; set; }
    public string? LastName { get; set; }
}
```

```il
.class public auto ansi beforefieldinit Person
    extends [System.Runtime]System.Object
{
    method public hidebysig specialname 
        instance string get_FirstName () cil managed;

    .method public hidebysig specialname 
        instance void set_FirstName (
            string 'value'
        ) cil managed;

    .method public hidebysig specialname 
        instance string get_LastName () cil managed;

    .method public hidebysig specialname 
        instance void set_LastName (
            string 'value'
        ) cil managed;
}
```

Therefore, without the property concept, it would require us to call `set_FirstName` to set a new value and `get_FirstName` to get it. Forgetting to use parenthesis for method calls leads to the same error if the `Get` method isn't invoked for a `Property<T>`.

```csharp
// Prints "System.Action`1[System.String]"
Console.WriteLine(new Person().SetName);

public class Person
{
    private string? _name;

    public string? GetName() => _name;
    public void SetName(string? value) => _name = value;
}
```

With our proposal, the issue is almost the same, but it leads to a bit different output.

```csharp
// Prints "Property`1[System.String]"
Console.WriteLine(new Person().Name);

public class Person
{
    public Property<string?> Name { get; }
}
```

There are not so many options left. Either we abandon the attempt and live in comfort with sugar around .NET properties, or take it as an inevitable cost and move forward with direct method calls. I preferred the last and moved to another not less interesting issue which hasn't been touched yet. As you might found, value storage was completely omitted in the examples above. Let's fix it.

Maybe like that?

```csharp
public struct Property<T>
{
    private T _value;

    public T Get() => _value;
    public void Set(T value) { /* set _value and notify observers */ }

    public IObserveable<T> Changed { get; }
}
```

Nope, it will never work as expected because each time a property object is accessed its copy will be allocated on the stack. Then it should be close to how direct properties are implemented in Avalonia in terms of keeping the backing fields inside the object directly.

```csharp
public ref struct Property<T>
{
    private ref T _field;

    public T Get() => _field;
    public void Set(T value) { /* set _field and notify observers */ }

    public IObserveable<T> Changed { get; }
}
```

Closer, but what about "notify observers"? Yeah, an observable object should be stored inside the property owner too. It implies at least one more field, and better not to repeat ourselves and make a base class. Since the number of properties is unknown at compile time and might differ thankfully to type inheritance, property observables will be stored in a list and fetched by relative field addresses.

```csharp
public abstract class ObservableObject
{
    private List<PropertyObservable> _observables = new();

    internal PropertyObservable<T> GetObservable<T>(ref T field)
    {
        var offset = Unsafe.ByteOffset(
            ref Unsafe.As<List<PropertyObservable>, IntPtr>(ref _observables),
            ref Unsafe.As<T, IntPtr>(ref field))
    
        foreach (var observable in _observables)
        {
            if (observable.Offset == offset)
                return observable;
        }

        var result = new PropertyObservable<T> { Offset = offset };

        _observables.Add(result);

        return result;
    }

    protected Property<T> Property<T>(ref T field) => new(this, ref field);
}

public ref struct Property<T>
{
    private readonly ObservableObject _owner;
    private ref T _field;

    internal Property(ObservableObject owner, ref T field)
    {
        _owner = owner;
        _field = ref field;
    }

    public T Get() => _field;

    public void Set(T value)
    {
        if (EqualityComparer<T>.Default.Equals(_field, value))
        {
            _field = value;
        }

        _field = value;
        _owner.GetObservable(ref _field).Changed(value);
    }

    public IObserveable<T> Changed => _owner.GetObservable(ref _field);
}

internal abstract class PropertyObservable
{
    public IntPtr Offset { get; required init; }
}

internal sealed class PropertyObservable<T> : PropertyObservable, IObservable<T>
{
    public void Changed(T value);

    public IDisposable Subscribe(IObserver<T> observer);
}
```

Now the `Person` type should be inherited from `ObservableObject` and instantiate a property when `Name` is accessed:

```csharp
public class Person : ObservableObject
{
    private string? _name;

    public string? Name => Property(ref _name);
}
```

With usage like in the following snippet:

```csharp
var person = new Person();

person.Name.Set("Duffy");

var name = person.Name.Get();
var changes = person.Name.Changed.Subscribe(v => Console.WriteLine(v));
```

Have we forgotten anything? Yes, there are two things still left. First, `ref` types cannot be used at the moment in generics which quite limits use cases of the `Property<T>` type. It can be solved either by making a special delegate type like `SpanAction<T, TArg>` for `Span<T>`, or by switching to the field offset which is less safe, but more flexible in my opinion.

```csharp
public readonly struct Property<T>
{
    private readonly ObservableObject _owner;
    private readonly IntPtr _offset;

    internal Property(ObservableObject owner, IntPtr offset)
    {
        _owner = owner;
        _offset = offset;
    }

    public T Get() => _ownerOwner.Get<T>(_offset);

    public void Set(T value) => _owner.Set(_offset, value);

    public IObservable<T> Changed => _owner.GetObservable<T>(_offset);
}
```

As you might know, to get a field offset in .NET one should use reflection since fields might be reordered by runtime if the type isn't marked as sequential. Reflection isn't cheap and can be stripped out during AOT compilation, so not an option. Maybe there's another trick? Sure, a bit of `Unsafe` from `System.Runtime.CompilerServices`. That type has a pair of methods for address manipulations, `ByteOffset<T>` and `AddByteOffset`. Surely, we need one more to convert an `IntPtr` back to a reference, that's `As<TFrom, TTo>`.

```csharp
private protected IntPtr GetOffsetOf<T>(ref T field)
{
    return Unsafe.ByteOffset(
        ref /* wait, we cannot get an instance address easily */,
        ref Unsafe.As<T, IntPtr>(ref field));
}
```

Oh, yeah, there should be the current instance address to get a filed offset, but really, should it be the same as reflection reports? Maybe it can be relative to another field in the base type like it already happens for `GetObservable<T>`? Yeah, why not?

```csharp
private protected IntPtr GetOffsetOf<T>(ref T field)
{
    return Unsafe.ByteOffset(
        ref GetReference(),
        ref Unsafe.As<T, IntPtr>(ref field));
}

private ref IntPtr GetReference() =>
    ref Unsafe.As<PropertyObservable?, IntPtr>(ref _observables);

private ref T GetReference<T>(IntPtr offset)
{
    ref var baseRef = ref GetReference();
    ref var valueRef = ref Unsafe.AddByteOffset(ref baseRef, offset);
    return ref Unsafe.As<IntPtr, T>(ref valueRef);
}

internal T Get<T>(IntPtr offset) =>
    GetReference<T>(offset);

internal void Set<T>(IntPtr offset, T value)
{
    if (EqualityComparer<T>.Default.Equals(value, Get<T>(offset)))
    {
        return;
    }

    GetReference<T>(offset) = value;
    GetObservable(ref _field).Changed(value);
}
```

Almost ready! Now it's time for the last thing which is benchmarking to measure how efficient is the alternative implementation for properties. I will not write the full code here. It always can be found in [Kinetic's repository]*(https://github.com/YohDeadfall/Kinetic/blob/ea0c970dc3d3e184c7ec6d733f4b378a74f20ca7/benches/Kinetic.Benchmarks/Benchmarks.cs#L21-L111) with the tuned version of `ObservableObject` and `Property<T>`.

```
BenchmarkDotNet v0.13.11, Arch Linux
AMD Ryzen 7 4700U with Radeon Graphics, 1 CPU, 8 logical and 8 physical cores
.NET SDK 8.0.100
  [Host]     : .NET 8.0.0 (8.0.23.53103), X64 RyuJIT AVX2
  DefaultJob : .NET 8.0.0 (8.0.23.53103), X64 RyuJIT AVX2
```

Getters go first as being the most interesting part. If the implementation is noticeably slower than reading a field, it shall not be used in any real application.

| Method         | Mean      | Error     | StdDev    | Median    | Code Size | Allocated |
|--------------- |----------:|----------:|----------:|----------:|----------:|----------:|
| NpcGetter      | 0.0203 ns | 0.0037 ns | 0.0035 ns | 0.0205 ns |       8 B |         - |
| KineticGetter  | 0.0076 ns | 0.0004 ns | 0.0003 ns | 0.0077 ns |      10 B |         - |
| ReactiveGetter | 0.0157 ns | 0.0081 ns | 0.0075 ns | 0.0192 ns |       8 B |         - |

Whoa! Not even measurable and just one instruction longer than getting a field via the most simple getter.

Setters time!

| Method         | WithObserver | WithSameValue | Mean        | Error     | StdDev    | Gen0   | Code Size | Allocated |
|--------------- |------------- |-------------- |------------:|----------:|----------:|-------:|----------:|----------:|
| NpcSeter       | False        | False         |   0.8584 ns | 0.0024 ns | 0.0022 ns |      - |      93 B |         - |
| KineticSetter  | False        | False         |   1.9128 ns | 0.0105 ns | 0.0082 ns |      - |     143 B |         - |
| ReactiveSetter | False        | False         |  77.6664 ns | 1.5666 ns | 1.4654 ns | 0.1147 |     668 B |     240 B |
| NpcSeter       | False        | True          |   0.5605 ns | 0.0014 ns | 0.0012 ns |      - |      93 B |         - |
| KineticSetter  | False        | True          |   0.6122 ns | 0.0381 ns | 0.0356 ns |      - |     107 B |         - |
| ReactiveSetter | False        | True          |   2.3611 ns | 0.0004 ns | 0.0003 ns |      - |     113 B |         - |
| NpcSeter       | True         | False         |   5.2034 ns | 0.0030 ns | 0.0026 ns | 0.0115 |     109 B |      24 B |
| KineticSetter  | True         | False         |   3.2227 ns | 0.0051 ns | 0.0048 ns |      - |     186 B |         - |
| ReactiveSetter | True         | False         | 697.0514 ns | 4.2127 ns | 3.9406 ns | 0.3901 |     668 B |     816 B |
| NpcSeter       | True         | True          |   0.5611 ns | 0.0018 ns | 0.0017 ns |      - |      93 B |         - |
| KineticSetter  | True         | True          |   0.5847 ns | 0.0097 ns | 0.0091 ns |      - |     276 B |         - |
| ReactiveSetter | True         | True          |   1.1853 ns | 0.0061 ns | 0.0057 ns |      - |     113 B |         - |

Zero allocations are what should be expected because there are no events at all. Just pure propagation of a new value to observers. Still, there's a surprise even for me, setting a property through `Property<T>` isn't much slower than for a usual .NET property with `INotifyPropertyChanged` implemented.

And yet, there's one open question left. I started the post saying that Avalonia UI is used as a frontend, but how the heck that can be handled in XAML? Well, it's not the first time I do some magic with custom properties and there's always a way to do that. It's even possible to use befriend WPF with it using custom type descriptors from `System.ComponentModel` though I haven't implemented it yet. In Avalonia it differs, the framework has no idea about the component model. Instead, it has data binding extensions that are registered by adding them to the `BindingPlugins.PropertyAccessors` list. Even `INotifyPropertyChanged` support is done as an extension.

## Summary

As the title states, it's definitely all about properties. While they make code prettier and easier to write by hiding getter and setter invocations, it doesn't come without a cost in case of more complex scenarios like data binding and notifications which came with .NET Framework 2.0. Or maybe it's better to say that the component model doesn't fit properties well, as it should be done differently if there was a time machine. Who knows?

I feel that it doesn't matter, but what matters is that there always should be an alternative solution or point of view. It's even better when none of them are included in the standard library, but very basic blocks to build anything on top of them and not to limit the imagination of developers by some standard implementation written in stone.

## Acknowledgements

Many thanks to [Oskar Dudycz](https://event-driven.io/) and [JeanHeyd Meneide](https://thephd.dev/) who did the initial review of this article and provided invaluable suggestions I hope made it less tiresome and easy to read.
