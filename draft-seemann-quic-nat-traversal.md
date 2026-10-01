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

The server MAY withdraw an address by omitting all its entries from a subsequent
address-set update. For addresses with multiple sending bases, see the entry
retention rules in {{multiple-sending-bases}}.

Since address matching is run on the client side, only the server advertises
address candidates. The client communicates selected address pairs to the server
using PUNCH_REQUEST frames.

### Multiple Sending Bases {#multiple-sending-bases}

For each address pair, the client SHOULD request one attempt per advertised
occurrence of the server address.

Accepted attempts for the same address pair MUST use different available
advertised bases on this connection. If none remains, the server MUST send
a PUNCH_DONE frame with Status REJECTED.

Each advertisement of an address MUST retain one entry per base ever advertised
for it on this connection. Without an increase in the entry count, the client
cannot distinguish a replaced base from an unchanged one and might not schedule
another attempt.

## Forming Candidate Pairs

The client matches the address candidates sent by the server with its own
address candidates, forming candidate pairs. {{Section 5.1 of RFC8445}}
describes an algorithm for pairing address candidates. Since the pairing
algorithm is only run on the client side, the endpoints do not need to agree on
the algorithm used, and the client is free to use a different algorithm.

# Coordinated Path Probing {#coordinated-probing}

The server authorizes attempts using PUNCH_GRANT ({{punch-grant-frame}}). Each
grant permits one independent path validation attempt for an address pair. The
client uses a grant by sending PUNCH_REQUEST with its Grant ID and the selected
address pair on an established path. It SHOULD start validation immediately
after sending this request; the server SHOULD start immediately upon accepting
it. Validation follows {{Section 8.2 of RFC9000}}, with additional rate limits
({{amplification-attack}}). Each endpoint MUST set its own timeout following
{{Section 8.2.4 of RFC9000}}.

Each endpoint MUST send probe packets containing PATH_CHALLENGE frames for an
attempt from a single sending base to the peer address specified in the
PUNCH_REQUEST.

Each endpoint MUST report success or timeout using PUNCH_DONE
({{punch-done-frame}}); the server MUST also report rejection. Before sending it,
the endpoint MUST permanently stop sending probe packets containing
PATH_CHALLENGE frames for that attempt.

PUNCH_DONE does not stop the peer's probing or change its validation result.

The client SHOULD request attempts as candidate pairs and unused grants become
available, but MAY delay requests to prioritize pairs.

## Grant Limits {#grant-limits}

The client's nat_traversal transport parameter ({{negotiate-extension}}) sets
the initial cumulative grant limit. MAX_PUNCH_GRANTS
({{max-punch-grants-frame}}) can increase it. The server MUST NOT issue a grant
whose Grant ID is greater than or equal to the largest limit received. The
client MUST treat receipt of a grant at or above its largest advertised limit
as a connection error of type PROTOCOL_VIOLATION.

The server decides when to issue grants within the limit; the client decides
when to increase it.

## Punching Connection IDs {#punching-cids}

This document introduces punching connection IDs, a new type of connection ID
for coordinated path probing. They have the same wire format in packet headers
as ordinary connection IDs and MUST satisfy the unlinkability requirements in
{{Section 5.1 of RFC9000}}.

Each endpoint issues two punching connection IDs per grant, using PUNCH_GRANT or
PUNCH_REQUEST, respectively. The IDs MUST be distinct from all others issued by
that endpoint on the connection.

In most circumstances, only one of the two connection IDs is needed, but there
are certain network setups where both need to be used ({{different-base}}).

The NEW_CONNECTION_ID and RETIRE_CONNECTION_ID mechanisms of {{RFC9000}} and the
active_connection_id_limit transport parameter do not apply to punching
connection IDs. When QUIC multipath {{MULTIPATH}} is negotiated, punching
connection IDs are not tied to any path.

Packets containing PATH_CHALLENGE or PATH_RESPONSE frames for an attempt MUST
use one of the peer's punching connection IDs for that grant. Non-probing
packets MUST NOT use these IDs. If a peer receives a non-probing packet that
used a punching connection ID, it MUST close the connection with an error of
type PROTOCOL_VIOLATION.

An endpoint retires a grant's punching connection IDs after both sending and
receiving PUNCH_DONE ({{punch-done-frame}}) for that attempt.

## Probes Received on a Different Base {#different-base}

Under certain network configurations, a probe packet can arrive at a different
base than the one the receiving endpoint selected for its attempt. This can
happen when overlapping address spaces give candidates with different bases
the same address ({{Appendix B.2 of RFC8445}}).

Both endpoints follow the path validation rules of {{RFC9000}} in addition to
the coordinated probing rules above. An endpoint sends a packet containing the
PATH_RESPONSE from the receiving base to the probe's source address. A matching
PATH_RESPONSE validates the path on which the corresponding probe was sent,
regardless of where the response arrives.

## Amplification Attack Mitigation {#amplification-attack}

TODO describe exactly how to mitigate amplification attacks

# Extension Negotiation {#negotiate-extension}

Endpoints advertise their support of the extension by sending the nat_traversal
(0x3d7e9f0bca12fea6) transport parameter ({{Section 7.4 of RFC9000}}).

The client's value is a single variable-length integer specifying the initial
cumulative grant limit ({{grant-limits}}). The server MUST send an empty value.
An endpoint that understands this transport parameter MUST treat receipt of an
invalid value as a connection error of type TRANSPORT_PARAMETER_ERROR.

The client MUST also send the alternative_address transport parameter defined in
{{ALTERNATIVE-ADDRESS}}. A server that understands nat_traversal MUST treat
receipt of nat_traversal without alternative_address as a connection error of
type TRANSPORT_PARAMETER_ERROR.

