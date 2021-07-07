---
title: "Generic HTTP Binding for WebTransport using QUIC"
abbrev: "WebTransport over QUIC over HTTP"
docname: draft-thomson-webtrans-hwtq-latest
category: info

ipr: trust200902
area: ART
workgroup: WEBTRANS
keyword: Internet-Draft

stand_alone: yes
pi: [toc, sortrefs, symrefs]

author:
 -
    name: Martin Thomson
    organization: Mozilla
    email: mt@lowentropy.net

normative:

informative:



--- abstract

The WebTransport API provides an interface to HTTP resources that provides
transport-layer capabilities.  This document describes how a subset of the QUIC
protocol can be used to provide these transport capabilities.

--- middle

# Introduction

This document is input to discussion of the design of
{{!WTH2=I-D.ietf-webtrans-http2}} and -- to a lesser extent --
{{!WTH3=I-D.ietf-webtrans-http3}}.  The author has no intent of pursuing
publication of this document, it exists only to provide a more complete
exploration of an alternative design.

This document describes a means of providing a WebTransport-capable mapping to
any HTTP version, with a specific goal of providing a TCP-based design that
fulfills all of the basic WebTransport requirements.

The proposed design transplants the design of QUIC streams {{!QUIC=RFC9000}}
almost directly, saving specification effort.  This might also save
implementation and validation effort through reuse.


# Conventions and Definitions

{::boilerplate bcp14-tagged}


# Overview

The WebTransport API depends on three basic functions: bidirectional and
unidirectional streams, plus datagrams.  When using a QUIC transport, using
native QUIC support for these capabilities ({{!QUIC=RFC9000}},
{{!QD=I-D.ietf-quic-datagram}}) provides the best performance.  However, the
need to provide a TCP-based protocol means finding a protocol that works with --
at a minimum -- HTTP/2.

This document proposes that an extended CONNECT {{!EXT-CONNECT=RFC8441}} be used
to establish a new connection using a new protocol identifier (with an
identifier to be decided).  Using CONNECT results in a bidirectional
communications medium with stream characteristics between client and server.

Once established, this document proposes using a subset of QUIC frames to carry
the control messages, stream data, and datagrams.  The following frames would be permitted:

* PADDING
* RESET_STREAM
* STOP_SENDING
* STREAM (with the 0x02 bit set only)
* MAX_DATA
* MAX_STREAM_DATA
* MAX_STREAMS
* DATA_BLOCKED
* STREAM_DATA_BLOCKED
* STREAMS_BLOCKED
* DATAGRAM (from {{!QD=I-D.ietf-quic-datagram}})

This set of frames is sufficient to carry data for streams of the necessary
types.  It also provides a complete set of resource management functions that
operate within the scope of the bidirectional WebTransport communication.

This would use a subset of the stream states in {{!QUIC}}.  This ensures that
the API is able to present a very similar interface to streams as that provided
in QUIC.  Implementations might be simplified slightly as some state transitions
are not possible in a context where ordered delivery can be guaranteed.

Endpoints MUST NOT send stream data that is not contiguous or out of order.  The
underlying transport provides reliable, ordered delivery.  Implementations will
still need to buffer stream data, but the implementation of that buffering does
not need to handle gaps in incoming flow.


## Open Design Questions

It is not clear whether "connection"-level flow control (the QUIC MAX_DATA
frame) is necessary in the context of this integration.  TCP or HTTP/2 flow
control mechanisms already exist to control the flow of information on the
request and response, which makes the control redundant in some circumstances.
The inclusion is justified by the potential for DATAGRAM frames and other
control frames to be exchanged independent of this limit.

A connection close signal at the level of the WebTransport is possible.  The
QUIC application CONNECTION_CLOSE (0x1d) might be used for this purpose.


# Comparison to an HTTP/2-Only Design

The main claim made here is that this design is comparatively simpler than the
design in {{!WTH2}}.  However, there are other consequences that are worth
consideration.  This section attempts to document these more fully.


## Session Control and Resource Management

The QUIC binding in {{!WTH3}} is complicated by the need to manage the total
number of WebTransport sessions on the one connection.  Furthermore, the use of
shared resources at the level of the connection (streams especially) means that
it is necessary to carefully manage the commitment of resources to a

{{!WTH2}} has similar constraints on operation.  This proposal avoids using a
shared resource for its functions, avoiding that problem. The cost is that
sessions are exposed to additional head-of-line blocking performance costs.  As
the goal of the protocol is to support TCP-based HTTP versions, the marginal
impact of head-of-line blocking is expected to be minimal.


## Stream States

{{!WTH2}} describes a design that integrates with the HTTP/2 stream state
machine.  This is challenging as it results in the details of the HTTP/2 stream
state machine being visible in the API.  In particular, the closing of send and
receive parts of HTTP/2 streams are coupled, where they are independent in QUIC.

As a protocol extension, this difference could be addressed with adjustments to
the state machine, but this results in the implementation of an entirely new
state management logic, which increases implementation complexity.

The other way to resolve this discrepancy is to require the same sort of
close-state coupling when QUIC is used so that behavior is consistent across
different protocol versions.


## HTTP/1.1

Though not a hard requirement of the design, the ability to support HTTP/1.1 is
an advantage of this design.

It is also possible to use this design with HTTP/3, albeit with worse
performance characteristics than the more complete design of {{!WTH3}}.  This is
offered merely as an observation.


## Additional Framing

This proposal adds an additional layer framing, which increases overheads in
HTTP/2.  This is less efficient in terms of overheads than native use of HTTP/2
layer constructs.  This is partially mitigated by the relatively good efficiency
of QUIC framing and the potential to send larger frames with the HTTP/2 stream.
The QUIC-derived frames this uses do not need to be limited in size in the same
way as those at the HTTP/2 layer.


# Security Considerations

Relatively few of the security considerations of QUIC apply, though a small few
do, such as {{Section 21.6 of RFC9000}}.


# IANA Considerations

This document has no IANA actions. actions.


--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge someone.
