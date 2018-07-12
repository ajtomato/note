# OVERVIEW

## QUICK START

*nsqlookupd*

* tcp port: 4160
* http port: 4161

*nsqd*

* tcp port: 4150
* http port: 4151
* https port: 4152

*nsqadmin*

* http port: 4171
    
Both *nsqd* AND *nsqadmin* connect to *nsqlookupd*.
    
*nsq_to_file* (the client) is not explicitly told where the topic is produced, it retrieves this information from *nsqlookupd* and, despite the timing of the connection, no messages are lost.

## FEATURES & GUARANTEES

### Features

* support distributed topologies with no SPOF
* horizontally scalable (no brokers, seamlessly add more nodes to the cluster)
* low-latency push based message delivery (performance)
* combination load-balanced and multicast style message routi
ng
* excel at both streaming (high-throughput) and job oriented (low-throughput) workloads
* primarily in-memory (beyond a high-water mark messages are transparently kept on disk)
* runtime discovery service for consumers to find producers (*nsqlookupd*)
* transport layer security (TLS)
* data format agnostic
* few dependencies (easy to deploy) and a sane, bounded, default configuration
* simple TCP protocol supporting client libraries in any language
* HTTP interface for stats, admin actions, and producers (no client library needed to publish)
* integrates with *statsd* for realtime instrumentation
* robust cluster administration interface (*nsqadmin*)

### Guarantees

* messages are not durable (by default)Anchor link for: messages are not durable by default
* messages are delivered at least once
* messages received are un-ordered
* consumers eventually find all topic producers

## FAQ

### Deployment

#### What is the recommended topology for nsqd?

We strongly recommend running an nsqd alongside any service(s) that produce messages.

This pattern aids in structuring message flow as a consumption problem rather than a production one.

Another benefit is that it essentially forms an independent, sharded, silo of data for that topic on a given host.

#### How many nsqlookupd should I run?

3 or 5 works really well for deployments involving up to several hundred hosts and thousands of consumers.

### Publishing

#### Do I need a client library to publish messages?

In fact, the overwhelming majority of NSQ deployments use HTTP to publish.

### Design and Theory

#### How do you recommend naming topics and channels?

A topic name should describe the data in the stream.

A channel name should describe the work performed by its consumers.

## DESIGN

NSQ is a successor to *simplequeue* (part of *simplehttp*) and as such is designed to (in no particular order):

* support topologies that enable high-availability and eliminate SPOFs
* address the need for stronger message delivery guarantees
* bound the memory footprint of a single process (by persisting some messages to disk)
* greatly simplify configuration requirements for producers and consumers
* provide a straightforward upgrade path
* improve efficiency

### Simplifying Configuration and Administration

A single *nsqd* instance is designed to handle multiple streams of data at once. Streams are called “topics” and a topic has 1 or more “channels”. Each channel receives a copy of all the messages for a topic. In practice, a channel maps to a downstream service consuming a topic.

Topics and channels all buffer data independently of each other, preventing a slow consumer from causing a backlog for other channels (the same applies at the topic level).

Messages are multicast from topic -> channel (every channel receives a copy of all messages for that topic) but evenly distributed from channel -> consumers (each consumer receives a portion of the messages for that channel).

*NSQ* also includes a helper application, *nsqlookupd*, which provides a directory service where consumers can lookup the addresses of nsqd instances that provide the topics they are interested in subscribing to.

At a lower level each *nsqd* has a long-lived TCP connection to *nsqlookupd* over which it periodically pushes its state. This data is used to inform which *nsqd* addresses *nsqlookupd* will give to consumers. For consumers, an HTTP /lookup endpoint is exposed for polling.

It is important to note that the *nsqd* and *nsqlookupd* daemons are designed to operate independently, without communication or coordination between siblings.

### Eliminating SPOFs

*NSQ* is designed to be used in a distributed fashion. *nsqd* clients are connected (over TCP) to **all** instances providing **the specified topic**.

For *nsqlookupd*, high availability is achieved by running multiple instances. They don’t communicate directly to each other and data is considered eventually consistent. Consumers poll **all** of their configured *nsqlookupd* instances and **union** the responses.

### Message Delivery Guarantees

