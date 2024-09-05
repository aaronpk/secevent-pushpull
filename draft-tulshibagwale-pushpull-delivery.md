---
title: Push And Pull Based Security Event Token (SET) Delivery
abbrev: pushpull
docname: draft-tulshibagwale-pushpull-delivery-latest
stand_alone: true
ipr: trust200902
cat: info # Check
submissiontype: IETF
# date autofilled by xml2rfc
area: sec
wg: saag

venue:
  github: "SGNL-ai/pushpull"
  latest: "https://sgnl-ai.github.io/pushpull/"


lang: en
kw:
  - JSON Web Token
  - JWT
  - Security Event Token
  - SET
  - Delivery
  - JavaScript Object Notation
  - JSON
author:
- ins: A. Tulshibagwale
  name: Atul Tulshibagwale
  org: SGNL
  email: atul@sgnl.ai

contributor:
- ins: E. Gustavson
  name: Erik Gustavson
  org: SGNL
  email: erik@sgnl.ai
- ins: A. Deshpande
  name: Apoorva Deshpande
  org: Okta
  email: apoorva.deshpande@okta.com

normative:
  RFC2119: # Keywords
  RFC6455: # WebSocket
  RFC6749: # OAuth 2.0
  RFC8174: # uppercase / lowercase in keywords
  RFC8259: # JSON
  RFC8417: # SET
  RFC8446: # TLS
  RFC8935: # Push delivery
  RFC8936: # Poll delivery
  RFC9110: # HTTP
  OPRM:
    title: OAuth 2.0 Protected Resource Metadata
    target: https://datatracker.ietf.org/doc/draft-ietf-oauth-resource-metadata/
    date: 2024-05
    author:
    - ins: M.B. Jones
      name: Michael B. Jones
    - ins: P.Hunt
      name: Phil Hunt
    - ins: A. Parecki
      name: Aaron Parecki
  IANA.WebSocket.Subprotocol:
    title: WebSocket Subprotocol Name Registry
    target: https://www.iana.org/assignments/websocket/websocket.xml#subprotocol-name
    author:
     - name: IANA


--- abstract
In situations where a transmitter of Security Event Tokens (SETs) to a network peer is also a receiver of SETs from the same peer, it is helpful to have an efficient way of sending and receiving SETs in one HTTP transaction. Using current mechanisms such as "Push-Based Delivery of Security Event Tokens (SETs) Using HTTP" or "Poll-Based Delivery of Security Event Tokens (SETs) Using HTTP" both require two or more HTTP connections to exchange SETs between peers. In many cases, such as when using the OpenID Shared Signals Framework (SSF), the situation where each entity is both a transmitter and receiver is getting increasingly common. In addition, this specification enables the transmission and reception of multiple SETs in one HTTP connection.
--- middle

# Introduction
Workloads that exchange SETs {{RFC8417}} with each other ("Transceivers") can do so efficiently using the protocol defined in this specification. Although this specification works along the lines of the DeliveryPush {{RFC8935}} and DeliveryPoll {{RFC8936}} specifications, it makes a few important additions:

* A Transceiver initiating a communication can send multiple SETs in one HTTP connection to a Peer
* The Transceiver initiating communication can acknowledge previously received SETs in the same HTTP connection to the Peer
* The Peer responding to the communication can send multiple SETs in its response to a connection from the Transceiver
* The Peer responding to the communication can acknowledge previously received SETs in its response to the Transceiver

# Notational Conventions

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL
NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED",
"MAY", and "OPTIONAL" in this document are to be interpreted as
described in BCP 14 {{RFC2119}} {{RFC8174}} when, and only when,
they appear in all capitals, as shown here.

# Terminology

Transceiver
: A networked workload that can act both as a transmitter of SETs and a receiver of SETs. It communicates with other trusted Transceivers to transmit and receive SETs using the protocol defined herein.

Peer
: Another name for a Transceiver, used to signify the other end of the communication from a Transceiver.

Initiator
: A Transceiver initiating communication with a Peer.

Responder
: A Transceiver responding to communication from a Peer.

DeliveryPush
: The IETF RFC titled "Push-Based Delivery of Security Event Tokens (SETs) Using HTTP" {{RFC8935}}.

DeliveryPoll
: The IETF RFC titled "Poll-Based Delivery of Security Event Tokens (SETs) Using HTTP" {{RFC8936}}.

