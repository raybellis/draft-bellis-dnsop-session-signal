---
title: DNS Session Signaling
docname: draft-bellis-dnsop-session-signal-00
date: 2016-07

ipr: trust200902
area: Internet
wg: DNSOP Working Group
kw: Internet-Draft
cat: std

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
    phone: +1 843 473 7394
    email: pusateri@bangj.com


--- abstract

The Extension Mechanisms for DNS (EDNS(0)) {{!RFC6891}} is explicitly
defined to only have "per-message" semantics.  This document defines a
new Session Signaling OpCode used to carry persistent "per-session"
type-length-values (TLVs), and defines an initial set of TLVs used to
manage session timeouts and termination.

--- middle

# Introduction

The Extension Mechanisms for DNS (EDNS(0)) {{!RFC6891}} is explicitly
defined to only have "per-message" semantics.  This document defines a
new Session Signaling OpCode used to carry persistent "per-session"
type-length-values (TLVs), and defines an initial set of TLVs used to
manage session timeouts and termination.

A further issue with EDNS(0) is that there is no standard mechanism for
a client to be able to tell whether a server has processed or otherwise
acted upon the individual options contained with an OPT RR.  The Session
Signaling OpCode therefore requires an explicit response to each request
message.

It should be noted that the message format (see {{format}}) does not
conform to the standard DNS packet format.

# Terminology

The terms "initiator" and "responder" correspond respectively to the
initial sender and subsequent receiver of a Session Signaling TLV,
regardless of which was the "client" and "server" in the usual DNS
sense.  The term "sender" may apply to either an initiator or responder.

The term "session" in the context of this document means the exchange of
DNS messages over a single connection using an end-to-end transport
protocol where:

- connections are long-lived
- either end of the connection may initiate requests
- message delivery order is guaranteed
- it is guaranteed that the same two endpoints are in communication for
the entire lifetime of the session.

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT",
"SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this
document are to be interpreted as described in {{!RFC2119}}.

# Protocol Details

Session Signaling messages MUST only be carried in protocols and in
environments where a session may be established according to the
definition above.  Standard DNS over TCP, and DNS over TLS {{?RFC7858}}
are appropriate protocols.  DNS over plain UDP is not appropriate since
it fails on both the bi-directional initiation requirement and the
message order delivery requirement.

Session Signaling messages relate only to the specific session in which
they are being carried.  Where a middle box (e.g. a DNS proxy,
forwarder, or session multiplexer) is in the path the message MUST NOT
be blindly forwarded in either direction by that middle box.  This does
not preclude the use of these messages in the presence of a NAT box that
rewrites Layer 3 or Layer 4 headers but otherwise maintains the effect
of a single session.

A server MUST NOT initiate Session Signaling messages until a
client-initiated Session Signaling message is observed first.  This
requirement is to ensure that the client does not observe unsolicited
inbound messages until it has indicated its ability to handle them.

Session Signaling support is therefore said to be confirmed after the
first session signaling TLV has been sent by a client and successfully
acknowledged by the server.

Use of Session Signaling by a client should be taken as an implicit
request for a long-lived session.

## Message Format {#format}

A message containing a Session Signaling OpCode does not conform to the
usual DNS message format.  The 4 octet header format from {{!RFC1035}}
is however preserved, since that includes the message ID and OpCode and
RCODE fields, and the QR bit that differentiates requests from responses.

Each message MUST contain only a single TLV.

                                                 1   1   1   1   1   1
         0   1   2   3   4   5   6   7   8   9   0   1   2   3   4   5
       +---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+
       |                          MESSAGE ID                           |
       +---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+
       |QR |    OpCode     |            Z              |     RCODE     |
       +---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+
       |                                                               |
       /                           TLV-DATA                            /
       /                                                               /
       +---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+

The MESSAGE ID, QR, OpCode and RCODE fields have their usual meaning as
defined in {{!RFC1035}}.

The Z bits are currently unused, and SHOULD be set to zero (0) in
requests and responses unless re-defined in a later specification.

## Message Handling

Both clients and servers may unilaterally send Session Signaling
messages at any point in the lifetime of a session and are therefore
considered to be the initiator with respect to that message.  The
initiator MUST set the value of the QR bit in the DNS header to zero
(0), and the responder MUST set it to one (1).

Every Session Signaling request message MUST elicit a response (which
MUST have the same ID in the DNS message header as in the request).

In order to preserve the correct sequence of state, Session Signaling
requests MUST NOT be processed out of order.

<< RB: should the presence of a SS message create a "sequencing point",
such that all pending responses must be answered? >>

The RCODE value in a response uses a subset of the standard
(non-extended) RCODE values from the IANA DNS RCODEs registry,
interpreted as follows:

| Code | Mnemonic | Description |
|-----:|----------|-------------|
| 0 | NOERROR | TLV processed successfully |
| 1 | FORMERR | TLV format error |
| 4 | NOTIMP | Session Signaling not supported |
| 5 | REFUSED | TLV declined for policy reasons |