NSQ guarantees that a message will be delivered at least once, though duplicate messages are possible. Consumers should expect this and de-dupe or perform idempotent operations.

This guarantee is enforced as part of the protocol and works as follows (assume the client has successfully connected and subscribed to a topic):

1. client indicates they are ready to receive messages
2. NSQ sends a message and temporarily stores the data locally (in the event of re-queue or timeout)
3. client replies FIN (finish) or REQ (re-queue) indicating success or failure respectively. If client does not reply NSQ will timeout after a configurable duration and automatically re-queue the message)

### Bounded Memory Footprint

The disk-backed queue is designed to survive unclean restarts (although messages might be delivered twice).

Also, related to message delivery guarantees, clean shutdowns (by sending a nsqd process the TERM signal) safely persist the messages currently in memory, in-flight, deferred, and in various internal buffers.

Note, a topic/channel whose name ends in the string #ephemeral will not be buffered to disk and will instead drop messages after passing the mem-queue-size. These ephemeral channels will also disappear after its last client disconnects.

### Efficiency

For the data protocol, we made a key design decision that maximizes performance and throughput by pushing data to the client instead of waiting for it to pull. This concept, which we call *RDY* state, is essentially a form of client-side flow control.

When a client connects to *nsqd* and subscribes to a channel it is placed in a *RDY* state of 0. This means that no messages will be sent to the client. When a client is ready to receive messages it sends a command that updates its *RDY* state to some # it is prepared to handle, say 100. Without any additional commands, 100 messages will be pushed to the client as they are available (each time decrementing the server-side *RDY* count for that client).

Client libraries are designed to send a command to update *RDY* count when it reaches ~25% of the configurable *max-in-flight* setting (and properly account for connections to multiple nsqd instances, dividing appropriately).

### Go

Regarding *NSQ*, Go channels (not to be confused with *NSQ* channels) and the language’s built in concurrency features are a perfect fit for the internal workings of *nsqd*.

## INTERNALS

NSQ is composed of 3 daemons:

* *nsqd* is the daemon that receives, queues, and delivers messages to clients.
* *nsqlookupd* is the daemon that manages topology information and provides an eventually * consistent discovery service.
* *nsqadmin* is a web UI to introspect the cluster in realtime (and perform various administrative tasks).

Data flow in *NSQ* is modeled as a tree of streams and consumers. A topic is a distinct stream of data. A channel is a logical grouping of consumers subscribed to a given topic.

### Topics and Channels

#### Topics

Go’s channels (henceforth referred to as “go-chan” for disambiguation) are a natural way to express queues, thus an NSQ topic/channel, at its core, is just a buffered go-chan of *Message* pointers. The size of the buffer is equal to the _--mem-queue-size_ configuration parameter.

After reading data off the wire, the act of publishing a message to a topic involves:

1. instantiation of a *Message* struct (and allocation of the message body _[]byte_)
2. read-lock to get the *Topic*
3. read-lock to check for the ability to publish
4. send on a buffered go-chan

Each topic maintains 3 primary goroutines.

* The first one, called *router*, is responsible for reading newly published messages off the incoming go-chan and storing them in a queue (memory or disk).
* The second one, called *messagePump*, is responsible for copying and pushing messages to channels as described above.
* The third is responsible for *DiskQueue* IO.

#### Channels

Channels share the underlying goal of exposing a single input and single output go-chan (to abstract away the fact that, internally, messages might be in memory or on disk).

Each channel maintains 2 time-ordered priority queues responsible for deferred and in-flight message timeouts (and 2 accompanying goroutines for monitoring them).

Parallelization is improved by managing a per-channel data structure, rather than relying on the Go runtime’s global timer scheduler.

### Backend/DiskQueue

Since the memory queue is just a go-chan, it’s trivial to route messages to memory first, if possible, then fallback to disk.

### Reducing GC Pressure

Use the *testing* package and *go test -benchmem* to benchmark hot code paths. It profiles the number of allocations per iteration (and benchmark runs can be compared with *benchcmp*).

Build using *go build -gcflags -m*, which outputs the result of escape analysis.

With that in mind, the following optimizations proved useful for nsqd:

