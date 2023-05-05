# FeedAPI specification

## Context

For now, this documentation is a working document for people
very familiar with ZeroEventHub; therefore no attempt is made
at explaining concepts.

## Endpoint name

FeedAPI is using RPC over HTTP. All communication for a given
event feed happens on a *single* request path.

I.e., the protocol will not use elements of the request path,
e.g., to specify partition or version or similar.

Why?

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

## Authorization, in particular of discovery call

Authorization is in general recommended to use a Bearer token in the Authorization
header; although the exact scheme is out of the scope of this specification.

Standard HTTP 401 and 403 response codes should be used.

It is **very important** that the discovery call is **not more open** than
the fetch call. I.e.; if fetching a page of events would return 401 or 403,
then the argument-less discovery call **MUST** also return 401/403.
The reason for this is so if one sets up caching proxies in front of the publishing
service, then a discovery call can be used by the proxy to do federated
authorization.

### Details on federated authorization

* The caching proxy should *for each new bearer token* make a discovery call
  to the original publishing service, in order to check that the token
  is accepted. Provided that the discovery call returns 200 OK, the bearer token
  can be trusted.
* The discovery call returns a field `authorizationExpires`. This should
  be used by the caching proxy. (The common implementation is that this is the `exp`
  field of the JWT, but important that this is done by the publisher, which
  can also opt for having a shorter expiry than the one in the token).
* The caching proxy should not in general need to understand the bearer token
  (e.g. even know that it is a JWT). It may do so, but should do so *in addition* to the
  mechanism described above.

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
      "id": 0,
      "lastCursor": "f1ceaa92eb7c11eda43d6fb319691265",
      "closed": true
    },
    {
      "id": 16232,
      "startsAfterParent": 0
    },
    {
      "id": 24223,
      "splitFromAncestors": [24001, 24100]
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
      "id": 0,
      "lastCursor": "f1ceaa92eb7c11eda43d6fb319691265",
      "closed": true
    },
    {
      "id": 1,
      "startsAfterPartition": 0
    },
    {
      "id": 2,
      "startsAfterPartition": 0
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
      "id": 0,
    },
  ]
}
```
Then, *after* the split, we have this:
```json
{
  "partitions": [
    {
      "id": 1,
      "cursorFromPartitions": [0]
    },
    {
      "id": 2,
      "cursorFromPartitions": [0]
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

This functionality can also be used to *merge* partitions. In that
case there may for instance be a single partition that has a `cursorFromPartitions`
listing two parent partitions.

## Discovery call

A simple argument-less `GET` returns information about the feed.
```
GET https://service/myfeed
```
Example response:
```json
{
    "id": "21c8eb10-ee46-11ed-86e5-4776acbe8530",
    "partitions": [
      {
        "id": 0,
        "name": "db1/split3/foo",
        "lastCursor": "f1ceaa92eb7c11eda43d6fb319691265",
        "lastOccurenceTime": "2023-05-05T21:43:01",
        "closed": true
      },
      {
        "id": 16000,
        "name": "db2/split5/bar",
        "startsAfterPartition": 0
      }
    ],
    "authorizationExpiry": "2023-05-05 21:50:23",
    "stream": true,
    "exactlyOnce": true,
    "links": [
      {"type": "jsonschema", "path": ".jsonschema"}
    ],
    "headers": {
      "key1": "value1",
    },
}
```

### partitions

Lists the partitions of the feed. In general this list
should *only be added to*. However, clients should deal gracefully
with partitions that are suddenly gone and should treat these
as closed.

Fields:

* **id**: Integer ID in the range `[0..32767]`. The ID is a 16-bit
  integer because it may be convenient for the consumer to put this
  in the primary key in the destination database, and because the
  producer is in a position to easily manage a restricted ID space.

* **name** (optional): A description of the partition, in case this
  makes sense to provide.

* **closed**: If set, it is guaranteed that no new events will appear on this feed.

* **lastCursor**: Required if partition is closed, optional otherwise.
  Provides the last cursor on the feed at the time of the poll.
  
* **startsAfterPartition** and **cursorFromPartitions**: Optional. Used
  to change the number of partitions; in each case either one mechanism
  or the other one is used. See section above for description.

### Feed metadata

* **id** is a UUID that identifies the feed. It should simply be hard-coded in the
  publisher and should never change. This is useful so that caches/cursors
  can safely be re-used even if the HTTP endpoint that is used is moved around.
* **headers** is a free form key-value store that provides generic metadata
  about the feed.
* **links** is a list of links to further information about the event format.
  (This may be a schema, or it can e.g. be details about mapping the event from
  JSON to SQL/Parquet, etc.)
  Rules for the links:
  1) Relative, `".jsonschema"` would be served at `https://service/myfeed/.jsonschema`.
     In this case, the same `Authorization` header should be used.
  
  2) Absolute, so `"/myfeed/.jsonschema"` can be used to point to the same place.
     In this case, the `Authorization` header **MUST** be blank. The request
     could end up in an API gateway and go to another service, and we want to avoid
     a compromised service being used in an attack at another service.
     
  3) External; `"https://anotherservice/some-jsonschema"`. Again, the `Authorization`
     header must be blank in this case.

  4) Relative paths in another subtree, `"../schemas/myevent"` is **disallowed**.
     This is again to avoid CSRF, that the client is using an `Authorization`
     header but the API gateway routes it to another service.

### Security

* **authorizationExpiry** replies back the time the authorization token
  used expires. This is useful for caching proxies to do federated authorization.

### Flags

Protocol flags are provided as fields on the root (boolean or sub-structs
with more information). The list will likely be extended in the future.

Currently supported flags:

* **stream**: The stream argument is supported. See below.
* **exactlyOnce**: Each event `id` will always be present exactly
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
GET https://service/myfeed?partition=16000&cursor=f1ceaa92eb7c11eda43d6fb319691265&client=myconsumer
```

Arguments:

* **partition**: ID of partition (see discovery call)
* **cursor**: The place in the feed to start reading. Special cursors `_first` and `_last`
  can be used for each end of the feed as initial values.
  * In some cases, the publisher may e.g. document that the cursor values are ULID
    or similar. In this case, the consumer constructing a cursor to start in a given
    position is perfectly OK; but outside the scope of this specification.
* **pageSizeHint**: How many events to return. Not compatible with **stream**.
* **stream**: Can be set to either a duration in *number of milliseconds*,
  or `y` which means "infinite". The service will then make the HTTP request
  live for this long, and keep returning events over the link. The effect
  is equivalent to downloading an infinitely large file over HTTP. This works
  fine with the HTTP protocol, but one has to be aware of the effect of any API
  gateways in the middle that may assume short-lived requests. Not compatible
  with **pageSizeHint**.


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


###




