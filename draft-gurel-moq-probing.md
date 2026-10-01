---
title: "Bandwidth Probing with Relay-Generated Tracks in Media over QUIC Transport"
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
 - bandwidth probing
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

This document defines probe tracks for Media over QUIC Transport (MOQT).
A probe track is a special track that a relay generates locally. It lives
under the reserved Track Namespace `.probe`. A subscriber selects the
shape of the probe through the track name: target bitrate, duration or
size, object rate, object size, group structure and key-object bursts.
The relay serves the probe on the same session as the subscriber's media
and never forwards the request upstream. The subscriber can therefore
measure, on the relay-to-subscriber hop, whether a higher bitrate is
sustainable before it switches to a higher-quality track.

--- middle

# Introduction

Adaptive bitrate (ABR) logic in a MOQT {{MoQTransport}} subscriber has to
decide when it is safe to switch to a higher-bitrate alternative of the
track it is consuming. Throughput measured on the current track is
limited by the bitrate of that track: a subscriber that receives a
1 Mbps track at 1 Mbps learns only that at least 1 Mbps is available.
To learn whether 3 Mbps is sustainable, the subscriber has to make the
sender send more.

MOQT offers some tools for this, but none of them is controlled by the
subscriber:

* The subscriber can subscribe to the higher-bitrate track itself at a
  low priority. This pulls real media through the whole relay chain, and
  may start upstream fetches and cache fills. The subscriber controls
  neither the amount of data nor its pacing. If the probe fails, the
  data was wasted.

* A publisher can send padding streams to probe for bandwidth. Padding
  is started by the sender, so the subscriber cannot request it, choose
  its rate or bound its duration. Padding also carries no sequence or
  timing information that the receiver could use to compute delay
  trends.

This document defines a probe track: a track whose content the relay
generates on demand, in a shape the subscriber describes in the track
name. Because the relay generates the data itself, a probe:

* exercises exactly the hop that is usually the bottleneck for ABR
  decisions, from the relay to the subscriber, and adds no load upstream;

* runs on the same MOQT session and congestion controller as the media,
  so it sees the same path conditions;

* can imitate a media track (paced objects, groups and key-object
  bursts) or run as a greedy bulk transfer, depending on what the
  subscriber needs to learn; and

* uses an object layout similar to the Low Overhead Media Container
  (LOC) {{LOC}}, so receivers can reuse their LOC parsing logic, and
  carries a send timestamp on every object, so the receiver can compute
  throughput, delay variation and loss without clock synchronization.

The result of a probe is input to a switching decision, for example one
made with the mechanism in {{TrackSwitching}}.

## Terms and Definitions

{::boilerplate bcp14-tagged}

This document uses the terms Track, Group, Subgroup, Object, Track
Namespace, Track Name, Full Track Name, Relay, Publisher and Subscriber
as defined in {{MoQTransport}}.

Probe Track:

: A track in the `.probe` namespace whose objects the relay generates
  locally for a single subscription.

Probe Name:

: The Track Name of a probe track. It is either a Dynamic Probe Name or
  a Preset Name.

Dynamic Probe Name:

: A Probe Name that encodes the probe parameters directly, as defined in
  {{grammar}}.

Preset Name:

: A Probe Name that the relay maps to a set of probe parameters, as
  advertised in the probe catalog ({{catalog}}).

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

# Probe Shapes {#shapes}

Probes answer two different questions. This document supports both and
expresses them through one set of parameters.