* Avoid []byte to string conversions.
* Re-use buffers or objects (and someday possibly sync.Pool aka issue 4720).
* Pre-allocate slices (specify capacity in make) and always know the number and size of items over the wire.
* Apply sane limits to various configurable dials (such as message size).
* Avoid boxing (use of interface{}) or unnecessary wrapper types (like a struct for a * “multiple value” go-chan).
* Avoid the use of defer in hot code paths (it allocates).

### TCP Protocol

The NSQ TCP protocol is a shining example of a section where these GC optimization concepts are utilized to great effect.

### HTTP

Its simplicity belies its power, as one of the most interesting aspects of Go’s HTTP tool-chest is the wide range of debugging capabilities it supports. The *net/http/pprof* package integrates directly with the native HTTP server, exposing endpoints to retrieve CPU, heap, goroutine, and OS thread profiles. These can be targeted directly from the go tool:

    $ go tool pprof http://127.0.0.1:4151/debug/pprof/profile

This is a tremendously valuable for debugging and profiling a running process!

### Dependencies

There are two main schools of thought:

1. Vendoring: copy dependencies at the correct revision into your application’s repo and modify your import paths to reference the local copy.
2. Virtual Env: list the revisions of dependencies you require and at build time, produce a pristine GOPATH environment containing those pinned dependencies.

NSQ uses *gpm* to provide support for (2) above.

It works by recording your dependencies in a *Godeps* file, which we later use to construct a GOPATH environment.

### Testing

A *Context* struct is passed around that contains configuration metadata and a reference to the parent *nsqd*. All references to global state were replaced with this local *Context*, allowing children (topics, channels, protocol handlers, etc.) to safely access this data and making it more reliable to test.

### Robustness

#### Heartbeats and Timeouts

Periodically, *nsqd* will send a heartbeat over the connection. The client can configure the interval between heartbeats but *nsqd* expects a response **before it sends the next one**.

To guarantee progress, all network IO is bound with deadlines relative to the configured heartbeat interval.

When a fatal error is detected the client connection is forcibly closed. In-flight messages are timed out and re-queued for delivery to another consumer. Finally, the error is logged and various internal metrics are incremented.

#### Managing Goroutines

WaitGroups

    type WaitGroupWrapper struct {
        sync.WaitGroup
    }

    func (w *WaitGroupWrapper) Wrap(cb func()) {
        w.Add(1)
        go func() {
            cb()
            w.Done()
        }()
    }

    // can be used as follows:
    wg := WaitGroupWrapper{}
    wg.Wrap(func() { n.idPump() })
    ...
    wg.Wait()

Exit Signaling

* The easiest way to trigger an event in multiple child goroutines is to provide a single go-chan that you close when ready. All pending receives on that go-chan will activate, rather than having to send a separate signal to each goroutine.

        func work() {
            exitChan := make(chan int)
            go task1(exitChan)
            go task2(exitChan)
            time.Sleep(5 * time.Second)
            close(exitChan)
        }
        func task1(exitChan chan int) {
            <-exitChan
            log.Printf("task1 exiting")
        }

        func task2(exitChan chan int) {
            <-exitChan
            log.Printf("task2 exiting")
        }

Synchronizing Exit

* Ideally the goroutine responsible for sending on a go-chan should also be responsible for closing it.
* If messages cannot be lost, ensure that pertinent go-chans are emptied (especially unbuffered ones!) to guarantee senders can make progress.
* Alternatively, if a message is no longer relevant, sends on a single go-chan should be converted to a _select_ with the addition of an exit signal (as discussed above) to guarantee progress.
* The general order should be:

    * Stop accepting new connections (close listeners)
    * Signal exit to child goroutines (see above)
    * Wait on _WaitGroup_ for goroutine exit (see above)
    * Recover buffered data
    * Flush anything left to disk

Logging

*nsqd* leans towards the side of having more information in the logs when a fault occurs rather than trying to reduce chattiness at the expense of usefulness.

# COMPONENTS

## NSQD

## NSQLOOKUPD

## NSQADMIN

## UTILITIES

### nsq_stat

Polls /stats for all the producers of the specified topic/channel and displays aggregate stats.

### nsq_tail

