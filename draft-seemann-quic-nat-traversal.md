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
    email: martenseemann@gmail.com

 -
    fullname: Eric Kinnear
    organization: Apple Inc.
    street: One Apple Park Way
    city: Cupertino, California 95014
    country: United States of America
    email: ekinnear@apple.com

normative:
   ALTERNATIVE-ADDRESS: I-D.munizaga-quic-alternative-server-address

informative:
   MULTIPATH: I-D.ietf-quic-multipath
   CONNECT-UDP-LISTEN: I-D.ietf-masque-connect-udp-listen


--- abstract

QUIC is well-suited to various NAT traversal techniques. As it operates over UDP
and because the QUIC header was designed to be demultiplexed from other
protocols, STUN can be used on the same UDP socket, enabling ICE to be used with
QUIC. Furthermore, QUIC’s path validation mechanism can be used to test the
viability of an address candidate pair while at the same time creating the NAT
bindings required for a direct connection, after which QUIC connection migration
can be used to migrate the connection to a direct path.

--- middle

# Introduction

This document describes two ways to use QUIC ({{!RFC9000}}) to traverse NATs:

1. Using ICE ({{!RFC8445}}) with an external signaling channel to select a pair
   of UDP addresses. Once candidate nomination is completed, a new QUIC
   connection between the two endpoints can be established.
2. Using a (proxied) QUIC connection as the signaling channel. QUIC's path
   validation logic is used to test connectivity of possible paths.

The first option documents how NAT traversal can be achieved using unmodified
QUIC and ICE stacks. The only requirement is the ability to send and receive
non-QUIC (STUN ({{!RFC5389}})) packets on the UDP socket that a QUIC server is
listening on. However, it necessitates running a separate signaling channel for
the communication between the two ICE agents.

The second option doesn't use ICE at all, although it makes use of some of the
concepts, in particular the address matching logic described in {{!RFC8445}}. It
is assumed that the nodes are connected via a proxied QUIC connection, for
example using {{CONNECT-UDP-LISTEN}}. Using the QUIC extension defined in this
document, the nodes coordinate QUIC path validation attempts that create the
necessary NAT bindings to achieve traversal of the NAT. This mechanism makes
extensive use of the path validation mechanism described in {{!RFC9000}}. In
addition, the QUIC server needs the capability to initiate path validation,
whereas {{!RFC9000}} assumes that it is initiated by the client. Starting with a
proxied QUIC connection allows the nodes to start exchanging application data
right away and switch to the direct connection once it has been established and
deemed suitable for the application's needs.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

Sending base:

: The local sending context used for an address candidate, such as a UDP socket
  on a particular network interface or a tunnel.

# Background: NAT Traversal with ICE

When an external signaling channel is used, the QUIC connection is established
after the two ICE agents have agreed on a candidate pair. This mode doesn't
require any modification to existing QUIC stacks. In particular, it does not
necessitate the negotiation of the extension defined in this document.

For address discovery to work, QUIC and ICE need to use the same UDP socket.
Since this requires demultiplexing of QUIC and STUN packets, the QUIC bit cannot
be greased as described in {{!RFC9287}}.

Once ICE has completed, the client immediately initiates a normal QUIC handshake
using the server's address from the nominated address pair. The ICE connectivity
checks should have created the necessary NAT bindings for the client's first
flight to reach the server and for the server's first flight to reach the
client.

# NAT Traversal Extension Overview

QUIC's path validation mechanism can be used to establish the required NAT
mappings that allow for a direct connection. Once the NAT mappings are
established, QUIC's connection migration can be used to migrate the connection
to a direct path. During the path validation phase, multiple different paths
might be established in parallel. When using QUIC Multipath {{MULTIPATH}}, these
paths may be used at the same time; however, the mechanism described in this
document does not require the use of QUIC multipath.

Although ICE is not directly used, the logic run on the client makes use of
ICE's candidate pairing logic (see especially {{Section 6.1.2.2 of RFC8445}}).
Implementations are free to implement different algorithms as they see fit.

