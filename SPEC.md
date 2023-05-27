# FeedAPI specification

## Context

For now, this documentation is a working document for people
very familiar with ZeroEventHub; therefore no attempt is made
at explaining concepts.

## Versioning

[ZeroEventHub](https://github.com/vippsas/zeroeventhub/blob/main/SPEC.md) is considered "version 1" of this
protocol, and this spec is considered "version 2".

FeedAPI server and client libraries are expected to support *both* versions for
a longer period of time; with the exception that the multiple-partition-per-call
feature of version 1 is unused and will remain unused. I.e., `cursor0=..&cursor1=..`
has never been used and should never be used.

#### Protocol negotiation

FeedAPI clients should start with a simple `GET` to the FeedAPI endpoint without any
arguments. For version 1 (ZeroEventHub) this should result in a 400
error (ErrNoCursors); for version 2 the result will be the discovery endpoint
JSON payload documented below.

In the events fetch endpoint, the presence of the argument `?partition=`
indicates that the client is using FeedAPI version 2.


## Endpoint name

FeedAPI is using RPC over HTTP. All communication for a given
event feed happens on a *single* request path.

I.e., the protocol will not use elements of the request path,
e.g., to specify partition or version or similar.

Why?

* This is how version 1 works. It will be simpler to keep using
  the same endpoints for version 1 of the protocol and version 2.
* Easier to develop the protocol in the server libraries, without having
  changes to also have to propagate to the request path routing
  in the service (of which there
  may be several ways to do things for each language)
* The users will use FeedAPI libraries to connect anyway
* FeedAPI does not work well with OpenAPI, Swagger, etc anyway.
* A single endpoint per feed is easier to relate to in
  API gateways and so on

Below `https://service/myfeed` is used as a placeholder in examples
for an arbitrary endpoint name.

## Authorization

Authorization is in general recommended to use a Bearer token in the Authorization
header; although the exact scheme is out of the scope of this specification.

Standard HTTP 401 and 403 response codes should be used.

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

The discovery endpoint's reference manual is below, but for now,
this is the relevant section for partitions:
```json
{
  "partitions": [
    {
      "id": "0",
      "lastCursor": "f1ceaa92eb7c11eda43d6fb319691265",
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

#### Starting and closing partitions

A new partition can appear in the list at any time.
In the future we are likely to
add a special message "new partition appeared" on *all* event streams when
this happens to signal live consumers to re-scan the discovery endpoint,
but this is left out of scope for now.

An existing partition that will never again receive writes
can be marked as `"closed": true`. Consumers should still
consume it from the start until the end for re-constitution, but they will
know that no *new* events will appear on it.


#### Method 1 to change partition count: New events on new partitions

The first method to increase the number of partitions is to *close* the
parent partition, and open a number of child partitions to replace it.
For instance, in this case partition 0 was split into partitions 1 and 2:

```json
{
  "partitions": [
    {
      "id": "0",
      "lastCursor": "f1ceaa92eb7c11eda43d6fb319691265",
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

## Discovery call

A simple argument-less `GET` returns information about the feed.
```
GET https://service/myfeed
```
Example response:
```json
{
    "partitions": [
      {
        "id": "0",
        "lastCursor": "f1ceaa92eb7c11eda43d6fb319691265",
        "closed": true
      },
      {
        "id": "16000",
        "startsAfterPartition": "0"
      }
    ],
    "stream": true,
    "exactlyOnce": true,
    "filters": ["subject"]
}
```

### partitions

Lists the partitions of the feed. Details about how this works
documented above.

Fields:

* `id`: String, but only allowed values is integers in the range `[0..32767]`. The ID is a 16-bit
  integer because it may be convenient for the consumer to put this
  in the primary key in the destination database, and because the
  producer is in a position to easily manage a restricted ID space.
  * This is an integer for compatability with version 1; but
    in general in JSON it is good form that IDs are strings even if
    they are integers.

* `closed`: If set, it is guaranteed that no new events will appear on this feed.

* `lastCursor`: Required if partition is closed, optional otherwise.
  Provides the last cursor on the feed at the time of the poll.
  
* `startsAfterPartition` and `cursorFromPartitions`: Optional. Used
  to change the number of partitions; in each case either one mechanism
  or the other one is used. See section above for description.

* `filters`: A list of additional filter flags that are supported.
  See `filter-*` below.

### Flags

Protocol flags are provided as fields on the root (boolean or sub-structs
with more information). The list will likely be extended in the future.

Currently supported flags:

* `stream`: The stream argument is supported. See below.
* `exactlyOnce`: Each event `id` will always be present exactly
  once on the feed and never seen again. If this is false, the same
  event `id` may appear several times (usually with the exact same contents,
  indicating a delivery retry further behind in the pipeline,
  although any such guarantees is application specific).
  For instance, an adaptor on top of Azure Event Hub should set
  `exactlyOnce` to `false`.

## Fetch call

### Request
A fetch is done to the same URL as the discovery, but comes with
some arguments:
```
GET https://service/myfeed?partition=16000&cursor=f1ceaa92eb7c11eda43d6fb319691265
```

Arguments:

* `partition`: ID of partition (see discovery call)
* `cursor`: The place in the feed to start reading. Special cursors `_first` and `_last`
  can be used for each end of the feed as initial values.
  * In some cases, the publisher may e.g. document that the cursor values are ULID
    or similar. In this case, the consumer constructing a cursor to start in a given
    position is perfectly OK; but outside the scope of this specification.
* `pageSizeHint`: How many events to return. Not compatible with **stream**.
* `stream`: Can be set to either a duration in *number of milliseconds*,
  or `y` which means "infinite". The service will then make the HTTP request
  live for this long, and keep returning events over the link. The effect
  is equivalent to downloading an infinitely large file over HTTP. This works
  fine with the HTTP protocol, but one has to be aware of the effect of any API
  gateways in the middle that may assume short-lived requests. Not compatible
  with **pageSizeHint**.
* `filter-*`: It is possible to provide additional filter argument to the
  server. For instance, to get all events that has a given `subject` property,
  pass `&filter-subject=somevalue`.
  * The specific filters supported for a FeedAPI endpoint is out of scope
    for this reference; they are documented by the server (in much the same way
    as the event payloads).
  * If a filter is not supported by a server, it should cause a 400 Bad Request
    response; not be ignored.
  * Usually the use of a filter will cause a different database index to be used;
    i.e. the server has to prepare for the use of filters.
  * Cursors on the event feed are **only valid for the identical set of filters**.


### Response
The response is in the NDJSON format; each line (separated by `\n`) represents
a protocol message from the publisher to consumer. The reason for this format
is to be able to *stream* events, which would not be as easy with a more
usual JSON response using a JSON list.
```
{"event": {/*...payload... */}}
{"cursor": "2394r7a98a7342qw34r2412rwa"}
{"event": {/*...payload... */}}
{"cursor": "24rw3afawowraqwl2ijur3lakj"}
```

There are 2 kinds of commands:
* If `event` is present, the line contains an individual event
* If `cursor` is present, the line is a checkpoint.

Consumers should gracefully handle not only new unknown fields
in the JSON objects (as is usual), but also ignore entirely new
commands (i.e., lines not containing either `event` nor `cursor`).
In particular a command saying "please re-sync the partition list"
is likely to be added in the future.

#### Event command

The event payload contained in the `event` key. This can *either* be
a string (e.g., a binary representation) *or* an embedded JSON document
in any format. (The value being JSON list, number or bool is not supported.)

#### Checkpoint command

Every response **must** always return at least one `cursor`, even if there
is no new `event`s available. Consumers should update cursors even if there
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