Consumes the specified topic/channel and writes to stdout (in the spirit of tail(1)).

### nsq_to_file

Consumes the specified topic/channel and writes out to a newline delimited file, optionally rolling and/or compressing the file.

### nsq_to_http

Consumes the specified topic/channel and performs HTTP requests (GET/POST) to the specified endpoints.

### nsq_to_nsq

Consumes the specified topic/channel and re-publishes the messages to destination nsqd via TCP.

### to_nsq

Takes a stdin stream and splits on newlines (default) for re-publishing to destination nsqd via TCP.

# CLIENTS

## CLIENT LIBRARIES

## BUILDING CLIENT LIBRARIES

NSQ’s design pushes a lot of responsibility onto client libraries in order to maintain overall cluster robustness and performance.

### Configuration

A consumer subscribes to a topic on a channel over a TCP connection to *nsqd* instance(s). You can only subscribe to one topic per connection so multiple topic consumption needs to be structured accordingly.

Most importantly, the client library should support some method of configuring callback handlers for message processing. The signature of these callbacks should be simple, typically accepting a single parameter (an instance of a “message object”).

### Discovery

When a consumer uses *nsqlookupd* for discovery, the client library should manage the process of polling all *nsqlookupd* instances for an up-to-date set of *nsqd* providing the topic in question, and should manage the connections to those *nsqd*.

A periodic timer should be used to repeatedly poll the configured *nsqlookupd* so that consumers will automatically discover new *nsqd*. The client library should automatically initiate connections to all newly discovered instances.

### Connection Handling

Once a consumer has an *nsqd* to connect to (via discovery or manual configuration), it should open a TCP connection to *broadcast_address:port*. A **separate** TCP connection should be made to each nsqd for each topic the consumer wants to subscribe to.

When connecting to an *nsqd* instance, the client library should send the following data, in order:

* the magic identifier
* an IDENTIFY command (and payload) and read/verify response
* a SUB command (specifying desired topic) and read/verify response
* an initial RDY count of 1 (see RDY State).

Client libraries should automatically handle reconnection as follows:

* If the consumer is configured with a specific list of *nsqd* instances, reconnection should be handled by delaying the retry attempt in an exponential backoff manner (i.e. try to reconnect in 8s, 16s, 32s, etc., up to a max).
* If the consumer is configured to discover instances via *nsqlookupd*, reconnection should be handled automatically based on the polling interval (i.e. if a consumer disconnects from an *nsqd*, the client library should only attempt to reconnect if that instance is discovered by a subsequent nsqlookupd polling round). This ensures that consumers can learn about *nsqd* that are introduced to the topology and ones that are removed (or failed).

### Feature Negotiation

The IDENTIFY command can be used to set *nsqd* side metadata, modify client settings, and negotiate features. It satisfies two needs:

* In certain cases a client would like to modify how *nsqd* interacts with it (such as modifying a client’s heartbeat interval and enabling compression, TLS, output buffering, etc. - for a complete list see the spec)
* _nsqd_ responds to the IDENTIFY command with a JSON payload that includes important server side configuration values that the client should respect while interacting with the instance.

### Data Flow and Heartbeats

Once a consumer is in a subscribed state, data flow in the NSQ protocol is asynchronous.

Due to their asynchronous nature, it would take a bit of extra state tracking in order to correlate protocol errors with the commands that generated them. Instead, we took the “fail fast” approach so the overwhelming majority of protocol-level error handling is fatal. This means that if the client sends an invalid command (or gets itself into an invalid state) the *nsqd* instance it’s connected to will protect itself (and the system) by forcibly closing the connection (and, if possible, sending an error to the client).

### Message Handling

When the IO loop unpacks a data frame containing a message, it should route that message to the configured handler for processing.

The sending *nsqd* expects to receive a reply within its configured message timeout (default: 60 seconds). There are a few possible scenarios:

* The handler indicates that the message was processed successfully.
* The handler indicates that the message processing was unsuccessful.
* The handler decides that it needs more time to process the message.
* The in-flight timeout expires and nsqd automatically re-queues the message.

In the first 3 cases, the client library should send the appropriate command on the consumer’s behalf (FIN, REQ, and TOUCH respectively).

### RDY State

