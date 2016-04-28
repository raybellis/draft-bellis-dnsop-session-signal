---
title: DNS Session Signaling
docname: draft-bellis-dnsop-session-signal-00
date: 2016-04

ipr: trust200902
area: Internet
wg: DNSOP Working Group
kw: Internet-Draft
cat: std

coding: us-ascii
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
  -
    ins: A. Mankin
    name: Allison Mankin
    org: Unaffiliated
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
new OpCode used to carry persistent "per-session" TLVs, and defines an
initial set of TLVs used to handle feature negotiation and to manage
session timeouts and termination.

--- middle

# Introduction

...

# Terminology

The terms "initiator" and "responder" correspond respectively to the
initial sender and subsequent receiver of a Session TLV, regardless of
which was the "client" and "server" in the usual DNS sense.  The term
"sender" may apply to either an initiator or responder.

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT",
"SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this
document are to be interpreted as described in {{!RFC2119}}.

# Protocol Details

A packet containing the Session OpCode MUST NOT be sent over a
connectionless protocol such as UDP {{?RFC0768}}.

Session TLVs relate only to the specific Layer 4 protocol connection
over which they are being carried.  In the presence of a middle box
(e.g. a DNS proxy or forwarder) they MUST NOT be blindly forwarded in
either direction by that middle box.

## Packet Format

A Session Signaling packet does not conform to the usual DNS packet
format.  The 12 octet header format from {{!RFC1035}} is preserved, but
the four count fields (QDOUNT, ANCOUNT, NSCOUNT and ARCOUNT) MUST all be
set to zero.

A list of TLVs are used in place of the usual sections, and MUST appear
immediately after the 12 octet header.  The total size of the TLVs is
calculated from the value of the standard two octet framing word minus
the 12 octets of the DNS header.

## Packet Handling

Both clients and servers may unilaterally send Session TLVs at any point
in the lifetime of a Layer 4 connection and are therefore considered to
be the initiator with respect to that TLV.

Not all Session TLVs require a response.  A single DNS message MAY
combine both responses to requests previously received, and new
requests.

