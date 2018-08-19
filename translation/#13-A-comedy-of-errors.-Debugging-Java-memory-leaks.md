# 13-A-comedy-of-errors-Debugging-Java-memory-leaks

We all make errors, but some errors seem so ridiculous we wonder how anyone, let alone we ourselves, could have done such a thing. This is, of course, easy to notice only after the fact. Below, I describe a series of such errors which we recently made in one of our applications. What makes it interesting is that initial symptoms indicated a completely different kind of problem than the one actually present.

人人都会犯错误，但是一些错误看起来是如此可笑，让我们怀疑除了我们自己，还会有谁会犯这种错误。当然，只有事后才能发现。以下，我会描述一系列此等错误，我们在我们自己的应用上犯过的。让他变得有趣的是最开始的极限揭示了实际发生的完全不同的问题。

## Once upon a midnight dreary
## 在一个凄凉的夜晚


I was woken up shortly after midnight by an alert from our monitoring system. Adventory, an application responsible for indexing ads in our [PPC (pay-per-click) advertising system](https://ads.allegro.pl/) had apparently restarted several times in a row. In a cloud environment, a restart of one single instance is a normal event and does not trigger any alerts, but this time the threshold had been exceeded by multiple instances restarting within a short period. I switched on my laptop and dived into the application’s logs.
我是半夜听到我们监控系统发出的报警就起来了的。Adventory，我们的 [PPC （每点一次付一次钱）广告系统](https://ads.allegro.pl/)中一个负责索引广告的系统，显然连续重启了好几次。在一个云端的环境里，实例的重启是很正常的，不会触发任何报警，但是这次示例重启的次数已经在短时间内超过了阈值。我打开笔记本电脑，一头扎进项目的日志里。

## It must be the network
## 一定是网络的问题

I saw several timeouts as the service attempted connecting to [ZooKeeper](https://zookeeper.apache.org/). We use ZooKeeper (ZK) to coordinate indexing between multiple instances and rely on it to be robust. Clearly, a Zookeeper failure would prevent indexing from succeeding, but it shouldn’t cause the whole app to die. Still, this was such a rare situation (the first time I ever saw ZK go down in production) that I thought maybe we had indeed failed to handle this case gracefully. I woke up the on-duty person responsible for ZooKeeper and asked them to check what was going on.
我看到服务尝试连接 [ZooKeeper](https://zookeeper.apache.org/) 时发生了数次超时。我们使用 ZooKeeper（ZK）协调多个实例的索引操作，并依赖它实现鲁棒性。很显然，一次 Zookeeper 失败会停止索引的运行，不过它不应该导致整个系统挂掉。而且，这种情况非常罕见（我第一次遇到 ZK 在生产环境挂掉），我想或许我们确实不能简单的解决这个问题了。我把负责 ZooKeeper 的值班人员叫醒了，让他们看看发生了什么。

Meanwhile, I checked our configuration and realized that timeouts for ZooKeeper connections were in the multi-second range. Obviously, ZooKeeper was completely dead, and given that other applications were also using it, this meant serious trouble. I sent messages to a few more teams who were apparently not aware of the issue yet.

同时，我检查了我们的配置，意识到 ZooKeeper 连接的超时时间是秒级的。很显然，ZooKeeper 完全挂了，鉴于其他服务也在使用它，这意味着非常严重的问题。我给其他几个团队发了消息，他们显然还不知道这事。

My colleague from ZooKeeper team got back to me, saying that everything looked perfectly normal from his point of view. Since other users seemed unaffected, I slowly realized ZooKeeper was not to blame. Logs clearly showed network timeouts, so I woke up the people responsible for networking.

ZooKeeper 团队的同事回复我了，他说在他看来，系统运行一切正常。由于其他用户看起来没有受到影响，我慢慢意识到不是 ZooKeeper 的问题。日志里明显是网络超时，于是我把负责网络的同事叫醒了。

Networking team checked their metrics but found nothing of interest. While it is possible for a single segment of the network or even a single rack to get cut off from the rest, they checked the particular hosts on which my app instances were running and found no issues. I had checked a few side ideas in the meantime but none worked, and I was at my wit’s end. It was getting really late (or rather early) and, independently from my actions, restarts somehow became less frequent. Since this app only affected the freshness of data but not its availability, together with all involved we decided to let the issue wait until morning.

网络团队检查了他们的监控，没有发现任何异常。由于网络中的一小段或者一个点和其余的节点隔断也是有可能的，他们检查了我的系统实例所在的几台机器，没有发现异常。同时，我也尝试了其他几种思路，不过都行不通，我已经达到能力的极限了。时间已经很晚了（或者说很早了），跟我的尝试没有任何关系的，重启变得不那么频繁了。由于服务仅仅影响了数据的刷新，并不会影响到数据的可用性，所有人，包括我在内，决定放着问题到上午再说。

## It must be garbage collection
## 一定是 GC 的问题

Sometimes it is a good idea to sleep on it and get back to a tough problem with a fresh mind. Nobody understood what was going on and the service behaved in a really magical way. Then it dawned on me. What is the main source of magic in Java applications? Garbage collection of course.
有时候睡一觉，等脑子清醒了再去解决难题是一个好主意。没人知道到底发生了什么，服务表现的非常怪异。突然我想到了什么。Java 服务表现怪异的主要根源是什么？当然是垃圾收集。

Just for cases like this, we keep GC logging on by default. I quickly downloaded the GC log and fired up [Censum](https://www.jclarity.com/censum/). Before my very eyes, a grisly sight opened: full garbage collections happening once every 15 minutes and causing 20-second long [!] stop-the-world pauses. No wonder the connection to ZooKeeper was timing out despite no issues with either ZooKeeper or the network!
就是为了目前这种情况的发生，我们默认把 GC 的日志打开了。我迅速下载了 GC 日志然后打开了 [Censum](https://www.jclarity.com/censum/)。在我仔细查看前，一个吓人的景象呈现了：每15分钟发生一次 full GC，每次 GC 引起长达 20 秒的服务中断。怪不得连接 ZooKeeper 超时了，即使 ZooKeeper 和 网络都没有问题。

These pauses also explained why the whole application kept dying rather than just timing out and logging an error. Our apps run inside [Marathon](https://mesosphere.github.io/marathon/), which regularly polls a healthcheck endpoint of each instance and if the endpoint isn’t responding within reasonable time, Marathon restarts that instance.
这些中断也解释了为什么整个服务一直是死掉的，而不是超时之后打一条错误日志。我们的服务运行在 [Marathon](https://mesosphere.github.io/marathon/)，它经常对每个服务的所有节点进行轮询，如果节点在合理的时间内没有响应，Marathon 就重启那个服务。

![20-second GC pauses — certainly not your average GC log](https://allegro.tech/img/articles/2018-02-09-a-comedy-of-errors-debugging-java-memory-leaks/adventory-gc-pause-20-s.png)

Knowing the cause of a problem is half the battle, so I was very confident that the issue would be solved in no time. In order to explain my further reasoning, I have to say a bit more about how Adventory works, for it is not your standard microservice.

知道问题的根源之后，问题就解决一半了，因为我很自信这个问题马上就能解决。为了解释我更深层的疑问，我需要多解释一下 Adventory 是如何工作的，它不是你们那种标准的微服务。

Adventory is used for indexing our ads into [ElasticSearch (ES)](https://www.elastic.co/). There are two sides to this story. One is acquiring the necessary data. To this end, the app receives events sent from several other parts of the system via [Hermes](http://hermes.allegro.tech/). The data is saved to [MongoDB](http://mongodb.org/) collections. The traffic is a few hundred requests per second at most, and each operation is rather lightweight, so even though it certainly causes some memory allocation, it doesn’t require lots of resources. The other side of the story is indexing itself. This process is started periodically (around once every two minutes) and causes data from all the different MongoDB collections to be streamed using [RxJava](https://github.com/ReactiveX/RxJava), combined into denormalized records, and sent to ElasticSearch. This part of the application resembles an offline batch processing job more than a service.

Adventory 是用来把我们的广告索引到 [ElasticSearch (ES)](https://www.elastic.co/)的。这需要两个步骤。第一个是获取所需的数据。到目前为止，这个服务从其他几个系统中接收通过 [Hermes](http://hermes.allegro.tech/) 发来的事件。数据保存到 [MongoDB](http://mongodb.org/) 集群中。数据量大概是每秒最多几百个请求，每个操作都是轻量级的，因此即使他会触发一些内存的回收，这也耗费不了多少资源。第二步就是数据的索引。这个操作定时执行（大概两分钟执行一次），把所有 MongoDB 集群中的数据通过 [RxJava](https://github.com/ReactiveX/RxJava) 收集到一个流里，组合为非规范化的记录，发送给 ElasticSearch。这部分操作类似离线的批处理任务，而不是一个服务。

During each run, the whole index is rebuilt since there are usually so many changes to the data that incremental indexing is not worth the fuss. This means that a whole lot of data has to pass through the system and that a lot of memory allocation takes place, forcing us to use a heap as large as 12 GB despite using streams. Due to the large heap (and to being the one which is currently fully supported), our GC of choice was G1.
每次执行过程中，整个索引都被重建了，因为数据经常有太多的更新，维护索引就没什么必要。这意味着一整块数据都要经过这个系统，引发大量的内存回收，这迫使我们把堆加到 12 GB 这么大，而且使用了 steam。由于如此巨大的堆（作为目前全力支持的），我们的 GC 选择了 G1。

Having previously worked with some applications which allocate a lot of short-lived objects, I increased the size of young generation by increasing both `-XX:G1NewSizePercent` and `-XX:G1MaxNewSizePercent` from their default values so that more data could be handled by the young GC rather than being moved to old generation, as Censum showed a lot of premature tenuring. This was also consistent with the full GC collections taking place after some time. Unfortunately, these settings didn’t help one bit.
由于在我之前处理过的服务中，都会回收大量生命周期很短的对象，我把新生代的大小调大了，通过同时增加 `-XX:G1NewSizePercent` 和 `-XX:G1MaxNewSizePercent` 的默认值，这样 young GC 可以处理更多的数据，而不是送到老年代，正如 Censum 展示的一些早期的阈值。这和一段时间以后发生的 full GC 也是一致的。不幸的是，这些设置都没有起到任何作用。

The next thing I thought was that perhaps the producer generated data too fast for the consumer to consume, thus causing records to be allocated faster than they could be freed. I tried to reduce the speed at which data was produced by the repository by decreasing the size of a thread pool responsible for generating the denormalized records while keeping the size of the consumer data pool which sent them off to ES unchanged. This was a primitive attempt at applying [backpressure](http://reactivex.io/documentation/operators/backpressure.html), but it didn’t help either.
我想到的下一个问题是，或许生产者制造数据太快了，消费者来不及消费，导致这些记录在他们释放前就被回收了。我尝试减小成产数据的线程数量，来降低数据产生的速度，同时保持消费者发送给 ES 的数据池大小不变。这主要是应用 [backpressure](http://reactivex.io/documentation/operators/backpressure.html)，不过它也没有起到作用。

## It must be a memory leak
## 一定是内存泄漏

At this point, a colleague who had kept a cooler head, suggested we do what we should have done in the first place, which is to look at what data we actually had in the heap. We set up a development instance with an amount of data comparable to the one in production and a proportionally scaled heap. By connecting to it with `jvisualvm` and running the memory sampler, we could see the approximate counts and sizes of objects in the heap. A quick look revealed that the number of our domain `Ad` objects was way larger than it should be and kept growing all the time during indexing, up to a number which bore a striking resemblance to the number of ads we were processing. But… this couldn’t be. After all, we were streaming the records using RX exactly for this reason: in order to avoid loading all of the data into memory.

这时，一个当时还保持头脑冷静的同事，建议我们应该做一开始就应该做的事情：查看堆中的数据。我们准备了一个开发环境的实例，拥有和线上相同的数据量，并且堆的大小也大致相同。通过把它连接到`jnisualvm` 运行内存的样本，我们可以看到堆中对象的大致数量和大小。大眼一看，可以发现我们域中`Ad`对象的数量不正常的高，并且在索引的过程中一直在增长，一直增长到我们需要处理的广告的数量的数字级别。但是…，这不应该啊。毕竟，我们实际上是通过 RX 把这些数据整理成流，就是为了把所有的数据都加载到内存里。

![Memory Sampler showed many more Ad objects than we expected](https://allegro.tech/img/articles/2018-02-09-a-comedy-of-errors-debugging-java-memory-leaks/memory-sampler-many-ad-objects.png)

With growing suspicion, I inspected the code, which had been written about two years before and never seriously revisited since. Lo and behold, we were actually loading all data into memory. It was, of course not intended. Not knowing RxJava well enough at that time, we wanted to parallelize the code in a particular way and resolved to using `CompletableFuture` along with a separate executor in order to offload some work from the main RX flow. But then, we had to wait for all the `CompletableFuture`s to complete… by storing references to them and calling `join()`. This caused the references to all futures, and thus also to all the data they referenced, to be kept alive until the end of indexing, and prevented the Garbage Collector from freeing them up earlier.

越来越怀疑，我检查了代码，他们都是大概两年前写完的，从未再被仔细的检查过。果不其然，我们实际上把所有的数据都加载到了内存里。这当然不是故意的。当时不太理解 RxJava，我们想让代码以一种特殊的方式并行运行，决定把 `CompetableFuture` 和 一个独立的 executor 结合起来，把一些工作从主要的 RX 工作流中剥离出来。但是，我们需要等到所有的 `CompetableFuture` 都工作完… 通过存储他们的引用，然后调用 `josn()`。这导致引用了所有的 future，以及所有他们引用到的数据，一直保持生存的状态直到索引完成，这阻止了垃圾收集器及时的把他们清理掉。

## Is it really so bad?
## 真有这么糟糕吗？

This is obviously a stupid mistake, and we were quite disgusted at finding it so late. I even remembered a brief discussion a long time earlier about the app needing a 12 GB heap, which seemed a bit much. But on the other hand, this code had worked for almost two years without any issues. We were able to fix it with relative ease at this point while it would probably have taken us much more time if we tried fixing it two years before and at that time there was a lot of work much more important for the project than saving a few gigabytes of memory.

当然这是一个很愚蠢的错误，我们对于发现得这么晚也感到很恶心。我甚至想起了一个简短的讨论，关于这个应用需要 12 GB 的堆，当时看起来有点多了。但是另一方面，这些代码已经工作了将近两年了，没有发生过任何毛病。我们当时可以花费同样的精力修复它，然而它可能要花费我们更多的时间，如果两年前修复的话，而且当时我们有很多更重要的工作，相比精简几 G 的内存来说。

So while on a purely technical level having this issue for such a long time was a real shame, from a strategic point of view maybe leaving it alone despite the suspected inefficiency was the pragmatically wiser choice. Of course, yet another consideration was the impact of the problem once it came into light. We got away with almost no impact for the users, but it could have been worse. Software engineering is all about trade-offs, and deciding on the priorities of different tasks is no exception.

因此虽然从纯技术的角度来说，这个问题保持这么长时间没解决确实很丢人，从一个战略性的角度来看，或许留着这个浪费内存的问题不管，是更务实的选择。当然，还有一方面就是这个问题爆发时的影响。我们没有对用户造成任何影响，不过它有可能更糟糕。软件工程就是交易，决定不同任务的优先级也不例外。

## Still not working

## 还是不行

Having more RX experience under our belt, we were able to quite easily get rid of the `CompletableFuture`s, rewrite the code to use only RX, migrate to RX2 in the process, and to actually stream the data instead of collecting it in memory. The change passed code review and went to testing in dev environment. To our surprise, the app was still not able to perform indexing with a smaller heap. Memory sampling revealed that the number of ads kept in memory was smaller than previously and it not only grew but sometimes also decreased, so it was not all collected in memory. Still, it seemed as if the data was not being streamed, either.

有了更多使用 RX 的经验之后，我们可以很简单的解决 `ComplerableFurue` 的问题，重写代码，仅仅使用 RX，在过程中升级到 RX2，并且真正的把数据归集成流，而不是把他们收集到内存里。这些改动通过了 code review 然后送去在开发环境测试。让我们吃惊的是，项目仍然不能以一个更小的堆来运行。内存抽样显示，内存中广告的数量稍微有些减少，不是一直增长，而是有时也下贱，因为没有全部收集起来。因为，看起来这些数据仍然没有真正的被归集成流。

## So what is it now?

## 那么现在是怎么回事呢？

The relevant keyword was already used in this post: backpressure. When data is streamed, it is common for the speeds of the producer and the consumer to differ. If the producer is faster than the consumer and nothing forces it to slow down, it will keep producing more and more data which can not be consumed just as fast. There will appear a growing buffer of outstanding records waiting for consumption and this is exactly what happened in our application. Backpressure is the mechanism which allows a slow consumer to tell the fast producer to slow down.

相关的关键词刚才已经提到了：负反馈。当数据被归集起来，生产者和消费的速度不同是很正常的。如果生产者比消费者要快，并且没有外界压力使速度降下来，他就会以同样的速度生产越来越多消费者处理不掉的数据。显现出来的就是大量的数据等待消费，从而占据越来越多的缓存，这就是我们应用中真正发生的。负反馈就是一套机制，允许一个较慢的消费者告诉快速的生产者去降速。

Our indexing stream had no notion of backpressure which was not a problem as long as we were storing the whole index in memory anyway. Once we fixed one problem and started to actually stream the data, another problem — the lack of backpressure — became apparent.

我们的收集系统没有负反馈的概念，这也没什么问题，因为我们把所有的索引都保存到内存里了。一旦我们解决了这个问题，开始真正流式处理数据，缺少负反馈的问题就变得很明显了。

This is a pattern I have seen multiple times when dealing with performance issues: fixing one problem reveals another which you were not even aware of because the other issue hid it from view. You may not be aware your house has a fire safety issue if it is regularly getting flooded.

这是我见过很多次的解决性能问题时的模式：解决一个问题时会浮现另一个你甚至没听说过的问题，因为其他问题把这个给隐藏起来了。你不会主意到你的房子有火灾隐患，如果它经常遭遇水灾。

## Fixing the fix

## 解决问题的问题

In RxJava 2, the original Observable class was split into `Observable` which does not support backpressure and `Flowable` which does. Fortunately, there are some neat ways of creating `Flowable`s which give them backpressure support out-of-the-box. This includes creating `Flowable`s from non-reactive sources such as `Iterable`s. Combining such `Flowable`s results in `Flowable`s which also support backpressure, so fixing just one spot quickly gave the whole stream backpressure support.

在 RxJava 2 里，原来的 Observable 类被拆成了不支持负反馈的 Observable 和负反馈的 Flowable。幸运的是，有一些简洁的办法来从交互性差的来源比如 Iterable 创建一些 Flowable。把这些 Flowable 结合起来会产生同样支持负反馈的 Flowable，因此只要快速解决一个点，整个系统就有了负反馈的支持。

With this change in place, we were able to reduce the heap from 12 GB to 3 GB and still have the app do its job just as fast as before. We still got a single full GC with a pause of roughly 2 seconds once every few hours, but this was already much better than the 20 second pauses (and crashes) we saw before.

有了这个改动之后，我们把堆从 12 GB 减少到了 3 GB 同时让系统保持和之前同样的速度。我们仍然每隔数小时就会有一次暂停长达 2 秒的 full GC，不过这比我们之前见到的 20 秒的暂停（还有系统崩溃）要好多了。

## GC tuning again

## 再次优化 GC

However, the story was not over yet. Looking at GC logs, we still noticed lots of premature tenuring — on the order of 70%. Even though performance was already acceptable, we tried to get rid of this effect, hoping to perhaps also prevent the full garbage collection at the same time.

但是，故事到此还没有结束。检查 GC 的日志，我们注意到大量的提前晋升，占到 70%。尽管性能已经可以接受了，我们尝试解决这个问题，希望也许可以同时解决 full GC 的问题。

![Lots of premature tenuring](https://allegro.tech/img/articles/2018-02-09-a-comedy-of-errors-debugging-java-memory-leaks/premature-tenuring-with-backpressure-added.png)

Premature tenuring (also known as premature promotion) happens when an object is short-lived but gets promoted to the old (tenured) generation anyway. Such objects may affect GC performance since they stuff up the old generation which is usually much larger and uses different GC algorithms than new generation. Therefore, premature promotion is something we want to avoid.

提前晋升（或者叫提前升级）发生于对象是短生命周期的但是仍然晋升到了老年代。这些对象会影响 GC 的性能因为他们占据了老年代的空间，而老年代里的对象通常都比较大，使用与新生代不同的 GC 算法。因此，我们想避免提前晋升。

We knew our app would produce lots of short-lived objects during indexing, so some premature promotion was no surprise, but its extent was. The first thing that comes to mind when dealing with an app that creates lots of short-lived objects is to simply increase the size of young generation. By default, G1GC can adjust the size of generations automatically, allowing for between 5% and 60% of the heap to be used by the new generation. I noticed that in the live app proportions of young and old generations changed all the time over a very wide range of proportions, but still went ahead and checked what would happen if I raised both bounds: `-XX:G1NewSizePercent=40` and `-XX:G1MaxNewSizePercent=90`. This did not help, and it actually made matters much worse, triggering full GCs almost immediately after the app started. I tried some other ratios, but the best I could arrive at was only increasing `G1MaxNewSizePercent` without modifying the minimum value: it worked about as well as defaults but not better.

我们知道我们的应用在索引的过程中会产生大量的短生命周期的对象，因此一些提前晋升是很正常的，但是这个程度不正常。当应用产生大量短生命周期的对象时，第一件事就是简单的增加新生代的空间。默认情况下，G1 的 GC 可以自动的调整新生代的空间，允许新生代使用堆内存的 5% 至 60%。我注意到生存的应用，新生代和老年代的比例一直在一个很宽的幅度变化，但是仍然动手同时修改了两个参数：`-XX:G1NewSizePercent=40` 和 `-XX:G1MaxNewSizePercent=90`看看会发生什么。这没起作用，甚至让事情变得更糟糕了，当应用刚启动就触发了 full GC。我尝试了其他的比例，不过我能达到的最好值就是只增加 `G1MaxNewSizePercent`而不修改最小值，这起作用但是和默认值差不多，没有强很多。

After trying a few other options, with as little success as in my first attempt, I gave up and e-mailed Kirk Pepperdine who is a renowned expert in Java performance and whom I had the opportunity to meet at Devoxx conference and during training sessions at Allegro. After viewing GC logs and exchanging a few e-mails, Kirk suggested an experiment which was to set `-XX:G1MixedGCLiveThresholdPercent=100`. This setting should force G1GC mixed collections to clean all old regions regardless of how much they were filled up, and thus to also remove any objects prematurely tenured from young. This should prevent old generation from filling up and causing a full GC at any point. However, we were again surprised to get a full garbage collection run after some time. Kirk concluded that this behavior, which he had seen earlier in other applications, was a bug in G1GC: the mixed collections were apparently not able to clean up all garbage and thus allowed it to accumulate until full GC. He said he had contacted Oracle about it but they claimed this was not a bug and the behavior we observed was correct.

通过尝试很多其他办法后，由于首次尝试很少成功，我放弃了然后给 Kirk Pepperdine 发了封邮件，这是为很致命的 Java 性能专家，我碰巧在 Allegro 举办的 Devoxx 的会议上的训练课程里认识了。通过查看 GC 的日志以及几封邮件的交流，Kirk 建议尝试设置 `-XX:G1MixedGCLiveThresholdPercent=100`。这个设置应该会强制 G1 GC 清理所有的老年代而不考虑他们被填充了多少，因此同时也删除了从新生代提前晋升的任何对象。这应该会阻止老年代被填满从而产生一次 full GC。然而，我们再次惊讶的发现了一次 full GC 在运行一段时间以后。Kirk 总结说这个行为，他在其他应用里也见到过的，是 G1 GC 的一个 bug：混合的显然没有清理所有的垃圾因此让他们去堆积直到 full GC。他说他已经把这个问题通知了 Oracle，不过他们坚称这不是一个 bug，我们观察到的这个行为是正常的。

## Conclusion

## 结论

What we ended up doing was just increasing the app’s heap size a bit (from 3 to 4 GB), and full garbage collections went away. We still see a lot of premature tenuring but since performance is OK now, we don’t care so much any more. One option we could try would be switching to CMS (Concurrent Mark Sweep) GC, but since it is deprecated by now, we’d rather avoid using it if possible.

我们最后做的就是把应用的内存调大了一点点（从 3 GB 到 4 GB），然后 full GC 就消失了。我们仍然观察到大量的提前晋升，不过既然性能是没问题的，我们不在乎这些了。一个我们可以尝试的选项是转换到 GMS（Concurrent Mark Sweep）GC，不过由于它已经被废弃了，我们还是尽量不去使用它。

![Problem fixed — GC pauses after all changes and 4 GB heap](https://allegro.tech/img/articles/2018-02-09-a-comedy-of-errors-debugging-java-memory-leaks/adventory-gc-pause-fixed-4-gb.png)

So what is the moral of the story? First, performance issues can easily lead you astray. What at first seemed to be a ZooKeeper or network issue, turned out to be an error in our own code. Even after realizing this, the first steps I undertook were not well thought out. I started tuning garbage collection in order to avoid full GC before checking in detail what was really going on. This is a common trap, so beware: even if you have an intuition of what to do, check your facts and check them again in order to not waste time solving a problem different from the one you are actually dealing with.

那么这个故事的寓意是什么呢？首先，性能问题很容易让你误入歧途。一开始看起来是 ZooKeeper 或者 网络的问题，最后发现是我们代码的问题。即使意识到了这一点，我首先采取的措施也没有考虑周全。我开始调优垃圾回收来防止 full GC 在检查到底发生什么之前。这是一个常见的陷阱，因此记住：即使你有一个直觉去做什么，检查你的事实，然后再检查一遍，防止解决一个和你真正遇到的问题不同的另一个问题。

Second, getting performance right is really hard. Our code had good test coverage and feature-wise worked perfectly, but failed to meet performance requirements which were not clearly defined at the beginning and which did not surface until long after deployment. Since it is usually very hard to faithfully reproduce your production environment, you will often be forced to test performance in production, regardless of how bad that sounds.

第二条，让性能跑起来太难了。我们的代码有良好的测试覆盖率，而且运行的特别好，但是它也没有满足性能的要求，这个要求在开始的时候没有清晰的定义好，因此没有浮现出来直到部署之后。由于通常很难真实的再现你的生产环境，我经常被迫在生产环境测试性能，而不管那听起来有多糟糕。

Third, fixing one issue may allow another, latent one, to surface, and force you to keep digging in for much longer than you expected. The fact that we had no backpressure was enough to break the app, but it didn’t become visible before we had fixed the memory leak.

第三条，解决一个问题有可能引发另一个潜在的问题浮现出来，强迫你不断的比你预想的挖的更深。我们没有负反馈的事实足以中断这个系统，但是他没有浮现知道我们解决了内存泄漏的问题。

I hope you find this funny experience of ours helpful when debugging your own performance issues!

我希望你通过我们这个有趣的经历，能在你解决自己遇到的性能问题时可以发挥作用。