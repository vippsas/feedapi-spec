# FeedAPI specification

## Overview

This spec details an HTTP API that:

1) Allows a service to publish events to an arbitrary number of subscribers
   without going through an event broker (such as Kafa or Azure Event Hub).

2) Can work as a supplement to a broker. It uses the same basic
   model and services can offer it *in addition* to publishing on a broker.

3) Even for consumers opting to consume through brokers instead, the
   API is a convenient alternative if one needs to backfill older data.

A parallel is ZeroMQ. In 2007 ZeroMQ launched as a framework and
series of patterns for doing message passing "better" without broker;
it turned out that for message passing, many applications become more
robust and scalable simply by removing the broker.

This spec attempts the same for Kafka-style event channels.

## Goal of proposal

Async events through a broker has some benefits:

* Events defined in standard contracts, culture of centralizing
  events as source of truth

* Events can be transmitted without producer depending on consumer being alive
  and receiving, and possibly needing to implement retries (as callbacks require)

There are also some problems:

* With a broker you don't have consistency; you don't know that if you did
  something with a service the effect will be on the event feed the moment
  you get a response.

* A broker adds a latency that is not directly controllable and
  may in certain "tight" situations add operational risk.

* A broker adds a extra piece of infra that must be up, which may be
  a liability on a critical path. Abusing a famous quote: "The only
  problem that cannot be solved by adding another moving piece is too
  many moving pieces."

The goal of this spec is to keep most of the first advantages, while eliminating
these latter problems.

The scope is primarily a backend producer which sits on top of a
database and first commits new state to the database, before
publishing events of what has happened and already been committed to
a broker. We are aware that brokers can have other usecases and
this spec may be less relevant for such usecases.

## Q & A

**What if the consumer goes down?**

The publisher doesn't know about consumers in this spec. The pattern
allow consumers to resume at any point, just like with a broker.

**What if the publisher goes down?**

In this situation a broker would seem to give higher uptime, *but, consider:*

Scenario 1: The consumer is OK with some latency and consumes
irregularly/lazily/in a batch.  In such a scenario, it may often be OK
that events are delayed a bit more if the producer is down.

Scenario 2: The consumer needs minimal latency. In this situation the consumer
hogs the feed and reads events as soon as they are published.
Now... when the producer goes down, new events will also stop! A broker
actually doesn't help in this scenario. A very few events may have been caused
by the producer but not read by the consumer, but it is also the case that a few
events may have been caused by the producer but not yet been written to Event Hub
when the publisher went down. So this is the same as with a broker.

**What about load on the publisher?**

In situations where the number of consumers is very high and all are reading a lot
of data this is a legitimate advantage of a broker as it helps distribute
the load of reading events.

However, in general the load needed to simply serve out events to a few
other services that need them is a minor part of the load of a typical
backend service. Due to the entirely stateless nature of
the FeedAPI protocol, it is easy enough to cache recent event history
in memory.

**Why not an existing API spec?**

If you know about one we'd love to use it instead of NIH.

The closest thing we found is Atom / RSS. But these are really made
for other usecases and in particular don't have features for sharding
or consumer groups. We would need to extend them anyway -- at that
point it is better to make something a lot simpler than those
standards.

## Protocol TL;DR

The TL;DR of the protocol is that first
the consumer calls the discovery endpoint
to list partitions:

```
GET https://myservice/myfeed
```

Response:
```json
{
    "token": "1",  // change this if you change number of partitions
    "partitions": [
      {
        "id": "0",
        // Extra options here used when changing number of partitions / re-scale.
        // Publishers who don't change number of partitions don't need to worry about this.
      },
    ],
    "exactlyOnce": true,  // false if you may re-publish over FeedAPI
}
```

Then, the next call is to the "/events" endpoint below the discovery
endpoint to consume events:

```
GET https://myservice/myfeed/events?token=1&partition=0&cursor=_first
```

Response is in NDJSON-format:

```json
{"data": {...}}
{"cursor": "2423423423"}
{"data": {...}}
{"data": {...}}
{"data": {...}}
{"cursor": "2423423424"}
```

## Authorization

Authorization is in general recommended to use a Bearer token in the Authorization
header; although the exact scheme is out of the scope of this specification.

Standard HTTP 401 and 403 response codes should be used.


## Discovery endpoint

A simple argument-less `GET` returns information about the feed.
```
GET https://service/myfeed
```
Example response:
```json
{
    "token": "xf3af",
    "partitions": [
      {
        "id": "0",
        "closed": true
      },
      {
        "id": "16000",
        "startsAfterPartition": "0"
      }
    ],
    "exactlyOnce": true,
}
```

#### Partitioning-related fields

