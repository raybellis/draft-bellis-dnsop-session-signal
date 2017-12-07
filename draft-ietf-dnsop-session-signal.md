---
title: DNS Stateful Operations
docname: draft-ietf-dnsop-session-signal-05
date: 2017-11-30
ipr: trust200902
area: Internet
wg: DNSOP Working Group
kw: Internet-Draft
cat: std
updates: RFC 7766, RFC 1035

coding: utf-8
pi:
  - toc
  - symrefs
  - sortrefs

author:
  -
    ins: R. Bellis
    name: Ray Bellis
    org: Internet Systems Consortium, Inc.
    abbrev: ISC
    street: 950 Charter Street
    city: Redwood City
    code: CA 94063
    country: USA
    phone: +1 650 423 1200
    email: ray@isc.org
  -
    ins: S. Cheshire
    name: Stuart Cheshire
    org: Apple Inc.
    street: 1 Infinite Loop
    city: Cupertino
    code: CA 95014
    country: USA
    phone: +1 408 974 3207
    email: cheshire@apple.com
  -
    ins: J. Dickinson
    name: John Dickinson
    org: Sinodun Internet Technologies
    abbrev: Sinodun
    street:
    - Magadalen Centre
    - Oxford Science Park
    city: Oxford
    code: OX4 4GA
    country: United Kingdom
    email: jad@sinodun.com
  -
    ins: S. Dickinson
    name: Sara Dickinson
    org: Sinodun Internet Technologies
    abbrev: Sinodun
    street:
    - Magadalen Centre
    - Oxford Science Park
    city: Oxford
    code: OX4 4GA
    country: United Kingdom
    email: sara@sinodun.com
  -
    ins: A. Mankin
    name: Allison Mankin
    org: Salesforce
    email: allison.mankin@gmail.com
  -
    ins: T. Pusateri
    name: Tom Pusateri
    org: Unaffiliated
    street: ""
    city: Raleigh
    code: NC 27608
    country: USA
    phone: +1 919 867 1330
    email: pusateri@bangj.com

informative:
  NagleDA:
    target: http://www.stuartcheshire.org/papers/nagledelayedack/
    title: TCP Performance problems caused by interaction between Nagle's Algorithm and Delayed ACK
    author:
      name: Stuart Cheshire
      ins: S. Cheshire
    date: 2005-05-20

--- abstract

This document defines a new DNS OPCODE, for DNS Stateful Operations (DSO).
DSO messages are used to communicate operations within persistent
stateful sessions, expressed using type-length-value (TLV) syntax.
This document defines an initial set of TLVs, including ones
used to manage session timeouts and termination.
This mechanism is intended to reduce the overhead of existing 
“per-packet” signaling mechanisms with “per-message” semantics as well as 
defining new stateful operations not defined in EDNS(0). 

--- middle

# Introduction

The use of transports for DNS other than UDP is being increasingly specified,
for example, DNS over TCP {{!RFC1035}}{{!RFC7766}} and DNS over TLS {{?RFC7858}}.
Such transports can offer persistent, long-lived sessions and therefore when
using them for transporting DNS messages it is of benefit to have a mechanism
that can establish parameters associated with those sessions, such as timeouts.
In such situations it is also advantageous to support server initiated messages.

The existing EDNS(0) Extension Mechanism for DNS {{!RFC6891}} is explicitly
defined to only have "per-message" semantics. Whilst EDNS(0) has been used to
signal at least one session-related parameter (the EDNS(0) TCP Keepalive option
{{?RFC7828}}) the result is less than optimal due to the restrictions
imposed by the EDNS(0) semantics and the lack of server-initiated signalling. 
For example, a server cannot arbitrarily 
instruct a client to close a connection because the server can only send EDNS(0) options 
in responses to queries that contained EDNS(0) options.

This document defines a new DNS OPCODE, for DNS Stateful Operations (DSO).
DSO messages are used to communicate operations within persistent
stateful sessions, expressed using type-length-value (TLV) syntax.
This document defines an initial set of TLVs, including ones
used to manage session timeouts and termination.

All three of the TLVs defined here are mandatory for all implementations of DSO.
Additional TLVs may be defined in additional specifications.

It should be noted that the message format for DNS Stateful Operations
(see {{format}}) differs from the traditional DNS packet
format used for standard queries and responses.
The standard twelve-octet header is used, but the four count fields
(QDCOUNT, ANCOUNT, NSCOUNT, ARCOUNT) are set to zero and their
corresponding sections are not present.
The actual data pertaining to DNS Stateful Operations
(expressed in TLV format) is appended to the end of the DNS message header.
When displayed using packet analyzer tools that have not been
updated to recognize the DNS Stateful Operations format, this
will result in the Stateful Operations data being displayed
as unknown additional data after the end of the DNS message.
It is likely that future updates to these tools will add the ability
to recognize, decode, and display the Stateful Operations data.

This new format has distinct advantages over an RR-based format because it
is more explicit and more compact. Each TLV definition is specific
to the use case, and as a result contains no redundant or overloaded fields.
Importantly, it completely avoids conflating DNS Stateful Operations in anyway 
with normal DNS operations or with existing EDNS(0) based functionality.
A goal of this approach is to avoid the operational issues that have
befallen EDNS(0), particularly relating to middle-box behaviour.

With EDNS(0), multiple options may be packed into a single OPT pseudo-RR,
and there is no generalized mechanism for a client to be able to tell
whether a server has processed or otherwise acted upon each individual
option within the combined OPT RR.
The specifications for each individual option need to define how each
different option is to be acknowledged, if necessary.

In contrast to EDNS(0), with DNS Stateful Operations there is no
compelling motivation to pack multiple operations into a single
message for efficiency reasons, because DNS Stateful Operations
always operates using a connection-oriented transport protocol.
Each Stateful operation is communicated in its own separate
DNS message, and the transport protocol can take care of packing
separate DNS messages into a single IP packet if appropriate.
For example, TCP can pack multiple small DNS messages into a single TCP segment.
This simplification allows for clearer semantics.
Each DSO request message communicates just one primary operation,
and the RCODE in the corresponding response message indicates the
success or failure of that operation.

# Terminology

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT",
"SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and
"OPTIONAL" in this document are to be interpreted as described in
"Key words for use in RFCs to Indicate Requirement Levels" {{!RFC2119}}.

"DSO" is used to mean DNS Stateful Operation.

The term "connection" means a bidirectional byte stream of reliable,
in-order messages, such as provided by using
DNS over TCP {{!RFC1035}}{{!RFC7766}} or DNS over TLS {{?RFC7858}}.

The unqualified term "session" in the context of this document means the exchange of
DNS messages over a connection where:

- The connection between client and server is persistent and relatively
  long-lived (i.e., minutes or hours, rather than seconds).
- Either end of the connection may initiate messages to the other.

A "DSO Session" is established between two endpoints that
acknowledge persistent DNS state via the exchange of DSO messages
over the connection. This is distinct from, for example a DNS-over-TCP session
as described in RC7766.

A "DSO Session" is terminated when the underlying connection is closed.

