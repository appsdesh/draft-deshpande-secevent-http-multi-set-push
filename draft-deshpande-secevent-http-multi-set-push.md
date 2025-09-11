---
title: "Push-Based Delivery For Multiple Security Event Token (SET) Using HTTP"
abbrev: "Multi-Push SET"
category: std

docname: draft-deshpande-secevent-http-multi-set-push-latest
submissiontype: IETF
number:
date:
v: 3
area: "Security"
workgroup: "Security Events"
keyword:
 - security event
 - secevent
venue:
  group: "Security Events"
  type: "Working Group"
  mail: "id-event@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/id-event/"
  github: "aaronpk/draft-deshpande-secevent-http-multi-push"
  latest: "https://drafts.aaronpk.com/draft-deshpande-secevent-http-multi-push/draft-deshpande-secevent-http-multi-push.html"

author:
 -
    fullname: Apoorva Deshpande
    organization: Okta
    email: apoorva.deshpande@okta.com
 -
    fullname: Aaron Parecki
    organization: Okta
    email: aaron@parecki.com

normative:
   RFC8417:
   RFC7231:
   RFC8935:
   RFC9110:
   RFC8446:
   I-D.ietf-oauth-resource-metadata:
   RFC8259:

informative:


--- abstract

This specification defines how multiple Security Event Tokens (SETs) can be
delivered to an intended recipient using HTTP POST over TLS.  The SETs
are transmitted in the body of an HTTP POST request to an endpoint
operated by the recipient, and the recipient indicates successful or
failed transmission via the HTTP response.

--- middle

# Introduction

This specification defines a mechanism by which a transmitter of a Security Event Token (SET) {{RFC8417}} can deliver multiple SETs to an intended SET Recipient via HTTP POST {{RFC7231}} over TLS in a single POST call. {{RFC8935}} focuses on the delivery of the single SET to the receiver. This specification builds onto {{RFC8935}} to transmit multiple SETs to the receiver in a single POST call.

Multi-push SET delivery is intended to help in following scenarios:

- The transmitter of the SET has multiple outstanding SETs to be communicated to the receiver
- The transmitter wants to reduce the number of outbound calls to the same receiver to optimize performance, avoid being ratelimited when number of SETs to be communicated is high
- The receiver wants to optimize processing multiple SETs

Multi-push specification will handle all the usecases and scenarios for the {{RFC8935}} and make it more extensible to support multiple SETs per one outbound POST call.

Similar to {{RFC8935}} this specification makes mechanism for exchanging configuration metadata such as endpoint URLs, cryptographic keys, and possible implementation constraints such as buffer size limitations between the transmitter and recipient is out of scope.

# Multi-Push Endpoint

Each Receiver that supports this specification MUST support a "multi-push" endpoint. This endpoint MUST be capable of serving HTTP POST {{RFC7231}} requests. This endpoint MUST be TLS {{RFC8446}} enabled and MUST reject any communication not using TLS.
The Transmitter obtains the multi-push endpoint outside the scope of this specification.


# SET delivery semantics

In a multi-push based SET delivery using HTTP over TLS, zero or more SETs are delivered in a JSON {{RFC8259}} document
to the SET Receiver. The receiver either acknowledges the successful receipt of the SETs or indicates failure in processing of one or more SETs in a JSON document to the Transmitter.

After successful (acknowledged) SET delivery, SET Transmitters are not required to retain or record SETs for retransmission. Once a SET is acknowledged, the SET Recipient SHALL be responsible for retention, if needed. Transmitters may also discard undelivered SETs under deployment-specific conditions, such as if they have not been acknowledged (successful or failure) for over too long a period of time or if an excessive amount of storage is needed to retain them.

Upon receiving a SET, the SET Recipient reads the SET and validates it in the manner described in {{Section 2 of RFC8935}}. The SET Recipient MUST acknowledge receipt to the SET Transmitter, and SHOULD do so in a timely fashion (e.g., miliseconds). The SET Recipient SHALL NOT use the event acknowledgement mechanism to report event errors other than those relating to the parsing and validation of the SET.

## Acknowledgement for all SETs
A Transmitter MUST ensure that it includes the `jti` value of each SET it receives, either in an ack or a setErrs value, to the Transmitter from which it received the SETs. A Transmitter SHOULD retry sending the same SET again if it was never responded to either in an ack value or in a setErrs value by a receiver in a reasonable time period. A Transmitter MAY limit the number of times it retries sending a SET. A Transmitter MAY publish the retry time period and maximum number of retries to its peers, but such publication is outside the scope of this specification.

## Uniqueness of SETs
A Transmitter MUST NOT send two SETs with the same `jti` value if the SET has been either acknowledged through ack value or produced an error indicated by a setErrs value. If a Transmitter wishes to re-send an event after it has received a error response through a setErrs value, then it MUST generate a new SET that has a new (and unique) jti value.

## Transmitting SETs

A Transmitter may initiate communication with the receiver in order to:

 -  Send SETs to the Receiver
 -  Receive acknowledgement of the SETs in response