This transport parameter MUST NOT be remembered for use in 0-RTT. The frames
defined in this document MUST only be sent in 1-RTT packets.

# Frames

This extension defines the PUNCH_GRANT, MAX_PUNCH_GRANTS, PUNCH_REQUEST, and
PUNCH_DONE frames.

## PUNCH_GRANT Frame {#punch-grant-frame}

~~~
PUNCH_GRANT Frame {
    Type (i) = 0x3d7e96,
    Grant ID (i),
    Connection ID 1 Length (8),
    Connection ID 1 (8..160),
    Connection ID 2 Length (8),
    Connection ID 2 (8..160),
}
~~~

PUNCH_GRANT authorizes one attempt and supplies the two server-issued punching
connection IDs ({{punching-cids}}). The server MUST number new grants
consecutively from 0 within each connection, subject to the client's grant
limit ({{grant-limits}}).

Both Connection ID Length fields specify a length in bytes from 1 to 20;
other values MUST be treated as FRAME_ENCODING_ERROR.

PUNCH_GRANT is ack-eliciting and sent on a validated path. The Grant ID SHOULD
be retransmitted on loss until acknowledged.

This frame is only sent from the server to the client. Servers MUST treat
receipt of a PUNCH_GRANT frame as a connection error of type PROTOCOL_VIOLATION.

## MAX_PUNCH_GRANTS Frame {#max-punch-grants-frame}

~~~
MAX_PUNCH_GRANTS Frame {
    Type (i) = 0x3d7e97,
    Maximum Grants (i),
}
~~~

Maximum Grants is the cumulative number of grants the server may issue,
including grants already used. Servers MUST ignore values that do not increase
the limit.

MAX_PUNCH_GRANTS is ack-eliciting and sent on a validated path. On loss, the
client MUST retransmit its current limit unless it has already been
acknowledged.

This frame is only sent from the client to the server. Clients MUST treat
receipt of a MAX_PUNCH_GRANTS frame as a connection error of type
PROTOCOL_VIOLATION.

## PUNCH_REQUEST Frame

~~~
PUNCH_REQUEST Frame {
    Type (i) = 0x3d7e92,
    Grant ID (i),
    Connection ID Length (8),
    Connection ID (8..160),
    Client Address Type (8),
    Client IP Address (32..128),
    Client Port (16),
    Server Address Type (8),
    Server IP Address (32..128),
    Server Port (16),
}
~~~

The PUNCH_REQUEST frame contains the following fields:

Grant ID:

: The grant used for this attempt ({{punch-grant-frame}}).

Connection ID Length and Connection ID:

: The length in bytes and value of a client-issued punching connection ID
  ({{punching-cids}}). Lengths outside 1 to 20 MUST be treated as
  FRAME_ENCODING_ERROR.

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

The client MUST use a received, unused grant for each new attempt. Sending
PUNCH_REQUEST permanently binds that grant to the address pair, even if the
request is rejected. The client MUST send PUNCH_REQUEST:

* On an established path, using an ordinary destination connection ID and
  carrying one of its punching connection IDs. Only this copy starts the
  server's validation.
* In every probe packet containing PATH_CHALLENGE for the attempt, carrying
  its other punching connection ID.

The server MUST process both forms in either arrival order, ignoring duplicates
of each form, and process the direct copy before answering its PATH_CHALLENGE.
Both forms MUST use the same Grant ID and addresses; retransmissions MUST repeat
each form's connection ID. Unissued Grant IDs, mismatched destination connection
IDs, and detected conflicting values MUST be treated as PROTOCOL_VIOLATION.

PUNCH_REQUEST is a probing frame ({{Section 9.1 of RFC9000}}) and is
ack-eliciting. The established-path copy MUST be retransmitted on loss until it
is acknowledged or the server's PUNCH_DONE is received, even after local probing
ends. Direct copies MUST NOT be retransmitted independently.

This frame is only sent from the client to the server. Clients MUST treat
receipt of a PUNCH_REQUEST frame as a connection error of type
PROTOCOL_VIOLATION.

## PUNCH_DONE Frame {#punch-done-frame}

~~~
PUNCH_DONE Frame {
    Type (i) = 0x3d7e95,
    Grant ID (i),
    Status (i),
}
~~~

The Grant ID identifies the attempt. Status reports the sender's result:

* SUCCEEDED (0x00): The sender's path validation succeeded.
* FAILED (0x01): The sender's path validation timed out.
* REJECTED (0x02): The server did not start the attempt.

Each endpoint's Status MUST remain unchanged for a given Grant ID. Endpoints
MUST treat detected conflicting statuses from the peer for a grant as a
connection error of type PROTOCOL_VIOLATION.

PUNCH_DONE can only be sent in response to a PUNCH_REQUEST. A client that
receives a PUNCH_DONE frame with a Grant ID for which it didn't send a
PUNCH_REQUEST frame MUST close the connection with an error of type
PROTOCOL_VIOLATION. Due to packet reordering, a server might receive a
PUNCH_DONE frame before receiving the corresponding PUNCH_REQUEST frame.
Servers MUST treat an unissued Grant ID as a connection error of type
PROTOCOL_VIOLATION.

An unknown Status, or REJECTED received by a server, MUST be treated as a
connection error of type FRAME_ENCODING_ERROR.

If PUNCH_DONE arrives before the request sent on the established path, the
server MUST retain the Status and process that request normally when it arrives.

PUNCH_DONE is a non-probing, ack-eliciting frame sent on a validated path. It
MUST be retransmitted on loss until acknowledged.

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