This mode needs to be negotiated during the handshake; see
{{negotiate-extension}}.

# Candidate Discovery and Exchange

## Gathering Address Candidates

The gathering of address candidates is out of scope for this document. Endpoints
MAY use the logic described in {{Sections 5.1.1 and 5.2 of RFC8445}}, or they
MAY use address candidates provided by the application.

## Advertising Server Address Candidates

The server advertises its address candidates using ALTERNATIVE_ADDRESS frames,
as defined in {{ALTERNATIVE-ADDRESS}}. Each frame advertises the complete set of
alternative addresses and replaces the previously advertised set. The server
SHOULD NOT wait until address candidate discovery has finished; instead, it
SHOULD update the advertised set as soon as new candidates become available.
This speeds up NAT traversal and is similar to Trickle ICE ({{?RFC8838}}).

The server stores the sending bases associated with each advertised address.
Candidates with different sending bases MUST be advertised as separate
entries, even if their advertised addresses are identical.

The server removes a stale address candidate by omitting it from a subsequent
address-set update, e.g., when the network interface becomes unavailable.

Since address matching is run on the client side, only the server advertises
address candidates. The client communicates selected address pairs to the server
using PUNCH_REQUEST frames.

## Forming Candidate Pairs

The client matches the address candidates sent by the server with its own
address candidates, forming candidate pairs. {{Section 5.1 of RFC8445}}
describes an algorithm for pairing address candidates. Since the pairing
algorithm is only run on the client side, the endpoints do not need to agree on
the algorithm used, and the client is free to use a different algorithm.

# Coordinated Path Probing {#coordinated-probing}

The client requests an independent path validation attempt for an address pair
using PUNCH_REQUEST. It SHOULD start validation immediately after sending the
request; the server SHOULD start immediately upon accepting it. Validation
follows {{Section 8.2 of RFC9000}}, with additional rate limits
({{amplification-attack}}). Each endpoint MUST set its own timeout following
{{Section 8.2.4 of RFC9000}}.

The server MUST report rejection, failure, or success using PUNCH_DONE
({{punch-done-frame}}), stopping its probes for that attempt before sending it.
All PUNCH_DONE frames for the same Attempt ID MUST carry the same Status.
PUNCH_DONE does not change the client's validation result or either endpoint's
obligation to answer PATH_CHALLENGE frames ({{Section 8.2.2 of RFC9000}}).

The client MUST NOT exceed the advertised concurrency limit. Each Attempt ID
counts once, from the first request transmission until local probing ends and
the matching PUNCH_DONE is received. The server counts accepted attempts until
it first sends PUNCH_DONE. A new request that would exceed the limit MUST be
treated as a connection error of type PROTOCOL_VIOLATION.

The client SHOULD request attempts as candidate pairs become available, but MAY
delay requests to prioritize pairs when the concurrency limit is small.

## Interaction with active_connection_id_limit

The active_connection_id_limit limits the number of connection IDs that are
active at any given time. Both endpoints need to use a previously unused
connection ID when validating a new path in order to avoid linkability.
Therefore, the active_connection_id_limit effectively places a limit on the
number of concurrent path validations.

Endpoints SHOULD set an active_connection_id_limit that is high enough to allow
for the desired number of concurrent path validation attempts.

## Amplification Attack Mitigation {#amplification-attack}

TODO describe exactly how to mitigate amplification attacks

# Extension Negotiation {#negotiate-extension}

Endpoints advertise their support of the extension by sending the nat_traversal
(0x3d7e9f0bca12fea6) transport parameter ({{Section 7.4 of RFC9000}}).

The client MUST send this transport parameter with an empty value. A server
implementation that understands this transport parameter MUST treat the receipt
of a non-empty value as a connection error of type TRANSPORT_PARAMETER_ERROR.

The client MUST also send the alternative_address transport parameter defined in
{{ALTERNATIVE-ADDRESS}}. A server that understands nat_traversal MUST treat
receipt of nat_traversal without alternative_address as a connection error of
type TRANSPORT_PARAMETER_ERROR.

