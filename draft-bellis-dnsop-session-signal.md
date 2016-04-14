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
new OPCODE used to carry "per-session" options, and defines an initial
set of options used to manage session timeouts and termination.

--- middle

# Introduction

TBW

# Terminology

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT",
"SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this
document are to be interpreted as described in {{!RFC2119}}.

# IANA Considerations

TBC

# Security Considerations

TBD

# Acknowledgements

TBW

--- back
