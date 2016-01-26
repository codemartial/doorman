# How It Works

## Introduction

The purpose of the Doorman system is to apportion and distribute capacity to clients based on some definition of fairness. The capacity a client gets for a resource depends on four things:

- The configured maximum capacity for the resource (world-wide).
- The capacity need (wants) of this client.
- The capacity needs (wants) of all other clients on the planet.
- The exact algorithm used (as defined by the configuration) to apportion the capacity among all the clients.

The Doorman master server remembers all clients that currently have capacity and whenever a client asks for capacity it inserts the clients request into its memory and runs the algorithm to figure out what this client should get.

## Lease length and refresh interval

Doorman only gives out capacity for a limited amount of time, in the form of leases. Each capacity grant comes with a lease length: The client is guaranteed that amount of capacity for the duration of the lease. A typical lease length is five minutes. On top of the lease length the Doorman server also returns a refresh interval. This is the interval after which the client is expected to check back in to get a new lease. A typical refresh interval is five seconds.

Note: The Doorman system is *cooperative*. The clients are expected to honour the capacity grant, the lease length, and the refresh interval. The system provides no protection against misbehaving clients.

In the normal operation of the system the clients all check in regularly with the server to refresh their capacity. The server knows of all clients and their resource needs, and on every request makes the best possible apportionment of the capacity. For optimization purposes (reduce qps on the Doorman server) the client code does bulk refreshes for all resources whenever it sends out a request to the Doorman server. This means that under specific circumstances (for instance when registering a new resource) a resource might get its capacity refreshed a bit sooner than expected.

The Doorman configuration specifies with Algorithm should be used to distribute capacity among all clients. The page on Algorithms explains which algorithms currently are available and how that algorithm apportions capacity.

The two parameters lease\_length and refresh\_interval optimize a number of different behaviors of the system:

- The load on the Doorman server.
- The speed with which the system converges as resource needs change and clients appear and disappear.
- How the system deals with the Doorman server being unreachable or slow.

## When the Doorman server goes unreachable and comes back

When a client cannot reach the Doorman server the following happens:

- The client misses one or more refresh intervals. This does not matter much for the client other than that the capacity the client has is not adjusted for potentially changed resource needs.
- When the Doorman server is unavailable for a longer period of time leases expire and the resources revert to their configured safe capacity. This can be either:
  * -1, meaning an unbounded (infinite) rate limit, or
  * 0, meaning that all access to the resource is blocked, or
a positive number
- As soon as the Doorman server becomes reachable again the clients will resume requesting capacity.

Note: Doorman uses multiple servers in different clusters and a master election procedure to determine the current master.

Doorman does not share or store its internal database. That means that when a Doorman server becomes the master it starts with an empty repository of clients and outstanding leases. However this is not as problematic as it seems, because once the server is available all clients will start calling it to refresh their leases. Since the Doorman server knows that it does not have enough information to run its algorithms it will simply return the currently assigned capacity, or zero if it is a request from a client which currently does not have capacity for the resource. The server knows the currently assigned capacity because clients helpfully include it in every GetCapacity RPC. This phase of the server is known as *learning mode*.

During the learning mode of a resource every request will be answered with a new lease for the same capacity the client currently has. Practically speaking after a couple of refresh intervals the server can be reasonably sure that it has been contacted by every existing client out there. However for reasons of safety the default learning mode duration is the same as the lease length. This decision ensures that when the learning mode duration expires we can be sure that there are no leases out there that we don't know about (because these would have expired by then). If you want the system to converge faster after a Doorman master election you can explicitly configure a learning\_mode\_duration in the resource template (see the page on the Configuration of the system for more information).

## Who wants what?

Doorman requires the clients to inform it of the desired capacity (the so-called *wants*). If you are using the low-level Doorman client you need to figure out your capacity need and call the appropriate methods to make sure that the client library requests that amount of capacity during its refresh cycle. However if you use the rate limiter objects provided by the Doorman clients the desired capacity is determined automatically by observing the behavior of the threads that want to access the resource. This automatic wants determination uses a moving average to smoothen out any spikes.