Because messages are pushed from *nsqd* to consumers we needed a way to manage the flow of data in user-land rather than relying on low-level TCP semantics. A consumer’s RDY state is NSQ’s flow control mechanism.

Client libraries have a few responsibilities:

* bootstrap and evenly distribute the configured *max_in_flight* to all connections.
* never allow the aggregate sum of RDY counts for all connections (*total_rdy_count*) to exceed the configured *max_in_flight*.
* never exceed the per connection nsqd configured *max_rdy_count*.
* expose an API method to reliably indicate message flow starvation.

### Backoff

Backoff should be implemented by sending RDY 0 to the appropriate *nsqd*, stopping message flow. The duration of time to remain in this state should be calculated based on the number of repeated failures (exponential). Similarly, successful processing should reduce this duration until the reader is no longer in a backoff state.

While a reader is in a backoff state, after the timeout expires, the client library should only ever send RDY 1 regardless of *max_in_flight*.

### Encryption/Compression

 Both *Snappy* and DEFLATE are supported for compression.

 It’s very important that you either prevent buffering until you’ve finished negotiating encryption/compression, or make sure to take care to read-to-empty as you negotiate features.

 ### Bringing It All Together

 ## TCP PROTOCOL SPEC

 After connecting, a client must send a 4-byte “magic” identifier indicating what version of the protocol they will be communicating (upgrades made easy).

 The V2 protocol also features client heartbeats. Every 30s (default but configurable), nsqd will send a *\_heartbeat\_* response and expect a command in return. If the client is idle, send *NOP*. After 2 unanswered *\_heartbeat\_* responses, nsqd will timeout and forcefully close a client connection that it has not heard from. The IDENTIFY command may be used to change/disable this behavior.

 Unless stated otherwise, all binary sizes/integers on the wire are network byte order (ie. big endian)

 ### Commands

 * IDENTIFY: Update client metadata on the server and negotiate features
 * SUB: Subscribe to a topic/channel
 * PUB: Publish a message to a topic
 * MPUB: Publish multiple messages to a topic (atomically)
 * DPUB: Publish a deferred message to a topic
 * RDY: Update RDY state (indicate you are ready to receive N messages)
 * FIN: Finish a message (indicate successful processing)
 * REQ: Re-queue a message (indicate failure to process)
 * TOUCH: Reset the timeout for an in-flight message
 * CLS: Cleanly close your connection (no more messages are sent)
 * NOP: No-op
 * AUTH: When *nsqd* receives an AUTH command it delegates responsibility to the configured _--auth-http-address_ by performing an HTTP request with client metadata in the form of query parameters
 
 ### Data Format

 Data is streamed asynchronously to the client and framed in order to support the various reply bodies.

 # DEPLOYMENT

 ## INSTALLING

 NSQ uses *dep* to manage dependencies and produce reliable builds. Using *dep* is the preferred method when compiling from source.

    $ git clone https://github.com/nsqio/nsq $GOPATH/src/github.com/nsqio/nsq
    $ cd $GOPATH/src/github.com/nsqio/nsq
    $ dep ensure

Testing

    $ ./test.sh

 ## PRODUCTION CONFIGURATION

 ### nsqd

 _--mem-queue-size_ adjusts the number of messages queued in memory per topic/channel. Messages over that watermark are transparently written to disk, defined by _--data-path_.

 *nsqd* will need to be configured with *nsqlookupd* addresses. Specify _--lookupd-tcp-address_ options for each instance.

 *nsqd* can be configured to push data to statsd by specifying _--statsd-address_.

 ### nsqlookupd

 It maintains no persistent state and does not need to coordinate with any other nsqlookupd instances to satisfy a query.

 Our recommendation is to run a cluster of at least 3 per datacenter.

 ### nsqadmin

 ### Monitoring

 ## TOPOLOGY PATTERNS

 ### Metrics Collection

 To perform the work of writing into your metrics system asynchronously - that is, place the data in some sort of local queue and write into your downstream system via some other process (consuming that queue).

### Persistence

Archiving an NSQ topic is such a common pattern that we built a utility, *nsq_to_file*, packaged with NSQ, that does exactly what you need.

### Distributed Systems

## DOCKER