`token`: Arbitrary string. Consumers must pass the same token
 back in the Fetch call. See [Partitions](#partitions) below for more information.
 
`partitions`: Lists the partitions that should be consumed.
See [Partitions](#partitions) below for more information. Fields
of each partition:

* `id`: String, but only allowed values is integers in the range `[0..32767]`. The ID is a 16-bit
    integer because it may be convenient for the consumer to put this
    in the primary key in the destination database, and because the
    producer is in a position to easily manage a restricted ID space.
    * This is an integer for compatibility with version 1; but
      in general in JSON it is good form that IDs are strings even if
      they are integers.

  * `closed`: If set, it is guaranteed that no new events will appear on this feed.

  * `startsAfterPartition` and `cursorFromPartitions`: Optional. Used
    to change the number of [partitions](#partitions).

#### Other fields
* `exactlyOnce`: True if the publisher guarantees that the same event
  `id` will never be seen twice by a consumer; i.e., exactly-once consumption
  patterns may be used. If this is set to `false`, events are seen at-least-once
  and the consumer has to be idempotent / de-duplicate events.


More flags are expected to be added later, in order
to develop the protocol in a backwards-compatible
manner.


## Events endpoint

### Request
A fetch is done to the same URL as the discovery with
the addition of `/events`.
```
GET https://service/myfeed/events?token=xaf32&partition=16000&cursor=f1ceaa92eb7c11eda43d6fb319691265
```

Arguments:

`token`: Pass pack the string received in the discovery endpoint.
 
`partition`: ID of partition (see discovery call)

`cursor`: The place in the feed to start reading. Special cursors `_first` and `_last`
  can be used for each end of the feed as initial values.

`pagesizehint` (optional): How many events to return.
  As indicated by the argument name, this is a *hint*,
  and consumers should be prepared to receive fewer or
  more events than requested. If not specified,
  an implementation-dependent default is used.

`event-types` (optional$^{*}$): If specified, indicates the subset of event types that
  should be returned by the stream. Any events that do not match will be
  skipped. Types should be delimited by `;` and implementations should be case-insensitive.

  ```http
  GET https://service/myfeed/events?event-types=ewallet.preparedepositrequested;ewallet.confirmdepositrequested&token=xaf32&partition=16000&cursor=f1ceaa92eb7c11eda43d6fb319691265
  ```

  \* Optional parameters are optional, not only for the clients, but also for
  the implementing servers, e.g. if you request filtering with an `event-types`
  parameter, the server can disregard it and still return the full stream.

### Response
The response is in the NDJSON format; each line (separated by `\n`) represents
a protocol message from the publisher to consumer. The reason for this format
is to be able to *stream* events, which would not be as easy with a more
usual JSON response using a JSON list.
```
{"data": {/*...payload... */}}
{"cursor": "2394r7a98a7342qw34r2412rwa"}
{"data": {/*...payload... */}}
{"data": {/*...payload... */}}
{"data": {/*...payload... */}}
{"cursor": "24rw3afawowraqwl2ijur3lakj"}
```

There are 2 kinds of commands:
* If `data` is present, the line contains an individual event
* If `cursor` is present, the line is a checkpoint.

Consumers should gracefully handle not only new unknown fields
in the JSON objects (as is usual), but also ignore entirely new
commands (i.e., lines not containing either `data` nor `cursor`).

#### Event command

The event payload contained in the `data` key. This can *either* be
a string (e.g., a binary representation) *or* an embedded JSON document
in any format. (The value being JSON list, number or bool is not supported.)

The payload can be either be a JSON sub-struct, or a string for any other kind of
data (such as binary).
Note that `{"data": ...}` is the FeedAPI *envelope* and not part of the event;
for instance, if one is following the Cloudevents spec, then a line from FeedAPI
will be
```json
{"data": {"id":  "3q432e", "data": {...}, "time": "..."}}
```
So, one `data` within another `data`.

#### Checkpoint command

Every response **must** always return at least one `cursor`, even if there
are no new `data` elements available. Consumers should update cursors even if there
are no new events. For instance, the publisher could be filtering away
a lot of internal events to produce the external feed, and being able to
make progress also in the case of no new (external) events is important.

Some publishers may emit a checkpoint for every event, others may
list several events between checkpoint.

In general, events transported over FeedAPI follow the
*partitioned log* event communication model of Kafka, Event Hub, etc;
i.e., events should arrive in-order and should be processed in the
order they are received in the response.

*However*, there is no guarantee that two requests
from a given `cursor` will produce an identical response each time.
This is because while from the perspective of the API we are consuming
a single partition, the server *could* be emulating this behaviour
from a larger set of real physical partitions, and it may be that
returning data from the underlying partitions happen in a non-deterministic
manner to minimize latency. What is important is:

* The client should consider events ordered in the order they arrive in
* Any ordering between events *that matters* should be reproducible
  (typically, events for a single aggregate (entity/object)).

In other words, the consumer can *assume* that the cursor is a simple
cursor into an ordered list, but publishers are free to deviate from this
model and give non-deterministic responses as long as *ordering that matters*
is preserved.


## Partitions

FeedAPI supports a dynamic number of partitions, and several partition
splitting schemes. While this puts some conceptual load on the client,
we believe the benefits for the publisher outweighs this:

* The client's implementation anyway need to run a singleton worker per
  partition and track cursors. Therefore, a library
  is likely to be used client side anyway.

* The added complexity to the client is merely in adding some lines
  of low level logic. Missing these features would lead to a lot more
  work for publishers that need to move between databases or scale
  the number of partitions.

In order to support a variety of databases and scaling approaches
there are several different features; the publisher decides which one
to use and clients should support all of them.

The discovery command's reference manual is below, but for now,
this is the relevant section for partitions:
```json
{
  "token": "afx53",
  "partitions": [
    {
      "id": "0",
      "closed": true
    },
    {
      "id": "16232",
      "startsAfterParent": "0"
    },
    {
      "id": "24223",
      "splitFromAncestors": ["24001", "24100"]
    }
  ]
}
```
#### Starting new partitions, and the token mechanism

A new partition can appear in the list at any time. The token
mechanism is used to make sure clients pick these up and refresh
the list of partitions when needed.

The consumer reads the `token` from the discovery response
and passes it in to all fetch commands. If the publisher wishes
to trigger consumers to discover new partitions (or other relevant
changes to the consumption process), it may change the token
provided in the discovery command, *and* return HTTP error 409 Conflict on the
events fetch command. Consumers should respond to 409 Conflict by
re-starting from the discovery phase.

PS: It is not legal for the publisher to only return 409 Conflict for
some partitions and not others over longer periods of time;
all partitions should be blocked at
*approximately* the same point time. This allows consumers to use this
as a signal for partition processes using old tokens to shut down.

#### Stopping partitions

An existing partition that will never again receive writes
can be marked as `"closed": true`. Consumers should still
consume it from the start until the end for re-constitution, but they will
know that no *new* events will appear on it.

Once a partition has been marked as closed, it is forbidden to ever
again add new events to that partition.

#### Method 1 to change partition count: New events on new partitions

The first method to increase the number of partitions is to *close* the
parent partition, and open a number of child partitions to replace it.
For instance, in this case partition 0 was split into partitions 1 and 2:

```json
{
  "partitions": [
    {
      "id": "0",
      "closed": true
    },
    {
      "id": "1",
      "startsAfterPartition": "0"
    },
    {
      "id": "2",
      "startsAfterPartition": "0"
    }
  ]
}
```
It is not legal to set `startsAfterPartition` to point to a non-closed partition.

A consumer who does not care about event ordering can simply read all
of these partitions in parallel. But, if the consumer cares about ordering,
it should:

* *First* read all events from partition 0.
* *Then*, when that partition is fully consumed, it continues with the
  children.

The idea is that a given aggregates (object, entity), to which the order
of events is important, will have its ordered events on the parent partition
0 for the time period before the split, and on one of the child partitions
(either 1 or 2) after the split. The parent partition sticks around indefinitely
to serve the history from before the split.

This method is the simplest one to implement oneself on top of e.g. SQL.

#### Method 2 to change partition count: Re-distribute entire history

This is the method used by Azure Cosmos DB. In this case, when a partition
is split to increase the number of partitions, *also the history* is
distributed. In this case we have this
situation before the split:
```json
{
  "partitions": [
    {
      "id": "0",
    },
  ]
}
```
Then, *after* the split, we have this:
```json
{
  "partitions": [
    {
      "id": "1",
      "cursorFromPartitions": ["0"]
    },
    {
      "id": "2",
      "cursorFromPartitions": ["0"]
    }
  ]
}
```
For a given aggregate (object, entity), before the split all the events
lived on partition 0. After the split, the *full history* of the given
aggregate is *either* on partition 1, *or* partition 2. Partition
0 can no longer be consumed.

The key now is: *A cursor that was valid on an ancestor is also valid
on the child*. During the split, the consumer has a cursor for partition
0, but partition 0 has disappeared from the list. This cursor can now
be used as the start partition of *both* partition 1 and 2 in order to
continue ordered event consumption.

Note that `cursorFromPartitions` is a list. It lists *all* partitions
one may transfer a cursor from. For instance this can be the path through
an inheritance tree to the root (first parent, then grand-parent, and so on).

## Versioning

[ZeroEventHub](https://github.com/vippsas/zeroeventhub/blob/main/SPEC.md) is considered "version 1" of this
protocol, and this spec is considered "version 2".

FeedAPI client libraries are expected to support *both* versions for
a longer period of time.

#### Deprecation of ZeroEventHub v1 features

* The multiple-partition-per-call feature of version 1 is unused and will remain
  unused. I.e., `cursor0=..&cursor1=..` has never been used and should
  never be used.

* The headers feature is similarly deprecated.

#### Protocol negotiation

FeedAPI clients should start with a simple `GET` to the FeedAPI endpoint without any
arguments. For version 1 (ZeroEventHub) this should result in a 400
error; for version 2 the result will be the discovery endpoint
JSON payload documented below.

In the events fetch endpoint, the presence of the argument `?token=`
indicates that the client is using FeedAPI version 2.
