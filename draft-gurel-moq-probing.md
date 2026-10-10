---
title: "Probe Tracks for Bandwidth Estimation in Media over QUIC Transport"
abbrev: moq-probing
docname: draft-gurel-moq-probing-latest
date: {DATE}
category: std

ipr: trust200902
area: Applications and Real-Time
submissionType: IETF
workgroup: MOQ
keyword:
 - Internet-Draft
 - bandwidth estimation
 - probing
 - adaptive bitrate

stand_alone: yes
smart_quotes: no
pi: [toc, sortrefs, symrefs, docmapping]

author:
  -
    ins: Z. Gurel
    name: Zafer Gurel
    organization: Ozyegin University
    email: zafer.gurel@ozu.edu.tr

  -
    ins: A. C. Begen
    name: Ali C. Begen
    organization: Networked Media
    email: ali.begen@networked.media

  -
    ins: D. Ugur
    name: Deniz Ugur
    organization: Ozyegin University
    email: deniz.ugur@ozu.edu.tr

normative:
  MoQTransport: I-D.ietf-moq-transport
  LOC: I-D.ietf-moq-loc
  CommonCatalogFormat: I-D.ietf-moq-catalogformat

informative:
  TrackSwitching: I-D.gurel-moq-track-switching

--- abstract

This document defines probe tracks for Media over QUIC Transport (MOQT):
tracks of synthetic objects, generated on demand, that measure the
bandwidth of a single hop. Probe tracks live under reserved,
session-scoped Track Namespaces and are never forwarded or cached. The
track name selects the shape of the probe: target bitrate, duration or
size, object rate, object size, group structure and key-object bursts.
For a downlink probe, a subscriber subscribes to a probe track in the
namespace (`.session`, `.probe`), and the relay generates it, so the
subscriber can measure whether a higher bitrate is sustainable on the
relay-to-subscriber hop before it switches to a higher-quality track.
For an uplink probe, a publisher generates a probe track and publishes
it to its relay in the namespace (`.session`, `.sink`), to measure the
publisher-to-relay hop.

--- middle

# Introduction

Endpoints of Media over QUIC Transport (MOQT) {{MoQTransport}} adapt to
the network on both sides of a relay. A subscriber runs adaptive bitrate
(ABR) logic and has to decide when it is safe to switch to a
higher-bitrate alternative of the track it is consuming. A publisher has
to tune its encoding parameters dynamically to the uplink network
conditions. Both need to estimate the bandwidth of their own hop, and
both face the same problem: throughput measured on the current media is
limited by the bitrate of that media. An endpoint that sends or
receives a 1 Mbps track at 1 Mbps learns only that at least 1 Mbps is
available. To learn whether 3 Mbps is sustainable, it has to make more
data flow over the hop.

This document defines probe tracks: tracks of synthetic objects,
generated on demand in a shape described by the track name, that
measure the bandwidth of a single hop. Probe tracks are session-scoped:
they are never forwarded beyond the hop they measure, and never cached.
They come in two directions ({{directions}}):

* In a downlink probe, a subscriber subscribes to a probe track and its
  relay generates it, to measure the hop from the relay to the
  subscriber.

* In an uplink probe, a publisher generates a probe track and publishes
  it to its first relay, which consumes it, to measure the hop from the
  publisher to the relay.

~~~
Publisher                    Relay                    Subscriber
    |                          |                           |
    | PUBLISH                  |                           |
    | (.session, .sink)        |                           |
    |------------------------->|                           |
    | PUBLISH_OK (Forward=1)   |                           |
    |<-------------------------|                           |
    | uplink probe objects     |                           |
    |=========================>|                           |
    |                          |                 SUBSCRIBE |
    |                          |        (.session, .probe) |
    |                          |<--------------------------|
    |                          | downlink probe objects    |
    |                          |==========================>|
