# 3 Things Every Java Developer Should Stop Doing

# 每个 Java 程序员都不要再做的三件事

### Looking for a few bad habits to drop? Let's take a look at null, functional programming, and getters and setters to see how you can improve your coding.
### 想改正一些坏习惯？让我们一起研究一下 null、函数式编程、getter 和 setter，来看看你能怎么提高自己的编程水平

From returning null values to overusing getters and setters, there are idioms that we as Java developers are accustom to making, even when unwarranted. While they may be appropriate in some occasions, they are usually forces of habit or fallbacks that we make to get the system working. In this article, we will go through three things that are common among Java developers, both novice and advanced, and explore how they can get us into trouble. It should be noted that these are not hard-and-fast rules that should *always* be adhered to, regardless of the circumstances. At times, there may be a good reason to use these patterns to solve a problem, but on the whole, they should be used a lot less than they are now. To start us off, we will begin with one of the most prolific, but double-edged keywords in Java: Null.
从返回 null 值到滥用 getter 和 setter，我们作为 Java 程序员，对于犯很多愚蠢的错误，已经习惯了，即使很多都是不必要的【即使它们没有什么根据】。虽然它们在某些情况下是适用的，但是它们经常都是习惯使然，或者是为了让系统工作起来而做的让步。本文将会介绍 3 种新手和老手都常犯的错误，然后看看它们怎么带给我们麻烦的。需要注意的是，这些规则并不是铁律，你不需要在任何情况下都遵守它们。有些情况，可能有很好的理由去使用这些模式去解决问题，然而整体上，它们都应该比目前使用的更少。我们从 Java 里最常见的双刃剑开始：Null。


## 1. Returning Null
## 1. 返回 Null

Null has been the best friend and worst enemy of developers for decades and null in Java is no different. In high-performance applications, null can be a solid means of reducing the number of objects and signaling that a method does not have a value to return. In contrast to throwing an exception, which has to capture the entire stack trace when it is created, null serves as a quick and low-overhead way to signal clients that no value can be obtained.
数十年来，null 一直是程序员最好的朋友和最坏的敌人。Java 中的 null 也不例外。在高性能的应用中，null 是降低对象数量、表示一个方法没有返回值的一种可靠的方法。相对于需要捕获整个异常栈的异常来说，null 作为一种迅速、低开销的方式，来告诉调用方，取不到任何返回值。

Outside the context of high-performance systems, null can wreak havoc in an application by creating more tedious checks for null return values and causing `NullPointerException`s when dereferencing a null object. In most applications, nulls are returned for three primary reasons: (1) to denote no elements could be found for a list, (2) to signal that no valid value could be found, even if an error did not occur, or (3) to denote a special case return value.
抛开高性能的系统除外，null 可以对应用造成很大的破坏，创建很多对返回的 null 值的检查，当引用一个 null 对象时引发的 `NullPointerException`。大部分应用里，返回 null 值主要有 3 个原因：（1）代指列表中找不到某个对象，（2）代指没有找到有效的值，即使并没有发生异常，（3）代指一种特殊的返回值。

Barring any performance reasons, each of these cases has a much better solution which does not use null and force developers to handle null cases. Whatsmore, clients of these methods are not left scratching their heads wondering if the method will return a null in some edge case. In each case, we will devise a cleaner approach that does not involve returning a null value.
抛开任何性能的理由，这 3 中情况都有更好的办法，它们不使用 null，不强迫程序员解决 null 的情况。而且，这些方法的调用方也不用挠头去考虑它们究竟是否会在某些边界情况返回 null。每种情况，我们都找到了一个更清晰的方法，不必返回 null。

### No Elements
### 没有值
When returning lists or other collections, it can be common to see a null collection returned in order to signal that elements for that collection could not be found. For example, we could create a service that manages users in a database that resembles the following (some method and class definitions have been left out for brevity):
当返回 list 或者其他的集合时，返回一个 null 来表示没有找到集合中的元素，是很常见的。比如，我们创建一个服务，管理数据库中的用户，类似于以下的代码（简便起见，一些方法和类的定义被忽略了）

