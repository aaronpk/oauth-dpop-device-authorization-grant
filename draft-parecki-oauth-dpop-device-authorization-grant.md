---
title: "DPoP for the OAuth 2.0 Device Authorization Grant"
abbrev: "DPoP for Device Authorization Grant"
category: std

docname: draft-parecki-oauth-dpop-device-authorization-grant-latest
submissiontype: IETF
number:
date:
consensus: true
v: 3
area: "Security"
workgroup: "Web Authorization Protocol"
keyword:
 - oauth
 - device
 - dpop
venue:
  group: "Web Authorization Protocol"
  type: "Working Group"
  mail: "oauth@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/oauth/"
  github: "aaronpk/oauth-dpop-device-authorization-grant"
  latest: "https://aaronpk.github.io/oauth-dpop-device-authorization-grant/draft-parecki-oauth-dpop-device-authorization-grant.html"

author:
 -
    fullname: Aaron Parecki
    organization: Okta
    email: aaron.parecki@okta.com

normative:

informative:

...

--- abstract

The OAuth 2.0 Device Authorization Grant [[RFC8628]] is an
authorization flow for devices with limited input capabilities.
Demonstrating Proof of Possession (DPoP) [[RFC9449]] is a mechanism
to sender-constrain OAuth 2.0 tokens. This document describes how to
use DPoP with the Device Authorization Grant to provide a higher
level of security for clients. It binds the DPoP key to the entire transaction,
from the initial device authorization request through the lifetime of
the issued tokens.


--- middle

# Introduction

The OAuth 2.0 Device Authorization Grant [[RFC8628]] provides a mechanism
for devices that lack a browser or have constrained input capabilities
to obtain authorization. The flow involves the device polling the token
endpoint while the user authorizes the request on a separate, more
capable device. Clients utilizing this flow are often public clients,
as defined in Section 2.1 of [[RFC6749]], making their issued tokens
susceptible to theft and misuse.

OAuth 2.0 Demonstrating Proof of Possession (DPoP) [[RFC9449]]
introduces a mechanism for sender-constraining access and refresh
tokens. It works by requiring the client to prove possession of a
cryptographic key with every request, binding the tokens to that key.
[[RFC9449]] explicitly details its application with Pushed Authorization
Requests (PAR) [[RFC9126]], a flow that, like the Device Authorization
Grant, begins with a direct back-channel request from the client to the
authorization server.

This specification formally defines the mechanism of using DPoP with the Device
Authorization Grant. By requiring a DPoP proof at the beginning of the
flow, the authorization server can bind the `device_code` to a specific
public key. This ensures that only the client possessing the
corresponding private key can complete the flow by polling the token
endpoint, thereby preventing a stolen `device_code` from being redeemed
by a malicious actor. The resulting access and refresh tokens are also
DPoP-bound, mitigating the risk of token leakage.

# Conventions and Definitions

{::boilerplate bcp14-tagged}


# Protocol Flow

The overall Device Authorization Grant flow remains as defined in
[[RFC8628]], with the addition of the DPoP header and associated
validation logic at two key steps: the device authorization request and
the device access token (polling) request.

## Device Authorization Request

To initiate the flow, the client makes a POST request to the device
authorization endpoint as specified in Section 3.1 of [[RFC8628]]. When
using DPoP, this request MUST include a DPoP header field containing a
valid DPoP proof JWT as defined in Section 4 of [[RFC9449]].

The authorization server MUST validate the DPoP proof according to the
rules in Section 4.3 of [[RFC9449]]. If the DPoP proof is invalid, the
authorization server MUST return an error response, and it is
RECOMMENDED that this be a 400 Bad Request with an error code of
`invalid_dpop_proof`.

If the DPoP proof is valid, the authorization server proceeds as defined
in Section 3.2 of [[RFC8628]] by generating a `device_code`, `user_code`,
etc. In addition, the authorization server MUST associate the public key
from the `jwk` header of the DPoP proof with the generated `device_code`
and store this association for later verification.

For example:

```
POST /device_authorization HTTP/1.1
Host: server.example.com
Content-Type: application/x-www-form-urlencoded
DPoP: eyJ0eXAiOiJkcG9wK2p3dCIsImFsZyI6IkVTMjU2IiwiandrIjp7Imt0eSI6Ik...

client_id=1406020730&scope=example_scope
```

## Device Access Token Request

After the user authorizes the request, the client begins polling the
token endpoint as described in Section 3.4 of [[RFC8628]]. Each access
token request (polling request) using the
`urn:ietf:params:oauth:grant-type:device_code` grant type MUST include a
DPoP header with DPoP proof JWT.

Upon receiving a token request, the authorization server MUST perform
the following steps in addition to the processing described in [[RFC8628]]:

* Look up the data associated with the received `device_code`.
* Retrieve the public key that was associated with the `device_code` during the device authorization request.
* Validate the DPoP proof JWT in the DPoP header of the current polling request per Section 4.3 of [[RFC9449]].
* Verify that the public key in the `jwk` header of the DPoP proof matches the public key associated with the `device_code`.

If any of these checks fail, the authorization server MUST reject
the request with an `invalid_grant` error.

If all checks are successful and the user has approved the grant, the
authorization server issues an access token and an optional refresh
token. The issued access token MUST be bound to the DPoP public key,
and the access token response MUST include `"token_type": "DPoP"``, as
specified in Section 5 of [[RFC9449]].

Any issued refresh token MUST also be bound to the same DPoP public key.


# Security Considerations

## Device Code Binding

The primary security benefit of this specification is the binding of
the DPoP key to the `device_code` at the start of the authorization
flow. In the standard Device Authorization Grant, an attacker who obtains
a valid `device_code` (e.g., through log inspection or a compromised
device) can start polling the token endpoint. If the attacker completes
the user authorization step (e.g., via a phishing attack that tricks the
user into entering the `user_code`), they can obtain the access token.

By binding the `device_code` to the client's DPoP key, this attack is
prevented. The attacker's polling requests to the token endpoint will
fail because they cannot produce a valid DPoP proof signed with the
private key corresponding to the public key bound to the `device_code`.
Only the legitimate client can successfully redeem the `device_code`.

## Client Impersonation

This specification does not prevent a malicious application on a device
from initiating a device flow and using DPoP correctly. It protects the
integrity of a single authorization flow by ensuring the same cryptographic
identity (the DPoP key pair) is used throughout. It is not a client
authentication mechanism. As such, the security considerations for public
clients in Section 5 of [[RFC8628]] and [[RFC9700]] remain relevant.


## General DPoP Considerations

All security considerations from [[RFC9449]] apply, including those
regarding DPoP proof replay, nonce usage, signature algorithms, and the
need to protect the private key on the client device.


# IANA Considerations

This document has no IANA actions.


--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
