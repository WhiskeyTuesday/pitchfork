# The Philosophy Behind pitchfork

## Avoid Complexity

Instead of attempting to be efficient at serving slow clients, pitchfork
relies on a buffering reverse proxy to efficiently deal with slow
clients.

## Improved Performance Through Reverse Proxying

By acting as a buffer to shield unicorn from slow I/O, a reverse proxy
will inevitably incur overhead in the form of extra data copies.
However, as I/O within a local network is fast (and faster still
with local sockets), this overhead is negligible for the vast majority
of HTTP requests and responses.

The ideal reverse proxy complements the weaknesses of pitchfork.
A reverse proxy for pitchfork should meet the following requirements:

1. It should fully buffer all HTTP requests (and large responses).
   Each request should be "corked" in the reverse proxy and sent
   as fast as possible to the backend unicorn processes.  This is
   the most important feature to look for when choosing a
   reverse proxy for pitchfork.

2. It should handle SSL/TLS termination. Requests should arrive
   decrypted to `pitchfork`. Reverse proxy can do this much more
   efficiently. If you don't trust your local network enough to
   make unencrypted traffic go throught it, you can have a reverse
   proxy on the same server than `pitchfork` to handle decryption.

3. It should handle HTTP/2 or HTTP/3 termination. Newer HTTP protocols
   do not provide any feature or improvements that are useful or even desirable
   for transactional HTTP applications. Your reverse proxy or load balancer
   should handle the HTTP/2 or HTTP/3 protocol with the client, but forward
   requests to `pitchfork` as HTTP/1.1.

4. It should efficiently manage persistent connections (and
   pipelining) to slow clients.

5. It should not be "sticky". Even if the client has a persistent
   connection, every request made as part of that peristent connection
   should be load balanced individually.

6. It should (optionally) serve static files. If you have static
   files on your site (especially large ones), they are far more
   efficiently served with as few data copies as possible (e.g. with
   sendfile() to completely avoid copying the data to userspace).

Suitable options include `nginx`, `caddy` and likely several others.

## Threads and Events Are Not Suited For Transactional Web Applications

pitchfork uses a preforking worker model with blocking I/O.
Our processing model is the antithesis of processing models using threads or
non-blocking I/O with events or fibers.

It is only meant to serve fast, transactional HTTP/1.1 applications.
These applications rarely if ever spend more than half their time on IOs, and
any remote API call made by the application should either have a strict SLA
and timeout, or be deferred to a background job processing system.

As such, when they are ran in a threaded or fiber processing model, they suffer
from GVL contention, which hurts latency.

WebSocket, Server-Sent Events and application that mainly act as a light proxy
to another service have radically different performance profiles and requirements,
and shouldn't be handled by the same process. It's preferable to host them on
the side with a threaded or evented processing model (`falcon`, `puma`, etc).

No processing model can efficiently handle both types of workload. Use
the right tool for the right job.

## Application Concurrency != Network Concurrency

Performance is asymmetric across the different subsystems of the machine
and parts of the network.  CPUs and main memory can process gigabytes of
data in a second; clients on the Internet are usually only capable of a
tiny fraction of that.  unicorn deployments should avoid dealing with
slow clients directly and instead rely on a reverse proxy to shield it
from the effects of slow I/O.

## Worse Is Better

Requirements and scope for applications change frequently and
drastically.  Thus languages like Ruby and frameworks like Rails were
built to give developers fewer things to worry about in the face of
rapid change.

On the other hand, stable protocols which host your applications (HTTP
and TCP) only change rarely.  This is why we recommend you NOT tie your
rapidly-changing application logic directly into the processes that deal
with the stable outside world.  Instead, use HTTP as a common RPC
protocol to communicate between your frontend and backend.

In short: separate your concerns.

Of course a theoretical "perfect" solution would combine the pieces
and _maybe_ give you better performance at the end of the day, but
that is not the Unix way.

## Just Worse in Some Cases

unicorn is not suited for all applications.  unicorn is optimized for
applications that are CPU/memory/disk intensive and spend little time
waiting on external resources (e.g. a database server or external API).

unicorn is highly inefficient for Comet/reverse-HTTP/push applications
where the HTTP connection spends a large amount of time idle.
Nevertheless, the ease of troubleshooting, debugging, and management of
unicorn may still outweigh the drawbacks for these applications.
