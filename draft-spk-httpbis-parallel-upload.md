---
title: "Parallel Appends for Resumable Uploads" abbrev: "Parallel Resumable Uploads" category: std docname: draft-spk-httpbis-parallel-upload-latest submissiontype: IETF number: date: consensus: true v: 3 area: "Web and Internet Transport" workgroup: "HTTP" keyword: ["resumable upload", "parallel upload", "intermediary", "content inspection"] venue: {group: "HTTP", type: "Working Group", mail: "ietf-http-wg@w3.org", arch: "https://lists.w3.org/Archives/Public/ietf-http-wg/", github: "santoshpallagatti/draft-spk-httpbis-parallel-upload", latest: "https://santoshpallagatti.github.io/draft-spk-httpbis-parallel-upload/draft-spk-httpbis-parallel-upload.html"} author: [{fullname: "Santosh Pallagatti", organization: "Zscaler", email: "santosh.pallagatti@gmail.com"}] normative: {RESUMABLE: I-D.ietf-httpbis-resumable-upload, RFC9000:, RFC9110:, RFC9112:, RFC9113:, RFC9114:, RFC9457:, RFC9530:, RFC9651:} informative: {BYTERANGE: I-D.ietf-httpapi-patch-byterange, RFC8470:, RFC9218:, TUS: {title: "tus resumable upload protocol, version 1.0.0", target: "https://tus.io/protocols/resumable-upload", author: [{org: "tus project"}], date: 2016-03-25}}

--- abstract

The resumable upload protocol lets a client continue an interrupted upload, but the upload has to be sent in order, one request at a time. This document extends that protocol so that a client can send different parts of the same upload in parallel. The amount of data that may arrive out of order is limited by a reorder window, so servers and intermediaries only need to buffer a bounded amount. The document also describes how intermediaries that inspect uploads (for data loss prevention or malware scanning, say) can continue to do so, and how the extension behaves over HTTP/1.1, HTTP/2, and HTTP/3.

--- middle

Introduction

The resumable upload protocol {{RESUMABLE}} lets a client continue an interrupted upload instead of starting over. Its state is a single offset: all bytes before the offset have been stored, and each append has to start exactly at that offset. {{Section 4.6 of RESUMABLE}} forbids a client from transferring data for the same upload in parallel.

Sequential transfer limits throughput. A single TCP or QUIC connection is limited by its congestion window and round-trip time, and streams multiplexed on one HTTP/2 or HTTP/3 connection share a single congestion controller. On long or lossy paths, one connection often uses a fraction of the available capacity. Sequential appends also add an idle round trip between appends, and one slow append delays everything after it.

Parallel uploads are widely deployed, but only in service-specific forms. Cloud storage APIs, and the Concatenation extension of the tus protocol {{TUS}}, upload parts as separate resources and join them at the end. There is no standard way to send parts of a single upload resource in parallel. The Byte Range PATCH draft {{BYTERANGE}} mentions parallel uploads and leaves them for "a later document".

This document defines such an extension for {{RESUMABLE}}. A client asks for parallel mode when it creates the upload ({{negotiation}}) and then sends appends for different byte ranges at the same time, within a window ({{appends}}). The server reports which ranges it has in a new response field ({{upload-received}}), and {{completion}} says when the upload is complete. Requirements for servers and for intermediaries are in {{server-reqs}} and {{intermediary-reqs}}.

Unless this document says otherwise ({{updates}}), all rules of {{RESUMABLE}} still apply. A client or server that does not implement this extension simply keeps using the sequential protocol.

