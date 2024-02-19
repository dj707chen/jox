# jox

Fast and Scalable Channels in Java. Designed to be used with Java 21+ and virtual threads,
see [Project Loom](https://openjdk.org/projects/loom/) (although the `core` module can be used with Java 17+).

Inspired by the "Fast and Scalable Channels in Kotlin Coroutines" [paper](https://arxiv.org/abs/2211.04986), and
the [Kotlin implementation](https://github.com/Kotlin/kotlinx.coroutines/blob/master/kotlinx-coroutines-core/common/src/channels/BufferedChannel.kt).

Articles:

* [Announcing jox: Fast and Scalable Channels in Java](https://softwaremill.com/announcing-jox-fast-and-scalable-channels-in-java/)
* [Go-like selects using jox channels in Java](https://softwaremill.com/go-like-selects-using-jox-channels-in-java/)

## Dependencies

Maven:

```xml

<dependency>
    <groupId>com.softwaremill.jox</groupId>
    <artifactId>core</artifactId>
    <version>0.1.0</version>
</dependency>
```

Gradle:

```groovy
implementation 'com.softwaremill.jox:core:0.1.0'
```

SBT:

```scala
libraryDependencies += "com.softwaremill.jox" % "core" % "0.1.0"
```

## Usage

### Rendezvous channel

```java
import com.softwaremill.jox.Channel;

class Demo1 {
    public static void main(String[] args) throws InterruptedException {
        // creates a rendezvous channel
        // (a sender & receiver must meet to pass a value: as if the buffer had size 0)
        var ch = new Channel<Integer>();

        Thread.ofVirtual().start(() -> {
            try {
                // send() will block, until there's a matching receive()
                ch.send(1);
                System.out.println("Sent 1");
                ch.send(2);
                System.out.println("Sent 2");
                ch.send(3);
                System.out.println("Sent 3");
            } catch (InterruptedException e) {
                throw new RuntimeException(e);
            }
        });

        System.out.println("Received: " + ch.receive());
        System.out.println("Received: " + ch.receive());
        System.out.println("Received: " + ch.receive());
    }
}
```

### Buffered channel

```java
import com.softwaremill.jox.Channel;

class Demo2 {
    public static void main(String[] args) throws InterruptedException {
        // creates a buffered channel (buffer of size 3)
        var ch = new Channel<Integer>(3);

        // send()-s won't block
        ch.send(1);
        System.out.println("Sent 1");
        ch.send(2);
        System.out.println("Sent 2");
        ch.send(3);
        System.out.println("Sent 3");
        // the next send() would block

        System.out.println("Received: " + ch.receive());
        System.out.println("Received: " + ch.receive());
        System.out.println("Received: " + ch.receive());
        // same for the next receive() - it would block
    }
}
```

Unlimited channels can be created with `Channel.newUnlimitedChannel()`. Such channels will never block on send().

### Closing a channel

Channels can be closed, either because the source is `done` with sending values, or when there's an `error` while
the sink processes the received values.

`send()` and `receive()` will throw a `ChannelClosedException` when the channel is closed. Alternatively, you can
use the `sendSafe()` and `receiveSafe()` methods, which return either a `ChannelClosed` value (reason of closure),
or `null` / the received value.

Channels can also be inspected whether they are closed, using the `isClosedForReceive()` and `isClosedForSend()`.

```java
import com.softwaremill.jox.Channel;

class Demo3 {
    public static void main(String[] args) throws InterruptedException {
        // creates a buffered channel (buffer of size 3)
        var ch = new Channel<Integer>(3);

        // send()-s won't block
        ch.send(1);
        ch.done();

        // prints: Received: 1
        System.out.println("Received: " + ch.receiveSafe());
        // prints: Received: ChannelDone[]
        System.out.println("Received: " + ch.receiveSafe());
    }
}
```

### Selecting from multiple channels

The `select` method selects exactly one clause to complete. For example, you can receive a value from exactly one
channel:

```java
import com.softwaremill.jox.Channel;

import static com.softwaremill.jox.Select.select;

class Demo4 {
    public static void main(String[] args) throws InterruptedException {
        // creates a buffered channel (buffer of size 3)
        var ch1 = new Channel<Integer>(3);
        var ch2 = new Channel<Integer>(3);
        var ch3 = new Channel<Integer>(3);

        // send a value to two channels
        ch2.send(29);
        ch3.send(32);

        var received = select(ch1.receiveClause(), ch2.receiveClause(), ch3.receiveClause());

        // prints: Received: 29
        System.out.println("Received: " + received);
        // ch3 still holds a value that can be received
    }
}
```

The received value can be optionally transformed by a provided function.

`select` is biased: if a couple of the clauses can be completed immediately, the one that appears first will be
selected.

Similarly, you can select from a send clause to complete. Apart from the `Channel.sendClause()` method, there's also a
variant which runs a callback, once the clause is selected:

```java
import com.softwaremill.jox.Channel;

import static com.softwaremill.jox.Select.select;

class Demo5 {
    public static void main(String[] args) throws InterruptedException {
        var ch1 = new Channel<Integer>(1);
        var ch2 = new Channel<Integer>(1);

        ch1.send(12); // buffer is now full

        var sent = select(ch1.sendClause(13, () -> "first"), ch2.sendClause(25, () -> "second"));

        // prints: Sent: second
        System.out.println("Sent: " + sent);
    }
}
```

Optionally, you can also provide a default clause, which will be selected if none of the other clauses can be completed
immediately:

```java
import com.softwaremill.jox.Channel;

import static com.softwaremill.jox.Select.defaultClause;
import static com.softwaremill.jox.Select.select;

class Demo6 {
    public static void main(String[] args) throws InterruptedException {
        var ch1 = new Channel<Integer>(3);
        var ch2 = new Channel<Integer>(3);

        var received = select(ch1.receiveClause(), ch2.receiveClause(), defaultClause(52));

        // prints: Received: 52
        System.out.println("Received: " + received);
    }
}
```

## Performance

The project includes benchmarks implemented using JMH - both for the `Channel`, as well as for some built-in Java
synchronisation primitives (queues), as well as the Kotlin channel implementation.

The test results for version 0.1.0, run on an M1 Max MacBook Pro, with Java 21.0.1, are as follows:

```
Benchmark                                                       (capacity)  (chainLength)  (parallelism)  Mode  Cnt     Score     Error  Units

// jox - multi channel

ChainedBenchmark.channelChain                                            0          10000            N/A  avgt   20   156.385 ±   1.428  ns/op
ChainedBenchmark.channelChain                                           16          10000            N/A  avgt   20    15.478 ±   0.109  ns/op
ChainedBenchmark.channelChain                                          100          10000            N/A  avgt   20     7.502 ±   0.141  ns/op

ParallelBenchmark.parallelChannels                                       0            N/A          10000  avgt   20   155.535 ±   4.153  ns/op
ParallelBenchmark.parallelChannels                                      16            N/A          10000  avgt   20    23.127 ±   0.188  ns/op
ParallelBenchmark.parallelChannels                                     100            N/A          10000  avgt   20     9.193 ±   0.111  ns/op

// kotlin - multi channel

ChainedKotlinBenchmark.channelChain_defaultDispatcher                    0          10000            N/A  avgt   20    74.912 ±   0.896  ns/op
ChainedKotlinBenchmark.channelChain_defaultDispatcher                   16          10000            N/A  avgt   20     6.958 ±   0.209  ns/op
ChainedKotlinBenchmark.channelChain_defaultDispatcher                  100          10000            N/A  avgt   20     4.917 ±   0.128  ns/op

ChainedKotlinBenchmark.channelChain_eventLoop                            0          10000            N/A  avgt   20    90.848 ±   1.633  ns/op
ChainedKotlinBenchmark.channelChain_eventLoop                           16          10000            N/A  avgt   20    30.055 ±   0.247  ns/op
ChainedKotlinBenchmark.channelChain_eventLoop                          100          10000            N/A  avgt   20    27.762 ±   0.201  ns/op

ParallelKotlinBenchmark.parallelChannels_defaultDispatcher               0            N/A          10000  avgt   20    74.002 ±   0.671  ns/op
ParallelKotlinBenchmark.parallelChannels_defaultDispatcher              16            N/A          10000  avgt   20    11.009 ±   0.223  ns/op
ParallelKotlinBenchmark.parallelChannels_defaultDispatcher             100            N/A          10000  avgt   20     4.145 ±   0.149  ns/op

// java built-in - multi queues

ChainedBenchmark.queueChain                                              0          10000            N/A  avgt   20    94.836 ±  14.374  ns/op
ChainedBenchmark.queueChain                                             16          10000            N/A  avgt   20     8.534 ±   0.119  ns/op
ChainedBenchmark.queueChain                                            100          10000            N/A  avgt   20     4.215 ±   0.042  ns/op

ParallelBenchmark.parallelQueues                                         0            N/A          10000  avgt   20    98.573 ±  12.233  ns/op
ParallelBenchmark.parallelQueues                                        16            N/A          10000  avgt   20    24.144 ±   0.957  ns/op
ParallelBenchmark.parallelQueues                                       100            N/A          10000  avgt   20    15.537 ±   0.112  ns/op

// jox - single channel

RendezvousBenchmark.channel                                            N/A            N/A            N/A  avgt   20   173.645 ±   4.181  ns/op

BufferedBenchmark.channel                                               16            N/A            N/A  avgt   20   178.973 ±  45.096  ns/op
BufferedBenchmark.channel                                              100            N/A            N/A  avgt   20   144.355 ±  28.172  ns/op

// kotlin - single channel

RendezvousKotlinBenchmark.channel_defaultDispatcher                    N/A            N/A            N/A  avgt   20   108.400 ±   1.227  ns/op

BufferedKotlinBenchmark.channel_defaultDispatcher                       16            N/A            N/A  avgt   20    35.717 ±   0.264  ns/op
BufferedKotlinBenchmark.channel_defaultDispatcher                      100            N/A            N/A  avgt   20    27.049 ±   0.060  ns/op

// jox - selects

SelectBenchmark.selectWithSingleClause                                 N/A            N/A            N/A  avgt   20   190.910 ±   2.997  ns/op
SelectBenchmark.selectWithTwoClauses                                   N/A            N/A            N/A  avgt   20   812.192 ±  36.830  ns/op

// kotlin - selects

SelectKotlinBenchmark.selectWithSingleClause_defaultDispatcher         N/A            N/A            N/A  avgt   20   171.426 ±   4.616  ns/op
SelectKotlinBenchmark.selectWithTwoClauses_defaultDispatcher           N/A            N/A            N/A  avgt   20   228.280 ±  10.847  ns/op

// java built-in - single queue                                                                             

BufferedBenchmark.arrayBlockingQueue                                    16            N/A            N/A  avgt   20   366.444 ±  67.573  ns/op
BufferedBenchmark.arrayBlockingQueue                                   100            N/A            N/A  avgt   20   110.189 ±   3.494  ns/op

RendezvousBenchmark.exchanger                                          N/A            N/A            N/A  avgt   20    90.830 ±   0.610  ns/op
RendezvousBenchmark.synchronousQueue                                   N/A            N/A            N/A  avgt   20  1501.291 ± 253.663  ns/op
```

## Feedback

Is what we are looking for!

Let us know in the issues, or our [community forum](https://softwaremill.community/c/open-source/11).

## Further work

There's some interesting features which we're planning to work on. Check out
the [open issues](https://github.com/softwaremill/jox/issues)!

## Project sponsor

We offer commercial development services. [Contact us](https://softwaremill.com) to learn more!

## Copyright

Copyright (C) 2023-2024 SoftwareMill [https://softwaremill.com](https://softwaremill.com).