The term "server" means the software with a listening socket, awaiting
incoming connection requests.

The term "client" means the software which initiates a connection
to the server's listening socket.

The terms "initiator" and "responder" correspond respectively to the
initial sender and subsequent receiver of a DSO request message,
regardless of which was the "client" and "server" in the usual DNS sense.

The term "sender" may apply to either an initiator
(when sending a DNS Stateful Operation request message)
or a responder (when sending a DNS Stateful Operation response message).

Likewise, the term "receiver" may apply to either a responder
(when receiving a DNS Stateful Operation request message)
or an initiator (when receiving a DNS Stateful Operation response message).

DNS Stateful Operations are expressed using type-length-value (TLV) syntax.

Two timers (elapsed time since an event) are defined in this document: 

* an inactivity timer (see {{keepalive}} and {{inactivetimer}})
* a keepalive timer (see {{keepalive}} and {{keepalivetimer}})

The timeouts associated with these timers are called the inactivity timeout and 
the keepalive interval respectively. 
The term "Session Timeouts" is used to refer to this pair of timeout values.

Reseting a timer means resetting the timer value to zero and starting the timer 
again. Clearing a timer means resetting the timer value to zero but NOT starting 
the time again. 

# Discussion

There are several use cases for DNS Stateful operations that can 
be described here. 

Firstly, establishing session parameters such as server defined timeouts is of 
great use in the 
general management of persistent connections. For example, using DSO sessions 
for stub to recursive DNS-over-TLS {{?RFC7858}} is more flexible for both the 
client and the server than attempting to manage sessions using just the EDNS(0)
TCP Keepalive option
{{?RFC7828}}. The simple set of TLVs defined in this document is
sufficient to greatly enhance connection management for this use case.

Secondly, DNS-SD {{?RFC6763}} has evolved into a naturally session based mechanism where,
for example, long-lived subscriptions lend themselves to 'push' mechanisms as 
opposed to polling. Long-lived stateful connections and server initiated 
messages align with this use case {{?I-D.ietf-dnssd-push}}.

A general use case is that DNS traffic is often bursty but session establishment 
can be expensive. One challenge with long-lived connections is to maintain 
sufficient traffic to maintain NAT and firewall state.
To mitigate this issue this document introduces a
new concept for the DNS, that is DSO "Keepalive traffic".
This traffic carries no DNS data and is not considered 'activity'
in the classic DNS sense, but serves to maintain state in middle boxes,
and to assure client and server that they still have connectivity to each other.

There are a myriad of other potential use cases for DSO given the versatility 
and extensibility of this specification.

{{details}} of this document describes the protocol details of DNS Stateful
Operations including definitions of three TLVs for session management and 
encryption padding. {{lifecycle}} presents a
detailed discussion of the DSO Session lifecycle including an
in-depth discussion of keepalive traffic and session termination.

# Protocol Details {#details}

## DSO Session Establishment {#establishment}

DSO messages MUST only be carried in protocols and in
environments where a session may be established according to the definition above.
Standard DNS over TCP {{!RFC1035}}{{!RFC7766}}, and DNS over TLS {{?RFC7858}}
are suitable protocols.

DNS over plain UDP {{?RFC0768}} is not appropriate since it fails on the requirement for
in-order message delivery, and, in the presence of NAT gateways and firewalls
with short UDP timeouts, it fails to provide a persistent bi-directional
communication channel unless an excessive amount of keepalive traffic is used.

In some environments it may be known in advance by external means
that both client and server support DSO, and in these cases either
client or server may initiate DSO messages at any time.

However, in the typical case a server will not know in advance whether a
client supports DSO, so in general, unless it is known in advance by other means
that a client does support DSO, a server MUST NOT initiate DSO request messages
until a DSO Session has been mutually established, as described below.
Similarly, unless it is known in advance by other means that a server
does support DSO, a client MUST NOT initiate non-response-requiring
DSO request messages until after a DSO Session has been mutually established.

Many, but not all, DSO request messages sent by an initiator
elicit a response from the responder.
Whether or not a given DSO request message elicits a response is
determined by whether or not the first DSO TLV (see {{tlvformat}})
in the message (the Primary TLV) is one that is specified to generate a response.

A DSO Session is established over a connection by the client
sending a DSO request message of a kind that requires a response,
such as the DSO Keepalive Operation TLV (see {{keepalive}}),
and receiving a response, with matching MESSAGE ID, and RCODE
set to NOERROR (0), indicating that the DSO request was successful.

If the RCODE is set to DSONOTIMP (tentatively 11) this indicates
that the server does support DSO, but does not support the particular
operation the client requested.
A server MUST NOT return DSONOTIMP for the DSO Keepalive Operation
TLV, but this could happen in the future, if a client attempts to
establish a DSO Session using a future response-requiring DSO TLV
which the server does not understand.
If the server returns DSONOTIMP then a DSO Session is not
considered established, but the client is permitted to continue
sending DNS messages on the connection,
including other response-requiring DSO messages such as the DSO Keepalive,
which may result in a successful NOERROR response,
yielding the establishment of a DSO Session.

If the RCODE is set to any value other than NOERROR (0) or DSONOTIMP
(tentatively 11), then the client should assume that the server does
not support DSO. In this case the client is permitted to continue
sending DNS messages on that connection, but the client SHOULD NOT
issue further DSO messages on that connection.

When the server receives a response-requiring DSO request message
from a client, and transmits a sucessful NOERROR response to that
request, the server considers the DSO Session established.

When the client receives the server's NOERROR response to its
DSO request message, the client considers the DSO Session established.

Once a DSO Session has been established,
either end may unilaterally send DSO messages at any time,
and therefore either client or server may be the initiator of a message.

Once a DSO Session has been established,
clients and servers should behave as described in this specification with
regard to inactivity timeouts and connection close, not as prescribed in
the previous specification for DNS over TCP {{!RFC7766}}.

### Middle-box Considerations

Where an application-layer middle box (e.g., a DNS proxy, forwarder, or
session multiplexer) is in the path the middle box MUST NOT blindly
forward DSO messages in either direction, and MUST treat the inbound
and outbound connections as separate sessions.  This does not preclude
the use of DSO messages in the presence of an IP-layer middle box such
as a NAT that rewrites IP-layer and/or transport-layer headers, but
otherwise preserves the effect of a single session between the client
and the server.

To illustrate the above, consider a network where a middle box
terminates one or more TCP connections from clients and multiplexes the
queries therein over a single TCP connection to an upstream server.  The
DSO messages and any associated state are specific to the individual
TCP connections.  A DSO-aware middle box MAY in some circumstances be
able to retain associated state and pass it between the client and
server (or vice versa) but this would be highly TLV-specific.  For
example, the middle box may be able to maintain a list of which clients
have made Push Notification subscriptions {{?I-D.ietf-dnssd-push}} and
make its own subscription(s) on their behalf, relaying any subsequent
notifications to the client (or clients) that have subscribed to that
particular notification.

## Message Format {#format}

