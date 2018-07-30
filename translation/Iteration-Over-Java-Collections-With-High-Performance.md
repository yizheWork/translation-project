---
editor: yizheWork
published: false
title: Iteration Over Java Collections With High Performance
---
# Iteration Over Java Collections With High Performance
# 高效遍历 Java 容器

###### Learn more about the forEach loop in Java and how it compares to C style and Stream API in this article on dealing with collections in Java.

###### Java 语言中的 for 循环，C 的方式和 Steam API 使用 Java 中的容器时。

#### Introduction
Java developers usually deal with collections such as ArrayList and HashSet. Java 8 came with lambda and the streaming API that helps us to easily work with collections. In most cases, we work with a few thousands of items and performance isn't a concern. But, in some extreme situations, when we have to travel over a few millions of items several times, performance will become a pain.

#### 简介
Java 开发者经常使用容器，比如 ArrayList 和 HashSet。Java 8 迎来了 lambda 语法和流的 API 可以帮助我们更方便的处理容易。最常见的情况是，我们处理上千个对象，效率并不是什么问题。但是，在一些比较极端的场景下，当我们需要处理数次数百万个项目时，性能就会成为瓶颈。

I use JMH for checking the running time of each code snippet.

我使用 JMH 来检查每块代码的运行时间。

#### forEach vs. C Style vs. Stream API
Iteration is a basic feature. All programming languages have simple syntax to allow programmers to run through collections. Stream API can iterate over Collections in a very straightforward manner.

遍历是一个基本的特性。所有的编程语言都有简单的语法，来让程序员可以走完一个容器。Steam API 可以遍历一个容器以一种非常直接的形式。

    public List<Integer> streamSingleThread(BenchMarkState state){
        List<Integer> result = new ArrayList<>(state.testData.size());
        state.testData.stream().forEach(item -> {
            result.add(item);
        });
        return result;
    }
    public List<Integer> streamMultiThread(BenchMarkState state){
        List<Integer> result = new ArrayList<>(state.testData.size());
        state.testData.stream().parallel().forEach(item -> {
            result.add(item);
        });
        return result;
    }


The forEach  loop is just as simple:

for循环很简单：

    public List<Integer> forEach(BenchMarkState state){
      List<Integer> result = new ArrayList<>(state.testData.size());
      for(Integer item : state.testData){
        result.add(item);
      }
      return result;
    }


C style is more verbose, but still very compact:

C 的形式比较复杂，不过依然很直观：

    public List<Integer> forCStyle(BenchMarkState state){
      int size = state.testData.size();
      List<Integer> result = new ArrayList<>(size);
      for(int j = 0; j < size; j ++){
        result.add(state.testData.get(j));
      }
      return result;
    }


Then, the performance:

下面是性能：

    Benchmark                               Mode  Cnt   Score   Error  Units
    TestLoopPerformance.forCStyle           avgt  200  18.068 ± 0.074  ms/op
    TestLoopPerformance.forEach             avgt  200  30.566 ± 0.165  ms/op
    TestLoopPerformance.streamMultiThread   avgt  200  79.433 ± 0.747  ms/op
    TestLoopPerformance.streamSingleThread  avgt  200  37.779 ± 0.485  ms/op


With C style, JVM simply increases an integer, then reads the value directly from memory. This makes it very fast. But forEach is very different, according to this answer on StackOverFlow and document from Oracle, JVM has to convert forEach to an iterator and call hasNext() with every item. This is why forEach is slower than the C style.

使用 C 的形式，JVM 仅仅增加一个数字，然后从内存里读出来。这使得非常迅速。但是 forEach 就非常不同了，根据 StackOverFlow 上面的这篇回答，和 Oracle 的文章， JVM 需要把 forEach 转换成一个 iterator 然后每一个项目都调用一次 hasNext() 方法。这就是为什么 forEach 比 C 的形式要慢的原因。

#### Which Is the High-Performance Way to Travelling Over Set?

#### 哪一个是遍历 Set 最快的方法呢？

We define test data:

我们定义测试数据集：

    @State(Scope.Benchmark)
    public static class BenchMarkState {
        @Setup(Level.Trial)
        public void doSetup() {
            for(int i = 0; i < 500000; i++){
                testData.add(Integer.valueOf(i));
            }
        }
        @TearDown(Level.Trial)
        public void doTearDown() {
            testData = new HashSet<>(500000);
        }
        public Set<Integer> testData = new HashSet<>(500000);
    }


The Java Set also supports Stream API and forEach loop. According to the previous test, if we convert Set to ArrayList, then travel over ArrayList, maybe the performance improve?

Java 集合同样支持 Steam API 和 forEach 循环。根据之前的测试，如果我们把 Set 转换成 ArrayList，然后遍历 ArrayList，或许性能会提高？

    public List<Integer> forCStyle(BenchMarkState state){
        int size = state.testData.size();
        List<Integer> result = new ArrayList<>(size);
        Integer[] temp = (Integer[]) state.testData.toArray(new Integer[size]);
        for(int j = 0; j < size; j ++){
            result.add(temp[j]);
        }
        return result;
    }


How about a combination of the iterator with the C style for loop?

如果把 iterator 和 C 形式的 for 循环结合起来呢？

    public List<Integer> forCStyleWithIteration(BenchMarkState state){
        int size = state.testData.size();
        List<Integer> result = new ArrayList<>(size);
        Iterator<Integer> iteration = state.testData.iterator();
            for(int j = 0; j < size; j ++){
            result.add(iteration.next());
            }
        return result;
    }


Or, what about simple travel?

或者，简单的遍历怎么样？

    public List<Integer> forEach(BenchMarkState state){
        List<Integer> result = new ArrayList<>(state.testData.size());
        for(Integer item : state.testData) {
            result.add(item);
        }
        return result;
    }


This is a nice idea, but it doesn't work because initializing the new ArrayList also consumes resources.

这是一个好的点子，不过他并不奏效，因为初始化一个新的 ArrayList 同样消耗资源。

    Benchmark                                   Mode  Cnt  Score   Error  Units
    TestLoopPerformance.forCStyle               avgt  200  6.013 ± 0.108  ms/op
    TestLoopPerformance.forCStyleWithIteration  avgt  200  4.281 ± 0.049  ms/op
    TestLoopPerformance.forEach                 avgt  200  4.498 ± 0.026  ms/op
    
HashMap (HashSet uses HashMap<E,Object>) isn't designed for iterating all items. The fastest way to iterate over
HashMap is a combination of Iterator and the C style for loop, because JVM doesn't have to call hasNext().

HashMap (使用 HashMap<E,Object> 的 HashSet) 不是设计用来遍历所有项目的。最快的方法来遍历一个 HashMap 是把 Iterator 和 C 形式的 for 循环结合起来，因为 JVM 不会去调用 hasNext()。

#### Conclusion
#### 结论

Foreach and Stream API are convenient to work with Collections. You can write code faster. But, when your system is stable and performance is a major concern, you should think about rewriting your loop.

Foreach 和 Stea API 用来处理集合是很方便的。你可以更快的写代码。不过，如果你的系统很稳定，效率是一个主要的瓶颈，你应该考虑重写一下你的循环。