```java
public class UserService {

    public List<User> getUsers() {

        User[] usersFromDb = getUsersFromDatabase();

        if (usersFromDb == null) {
            // 没有在数据库中找到用户
            return null;
        }
        else {
            return Arrays.asList(usersFromDb);
        }
    }
}

UserServer service = new UserService();
List<Users> users = service.getUsers();

if (users != null) {
    for (User user: users) {
        System.out.println("User found: " + user.getName());
    }
}
```

Since we have elected to return a null value in the case of no users, we are forcing our client to handle this case before iterating over the list of users. If instead, we returned an empty list to denote no users were found, the client can remove the null check entirely and loop through the users as normal. If there are no users, the loop will be skipped implicitly without having to manually handle that case; in essence, looping through the list of users functions as we intend for both an empty and populated list without having to manually handle one case or the other:
由于我们在没有用户时使用了 null 作为返回值，我们在强迫调用方，在遍历这个用户的列表钱，处理这种情况。如果反过来，我们返回一个空的 list 代表没有找到用户，调用发可以去掉 null 的检查，直接按正常的情况遍历整个列表。如果没有用户，循环会直接跳过，而不用手工去处理这种情况；其实，遍历整个用户的列表，我们就是为了同时解决空列表和有值的列表，而不用再手工处理某一种情况。

```java
public class UserService {

    public List<User> getUsers() {
        User[] usersFromDb = getUsersFromDatabase();

        if (usersFromDb == null) {
            // 没有在数据库中找到用户
            return Collections.emptyList();
        }
        else {
            return Arrays.asList(usersFromDb);
        }
    }
}

UserServer service = new UserService();
List<Users> users = service.getUsers();

for (User user: users) {
    System.out.println("User found: " + user.getName());
}
```

In the case above, we have elected to return an immutable, empty list. This is an acceptable solution, so long as we document that the list is immutable and should not be modified (doing so may throw an exception). If the list must be mutable, we can return an empty, mutable list, as in the following example:
上面的例子中，我们使用了不可修改的空列表。这种方案是可接受的，只要我们在文档里说明，返回的 list 是不可修改的而且不应该做任何修改（强行修改会抛异常）。如果返回的 list 必须允许修改，我们可以返回一个空的可以修改的 list，像下面的例子里那样：

```java
public List<User> getUsers() {

    User[] usersFromDb = getUsersFromDatabase();

    if (usersFromDb == null) {
        // 没有在数据库中找到用户
        return new ArrayList<>();    // 可修改的 list
    }
    else {
        return Arrays.asList(usersFromDb);
    }
}
```

In general, the following rule should be adhered to when signaling that no elements could be found:
整体上，当表明没有发现任何元素时，应该遵守以下规则：

> Return an empty collection (or list, set, queue, etc.) to denote that no elements can be found
> 通过返回一个空的集合（list、set、queue 等等）来表明找不到任何元素

Doing so not only reduces the special-case handling that clients must perform, but it also reduces the inconsistencies in our interface (i.e. we return a list object sometimes and not others).
这么做不仅仅减少了调用方需要处理的特殊情况，它也降低了我们接口的不一致性（比如，我们在某些情况下返回一个 list 对象，其他情况却不）。

### Optional Value
### 可选的值

Many times, null values are returned when we wish to inform a client that an optional value is not present, but no error has occurred. For example, getting a parameter from a web address. In some cases, the parameter may be present, but in other cases, it may not. The lack of this parameter does not necessarily denote an error, but rather, it denotes that the user did not want the functionality that is included when the parameter is provided (such as sorting). We can handle this by returning null if no parameter is present or the value of the parameter if one is supplied (some methods have been removed for brevity):
很多情况下，返回 null 值来表明我们没有找到任何可选的值，但是也没有发生任何异常。比如，从一个网络地址里获取一个参数。某些情况下，这个参数可能出现。但是其它情况，却找不到这个参数。缺少这个参数不一定表明一个异常，而是表明用户不想要这个参数提供的功能（比如排序）。我们可以通过返回 null 值表示没有找到参数，如果找到了就返回这个参数（简洁起见，某些方法被忽略了）：