A DSO message begins with
the standard twelve-octet DNS message header {{!RFC1035}}
with the OPCODE field set to the DSO OPCODE (tentatively 6).
However, unlike standard DNS messages, the question section, answer section,
authority records section and additional records sections are not present.
The corresponding count fields (QDCOUNT, ANCOUNT, NSCOUNT, ARCOUNT) MUST be
set to zero on transmission.

If a DSO message is received where any of the count fields are
not zero, then a FORMERR MUST be returned,
unless a future document specifies otherwise.

                                                 1   1   1   1   1   1
         0   1   2   3   4   5   6   7   8   9   0   1   2   3   4   5
       +---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+
       |                          MESSAGE ID                           |
       +---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+
       |QR |    OPCODE     |            Z              |     RCODE     |
       +---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+
       |                     QDCOUNT (MUST be zero)                    |
       +---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+
       |                     ANCOUNT (MUST be zero)                    |
       +---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+
       |                     NSCOUNT (MUST be zero)                    |
       +---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+
       |                     ARCOUNT (MUST be zero)                    |
       +---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+
       |                                                               |
       /                           DSO Data                            /
       /                                                               /
       +---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+

### Header {#header}

In a request the MESSAGE ID field MUST be set to a unique value, that the
initiator is not currently using for any other active operation on this 
connection.
For the purposes here, a MESSAGE ID is in use in this DSO Session if the
initiator has used it in a request for which it has not yet received a
response, or if the client has used it to setup state that it has not yet
deleted. For example, state could be a subscription {{?I-D.ietf-dnssd-push}}.

In a response the MESSAGE ID field MUST contain a copy of the value of the
MESSAGE ID field in the request being responded to.

In a request the DNS Header QR bit MUST be zero (QR=0).
If the QR bit is not zero the message is not a request.

In a response the DNS Header QR bit MUST be one (QR=1).
If the QR bit is not one the message is not a response.

The DNS Header OPCODE field holds the DSO OPCODE value (tentatively 6).

The Z bits are currently unused in DSO messages,
and in both DSO requests and DSO responses the
Z bits MUST be set to zero (0) on transmission and MUST be silently ignored
on reception, unless a future document specifies otherwise.

In a request message (QR=0) the RCODE is generally set to zero on transmission,
and silently ignored on reception, except where specified otherwise
(for example, the Retry Delay operation (see {{delay}}), where the RCODE indicates the reason
for termination).

The RCODE value in a response may be one of the following values:

| Code | Mnemonic | Description |
|-----:|----------|-------------|
| 0 | NOERROR | Operation processed successfully |
| 1 | FORMERR | Format error |
| 2 | SERVFAIL | Server failed to process request due to a problem with the server |
| 3 | NXDOMAIN | Name Error --- Named entity does not exist (TLV-dependent) |
| 4 | NOTIMP | DSO not supported |
| 5 | REFUSED | Operation declined for policy reasons |
| 9 | NOTAUTH | Not Authoritative (TLV dependent) |
| 11 | DSONOTIMP | DSO type code not supported |

Use of the above RCODEs is likely to be common in DSO but 
does not preclude the definition and use of other codes in future documents that 
make use of DSO.

If a document describing a DSO makes use of either NXDOMAIN (Name Error)
or NOTAUTH then that document MUST explain the meaning.

### DSO Data {#dsodata}

The standard twelve-octet DNS message header is followed by the DSO Data.

The first TLV in a DSO request message is called the Operation 
TLV. Any subsequent TLVs after this initial Operation TLV are called Modifier TLVs.

Depending on the operation a DSO response can contain:

* No TLVs
* Only an Operation TLV
* An Operation TLV followed by one or more Modifier TLVs
* Only Modifier TLVs

#### TLV Format {#tlvformat}

Operation and modifier TLVs both use the same encoding format.

The Acknowledgement bit in an Operation TLV of a request dictates if a response is to be sent. The Operation TLV may or may not be echoed back in the response according to the definition of the TLV. Each Operation TLV definition should stipulate whether an acknowledgement is REQUIRED. If the Operation TLV is not included in the response according to the TLV definition, the matching identifier in the standard DNS Header response is sufficient as an acknowledgement. If the TLV Acknowledgement bit is cleared in a request, a response MUST NOT be sent. The Acknowledgement bit is NEVER set in a response. Modifier TLVs MUST NEVER set the Acknowledgement bit in request or response.

It is by design that Operation TLVs SHOULD normally require a response and, therefore, set the TLV Acknowledgement bit in a request. However, for some Operation TLVs, this may be undesirable and the TLV Acknowledgement bit MAY be cleared in the request.  

                                                 1   1   1   1   1   1
         0   1   2   3   4   5   6   7   8   9   0   1   2   3   4   5
       +---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+
       | A |                        DSO-TYPE                           |
       +---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+
       |                        DSO DATA LENGTH                        |
       +---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+
       |                                                               |
       /                      TYPE-DEPENDENT DATA                      /
       /                                                               /
       +---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+

A:
: A 1-bit TLV Response flag indicating whether or not an Operation TLV requires
a response Acknowledgement. This bit is never set in responses to Operation TLVs and never set in Modifier TLVs in either direction.

DSO-TYPE:
: A 15-bit field in network order giving the type of the current DSO TLV per
the IANA DSO Type Codes Registry.

DSO DATA LENGTH:
: A 16-bit field in network order giving the size in octets of
the TYPE-DEPENDENT DATA.

TYPE-DEPENDENT DATA:
: Type-code specific format.

Where domain names appear within TYPE-DEPENDENT DATA, they MAY be compressed 
using standard DNS name compression. However, the compression MUST NOT point
outside of the TYPE-DEPENDENT DATA section and offsets MUST be from the start
of the TYPE-DEPENDENT DATA.

#### Operation TLVs

An "Operation TLV" specifies the operation to be performed.

A DSO message MUST contain at most one Operation TLV.

In all cases a DSO request message MUST contain exactly one 
Operation TLV, indicating the operation to be performed.

Depending on the operation, a DSO response message MAY contain no 
Operation TLV, because it is simply a response to a previous request message,
and the MESSAGE ID in the header is sufficient to identify the request in 
question. Or it may contain a single corresponding response Operation TLV, with 
the same DSO-TYPE as in the request message. The specification for each DSO
type determines whether a response for that operation type 
is required to carry the Operation TLV.

If a DSO response is received for an operation which requires
that the response carry an Operation TLV, and the required Operation TLV is not
the first DSO TLV in the response message, then this is a fatal 
error and the recipient of the defective response message MUST immediately
terminate the connection with a TCP RST (or equivalent for other protocols).

#### Modifier TLVs

A "Modifier TLV" specifies additional parameters
relating to the operation. Immediately following the Operation TLV, if present,
a DSO message MAY contain one or more Modifier TLVs.

#### Unrecognized TLVs

If a DSO request is received containing an unrecognized
Operation TLV, the receiver MUST send a response with matching
MESSAGE ID, and RCODE DSONOTIMP (tentatively 11). The response MUST NOT contain 
an Operation TLV.

If a DSO message (request or response) is received
containing one or more unrecognized Modifier TLVs, the unrecognized
Modifier TLVs MUST be silently ignored, and the remainder of the message
is interpreted and handled as if the unrecognized parts were not present.

