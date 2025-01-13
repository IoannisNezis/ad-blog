---
title: "Dynamic Observation and Interruption of SPARQL Queries"
date: 2024-04-12T16:44:11+02:00
author: "Robin Textor-Falconi"
authorAvatar: "img/ada.jpg"
tags: ["SPARQL", "WebSocket", "HTTP", "Synchronization", "Atomics", "QLever"]
categories: ["project"]
image: "img/teaser.png"
draft: false
---

A dive into the process of designing an architecture that allows
observers to interact with complex queries in real-time.

<!--more-->

## Content

1. [Real-Time is more than quality of life](#real-time-is-more-than-quality-of-life)
2. [Live query analysis for QLever](#live-query-analysis-for-qlever)
    1. [Bidirectional Web Communication](#bidirectional-web-communication)
    2. [Designing a WebSocket API](#designing-a-websocket-api)
        1. [The runtime information tree](#the-runtime-information-tree)
        2. [`Boost.Beast`, `Boost.Asio` and Concurrent connections](#boostbeast-boostasio-and-concurrent-connections)
3. [Embracing Cancel Culture](#embracing-cancel-culture)
    1. [Cancellation via WebSockets](#cancellation-via-websockets)
    2. [Cancellation Handles and Watchdogs](#cancellation-handles-and-watchdogs)
    3. [Collateral Benefits](#collateral-benefits)
4. [Conclusion](#conclusion)


## Real-Time is more than quality of life

QLever is a SPARQL-compliant knowledge graph engine that allows for blazingly fast execution of SPARQL queries on giant RDF
data graphs. However, even though most common queries run in less than a couple of seconds, of course, more expensive
queries exist that will take dozens of seconds, minutes or even whole hours to compute. These kinds of queries will
eventually be completed but generally, it's hard to tell how long a query will take from an outside perspective.
In addition to that, if you had this knowledge after starting the query and decide it's not worth the wait it would be
nice if you could cancel this query directly. Previously there was no mechanism to actually cancel a running query.
Instead, it would just compute the query until the very end potentially taking up a lot of limited resources in the process,
blocking other queries from running for a possibly long time in  the process. Also for us as QLever developers it would be
nice to see what's going on during a query to be able to identify bottlenecks in complicated queries. Obviously, this is a
state that's less than ideal. So my job was to improve this.

## Live query analysis for QLever

### Bidirectional Web Communication

HTTP and thus the whole world wide web is inherently centered around a simple request-response principle. The client (usually a
web browser) makes a request and waits for the server to respond with a response. This works very nicely in a lot of cases and
is easy to understand. So it makes sense for the SPARQL protocol to build on top of the HTTP protocol. In this particular case
though, it does not play well with what I'm trying to achieve. For a system to be observable in real-time the client needs to
wait for the server to notify it about any updates to its processing.

In the past web apps worked around this issue by doing so-called "long polling". In contrast to regular polling where the client
repeatedly makes short-lived requests to the server waiting for anything to happen eventually, long polling starts a request and
expects the server not to respond until anything happens. The main disadvantage of long polling is of course that it comes with
some overhead for every single server-side update. This overhead introduces latency because the server needs to wait for an
incoming client request before sending anything back to the client and requires an extra mechanism to keep track of the
state across polls.

<figure>

![HTTP server-side updates](img/HTTP%20Server-Side%20Updates.svg)

<figcaption>Figure 1: Example of Server-initiated communication using different techniques.</figcaption>
</figure>

To address this issue HTML5 introduced the WebSocket API that works similarly to regular TCP sockets but in a safer
browser-compatible way. WebSockets are initiated similarly to a regular HTTP request using some special headers and then proceed by taking over the underlying TCP connection and performing all further communication through that.

Alternatively, there's also a protocol called WebRTC which could be used to achieve similar things, but the complexity that comes
with it to support more advanced use-cases like peer-to-peer communication as well as UDP-like unreliable transport is not
really worth the hassle.

### Designing a WebSocket API

I designed the general architecture to be rather simple. While a query is running, you can connect to its respective WebSocket
endpoint. Once connected the endpoint will proceed to send you JSON data whenever there's an update to the processing stage
of the selected query. To make sure you don't miss anything, as long as the query is still running when establishing the
WebSocket connection the client will also receive all previous updates in chronological order. To make timing issues even
less of a problem, it's also allowed to establish WebSocket connections before an actual query has started. In this case,
the WebSocket connection will just stay open without sending anything until the corresponding query has started.

I have been talking about a "corresponding query" a lot, but so far I didn't mention anything about how to actually associate
a WebSocket connection with an actual query. The SPARQL specification states that regular queries have to be provided by using
the regular HTTP/1.1 protocol, so this part cannot be changed if we want to stay SPARQL compliant. So there needs to be a way
to tell the server which query is of any interest to us when establishing a WebSocket connection. So I introduced the concept
of "Query IDs". By default, every query gets assigned a randomly generated query ID, which uniquely identifies this exact query.

The problem with this is that it's impossible for the client to know this ID until the query has been computed and the result
is sent back to the client, which of course defeats its entire purpose. So this is why the client can provide a special
`Query-Id` header that sets the query ID to a client-defined value if this value is not already taken. This way the client can
generate a random ID on its own, establish a WebSocket connection on the path `/watch/<query-id>` and then finally run the
actual query via HTTP with the `Query-ID: <query-id>` header set. By using a UUID the chance for an actual ID collision even
across clients becomes almost zero. It also acts as a sort of authentication mechanism for query cancellation, because a
malicious third party can't simply guess an ID and cancel it as a sort of denial of service attempt.

<figure>

![Query Lifetime](img/Query%20Lifetime.svg)

<figcaption>Figure 2: Request lifecycle with the newly introduced query ID system.</figcaption>
</figure>

#### The runtime information tree

How is the processing status data serialized? Luckily there was already a mechanism present that could be repurposed for this
new task. QLever already tracked execution times and other metadata of its inner computation tree. This mechanism updates
the stats during computation and aggregates the final results into a whole "runtime information" tree at the end of every
subcomputation. So with some small modifications, this mechanism could easily get repurposed into something that provides
all of the information we want in 3 steps:

1. Create a "runtime information" tree when running an operation, so that all suboperations can modify it with visible changes during execution and not just afterwards.
2. Wire in a broadcast call that notifies all listeners of a change to the tree whenever it is being modified.
3. Add in a new status "in progress" so you can tell which of the suboperations is currently being executed, instead of just
being notified when they finish.

QLever UI is a convenient web UI for the QLever backend. It makes it easy to use QLever from your browser. It also already
rendered this tree in a human-readable form so everyone can easily analyse the bottlenecks of individual queries. Needless to
say it wasn't too difficult to expand this functionality to update this visual representation based on messages coming
from a newly created WebSocket connection instead of just a plain HTTP response. Of course, there are some pitfalls that
need to be avoided, for example making sure the final HTTP response doesn't get replaced with a delayed WebSocket message.
But apart from that this process was rather straightforward.

<figure>

![Example of a `RuntimeInformation` tree](img/RuntimeInformation%20Tree.png)

<figcaption>Figure 3: Example of how QLever UI would represent the runtime information tree.</figcaption>
</figure>

#### `Boost.Beast`, `Boost.Asio` and Concurrent connections

To implement all of this on the backend, the `Boost.Beast` library, built on top of `Boost.Asio` is used. It already
served as the HTTP backbone for QLever for quite a while now. To be honest, I might be spoiled by HTTP server frameworks
in a lot of other languages that do almost everything for you out of the box, but I always felt like even though `Boost.Beast`
comes with a lot of ready-to-use components it still expects you to build the server on your own. It gives you a toolbox
with all the parts to build an engine when in reality I would've liked to order the entire pre-built engine in the first place.
I don't think I could build a better engine than experts on the matter, even if it isn't tailored to fit my exact use case.

Because of this exposing a WebSocket endpoint in the backend was quite a hassle. The concept in itself wasn't really complex.
When a new connection is being established and the query ID is not yet in use (usually this means the query has not started yet)
the server just keeps the connection alive without sending anything else until a query with this specific ID starts. There can
be multiple WebSockets waiting for the exact same query which doesn't really help keeping things simple, mainly to allow
observations by server administrators in case queries are hogging up a lot of resources for no apparent reason. Once all
listeners are connected to QLever and a query makes progress it has to broadcast the latest update to all listeners. Of course,
there are some important considerations to be made here. The broadcast operation cannot be blocking in any way, otherwise,
a slow WebSocket client can just slow down queries artificially. Also ideally all WebSockets should be able to operate
independently of each other for maximum efficiency, but due to technical limitations of `Boost.Asio` we ended up having to 
run some operations in a somewhat synchronized manner to avoid race conditions.

Traditionally handling I/O with multiple threads is always a challenge. The "classic" approach to read input streams of files,
network traffic and so on dedicates a single thread to read all the data, waiting in a blocking manner until the data finally
becomes available. This makes the code rather easy to understand but is typically a very inefficient use of computing resources
because a whole thread is unable to do anything until there's new data to read. While this is not a problem when reading a
single file from disk, this very much becomes a problem when waiting for incoming connections over the network. Chances are
there will be more concurrent requests than threads available on your computer so spinning up more and more expensive threads
just to keep up with traffic is not really a feasible option.

```py3
def open_file():
    # Classic blocking approach, works in any function
    with open('file.txt') as file:
        for line in file:
            print(line)

async def open_file_async():
    # Async non-blocking approach, only works in async functions
    async with aiofiles.open('file.txt') as file:
        async for line in file:
            # Non-async functions can still be called here
            print(line)
```

<p style="text-align: center; font-size: 0.75em; transform: translateY(-1em)">Code snippet 1: Example of the colouring problem in python.</p>

Other options that are often used are callback-based approaches, sometimes wrapped inside so-called promises or futures.
Basically, instead of blocking a thread until data becomes available, we tell the operating system to run our code we
pass it once data becomes available and do something else in the meantime. This uses hardware much more efficiently
([unless you're optimizing for low-latency hardware](https://youtu.be/EJa6RatD_yo)) because we no longer need to keep
more threads around than we actually need. The downside of this is that it adds complexity to code, making it harder
to read and maintain in the process. In QLever we use `Boost.Asio` with C++20 coroutines, in a similar way to how other
languages like JavaScript, Rust, C# and more allow to deal with asynchronous code. While it is easier to read than
using callbacks it still suffers from the [function colouring problem](https://journal.stuffwithstuff.com/2015/02/01/what-color-is-your-function/)
which basically means that to make ideal use of coroutines eventually your whole codebase will end up consisting
purely of other coroutines in our case. If you're interested in these kinds of discussions, I can recommend
[this talk](https://youtu.be/EO9oMiL1fFo) by Ron Pressler, the technical lead for OpenJDK's "Project Loom" where
he explains why Java explicitly decided against introducing async/await keywords.

I'm telling you this to make it clear why the naive approach to solving the problem of multiple clients waiting for
something to happen is not an option. Of course, it would work to just spawn a new thread for all incoming requests
and put them to sleep until they are woken up the another thread that is computing the result of the current query,
but we just can't afford this overhead in case QLever is ever being used in a high traffic environment.

What ends up happening instead is that in C++ with `Boost.Asio` coroutines end up hiding the callback structure
of asynchronous I/O in a somewhat nice way. For the underlying non-coroutine logic, however, it still acts like
it is a collection of chained callbacks. So the architecture roughly looks as follows: For every query, there's a
list of events that happened so far. When a WebSocket is connected to a query it gets initialized with an index of 0.
This index is then compared against the size of the list with all events. If the index is smaller than the size
of the list it just reads this entry, increments the counter and repeats this cycle. If the index is equal to the
size of the list, this means we've reached the end of the event queue and need to wait for anything to happen. In
that case, we put a task that is supposed to be executed when new data is present into another list keeping track of all
waiting WebSocket clients. This task also continues the same cycle when called. When the query is updated because a
step has finished, it then clears this list of tasks after scheduling all of those tasks for execution, repeating the cycle
again until the query ends which signals all of those tasks that no data will follow and they can safely close the WebSocket.
Because of this complicated API, we also need to properly handle the case when a WebSocket connection closes unexpectedly
before a query can finish. Otherwise opening and closing connections would create somewhat of a memory leak, which is of
course also less than desirable. The naive approach would've been rather straightforward to implement. The actual implementation
on the other hand not so much.

<figure>

![Async index structure example](img/Async%20Read%20Access.svg)

<figcaption class="multi-line">Figure 4: Example of how different observers of the query
keep track of their own index for a specific query so they can all concurrently read the data structure
at that exact index without a mutex. Queries with an index higher than the amount of currently available
data are asynchronously put to sleep until there is an update and their index is safe to read.</figcaption>
</figure>

## Embracing Cancel Culture

Now that we've talked about how to implement live broadcasting of query updates, let's talk about how to cancel
those same queries when desired.

### Cancellation via WebSockets

The classic HTTP/1.x protocol is really simple. Send stuff over TCP, receive stuff back over TCP. Everything is text-based
everything is sequential. This makes HTTP rather simple to implement but is a major drawback in our case. TCP can only close
streams from a sender's perspective. This means it is perfectly legal (even though [you will likely end up in a timeout](https://www.excentis.com/blog/tcp-half-close-a-cool-feature-that-is-now-broken/)) for a client to close its connection for sending
but still keep receiving messages indefinitely. So for the server to notice if a connection is truly dead for one
reason or another it has to try and send a packet, wait for an acknowledgement packet and timeout after a while if it doesn't
arrive. Remember that HTTP is strictly sequential? Well, this means we can't just send out a test packet on the HTTP level
to see if our connection is still working unless we had some data ready which we clearly haven't when we're still computing
this exact data. So we'd have to go a level deeper onto the TCP level. TCP has a keep-alive mechanism for this exact purpose
but... [it isn't exactly very strict](https://stackoverflow.com/questions/1480236/does-a-tcp-socket-connection-have-a-keep-alive).
On Linux for example the operating system doesn't start sending keep-alive packets before 2 hours have elapsed, which is
almost comically late to save computing resources effectively. Different browsers start sending their keep-alive packets
after a couple of minutes, so you can't really rely on that either. But even if all timeouts were in the single digit seconds
range, turns out this feature of TCP is entirely optional, so you're not guaranteed to have your keep-alive packet relayed
all the way. If you're using a reverse proxy or the user is using an HTTP proxy, it probably won't even get past that.
In other words, there is no good way to detect if a user is no longer waiting for a response.

What we really want is a mechanism that (ideally without having to strictly cooperate) notifies the server when the
user has navigated away from QLever UI for example. Turns out WebSockets can help here too. The idea is very simple.
We can re-use all of the logic for live query analysis, but now also allow the client to send the following keywords:
`cancel` and `cancel_on_close`. The former keyword simply requests the query to be cancelled as soon as possible.
This could also just be a regular HTTP endpoint, but at least for QLever UI, we'll have a WebSocket open for every
query anyway, so that's just for convenience. In the future, we might add a normal HTTP endpoint if it ever becomes
beneficial. It should become clear here why using random query IDs makes sense. This way only we know which ID to use
to cancel our query. A malicious third party would have to guess a practically unlimited amount of possible queries to
get the right one. The latter keyword `cancel_on_close` however, is what justifies using WebSockets in the first place.
Web browsers always try to gracefully close WebSocket connections when closing or navigating away from a page. So within
milliseconds, we can detect that the client is no longer interested in a response when using QLever UI. If the connection
drops for some unexpected reason we won't notice immediately but our live-update mechanism sends out multiple messages
during computation, so once a subcomputation is completed the server will notice that a WebSocket packet wasn't received
by the client and thus detect a disconnected connection way sooner.

<figure>

![Query decision chart](img/Query%20Decision%20Chart.svg)

<figcaption class="multi-line">Figure 5: A decision chart that illustrates how to use
the new web API.</figcaption>
</figure>

I just described a somewhat cooperative approach. A WebSocket needs to be created, and the client needs to send a keyword
for everything to work nicely. This is a good option for cases where we control the front end (QLever UI in our case), but
many users prefer to use other tools like curl for queries that potentially produce gigabytes of data. How
can we handle those? There isn't a definitive answer so far. The most likely approach we'll be settling on is to make
queries timeout really quickly by default and require an active WebSocket connection to increase the timeout way beyond
that threshold. Other options may include newer HTTP versions, which are completely incompatible with HTTP/1.1 and below
but come with a [GOAWAY packet](https://datatracker.ietf.org/doc/html/rfc7540#section-6.8), that indicates an imminent
end of a connection.

### Cancellation Handles and Watchdogs

The server was notified that the result of a query is now no longer required. What now? How can it react to this event
and stop a query during execution? The solution is also very simple here. Every single query gets its own atomic flag
that can be set by either a cancelling WebSocket or after a configurable time delay by the server itself. We call it the
`CancellationHandle`. During computation, it's now the developer's responsibility to sprinkle the code with enough checks
that throw an exception if it has been set, cancelling computation right there. When using relaxed memory ordering to
access the atomic variable, reading an atomic variable this way is so cheap that it almost doesn't make any difference
at all for overall performance. As you can probably imagine it's rather easy to put these checks in place, but code
will change sooner or later, and it is incredibly easy to just forget adding these in when implementing a new feature
or refactoring existing code. That's why a Watchdog mechanism was introduced.

The Watchdog mechanism is enabled by default (the code can be compiled without it for maximum performance). After a while
(50ms) it will set the `CancellationHandle` into a "challenge" state, where it gives the computation code 50ms to check if
the `CancellationHandle` was set into a cancelled state and reset it back to the "not cancelled" state. If this reset
doesn't happen within the time interval, the watchdog proceeds to print a warning in the console, so developers can
see it and add more checks to the code accordingly. The "challenge" has to be separate from the "not cancelled" state
because it turns out that reading an atomic variable is a lot cheaper than actually writing to it. So only by writing
to it exclusively whenever the challenge is active, this greatly improves performance. This approach is of course not
perfect, mainly because it can be easily ignored in contrast to a failing unit test for example, but it makes it
simpler to spot those issues even in production, which is a step in the right direction.

<figure>

![Watchdog diagram](img/Watchdog%20Diagram%20QLever.svg)

<figcaption class="multi-line">Figure 6: Whenever a cancellation check is performed,
the query is cancelled or the watchdog timer expires (every 50ms by default) the current state of the CancellationHandle
is advanced according to the figure if there's a corresponding connection. Whenever the state is reset back to
"Not cancelled" from "Check window missed" a warning is printed in the console.</figcaption>
</figure>

### Collateral Benefits

An unintended benefit of the Watchdog was that because of it, we were able to spot some general performance issues with
the code. In particular, we were wondering why the "check window missed" messages seemed to increase exponentially at
a specific stage in the code. This revealed a piece of code that had to allocate an increasing amount of memory because
the initial array wasn't allocated to match the known size and the vector had to grow a lot of times. If the timeout window
is going to be reduced at some point in the future, this could again reveal code that might not operate ideally, making it
really neat overall.

A working cancellation mechanism is also very important to allow potential future fair resource distribution. Currently,
everyone can use the server as they wish, but computing power is limited. So to provide a good experience for everyone
there needs to be a mechanism that can just cancel an entire query if it takes too long or slows everything else down.
As a nice bonus, I also implemented an HTTP endpoint that just returns a list of all currently running queries along with
their ID if you have permission to access it. This way server admins can just take matters into their own hands and just
forcefully terminate computations that have been running for way too much time. Convenient!

## Conclusion

So that wraps everything up. A WebSocket endpoint was implemented to allow to interactively observe a running query and
cancel it on demand in a rather efficient manner without breaking SPARQL compatibility. It was a challenging, but overall
rewarding journey. I hope you enjoyed the read.

If you're curious feel free to try it out yourself, just head over to https://qlever.cs.uni-freiburg.de and start experimenting.

<style>
    figure {
        transform: translateY(-0.5em)
    }
    figcaption {
        text-align: center;
        font-size: 0.75em;
        transform: translateY(-1em);
    }
    figcaption.multi-line {
        text-align: justify;
        padding: 0 5em;
    }
</style>
