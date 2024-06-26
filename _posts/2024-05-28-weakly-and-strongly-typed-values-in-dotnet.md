---
layout: post
title: Weakly and strongly typed values in .NET
tags: [C#, .NET, types, optimizations, benchmarks, JIT, assembly, hash codes]
permalink: /weakly-and-strongly-typed-values-in-dotnet
excerpt_separator: <!--more-->
---

Recently I read [an interesting article](https://blog.codingmilitia.com/2024/04/13/primitive-vs-strongly-typed-dictionary-keys-feat-frozen-collections-and-benchmarkdotnet) by [João Antunes](https://antunes.dev) covering how different type of .NET collections handle strongly and weakly typed keys, but it lacks a very important part, an analysis of why some results are almost identical in some cases and not in others. So, let's dive in and find answers to these questions.

<!--more-->

# Preparations

The setup is the same as João used in his work ([the source](https://github.com/joaofbantunes/LooseBenchmarks/tree/main/src/PrimitiveVsStronglyTypedKeyLookup)) except a few things:
 * AMD Ryzen 7 4700U instead of Apple M1 silicone,
 * addition of the `[DisassemblyDiagnoser]` attribute to get the assembly code generated by the runtime,
 * and a change of the target framework to `net9.0` because with `net8.0` it crashes on my machine with `SIGSEGV`.

```csharp
public readonly record struct StronglyTypedKey<T>(T Value);

[RankColumn]
[MemoryDiagnoser]
[DisassemblyDiagnoser]
[GroupBenchmarksBy(BenchmarkLogicalGroupRule.ByParams, BenchmarkLogicalGroupRule.ByCategory)]
public class Benchmark
{
  // ...
}

public class BaseImplementation<T> where T : notnull
{
  private readonly T _toLookup;
  private readonly StronglyTypedKey<T> _toLookupStronglyTyped;
 
  private readonly Dictionary<T, string> _traditional;
  private readonly Dictionary<StronglyTypedKey<T>, string> _traditionalWithStronglyTypedKey;
  private readonly Dictionary<StronglyTypedKey<T>, string> _traditionalWithStronglyTypedKeyWithCustomComparer;

  private readonly FrozenDictionary<T, string> _frozen;
  private readonly FrozenDictionary<StronglyTypedKey<T>, string> _frozenWithStronglyTypedKey;
  private readonly FrozenDictionary<StronglyTypedKey<T>, string> _frozenWithStronglyTypedKeyWithCustomComparer;

  public BaseImplementation(int n, IEqualityComparer<StronglyTypedKey<T>> equalityComparer, Func<int, T> factory)
  {
    var contents = Enumerable.Range(0, n).Select(factory).ToArray();

    _toLookup = contents.Last();
    _toLookupStronglyTyped = new StronglyTypedKey<T>(_toLookup);

    _traditional = contents.ToDictionary(x => x, x => x.ToString()!);
    _traditionalWithStronglyTypedKey = contents.ToDictionary(x => new StronglyTypedKey<T>(x), x => x.ToString()!);
    _traditionalWithStronglyTypedKeyWithCustomComparer = contents.ToDictionary(x => new StronglyTypedKey<T>(x), x => x.ToString()!, equalityComparer);

    _frozen = _traditional.ToFrozenDictionary();
    _frozenWithStronglyTypedKey = _traditionalWithStronglyTypedKey.ToFrozenDictionary();
    _frozenWithStronglyTypedKeyWithCustomComparer = _traditionalWithStronglyTypedKey.ToFrozenDictionary(equalityComparer);
  }

  public string LookupTraditional() =>
    _traditional[_toLookup];

  public string LookupTraditionalWithStronglyTypedKey() =>
    _traditionalWithStronglyTypedKey[_toLookupStronglyTyped];

  public string LookupTraditionalWithStronglyTypedKeyWithCustomComparer() =>
    _traditionalWithStronglyTypedKeyWithCustomComparer[_toLookupStronglyTyped];

  public string LookupFrozen() =>
    _frozen[_toLookup];

  public string LookupFrozenWithStronglyTypedKey() =>
    _frozenWithStronglyTypedKey[_toLookupStronglyTyped];

  public string LookupFrozenWithStronglyTypedKeyWithCustomComparer() =>
    _frozenWithStronglyTypedKeyWithCustomComparer[_toLookupStronglyTyped];
}
```

# Numbers time!

Actually, there are so many numbers that showing all of them is pointless and hell as boring. Therefore, let's limit the area of discussion by excluding the frozen dictionary and focusing on traditional lookups using `Dictionary<TKey, TValue>`.

Still, there are lots of numbers that can be found in the original blog post, I won't repeat them here and will just mention that there are two cases. In the first one the benchmarks for `T` and `StroglyTypedKey<T>` give us very close results, and with a custom comparer it's a bit slower. It happens for `int` and `Guid`. If `float` is used too the picture will be the same. And in the second case, for `string`, lookups over `T` are faster than over `StroglyTypedKey<T>`, and even with a custom comparer over `StroglyTypedKey<T>` are better than without it. The difference between these two cases is the kind of `T` being used, a value type or a reference type.

## Generic keys over a value type

The original article used `int` and `Guid`, and the picture is the same for both. Therefore, we will continue only with one of these types, with `int`. 

| Method                                             | N     | Mean     | Error     | StdDev    | Code Size |
|--------------------------------------------------- |------ |---------:|----------:|----------:|----------:|
| LookupTraditionalInt                               | 1     | 4.742 ns | 0.0091 ns | 0.0085 ns |     445 B |
| LookupTraditionalWithStronglyTypedKeyInt           | 1     | 4.308 ns | 0.0069 ns | 0.0062 ns |     438 B |
|                                                    |       |          |           |           |           |
| LookupTraditionalInt                               | 10    | 4.231 ns | 0.0050 ns | 0.0047 ns |     445 B |
| LookupTraditionalWithStronglyTypedKeyInt           | 10    | 4.436 ns | 0.0058 ns | 0.0054 ns |     438 B |
|                                                    |       |          |           |           |           |
| LookupTraditionalInt                               | 100   | 4.858 ns | 0.0062 ns | 0.0058 ns |     436 B |
| LookupTraditionalWithStronglyTypedKeyInt           | 100   | 4.251 ns | 0.0122 ns | 0.0114 ns |     447 B |
|                                                    |       |          |           |           |           |
| LookupTraditionalInt                               | 1000  | 4.951 ns | 0.0314 ns | 0.0294 ns |     436 B |
| LookupTraditionalWithStronglyTypedKeyInt           | 1000  | 4.317 ns | 0.0440 ns | 0.0411 ns |     447 B |
|                                                    |       |          |           |           |           |
| LookupTraditionalInt                               | 10000 | 4.749 ns | 0.0495 ns | 0.0439 ns |     436 B |
| LookupTraditionalWithStronglyTypedKeyInt           | 10000 | 4.378 ns | 0.0031 ns | 0.0027 ns |     438 B |

So, why there's almost no difference? Well, the earlier addition of `[DisassemblyDiagnoser]` was made to answer that question by comparing the assembly code generated by the runtime.

```diff
-; Benchmark.LookupTraditionalInt()
+; Benchmark.LookupTraditionalWithStronglyTypedKeyInt()
    push   rbp
    push   rbx
    push   rax
    lea   rbp,[rsp+10]
    mov   rdi,[rdi+10]
-   mov   rsi,[rdi+8]
-   mov   ebx,[rdi+38]
+   mov   rsi,[rdi+10]
+   mov   ebx,[rdi+3C]
    cmp   [rsi],sil
    mov   rdi,rsi
    mov   esi,ebx
-   call   qword ptr [7E75ED89F540]; System.Collections.Generic.Dictionary`2[[System.Int32, System.Private.CoreLib],[System.__Canon, System.Private.CoreLib]].FindValue(Int32)
+   call   qword ptr [70E3098942E8]; System.Collections.Generic.Dictionary`2[[StronglyTypedKey`1[[System.Int32, System.Private.CoreLib]], PrimitiveVsStronglyTypedKeyLookup],[System.__Canon, System.Private.CoreLib]].FindValue(StronglyTypedKey`1<Int32>)
    test   rax,rax
    je    short M00_L00
    mov   rax,[rax]
    add   rsp,8
    pop   rbx
    pop   rbp
    ret
 M00_L00:
    mov   edi,ebx
-   call   qword ptr [7E75ED9C6790]
+   call   qword ptr [70E30996FE28]
    int   3
 ; Total bytes of code 57
```

The only difference we can see between these snipped is only the target method names. Nothing interesting to see here.

```diff
-; System.Collections.Generic.Dictionary`2[[System.Int32, System.Private.CoreLib],[System.__Canon, System.Private.CoreLib]].FindValue(Int32)
+; System.Collections.Generic.Dictionary`2[[StronglyTypedKey`1[[System.Int32, System.Private.CoreLib]], PrimitiveVsStronglyTypedKeyLookup],[System.__Canon, System.Private.CoreLib]].FindValue(StronglyTypedKey`1<Int32>)
    push   rbp
    push   r15
    push   r14
    push   r13
    push   r12
    push   rbx
    sub   rsp,18
    lea   rbp,[rsp+40]
    mov   rbx,rdi
    mov   r15d,esi
    mov   rdi,[rbx+8]
    test   rdi,rdi
-   je    near ptr M01_L06
+   je    near ptr M01_L03
    mov   r14,[rbx+18]
    test   r14,r14
-   jne   near ptr M01_L03
+   jne   near ptr M01_L04
    mov   eax,r15d
    mov   esi,eax
    imul   rsi,[rbx+30]
    shr   rsi,20
    inc   rsi
    mov   r11d,[rdi+8]
    mov   ecx,r11d
    imul   rsi,rcx
    shr   rsi,20
    cmp   esi,r11d
    jae   near ptr M01_L11
    mov   esi,esi
    mov   r13d,[rdi+rsi*4+10]
    mov   r12,[rbx+10]
    xor   ecx,ecx
    dec   r13d
    mov   edx,[r12+8]
 M01_L00:
    cmp   edx,r13d
-   jbe   near ptr M01_L06
+   jbe   short M01_L03
    mov   edi,r13d
    lea   rdi,[rdi+rdi*2]
    lea   r13,[r12+rdi*8+10]
    cmp   [r13+8],r15d
-   jne   near ptr M01_L07
-   cmp   [r13+10],eax
-   jne   near ptr M01_L07
+   jne   near ptr M01_L08
+   mov   edi,[r13+10]
+   cmp   edi,eax
+   jne   near ptr M01_L08
 M01_L01:
    cmp   [r13],r13b
    mov   rax,r13
 M01_L02:
    add   rsp,18
    pop   rbx
    pop   r12
    pop   r13
    pop   r14
    pop   r15
    pop   rbp
    ret
 M01_L03:
+   xor   eax,eax
+   jmp   short M01_L02
+M01_L04:
    mov   rdi,r14
    mov   esi,r15d
-   mov   r11,7E75EBE70720
+   mov   r11,70E307E40730
    call   qword ptr [r11]
    mov   r12d,eax
    mov   rax,[rbx+8]
    mov   esi,r12d
    imul   rsi,[rbx+30]
    shr   rsi,20
    inc   rsi
    mov   edi,[rax+8]
    mov   edx,edi
    imul   rsi,rdx
    shr   rsi,20
    cmp   esi,edi
    jae   near ptr M01_L11
    mov   esi,esi
-   mov   esi,[rax+rsi*4+10]
+   mov   eax,[rax+rsi*4+10]
    mov   rbx,[rbx+10]
    xor   r13d,r13d
-   dec   esi
-M01_L04:
+   dec   eax
+M01_L05:
    mov   ecx,[rbx+8]
-   cmp   ecx,esi
-   jbe   short M01_L06
-   mov   esi,esi
-   lea   rax,[rsi+rsi*2]
+   cmp   ecx,eax
+   jbe   short M01_L03
+   mov   eax,eax
+   lea   rax,[rax+rax*2]
    lea   rax,[rbx+rax*8+10]
    cmp   [rax+8],r12d
-   je    short M01_L08
-M01_L05:
-   mov   esi,[rax+0C]
+   je    short M01_L09
+M01_L06:
+   mov   eax,[rax+0C]
    inc   r13d
    cmp   ecx,r13d
-   jb    short M01_L10
-   jmp   short M01_L04
-M01_L06:
-   xor   eax,eax
-   jmp   near ptr M01_L02
+   jae   short M01_L05
 M01_L07:
+   call   qword ptr [70E308F85278]
+   int   3
+M01_L08:
    mov   r13d,[r13+0C]
    inc   ecx
    cmp   edx,ecx
    jae   near ptr M01_L00
-   jmp   short M01_L10
-M01_L08:
+   jmp   short M01_L07
+M01_L09:
    mov   [rbp-2C],ecx
    mov   [rbp-38],rax
    mov   esi,[rax+10]
    mov   rdi,r14
    mov   edx,r15d
-   mov   r11,7E75EBE70728
+   mov   r11,70E307E40738
    call   qword ptr [r11]
    test   eax,eax
-   jne   short M01_L09
+   jne   short M01_L10
    mov   eax,r13d
    mov   ecx,[rbp-2C]
    mov   r13,[rbp-38]
    mov   edx,eax
    mov   rax,r13
    mov   r13d,edx
-   jmp   short M01_L05
-M01_L09:
+   jmp   short M01_L06
+M01_L10:
    mov   r13,[rbp-38]
    jmp   near ptr M01_L01
-M01_L10:
-   call   qword ptr [7E75ECFB5278]
-   int   3
 M01_L11:
    call   CORINFO_HELP_RNGCHKFAIL
    int   3
-; Total bytes of code 388
+; Total bytes of code 381
```

Now there is more diversity one might say, but even here the outputs are almost identical except for a few minor things:
* used registers
* memory addresses of some methods
* location of the `M01_L07`/`M01_L10` branch which seems to be `ThrowInvalidOperationException_ConcurrentOperationsNotSupported`.

The source code of `FindValue` is identical for both cases and it looks like:

```csharp
internal ref TValue FindValue(TKey key)
{
  ref Entry entry = ref Unsafe.NullRef<Entry>();
  if (_buckets != null)
  {
    Debug.Assert(_entries != null, "expected entries to be != null");
    IEqualityComparer<TKey>? comparer = _comparer;
    if (typeof(TKey).IsValueType && // comparer can only be null for value types; enable JIT to eliminate entire if block for ref types
      comparer == null)
    {
      uint hashCode = (uint)key.GetHashCode();
      int i = GetBucket(hashCode);
      Entry[]? entries = _entries;
      uint collisionCount = 0;

      // ValueType: Devirtualize with EqualityComparer<TKey>.Default intrinsic
      i--; // Value in _buckets is 1-based; subtract 1 from i. We do it here so it fuses with the following conditional.
      do
      {
        // Test in if to drop range check for following array access
        if ((uint)i >= (uint)entries.Length)
        {
          goto ReturnNotFound;
        }

        entry = ref entries[i];
        if (entry.hashCode == hashCode && EqualityComparer<TKey>.Default.Equals(entry.key, key))
        {
          goto ReturnFound;
        }

        i = entry.next;

        collisionCount++;
      } while (collisionCount <= (uint)entries.Length);
    }
    else
    {
      // ...
    }
  }

  goto ReturnNotFound;

ReturnFound:
  ref TValue value = ref entry.value;
Return:
  return ref value;
ReturnNotFound:
  value = ref Unsafe.NullRef<TValue>();
  goto Return;
}
```

The runtime generates a separate method implementation for each generic type if at least one of its generic parameters is a value type, our current case, `TKey` is a value type for `int` and `StronglyTypedKey<int>`. Since there's a non-shared implementation the runtime attempts to apply another technique named "devirtualization". It replaces virtual calls with direct invocations if it's exactly determined. In some cases, it even inlines the body of the callee, and that's happened for `EqualityComparer<TKey>.Default.Equals`.

Now it's clear why there's no difference for the observed case without a custom comparer. There is simply no indirect call inside of the lookup method. And when there is a virtual call to `GetHashCode` at the beginning of the lookup, and then for each matching bucket the comparer's `Equals` method is also virtually invoked. It takes some additional time and makes the code a bit slower.

## Generic keys over a reference type

After discussing about shared and non-shared generics, and devirtualization no wonder that in the table below we see a different picture for the `string` benchmarks. All we know is that the `string` type is by-reference even if it has the by-value semantics. The JIT compiler generates an universal implementation for `Dictionary<StronglyTypedKey<string>, string>` that can be used for any `Dictionary<StronglyTypedKey<T>, TValue>` where `T` and `TValue` are references.

| Method                                                        | N     | Mean      | Error     | StdDev    | Code Size |
|-------------------------------------------------------------- |------ |----------:|----------:|----------:|----------:|
| LookupTraditionalString                                       | 1     | 19.302 ns | 0.0387 ns | 0.0362 ns |     401 B |
| LookupTraditionalWithStronglyTypedKeyString                   | 1     | 38.275 ns | 0.0404 ns | 0.0378 ns |     792 B |
| LookupTraditionalWithStronglyTypedKeyStringWithCustomComparer | 1     | 31.926 ns | 0.0499 ns | 0.0467 ns |     614 B |
|                                                               |       |           |           |           |           |
| LookupTraditionalString                                       | 10    | 16.565 ns | 0.0393 ns | 0.0368 ns |     401 B |
| LookupTraditionalWithStronglyTypedKeyString                   | 10    | 37.746 ns | 0.0299 ns | 0.0280 ns |     792 B |
| LookupTraditionalWithStronglyTypedKeyStringWithCustomComparer | 10    | 32.230 ns | 0.3138 ns | 0.2935 ns |     614 B |
|                                                               |       |           |           |           |           |
| LookupTraditionalString                                       | 100   | 17.792 ns | 0.0362 ns | 0.0339 ns |     410 B |
| LookupTraditionalWithStronglyTypedKeyString                   | 100   | 38.961 ns | 0.0492 ns | 0.0460 ns |     792 B |
| LookupTraditionalWithStronglyTypedKeyStringWithCustomComparer | 100   | 31.624 ns | 0.1461 ns | 0.1220 ns |     614 B |
|                                                               |       |           |           |           |           |
| LookupTraditionalString                                       | 1000  | 20.657 ns | 0.1705 ns | 0.1511 ns |     410 B |
| LookupTraditionalWithStronglyTypedKeyString                   | 1000  | 40.340 ns | 0.0433 ns | 0.0405 ns |     792 B |
| LookupTraditionalWithStronglyTypedKeyStringWithCustomComparer | 1000  | 32.797 ns | 0.1520 ns | 0.1422 ns |     614 B |
|                                                               |       |           |           |           |           |
| LookupTraditionalString                                       | 10000 | 18.818 ns | 0.0832 ns | 0.0778 ns |     410 B |
| LookupTraditionalWithStronglyTypedKeyString                   | 10000 | 39.701 ns | 0.0759 ns | 0.0710 ns |     792 B |
| LookupTraditionalWithStronglyTypedKeyStringWithCustomComparer | 10000 | 32.882 ns | 0.0286 ns | 0.0253 ns |     620 B |

In any produced assembly a shared generic type can be spotted by usage of the `__Canon` type instead of the actual generic parameters. That's exactly what we have for the strongly typed generic keys when `T` is `string`:

```nasm
; Benchmark.LookupTraditionalWithStronglyTypedKeyString()
    push   rbp
    push   rbx
    push   rax
    lea   rbp,[rsp+10]
    mov   rdi,[rdi+8]
    mov   rsi,[rdi+18]
    mov   rbx,[rdi+28]
    cmp   [rsi],sil
    mov   rdi,rsi
    mov   rsi,rbx
    call   qword ptr [76939B3677C8]; System.Collections.Generic.Dictionary`2[[StronglyTypedKey`1[[System.__Canon, System.Private.CoreLib]], PrimitiveVsStronglyTypedKeyLookup],[System.__Canon, System.Private.CoreLib]].FindValue(StronglyTypedKey`1<System.__Canon>)
    test   rax,rax
    je    short M00_L00
    mov   rax,[rax]
    add   rsp,8
    pop   rbx
    pop   rbp
    ret
M00_L00:
    mov   rsi,rbx
    mov   rdi,offset MD_System.ThrowHelper.ThrowKeyNotFoundException[[StronglyTypedKey`1[[System.String, System.Private.CoreLib]], PrimitiveVsStronglyTypedKeyLookup]](StronglyTypedKey`1<System.String>)
    call   qword ptr [76939B46FBA0]
    int   3
; Total bytes of code 70
```

Expectedly a shared code might be slower than a non-shared one, but there's a good reason for having sharing in general. Since any reference type is a pointer under the hood to the corresponding object on the heap, most of the code is the same. No need to waste storage and RAM on duplicating pieces when they can be collapsed into a single block. If there's a type dependant branching it's always possible to obtain the type handle from the object or a special register for generic methods. Still, it doesn't answer why a custom comparer boosts the execution.

## Comparision magic

To understand why the usage of a custom comparer outperforms the case when it's not provided, the `Dictionary<TKey, TValue>`'s code should be checked.

```csharp
public Dictionary(int capacity, IEqualityComparer<TKey>? comparer)
{
  if (capacity < 0)
  {
    ThrowHelper.ThrowArgumentOutOfRangeException(ExceptionArgument.capacity);
  }

  if (capacity > 0)
  {
    Initialize(capacity);
  }

  // For reference types, we always want to store a comparer instance, either
  // the one provided, or if one wasn't provided, the default (accessing
  // EqualityComparer<TKey>.Default with shared generics on every dictionary
  // access can add measurable overhead). For value types, if no comparer is
  // provided, or if the default is provided, we'd prefer to use
  // EqualityComparer<TKey>.Default.Equals on every use, enabling the JIT to
  // devirtualize and possibly inline the operation.
  if (!typeof(TKey).IsValueType)
  {
    _comparer = comparer ?? EqualityComparer<TKey>.Default;

    // Special-case EqualityComparer<string>.Default, StringComparer.Ordinal, and StringComparer.OrdinalIgnoreCase.
    // We use a non-randomized comparer for improved perf, falling back to a randomized comparer if the
    // hash buckets become unbalanced.
    if (typeof(TKey) == typeof(string) &&
      NonRandomizedStringEqualityComparer.GetStringComparer(_comparer!) is IEqualityComparer<string> stringComparer)
    {
      _comparer = (IEqualityComparer<TKey>)stringComparer;
    }
  }
  else if (comparer is not null && // first check for null to avoid forcing default comparer instantiation unnecessarily
       comparer != EqualityComparer<TKey>.Default)
  {
    _comparer = comparer;
  }
}
```

Our wrapper is a value type, so the last `if` block applies. And indeed, the `_comparer` is set only when provided to the constructor and if it's not the default one in case of a value type. Therefore, we have no comparer assigned, but something shall be used instead, right? That's exactly what we have seen, `EqualityComparer<TKey>.Default.Equals`:

```csharp
internal ref TValue FindValue(TKey key)
{
  // ...

  ref Entry entry = ref Unsafe.NullRef<Entry>();
  if (_buckets != null)
  {
    Debug.Assert(_entries != null, "expected entries to be != null");
    IEqualityComparer<TKey>? comparer = _comparer;
    if (typeof(TKey).IsValueType && // comparer can only be null for value types; enable JIT to eliminate entire if block for ref types
      comparer == null)
    {
      // ..
      do
      {
        // ..
        // Resolution of the default equality comparer 👇
        if (entry.hashCode == hashCode && EqualityComparer<TKey>.Default.Equals(entry.key, key))
        {
          goto ReturnFound;
        }

        // ..
      } while (collisionCount <= (uint)entries.Length);
    }
    else
    {
      Debug.Assert(comparer is not null);
      // ..
      do
      {
        // ..
        // Just a virtual call, nothing else 👇
        if (entry.hashCode == hashCode && comparer.Equals(entry.key, key))
        {
          goto ReturnFound;
        }

        // ..
      } while (collisionCount <= (uint)entries.Length);
    }
  // ..
}
```

Yeah, babe, the default key comparer is resolved each time when there's a matching hash code even if it's a collision. It's done that way for a very good reason we already discussed, devirtualization. The JIT compiler can only spot a possible optimization site by the full match, i.e. exactly `EqualityComparer<T>.Default.Equals`. Otherwise, if the comparer is stored in a variable in the same method and used later, the optimization cannot be applied. It even works in our current case:

```nasm
; System.Collections.Generic.Dictionary`2[[StronglyTypedKey`1[[System.__Canon, System.Private.CoreLib]], PrimitiveVsStronglyTypedKeyLookup],[System.__Canon, System.Private.CoreLib]].FindValue(StronglyTypedKey`1<System.__Canon>)
    ; ...
M01_L02:
    mov   rdi,rsi
    call   qword ptr [76939B36CD08]; System.Collections.Generic.EqualityComparer`1[[StronglyTypedKey`1[[System.__Canon, System.Private.CoreLib]], PrimitiveVsStronglyTypedKeyLookup]].get_Default()
    mov   rdi,rax
    mov   rcx,[r14+10]
    mov   rax,offset MT_System.Collections.Generic.GenericEqualityComparer`1[[StronglyTypedKey`1[[System.String, System.Private.CoreLib]], PrimitiveVsStronglyTypedKeyLookup]]
    cmp   [rdi],rax
    jne   near ptr M01_L17
    ; Inlined body of "StronglyTypedKey<T>.Equals" which contains inlined "string.Equals"
    mov   rsi,[rbp-38]
    test   rcx,rcx
    je    near ptr M01_L16
    test   rsi,rsi
    je    near ptr M01_L15
    cmp   rcx,rsi
    jne   near ptr M01_L14
    mov   r8d,1
M01_L03:
    test   r8d,r8d
    je    short M01_L12
    ; ...
M01_L14:
    mov   edx,[rcx+8]
    cmp   edx,[rsi+8]
    jne   short M01_L13
    lea   rdi,[rcx+0C]
    add   rsi,0C
    mov   edx,[rcx+8]
    add   edx,edx
    call   qword ptr [76939A724B58]; System.SpanHelpers.SequenceEqual(Byte ByRef, Byte ByRef, UIntPtr)
    mov   r8d,eax
    jmp   near ptr M01_L03
    ; ...
```

It fetches the default comparer first and then checks if it's a cached instance of `GenericEqualityComparer<StronglyTypedKey<string>>`. If so, then it uses the devirtualized equality method of `StronglyTypedKey<string>` which in turn is a simple string equality inlined too.

The problem here is that for shared generics the default comparer has to be retrieved each time and it isn't cheap. Perhaps, if there would be a method similar to `RuntimeHelpers.IsReferenceOrContainsReferences<T>`, but which would tell us if `T` is a shared generic or not, the dictionary's constructor can be changed in such a way that for shared generics it will set the comparer like for reference types.

## It's not over yet!

Do you think that the troubles came to an end? No, there's one more reason for the strongly typed key over `string` benchmark to be slow. Earlier I showed you the dictionary's constructor which has a specially crafted logic when the key is a type of `string`.

```csharp
public Dictionary(int capacity, IEqualityComparer<TKey>? comparer)
{
  // ..

  if (!typeof(TKey).IsValueType)
  {
    _comparer = comparer ?? EqualityComparer<TKey>.Default;

    // Special-case EqualityComparer<string>.Default, StringComparer.Ordinal, and StringComparer.OrdinalIgnoreCase.
    // We use a non-randomized comparer for improved perf, falling back to a randomized comparer if the
    // hash buckets become unbalanced.
    if (typeof(TKey) == typeof(string) &&
      NonRandomizedStringEqualityComparer.GetStringComparer(_comparer!) is IEqualityComparer<string> stringComparer)
    {
      _comparer = (IEqualityComparer<TKey>)stringComparer;
    }

    //..
  }
}
```

And here it is, `NonRandomizedStringEqualityComparer`. a special equality comparer which instead of `GetHashCode` invocations on a string calls `GetNonRandomizedHashCode`. Yeah, I know, it feels very suspicious. While one might expect that `GetHashCode` on a string produces a stable value only depending on its content, it's not the case. It also uses a different algorithm, Marvin hash, and a random seed generated only once every time the application runs. The whole point of that thing is the prevention of hash flooding attacks, and since it's about huge amounts of data the protection doesn't apply for small collections when collision probabilities are low and equality operations aren't expensive, i.e. till the collection reaches some threshold. After that, the .NET switches to a randomized equality comparer and rehashes all entries in the collection.

```csharp
private bool TryInsert(TKey key, TValue value, InsertionBehavior behavior)
{
  // ...
  // Value types never rehash
  if (!typeof(TKey).IsValueType && collisionCount > HashHelpers.HashCollisionThreshold && comparer is NonRandomizedStringEqualityComparer)
  {
    // If we hit the collision threshold we'll need to switch to the comparer which uses randomized string hashing
    // i.e. EqualityComparer<string>.Default.
    Resize(entries.Length, true);
  }
  // ...
}

private void Resize(int newSize, bool forceNewHashCodes)
{
  // Value types never rehash
  Debug.Assert(!forceNewHashCodes || !typeof(TKey).IsValueType);
  Debug.Assert(_entries != null, "_entries should be non-null");
  Debug.Assert(newSize >= _entries.Length);

  Entry[] entries = new Entry[newSize];

  int count = _count;
  Array.Copy(_entries, entries, count);

  if (!typeof(TKey).IsValueType && forceNewHashCodes)
  {
    Debug.Assert(_comparer is NonRandomizedStringEqualityComparer);
    IEqualityComparer<TKey> comparer = _comparer = (IEqualityComparer<TKey>)((NonRandomizedStringEqualityComparer)_comparer).GetRandomizedEqualityComparer();

    for (int i = 0; i < count; i++)
    {
      if (entries[i].next >= -1)
      {
        entries[i].hashCode = (uint)comparer.GetHashCode(entries[i].key);
      }
    }
  }
  // ..
}
``` 

# A positive note

Hopefully, you won't use `StrongTypedKey<T>` in your code and will create a new wrapper for each specific case, so the compiler will protect you from messing up with unrelated values. Really, `StronglyTypeKey<T>` adds nothing to `T` except hiding its operators and methods, you still can assign a `StronglyTypeKey<T>` instance to a logically unrelated local or a wrong parameter. What you should do is write down a completely new type for each case like, let's say, a serial number:

```csharp
public readonly record struct SerialNumber(string Value);
```

Now you cannot accidentally use it for anything else than a serial number, but how is it compared to `string` in performance? A simple benchmark for rescue:

```csharp
private Dictionary<SerialNumber, string> _serialNumbers = null!;
private SerialNumber _serialNumberToLookup;

private Dictionary<string, string> _strings = null!;
private string _stringToLookup = null!;

[GlobalSetup]
public void Setup()
{
  var data = Enumerable.Range(0, N).Select(x => x.ToString());

  _serialNumbers = data.ToDictionary(x => new SerialNumber(x));
  _serialNumberToLookup = _serialNumbers.Last().Key;

  _strings = data.ToDictionary(x => x);
  _stringToLookup = _strings.Last().Key;
}

[Params(1, 10, 100, 1_000, 10_000)]
public int N { get; set; }

[Benchmark]
public string LookupSerialNumber() =>
  _serialNumbers[_serialNumberToLookup];

[Benchmark]
public string LookupString() =>
  _strings[_stringToLookup];
```

| Method             | N     | Mean      | Error     | StdDev    | Code Size |
|------------------- |------ |----------:|----------:|----------:|----------:|
| LookupSerialNumber | 1     |  7.001 ns | 0.0127 ns | 0.0119 ns |     576 B |
| LookupString       | 1     | 11.311 ns | 0.1537 ns | 0.1437 ns |     406 B |
| LookupSerialNumber | 10    |  6.943 ns | 0.0154 ns | 0.0128 ns |     576 B |
| LookupString       | 10    | 10.798 ns | 0.0971 ns | 0.0811 ns |     406 B |
| LookupSerialNumber | 100   |  7.610 ns | 0.0100 ns | 0.0093 ns |     576 B |
| LookupString       | 100   | 11.104 ns | 0.2189 ns | 0.1941 ns |     406 B |
| LookupSerialNumber | 1000  |  7.791 ns | 0.0771 ns | 0.0721 ns |     576 B |
| LookupString       | 1000  | 11.834 ns | 0.0832 ns | 0.0695 ns |     397 B |
| LookupSerialNumber | 10000 |  9.538 ns | 0.0773 ns | 0.0686 ns |     576 B |
| LookupString       | 10000 | 11.463 ns | 0.0966 ns | 0.0856 ns |     406 B |

Surprisingly, usage of `SerialNumber` as a key is a bit faster than with `string`, hardly noticeable, but it goes number one. The credit here goes to the JIT compiler which not only used non-shared generics but also devirtualized the string comparison and hash code computation. Now, my readers, you can breathe out and continue using strongly typed identifiers or values without any hesitation.

# Outro

Hope you enjoyed the reading and learned something new about type safety, generics, optimizations, and not less importantly making representative tests. For further reading on the touched topics here's a list of recommendations:
* [CoreCLR Design: Guarded Devirtualization](https://github.com/dotnet/runtime/blob/main/docs/design/coreclr/jit/GuardedDevirtualization.md)
* [Book of the Runtime: Shared Generics Design](https://github.com/dotnet/runtime/blob/main/docs/design/coreclr/botr/shared-generics.md)
* [Why is string.GetHashCode() different each time I run my program in .NET Core?](https://andrewlock.net/why-is-string-gethashcode-different-each-time-i-run-my-program-in-net-core/)