### EDNS(0) and TSIG

Since the ARCOUNT field MUST be zero, a DSO message
MUST NOT contain an EDNS(0) option in the additional records section.
If functionality provided by current or future EDNS(0) options is desired
for DSO messages, an Operation TLV or
Modifier TLV needs to be defined to carry the necessary information.

For example, the EDNS(0) Padding Option {{!RFC7830}} used for security purposes
is not permitted in a DSO message,
so if message padding is desired for DSO messages
then the Encryption Padding TLV described in {{padding}} MUST be used.

Similarly, a DSO message MUST NOT contain a TSIG record.
A TSIG record in a conventional DNS message is added as the last record
in the additional records section, and carries a signature computed over
the preceding message content. Since DSO data appears
after the additional records section, it would not be included in the
signature calculation. 
If use of signatures with DSO messages becomes necessary in the 
future, an explicit Modifier TLV needs to be defined to 
perform this function.

Note however that, while DSO *messages* cannot include
EDNS(0) or TSIG records, a DSO *session* is typically used to 
carry a whole series of DNS messages of different kinds, including DSO
messages, and other DNS message types like Query {{!RFC1034}} 
{{!RFC1035}} and Update {{!RFC2136}}, and those messages can carry EDNS(0) and 
TSIG records.

This specification explicitly prohibits use of the
EDNS(0) TCP Keepalive Option {{!RFC7828}}
in *any* messages sent on a DSO Session (because it duplicates
the functionality provided by the DSO Keepalive operation),
but messages may contain other EDNS(0) options as appropriate.

## Message Handling

The initiator MUST set the value of the QR bit in the DNS header to
zero (0), and the responder MUST set it to one (1).
Every DSO request message (QR=0) with the Acknowledgement bit set in the
operation TLV MUST elicit a response (QR=1), which MUST have the same
MESSAGE ID in the DNS message header as in the corresponding request.
DSO request messages sent by the client with the Acknowledgement
bit set in the operation TLV elicit a response from the server, and
DSO request messages sent by the server with the Acknowledgement
bit set in the operation TLV elicit a response from the client.

A DSO request message (QR=0) with the Acknowledgement bit clear in the
operation TLV MUST NOT elicit a response.

The namespaces of 16-bit MESSAGE IDs are disjoint in each direction.
For example, it is *not* an error for both client and server to send a request
message with the same ID. In effect, the 16-bit MESSAGE ID combined with the 
identity of the initiator (client or server) serves as a 17-bit unique 
identifier for a particular operation on a DSO Session.

As described in {{header}} An initiator MUST NOT reuse a MESSAGE ID that is 
already in use for an 
outstanding request, unless specified otherwise by the relevant specification 
for the DSO in question. At the very least, this means 
that a MESSAGE ID MUST NOT be reused in a particular direction
on a particular DSO Session while the initiator is waiting for a response to a 
previous request using that MESSAGE ID on that DSO Session,
unless specified otherwise by the relevant 
specification for the DSO in question.
(For a long-lived state the MESSAGE ID for the operation
MUST NOT be reused whilst that state remains active.)

If a client or server receives a response (QR=1) where the MESSAGE ID does not
match any of its outstanding operations, this is a fatal error and it MUST 
immediately terminate the connection with a TCP RST (or equivalent for other 
protocols).

## DSO Response Generation

With most TCP implementations, for DSO requests that generate a response,
the TCP data acknowledgement (generated because data has been received by TCP),
the TCP window update (generated because TCP has delivered that data to the receiving software),
and the DSO response (generated by the receiving application-layer software itself)
are all combined into a single packet.
Combining these three elements into a single packet gives a
potentially significant improvement in network efficiency.

For DSO requests that do not generate a response,
the TCP implementation generally doesn't have any way to know
that no response will be forthcoming, so it waits fruitlessly
for the application-layer software to generate a response,
until the Delayed ACK timer fires {{?RFC1122}} (typically 200 milliseconds)
and only then does it send the TCP ack and window update.
In conjunction with Nagle's Algorithm at the sender,
this can delay the sender's transmission of its next
(non-full-sized) TCP segment, while the sender is waiting for
its previous (non-full-sized) TCP segment to be acknowledged.
Nagle's Algorithm exists to combine multiple small
application writes into more efficient large TCP segments,
to guard against wasteful use of the network by applications
that would otherwise transmit a stream of small TCP segments,
but in this case Nagle's Algorithm (created to improve network efficiency)
can interact badly with TCP's Delayed ACK feature
(also created to improve network efficiency) {{NagleDA}}.

Possible mitigations for this problem include:

* Disabling Nagle's Algorithm at the sender.
This is not great, because it results in less efficient use of the network.

* Disabling Delayed ACK at the receiver.
This is not great, because it results in less efficient use of the network.

* Using a networking API that lets the receiver signal to the TCP
implementation that the receiver has received and processed a client
request for which it will not be generating any immediate response.
This allows the TCP implementation to operate efficiently in both cases;
for requests that generate a response, the TCP ack, window update, and
DSO response are transmitted together in a single TCP segment,
and for requests that do not generate a response,
the application-layer software informs the TCP implementation
that it should go ahead and send the TCP ack and window update
immediately, without waiting for the Delayed ACK timer.
Unfortunately it is not known at this time which (if any) of the
widely-available networking APIs currently include this capability.

# DSO Session Lifecycle and Timers {#lifecycle}

## DSO Session Initiation

A session begins when a client makes a new connection to a server.

A DSO Session begin as described in {{establishment}}.