Use Cases {#use-cases}

The main motivation is large uploads over paths where a single connection cannot use the available bandwidth: video and audio, disk images and backups, and datasets or model files that often run to several gigabytes.

Uploads to AI services are a growing case. Users attach documents, recordings, and video for a model to analyze, and the model cannot start until the upload has finished, so upload time shows up directly as waiting time. For small files the gain from parallel appends is small, because connection setup and round trips dominate. For large inputs it can be significant.

The same trend affects organizations that inspect outbound uploads, including uploads to AI services, to keep sensitive data from leaving their networks. Their proxies have to handle more and larger uploads with limited memory, and must not let a file be committed before it has been checked. {{inspecting}} describes how they can do that.

Mobile clients benefit as well. Their connections drop and change often, and with this extension only the missing ranges need to be sent again.

Conventions and Definitions

{::boilerplate bcp14-tagged}

This document uses the terms "upload resource", "append", "upload creation", "offset retrieval", and "upload cancellation" as defined in {{RESUMABLE}}, and the terms "client", "server", "intermediary", "proxy", and "gateway" as defined in {{RFC9110}}. Structured Field types (Boolean, Integer, Token, List, Inner List, Dictionary) are defined in {{RFC9651}}.

The following terms are also used:

Parallel mode: : The mode of an upload for which the extension defined in this document has been negotiated.

Contiguous offset (C): : The number of bytes, starting at byte 0, that have been received without a gap. This is the value that {{RESUMABLE}} conveys in the Upload-Offset field.

Reorder window (W): : The maximum distance, in bytes, beyond the contiguous offset at which an append may end. See {{window}}.

Received ranges: : The set of byte ranges of the upload that the server has received and stored.

Tail append: : An append that ends at the upload length and carries Upload-Complete set to true. Without it the upload cannot complete.

Inspecting intermediary: : An intermediary that examines the content of an upload before allowing it to reach or be committed at the server, typically to apply data loss prevention (DLP) or malware scanning policy.

Byte ranges in this document are written as [start, end), meaning the range includes start and excludes end. "MiB" means 1,048,576 bytes.

Overview

A client asks for parallel mode when it creates an upload. The server, and any intermediary on the path, answer with the limits they accept: how many appends may be in progress at once and how large the reorder window is. Intermediaries can only make these limits stricter.

The client then sends several appends at the same time, each covering its own byte range. Every append has to end within the reorder window, which starts at the contiguous offset. As gaps at the start of the window fill, the contiguous offset advances and the window slides forward. The server reports the ranges it holds in the Upload-Received field, so a client that loses an append re-sends only the missing range. The upload completes when the received ranges cover the whole upload.

ascii
 0                 C                              C+W        Length
 |=================|----+=====+------+=====+------|----------|
   all received     gap  recv   gap    recv  gap    outside
                   |<------ reorder window ------>|  window

{: #fig-window title="The Reorder Window"}

{{fig-overview}} shows a 12 MiB upload sent as 2 MiB appends with a window of 6 MiB and at most three appends in progress.

ascii
Client                                   Server
  |  POST /files                            |
  |  Upload-Parallel: ?1                    |
  |  Upload-Length: 12582912                |
  |---------------------------------------->|
  |  201 Created                            |
  |  Location: /uploads/abc                 |
  |  Upload-Limit: max-parallel=3,          |
  |                reorder-window=6291456   |
  |<----------------------------------------|
  |  PATCH, Upload-Offset: 0         ------>|  \
  |  PATCH, Upload-Offset: 2097152   ------>|   > concurrent,
  |  PATCH, Upload-Offset: 4194304   ------>|  /  window [0, 6 MiB)
  |<------------------ 204 (three times) ---|
  |  PATCH, Upload-Offset: 6291456   ------>|  \
  |  PATCH, Upload-Offset: 8388608   ------>|   > window slid to
  |  PATCH, Upload-Offset: 10485760  ------>|  /  [6 MiB, 12 MiB)
  |         Upload-Complete: ?1             |
  |<----------- 204, 204, 201 Created ------|  upload complete

{: #fig-overview title="Parallel Upload Overview"}

Inspecting intermediaries use the same window to process content in order with bounded memory, and hold the tail append until they reach a verdict ({{inspecting}}).

Negotiating Parallel Mode {#negotiation}
The Upload-Parallel Request Field {#upload-parallel}

The Upload-Parallel request header field is a Boolean {{RFC9651}}. A client sends it with the value true in an upload creation request to ask for parallel mode:

http
Upload-Parallel: ?1

A client that sends Upload-Parallel with the value true MUST also send Upload-Length in the same request. Parallel mode requires the total length to be known before appends arrive out of order. A server that receives Upload-Parallel without Upload-Length MUST treat the upload as sequential.

A client MAY create the upload with an empty body (Upload-Complete set to false and no content) so that it learns the upload resource and the limits before it sends any data. This is RECOMMENDED when HTTP/1.1 is used ({{http11}}).

Upload-Limit Parameters {#limits}

This document defines the following parameters for the Upload-Limit response field of {{RESUMABLE}}:

max-parallel: : An Integer. The maximum number of appends to this upload that may be in progress at the same time. A value of 1, or the absence of the parameter, means the upload is sequential.

reorder-window: : An Integer. The reorder window W in bytes. If absent while max-parallel is greater than 1, the window is unlimited, that is, equal to the upload length.

append-alignment: : An Integer. The offset of every append, and the length of every append except the tail append, MUST be a multiple of this value. If absent, no alignment applies.

commit: : A Token, either "auto" or "explicit". See {{completion}}. If absent, the value is "auto". {:vspace}

A server that accepts parallel mode MUST include max-parallel with a value greater than 1 in the Upload-Limit field of the upload creation response and of every offset retrieval response.

Tightening by Intermediaries {#tightening}

An intermediary MAY modify the Upload-Limit field in a response it forwards, but only in ways that make the limits stricter:

it MAY lower max-parallel, including to 1;
it MAY lower reorder-window, or add it if absent;
it MAY replace append-alignment with a multiple of the current value, or add it if absent;
it MAY change commit from "auto" to "explicit".

An intermediary MUST NOT raise max-parallel or reorder-window, remove any of these parameters, or change commit from "explicit" to "auto". Because each hop can only tighten the limits, the values a client receives are the strictest on the path.

Falling Back to Sequential Uploads {#fallback}

The upload is in parallel mode only if the client sent Upload-Parallel with the value true and the response to upload creation contains max-parallel greater than 1. Otherwise, client and server MUST follow {{RESUMABLE}} without the changes in this document. Limits can also tighten later: a client MUST apply the values from the most recent offset retrieval or append response.

Parallel Appends {#appends}
The Reorder Window {#window}

Let C be the contiguous offset, L the upload length, and W the reorder window. In parallel mode, an append that covers bytes [o, o + n), where n is the length of its content, is inside the window if and only if:

ascii
    o + n <= min(C + W, L)

A client MUST NOT start an append that is outside the window as computed from the most recent Upload-Offset it has received. Because C only grows, an append that was inside the window when it started stays inside it.

A window equal to L allows unrestricted parallelism. A window equal to the size of one append results in sequential behavior while still permitting a new append to start as soon as the previous one completes.

Append Requests {#append-requests}

Appends in parallel mode use the PATCH method and the application/partial-upload media type, as defined in {{RESUMABLE}}. The Upload-Offset request field carries the offset o at which the content starts; the content length gives n.

In parallel mode, Upload-Offset in an append request is not required to equal the server's current offset. It is the start of the byte range the append writes. A client MUST send Content-Length on every append in parallel mode, so that a recipient knows the full range before the content arrives.

A client MUST NOT have more than max-parallel appends to the same upload in progress at once. An append is in progress from when its request starts until its response is received or the request is abandoned.

A client SHOULD include a Content-Digest field {{RFC9530}} on each append so that the server and intermediaries can verify each range on its own.

http
PATCH /uploads/abc HTTP/1.1
Host: example.com
Content-Type: application/partial-upload
Upload-Offset: 4194304
Upload-Complete: ?0
Content-Length: 2097152
Content-Digest: sha-256=:(base64 digest):

[2097152 bytes]
Overlapping and Repeated Ranges {#overlap}

An append may overlap ranges the server has already received, for example when a client retries an append whose response it did not receive. Retrying a range with identical content is always safe.

A server MAY discard overlapping bytes without comparing them. If a server detects that overlapping bytes differ from the stored bytes (using Content-Digest, say), it MUST reject the append with status 409 (Conflict) and the problem type "range-conflict" ({{problems}}).

If a new append overlaps a range that is being written by an append still in progress, the server SHOULD terminate the older append before processing the new one. This applies the rule of {{Section 4.6 of RESUMABLE}} per byte range instead of per upload: the newer request is taken to be a retry of the older one. Bytes the older append stored before termination remain received.

Interrupted Appends {#interrupted}

When an append is interrupted, the server MAY keep the bytes it received as a received range, as in {{RESUMABLE}}. In parallel mode, this can leave a gap after the kept bytes. The client learns about the gap through offset retrieval ({{upload-received}}) and re-sends only the missing bytes.

Reporting Received Ranges {#upload-received}

The Upload-Received response header field is a List of Inner Lists {{RFC9651}}. Each Inner List contains exactly two Integers, the start and end of a received range [start, end):

http
Upload-Offset: 1048576
Upload-Received: (0 1048576), (2097152 6291456)

The ranges MUST be listed in ascending order and MUST NOT overlap or touch; adjacent ranges are merged. If the contiguous offset is greater than 0, the first range starts at 0 and ends at the contiguous offset.

In parallel mode, a server MUST include Upload-Received in every offset retrieval response and SHOULD include it in responses to appends. The Upload-Offset field keeps the meaning defined in {{RESUMABLE}}: it is the contiguous offset, and it never decreases.

A server MUST NOT list a range it has not stored. A server MAY omit ranges beyond the contiguous offset, for example to limit the size of the field; the client then re-sends those bytes, which is safe ({{overlap}}).

Upload Completion {#completion}

In parallel mode an upload is complete when its received ranges cover [0, L) and the server has received the tail append. The tail append is the append that carries Upload-Complete with the value true. Every other append MUST carry Upload-Complete with the value false.

Automatic Commit {#commit-auto}

When commit is "auto", the tail append is the data append that ends at L. A client MUST set Upload-Complete to true on the append that ends at L and on no other append. A server that receives Upload-Complete set to true on an append that does not end at L MUST reject it with status 400 (Bad Request).

If gaps remain when the tail append has been received, the server responds to the tail append with 204 (No Content) and the current Upload-Offset and Upload-Received. The upload completes when the last gap is filled. The append that fills it receives the final response for the upload, as described in {{RESUMABLE}}. Other appends receive 204 (No Content).

Explicit Commit {#commit-explicit}

When commit is "explicit", every data append, including the one that ends at L, carries Upload-Complete set to false. After the client has received successful responses for all ranges, it sends an append with empty content, Upload-Offset set to L, and Upload-Complete set to true. That empty append is the tail append and receives the final response.

If the server receives the tail append while gaps remain, it MUST reject it with status 409 (Conflict), the problem type "incomplete-ranges", and the Upload-Received field.

Explicit commit separates receiving the data from committing it. It lets a client check the whole representation, using Repr-Digest {{RFC9530}}, before committing. It also keeps the tail append small, which helps intermediaries that hold it ({{hold-tail}}).

Changes to Resumable Upload Rules in Parallel Mode {#updates}

For uploads in parallel mode only, this document changes the following rules of {{RESUMABLE}}. Section numbers refer to draft version 12. All other rules apply unchanged.

Rule in RESUMABLE	Change in parallel mode
Section 4.6: a client must not transfer data for one upload in parallel	Up to max-parallel appends may be in progress ({{append-requests}}).
Section 4.6: a new request terminates the older one	Applies per overlapping byte range ({{overlap}}).
Section 4.4.2: Upload-Offset must match the current offset, otherwise 409	Any append inside the reorder window is accepted ({{window}}).
Upload-Complete marks the final append	Marks the tail append; completion requires full coverage ({{completion}}).
{: #tab-updates title="Rules Changed in Parallel Mode"}	

{{RESUMABLE}} is sequential for a good reason. Without negotiation, a server cannot tell a deliberate concurrent append from a stale request that the client gave up on, such as one left behind on a half-open connection. Negotiating parallel mode removes that ambiguity, and applying the termination rule per range keeps the protection against stale requests.

Client Behavior {#client-reqs}
A client SHOULD choose append sizes that are multiples of append-alignment and small enough that several fit inside the reorder window.
A client SHOULD keep the append that starts at the contiguous offset in progress at all times, because the window cannot slide until that range arrives.
If the append at the contiguous offset is much slower than the others, a client MAY send the same range again on another stream or connection and cancel the other copy when one finishes. Both copies count against max-parallel while in progress.
After any failed or abandoned append, a client SHOULD perform offset retrieval and re-send only ranges missing from Upload-Received.
A client that receives status 409 with the problem type "offset-outside-window" MUST wait until the contiguous offset has advanced before retrying that range.
A client SHOULD send Repr-Digest {{RFC9530}} for the whole representation, either at upload creation or on the tail append.
Server Requirements {#server-reqs}
Upload State {#server-state}

A server that supports parallel mode MUST track the received ranges of each upload in parallel mode, and update them atomically: a range is either fully recorded or not recorded at all. Offset retrieval responses MUST reflect every range recorded before the request was received.

When requests for one upload can reach different server instances, for example behind a load balancer, all instances MUST share a consistent view of the received ranges, or requests for an upload MUST be routed to a single instance.

Enforcing Limits {#server-window}

A server MUST reject an append that is outside the reorder window with status 409 (Conflict), the problem type "offset-outside-window", and the current Upload-Offset, unless it delays the append as described in {{h2h3-flow}}.

A server MAY reject an append that exceeds max-parallel with status 429 (Too Many Requests), the problem type "too-many-parallel", and a Retry-After field.

A server MUST reject an append whose offset or length violates append-alignment with status 400 (Bad Request) and the problem type "misaligned-append".

A server MAY lower its limits during an upload, for example under load, and signals new values in its next responses. It MUST NOT reject an append that was inside the limits it last advertised when the append started solely because the limits changed.

Storage and Visibility {#server-storage}

A server MAY write ranges directly into a single sparse object or store them separately and join them on completion. The choice is not visible to clients.

Until an upload is complete, a server MUST NOT make its content available as the representation of any target resource, and MUST NOT fill gaps with placeholder bytes that could be mistaken for uploaded content. An upload that is cancelled or expires MUST be discarded without being committed.

If a Repr-Digest was provided, the server SHOULD verify it against the joined content before committing and MUST NOT commit on a mismatch.

Resource Management {#server-resources}

The reorder window bounds how much out-of-order data a server has to buffer per upload. A server that buffers in memory SHOULD choose reorder-window and max-parallel according to its memory budget, and SHOULD also limit the number of concurrent uploads per client.

Intermediary Requirements {#intermediary-reqs}
All Intermediaries {#all-intermediaries}

Appends in parallel mode are ordinary HTTP requests. An intermediary that does not implement this document forwards them without change, and parallel mode works through it.

An intermediary that implements this document:

MAY tighten Upload-Limit as described in {{tightening}}, and MUST NOT loosen it;
MAY force sequential behavior by removing Upload-Parallel from an upload creation request or by setting max-parallel to 1 in the response;
MUST NOT remove or alter Upload-Received, except as described in {{hold-tail-retrieval}};
when it balances load across servers, MUST meet the consistency requirement of {{server-state}}, for example by routing all requests for an upload resource to the same server.

Responses to appends and to offset retrieval are not cacheable, as specified in {{RESUMABLE}}; this is unchanged.

Terminating Parallel Mode at an Intermediary {#translation}

An intermediary MAY accept parallel mode from clients while forwarding the upload to the server sequentially, for example because the server does not support this extension or is reached over a different HTTP version. Such an intermediary acts as the server for parallel mode on the client-facing hop: it MUST meet the requirements of {{server-reqs}} for that hop, buffer out-of-order data within the window it advertises, and forward bytes upstream in order as the contiguous offset advances.

Inspecting Intermediaries {#inspecting}

An inspecting intermediary examines upload content before it reaches, or is committed at, the server. Such intermediaries only see content where they already terminate TLS, usually under an enterprise policy. This document does not change that.

In-Order Processing {#in-order}

Many inspection engines need content in order: file types are recognized from the first bytes, and patterns can span append boundaries. An inspecting intermediary that processes content in order:

SHOULD advertise a reorder-window no larger than the buffer it can dedicate to the upload;
buffers appends that arrive ahead of the contiguous offset and passes bytes to its engines in order as gaps fill;
MUST NOT evaluate appends only in isolation when its policy can match content that spans appends;
MAY use Content-Digest to recognize a retried range it has already inspected and skip inspecting it again.

Some formats can only be analyzed as a whole, such as archives that keep their index at the end. For those, an intermediary MAY set reorder-window to the upload length and inspect the complete content before releasing the tail.

Forwarding Strategies {#forwarding}

The RECOMMENDED approach is to forward every append as soon as it arrives, inspect the content in order alongside, and hold back only the tail append until there is a verdict ({{hold-tail}}). Upload throughput is unaffected; the only added delay is the time needed to inspect the last data received.

An intermediary that does not want the server to store uninspected bytes at all can instead forward each range only after inspecting it, at the cost of more delay. It can also forward the upload sequentially, as described in {{translation}}.

Holding the Tail Append {#hold-tail}

Whatever forwarding strategy it uses, an inspecting intermediary MUST NOT forward the tail append to the server before its verdict for the upload is that the content is allowed. Since a server cannot complete an upload without its tail append ({{completion}}), this is enough to keep uninspected content from being committed, even if the server knows nothing about inspection or about this document.

The intermediary always knows which append is the tail: it carries Upload-Complete set to true and, with automatic commit, ends at the upload length.

With automatic commit the tail append can be large. The intermediary MAY forward all of it except a final part, as an append with Upload-Complete set to false, and hold only that final part as the tail it forwards later. The final part SHOULD be one append-alignment unit if an alignment applies. With explicit commit the tail append is empty and nothing needs to be split.

An intermediary MAY set commit to "explicit" when tightening limits ({{tightening}}) so that the tail append is always empty.

Offset Retrieval While Holding {#hold-tail-retrieval}

While it holds the tail append, the intermediary SHOULD answer offset retrieval requests for the upload itself, or modify the server's response, so that the held range is reported as received and the Upload-Inspection field ({{upload-inspection}}) is included. Otherwise the client would see a gap at the end and re-send the tail, which is safe but wasteful.

Verdicts {#verdicts}

Allowed: : The intermediary forwards the tail append. It relays the server's final response to the client and adds Upload-Inspection with state "clean".

Blocked: : The intermediary MUST NOT forward the tail append. It SHOULD cancel the upload at the server using upload cancellation as defined in {{RESUMABLE}}, and responds to the client with status 403 (Forbidden) and the problem type "upload-blocked". Later requests for the upload receive the same response.

Pending: : If inspection takes longer than the intermediary is willing to hold the request, it MAY respond to the tail append with status 202 (Accepted), a Retry-After field, and Upload-Inspection with state "scanning". The client then uses offset retrieval until the response shows the upload as complete or blocked. {:vspace}

The Upload-Inspection Response Field {#upload-inspection}

The Upload-Inspection response header field is a Dictionary {{RFC9651}} that an inspecting intermediary uses to report progress:

http
Upload-Inspection: state=scanning, scanned=7340032

state: : A Token: "scanning", "clean", or "blocked".

scanned: : An Integer, optional: the offset up to which content has been inspected.

Only inspecting intermediaries set this field. A client MUST NOT treat it as a statement by the server, and MAY use it to explain delays to users. Unknown keys MUST be ignored.

Reusing Earlier Verdicts {#verdict-cache}

If the client sends Repr-Digest at upload creation, an inspecting intermediary MAY look up an earlier verdict for that digest under the same policy. A digest known to be blocked can be refused at upload creation, before any content is transferred. For a digest known to be allowed, the intermediary MAY skip its inspection engines, but MUST still compute the digest of the content as it passes and treat a mismatch as blocked.

HTTP Version Considerations {#versions}
HTTP/2 and HTTP/3 {#h2h3}

Over HTTP/2 {{RFC9113}} and HTTP/3 {{RFC9114}}, each append is a request on its own stream, and several appends can share one connection. Because streams on one connection share one congestion controller, a client MAY also open additional connections, within max-parallel, when a single connection does not use the available capacity.

Priorities {#h2h3-priority}

A client SHOULD use the Priority field {{RFC9218}} to give the append that starts at the contiguous offset a lower urgency value (that is, higher importance) than appends further into the window, for example u=1 for the first and u=3 to u=5 for later ones. Servers and intermediaries MAY send PRIORITY_UPDATE frames to adjust this. Delivering the start of the window first lets the window slide sooner and reduces the data buffered out of order. Priorities are hints; the protocol works without them.

Flow Control {#h2h3-flow}

A request's header section, which carries Upload-Offset and Content-Length, arrives before its content. A recipient therefore knows the range of an append before any of its content arrives, and can use per-stream flow control to delay an append that is outside the window instead of rejecting it:

For an append inside the window, the recipient grants flow control credit as usual (WINDOW_UPDATE in HTTP/2, MAX_STREAM_DATA {{RFC9000}} in HTTP/3).
For an append outside the window, the recipient withholds credit beyond the initial stream window until the window reaches the append. The sender pauses on that stream only.
The recipient MUST continue to grant connection-level credit to other streams, so that a delayed stream does not stall them.
A recipient MAY reject an append that has been delayed longer than it is willing to wait, as described in {{server-window}}.

A delayed stream can still send up to its initial stream window before pausing. A recipient that delays appends can lower its initial stream window to bound that data.

ascii
Client                           Recipient
  | HEADERS 1, Upload-Offset: 0       |  C = 0,
  | HEADERS 3, Upload-Offset: 2097152 |  window [0, 6 MiB)
  | HEADERS 5, Upload-Offset: 8388608 |
  |---------------------------------->|
  |      WINDOW_UPDATE streams 1, 3   |  stream 5 is outside
  |<----------------------------------|  the window: no
  | DATA streams 1, 3                 |  further credit
  |---------------------------------->|
  |                                   |  C reaches 4 MiB,
  |      WINDOW_UPDATE stream 5       |  window [4, 10 MiB)
  |<----------------------------------|
  | DATA stream 5                     |
  |---------------------------------->|

{: #fig-flow title="Delaying an Append with Flow Control (W = 6 MiB)"}

Cancellation and Early Data {#h2h3-other}

A client cancels a redundant or abandoned append with RST_STREAM in HTTP/2 or by resetting the stream in HTTP/3. The server treats it as an interrupted append ({{interrupted}}).

Upload creation and appends MUST NOT be sent in TLS early data. A server that receives them in early data responds with status 425 (Too Early) {{RFC8470}}.

HTTP/1.1 {#http11}

Over HTTP/1.1 {{RFC9112}}, parallel appends use separate connections, one append per connection at a time. The following rules apply:

A client MUST NOT use more than max-parallel connections for appends to one upload. Browsers are additionally limited by their per-origin connection limits.
HTTP/1.1 has no per-request flow control. A client MUST NOT start an append outside the window, and a recipient SHOULD reject such an append with 409 and "offset-outside-window" rather than delay it.
HTTP/1.1 has no priorities. A client keeps the append at the contiguous offset in progress first and MAY send it again on a second connection if it stalls ({{client-reqs}}).
Some intermediaries do not forward informational responses. A client MUST NOT depend on receiving 104 (Upload Resumption Supported) and SHOULD create the upload with an empty body, taking the upload resource and limits from the final response.
A client MUST NOT pipeline requests.
Appends in parallel mode carry Content-Length ({{append-requests}}) and therefore do not use chunked transfer coding.
A client MAY send Expect: 100-continue, and MUST continue if no 100 (Continue) response arrives, as described in {{RFC9110}}.

If max-parallel is absent or 1, client and server use the sequential behavior of {{RESUMABLE}} unchanged.

Problem Types {#problems}

Error responses defined in this document use Problem Details {{RFC9457}} with the following problem types. Each type URI has the form https://iana.org/assignments/http-problem-types#NAME.

Name	Status	Meaning
offset-outside-window	409	The append ends beyond the reorder window. The response includes Upload-Offset.
range-conflict	409	The append overlaps received bytes with different content.
incomplete-ranges	409	An explicit commit was sent while gaps remain. The response includes Upload-Received.
misaligned-append	400	The offset or length violates append-alignment.
too-many-parallel	429	More than max-parallel appends are in progress.
upload-blocked	403	An inspecting intermediary blocked the upload by policy.
{: #tab-problems title="Problem Types"}		

Problem details for "upload-blocked" MUST NOT identify the inspection engine, signature, or policy rule that matched.

Security Considerations

The security considerations of {{RESUMABLE}} apply to this extension as well.

A client cannot get more parallelism than every party on the path allows, because intermediaries can only tighten the limits ({{tightening}}). An inspecting intermediary controls whether an upload is committed by holding its tail append ({{hold-tail}}), so content cannot be committed without passing through any inspection on the path.

An attacker may try to split content across appends so that no single append matches an inspection policy. Processing appends in order, across append boundaries ({{in-order}}), handles this. Evaluating each append on its own does not.

A digest sent by the client is only a claim. Servers and intermediaries can use it to detect errors or to look up earlier verdicts, but have to verify it against the actual content before relying on it ({{verdict-cache}}).

The reorder window bounds how much out-of-order data is buffered per upload, but an attacker can still open many uploads. Servers and intermediaries need to limit the number of uploads per client, the total amount of buffered data, and how long they hold delayed appends. They can respond with 429 (Too Many Requests) when overloaded.

Incomplete uploads contain gaps and possibly uninspected data. They are never exposed as the representation of a resource ({{server-storage}}), and gaps are never filled with bytes that could reveal other stored data.

Detailed inspection results would help an attacker adjust content until it passes. For that reason Upload-Inspection and "upload-blocked" responses carry only coarse information ({{problems}}).

Parallel mode sends the upload resource URI over more connections. As in {{RESUMABLE}}, the URI needs to be hard to guess, and access to it needs to be tied to the client's authorization.

Appends are not sent in TLS early data ({{h2h3-other}}), and repeating an append with the same content has no effect, so replay does not cause harm.

IANA Considerations
HTTP Field Names

IANA is asked to register the following fields in the "Hypertext Transfer Protocol (HTTP) Field Name Registry" established by {{RFC9110}}:

Field Name	Status	Structured Type	Reference
Upload-Parallel	permanent	Item	{{upload-parallel}} of this document
Upload-Received	permanent	List	{{upload-received}} of this document
Upload-Inspection	permanent	Dictionary	{{upload-inspection}} of this document
{: #tab-iana-fields title="Field Registrations"}			
Upload-Limit Parameters

If {{RESUMABLE}} establishes a registry of Upload-Limit parameters, IANA is asked to register max-parallel, reorder-window, append-alignment, and commit, with a reference to {{limits}} of this document.

HTTP Problem Types

IANA is asked to register the problem types listed in {{tab-problems}} in the "HTTP Problem Types" registry established by {{RFC9457}}, each with a reference to {{problems}} of this document.

--- back

Acknowledgments

{:numbered="false"}

This document builds on the work of the authors of {{RESUMABLE}} and {{BYTERANGE}}, and on the tus protocol {{TUS}}.
