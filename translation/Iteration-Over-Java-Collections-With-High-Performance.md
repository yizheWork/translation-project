# Iteration Over Java Collections With High Performance

###### Learn more about the forEach loop in Java and how it compares to C style and Stream API in this article on dealing with collections in Java.

#### Introduction
Java developers usually deal with collections such as ArrayList and HashSet. Java 8 came with lambda and the streaming API that helps us to easily work with collections. In most cases, we work with a few thousands of items and performance isn't a concern. But, in some extreme situations, when we have to travel over a few millions of items several times, performance will become a pain.

I use JMH for checking the running time of each code snippet.

#### forEach vs. C Style vs. Stream API
Iteration is a basic feature. All programming languages have simple syntax to allow programmers to run through collections. Stream API can iterate over Collections in a very straightforward manner.

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

    public List<Integer> forEach(BenchMarkState state){
      List<Integer> result = new ArrayList<>(state.testData.size());
      for(Integer item : state.testData){
        result.add(item);
      }
      return result;
    }


C style is more verbose, but still very compact:

    public List<Integer> forCStyle(BenchMarkState state){
      int size = state.testData.size();
      List<Integer> result = new ArrayList<>(size);
      for(int j = 0; j < size; j ++){
        result.add(state.testData.get(j));
      }
      return result;
    }


Then, the performance:

    Benchmark                               Mode  Cnt   Score   Error  Units
    TestLoopPerformance.forCStyle           avgt  200  18.068 ± 0.074  ms/op
    TestLoopPerformance.forEach             avgt  200  30.566 ± 0.165  ms/op
    TestLoopPerformance.streamMultiThread   avgt  200  79.433 ± 0.747  ms/op
    TestLoopPerformance.streamSingleThread  avgt  200  37.779 ± 0.485  ms/op


With C style, JVM simply increases an integer, then reads the value directly from memory. This makes it very fast. But forEach is very different, according to this answer on StackOverFlow and document from Oracle, JVM has to convert forEach to an iterator and call hasNext() with every item. This is why forEach is slower than the C style.

#### Which Is the High-Performance Way to Travelling Over Set?
We define test data:

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

    public List<Integer> forEach(BenchMarkState state){
        List<Integer> result = new ArrayList<>(state.testData.size());
        for(Integer item : state.testData) {
            result.add(item);
        }
        return result;
    }


This is a nice idea, but it doesn't work because initializing the new ArrayList also consumes resources.

    Benchmark                                   Mode  Cnt  Score   Error  Units
    TestLoopPerformance.forCStyle               avgt  200  6.013 ± 0.108  ms/op
    TestLoopPerformance.forCStyleWithIteration  avgt  200  4.281 ± 0.049  ms/op
    TestLoopPerformance.forEach                 avgt  200  4.498 ± 0.026  ms/op
    HashMap (HashSet uses HashMap<E,Object>) isn't designed for iterating all items. The fastest way to iterate over HashMap is a combination of Iterator and the C style for loop, because JVM doesn't have to call hasNext().

#### Conclusion
Foreach and Stream API are convenient to work with Collections. You can write code faster. But, when your system is stable and performance is a major concern, you should think about rewriting your loop.