```java
public class UserListUrl {

    private final String url;

    public UserListUrl(String url) {
        this.url = url;
    }

    public String getSortingValue() {

        if (urlContainsSortParameter(url)) {
            return extractSortParameter(url);
        }
        else {
            return null;
        }
    }
}

UserService userService = new UserService();
UserListUrl url = new UserListUrl("http://localhost/api/v2/users");
String sortingParam = url.getSortingValue();

if (sortingParam != null) {
    UserSorter sorter = UserSorter.fromParameter(sortingParam);
    return userService.getUsers(sorter);
}
else {
    return userService.getUsers();
}
```

When no parameter is supplied, a null is returned and a client must handle this case, but nowhere in the signature of the `getSortingValue` method does it state that the sorting value is optional. For us to know that this method is optional and may return a null if no parameter is present, we would have to read the documentation associated with the method (if any were provided).
如果没有提供任何参数，返回了一个 null 值，调用方必须处理这种情况，但是 `getSortingValue` 方法的签名中并没有说明返回值是可选的。如果我们需要知道返回值是可选的，在没有找到任何参数下返回 null，我们得去找这个方法的文档（如果提供了的话）。

Instead, we can make the optionality explicit returning an `Optional` object. As we will see, the client still has to handle the case when no parameter is present, but now that requirement is made explicit. Whatsmore, the `Optional` class provides more mechanisms to handle a missing parameter than a simple null check. For example, we can simply check for the presence of the parameter using the query method (a state-testing method) provided by `Optional`:
相反，我们可以通过返回 `Optional` 对象明确表示返回值的可选择性。正如我们将看到的，调用方仍然需要处理没有找到参数的情况，但是这种要求是很明确的。而且，`Optional` 对象提供了比简单检查 null 更多的机制来处理找不到参数的情况。比如，我们可以通过  `Optional`  提供的查询方法（一个状态检查函数）来检查参数是否存在。

```java
public class UserListUrl {

    private final String url;

    public UserListUrl(String url) {
        this.url = url;
    }
    
    public Optional<String> getSortingValue() {

        if (urlContainsSortParameter(url)) {
            return Optional.of(extractSortParameter(url));
        }
        else {
            return Optional.empty();
        }
    }
}

UserService userService = new UserService();
UserListUrl url = new UserListUrl("http://localhost/api/v2/users");
Optional<String> sortingParam = url.getSortingValue();

if (sortingParam.isPresent()) {
    UserSorter sorter = UserSorter.fromParameter(sortingParam.get());
    return userService.getUsers(sorter);
}
else {
    return userService.getUsers();
}
```

This is nearly identical to that of the null-check case, but we have made the optionality of the parameter explicit (i.e. the client cannot access the parameter without calling `get()`, which will throw a `NoSuchElementException` if the optional is empty). If we were not interested in returning the list of users based on the optional parameter in the web address, but rather, consuming the parameter in some manner, we could use the `ifPresentOrElse` method to do so:
它和检查 null 的那种方法基本一样，但是我们已经指明了参数的可选择性（比如，调用方只能通过调用 `get()` 方法获取参数，如果 optional 是空的，就会抛出一个 `NoSuchElementException`）。如果我们对网络地址中的参数返回的用户不感兴趣，而是想以某种模式处理参数，我们可以使用 `ifPresentOrElse` 方法：

```java
sortingParam.ifPresentOrElse(
    param -> System.out.println("Parameter is :" + param),
    () -> System.out.println("No parameter supplied.")
);
```

This greatly reduces the noise required for null checking. If we wished to disregard the parameter if no parameter is supplied, we could do so using the `ifPresent` method:
它极大的减少了 null 检查的干扰。如果我们不想考虑没有参数出现的情况，我们可以使用 `ifPresent` 方法：