| Shape | Question it answers | Properties |
|:------|:--------------------|:-----------|
| Paced | Can the path sustain X bps right now, alongside my media? | Low impact. The flow is application-limited, so it confirms X but does not reveal capacity above X. |
| Greedy | How much headroom does the path have? | High impact and bursty. Slow start and congestion window growth affect short probes. |
{: #shape-table title="Probe shapes."}

A paced probe can be a smooth stream at a target rate, or it can imitate
a video track: objects at a frame rate, grouped into groups of pictures,
with a larger first object in each group that stands in for a key frame.
The idle time between objects is the same as for real media, so the
congestion controller and any pacing on the path behave as they would
for the media the subscriber plans to switch to.

A greedy probe is closer to downloading a segment in HTTP adaptive
streaming. It sends a fixed number of bytes, or sends for a fixed time,
as fast as possible.

# Naming and Addressing {#naming}

## Probe Namespace

Probe tracks live in the Track Namespace that has a single Track
Namespace Field containing the four bytes `.probe`. A subscriber can
subscribe to a probe track without knowing any publisher namespace. A
relay that implements this specification MUST NOT forward a SUBSCRIBE,
FETCH or SUBSCRIBE_NAMESPACE for the `.probe` namespace upstream, and
MUST answer it locally.

Using a dedicated namespace lets a relay recognize probe requests with
the namespace prefix check it already performs for routing, before any
upstream lookup. The leading dot keeps the namespace apart from
application namespaces.

Relays SHOULD also accept the following equivalent form: an empty Track
Namespace (zero Track Namespace Fields) and a Track Name that starts with
the bytes `.probe/`, followed by a Probe Name. For example, the empty
namespace with Track Name `.probe/d1000-r300000` is equivalent to the
namespace `.probe` with Track Name `d1000-r300000`.

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

A relay parses a Dynamic Probe Name by splitting it at each `-` and
reading the first character of each field as the parameter key and the
rest as its value. Fields can appear in any order. Subscribers SHOULD
order fields by key, because MOQT compares Full Track Names byte by byte
when it detects duplicate subscriptions.

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
Name. The relay looks it up in its table of presets, which it can
advertise in the probe catalog ({{catalog}}). A relay MUST NOT define a
Preset Name that matches the grammar in {{grammar}}. For example,
`hd-check` is a valid Preset Name, because `hd` is not a letter followed
by digits.

# Resolving Probe Parameters {#resolve}

The parameters depend on each other, so the subscriber usually gives
only some of them and the relay derives the rest. The relay resolves the
parameters before it accepts the subscription.

Let s be the base object payload size. The first object of each group is
k*s bytes and the other objects are s bytes, so the mean object size is
s*(k+g-1)/g. A paced probe therefore has:

~~~
r = 8 * f * s * (k + g - 1) / g     (bits per second)
b = r * d / 8000                    (bytes, with d in milliseconds)
~~~
{: #probe-relations title="Relations between probe parameters."}

The defaults for parameters a subscriber omits are fixed by this
document, so that the subscriber can resolve any Dynamic Probe Name
itself and get the same result as the relay. The default object rate
(DEFAULT_F) is 50 objects per second, that is, one object every 20 ms.
The default greedy object size (DEFAULT_GREEDY_S) is 16384 bytes. The
default key multiplier is 1.
When g is absent, the whole probe is one group.

The resolution procedure is shown below. Divisions are real-valued
unless floor() is applied. MAX_BYTES and MAX_DURATION are the relay's
limits ({{limits}}).

~~~ pseudocode
resolve(p):             // p: parsed keys, absent keys are null
  if p.d == null and p.b == null:  fail DOES_NOT_EXIST
  if p.f != null and p.s != null:  fail DOES_NOT_EXIST
  if p.k != null and p.g == null:  fail DOES_NOT_EXIST  // needs g
  k = p.k or 1
  g = p.g                 // null: the whole probe is one group
  ratio = (g != null) ? (k + g - 1) / g : 1

  if p.r == null:                  // greedy probe
    if p.f != null:  fail DOES_NOT_EXIST  // greedy is not paced
    s = p.s or DEFAULT_GREEDY_S
    b = p.b or MAX_BYTES           // only d: send for d ms
    d = p.d or MAX_DURATION        // only b: send b bytes
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

  if s < 1:  fail DOES_NOT_EXIST  // rate too low for f
  check s, k * s, r, f, b, d against per-probe limits
                       else fail DOES_NOT_EXIST
  check concurrent probes and session byte budget
                       else fail EXCESSIVE_LOAD

  // Object i (i = 0, 1, ...) is in Group floor(i / g) (Group 0
  // if g is null), with Object ID i mod g (i if g is null). Its
  // payload size is k * s if its Object ID is 0, and s otherwise.
  // The relay stops after sending b bytes or after d ms, whichever
  // comes first, and truncates the last object so that the total
  // does not exceed b.
~~~
{: #probe-resolve title="Resolving probe parameters."}

The following table resolves the examples from {{grammar}} with the
defaults.

| Probe Name | Shape | Resolved values |
|:-----------|:------|:----------------|
| `d2000-r300000` | paced | f = 50, s = 750, 100 objects, b = 75,000, one group |
| `b500000` | greedy | s = 16,384, 31 objects (the last one truncated), one group |
| `d2000-f30-g30-k5-r3000000` | paced | s = 11,029, first object of each group 55,145, 60 objects in 2 groups |
{: #probe-examples title="Resolved examples."}

# Validation and Limits {#validation}

## Validation Rules {#checks}

A relay validates a probe request before it sends SUBSCRIBE_OK. The
relay MUST reject the request with error code DOES_NOT_EXIST if:

* the Probe Name matches neither the grammar in {{grammar}} nor a known
  Preset Name;

* a key is not defined in {{probe-keys}}, or a key appears more than
  once;

* a value has a leading zero, does not fit in 62 bits, or is zero for
  any of r, d, b, f, s, g or k;

* neither d nor b is present, both f and s are present, k is present
  without g, or f is present without r; or

* a derived value is unusable, for example the resolved object size s
  is zero because r is too low for the object rate f.

## Limits {#limits}

A relay MUST enforce limits on the resources a probe can use. At a
minimum, a relay MUST limit the rate, the duration, the total bytes and
the object size of a single probe, the number of probes running at the
same time on a session, and the total probe bytes per session over a
period of time.

A per-probe limit (rate, duration, total bytes or object size) that is
exceeded can never be met at this relay, so the relay MUST reject the
request with error code DOES_NOT_EXIST and a Retry Interval of 0. From
the subscriber's point of view, no probe track with that name exists at
this relay.

A session limit (concurrent probes or the per-session byte budget) that
is exceeded can be met later, so the relay MUST reject the request with
error code EXCESSIVE_LOAD and SHOULD set the Retry Interval to the time
after which the request is expected to succeed.

The relay MUST NOT silently clamp the parameters, because a subscriber
that measures a different probe from the one it asked for draws the
wrong conclusion.

## Error Reporting

This document defines no new error codes. A relay rejects a probe with
REQUEST_ERROR {{MoQTransport}}, using the existing error codes as
follows:

| Condition | Error Code | Retry Interval |
|:----------|:-----------|:---------------|
| Invalid Probe Name or parameters ({{checks}}) | DOES_NOT_EXIST | 0 |
| Per-probe limit exceeded | DOES_NOT_EXIST | 0 |
| Session limit exceeded | EXCESSIVE_LOAD | time until the request can succeed |
| Session not allowed to use probes | UNAUTHORIZED | 0 |
| FETCH for a probe track | NOT_SUPPORTED | 0 |
{: #probe-errors title="Error codes for probe requests."}

Because DOES_NOT_EXIST covers several conditions, the Reason Phrase
carries the detail. It MAY name the field and the limit at fault, for
example "r exceeds 20000000". A relay that does not want to reveal its
limits MAY send a generic Reason Phrase instead, for example "probe
rejected, use a preset". A subscriber that receives DOES_NOT_EXIST for a
Dynamic Probe Name can switch to a Preset Name from the catalog, or retry
with smaller values.

A relay that does not implement this specification also responds to a
SUBSCRIBE for the `.probe` namespace with DOES_NOT_EXIST, or forwards it
to a publisher that does the same. Subscribers therefore use the probe
catalog to discover support ({{catalog}}), not the result of a probe
request.

# Delivering a Probe {#delivery}

## Subscription Handling

A probe is generated for a single subscription. A relay MUST NOT
aggregate two subscriptions to the same Probe Name into one generated
track, MUST NOT cache probe objects, and MUST NOT serve them to another
subscriber. Every probe starts at Group 0, Object 0, whatever the
subscription filter. A relay responds to a FETCH for a probe track with
REQUEST_ERROR and error code NOT_SUPPORTED.

After SUBSCRIBE_OK, the relay sends the objects according to the
resolved schedule. When the probe ends, it ends the subscription with
PUBLISH_DONE. A subscriber can stop a probe early by unsubscribing.

The relay sends each group on its own subgroup stream with Subgroup ID
0. A greedy probe, and a paced probe without g, is one group on one
stream.

## Priority

The relay MUST send probe objects with the lowest publisher priority.
Subscribers SHOULD subscribe to probe tracks with subscriber priority
255. The probe then uses only the capacity that is left over after the
subscriber's media, which is the capacity the subscriber wants to
measure.

## Object Format {#payload}

Probe objects are similar to objects in the Low Overhead Media
Container (LOC) {{LOC}}: they use the same layout and the same
properties, so developers can parse probe objects with the same logic
they use for LOC media objects. Probe objects are not LOC media,
however: their payload is padding, not an encoded audio or video frame.

Each probe object carries the following LOC Public Properties in its
MOQT Object Properties:

Timestamp (0x10):

: The wall-clock time, in microseconds since the Unix epoch, at which
  the relay released the object for sending. The relay MUST include it
  in every probe object. The relay MUST NOT send a Timescale property,
  so that the Timestamp is interpreted as wall-clock time. Receivers
  can use differences between Timestamps without synchronized clocks.
  Receivers whose clock is synchronized with the relay can also compute
  one-way delay.

Video Frame Marking (0x09):

: For a paced probe with g present, the relay MAY include this property
  and set the independent flag on the first object of each group only,
  to imitate the key frames of a video track. The relay MUST NOT include
  it otherwise.

The payload of a probe object takes the place of the LOC Payload, and
consists of padding: bytes that bring the payload to its resolved size
(s, or k*s for the first object of a group). The relay SHOULD set them
to zero, and receivers MUST ignore their values. Probe objects carry no
LOC Private Properties. Because the payload is not media, a receiver
MUST NOT pass a probe payload to a media decoder, even if it parses
probe objects with a LOC parser.

Probe objects carry no other per-object fields:

* The position of an object in the probe follows from its Location.
  Object i is Group floor(i / g), Object ID i mod g ({{resolve}}).
  A gap in Locations shows objects that were not delivered.

* The rate and the number of objects follow from the Probe Name, which
  the receiver can resolve itself with the procedure and the fixed
  defaults in {{resolve}}. For a Preset Name, the receiver resolves the
  Dynamic Probe Name that the catalog gives for it ({{catalog}}).

* The end of the probe is signaled by PUBLISH_DONE.

# Probe Catalog {#catalog}

A relay that implements this specification MUST serve a catalog track
named `catalog` in the `.probe` namespace. The catalog uses the Common
Catalog Format {{CommonCatalogFormat}}.

## Catalog as Proof of Support

A subscriber learns whether a relay supports probing by subscribing to
the `.probe` catalog. If the subscription succeeds, the relay supports
this specification. If it fails with DOES_NOT_EXIST, the relay does not,
and the subscriber does not send probe requests to it. A rejected probe
request alone does not tell the two cases apart (see {{validation}}).

## Catalog Contents

The catalog lists the relay's presets as tracks. For each preset, the
catalog gives:

* `name`: the Preset Name, which is the Track Name to subscribe to;

* `selectionParams`: the `bitrate` (r) and, for a paced probe with an
  object rate, the `framerate` (f), so that subscribers can select a
  preset the same way they select media tracks; and

* `probeName`: the Dynamic Probe Name that the preset stands for, which
  lets the receiver resolve the preset with the procedure in
  {{resolve}}.

The common track fields set the namespace to `.probe` and the packaging
to `loc` ({{payload}}). A catalog with no presets has an empty `tracks`
array; it still proves support for Dynamic Probe Names.

~~~ json
{
  "version": 1,
  "streamingFormat": "TBD",
  "streamingFormatVersion": "1",
  "commonTrackFields": {
    "namespace": ".probe",
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

TODO: The Common Catalog Format requires a registered `streamingFormat`
value, and recommends a reverse-domain prefix for custom fields such as
`probeName`. Discuss whether probes need a streaming format
registration and a registered `probeName` field, or whether the probe
catalog should be a profile of an existing streaming format.

# Measuring with Probes {#measurement}

This section is informative.

For a greedy probe, the receiver can estimate throughput as the number
of bytes received between the arrival of the first and the last object,
divided by the time between those arrivals. Excluding the first object
reduces the effect of the request round trip.

For a paced probe, the receiver compares, for each object, the gap
between arrivals with the gap between LOC Timestamps. If the arrival
gaps keep growing relative to the send gaps, queuing delay is building
up and the target rate is not sustainable. The receiver can also
compare the receive rate over the probe with r. Gaps in object
Locations show objects that were not delivered, for example because the
subscription ended early.

A subscriber that wants to switch from a track at rate R1 to a track at
rate R2 can run a paced probe at r = R2 - R1 at the lowest priority
while it keeps receiving the current track. If the probe is delivered on
time, the path is likely to sustain R2, and the subscriber can switch
(see {{TrackSwitching}}).

# Security Considerations {#security}

A small SUBSCRIBE can cause a relay to send a large amount of data.
Without limits, a client could use probes to use up relay bandwidth,
CPU or memory. Relays MUST enforce the limits in {{limits}}, and MAY
require that a session be authorized before it can use probes.

Probe data goes only to the session that asked for it, over a
congestion-controlled QUIC connection. Probes therefore cannot be used
to reflect or amplify traffic toward a third party.

Probe objects expose the relay's wall clock, at microsecond
resolution, to subscribers through the LOC Timestamp property. Relays
that consider this a concern MAY reduce the clock precision; receivers
that use only differences between Timestamps are not affected by a
constant offset.

A relay MAY keep its limits out of Reason Phrases, so that clients
cannot use them to fingerprint the relay or tune requests to sit just
below the limits. The catalog does not carry limits ({{catalog}}).

# IANA Considerations {#iana}

This document has no IANA actions. It uses the existing REQUEST_ERROR
codes as described in {{validation}}.

TODO: Discuss whether the probe parameter keys in {{probe-keys}} need an
IANA registry, so that new keys can be added later without updating
this document.

# Open Issues

* Whether the probe parameter keys need an IANA registry (see
  {{iana}}).

* The `streamingFormat` value and the `probeName` field of the probe
  catalog (see {{catalog}}).

* Whether to keep the empty-namespace form of probe names, or to require
  the `.probe` namespace.

* Whether a relay should be allowed to clamp parameters to its limits
  and report the values it actually uses in SUBSCRIBE_OK, as an
  alternative to rejecting the request.

* Probes delivered as datagrams, which would let a receiver measure loss
  directly.

* Probing paths with more than one relay hop.

--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