The client may perform as many DNS operations as it wishes using the
newly created DSO Session. Operations SHOULD be pipelined (i.e., the
client doesn't need wait for a response before sending the next message).
The server MUST act on messages in the order they are transmitted, but
responses to those messages MAY be sent out of order, if appropriate.

## DSO Session Timeouts {#sessiontimeouts}

Two timeout values are associated with a DSO Session: the inactivity timeout, and 
the keepalive interval. 

The first timeout value, the inactivity timeout, is the maximum time for which
a client may speculatively keep a DSO Session open in the expectation that
it may have future requests to send to that server.

The second timeout value, the keepalive interval, is the maximum permitted
interval between client messages to the server if the client wishes to keep
the DSO Session alive.

The two timeout values are independent. The inactivity timeout may be lower, the 
same, or higher than the keepalive interval, though in most cases the inactivity 
timeout is expected to be shorter than the keepalive interval.

Only when the client has a very long-lived low-traffic state does the 
keepalive interval come into play, to ensure that a sufficient residual
amount of traffic is generated to maintain NAT and firewall state
and to assure client and server that they still have connectivity to each other.

On a new DSO Session, if no explicit DSO Keepalive message exchange has taken 
place, the default value for both timeouts is 15 seconds.
For both timeouts, lower values of the timeout result in higher network traffic
and higher CPU load on the server.

## Inactive DSO Sessions {#inactivetimer}

At both servers and clients, the generation or reception of any complete
DNS message, including DNS requests, responses, updates, or DSO
messages, resets both timers for that DSO Session, with the exception
that a DSO Keepalive message resets only the keepalive timer, not the inactivity 
timeout timer.

In addition, for as long as the client has an outstanding operation in progress,
the inactivity timer remains cleared, and an inactivity 
timeout cannot occur.

For short-lived DNS operations like traditional queries and updates,
an operation is considered in progress for the time between request and 
response, typically a period of a few hundred milliseconds at most.
At the client, the inactivity timer is cleared upon transmission of a 
request and remains cleared until reception of the corresponding response.
At the server, the inactivity timer is cleared upon reception of a request
and remains cleared until transmission of the corresponding response.

For long-lived DNS Stateful operations, an operation is considered in progress
for as long as the state is active, until it is cancelled.
This means that a DSO Session can exist, with a state 
active, with no messages flowing in either direction, for far longer than the 
inactivity timeout, and this is not an error. This is why there are two separate 
timers: the inactivity timeout, and the keepalive interval.
Just because a DSO Session has no traffic for an extended period of time
does not automatically make that DSO Session "inactive",
if it has an active state that is awaiting events.

## The Inactivity Timeout

The purpose of the inactivity timeout is for the server to balance its trade off 
between the costs of setting up new DSO Sessions and the costs of maintaining inactive 
DSO Sessions. A server with abundant DSO Session capacity can offer a high inactivity timeout, 
to permit clients to keep a speculative DSO Session open for a long time, to save 
the cost of establishing a new DSO Session for future communications with that 
server. A server with scarce memory resources can offer a low inactivity timeout,
to cause clients to promptly close DSO Sessions whenever they have no outstanding
operations with that server, and then create a new DSO Session later when needed.

### Closing Inactive DSO Sessions

A client is NOT required to wait until the inactivity timeout expires
before closing a DSO Session.
A client MAY close a DSO Session at any time, at the client's discretion.
If a client determines that it has no current or reasonably anticipated
future need for an inactive DSO Session, then the client SHOULD close that connection.

If, at any time during the life of the DSO Session,
the inactivity timeout value (i.e., 15 seconds by default) elapses
without there being any operation active on the DSO Session,
the client MUST gracefully close the connection with a
TLS close_notify followed by a TCP FIN (or equivalent for other protocols).

If, at any time during the life of the DSO Session,
twice the inactivity timeout value (i.e., 30 seconds by default),
or five seconds, if twice the inactivity timeout value is less than five seconds,
elapses without there being any operation active on the DSO Session,
the server SHOULD consider the client delinquent,
and forcibly abort the DSO Session.
For DSO Sessions over TCP (or over TLS over TCP),
to avoid the burden of having a connection in TIME-WAIT state,
instead of closing the connection gracefully the server SHOULD abort
the connection with a TCP RST (or equivalent for other protocols).
(In the BSD Sockets API this is achieved by setting the
SO_LINGER option to zero before closing the socket.)

In this context, an operation being active on a DSO Session includes
a query waiting for a response, an update waiting for a response,
or active state,
but not a DSO Keepalive message exchange itself.
A DSO Keepalive message exchange resets only the keepalive
interval timer, not the inactivity timeout timer.

If the client wishes to keep an inactive DSO Session open for longer than
the default duration without having to send traffic every 15 seconds,
then it uses the DSO Keepalive message to request
longer timeout values, as described in {{keepalive}}.

### Values for the Inactivity Timeout

For the inactivity timeout value, lower values result in
more frequent DSO Session teardown and re-establishment.
Higher values result in larger memory burden to maintain state for
inactive DSO Sessions, but lower traffic and CPU load on the server.

A server may dictate (in a server-initiated Keepalive message,
or in a response to a client-initiated Keepalive request message)
any value it chooses for the inactivity timeout.
When a connection's inactivity timeout is reached the client MUST
begin closing the idle connection, but a client is NOT REQUIRED
to keep an idle connection open until the inactivity timeout is
reached --- a client SHOULD begin closing the connection sooner
if it has no reason to expect future operations with that server
before the inactivity timeout is reached.

A shorter inactivity timeout with a longer keepalive interval signals
to the client that it should not speculatively keep an inactive DSO
Session open for very long without reason, but when it does have an
active reason to keep a DSO Session open, it doesn't need to be sending
an aggressive level of keepalive traffic to maintain that session.

A longer inactivity timeout with a shorter keepalive interval
signals to the client that it may speculatively keep an inactive
DSO Session open for a long time, but to maintain that inactive
DSO Session it should be sending a lot of keepalive traffic.
This configuration is expected to be less common.

A server may dictate any value it chooses for the inactivity timeout
(either in a response to a client-initiated request, or in a server-initiated message)
including values under one second, or even zero.
An inactivity timeout of zero informs the client that it
should not speculatively maintain idle connections at all, and
as soon as the client has completed the operation or operations relating
to this server, the client should immediately begin closing this session.

A server will abort an idle client session after twice the
inactivity timeout value, or five seconds, whichever is greater.
In the case of a zero inactivity timeout value, this means that
if a client fails to close an idle client session then the server
will forcibly abort the idle session after five seconds.

## The Keepalive Interval {#keepalivetimer}

The purpose of the keepalive interval is to manage the generation of
sufficient messages to maintain state in middle boxes (such at NAT gateways
or firewalls) and for the client and server to periodically verify that they
still have connectivity to each other. This allows them to clean up state
when connectivity is lost, and attempt re-connection if appropriate.

### Keepalive Interval Expiry

If, at any time during the life of the DSO Session,
the keepalive interval value (i.e., 15 seconds by default) elapses
without any DNS messages being sent or received on a DSO Session,
the client MUST take action to keep the DSO Session alive,
by sending a DSO Keepalive message (see {{keepalive}}).
A DSO Keepalive message exchange resets only the keepalive timer, 
not the inactivity timer.

If a client disconnects from the network abruptly,
without cleanly closing its DSO Session,
leaving long-lived state uncanceled,
the server learns of this after failing to
receive the required keepalive traffic from that client.
If, at any time during the life of the DSO Session,
twice the keepalive interval value (i.e., 30 seconds by default) elapses
without any DNS messages being sent or received on a DSO Session,
the server SHOULD consider the client delinquent,
and forcibly abort the connection with a TCP RST (or equivalent for other 
protocols).

### Values for the Keepalive Interval

For the keepalive interval value, lower values result in higher volume keepalive 
traffic. Higher values of the keepalive interval reduce traffic and CPU load, 
but have minimal effect on the memory burden
at the server, because clients keep a DSO Session open for the same length of time
(determined by the inactivity timeout) regardless of the level of keepalive traffic 
required.

It may be appropriate for clients and servers to select different keepalive 
interval values depending on the nature of the network they are on.

A corporate DNS server that knows it is serving only clients on the internal 
network, with no intervening NAT gateways or firewalls, can impose a higher 
keepalive interval, because frequent keepalive traffic is not required.

A public DNS server that is serving primarily residential consumer clients, 
where it is likely there will be a NAT gateway on the path, may impose a lower 
keepalive interval, to generate more frequent keepalive traffic.

A smart client may be adaptive to its environment. A client using
a private IPv4 address {{!RFC1918}} to communicate with a DNS server
at an address outside that IPv4 private address block,
may conclude that there is likely to be a NAT gateway on the path,
and accordingly request a lower keepalive interval.

For environments where there is a NAT gateway or firewalls on the path, it is
RECOMMENDED that clients request, and servers grant, a keepalive interval of 15
minutes. In other environments it is RECOMMENDED that clients request, and 
servers grant, a keepalive interval of 60 minutes.

Note that the lower the keepalive interval value, the higher the load on client
and server. For example, an keepalive interval value of 100ms would result in a
continuous stream of at least ten messages per second, in both directions,
to keep the DSO Session alive. And, in this extreme example, a single packet loss and
retransmission over a long path could introduce a momentary pause in the stream of messages,
long enough to cause the server to overzealously abort the connection.

Because of this concern, the server MUST NOT send a Keepalive message
(either a response to a client-initiated request, or a server-initiated message)
with an keepalive interval value less than ten seconds.
If a client receives an Keepalive message specifying an keepalive interval value
less than ten seconds this is an error and the client MUST immediately
terminate the connection with a TCP RST (or equivalent for other protocols).

## Server-Initiated Termination on Error

After sending an error response to a client, the server MAY end the DSO Session,
or may allow the DSO Session to remain open. For error conditions
that only affect the single operation in question, the server SHOULD return an
error response to the client and leave the DSO Session open for further operations.
For error conditions that are likely to make all operations unsuccessful in the
immediate future, the server SHOULD return an error response to the client and 
then end the DSO Session by sending a Retry Delay request message, as described in 
{{delay}}.

## Client Behaviour in Receiving an Error {#error}

Upon receiving an error response from the server, a client SHOULD NOT
automatically close the DSO Session. An error relating to one particular operation
on a DSO Session does not necessarily imply that all other operations on that
DSO Session have also failed, or that future operations will fail. The client
should assume that the server will make its own decision about whether or not to
end the DSO Session, based on the server's determination of whether the error
condition pertains to this particular operation, or would also apply to any
subsequent operations. If the server does not end the DSO Session by
sending the client a Retry Delay message (see {{delay}}) then the client
SHOULD continue to use that DSO Session for subsequent operations.

## Server-Initiated Termination on Overload

A server MUST NOT close a DSO Session with a client,
except in certain exceptional circumstances, as outlined below.
In normal operation, closing a DSO Session is the client's responsibility.
The client makes the determination of when to close a DSO
Session based on an evaluation of both its own needs,
and the inactivity timeout value dictated by the server.

Some exceptional situations where a server may terminate a DSO Session include:

* The server is undergoing reconfiguration or maintenance procedures
that require clients to be disconnected.

* The server application software or underlying operating system
is shutting down or restarting.

* The server application software terminates unexpectedly
(perhaps due to a bug that makes it crash).

* The client fails to meets its obligation to generate keepalive
traffic or close an inactive session by the prescribed times
(twice the time interval dictated by the server, or five seconds,
whichever is greater, as described in {{sessiontimeouts}}).

* The client sends a grossly invalid or malformed request that is
indicative of a seriously defective client implementation (see {{error}}).

* The server is over capacity and needs to shed some load (see {{retry}}).

When a server has to close a DSO Session with a client
(because of exceptional circumstances such as those outlined above)
the server SHOULD, whenever possible, send a Retry Delay Operation TLV
(see below) informing the client of the reason for the DSO Session
being closed, and allow the client five seconds to receive it
before the server resorts to forcibly aborting the connection.

## Retry Delay Operation TLV {#retry}

There may be rare cases where a server is overloaded and wishes to shed load.
If a server is low on resources it MAY simply terminate a client connection with 
a TCP RST (or equivalent for other protocols).
However, the likely behavior of the client may be simply to to treat this as a
network failure and connect
immediately, putting more burden on the server.

Therefore to avoid this reconnection implosion, a server SHOULD instead choose 
to shed client load by sending a Retry Delay request message, with an RCODE of 
SERVFAIL, to inform the client of the overload situation. After sending a Retry 
Delay request message, the server MUST NOT send any further messages on that 
DSO Session.

After sending the Retry Delay request the server SHOULD allow the
client five seconds to close the connection, and if the client has not
closed the connection after five seconds then the server SHOULD abort
the connection with a TCP RST (or equivalent for other protocols).

Upon receipt of a Retry Delay request from the server, the client MUST
make note of the reconnect delay for this server, and then immediately
close the connection.
This is to place the burden of TCP's TIME-WAIT state on the client.

A Retry Delay request message MUST NOT be initiated by a client.
If a server receives a Retry Delay request message this is an error
and the server MUST immediately terminate the connection with a TCP RST
(or equivalent for other protocols).

### Outstanding Operations

At the moment a server chooses to initiate a Retry Delay request message
there may be DNS requests already in flight from client to server on this 
DSO Session, which will arrive at the server after its Retry Delay request message 
has been sent.
The server MUST silently ignore such incoming requests, and MUST NOT generate
any response messages for them. When the Retry Delay request message from the
server arrives at the client, the client will determine that any DNS requests
it previously sent on this DSO Session, that have not yet received a response, now 
will certainly not be receiving any response. Such requests should be considered
failed, and should be retried at a later time, as appropriate.

In the case where some, but not all, of the existing operations on a DSO Session 
have become invalid (perhaps because the server has been reconfigured and is no 
longer authoritative for some of the names),
but the server is terminating all DSO Sessions en masse with a REFUSED (5) RCODE,
the RECONNECT DELAY MAY be zero, indicating that the clients SHOULD immediately
attempt to re-establish operations.
It is likely that some of the attempts will be successful and some will not.

In the case where a server is terminating a large number of DSO Sessions at once
(e.g., if the system is restarting) and the server doesn't want to be inundated 
with a flood of simultaneous retries, it SHOULD send different RECONNECT delay 
values to each client.
These adjustments MAY be selected randomly, pseudorandomly, or deterministically
(e.g., incrementing the time value by one tenth of a second for each successive
client, yielding a post-restart reconnection rate of ten clients per second).

### Client Reconnection

After a DSO Session is closed by the server, the client SHOULD try to reconnect,
to that server, or to another suitable server, if more than one is available.
If reconnecting to the same server, the client MUST respect the indicated delay
before attempting to reconnect.

If a particular server does not want a client to reconnect (it is being
de-commissioned), it SHOULD set the retry delay to the maximum value (which is
approximately 497 days). If the server will only be out of service for a maintenance
period, it should use a value closer to the expected maintenance window and
not default to a very large delay value or clients may not attempt to reconnect
after it resumes service.

# Connection Sharing {#sharing}

As previously specified for DNS over TCP {{!RFC7766}},
to mitigate the risk of unintentional server overload, DNS clients
MUST take care to minimize the number of concurrent TCP connections
made to any individual server.  It is RECOMMENDED that for any given
client/server interaction there SHOULD be no more than one connection
for regular queries, one for zone transfers, and one for each
protocol that is being used on top of TCP (for example, if the
resolver was using TLS).
However, it is noted that certain primary/secondary configurations
with many busy zones might need to use more than one TCP
connection for zone transfers for operational reasons
(for example, to support concurrent transfers of multiple zones).

A single server may support multiple services, including DNS Updates 
{{!RFC2136}}, DNS Push Notifications {{?I-D.ietf-dnssd-push}},
and other services, for one or more DNS zones.
When a client discovers that the target server for several different operations
is the same target hostname and port, the client SHOULD use a single
shared DSO Session for all those operations.
A client SHOULD NOT open multiple connections to the same target host and port
just because the names being operated on are different or
happen to fall within different zones.
This is to reduce unnecessary connection load on the DNS server.

However, server implementers and operators should be aware that connection
sharing may not be possible in all cases.
A single client device may be home to multiple independent client software
instances that don't coordinate with each other.
Similarly, multiple independent client devices behind the same NAT gateway
will also typically appear to the DNS server as different source ports on
the same client IP address.
Because of these constraints, a DNS server MUST be prepared to accept
multiple connections from different source ports on the same client IP address.

# Base TLVs for DNS Stateful Operations

## Retry Delay TLV {#delay}

The Retry Delay TLV (DSO-TYPE=0) can be used as an Operation TLV or as
a Modifier TLV. 

The TYPE-DEPENDENT DATA for the the Retry Delay TLV is as follows:

                            1 1 1 1 1 1 1 1 1 1 2 2 2 2 2 2 2 2 2 2 3 3
        0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
       +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
       |                     RETRY DELAY (32 bits)                     |
       +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

RETRY DELAY:
: a time value, specified as a 32-bit word in network order in units of 
milliseconds, within which the client MUST NOT retry this operation, or retry 
connecting to this server.

The RECOMMENDED value is 10 seconds.

### Use as an Operation TLV

When sent in a DSO request message, from server to client, the 
Retry Delay TLV (0) is considered an Operation TLV. It is used by a server 
to request that a client close the DSO Session and underlying connection, and 
not to reconnect for the indicated time interval.

In this case it applies to the DSO Session as a whole, and the client MUST begin closing the 
DSO Session, as described in {{retry}}. The RCODE in the message header 
MUST indicate the reason for the termination:

* NOERROR indicates a routine shutdown. 
* SERVFAIL indicates that the server is overloaded due to resource exhaustion. 
* REFUSED indicates that the server has been reconfigured and is no longer able 
to perform one or more of the functions 
currently being performed on this DSO Session (for example, a DNS Push Notification 
server could be reconfigured such that is is no longer accepting DNS Push 
Notification requests for one or more of the currently subscribed names).

This document specifies only these three RCODE values for Retry Delay request.
Servers sending Retry Delay requests SHOULD use one of these three values.
However, future circumstances may create situations where other RCODE values
are appropriate in Retry Delay requests, so clients MUST be prepared
to accept Retry Delay requests with any RCODE value.

An acknowledgement is not desired for a Retry Delay Operation TLV and the TLV
Acknowledgement bit MUST be cleared in the request.

### Use as a Modifier TLV

In the case of a client request that returns a nonzero RCODE value,
the server MAY append a Retry Delay TLV (0) to the response,
indicating the time interval during which the client
SHOULD NOT attempt this operation again.

When appended to a DSO response message for some client request,
the Retry Delay TLV (0) is considered a Modifier TLV.

The indicated time interval during which the client SHOULD NOT retry
applies only to the failed operation, not to the DSO Session as a whole.

## Keepalive Operation TLV {#keepalive}

The Keepalive Operation TLV (DSO-TYPE=1) performs two functions: to reset the
keepalive timer for the DSO Session and to establish the values for the Session Timeouts. 

The Keepalive Operation TLV resets only the keepalive timer, not the inactivity timer.
The reason for this is that periodic Keepalive Operation TLVs are sent for the
sole purpose of keeping a DSO Session alive, because that DSO Session has current
or recent non-maintenance activity that warrants keeping the DSO Session alive.
If sending keepalive traffic itself were to reset the inactivity timer, then 
that would create a circular livelock where keepalive traffic would be sent 
indefinitely to keep a DSO Session alive, where the only activity on that DSO 
Session would be the keepalive traffic keeping the DSO Session alive so that further 
keepalive traffic can be sent.

Sending keepalive traffic is considered a maintenance activity
that is performed in service of other client activities.
Sending keepalive traffic itself is not considered a client activity.
For a DSO Session to be considered active, it must be carrying something more than just keepalive traffic.
This is why merely sending a Keepalive Operation TLV does not reset the inactivity timer.

When sent by a client, the Keepalive Operation TLV MUST have the Acknowledgement bit set.
It resets a DSO Session's keepalive timer, and at the same time requests what the
Session Timeout values should be from this point forward in the DSO Session.
If a server receives a Keepalive Operation TLV without the Acknowledgement bit set
then this is a fatal error and the server MUST immediately terminate
the connection with a TCP RST (or equivalent for other protocols).

Once a DSO Session is in progress (see {{details}}) the Keepalive TLV also MAY be initiated by a server.
When sent by a server, the Keepalive Operation TLV MUST NOT have the Acknowledgement bit set.
It resets a DSO Session's keepalive timer, and unilaterally informs the client of
the new Session Timeout values to use from this point forward in this DSO Session.
No client response to this unilateral declaration is required.
If a client receives a Keepalive Operation TLV with the Acknowledgement bit set
then this is a fatal error and the client MUST immediately terminate
the connection with a TCP RST (or equivalent for other protocols).

It is not required that the Keepalive TLV be used in every DSO Session.
While many DNS Stateful operations
will be used in conjunction with a long-lived session state,
not all DNS Stateful operations require long-lived session state,
and in some cases the default 15-second value for both the inactivity timeout
and keepalive interval may be perfectly appropriate.
However, note that for clients that implement only the TLVs defined in
this document it is the only way for a client to initiate a DSO Session.

The TYPE-DEPENDENT DATA for the the Keepalive TLV is as follows:

                            1 1 1 1 1 1 1 1 1 1 2 2 2 2 2 2 2 2 2 2 3 3
        0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
       +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
       |                 INACTIVITY TIMEOUT (32 bits)                  |
       +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
       |                 KEEPALIVE INTERVAL (32 bits)                  |
       +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

INACTIVITY TIMEOUT:
: the inactivity timeout for the current DSO Session, specified as a
32-bit word in network (big endian) order in units of milliseconds.
This is the timeout at which the client MUST begin closing an inactive DSO Session.
The inactivity timeout can be any value of the server's choosing.
If the client does not gracefully close an inactive DSO
Session, then after twice this interval, or five seconds,
whichever is greater, the server will forcibly terminate the
connection with a TCP RST (or equivalent for other protocols).

KEEPALIVE INTERVAL:
: the keepalive interval for the current DSO Session, specified as a
32-bit word in network (big endian) order in units of milliseconds.
This is the interval at which a client MUST generate keepalive
traffic to maintain connection state.
The keepalive interval MUST NOT be less than ten seconds.
If the client does not generate the mandated keepalive traffic,
then after twice this interval the server will forcibly terminate
the connection with a TCP RST (or equivalent for other protocols).
Since the minimum allowed keepalive interval is ten seconds, the
minimum time at which a server will forcibly disconnect a client for
failing to generate the mandated keepalive traffic is twenty seconds.

In a client-initiated DSO Keepalive message,
the Session Timeouts contain the client's requested values.
In a server response to a client-initiated message, the Session Timeouts contain 
the server's chosen values, which the client MUST respect.
This is modeled after the DHCP protocol, where the client
requests a certain lease lifetime using DHCP option 51 {{!RFC2132}},
but the server is the ultimate authority
for deciding what lease lifetime is actually granted.

In a server-initiated DSO Keepalive message, the Session Timeouts
unilaterally inform the client of the new values from this point
forward in this DSO Session. The client MUST NOT generate a
response to the server-initiated DSO Keepalive message.

When a client is sending its second and subsequent Keepalive DSO 
requests to the server, the client SHOULD continue to request its preferred 
values each time. This allows flexibility, so that if conditions change during 
the lifetime of a DSO Session, the server can adapt its responses to better fit 
the client's needs.

### Client handling of received Session Timeout values

When a client receives a response to its client-initiated DSO Keepalive message,
or receives a server-initiated DSO Keepalive message, the client has then 
received Session Timeout values dictated by the server. The two timeout values 
contained in the DSO Keepalive TLV from the server may each be higher, lower, or 
the same as the respective Session Timeout values the client previously had for 
this DSO Session.

In the case of the keepalive timer, the handling of the received value is 
straightforward. The act of receiving the message containing the DSO Keepalive 
TLV itself resets the keepalive timer and updates the keepalive interval for the 
DSO Session. The new keepalive interval indicates the 
maximum time that may elapse before another message must be sent
or received on this DSO Session, if the DSO Session is to remain alive.

In the case of the inactivity timeout, the handling of the received value 
is a little more subtle, though the meaning of the inactivity 
timeout is unchanged --- it still indicates the maximum permissible time allowed 
without activity on a DSO Session.
The act of receiving the message containing the DSO Keepalive TLV does not
itself reset the inactivity timer. The time elapsed since the last useful
activity on this DSO Session is unaffected by exchange of DSO Keepalive messages.
The new inactivity timeout value in the DSO Keepalive TLV in the received message
does update the timeout associated with the running inactivity timer;
that becomes the new maximum permissible time without activity on a DSO Session.

* If the current inactivity timer value is not greater than the
new inactivity timeout, then the DSO Session may remain open for now.
When the inactivity timer value exceeds the new inactivity timeout,
the client MUST then begin closing the DSO Session, as described above.

* If the current inactivity timer value is already greater
than the new inactivity timeout, then this DSO Session has
already been inactive for longer than the server permits,
and the client MUST immediately begin closing this DSO Session.

* If the current inactivity timer value is already more than twice the
new inactivity timeout, then the client is immediately considered delinquent
(this DSO Session is immediately eligible to be forcibly terminated by the server)
and the client MUST immediately begin closing this DSO Session.
However if a server abruptly reduces the inactivity timeout in this
way, then, to give the client time to close the connection gracefully
before the server resorts to terminating it forcibly, the server
SHOULD give the client an additional grace period of one quarter
of the new inactivity timeout, or five seconds, whichever is greater.

### Relation to EDNS(0) TCP Keepalive Option

The inactivity timeout value in the Keepalive TLV (DSO-TYPE=1) has similar
intent to the EDNS(0) TCP Keepalive Option {{!RFC7828}}.
A client/server pair that supports DSO MUST NOT use the
EDNS(0) TCP KeepAlive option within any message after a DSO 
Session has been established.
Once a DSO Session has been established, if either
client or server receives a DNS message over the DSO Session that contains an
EDNS(0) TCP Keepalive option, this is an error and the receiver of the
EDNS(0) TCP Keepalive option MUST immediately
terminate the connection with a TCP RST (or equivalent for other protocols).

## Encryption Padding TLV {#padding}

The Encryption Padding TLV (DSO-TYPE=2) can only be used as a Modifier TLV.
It is only applicable when the DSO Transport layer uses encryption
such as TLS.

The TYPE-DEPENDENT DATA for the the Padding TLV is optional and is a
variable length field containing non-specified values. A DATA LENGTH
of 0 essentially provides for 4 octets of padding (the minimum amount).

                                                 1   1   1   1   1   1
         0   1   2   3   4   5   6   7   8   9   0   1   2   3   4   5
       +---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+
       /                                                               /
       /                   VARIABLE NUMBER OF OCTETS                   /
       /                                                               /
       +---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+

As specified for the EDNS(0) Padding Option {{!RFC7830}}
the PADDING octets SHOULD be set to 0x00.  Other values MAY be used,
for example, in cases where there is a concern that the padded
message could be subject to compression before encryption.
PADDING octets of any value MUST be accepted in the messages received.
   
The Encryption Padding TLV may be included in either a DSO request, response, or both.
As specified for the EDNS(0) Padding Option {{!RFC7830}}
if a request is received with a Encryption Padding TLV,
then the response MUST also include an Encryption Padding TLV.

The length of padding is intentionally not specified in this document and
is a function of current best practices with respect to the type and length
of data in the preceding TLVs {{?I-D.ietf-dprive-padding-policy}}.

# IANA Considerations

## DSO OPCODE Registration

IANA are directed to assign a value (tentatively 6)
in the DNS OPCODEs Registry for the DSO OPCODE.

## DSO RCODE Registration

IANA are directed to assign a value (tentatively 11)
in the DNS RCODE Registry for the DSONOTIMP error code.

## DSO Type Codes Registry

IANA are directed to create the 15-bit DSO Type Codes
Registry, with initial values as follows:

| Type | Name | Status | Reference |
|--:|------|--------|-----------|
| 0x0000 | RetryDelay | Standard | RFC-TBD |
| 0x0001 | KeepAlive | Standard | RFC-TBD |
| 0x0002 | Encryption Padding | Standard | RFC-TBD |
| 0x0003 - 0x003F | Unassigned, reserved for DSO session-management TLVs | | |
| 0x0040 - 0x77FF | Unassigned | | |
| 0x7800 - 0x7BFF | Reserved for local / experimental use | | |
| 0x7C00 - 0x7FFF | Reserved for future expansion | | |

Registration of additional DSO Type Codes requires publication
of an appropriate IETF "Standards Action" or "IESG Approval" document {{!RFC5226}}.

# Security Considerations

If this mechanism is to be used with DNS over TLS, then these messages
are subject to the same constraints as any other DNS over TLS messages
and MUST NOT be sent in the clear before the TLS session is established.

The data field of the "Encryption Padding" TLV could be used as a covert channel.

# Acknowledgements

Thanks to
Tim Chown,
Ralph Droms,
Jan Komissar,
Manju Shankar Rao,
and Ted Lemon
for their helpful contributions to this document.

--- back