# Pushpull Endpoint {#pushpull-endpoint}
Each Transceiver that supports this specification MUST support a "Pushpull" endpoint. This endpoint MUST be capable of serving HTTP {{RFC9110}} requests. This endpoint MUST be TLS {{RFC8446}} enabled and MUST reject any communication not using TLS.

# Communication Object {#communication-object}
A Communication Object is a JSON object {{RFC8259}}, and is a unit of communication used in this specification used both in requests and responses. When used in a request, the Initiator MAY have additional fields defined the later sections below. The common fields of this object are:

sets
: OPTIONAL. A JSON object containing key-value pairs in which the key of a field is a string that contains the `jti` value of the SET that is specified in the value of the field. This field MAY be omitted to indicate that no SETs are being delivered by the initiator in this communication.

ack
: OPTIONAL. An array of strings, in which each string is the `jti` value of a previously received SET that is acknowledged in this object. This array MAY be empty or this field MAY be omitted to indicate that no previously received SETs are being acknowledged in this communication.

setErrs
: OPTIONAL. A JSON object containing key-value pairs in which the key of a field is a string that contains the `jti` value of a previously received SET that the sender of the communication object was unable to process. The value of the field is a JSON object that has the following fields:

  err
  : OPTIONAL. The short reason why the specified SET failed to be processed.

  description
  : OPTIONAL. An explanation of why the SET failed to be processed.

## Example
The following is a non-normative example of a Communication Object