```java
sortingParam.ifPresent(param -> System.out.println("Parameter is :" + param));
```

In either case, using an `Optional` object, rather than returning null, explicitly forces clients to handle the case that a return value may not be present and provides many more avenues for handling this optional value. Taking this into account, we can devise the following rule:
没用情况，相比如返回 null来说，使用一个 `Optional` 对象都明确的强制调用方处理没有返回值的情况，而且提供了更多的方法来处理这个可选的值。考虑这种情况，我们可以提出下面的规则：

> If a return value is optional, ensure clients handle this case by returning an `Optional`  that contains a value if one is found and is empty if no value can be found
> 如果返回值是可选的，通过提供一个 `Optional` 对象，如果找到返回了就包含一个返回值，如果找不到就是空的，可以保证调用方处理这种情况。

### Special-Case Value
### 特殊情况的返回值

The last common use case is that of a special case, where a normal value cannot be obtained and a client should handle a corner case different than the others. For example, suppose we have a command factory from which clients periodically request commands to complete. If no command is ready to be completed, the client should wait 1 second before asking again. We can accomplish this by returning a null command, which clients must handle, as illustrated in the example below (some methods are not shown for brevity):
最后一种常用的情况就是特殊值，获取不到正常的值，而调用方需要处理和其他正常值不同的边界情况。比如，假设我们有一个命令的工厂，调用方定期获取准备好的指令。如果没有准备好的指令，调用方应该在重新发起请求前等待 1 秒。我们可以通过返回 null 值表示这种情况，调用方必须处理这个 null 值，如下例子所示（简洁起见，没有展示某些方法）：

```java
public interface Command {
    public void execute();
}

public class ReadCommand implements Command {

    @Override
    public void execute() {
        System.out.println("Read");
    }
}

public class WriteCommand implements Command {

    @Override
    public void execute() {
        System.out.println("Write");
    }
}

public class CommandFactory {

    public Command getCommand() {
        if (shouldRead()) {
            return new ReadCommand();
        }
        else if (shouldWrite()) {
            return new WriteCommand();
        }
        else {
            return null;
        }
    }
}

CommandFactory factory = new CommandFactory();
while (true) {
    Command command = factory.getCommand();

    if (command != null) {
        command.execute();
    }
    else {
        Thread.sleep(1000);
    }
}
```