(editor's note - I was originally against both of the points in the
previous paragraph, but once you've got per-TLV status codes and a field
indicating "requests" there doesn't seem to be any reason not to.)

## TLV Format

       +---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+
    0: |  SESS-STATUS  |         SESSION-TYPE                          |
       +---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+
    2: |                        SESSION-LENGTH                         |
       +---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+
    4: |                                                               |
       /                         SESSION-DATA                          /
       /                                                               /
       +---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+

SESS-STATUS:
: A 4 bit field used to differentiate between requests and
responses (and the success or otherwise of the latter), per the DNS
Session Status Codes Registry.  It MUST contain "REQUEST" (0) within
request packets and a non-zero code in response packets.

SESSION-TYPE:
: A 12 bit field in network order giving the type of the
current Session TLV per the IANA DNS Session Type Codes Registry.

SESSION-LENGTH:
: A 16 bit field in network order giving the size in octets of
SESSION-DATA.

SESSION-DATA:
: Type-code specific.  The SESSION-DATA field MUST be NUL padded to an
even number of octets such that each Session TLV is aligned on a two
octet boundary relative to the start of the first Session TLV.  Padding
octets MUST NOT be included in the calculation of SESSION-LENGTH but
MUST be included in the calculation of the overall message length.

# Mandatory TLVs

## Feature Negotiation

### Session Support

The Session Support TLV (1) is used to allow a client and server
to exchange information about which Session Type Codes they support.

The SESSION-DATA contains a list of the Session Type Codes supported by
the sender.

       +---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+
       |                           TYPE CODEs                          |
       /                              ...                              /
       /                                                               /
       +---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+

TYPE CODEs:
: A list of 16 bit words in network order comprising the complete list
of Session Type Codes supported by the sender.  Since a Session Type
Code is in reality only a 12 bit value, the four most significant bits
of each word MUST be zero.  The number of TYPE CODEs can be calculated
from the total length of the TLV.

An initiator MAY send its own list of supported Session Type Codes in a
Session Support TLV, and if sent they MUST be complete.  Otherwise the
SESSION-DATA MUST be empty.  In either case the responder MUST response
with its complete list of supported Type Codes.

### EDNS Support

The EDNS Support TLV (2) is used to allow a client and server to
exchange information about which EDNS Version, Flags and Option Codes
they support.

The SESSION-DATA contains the supported EDNS version number and EDNS
flags followed by a list of of the EDNS Option Codes supported by the
sender.

       +---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+
       |               Z               |            VERSION            |
       +---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+
       |                             FLAGS                             |
       +---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+
       |                           OPT CODEs                           |
       /                              ...                              /
       /                                                               /
       +---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+

Z:
: Set to zero by initiators and ignored by responders, unless modified
in a subsequent specification.

VERSION:
: The maximum EDNS version number supported by the sender.

FLAGS:
: A bit mask containing a one (1) for each corresponding EDNS Flag bit
(e.g. "DO") supported by the sender, and zero (0) otherwise.

OPT CODEs:
: A list of 16 bit words in network order containing the complete list
of EDNS Option Codes supported by the sender.  The number of OPT CODEs
can be calculated from the total length of the TLV minus the four octets
for the preceding fields.

A client MAY send its own EDNS Support data in an EDNS Support TLV, and
if sent it MUST be complete.  Otherwise the SESSION-DATA MUST be empty.
In either case the responder MUST respond with its complete EDNS Support
data.

TODO: A server SHOULD NOT sent an unsoliticited populated EDNS Support TLV.

## Layer 4 Connection Management TLVs

### Terminate

The Terminate TLV (64) MAY be sent be a server to request that the client
terminate the connection.  It MUST NOT be sent by a client.

The client SHOULD terminate the connection as soon as possible, but MAY
wait for any inflight queries to be answered.  It SHOULD NOT initiate
any new queries over the existing connection.

This TLV has no SESSION-DATA.

### Idle Timeout

The Idle Timeout TLV (65) has similar semantics to the EDNS TCP
Keepalive Option {{!RFC7828}}.  It is used by a server to tell the
client how long it may leave the current connection idle for.

The SESSION-DATA is as follows:

       +---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+
       |                         IDLE TIMEOUT                          |
       +---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+

IDLE TIMEOUT:
: the idle timeout for the current Layer 4 connection, specified as a 16
bit word in network order in units of 100 milliseconds.

It is NOT an error for this TLV and the similar EDNS option to appear
within the same connection.  The client SHOULD pay attention to the most
recently received value, regardless of which method was used to send it.

The client SHOULD terminate the current connection if it remains idle
for longer than the specified timeout (and MAY of course terminate the
connection earlier if desired).

(author's note - this assumes that the EDNS OPT RR is added at the final
stage of packet processing, and therefore not affected by out-of-order
processing)

# IANA Considerations

## DNS Session OpCode Registration

IANA are directed to assign the value TBD for the Session OpCode in the
DNS OpCodes Registry.

## DNS Session Status Codes Registry

IANA are directed to create the DNS Session Status Codes Registry, with
initial values as follows:

| Code | Mnemonic | Description | Reference |
|-----:|----------|-------------|-----------|
| 0 | REQUEST | Request packet| RFC-TBD1 |
| 4 | NOTIMP | TLV not implemented | RFC-TBD1 |
| 5 | REFUSED | TLV declined for policy reasons | RFC-TBD1 |
| 15 | SUCCESS | TLV processed sucessfully | RFC-TBD1 |

Registration of additional Session Status Codes requires Standards Action.

## DNS Session Type Codes Registry

IANA are directed to create the DNS Session Type Codes Registry, with
initial values as follows:

| Type | Name | Status | Reference |
|--:|------|--------|-----------|
| 0 | Reserved | | RFC-TBD1 |
| 1 | Session Support | Standard | RFC-TBD1 |
| 2 | EDNS Support | Standard | RFC-TBD1 |
| 3 - 63 | Unassigned, reserved for feature negotiation TLVs | | |
| 64 | Terminate | Standard | RFC-TBD1 |
| 65 | Idle Timeout | Standard | RFC-TBD1 |
| 66 - 127 | Unassigned, reserved for layer 4 connection TLVs | | |
| 127 - 3965 | Unassigned | | |
| 3968 - 4031 | Reserved for local / experimental use | | |
| 4032 - 4095 | Reserved for future expansion | | |

Registration of additional Session Type Codes requires Expert Review.

# Security Considerations

TBD

# Acknowledgements

TBW

--- back