~~~ json
{
  "sets": {
    "4d3559ec67504aaba65d40b0363faad8": 
    "eyJhbGciOiJub25lIn0.
    eyJqdGkiOiI0ZDM1NTllYzY3NTA0YWFiYTY1ZDQwYjAzNjNmYWFkOCIsImlhdC
    I6MTQ1ODQ5NjQwNCwiaXNzIjoiaHR0cHM6Ly9zY2ltLmV4YW1wbGUuY29tIiwi
    YXVkIjpbImh0dHBzOi8vc2NpbS5leGFtcGxlLmNvbS9GZWVkcy85OGQ1MjQ2MW
    ZhNWJiYzg3OTU5M2I3NzU0IiwiaHR0cHM6Ly9zY2ltLmV4YW1wbGUuY29tL0Zl
    ZWRzLzVkNzYwNDUxNmIxZDA4NjQxZDc2NzZlZTciXSwiZXZlbnRzIjp7InVybj
    ppZXRmOnBhcmFtczpzY2ltOmV2ZW50OmNyZWF0ZSI6eyJyZWYiOiJodHRwczov
    L3NjaW0uZXhhbXBsZS5jb20vVXNlcnMvNDRmNjE0MmRmOTZiZDZhYjYxZTc1Mj
    FkOSIsImF0dHJpYnV0ZXMiOlsiaWQiLCJuYW1lIiwidXNlck5hbWUiLCJwYXNz
    d29yZCIsImVtYWlscyJdfX19.",
    "3d0c3cf797584bd193bd0fb1bd4e7d30":
    "eyJhbGciOiJub25lIn0.
    eyJqdGkiOiIzZDBjM2NmNzk3NTg0YmQxOTNiZDBmYjFiZDRlN2QzMCIsImlhdC
    I6MTQ1ODQ5NjAyNSwiaXNzIjoiaHR0cHM6Ly9zY2ltLmV4YW1wbGUuY29tIiwi
    YXVkIjpbImh0dHBzOi8vamh1Yi5leGFtcGxlLmNvbS9GZWVkcy85OGQ1MjQ2MW
    ZhNWJiYzg3OTU5M2I3NzU0IiwiaHR0cHM6Ly9qaHViLmV4YW1wbGUuY29tL0Zl
    ZWRzLzVkNzYwNDUxNmIxZDA4NjQxZDc2NzZlZTciXSwic3ViIjoiaHR0cHM6Ly
    9zY2ltLmV4YW1wbGUuY29tL1VzZXJzLzQ0ZjYxNDJkZjk2YmQ2YWI2MWU3NTIx
    ZDkiLCJldmVudHMiOnsidXJuOmlldGY6cGFyYW1zOnNjaW06ZXZlbnQ6cGFzc3
    dvcmRSZXNldCI6eyJpZCI6IjQ0ZjYxNDJkZjk2YmQ2YWI2MWU3NTIxZDkifSwi
    aHR0cHM6Ly9leGFtcGxlLmNvbS9zY2ltL2V2ZW50L3Bhc3N3b3JkUmVzZXRFeH
    QiOnsicmVzZXRBdHRlbXB0cyI6NX19fQ."
  },
  "ack": [
    "f52901c4-3996-11ef-9454-0242ac120002",
    "0636e274-3997-11ef-9454-0242ac120002",
    "d563c724-79a0-4ff0-ba41-657fa5e2cb11"
  ],
  "setErrs": {
    "5c436b19-0958-4367-b408-2dd542606d3b" : {
      "err": "invalid subject",
      "description": "subject format not supported"
    }
  }
}
~~~
{: #fig-communication-object-example title="Example of a Communication Object"}

# HTTP Request Response Binding {#http-request-response-binding}
This section describes how Transceivers can use HTTP Requests and Responses to exchange Communication Objects described in  {{communication-object}}.

## Initiating Communication

A Transceiver can initiate communication with a Peer in order to:

* Acknowledge previously received SETs from the Peer.
* Send SETs to the Peer.
* Both acknowledge previously received SETs from the Peer and send SETs to the Peer.

To initiate communication, the Initiator makes a HTTP POST request to the Responder's Pushpull Endpoint {{pushpull-endpoint}}. The body of this request is of the content type "application/json". It contains a Communication Object {{communication-object}}, and the following additional field MAY be present:

maxResponseEvents
: OPTIONAL. A number which specifies the maximum number of events the Responder can include in its response to the Initiator. If this field is absent in the request, the Responder MAY include any number of events in the response. If this field is present, then the Responder MUST NOT include more events than the value of "maxResponseEvents" in its response to the specific request.

## Response Communication

A Responder MUST respond to a communication from an Initiator by sending an HTTP Response.

### Success Response {#http-success-response}
If the Responder is successful in processing the request, it MUST return the HTTP status code 200 (OK). The response MUST have the content-type "application/json" and the response MUST include a Communication Object {{communication-object}}.

### Error Response
The Responder MUST respond with an error response if it is unable to process the request. The error response MUST include the appropriate error code as described in Section 2.4 of DeliveryPush {{RFC8935}}.

### Out Of Order Responses
A Communication Object in a Response may contain `jti` values in its `ack` or `setErrs` that do not correspond to the SETs received in the same Request to which the Response is being sent. They MAY consist of values received in previous Requests.

## Example Request and Response
The following is a non-normative example of a request and its corresponding response

### Request

~~~ http
POST /pushpull-endpoint HTTP/1.1
Host: sharedsignals-transceiver.myorg.example
Content-type: application/json
Authorization: Bearer eyJraWQiOiIyMDIwXzEiLCJJhbGciOiJSUzI1NiJ9...

{
  "ack": [],
  "sets": {
    "4d3559ec67504aaba65d40b0363faad8": 
    "eyJhbGciOiJub25lIn0.
    eyJqdGkiOiI0ZDM1NTllYzY3NTA0YWFiYTY1ZDQwYjAzNjNmYWFkOCIsImlhdC
    I6MTQ1ODQ5NjQwNCwiaXNzIjoiaHR0cHM6Ly9zY2ltLmV4YW1wbGUuY29tIiwi
    YXVkIjpbImh0dHBzOi8vc2NpbS5leGFtcGxlLmNvbS9GZWVkcy85OGQ1MjQ2MW
    ZhNWJiYzg3OTU5M2I3NzU0IiwiaHR0cHM6Ly9zY2ltLmV4YW1wbGUuY29tL0Zl
    ZWRzLzVkNzYwNDUxNmIxZDA4NjQxZDc2NzZlZTciXSwiZXZlbnRzIjp7InVybj
    ppZXRmOnBhcmFtczpzY2ltOmV2ZW50OmNyZWF0ZSI6eyJyZWYiOiJodHRwczov
    L3NjaW0uZXhhbXBsZS5jb20vVXNlcnMvNDRmNjE0MmRmOTZiZDZhYjYxZTc1Mj
    FkOSIsImF0dHJpYnV0ZXMiOlsiaWQiLCJuYW1lIiwidXNlck5hbWUiLCJwYXNz
    d29yZCIsImVtYWlscyJdfX19.",
    "3d0c3cf797584bd193bd0fb1bd4e7d30":
    "eyJhbGciOiJub25lIn0.
    eyJqdGkiOiIzZDBjM2NmNzk3NTg0YmQxOTNiZDBmYjFiZDRlN2QzMCIsImlhdC
    I6MTQ1ODQ5NjAyNSwiaXNzIjoiaHR0cHM6Ly9zY2ltLmV4YW1wbGUuY29tIiwi
    YXVkIjpbImh0dHBzOi8vamh1Yi5leGFtcGxlLmNvbS9GZWVkcy85OGQ1MjQ2MW
    ZhNWJiYzg3OTU5M2I3NzU0IiwiaHR0cHM6Ly9qaHViLmV4YW1wbGUuY29tL0Zl
    ZWRzLzVkNzYwNDUxNmIxZDA4NjQxZDc2NzZlZTciXSwic3ViIjoiaHR0cHM6Ly
    9zY2ltLmV4YW1wbGUuY29tL1VzZXJzLzQ0ZjYxNDJkZjk2YmQ2YWI2MWU3NTIx
    ZDkiLCJldmVudHMiOnsidXJuOmlldGY6cGFyYW1zOnNjaW06ZXZlbnQ6cGFzc3
    dvcmRSZXNldCI6eyJpZCI6IjQ0ZjYxNDJkZjk2YmQ2YWI2MWU3NTIxZDkifSwi
    aHR0cHM6Ly9leGFtcGxlLmNvbS9zY2ltL2V2ZW50L3Bhc3N3b3JkUmVzZXRFeH
    QiOnsicmVzZXRBdHRlbXB0cyI6NX19fQ."
  },
  "setErrs": {
    "5c436b19-0958-4367-b408-2dd542606d3b" : {
      "err": "invalid subject",
      "description": "subject format not supported"
    }
  },
  "maxResponseEvents": 10
}
~~~
{: #fig-example-request title="Example Pushpull request"}
In the above example request, the Initiator does acknowledge any previous events. Delivers two SETs and reports an error on a previously received SET.

### Response
The following is a non-normative example of a response:

~~~ http
HTTP/1.1 200 OK
Content-type: application/json

{
  "ack": [
    "3d0c3cf797584bd193bd0fb1bd4e7d30"
  ],
  "sets": {
    "756E69717565206964656E746966696572":
    "eyJ0eXAiOiJzZWNldmVudCtqd3QiLCJhbGciOiJIUzI1NiJ9Cg.
  eyJpc3MiOiJodHRwczovL2lkcC5leGFtcGxlLmNvbS8iLCJqdGkiOiI3NTZFNjk
  3MTc1NjUyMDY5NjQ2NTZFNzQ2OTY2Njk2NTcyIiwiaWF0IjoxNTA4MTg0ODQ1LC
  JhdWQiOiI2MzZDNjk2NTZFNzQ1RjY5NjQiLCJldmVudHMiOnsiaHR0cHM6Ly9zY
  2hlbWFzLm9wZW5pZC5uZXQvc2VjZXZlbnQvcmlzYy9ldmVudC10eXBlL2FjY291
  bnQtZGlzYWJsZWQiOnsic3ViamVjdCI6eyJzdWJqZWN0X3R5cGUiOiJpc3Mtc3V
  iIiwiaXNzIjoiaHR0cHM6Ly9pZHAuZXhhbXBsZS5jb20vIiwic3ViIjoiNzM3NT
  YyNkE2NTYzNzQifSwicmVhc29uIjoiaGlqYWNraW5nIn19fQ.
  Y4rXxMD406P2edv00cr9Wf3_XwNtLjB9n-jTqN1_lLc"
  }
}
~~~
{: #fig-example-response title="Example Pushpull response"}
In the above example, the Responder acknowledges one of the SETs it previously received and provides a SET to deliver to the initiator. There are no errors reported by the Responder.

# WebSocket Binding
Transceivers MAY use WebSockets {{RFC6455}} to send and receive Communication Objects described in {{communication-object}}. Since WebSockets are a symmetric protocol, a Transceiver MAY send a Communication Object at any time to its Peer. In such communication, a Transceiver sends a Communication Object as Payload data over the WebSocket protocol to a Peer. Similarly, a Transceiver MAY receive a Communication Object from a Peer over a WebSocket connection, wherein the Communication Object is the Payload data. In all such WebSocket communication, the Payload data does not have any Extension data in it.

## Using WebSockets
During any communication initiated by a Transceiver, the Transceiver MAY request the Peer to use WebSockets {{RFC6455}} by requesting that the connection be upgraded to a WebSocket connection. If the Transceiver and its Peer can successfully perform the WebSocket handshake for the Pushpull Subprotocol described in {{pushpull-subprotocol-handshake}}, then the Transceiver and Peer MUST use WebSockets until the connection is closed. If the handshake fails, the Transceiver and Peer MAY use the HTTP Request Response Binding as described in {{http-request-response-binding}}

### Pushpull Subprotocol Handshake {#pushpull-subprotocol-handshake}
The Pushpull subprotocol is used to transport Communication Objects {{communication-object}} over a WebSocket connection. The Transceiver and its Peer agree to this subprotocol during the WebSocket handshake (see {{Section 1.3 of RFC6455}}).

During the Websocket handshake, the Initiator MUST include the value `pushpull` in the list of protocols for the `Sec-WebSocket-Protocol` header. The reply from the Responder MUST also include the value `pushpull` in the list of values in its own `Sec-WebSocket-Protocol` header, in order for the Initiator and Responder to use WebSockets.

# Authentication and Authorization {#authn-and-authz}
The Initiator MUST verify the identity of the Responder by validating the TLS certification presented by the Responder, and verifying that it is the intended recipient of the request, before sending the Communication Object {{communication-object}}.

The Initiator MUST attempt to obtain the OAuth Protected Resource Metadata {{OPRM}} for the Responder endpoint. If such metadata is found, the Initiator MUST obtain an access token using the metadata. If no such metadata is found, then the Initiator MAY use any means to authorize itself to the Responder.

The Responder MUST verify the identity and authorization of the Initiator. The Responder MAY use OAuth Protected Resource Metadata {{OPRM}} for this purpose, but the Responder MAY use other means to authorize the Initiator, which are beyond the scope of this specification.

# Delivery Reliability
A Transceiver MUST attempt to deliver any SETs it has previously attempted to deliver to a Peer until:
* It receives an acknowledgement through the `ack` value for that SET in a subsequent communication with the Peer
* It receives a `setErrs` object for that SET in a subsequent communication with the Peer
* It has attempted to deliver the SET a maximum number of times and has failed to communicate either due to communication errors or lack of inclusion in `ack` or `setErrs` in subsequent communications that were conducted for the maximum number of times. The maximum number of attempts MAY be set by the Transceiver for itself and SHOULD be communicated offline to the Peers.

If a Transceiver previously attempted to deliver a SET in a response to a Peer's request, the Transceiver MAY Initiate a request to the Peer in order to retry delivery of the SET. A Peer MUST be able to either provide `ack`s or `setErrs` for the same SETs either through requests or responses.

# All SETs Accounted For {#all-sets-accounted-for}
A Transceiver MUST ensure that it includes the `jti` value of each SET it receives, either in an `ack` or a `setErrs` value, to the Transceiver from which it received the SETs. A Transceiver SHOULD retry sending the same SET again if it was never responded to either in an `ack` value or in a `setErrs` value by a receiving Transceiver in a reasonable time period. A Transceiver MAY limit the number of times it retries sending a SET. A Transceiver MAY publish the retry time period and maximum number of retries to its peers, but such publication is outside the scope of this specification.

# Uniqueness of SETs
A Transceiver MUST NOT send two SETs with the same `jti` value if the SET has been either acknowledged through `ack` value or produced an error indicated by a `setErrs` value. If a Transceiver wishes to re-send an event after it has received a error response through a `setErrs` value, then it MUST generate a new SET that has a new (and unique) `jti` value.

# Security Considerations

## Authentication and Authorization
Transceivers MUST follow the procedures described in section {{authn-and-authz}} in order to securely authenticate and authorize Peers

## HTTP and TLS
Transceivers MUST use TLS {{RFC8446}} to communicate with Peers and is subject to the security considerations of HTTP {{RFC9110}} Section 17.

## Denial of Service
A Responder may be vulnerable to denial of service attacks wherein a large number of spurious requests need to be processed. Having efficient authorization mechanisms such as OAuth 2.0 {{RFC6749}} can mitigate such attacks by leveraging standard infrastructure that is designed to handle such attacks.

## Temporary Disconnection
Transceivers must make sure they respond to each SET received in a timely manner as described in the "All SETs Accounted For" section {{all-sets-accounted-for}}. This ensures that if there was a temporary disconnection between two Transceivers, say when a Responding Transceiver sent a Communication Object in the HTTP Response, that such disconnection is detected and the missing SETs can be retried.

# Privacy Considerations
SETs may contain confidential information, and Transceivers receiving SETs must be careful not to log such content or ensure that sensitive information from the SET is redacted before logging.

# IANA Considerations
The following WebSocket subprotocol will be added to the "WebSocket Subprotocol Name Registry" {{IANA.WebSocket.Subprotocol}}

* Subprotocol Identifier: `puhspull`
* Subprotocol Common Name: `WebSocket transport for Pushpull delivery of SETs`
* Subprotocol Definition: Section {{pushpull-subprotocol-handshake}} of this document.

--- back