Since the `CommandFactory` can return null commands, clients are obligated to check if the command received is null and if it is, sleep for 1 second. This creates a set of conditional logic that clients must handle on their own. We can reduce this overhead by creating a [null-object](https://en.wikipedia.org/wiki/Null_object_pattern) (sometimes called a [special-case object](https://martinfowler.com/eaaCatalog/specialCase.html)). A null-object encapsulates the logic that would have been executed in the null scenario (namely, sleeping for 1 second) into an object that is returned in the null case. For our command example, this means creating a `SleepCommand` that sleeps when executed:
由于 `CommandFactory` 会返回 null 的指令，调用方必须检查收到的指令是不是 null 的，如果是，就 sleep 1 秒。这引起了一个可能得逻辑集合，调用方必须自己处理。我们可以通过创建一个 [空对象（null-object）](https://en.wikipedia.org/wiki/Null_object_pattern) (也叫 [特殊情况对象（special-case object）](https://martinfowler.com/eaaCatalog/specialCase.html)) 减少这个开销。一个空对象把 null 情况下的解决方案（也就是，sleep 1 秒钟）包装到一个返回的对象里。在我们这个指令的例子里，这一位置创建一个 `SleepCommand` 的指令，当执行时就 sleep：


```java
public class SleepCommand implements Command {

    @Override
    public void execute() {
        Thread.sleep(1000);
    }
}

public class CommandFactory {

    public Command getCommand() {
        if (shouldRead()) {
            return new ReadCommand();
        }
        else if (shouldWrite()) {
            return new WriteCommand();
        }
        else {
            return new SleepCommand();
        }
    }
}

CommandFactory factory = new CommandFactory();
while (true) {
    Command command = factory.getCommand();
    command.execute();
}
```

As with the case of returning empty collections, creating a null-object allows clients to implicitly handle special cases as if they were the normal case. This is not always possible, though; there may be instances where the decision for dealing with a special case must be made by the client. This can be handled by allowing the client to supply a default value, as is done with the `Optional` class. In the case of `Optional`, clients can obtain the contained value or a default using the `orElse` method:

返回空列表的情况，创建一个空对象允许调用方默认得把特殊情况当做正常情况处理。尽管不总是适用，有些情况如果处理特殊情况可以由调用方自己来决定。可以像 `Optional` 里做的那样，让调用方提供一个默认值。在 `Optional` 里，调用方可以获取返回值或者使用 `orElse` 方法获取默认值：

```java
UserListUrl url = new UserListUrl("http://localhost/api/v2/users");
Optional<String> sortingParam = url.getSortingValue();
String sort = sortingParam.orElse("ASC");
```



If there is a supplied sorting parameter (i.e. if the `Optional` contains a value), this value will be returned. If no value exists, `"ASC"` will be returned by default. The `Optional` class also allows a client to create a default value when needed, in case the default creation process is expensive (i.e. the default will be created only when needed):

如果提供了一个排序的参数（比如，`Optional` 包含一个返回值），这个返回值会被返回。如果没有返回值，`"ASC"` 默认会被返回。`Optional` 也允许调用方需要时再创建默认值，防止默认值的创建需要耗费很多资源（比如，默认值只有在需要得时候才创建）：

```java
UserListUrl url = new UserListUrl("http://localhost/api/v2/users");
Optional<String> sortingParam = url.getSortingValue();
String sort = sortingParam.orElseGet(() -> {
    // 耗费资源较多的计算
});
```



Using a combination of null-objects and default values, we can devise the following rule:

结合使用空对象和默认值，我们可以得出以下规则：

> When possible, handle null cases with a null-object or allow clients to supply a default value
>
> 如有可能，使用空对象来处理 null 的情况，或者允许调用方提供一个默认值。

## 2. Defaulting to Functional Programming

Since streams and lambdas were introduced in Java Development Kit (JDK) 8, there has been a push to migrate towards functional programming, and rightly so. Before lambdas and streams, performing simple functional tasks were cumbersome and resulted in severely unreadable code. For example, filtering a collection in the traditional style resulted in code that resembled the following:



```java
public class Foo {

    private final int value;

    public Foo(int value) {
        this.value = value;
    }

    public int getValue() {
        return value;
    }
}

Iterator<Foo> iterator = foos.iterator();
while(iterator.hasNext()) {
    if (iterator.next().getValue() > 10) {
        iterator.remove();
    }
}
```





While this code is compact, it does not tell us in an obvious way that we are trying to remove elements of a collection if some criterion is satisfied. Instead, it tells us that we are iterating over a collection while there are more elements in the collection and removing each element if its value is greater than 10 (we can surmise that filtering is occurring, but it obscured in the verbosity of the code). We can shrink this logic down to one statement using functional programming:



```java
foos.removeIf(foo -> foo.getValue() > 10);
```





Not only is this statement much more concise than its iterative alternative, it also tells us exactly what it is trying to do. We can even make it more readable if we name the predicate and pass it to the `removeIf` method:



```java
Predicate<Foo> valueGreaterThan10 = foo -> foo.getValue() > 10;
foos.removeIf(valueGreaterThan10);
```





The final line of this snippet reads like a sentence in English, informing us *exactly* of what the statement is doing. With code that looks so compact and readable, it is tempting to try and use functional programming in *every* situation where iteration is required, but this is a naive philosophy. Not every situation lends itself to functional programming. For example, if we tried to print the cross product of the set of suits and ranks in a deck of cards (every combination of suits and ranks), we could create the following (see [Effective Java, 3rd Edition](https://www.amazon.com/Effective-Java-3rd-Joshua-Bloch/dp/0134685997) for a more detailed listing of this example):



```java
public static enum Suit {
    CLUB, DIAMOND, HEART, SPADE;
}

public static enum Rank {
    ONE, TWO, THREE, FOUR, FIVE, SIX, SEVEN, EIGHT, NINE, TEN, JACK, QUEEN, KING;
}

Collection<Suit> suits = EnumSet.allOf(Suit.class);
Collection<Rank> ranks = EnumSet.allOf(Rank.class);

suits.stream()
    .forEach(suit -> {
        ranks.stream().forEach(rank -> System.out.println("Suit: " + suit + ", rank: " + rank));
    });
```





While this is not over-complicated to read, it is not the most straightforward implementation we could devise. It is pretty clear that we are trying to force streams into a realm where traditional iteration is much more favorable. If we used traditional iteration, we could have simplified the cross product of suits and ranks to the following:



```java
for (Suit suit: suits) {
    for (Rank rank: ranks) {
        System.out.println("Suit: " + suit + ", rank: " + rank);
    }
}
```





This style, although much less flashy, is much more straightforward. We can quickly see that we are attempting to iterate over each suit and rank and pair each rank with each suit. The tediousness of functional programming becomes much more acute the larger the stream expression becomes. Take for example the following code snippet created by Joshua Bloch in [Effective Java, 3rd Edition](https://www.amazon.com/Effective-Java-3rd-Joshua-Bloch/dp/0134685997) (pp. 205, Item 45) to find all the anagrams over a specified length contained in a dictionary at the path supplied by the user:



```java
public class Anagrams {
    public static void main(String[] args) throws IOException {
        Path dictionary = Paths.get(args[0]);
        int minGroupSize = Integer.parseInt(args[1]);

        try (Stream<String> words = Files.lines(dictionary)) {
            words.collect(
                groupingBy(word -> word.chars().sorted()
                           .collect(StringBuilder::new, 
                               (sb, c) -> sb.append((char) c),
                               StringBuilder::append).toString()))
                .values().stream()
                    .filter(group -> group.size() >= minGroupSize)
                    .map(group -> group.size() + ": " + group)
                    .forEach(System.out::println);
        }
    }
}
```





Even the most seasoned stream adherents would probably balk at this implementation. It is unclear as to the intention of the code and would take a decent amount of thinking to uncover what the above stream manipulations are trying to accomplish. This does not mean that streams are complicated or that they are too wordy, but they are not *always* the best choice. As we saw above, using the `removeIf` reduced a complicated group of statements into a single, easily-comprehensible statement. Therefore, we should not try to replace *every* instance of traditional iteration with streams or even lambdas. Instead, we should abide by the following rule when deciding whether to functional programming or use the traditional route:

> Functional programming and traditional iteration both have their benefits and disadvantages: Use whichever results in the simplest and most readable code

Although it may be tempting to use the flashiest, most up-to-date features of Java in every possible scenario, this is not always the best route. Sometimes, the old-school features work best.

## 3. Creating Indiscriminate Getters and Setters

One of the first things that novice programmers are taught is to encapsulate the data associated with a class in private fields and expose them through public methods. In practice, this results in creating getters to access the private data of a class and setters to modify the private data of a class:



```java
public class Foo {

    private int value;

    public void setValue(int value) {
        this.value = value;
    }

    public int getValue() {
        return value;
    }
}
```





While this is a great practice for newer programmers to learn, it is not a practice that should go unrefined into intermediate or advanced programming. What normally occurs in practice is that every private field is given a pair of getters and setters, exposing the internals of the class to external entities. This can cause some serious issues, especially if the private fields are mutable. This is not only a problem with setters but even when only a getter is present. Take for example the following class, which exposes its only field using a getter:



```java
public class Bar {

    private Foo foo;

    public Bar(Foo foo) {
        this.foo = foo;
    }

    public Foo getFoo() {
        return foo;
    }
}
```





This exposure may seem innocuous since we have wisely restricted removed a setter method, but it is far from it. Suppose that another class accesses an object of type `Bar` and changes the underlying value of `Foo` without the `Bar` object knowing:



```java
Foo foo = new Foo();
Bar bar = new Bar(foo);

// Another place in the code
bar.getFoo().setValue(-1);
```





In this case, we have changed the underlying value of the `Foo` object without informing the `Bar` object. This can cause some serious problems if the value that we provided the `Foo` object breaks an invariant of the `Bar` object. For example, if we had an invariant that stated the value of `Foo` could not be negative, then the above snippet silently breaks this invariant without notifying the `Bar` object. When the `Bar` object goes to use the value of its `Foo` object, things may go south very quickly, especially if the `Bar` object *assumed* that the invariant held since it did not expose a setter to directly reassign the `Foo` object it held. This can even cause failure to a system if data is severely altered, as in the following example of an array being inadvertently exposed:



```java
public class ArrayReader {

    private String[] array;

    public String[] getArray() {
        return array;
    }

    public void setArray(String[] array) {
        this.array = array;
    }

    public void read() {
        for (String e: array) {
            System.out.println(e);
        }
    }
}

public class Reader {

    private ArrayReader arrayReader;

    public Reader(ArrayReader arrayReader) {
        this.arrayReader = arrayReader;
    }

    public ArrayReader getArrayReader() {
        return arrayReader;
    }

    public void read() {
        arrayReader.read();
    }
}

ArrayReader arrayReader = new ArrayReader();
arrayReader.setArray(new String[] {"hello", "world"});
Reader reader = new Reader(arrayReader);
reader.getArrayReader().setArray(null);
reader.read();
```





Executing this code would cause a `NullPointerException` because the array associated with the `ArrayReader` object is null when it tries to iterate over the array. What is disturbing about this `NullPointerException` is that it can occur long after the change to the `ArrayReader` was made and maybe even in an entirely different context (such as in a different part of the code or maybe even in a different thread), making the task of tracking down the problem very difficult.

The astute reader may also notice that we could have made the private `ArrayReader` field `final` since we did not expose a way to reassign it after it has been set through the constructor. Although it might seem that this would make the `ArrayReader` constant, ensuring that the `ArrayReader` object we return cannot be changed, this is not the case. Instead, adding `final` to a field only ensures that the field itself is not reassigned (i.e. we cannot create a setter for that field). It does not stop the state of the object itself from being changed. If we tried to add `final` to the getter method, this is futile as well, since `final` modifier on a method only means that the method cannot be overridden by subclasses.

We can even go one step further and [defensively copy](http://www.javacreed.com/what-is-defensive-copying/) the `ArrayReader` object in the constructor of `Reader`, ensuring that the object that was passed into the object cannot be tampered with after it has been supplied to the `Reader` object. For example, the following cannot happen:



```java
ArrayReader arrayReader = new ArrayReader();
arrayReader.setArray(new String[] {"hello", "world"});
Reader reader = new Reader(arrayReader);
arrayReader.setArray(null);    // Change arrayReader after supplying it to Reader
reader.read();    // NullPointerException thrown
```





Even with these three changes (the `final` modifier on the field, the `final` modifier on the getter, and the defensive copy of the `ArrayReader` supplied to the constructor), we still have not solved the problem. The problem is not found in *how* we are exposing the underlying data of our class, but in the fact that we are doing it in the first place. For us to solve this issue, we have to stop exposing the internal data of our class and instead provide a method to change the underlying data, while still adhering to the class invariants. The following code solves this problem, while at the same time introducing a defensive copy of the supplied `ArrayReader` and marking the `ArrayReader` field final, as should be the case since there is no setter:



```java
public class ArrayReader {
    public static ArrayReader copy(ArrayReader other) {
        ArrayReader copy = new ArrayReader();
        String[] originalArray = other.getArray();
        copy.setArray(Arrays.copyOf(originalArray, originalArray.length));
        return copy;
    }

    // ... Existing class ...
}

public class Reader {

    private final ArrayReader arrayReader;

    public Reader(ArrayReader arrayReader) {
        this.arrayReader = ArrayReader.copy(arrayReader);
    }

    public ArrayReader setArrayReaderArray(String[] array) {
        arrayReader.setArray(Objects.requireNonNull(array));
    }

    public void read() {
        arrayReader.read();
    }
}

ArrayReader arrayReader = new ArrayReader();
arrayReader.setArray(new String[] {"hello", "world"});
Reader reader = new Reader(arrayReader);
reader.read();

Reader flawedReader = new Reader(arrayReader);
flawedReader.setArrayReaderArray(null);    // NullPointerException thrown
```





If we look at the flawed reader, a `NullPointerException` is still thrown, but it is thrown immediately when the invariant (that a non-null array is used when reading) is broken, not at some later time. This ensures that the invariant fails-fast, which makes debugging and finding the root of the problem much easier. 

We can take this principle one step further and state that it is a good idea to make the fields of a class completely inaccessible if there is no pressing need to allow for the state of a class to be changed. For example, we could make the `Reader` class fully encapsulated by removing any methods that modify its state after creation:



```java
public class Reader {

    private final ArrayReader arrayReader;

    public Reader(ArrayReader arrayReader) {
        this.arrayReader = ArrayReader.copy(arrayReader);
    }

    public void read() {
        arrayReader.read();
    }
}

ArrayReader arrayReader = new ArrayReader();
arrayReader.setArray(new String[] {"hello", "world"});
Reader reader = new Reader(arrayReader);
// No changes can be made to the Reader after instantiation
reader.read();
```





Taking this concept to its logical conclusion, it is a good idea to make a class immutable if it is possible. Thus, the state of the object never changes after the object has been instantiated. For example, we can create an immutable `Car` object as follows:



```java
public class Car {

    private final String make;
    private final String model;

    public Car(String make, String model) {
        this.make = make;
        this.model = model;
    }

    public String getMake() {
        return make;
    }

    public String getModel() {
        return model;
    }
}
```





It is important to note that if the fields of the class are non-primitive, a client can modify the underlying object as we saw above. Thus, immutable objects should return defensive copies of these objects, disallowing clients to modify the internal state of the immutable object. Note, though, that defensive copying can reduce performance since a new object is created each time the getter is called. This issue should not be prematurely optimized (disregarding immutability for the promise of possible performance increases), but it should be noted. The following snippet provides an example of defensive copying for method return values:



```java
public class Transmission {

    private String type;

    public static Transmission copy(Transmission other) {
        Transmission copy = new Transmission();
        copy.setType(other.getType);
        return copy;
    }

    public String setType(String type) {
        this.type = type;
    }

    public String getType() {
        return type;
    }
}

public class Car {

    private final String make;
    private final String model;
    private final Transmission transmission;

    public Car(String make, String model, Transmission transmission) {
        this.make = make;
        this.model = model;
        this.transmission = Transmission.copy(transmission);
    }

    public String getMake() {
        return make;
    }

    public String getModel() {
        return model;
    }

    public Transmission getTransmission() {
        return Transmission.copy(transmission);
    }
}
```





This leaves us with the following principle:

> Make classes immutable, unless there is a pressing need to change the state of a class. All fields of an immutable class should be marked as private and final to ensure that no reassignments are performed on the fields and no indirect access should be provided to the internal state of the fields

Immutability also brings with it some very important advantages, such as the ability of the class to be easily used in a multi-threaded context (i.e. two threads can share the object without fear that one thread will alter the state of the object while the other thread is accessing that state). In general, there are many more instances that we can create immutable classes than we realize at first: Many times, we add getters or setters out of habit.

## Conclusion

Many of the applications we create end up working, but in a large number of them, we introduce sneaky problems that tend to creep up at the worst possible times. In some of those cases, we do things out of convenience, or even out of habit, and pay little mind to whether these idioms are practical (or safe) in the context we use them. In this article, we delved into three of the most common of these practices, such null return values, affinity for functional programming, and careless getters and setters, along with some pragmatic alternatives. While the rules in this article should not be taken as absolute, they do provide some insight into the uncommon dangers of common practices and may help in fending off laborious errors in the future.
