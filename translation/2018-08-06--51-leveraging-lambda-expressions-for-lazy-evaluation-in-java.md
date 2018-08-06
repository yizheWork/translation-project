---
editor: yizheWork
published: false
title: '#51-Leveraging-Lambda-Expressions-for-Lazy-Evaluation-in-Java'
---
## Leveraging Lambda Expressions for Lazy Evaluation in Java

In Java, the potential of lazy evaluation is quite neglected (actually, at the language level, it's pretty much limited to the implementation of minimal evaluation). Advanced languages, like Scala, for example, differentiate between call-by-value and call-by-name calls or introduce dedicated keywords like lazy.

Although Java 8 brought an improvement on that field by providing the implementation of the lazy sequence concept (we all know it as java.util.stream.Stream), today, we'll skip that and focus on how the introduction of lambda expressions brought us a new lightweight method of leveraging delayed evaluation.

Lambda-Backed Lazy Evaluation
Scala
Whenever we want to lazily evaluate method parameters in Scala, we can do it the "call-by-name" way.

Let's create a simple foo  method that accepts a String  instance and returns it:

def foo(b: String): String = b


Everything is eager, just like in Java Now, if we wanted to make b computed lazily, we could leverage the call-by-name syntax and simply add two signs to the b 's type declaration, and voilà:

def foo(b: => String): String = b


If we try to javap reverse-engineer the generated  *.class file , we'd see:

Compiled from "LazyFoo.scala"
public final class LazyFoo {
    public static java.lang.String foo(scala.Function0<java.lang.String>);
    Code:   
    0: getstatic #17 // Field LazyFoo$.MODULE$:LLazyFoo$;
    3: aload_0
    4: invokevirtual #19 // Method LazyFoo$.foo:(Lscala/Function0;)Ljava/lang/String;
    7: areturn
}


It turns out that the parameter passed to our method is no longer a String , but a Function0<String> instead, which makes it possible to evaluate the expression lazily — as long as we don't call it, the computation isn't triggered. It is as simple as that.

In Java
Now, whenever we need to lazily evaluate a computation returning T, we can reuse the above idea and simply wrap the computation into a Java Function0 equivalent to the Supplier<T>  instance:

Integer v1 = 42; // eager
Supplier<Integer> v2 = () -> 42; // lazy


That would prove to be much more practical if we needed to obtain a result of some long-lasting method instead:

Integer v1 = compute(); //eager
Supplier<Integer> value = () -> compute(); // lazy


And, again, this time as a method param:

private static int computeLazily(Supplier<Integer> value) {
    // ...
}


If you look closely at the APIs introduced in Java 8, you will notice that this pattern is used quite often. One of the most evident examples would be Optional#orElseGet, which is a lazy equivalent of Optional#orElse.

If not this pattern, it'd make the  Optional 's usefulness...  questionable. Naturally, we're not limited to suppliers only. We can reuse all functional interfaces in a similar manner.

Thread-Safety and Memoization
Unfortunately, that simple approach is flawed. The computation will be triggered for every function call. This applies not only to multithreaded environments but also when a Supplier is called consecutive times by the same thread. And, it's fine as long as we're aware of the fact and applying the technique reasonably.

Lazy Evaluation with Memoization
As mentioned already, the lambda-based approach can be perceived as flawed in certain contexts because of the fact that the value never gets memorized. In order to fix that, we'd need to construct a dedicated tool, let's say Lazy:

public class Lazy<T> { ... }


That one would need to hold both a Supplier<T> and the value T itself:

@RequiredArgsConstructor
public class NaiveLazy<T> { 
    private final Supplier<T> supplier;
    private T value;
    public T get() {
        if (value == null) {
            value = supplier.get();
         }
        return value;
    }
}


It is as simple as that. Keep in mind that the above implementation serves as a PoC and is not thread-safe, yet.

Luckily, making it thread-safe involves, pretty much, just making sure that multiple threads do not trigger the same computation when trying to obtain the value. This can be easily achieved by utilizing the double-checked locking pattern (we could've simply synchronized on the get()  method, but that would introduce unwanted contention):

@RequiredArgsConstructor
public class Lazy<T> {
    private final Supplier<T> supplier;
    private volatile T value;
    public T get() {
        if (value == null) {
            synchronized (this) {
                if (value == null) {
                    value = supplier.get();
                }
            }
        }
       return value;
    }
}


And, now, we have a fully functional implementation of the lazy evaluation pattern in Java. Since it's not implemented at the language level, additional costs associated with creating a new object need to be paid.

Further Development
Of course, we don't need to stop here, and can further improve the tool itself. For example, by adding a lazy filter()/flatMap()/map() methods that would enable a more fluent interaction and composability:

public <R> Lazy<R> map(Function<T, R> mapper) {
    return new Lazy<>(() -> mapper.apply(this.get()));
}
public <R> Lazy<R> flatMap(Function<T, Lazy<R>> mapper) {
    return new Lazy<>(() -> mapper.apply(this.get()).get());
}
public Lazy<Optional<T>> filter(Predicate<T> predicate) {
    return new Lazy<>(() -> Optional.of(get()).filter(predicate));
}
Sky's the limit.

Let's also throw in a handy factory method:

public static <T> Lazy<T> of(Supplier<T> supplier) {
    return new Lazy<>(supplier);
}
And in action:

Lazy.of(() -> compute(42))
  .map(s -> compute(13))
  .flatMap(s -> lazyCompute(15))
  .filter(v -> v > 0);
As you can see, as long as #get isn't called at the end of the chain, nothing gets computed.

Nulls
In some situations, null can be a valid value, but it won't work properly with our implementation - a valid null value gets treated just like an uninitialized value, which is not ideal.

The way to go around it would be simply to be explicit about the optional result by... returning it wrapped in an Optional instance.

Other than that, it'd be a good idea to explicitly forbid null values, for example, by doing:

value = Objects.requireNonNull(supplier.get());
GCing an Unused Supplier
As some of you have probably already noticed, after the value gets evaluated, a supplier will never be used again, but it still occupies some resources.

The way to handle that would be to make the Supplier non-final and free it by setting it to a null after our value gets evaluated.

A Complete Example
public class Lazy<T> {
    private transient Supplier<T> supplier;
    private volatile T value;
    public Lazy(Supplier<T> supplier) {
        this.supplier = Objects.requireNonNull(supplier);
    }
    public T get() {
        if (value == null) {
            synchronized (this) {
                if (value == null) {
                    value = Objects.requireNonNull(supplier.get());
                    supplier = null;
                }
            }
        }
        return value;
    }
    public <R> Lazy<R> map(Function<T, R> mapper) {
        return new Lazy<>(() -> mapper.apply(this.get()));
    }
    public <R> Lazy<R> flatMap(Function<T, Lazy<R>> mapper) {
        return new Lazy<>(() -> mapper.apply(this.get()).get());
    }
    public Lazy<Optional<T>> filter(Predicate<T> predicate) {
        return new Lazy<>(() -> Optional.of(get()).filter(predicate));
    }
    public static <T> Lazy<T> of(Supplier<T> supplier) {
        return new Lazy<>(supplier);
    }
}
The above code can be also found on GitHub.