The body of this request is of the content type "application/json". It MAY contain the following fields:

`sets`
OPTIONAL. A JSON object containing key-value pairs in which the key of a field is a string that contains the jti claim of the SET that is specified in the value of the field. This field MAY be omitted to indicate that no SETs are being delivered by the initiator in this communication. The Transmitter SHOULD limit 20 SETs in the sets.


The following is a non-normative example of a response.

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
        }
      }

_Figure 1: Example of SET Transmission_

In the above example, the Transmitter is sending 2 SETs to the Receiver.

      {
        "sets": {},
      }

_Figure 2: Example of empty SET transmission_

In the above example, the Transmitter is sending zero SETs to the Receiver. This placeholder/empty request provides the Receiver to respond back with ack/err for previously transmitted SETs

## Response Communication

A Receiver MUST repond to the communication by sending an HTTP response. The body of this response is of the content type "application/json". It contains MAY contain the following fields:

`ack`
OPTIONAL. An array of strings, in which each string is the jti value of a previously received SET that is acknowledged in this object. This array MAY be empty or this field MAY be omitted to indicate that no previously received SETs are being acknowledged in this communication.

`setErrs`
OPTIONAL. A JSON object containing key-value pairs in which the key of a field is a string that contains the jti value of a previously received SET that the sender of the communication object was unable to process. The value of the field is a JSON object that has the following fields:

`err`
OPTIONAL. The short reason why the specified SET failed to be processed.

`description`
OPTIONAL. An explanation of why the SET failed to be processed

### Success Response

If the Receiver is successful in processing the request, it MUST return the HTTP status code 200 (OK). The response MUST have the content-type "application/json".

      HTTP/1.1 200 OK
      Content-type: application/json

      {
        "ack": [
          "3d0c3cf797584bd193bd0fb1bd4e7d30"
        ]
      }

_Figure 3: Example of SET Transmission response with ack_

In the above example, the Receiver acknowledges one of the SETs it previously received. There are no errors reported by the Receiver.

      HTTP/1.1 200 OK
      Content-type: application/json

      {
         "ack": [
          "f52901c499611ef94540242ac12000322",
          "0636e274399711ef9454-0242ac120002",
          "d563c72479a04ff0ba415657fa5e2cb11"
         ],
         "setErrs": {
          "4d3559ec67504aaba65d40b0363faad8" : {
            "err": "invalid subject",
            "description": "subject format not supported"
          }
         }
      }

_Figure 4: Example of SET Transmission response, ack and errors_

In the above example, the Receiver acknowledges three of the SETs it previously received. There are errors reported by the Receiver for acklowledging one SET.

#### Out of order delivery

A Response may contain `jti` values in its ack or setErrs that do not correspond to the SETs received in the same Request to which the Response is being sent. They MAY consist of values received in previous Requests.

### Error Response

The receiver MUST respond with an error response if it is unable to process the request. The error response MUST include the appropriate error code as described in {{Section 2.4 of RFC8935}}.

# Authentication and Authorization {#authn-and-authz}

The Transmitter MUST verify the identity of the Receiver by validating
the TLS certification presented by the Receiver, and verifying that
it is the intended recipient of the request, before sending the SETs.

The Transmitter MUST attempt to obtain the OAuth Protected Resource
Metadata {{I-D.ietf-oauth-resource-metadata}} for the Receiver's multi-push endpoint.  If such metadata is
found, the Transmitter MUST obtain an access token using the metadata.
If no such metadata is found, then the Transmitter MAY use any means to
authorize itself to the Receiver.

# Delivery Reliability
A Transmitter MUST attempt to deliver any SETs it has previously attempted to deliver to a Peer until:
   - It receives an acknowledgement through the ack value for that SET in a subsequent communication with the Peer
   - It receives a setErrs object for that SET in a subsequent communication with the Peer
   - It has attempted to deliver the SET a maximum number of times and has failed to communicate either due to communication errors or lack of inclusion in ack or setErrs in subsequent communications that were conducted for the maximum number of times. The maximum number of attempts MAY be set by the Transmitter for itself and SHOULD be communicated offline to the Peers

Additionally consider Delivery Relieability aspects discussed in {{Section 4 of RFC8935}} .

# Conventions and Definitions

{::boilerplate bcp14-tagged}


# Security Considerations

## Too many SETs in the response
Receiver MUST inform the transmitter (out of band) the maximum number of SETs that could be consumed in a single call. The Transmitter MUST obey the maximum number of SETs to be communicated to the receiver. This will avoid any potential truncations/loss of information at the receiver.

## Authentication and Authorization
Transmitter MUST follow the procedures described in section {{authn-and-authz}} in order to securely authenticate and authorize Peers

## HTTP and TLS
Transmitter MUST use TLS {{RFC8446}} to communicate with Receiver and is subject to the security considerations of HTTP {{Section 17 of RFC9110}}.

Additional security consideration in {{Section 5 of RFC8935}}.

# Privacy Considerations

Privacy Considerations from {{Section 6 of RFC8935}} apply.

# IANA Considerations

This document has no IANA actions.


--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