~~~
{: #probe-overview title="Uplink and downlink probes."}

In both directions, a probe:

* exercises exactly one hop, which is usually the bottleneck for the
  decision at hand, and adds no load beyond it;

* runs on the same MOQT session and congestion controller as the media,
  so it sees the same path conditions;

* can imitate a media track (paced objects, groups and key-object
  bursts) or run as a greedy bulk transfer, depending on what the
  endpoint needs to learn; and

* uses an object layout similar to the Low Overhead Media Container
  (LOC) {{LOC}}, so receivers can reuse their LOC parsing logic, and
  carries a send timestamp on every object, so the receiver can compute
  throughput, delay variation and loss without clock synchronization.

The result of a probe is input to an adaptation decision: for a
subscriber, a switch to another track, for example with the mechanism
in {{TrackSwitching}}; for a publisher, a change of its encoding
parameters.

## Terms and Definitions

{::boilerplate bcp14-tagged}

This document uses the terms Track, Group, Subgroup, Object, Track
Namespace, Track Name, Full Track Name, Relay, Publisher and Subscriber
as defined in {{MoQTransport}}.

Probe Track:

: A track of synthetic objects that the probe sender generates for a
  single subscription, whether the subscription was initiated with
  SUBSCRIBE or with PUBLISH.

Probe Sender:

: The endpoint that generates and sends the objects of a probe track:
  the relay for a downlink probe, the publisher for an uplink probe.

Probe Receiver:

: The endpoint that receives the objects of a probe track: the
  subscriber for a downlink probe, the relay for an uplink probe.

Probe Requester:

: The endpoint that sends the request that starts a probe: the
  subscriber (SUBSCRIBE) for a downlink probe, the publisher (PUBLISH)
  for an uplink probe.

Downlink Probe:

: A probe that a subscriber requests with SUBSCRIBE in the (`.session`,
  `.probe`) namespace, and that the relay generates and sends to the
  subscriber ({{downlink}}).

Uplink Probe:

: A probe that a publisher sends with PUBLISH in the (`.session`,
  `.sink`) namespace to a relay, which consumes and discards it
  ({{uplink}}).

Probe Name:

: The Track Name of a probe track. It is either a Dynamic Probe Name or
  a Preset Name.

Dynamic Probe Name:

: A Probe Name that encodes the probe parameters directly, as defined in
  {{grammar}}.

Preset Name:

: A Probe Name that the relay maps to a set of probe parameters, as
  advertised in the probe catalog ({{catalog}}). Preset Names are used
  only for downlink probes.

Paced Probe:

: A probe sent at a target rate, with objects released on a schedule.

Greedy Probe:

: A probe sent as fast as the congestion controller and flow control
  allow, bounded by size or duration.

## Notational Conventions

This document uses the conventions detailed in ({{?RFC9000, Section 1.3}})
when describing the binary encoding.

As a quick reference, the following list provides a non-normative summary
of the parts of RFC9000 field syntax that are used in this specification.

x (L):

: Indicates that x is L bits long

x (i):

: Indicates that x holds an integer value using the variable-length
  encoding as described in ({{?RFC9000, Section 16}})

x (..):

: Indicates that x can be any length including zero bits long.  Values
  in this format always end on a byte boundary.

To reduce unnecessary use of bandwidth, variable length integers SHOULD
be encoded using the least number of bytes possible to represent the
required value.

# Probe Directions {#directions}

Downlink and uplink probes share the naming, parameter resolution,
validation, limits and object format defined in {{session-ns}} through
{{payload}}. They differ in who requests, generates and measures the
probe, as summarized below and specified in {{downlink}} and
{{uplink}}.

| | Downlink Probe | Uplink Probe |
|:--|:--|:--|
| Request | SUBSCRIBE | PUBLISH with FORWARD=0 |
| Namespace | (`.session`, `.probe`) | (`.session`, `.sink`) |
| Probe requester | subscriber | publisher |
| Probe sender | relay | publisher |
| Probe receiver | subscriber | relay |
| Endpoint that uses the result | subscriber | publisher |
| Probe Names | Dynamic Probe Names and Preset Names | Dynamic Probe Names only |
| Support discovery | probe catalog | TBD |
{: #direction-table title="Downlink and uplink probes."}

# Session-Scoped Namespaces {#session-ns}

## Session Namespace

This document reserves the Track Namespace Field `.session` (eight
bytes) as the first field of a session-scoped namespace. A track whose
namespace starts with `.session` belongs to the MOQT session on which it
is requested, and is served by the peer at the other end of that
session. An endpoint that implements this specification:

* MUST answer a SUBSCRIBE, FETCH, TRACK_STATUS or SUBSCRIBE_NAMESPACE
  for a namespace that starts with `.session` itself, and MUST NOT
  forward it upstream or to any other session;

* MUST consume a PUBLISH for a namespace that starts with `.session`
  itself, and MUST NOT forward the track or its objects upstream, to any
  subscriber, or to any other session;

* MUST NOT share a session-scoped track between sessions or serve it
  from a cache; and

* MUST NOT announce a namespace that starts with `.session` to its
  peers or upstream. A PUBLISH of a session-scoped track goes only to
  the peer of the session and ends there.

Using a dedicated first field lets a relay recognize session-scoped
requests with the namespace prefix check it already performs for
routing, before any upstream lookup. The leading dot keeps the namespace
apart from application namespaces. Probing is the first use of the
session namespace; other session-scoped services can be defined in the
same way, each with its own second field.

## Probe Namespaces

This document defines two session-scoped namespaces, one per direction:

(`.session`, `.probe`):

: Downlink probes. The second field is `.probe` (six bytes). The only
  request a probe requester sends in this namespace is SUBSCRIBE
  ({{downlink}}). A subscriber can subscribe to a probe track without
  knowing any publisher namespace.

(`.session`, `.sink`):

: Uplink probes. The second field is `.sink` (five bytes). Messages
  except PUBLISH MUST NOT be used in this namespace; a relay rejects any
  other request that names it with REQUEST_ERROR and error code
  NOT_SUPPORTED. This does not restrict the messages of an established
  PUBLISH-initiated subscription, such as PUBLISH_OK, REQUEST_UPDATE and
  PUBLISH_DONE ({{uplink}}).

Because the two directions use separate namespaces, the same Full Track
Name never names both a downlink and an uplink probe. A SUBSCRIBE in the
(`.session`, `.probe`) namespace is always served by the relay's own
probe generator. A relay MUST NOT treat it as a subscription to a track
that the same endpoint publishes.

# Probe Shapes {#shapes}

Probes answer two different questions. This document supports both and
expresses them through one set of parameters.

| Shape | Question it answers | Properties |
|:------|:--------------------|:-----------|
| Paced | Can the hop sustain X bps right now, alongside the media? | Low impact. The flow is application-limited, so it confirms X but does not reveal capacity above X. |
| Greedy | How much headroom does the hop have? | High impact and bursty. Slow start and congestion window growth affect short probes. |
{: #shape-table title="Probe shapes."}

A paced probe can be a smooth stream at a target rate, or it can imitate
a video track: objects at a frame rate, grouped into groups of pictures,
with a larger first object in each group that stands in for a key frame.
The idle time between objects is the same as for real media, so the
congestion controller and any pacing on the path behave as they would
for the media the probe stands in for.

A greedy probe is closer to downloading a segment in HTTP adaptive
streaming. It sends a fixed number of bytes, or sends for a fixed time,
as fast as possible.

# Probe Names {#naming}

## Dynamic Probe Names {#grammar}

A Dynamic Probe Name consists of fields separated by `-`. Each field is
a single-letter parameter key followed by a decimal value. The grammar,
in ABNF {{!RFC5234}}, is:

~~~ abnf
probe-name = field *( "-" field )
field      = key value
key        = %x61-7A                 ; a-z
value      = "0" / ( %x31-39 *DIGIT ) ; no leading zeros
~~~
{: #probe-name-abnf title="Dynamic Probe Name grammar."}

The endpoint that receives the request parses a Dynamic Probe Name by
splitting it at each `-` and reading the first character of each field
as the parameter key and the rest as its value. Fields can appear in any
order. Probe requesters SHOULD order fields by key, because MOQT
compares Full Track Names byte by byte when it detects duplicate
subscriptions.

The parameter keys are:

| Key | Name | Unit | Meaning |
|:---:|:-----|:-----|:--------|
| r | Rate | bits per second | Target payload rate. If absent, the probe is greedy. |
| d | Duration | milliseconds | Maximum duration of the probe. |
| b | Bytes | bytes | Maximum total payload bytes of the probe. |
| f | Object rate | objects per second | Pacing rate of objects; the frame rate of an imitated video track. |
| s | Object size | bytes | Base object payload size. |
| g | Group size | objects | Number of objects per group. |
| k | Key multiplier | factor | Size of the first object of each group, as a multiple of s. |
{: #probe-keys title="Probe parameter keys."}

Examples:

`d2000-r300000`:
: A smooth 300 kbps stream for 2 seconds.

`b500000`:
: A greedy probe that sends 500,000 bytes as fast as possible.

`d2000-f30-g30-k5-r3000000`:
: An imitation of a 3 Mbps video track at 30 frames per second, with
  one-second groups whose first object is five times the size of the
  others, for 2 seconds.

## Preset Names

A Probe Name that does not match the grammar in {{grammar}} is a Preset
Name. Preset Names are used only for downlink probes: the relay looks
them up in its table of presets, which it can advertise in the probe
catalog ({{catalog}}). A relay MUST NOT define a Preset Name that
matches the grammar in {{grammar}}. For example, `hd-check` is a valid
Preset Name, because `hd` is not a letter followed by digits.

# Resolving Probe Parameters {#resolve}

The parameters depend on each other, so the probe requester usually
gives only some of them, and the rest are derived. The probe sender and
the probe receiver both resolve the parameters with the procedure in
this section, and get the same result. The endpoint that receives the
request resolves them before it accepts the request.

Let s be the base object payload size. The first object of each group is
k*s bytes and the other objects are s bytes, so the mean object size is
s*(k+g-1)/g. A paced probe therefore has:

~~~
r = 8 * f * s * (k + g - 1) / g     (bits per second)
b = r * d / 8000                    (bytes, with d in milliseconds)
~~~
{: #probe-relations title="Relations between probe parameters."}

The defaults for parameters a probe requester omits are fixed by this
document, so that both endpoints can resolve any Dynamic Probe Name
without asking the other. The default object rate (DEFAULT_F) is 50
objects per second, that is, one object every 20 ms. The default greedy
object size (DEFAULT_GREEDY_S) is 16384 bytes. The default key
multiplier is 1. When g is absent, the whole probe is one group.

A greedy probe that gives only d has no byte bound, and a greedy probe
that gives only b has no time bound. Resolution leaves such a bound
open, so that it does not depend on the relay's limits, which the other
endpoint may not know. The relay's limits ({{limits}}) still apply
while the probe runs: a greedy probe that reaches one of them ends
early, as described in {{downlink}} and {{uplink}}. This is not
clamping, because the Probe Name did not ask for more.

The resolution procedure is shown below. Divisions are real-valued
unless floor() is applied. A failure is one of the outcomes INVALID,
PER_PROBE_LIMIT or SESSION_LIMIT, which map to error codes as described
in {{errors}}.

~~~ pseudocode
resolve(p):             // p: parsed keys, absent keys are null
  if p.d == null and p.b == null:  fail INVALID
  if p.f != null and p.s != null:  fail INVALID
  if p.k != null and p.g == null:  fail INVALID  // needs g
  k = p.k or 1
  g = p.g                 // null: the whole probe is one group
  ratio = (g != null) ? (k + g - 1) / g : 1

  if p.r == null:                  // greedy probe
    if p.f != null:  fail INVALID  // greedy is not paced
    s = p.s or DEFAULT_GREEDY_S
    b = p.b                        // null: no byte bound
    d = p.d                        // null: no time bound
    schedule = as fast as possible
  else:                            // paced probe
    r = p.r
    if p.f != null:
      f = p.f
      s = floor(r / (8 * f * ratio))
    else if p.s != null:
      s = p.s
      f = r / (8 * s * ratio)
    else:
      f = DEFAULT_F
      s = floor(r / (8 * f * ratio))
    if p.d != null and p.b != null:  // stop at first limit hit
      d = p.d;  b = p.b
    else if p.d != null:
      d = p.d;  b = r * d / 8000
    else:
      b = p.b;  d = 8000 * b / r
    schedule = object i is released at t0 + i / f

  if s < 1:  fail INVALID  // rate too low for f
  check s, k * s, r, f and every bound of b and d that is
    not open against the per-probe limits
                       else fail PER_PROBE_LIMIT
  check concurrent probes and session byte budget
                       else fail SESSION_LIMIT

  // Object i (i = 0, 1, ...) is in Group floor(i / g) (Group 0
  // if g is null), with Object ID i mod g (i if g is null). Its
  // payload size is k * s if its Object ID is 0, and s otherwise.
  // The probe sender stops after sending b bytes or after d ms,
  // whichever comes first, and truncates the last object so that
  // the total does not exceed b.
~~~
{: #probe-resolve title="Resolving probe parameters."}

The following table resolves the examples from {{grammar}} with the
defaults.

| Probe Name | Shape | Resolved values |
|:-----------|:------|:----------------|
| `d2000-r300000` | paced | f = 50, s = 750, 100 objects, b = 75,000, one group |
| `b500000` | greedy | s = 16,384, 31 objects (the last one truncated), one group, no time bound |
| `d2000-f30-g30-k5-r3000000` | paced | s = 11,029, first object of each group 55,145, 60 objects in 2 groups |
{: #probe-examples title="Resolved examples."}

# Validation, Limits and Errors {#validation}

## Validation Rules {#checks}

The endpoint that receives a probe request validates it before it
accepts the request with SUBSCRIBE_OK or PUBLISH_OK. The request is
invalid if:

* the Probe Name matches neither the grammar in {{grammar}} nor, for a
  downlink probe, a known Preset Name;

* the Probe Name is a Preset Name in an uplink probe;

* a key is not defined in {{probe-keys}}, or a key appears more than
  once;

* a value has a leading zero, does not fit in 62 bits, or is zero for
  any of r, d, b, f, s, g or k;

* neither d nor b is present, both f and s are present, k is present
  without g, or f is present without r; or

* a derived value is unusable, for example the resolved object size s
  is zero because r is too low for the object rate f.

An invalid request MUST be rejected with the error code for invalid
probes in {{errors}}.

## Limits {#limits}

A relay MUST enforce limits on the resources a probe can use, in both
directions. At a minimum, a relay MUST limit the rate, the duration, the
total bytes and the object size of a single probe, the number of probes
running at the same time on a session, and the total probe bytes per
session over a period of time.

A per-probe limit (rate, duration, total bytes or object size) that is
exceeded can never be met at this relay, so the relay MUST reject the
request with the per-probe limit error code in {{errors}} and a Retry
Interval of 0.

A session limit (concurrent probes or the per-session byte budget) that
is exceeded can be met later, so the relay MUST reject the request with
error code EXCESSIVE_LOAD and SHOULD set the Retry Interval to the time
after which the request is expected to succeed.

The relay MUST NOT silently clamp the parameters, because an endpoint
that measures a different probe from the one it asked for draws the
wrong conclusion.

Unless configured otherwise, a relay SHOULD apply the following default
limits. They apply to downlink and uplink probes alike, separately for
each direction.

| Limit | Default |
|:------|:--------|
| Maximum duration (d) | 5000 ms |
| Maximum total bytes (b) | 10,000,000 bytes |
| Maximum rate (r) | 50,000,000 bits per second |
| Maximum object size (k*s) | 1,048,576 bytes |
| Concurrent probes per session | 1 |
| Minimum time between the end of one probe and the start of the next | 1000 ms |
| Probe bytes per session | 50,000,000 bytes per 60 s |
{: #default-limits title="Default probe limits."}

## Error Codes {#errors}

This document defines no new error codes. The endpoint that receives a
probe request rejects it with REQUEST_ERROR {{MoQTransport}}, using
existing error codes. In a downlink probe the relay is the publisher of
the probe track, and in an uplink probe it is the subscriber, so the
codes differ by direction:

| Outcome | Downlink (SUBSCRIBE) | Uplink (PUBLISH) | Retry Interval |
|:--------|:---------------------|:-----------------|:---------------|
| Invalid probe ({{checks}}) | DOES_NOT_EXIST | UNINTERESTED | 0 |
| PUBLISH with Forward State 1 | - | UNINTERESTED | 0 |
| Per-probe limit exceeded | DOES_NOT_EXIST | UNINTERESTED | 0 |
| Session limit exceeded | EXCESSIVE_LOAD | EXCESSIVE_LOAD | time until the request can succeed |
| Session not allowed to use probes | UNAUTHORIZED | UNAUTHORIZED | 0 |
| Other request in the namespace, such as FETCH | NOT_SUPPORTED | NOT_SUPPORTED | 0 |
{: #probe-errors title="Error codes for probe requests."}

Because one code covers several outcomes, the Reason Phrase carries the
detail. It MAY name the field and the limit at fault, for example "r
exceeds 20000000". A relay that does not want to reveal its limits MAY
send a generic Reason Phrase instead, for example "probe rejected, use
a preset". A subscriber that receives DOES_NOT_EXIST for a Dynamic
Probe Name can switch to a Preset Name from the catalog, or retry with
smaller values; a publisher can retry an uplink probe with smaller
values.

A relay that does not implement this specification also responds to a
SUBSCRIBE in the (`.session`, `.probe`) namespace with DOES_NOT_EXIST,
or forwards it to a publisher that does the same. A rejected probe
request alone therefore does not show whether the relay supports
probes; support is discovered separately for each direction
({{catalog}}, {{uplink-discovery}}).

# Probe Objects {#payload}

## Streams and Priority

The probe sender sends each group on its own subgroup stream with
Subgroup ID 0. A greedy probe, and a paced probe without g, is one group
on one stream.

The probe sender MUST send probe objects with the lowest publisher
priority, so that the probe uses only the capacity that is left over
after the media on the same session, which is the capacity the probe is
meant to measure.

## Object Format

Probe objects are similar to objects in the Low Overhead Media
Container (LOC) {{LOC}}: they use the same layout and the same
properties, so developers can parse probe objects with the same logic
they use for LOC media objects. Probe objects are not LOC media,
however: their payload is padding, not an encoded audio or video frame.

Each probe object carries the following LOC Public Properties in its
MOQT Object Properties:

Timestamp (0x10):

: The wall-clock time, in microseconds since the Unix epoch, at which
  the probe sender released the object for sending. The probe sender
  MUST include it in every probe object. The probe sender MUST NOT send
  a Timescale property, so that the Timestamp is interpreted as
  wall-clock time. Probe receivers can use differences between
  Timestamps without synchronized clocks. Probe receivers whose clock
  is synchronized with the probe sender can also compute one-way delay.

Video Frame Marking (0x09):

: For a paced probe with g present, the probe sender MAY include this
  property and set the independent flag on the first object of each
  group only, to imitate the key frames of a video track. The probe
  sender MUST NOT include it otherwise.

The payload of a probe object takes the place of the LOC Payload, and
consists of padding: bytes that bring the payload to its resolved size
(s, or k*s for the first object of a group). The probe sender SHOULD set
them to zero, and receivers MUST ignore their values. Probe objects
carry no LOC Private Properties. Because the payload is not media, a
receiver MUST NOT pass a probe payload to a media decoder, even if it
parses probe objects with a LOC parser.

Probe objects carry no other per-object fields:

* The position of an object in the probe follows from its Location.
  Object i is Group floor(i / g), Object ID i mod g ({{resolve}}).
  A gap in Locations shows objects that were not delivered.

* The rate and the number of objects follow from the Probe Name, which
  both endpoints resolve with the procedure and the fixed defaults in
  {{resolve}}. For a Preset Name, the subscriber resolves the Dynamic
  Probe Name that the catalog gives for it ({{catalog}}).

* The end of the probe is signaled by PUBLISH_DONE.

TBD: Whether padding should be zeros or the output of a pseudorandom
number generator. Zeros are free to produce. MOQT always runs over an
encrypted QUIC connection, so zero padding looks random on the wire,
cannot be compressed by the network, and does not weaken QUIC's AEAD
ciphers, which resist known-plaintext attacks. Only the two endpoints
see the plaintext, and probe objects are never cached.

# Downlink Probes {#downlink}

A downlink probe measures the hop from a relay to a subscriber. The
subscriber requests it with SUBSCRIBE in the (`.session`, `.probe`)
namespace, and the relay generates it.

## Requesting a Downlink Probe

A downlink probe is generated for a single subscription. A relay MUST
NOT aggregate two subscriptions to the same Probe Name into one
generated track, MUST NOT cache probe objects, and MUST NOT serve them
to another subscriber. Every probe starts at Group 0, Object 0,
whatever the subscription filter. A FETCH for a probe track is rejected
({{errors}}).

After SUBSCRIBE_OK, the relay sends the objects according to the
resolved schedule. When the probe ends, it ends the subscription with
PUBLISH_DONE. A greedy probe with an open bound ends when the relay
reaches its own limit ({{resolve}}). A subscriber can stop a probe
early by terminating the subscription.

Subscribers SHOULD subscribe to probe tracks with subscriber priority
255, so that, together with the lowest publisher priority
({{payload}}), the probe measures only the capacity left over after the
subscriber's media.

Relays SHOULD also accept the following equivalent form: an empty Track
Namespace (zero Track Namespace Fields) and a Track Name that starts with
the bytes `.probe/`, followed by a Probe Name. For example, the empty
namespace with Track Name `.probe/d1000-r300000` is equivalent to the
namespace (`.session`, `.probe`) with Track Name `d1000-r300000`.

## Probe Catalog {#catalog}

A relay that implements this specification MUST serve a catalog track
named `catalog` in the (`.session`, `.probe`) namespace. The catalog
uses the Common Catalog Format {{CommonCatalogFormat}}.

### Catalog as Proof of Support

A subscriber learns whether a relay supports downlink probes by
subscribing to the probe catalog. If the subscription succeeds, the
relay supports downlink probes. If it fails with DOES_NOT_EXIST, the
relay does not, and the subscriber does not send probe requests to it.
A rejected probe request alone does not tell the two cases apart (see
{{errors}}).

### Catalog Contents

The catalog lists the relay's presets as tracks. For each preset, the
catalog gives:

* `name`: the Preset Name, which is the Track Name to subscribe to;

* `selectionParams`: the `bitrate` (r) and, for a paced probe with an
  object rate, the `framerate` (f), so that subscribers can select a
  preset the same way they select media tracks; and

* `probeName`: the Dynamic Probe Name that the preset stands for, which
  lets the subscriber resolve the preset with the procedure in
  {{resolve}}.

The `streamingFormat` field is set to the custom value `probe`, and
`streamingFormatVersion` to `1`. The probe catalog is read only by
subscribers that implement this specification, so a custom value is
enough to identify it, and other subscribers never need to parse it.
`probeName` is likewise a custom field. The common track fields set the
packaging to `loc` ({{payload}}). The catalog does not set `namespace`,
so the tracks inherit the namespace of the catalog track itself. A
catalog with no presets has an empty `tracks` array; it still proves
support for Dynamic Probe Names.

~~~ json
{
  "version": 1,
  "streamingFormat": "probe",
  "streamingFormatVersion": "1",
  "commonTrackFields": {
    "packaging": "loc"
  },
  "tracks": [
    {
      "name": "sd-check",
      "label": "1 Mbps video-like probe, 1 s",
      "selectionParams": { "bitrate": 1000000, "framerate": 30 },
      "probeName": "d1000-f30-g30-r1000000"
    },
    {
      "name": "hd-check",
      "label": "5 Mbps video-like probe with key frames, 1 s",
      "selectionParams": { "bitrate": 5000000, "framerate": 30 },
      "probeName": "d1000-f30-g30-k5-r5000000"
    },
    {
      "name": "burst",
      "label": "Greedy 1 MB burst",
      "probeName": "b1000000"
    }
  ]
}
~~~
{: #catalog-example title="Probe catalog example."}

The catalog MUST NOT carry the relay's limits. Session limits change
with the state of the session ({{limits}}), so limits in a catalog would
go stale, and a request that fits them can still be rejected. A
subscriber learns about limits only from the result of a probe request.
For the same reason, a preset in the catalog is not a promise: a relay
SHOULD list only presets that fit its per-probe limits, but a request
for a preset can still fail with EXCESSIVE_LOAD.

When the presets change, the relay publishes the new catalog as a new
group.

TODO: The Common Catalog Format expects `streamingFormat` values to be
registered, and recommends a reverse-domain prefix for custom fields
such as `probeName`. Discuss whether the custom `probe` value and the
unprefixed `probeName` field are acceptable, or whether they should be
registered.

## Downlink Probes Served by Publishers {#publisher}

Any MOQT endpoint that acts as a publisher can serve probe tracks with
the same logic as a relay: the same Probe Names, parameter resolution,
validation, errors and object format. When a subscriber is connected
directly to a publisher, the publisher answers the (`.session`,
`.probe`) namespace like a relay ({{session-ns}}).

A publisher MAY also offer probe tracks in its own namespace and list
them in its catalog, so that subscribers can reach them through relays
and measure the whole path from the publisher. A publisher MUST respond
to a FETCH for such a track with REQUEST_ERROR and error code
NOT_SUPPORTED.

TODO: Relays do not read catalogs, so a relay cannot tell a publisher
probe track from a media track, and would cache and aggregate it like
any other track. Considerations:

* Probe tracks must not be cached.

* Probe tracks must not be fetched.

* These constraints can be put in track properties, so that relays can
  apply them without access to the catalog.

# Uplink Probes {#uplink}

An uplink probe measures the hop from a publisher to its first relay.
The publisher generates the probe and publishes it with PUBLISH in the
(`.session`, `.sink`) namespace, and the relay consumes it: the relay
never caches it and never forwards it upstream or to any subscriber,
including the publisher itself ({{session-ns}}).

## Requesting an Uplink Probe

The Track Name of an uplink probe MUST be a Dynamic Probe Name
({{grammar}}). Preset Names are not used for uplink probes, because the
publisher has to resolve the parameters ({{resolve}}) to generate the
probe anyway.

The publisher sends PUBLISH for the uplink probe track with the FORWARD
parameter set to 0, and MUST NOT send any objects until the relay sets
the Forward State to 1. The relay validates the request with the rules
in {{validation}}. If the probe is acceptable, the relay sends
PUBLISH_OK with Forward State 1, and the publisher starts the probe.
Otherwise, the relay rejects the PUBLISH ({{errors}}).

Waiting for the relay keeps it in control of what it receives: the
relay checks the request against its limits before any probe data
arrives. The time at which the publisher receives Forward State 1 is
also a clean start time for the measurement.

A relay that receives a PUBLISH in the (`.session`, `.sink`) namespace
with Forward State 1, or without the FORWARD parameter (which defaults
to 1), MUST reject it with REQUEST_ERROR, MUST discard any objects
already received for it, and SHOULD count those bytes against the
session limits. The relay MUST NOT close the session for this reason: a
bad probe request affects only the probe, not the media on the same
session.

## Sending and Enforcement

The publisher sends the probe according to the resolved parameters,
with the object format in {{payload}}; the LOC Timestamp carries the
publisher's send time. The publisher ends the probe with PUBLISH_DONE.

A relay MUST enforce the declared shape. If the publisher sends more
bytes than b, sends for longer than d, or sends faster than r by a
margin that the relay chooses, the relay SHOULD pause the probe by
sending REQUEST_UPDATE with Forward State 0, or end it by terminating
the subscription. A greedy probe with an open bound ends when it
reaches the relay's limit ({{resolve}}): the relay then terminates the
subscription. The relay discards all probe objects after it has
measured their arrival.

A relay SHOULD apply the same flow control that it applies to ordinary
(e.g., media) streams to uplink probe streams as long as the probe track
stays within its pre-defined limits, that is, the parameters in its
Probe Name. Flow control is then part of the path being measured, as it
will be for the publisher's media.

## Support Discovery {#uplink-discovery}

TBD: How a publisher learns whether a relay accepts uplink probes. The
probe catalog proves support for downlink probes only. Options include a
custom root field in the probe catalog, such as `"acceptsPublish":
true`, or a SETUP option that the relay sends when the session is
established. A SETUP option is likely the better fit, because it tells
the publisher before it sends any request, and it covers relays that do
not serve downlink probes.

## Measurement Result

The relay receives the probe, but the publisher needs the result. The
publisher can estimate the uplink capacity on its own side, from the
delivery rate that its congestion controller observes while it sends
the probe; a greedy probe is the most suitable shape for this. How much
of this information is available depends on the publisher's transport
API.

TBD: A way for the relay to report what it measured, for example a
session-scoped result track, such as (`.session`, `.probe`, `.result`),
on which the relay publishes one object with the measured throughput and
delay for each uplink probe, and to which the publisher subscribes.

# Measuring with Probes {#measurement}

This section is informative.

The probe receiver measures a probe in the same way in both directions.
For a greedy probe, it can estimate throughput as the number of bytes
received between the arrival of the first and the last object, divided
by the time between those arrivals. Excluding the first object reduces
the effect of the request round trip.

For a paced probe, the probe receiver compares, for each object, the gap
between arrivals with the gap between LOC Timestamps. If the arrival
gaps keep growing relative to the send gaps, queuing delay is building
up and the target rate is not sustainable. The probe receiver can also
compare the receive rate over the probe with r. Gaps in object
Locations show objects that were not delivered, for example because the
subscription ended early.

## Downlink Use

A subscriber that wants to switch from a track at rate R1 to a track at
rate R2 can run a paced probe at r = R2 - R1 at the lowest priority
while it keeps receiving the current track. If the probe is delivered on
time, the path is likely to sustain R2, and the subscriber can switch
(see {{TrackSwitching}}).

## Uplink Use

In an uplink probe, the relay is the probe receiver, but the publisher
needs the result. A publisher that estimates from its own congestion
controller uses a greedy probe and reads the delivery rate at the end
of the probe; a publisher that wants to raise its encoding bitrate from
R1 to R2 can instead run a paced probe at r = R2 - R1 and use a result
reported by the relay ({{uplink}}).

# Security Considerations {#security}

A probe request is small, but the probe it starts can carry a large
amount of data: from the relay for a downlink probe, and to the relay
for an uplink probe. Without limits, an endpoint could use probes to use
up relay bandwidth, CPU or memory. Relays MUST enforce the limits in
{{limits}} in both directions, and MAY require that a session be
authorized before it can use probes.

For uplink probes, the relay keeps control by requiring Forward State 0
in the PUBLISH and checking limits before it allows any probe data, by
enforcing the declared shape, and by counting rejected data against the
session limits ({{uplink}}). Relays MAY accept uplink probes only from
authorized publishers.

Probe data only crosses the hop of the session that asked for it, over
a congestion-controlled QUIC connection. Probes therefore cannot be used
to reflect or amplify traffic toward a third party.

Probe objects expose the probe sender's wall clock, at microsecond
resolution, through the LOC Timestamp property: the relay's clock to
subscribers, and the publisher's clock to the relay. Probe senders that
consider this a concern MAY reduce the clock precision; probe receivers
that use only differences between Timestamps are not affected by a
constant offset.

A relay MAY keep its limits out of Reason Phrases, so that clients
cannot use them to fingerprint the relay or tune requests to sit just
below the limits. The catalog does not carry limits ({{catalog}}).

# IANA Considerations {#iana}

This document has no IANA actions. It uses the existing REQUEST_ERROR
codes as described in {{errors}}.

TODO: Discuss whether the probe parameter keys in {{probe-keys}} need an
IANA registry, so that new keys can be added later without updating
this document.

# Open Issues

Common:

* Whether probe padding should be zeros or pseudorandom bytes (see
  {{payload}}).

* Whether the probe parameter keys need an IANA registry (see
  {{iana}}).

* Whether the session namespace ({{session-ns}}) belongs in this
  document or in a separate one, since it is useful beyond probing.

* Whether a relay should be allowed to clamp parameters to its limits
  and report the values it actually uses in SUBSCRIBE_OK or PUBLISH_OK,
  as an alternative to rejecting the request.

* Probes delivered as datagrams, which would let a probe receiver
  measure loss directly.

Downlink probes:

* Whether the custom `probe` streaming format value and the
  `probeName` catalog field should be registered (see {{catalog}}).

* Whether to keep the empty-namespace form of probe names, or to require
  the (`.session`, `.probe`) namespace (see {{downlink}}).

* How relays learn that a publisher probe track must not be cached or
  fetched (see {{publisher}}).

Uplink probes:

* How a publisher discovers support for uplink probes: a catalog field
  or a SETUP option (see {{uplink-discovery}}).

* How a relay reports the result of an uplink probe to the publisher
  (see {{uplink}}).

--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