For the server, the value of this transport parameter is a variable-length
integer, the concurrency limit defined in {{coordinated-probing}}. Any value
larger than 0 is valid. A client implementation that understands this transport
parameter MUST treat the receipt of a value that is not a variable-length
integer, or the receipt of the value 0, as a connection error of type
TRANSPORT_PARAMETER_ERROR.

To enable the use of this extension in 0-RTT packets, the client MUST remember
the value of this transport parameter. If 0-RTT data is accepted by the server,
the server MUST not disable this extension on the resumed connection.

# PUNCH_REQUEST Frame

~~~
PUNCH_REQUEST Frame {
    Type (i) = 0x3d7e92,
    Attempt ID (i),
    Client Address Type (8),
    Client IP Address (32..128),
    Client Port (16),
    Server Address Type (8),
    Server IP Address (32..128),
    Server Port (16),
}
~~~

The PUNCH_REQUEST frame contains the following fields:

Attempt ID:

: The client MUST number new attempts consecutively from 0 within each
   connection.

Client Address Type and Server Address Type:

: The address family of the corresponding IP Address field. The values 0x01 and
   0x02 indicate IPv4 and IPv6, respectively, as in {{ALTERNATIVE-ADDRESS}}.
   Receipt of any other value MUST be treated as a connection error of type
   FRAME_ENCODING_ERROR.

Client IP Address:

: The client's address candidate. This field is 32 bits long for IPv4 and 128
   bits long for IPv6, as indicated by Client Address Type.

Client Port:

: The port number of the client's address candidate.

Server IP Address:

: The server's address candidate, selected from the addresses advertised using
   ALTERNATIVE_ADDRESS frames. This field is 32 bits long for IPv4 and 128 bits
   long for IPv6, as indicated by Server Address Type.

Server Port:

: The port number of the server's address candidate.

For a given Attempt ID, address fields MUST NOT change. Servers MUST ignore
duplicate requests, including for completed attempts, and MUST treat detected
conflicts as a connection error of type PROTOCOL_VIOLATION.

PUNCH_REQUEST frames are ack-eliciting. If lost, they MUST be retransmitted
unless the request has been acknowledged or the corresponding PUNCH_DONE has
been received.

This frame is only sent from the client to the server. Clients MUST treat
receipt of a PUNCH_REQUEST frame as a connection error of type
PROTOCOL_VIOLATION.

# PUNCH_DONE Frame {#punch-done-frame}

~~~
PUNCH_DONE Frame {
    Type (i) = 0x3d7e95,
    Attempt ID (i),
    Status (i),
}
~~~

Attempt ID identifies the PUNCH_REQUEST. Status reports the server's result:

* SUCCEEDED (0x00): The server's path validation succeeded.
* FAILED (0x01): The server's path validation timed out.
* REJECTED (0x02): The server did not start the attempt.

Clients MUST ignore duplicates. An unknown Status MUST be treated as a
connection error of type FRAME_ENCODING_ERROR; an unissued Attempt ID or
detected conflicting statuses MUST be treated as a connection error of type
PROTOCOL_VIOLATION.

PUNCH_DONE is ack-eliciting, sent on a validated path, and MUST be retransmitted
on loss until acknowledged.

This frame is only sent from the server to the client. Servers MUST treat
receipt of a PUNCH_DONE frame as a connection error of type PROTOCOL_VIOLATION.

# Security Considerations

This document expands QUIC's path validation logic to QUIC servers, allowing a
QUIC client to request the sending of path validation packets on unverified
paths. A malicious client can direct traffic to a target IP. This attack is
similar to the IP address spoofing attack that address validation during
connection establishment (see {{Section 8.1 of RFC9000}}) is designed to
prevent. In practice, however, IP address spoofing is often additionally
mitigated by ingress and egress filtering at the IP layer. This mitigation is
not possible when using this extension. The server therefore needs to carefully
limit the amount of data it sends on unverified paths.


# IANA Considerations

TODO: fill out registration request for the transport parameter and frame types

--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
