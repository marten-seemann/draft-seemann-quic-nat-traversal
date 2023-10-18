---
title: "Using QUIC to traverse NATs"
abbrev: "QUIC NAT Traversal"
category: std

docname: draft-seemann-quic-nat-traversal-latest
submissiontype: IETF
consensus: true
v: 3
area: "Transport"
workgroup: "QUIC"
keyword:
 - QUIC
 - ICE
 - NAT traversal
 - hole punching

author:
 -
    fullname: Marten Seemann
    organization: Protocol Labs
    email: martenseemann@gmail.com

normative:

informative:


--- abstract

QUIC is well-suited to various NAT traversal techniques. As it operates over
UDP, and because the QUIC header was designed to be demultipexed from other
protocols, STUN can be used on the same UDP socket. This allows for using ICE
with QUIC. Furthermore, QUIC’s path validation mechanism can be used to test the
viability of an address candidate pair while at the same time creating the NAT
bindings required for a direction connection, after which QUIC connection
migration can be used to migrate the connection to a direct path.

--- middle

# Introduction

This document describes two ways to use QUIC ({{!RFC9000}}) to traverse NATs:

1. Using ICE ({{!RFC8445}}) with an external signaling channel to select a pair
   of UDP addresses. Once candidate nomination is completed, a new QUIC
   connection between the two endpoints can be established.
2. Using a (proxied) QUIC connection as the signaling channel.  QUIC's path
   validation logic is used to test connectivity of possible paths.

The first mode doesn't require any changes to existing QUIC and ICE stacks. The
only requirement is the ability to send and receive non-QUIC (STUN
({{!RFC5389}})) packets on the UDP socket that a QUIC server is listening on.
However, it necessitates running a separate signaling channel for the
communication between the two ICE agents.

The second mode doesn't use ICE at all, although it makes use of some of the
concepts, in particular the address matching logic described in {{!RFC8445}}. It
is assumed that the nodes are connected via a proxied QUIC connection. Using the
logic described in this documents, the nodes coordinate QUIC path validation
attempts that create the necessary NAT bindings to achieve traversal of the NAT.
This mechanism makes extensive use of the path validation mechanism described in
{{!RFC9000}}. In addition to that, the QUIC server needs the capability to
initiate path validation, which, as per {{!RFC9000}}, is solely executed by the
client. Starting with a proxied QUIC connection allows the nodes to begin
exchanging application data, and switch to the direct connection once it has
been established and deemed suitable for the application's needs.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

# Mode 1: NAT Traveersal Using an External Signaling Channel

When an external signaling channel is used, the QUIC connection is established
after the two ICE agents have agreed on a candidate pair. This mode doesn't
require any modification to existing QUIC stacks. In particular, it does not
necessitate the negotiation of the extension defined in this document.

For address discovery to work, QUIC and ICE need to use the same UDP socket
Since this requires demultiplexing of QUIC and STUN packets, the QUIC bit cannot
be greased as described in {{!RFC9287}}.

Once ICE has completed, the client immediately initiates a normal QUIC handshake
using the server's address from the nominated address pair. The ICE connectivity
checks should have created the necessary NAT bindings for the client's first
flight to reach the server, and for the server's first flight to reach the
client.

# Mode 2: QUIC NAT Traversal

ICE is not directly used in this mode. However, the logic run on the client
makes use of ICE's candidate pairing logic (see especially {{Section 6.1.2.2 of
RFC8445}}). Implementations are free to implement different algorithms as they
see fit.

This mode needs be negotiated during the handshake, see {{negotiate-extension}}.

## Gathering Address Candidates

The gathering of address candidates is out of scope for this document. Endpoints
MAY use the logic described in {{Sections 5.1.1 and 5.2 of RFC8445}}, or use
address candidates provided by the application.

## Address Transmission

The server transmits its address candidates to the client using ADD_ADDRESS
frames. It SHOULD NOT wait until address candidate discovery has finished,
instead, it SHOULD send address candidates as soon as they become available.
This allows speeding up the NAT traversal, and is similar to the Trickle ICE
({{!RFC8838}}) logic.

Addresses transmitted to the client can be removed using the REMOVE_ADDRESS
frame if the address candidate becomes stale, e.g. because the network interface
becomes unavailable.

Address candidates are only transmitted from the server to the client, since the
entire address matching logic is run on the client side.

## Address Matching

The client matches the address candidates sent by the server with its own
address candidates, forming candidate pairs. {{Section 5.1 of RFC8445}}
describes an algorithm for pairing address candidates. Since the pairing
algorithm is only run on the client side, the peers do not need to agree on the
algorithm used, and the client is free to use other algorithms.

## Probing paths

The client sends candidate pairs to the server using PUNCH_ME_NOW frames. The
client SHOULD start path validation (see {{Section 8.2 of RFC9000}}) for the
respective path immediately after sending the PUNCH_ME_NOW frame.