## TLV Format

                                                 1   1   1   1   1   1
         0   1   2   3   4   5   6   7   8   9   0   1   2   3   4   5
       +---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+
       |                         SESSION-TYPE                          |
       +---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+
       |                        SESSION-LENGTH                         |
       +---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+
       |                                                               |
       /                         SESSION-DATA                          /
       /                                                               /
       +---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+

SESSION-TYPE:
: A 16 bit field in network order giving the type of the current Session
Signaling TLV per the IANA DNS Session Signaling Type Codes Registry.

SESSION-LENGTH:
: A 16 bit field in network order giving the size in octets of
SESSION-DATA.

SESSION-DATA:
: Type-code specific.

# Mandatory TLVs

## Session Management Support TLVs

### "Not Implemented"

Since the "NOTIMP" RCODE is required to indicate lack of support for the
Session Signaling OpCode itself, the "Not Implemented" TLV (0) MUST
be returned in response to a TLV that is not implemented by the
responder.

This TLV has no SESSION-DATA.

## Session Management TLVs

### Start Session

The Start Session TLV (1) SHOULD be used by a client to indicate support
for Session Signaling.  It MUST NOT be initiated by a server.

It is not required that this TLV be used in every session - any valid
client-initiated TLV will suffice to initiate Session Signaling support.
The intention of this TLV is to provide a suitable "No-Op" TLV to permit
Session Signaling support to be negotiated without carrying any other
information.

This TLV has no SESSION-DATA.

<< RB: this could perhaps also be used as a real "no-op" message to
provide application-level keep-alive pings >>

### Terminate Session

The Terminate Session TLV (2) MAY be sent by a server to request that
the client terminate the session.  It MUST NOT be initiated by a client.

The client SHOULD terminate the session as soon as possible, but MAY
wait for any inflight queries to be answered.  It MUST NOT initiate any
new requests over the existing session.

The SESSION-DATA is as follows:

                                                 1   1   1   1   1   1
         0   1   2   3   4   5   6   7   8   9   0   1   2   3   4   5
       +---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+
       |                        RECONNECT DELAY                        |
       +---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+

RECONNECT DELAY:
: a time value, specified as a 16 bit word in network order in units of
100 milliseconds, within which the client MUST NOT establish a new
session to the current server.

The RECOMMENDED value is 10 seconds.  << RB: text required here about
default values for load balancers, etc >>

### Idle Timeout

The Idle Timeout TLV (3) has similar intent to the EDNS TCP Keepalive
Option {{!RFC7828}}.  It is used by a server to tell the client how long
it may leave the current session idle for.  a client.  The definition of
an idle session is as specified in {{!RFC7766}}.

Messages generate by the client have no SESSION-DATA (whether in
requests or responses).  A client-initiated Idle Timeout TLV allows the
client to request the current timeout value, whereas a server-initiated
request allows the server to unilaterally update the current timeout
value.

Messages generated by the server contain SESSION-DATA as follows:

                                                 1   1   1   1   1   1
         0   1   2   3   4   5   6   7   8   9   0   1   2   3   4   5
       +---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+
       |                         IDLE TIMEOUT                          |
       +---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+

IDLE TIMEOUT:
: the idle timeout for the current session, specified as a 16
bit word in network order in units of 100 milliseconds.

The client SHOULD terminate the current session if it remains idle for
longer than the specified timeout (and MAY of course terminate the
session earlier).  The server MAY unilaterally terminate the connection
at any time, but SHOULD allow the client to keep the connection open if
further messages are received before the idle timeout expires.

A client / server pair that supports Session Signaling MUST NOT use the
EDNS TCP KeepAlive option within any message once bi-directional Session
Signaling support has been confirmed.

# IANA Considerations

## DNS Session Signaling Opcode Registration

IANA are directed to assign the value TBD for the Session Signaling
OpCode in the DNS OpCodes Registry.

## DNS Session Signaling Type Codes Registry

IANA are directed to create the DNS Session Signaling Type Codes
Registry, with initial values as follows:

| Type | Name | Status | Reference |
|--:|------|--------|-----------|
| 0 | Not implemented | | RFC-TBD1 |
| 1 | Start Session | Standard | RFC-TBD1 |
| 2 | Terminate Session | Standard | RFC-TBD1 |
| 3 | Idle Timeout | Standard | RFC-TBD1 |
| 4 - 63 | Unassigned, reserved for session management TLVs | | |
| 64 - 63487 | Unassigned | | |
| 63488 - 64511 | Reserved for local / experimental use | | |
| 64512 - 65535 | Reserved for future expansion | | |

Registration of additional Session Signaling Type Codes requires Expert
Review. << RB: definition of process required? >>

# Security Considerations

If this mechanism is to be used over TLS, the TLS session SHOULD be
established before any Session Signaling messages are used.

# Acknowledgements

TBW

--- back
