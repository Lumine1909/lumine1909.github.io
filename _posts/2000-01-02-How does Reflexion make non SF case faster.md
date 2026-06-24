---
layout: post
title: "How does Reflexion make non SF case faster"
date: 2000-01-02 00:00:00 +0000
categories: [General]
---

In my project [Reflexion](https://github.com/Lumine1909/Reflexion), non static final [Method](https://github.com/Lumine1909/Reflexion/blob/master/src/main/java/io/github/lumine1909/reflexion/Method.java) shows lower overhead than [MethodHandle](https://docs.oracle.com/en/java/javase/25/docs/api/index.html) in this specific benchmark:

| Benchmark          | Mode | Cnt | Score  | Error  | Units |
|--------------------|------|-----|--------|--------|-------|
| customCall         | avgt | 20  | 1.317  | ±0.008 | ns/op |
| mhCall             | avgt | 20  | 4.510  | ±0.081 | ns/op |

<details markdown="1">
  <summary>Benchmark code</summary>

```java
package io.github.lumine1909.reflexion;

import org.openjdk.jmh.annotations.Mode;
import org.openjdk.jmh.results.format.ResultFormatType;
import org.openjdk.jmh.runner.Runner;
import org.openjdk.jmh.runner.RunnerException;
import org.openjdk.jmh.runner.options.Options;
import org.openjdk.jmh.runner.options.OptionsBuilder;
import org.openjdk.jmh.runner.options.TimeValue;

import java.lang.invoke.MethodHandle;
import java.lang.reflect.Method;
import java.util.concurrent.TimeUnit;

import static io.github.lumine1909.reflexion.internal.UnsafeUtil.IMPL_LOOKUP;

public class Benchmark {

  static class A {
    private String value = "42";

    public String call(int arg) {
      value = "42";
      return value;
    }
  }

  private static final A OBJECT = new A();

  private static Method reflect$call;

  private static MethodHandle mh$call;

  private static io.github.lumine1909.reflexion.Method<String> custom$call =
    io.github.lumine1909.reflexion.Method.of(A.class, "call", String.class, int.class);

  static {
    try {
      reflect$call = A.class.getDeclaredMethod("call", int.class);
      reflect$call.setAccessible(true);
      mh$call = IMPL_LOOKUP.unreflect(reflect$call);
    } catch (Exception e) {
      throw new ExceptionInInitializerError(e);
    }
  }

  public static void main(String[] args) throws RunnerException {
    new java.io.File("build/reports/jmh").mkdirs();

    Options options = new OptionsBuilder()
      .include(Benchmark.class.getName())
      .forks(1)
      .warmupIterations(10)
      .measurementIterations(20)
      .warmupTime(TimeValue.seconds(1))
      .measurementTime(TimeValue.seconds(1))
      .shouldDoGC(false)
      .threads(1)
      .mode(Mode.AverageTime)
      .timeUnit(TimeUnit.NANOSECONDS)
      .resultFormat(ResultFormatType.JSON)
      .result("build/reports/jmh/results.json")
      .build();

    new Runner(options).run();
  }

  @org.openjdk.jmh.annotations.Benchmark
  public String customCall() {
    return custom$call.invoke(OBJECT, 42);
  }

  @org.openjdk.jmh.annotations.Benchmark
  public String mhCall() throws Throwable {
    return (String) mh$call.invokeExact(OBJECT, 42);
  }
}
```
</details>

This blog is written to explain why this can happen.

Everything start [here](https://github.com/Lumine1909/Reflexion/blob/master/src/main/java/io/github/lumine1909/reflexion/internal/MethodHolder.java)
```java
package io.github.lumine1909.reflexion.internal;

import java.lang.invoke.MethodHandle;
import java.util.concurrent.atomic.AtomicInteger;
import java.util.function.Supplier;

@SuppressWarnings("unchecked")
public final class MethodHolder {

    // Placeholders to exploit java inline
    private static final MethodHandle o0 = null, o1 = null, o2 = null, o3 = null, o4 = null, o5 = null, o6 = null, o7 = null, o8 = null, o9 = null, o10 = null, o11 = null, o12 = null, o13 = null, o14 = null, o15 = null, o16 = null, o17 = null, o18 = null, o19 = null, o20 = null, o21 = null, o22 = null, o23 = null, o24 = null, o25 = null, o26 = null, o27 = null, o28 = null, o29 = null, o30 = null, o31 = null, o32 = null, o33 = null, o34 = null, o35 = null, o36 = null, o37 = null, o38 = null, o39 = null, o40 = null, o41 = null, o42 = null, o43 = null, o44 = null, o45 = null, o46 = null, o47 = null, o48 = null, o49 = null, o50 = null, o51 = null, o52 = null, o53 = null, o54 = null, o55 = null, o56 = null, o57 = null, o58 = null, o59 = null, o60 = null, o61 = null, o62 = null, o63 = null;
    private static final Supplier<MethodHandle>[] OBJ = new Supplier[]{() -> o0, () -> o1, () -> o2, () -> o3, () -> o4, () -> o5, () -> o6, () -> o7, () -> o8, () -> o9, () -> o10, () -> o11, () -> o12, () -> o13, () -> o14, () -> o15, () -> o16, () -> o17, () -> o18, () -> o19, () -> o20, () -> o21, () -> o22, () -> o23, () -> o24, () -> o25, () -> o26, () -> o27, () -> o28, () -> o29, () -> o30, () -> o31, () -> o32, () -> o33, () -> o34, () -> o35, () -> o36, () -> o37, () -> o38, () -> o39, () -> o40, () -> o41, () -> o42, () -> o43, () -> o44, () -> o45, () -> o46, () -> o47, () -> o48, () -> o49, () -> o50, () -> o51, () -> o52, () -> o53, () -> o54, () -> o55, () -> o56, () -> o57, () -> o58, () -> o59, () -> o60, () -> o61, () -> o62, () -> o63};
    private static final AtomicInteger ID = new AtomicInteger();

    public static Supplier<MethodHandle> inline(MethodHandle handle) {
        int index = ID.getAndIncrement();
        try {
            UnsafeUtil.putObject(MethodHolder.class, "o" + index, handle);
            return OBJ[index];
        } catch (Throwable t) {
            return null;
        }
    }
}
```

This creates an inlineable lambda-based access path, and when a MethodHandle field is static final, JIT can often treat it as a stable value after inlining and escape analysis, enabling further optimization that fully eliminate MethodHandle overhead. And when call `inline`, it will replace an unused slot by the given MethodHandle.

Then how does lambda act as an accessor preserve the constantness?

Using `javap -c -p -v MethodHolder.class`, we can find something like this:
```text
  private static java.lang.Object lambda$static$0();
    descriptor: ()Ljava/lang/Object;
    flags: (0x100a) ACC_PRIVATE, ACC_STATIC, ACC_SYNTHETIC
    Code:
      stack=1, locals=0, args_size=0
         0: getstatic     #225                // Field o0:Ljava/lang/invoke/MethodHandle;
         3: areturn
      LineNumberTable:
        line 12: 0
```
Which looks like this in java code
```java
private static /*synthetic*/ Object lambda$static$0() {
    return MethodHolder.o0;
}
```
And by adding VM option `-Djdk.invoke.LambdaMetafactory.dumpProxyClassFiles`, we can find the generated lambda class
```java
package io.github.lumine1909.reflexion.internal;

import java.util.function.Supplier;

// $FF: synthetic class
final class MethodHolder$$Lambda implements Supplier {
    private MethodHolder$$Lambda() {
    }

    public Object get() {
        return MethodHolder.lambda$static$0();
    }
}
```
So when calling `get`, it will use the generated synthetic method to access the static final MethodHandle, so that the invocation becomes more amenable to inlining and JIT optimization.

These codes exploit static final fields combined with trivially inlined lambda accessors. As a result, the wrapper call chain is fully inlined and collapsed, leaving the MethodHandle invocation in a more optimized execution context with reduced surrounding overhead.

That's the reason why in this benchmark, the custom layer outperforms MethodHandle invocation.

Note: this optimization only holds when the call site remains monomorphic and the JVM successfully inlines through the Supplier lambda chain. If the ID space becomes megamorphic or the lambda type becomes unstable, the performance advantage may disappear.