On the server side, path validation SHOULD be started immediately when receiving
a PUNCH_ME_NOW frame. This document introduces path validation on the server
side, since {{!RFC9000}} assumes that any QUIC server is able to receive packets
on a path without creating a NAT binding first. Path validation on the server
side works as described in {{Section 8.2.1 of RFC9000}}, with additional
rate-limiting to prevent amplification attacks.

Path probing happens in rounds, allowing the peers to limit the bandwidth
consumed by path validation packets. For every round, the client MUST NOT send
more PUNCH_ME_NOW frames than allowed by the server's transport parameter. A new
round is started when a PUNCH_ME_NOW with a higher Round value is received. This
immediately cancels all path probes in progress.

To speed up NAT traversal, the client SHOULD send address pairs as soon as they
become available. However, for small concurrency limits, it MAY delay sending of
address pairs in order prioritize them first and only initiate path validation
for the highest-priority candidate pairs.

### Amplification Attack Mitigation

TODO describe exactly how to migitate amplification attacks

## Negotiating Extension Use {#negotiate-extension}

Endpoints advertise their support of the extension by sending the nat_traversal
(0x3d7e9f0bca12fea6) transport parameter ({{Section 7.4 of RFC9000}}).

The client MUST send this transport parameter with an empty value. A server
implementation that understands this transport parameter MUST treat the receipt
of a non-empty value as a connection error of type TRANSPORT_PARAMETER_ERROR.

For the server, the value of this transport parameter is a variable-length
integer, the concurrency limit. The concurrency limit limits the amount of
concurrent NAT traversal attempts, and can be used to limit the bandwith
required to execute the path validation. Any value larger than 0 is valid. A
client implementation that understands this transport parameter MUST treat the
receipt of a value that is not a variable-length integer, or the receipt of the
value 0, as a connection error of type TRANSPORT_PARAMETER_ERROR.

In order to the use of this extension in 0-RTT packets, the client MUST remember
the value of this transport parameter. If 0-RTT data is accepted by the server,
the server MUST not disable this extension on the resumed connection.

## Frames

### ADD_ADDRESS Frame

~~~
ADD_ADDRESS Frame {
    Type (i) = 0x3d7e90..0x3d7e91,
    Sequence Number (i),
    [ IPv4 (32) ],
    [ IPv6 (128) ],
    Port (16),
}
~~~

The ADD_ADDRESS frame contains the following fields:

Sequence Number:

: A variable-length integer encoding the sequence number of this address
   advertisement.

IPv4:

: The IPv4 address. Only present if the least significant bit of the frame type
   is 0.

IPv6:

: The IPv6 address. Only present if the least significant bit of the frame
type is 1.

Port: The port number.

ADD_ADDRESS frames are ack-eliciting. When lost, they SHOULD be retransmitted,
unless the address is not active anymore.

This frame is only sent from the server to the client. Servers MUST treat
receipt of an ADD_ADDRESS frame as a connection error of type
PROTOCOL_VIOLATION.

### PUNCH_ME_NOW Frame

~~~
PUNCH_ME_NOW Frame {
    Type (i) = 0x3d7e92..0x3d7e93,
    Round (i),
    Paired With Sequence Number (i),
    [ IPv4 (32) ],
    [ IPv6 (128) ],
    Port (16),
}
~~~

The ADD_ADDRESS frame contains the following fields:

Round:

: The sequence number of the NAT Traversal attempts.

Paired With Sequence Number:

: A variable-length integer encoding the sequence number of the address that was
   paired with this address.

IPv4:

: The IPv4 address. Only present if the least significant bit of the frame type
   is 0.

IPv6:

: The IPv6 address. Only present if the least significant bit of the frame type
   is 1.

Port:

: The port number.

PUNCH_ME_NOW frames are ack-eliciting.

This frame is only sent from the client to the server. Clients MUST treat
receipt of a PUNCH_ME_NOW frame as a connection error of type
PROTOCOL_VIOLATION.

### REMOVE_ADDRESS Frame

~~~
REMOVE_ADDRESS Frame {
    Type (i) = 0x3d7e94,
    Sequence Number (i),
}
~~~

The REMOVE_ADDRESS frame contains the following fields:

Sequence Number:

: A variable-length integer encoding the sequence number of the address
   advertisement to be removed.

REMOVE_ADDRESS frames are ack-eliciting. When lost, they SHOULD be
retransmitted.

This frame is only sent from the server to the client. Servers MUST treat
receipt of an REMOVE_ADDRESS frame as a connection error of type
PROTOCOL_VIOLATION.


# Security Considerations

This document expands QUIC's path validation logic to the server side, allowing
the client to request sending of path validation packets on unverified paths. A
malicious client can direct traffic to a target IP. This attack is similar to
the IP address spoofing attack that address validation during connection
establishment (see {{Section 8.1 of RFC9000}}) is designed to prevent. In
practice however, IP address spoofing is often additionally mitigated by both
the ingress and egress network at the IP layer, which is not possible when using
this extension. The server therefore needs to carefully limit the amount of data
it sends on unverified paths.


# IANA Considerations

TODO: fill out registration request for the transport parameter and frame types

--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
