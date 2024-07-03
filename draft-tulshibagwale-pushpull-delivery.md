---
title: PushPull Based Security Event Token (SET) Delivery Using HTTP
abbrev: pushpull
docname: draft-tulshibagwale-pushpull-delivery-00
stand_alone: true
ipr: trust200902
cat: info # Check
submissiontype: IETF
# date autofilled by xml2rfc
area: sec

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


normative:
  RFC2119: # Keywords
  RFC6749: # OAuth 2.0
  RFC7519: # JWT
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
  SSF:
    target: http://openid.net/specs/openid-sse-framework-1_0.html
    title: OpenID Shared Signals and Events Framework Specification 1.0
    author:
      -
        ins: A. Tulshibagwale
        name: Atul Tulshibagwale
        org: Google
      -
        ins: T. Cappalli
        name: Tim Cappalli
        org: Microsoft
      -
        ins: M. Scurtescu
        name: Marius Scurtescu
        org: Coinbase
      -
        ins: A. Backman
        name: Annabelle Backman
        org: Amazon
      -
        ins: John Bradley
        name: John Bradley
        org: Yubico
    date: 2024-06


--- abstract
In situations where a transmitter of Security Event Tokens (SETs) {{RFC8417}} to a network peer is also a receiver of SETs from the same peer, it is helpful to have an efficient way of sending an receiving SETs in one HTTP transaction. Using current mechanisms such as Push-Based Delivery of Security Event Tokens (SETs) Using HTTP {{RFC8935}} or Poll-Based Delivery of Security Event Tokens (SETs) Using HTTP {{RFC8936}} both require two or more HTTP connections to exchange SETs between peers. In many cases, such as when using the OpenID Shared Signals Framework {{SSF}}, the situation where each entity is both a transmitter and receiver is getting increasingly common. In addition, this specification enables the transmission and reception of multiple SETs in one HTTP connection.
--- middle

# Introduction
Workloads that exchange SETs with each other ("Transceivers") can do so efficiently using the protocol defined in this specification. Although this specification works along the lines of the DeliveryPush {{RFC8935}} and DeliveryPoll {{RFC8936}} specifications, it makes a few important additions:

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

# Initiating Communication

A Transceiver can initiate communication with a Peer in order to:

* Acknowledge previously received SETs from the Peer.
* Send SETs to the Peer.
* Both acknowledge previously received SETs from the Peer and send SETs to the Peer.

To initiate communication, the Initiator makes a HTTP POST request to the Responder's Pushpull Endpoint {{pushpull-endpoint}}. The body of this request is of the content type "application/json". It contains a Communication Object {{communication-object}}, and the following additional field MAY be present:

maxResponseEvents
: OPTIONAL. A number which specifies the maximum number of events the Responder can include in its response to the Initiator. If this field is absent in the request, the Responder MAY include any number of events in the response.

# Response Communication

A Responder MUST respond to a communication from an Initiator by sending an HTTP Response.

## Success Response
If the Responder is successful in processing the request, it MUST return the HTTP status code 200 (OK). The response MUST have the content-type "application/json" and the response MUST include a Communication Object {{communication-object}}.

## Error Response
The Responder MUST respond with an error response if it is unable to process the request. The error response MUST include the appropriate error code as described in Section 2.4 of DeliveryPush {{RFC8935}}.

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

# Security Considerations

## Authentication and Authorization
Transceivers MUST follow the procedures described in section {{authn-and-authz}} in order to securely authenticate and authorize Peers

## HTTP and TLS
Transceivers MUST use TLS {{RFC8446}} to communicate with Peers and is subject to the security considerations of HTTP {{RFC9110}} Section 17.

## Denial of Service
A Responder may be vulnerable to denial of service attacks wherein a large number of spurious requests need to be processed. Having efficient authorization mechanisms such as OAuth 2.0 {{RFC6749}} can mitigate such attacks by leveraging standard infrastructure that is designed to handle such attacks.

# Privacy Considerations
SETs may contain confidential information, and Transceivers receiving SETs must be careful not to log such content or ensure that sensitive information from the SET is redacted before logging.

# IANA Considerations
This specification does not add any new IANA considerations.


