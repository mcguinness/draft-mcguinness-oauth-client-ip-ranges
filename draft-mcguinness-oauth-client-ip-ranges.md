---
title: "OAuth Client IP Range Metadata"
abbrev: "OAuth Client IP Ranges"
category: std

docname: draft-mcguinness-oauth-client-ip-ranges-latest
submissiontype: IETF
number:
date:
consensus: true
v: 3
ipr: trust200902
area: "Security"
workgroup: "Web Authorization Protocol"
keyword:
 - OAuth
 - client metadata
 - IP address
 - network access control
venue:
  group: "Web Authorization Protocol"
  type: "Working Group"
  mail: "oauth@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/oauth/"
  github: "mcguinness/draft-mcguinness-oauth-client-ip-ranges"
  latest: "https://mcguinness.github.io/draft-mcguinness-oauth-client-ip-ranges/draft-mcguinness-oauth-client-ip-ranges.html"

author:
 -
    fullname: Karl McGuinness
    organization: Independent
    email: public@karlmcguinness.com
 -
    fullname: Xiaoyi Zhang
    organization: Databricks
    email: xiaoyi.zhang@databricks.com

normative:
  RFC3986:
  RFC4291:
  RFC5952:
  RFC6749:
  RFC6890:
  RFC8190:
  RFC7591:
  RFC8259:
  RFC9110:
  RFC9111:
  RFC9911:

informative:
  RFC7239:
  RFC7592:
  RFC8628:
  SSF:
    title: "OpenID Shared Signals Framework Specification 1.0"
    target: https://openid.net/specs/openid-sharedsignals-framework-1_0-final.html
    author:
      - ins: A. Tulshibagwale
      - ins: T. Cappalli
      - ins: M. Scurtescu
      - ins: A. Backman
      - ins: J. Bradley
      - ins: S. Miel
    date: 2025-08-29
  CIMD: I-D.ietf-oauth-client-id-metadata-document
  JAFAR: I-D.illyes-webbotauth-jafar

--- abstract

This specification defines OAuth client metadata parameters for
declaring the IP address ranges from which a client originates network
requests. The `client_ip_ranges` parameter carries a set of IPv4 and
IPv6 prefixes by value. The `client_ip_ranges_uri` parameter references
an HTTPS resource containing the set, which supports large and
frequently changing sets through standard HTTP retrieval and caching.
The parameters can be used with OAuth 2.0 Dynamic Client Registration,
OAuth Client ID Metadata Documents, and other mechanisms that convey
client metadata. Authorization servers and other parties that obtain a
client's metadata can use the declared ranges as an input to network
access control.

--- middle

# Introduction {#intro}

OAuth clients that run as hosted services commonly originate requests
from a known set of network addresses. Authorization servers, resource
servers, and the network infrastructure in front of them use these
addresses to restrict which sources can reach the token endpoint and
protected resources. In current practice, client operators publish their
address ranges through vendor-specific mechanisms, and deployments
transcribe them into network access controls by hand. Client metadata
({{Section 2 of RFC7591}}) has no parameter for the network origins of a
client.

This specification defines two client metadata parameters that declare
the IP address ranges from which a client originates requests:
`client_ip_ranges`, which carries the ranges by value, and
`client_ip_ranges_uri`, which references a Client IP Range Document that
contains them. As with the `jwks` and `jwks_uri` parameters of
{{RFC7591}}, the two forms have the same semantics and are mutually
exclusive. The reference form lets a client publish a large or
frequently changing set without updating its registration. The
parameters can be used with OAuth 2.0 Dynamic Client Registration
{{RFC7591}}, OAuth Client ID Metadata Documents {{CIMD}}, and other
mechanisms that convey client metadata.

Declared ranges are an input to network access control. Deployments use
source address restrictions to limit where a client's credentials and
grants can be used. Declared ranges do not authenticate the client or
authorize a request ({{use}}).

This specification defines the metadata and the processing requirements
for authorization servers that use it. Other parties that obtain a
client's metadata can apply the same processing. Selecting the clients
subject to enforcement, approving declared ranges, and configuring
network enforcement points are local policy; {{enforcement-points}}
describes typical deployments. Enforcement by resource servers is
outside the scope of this specification.

## Requirements Notation and Conventions {#conventions}

{::boilerplate bcp14-tagged-bcp14}

## Terminology {#terminology}

This specification uses the terms "access token", "authorization
server", "client", "client authentication", "protected resource",
"redirection endpoint", "resource owner", "resource server", and "token
endpoint" defined by OAuth 2.0 {{RFC6749}}, and the terms "client
metadata" and "software statement" defined by OAuth 2.0 Dynamic Client
Registration {{RFC7591}}.

This specification defines the following terms:

IP range set:
: The set of IPv4 and IPv6 prefixes that a client declares, either by
  value with `client_ip_ranges` or by reference with
  `client_ip_ranges_uri`.

Client IP Range Document:
: A JSON document, defined in {{document}}, that contains an IP range
  set and is referenced by `client_ip_ranges_uri`.

accepted IP range set:
: The IP range set, if any, that an authorization server has validated
  and accepted for a client and currently uses for it, including such a
  set after narrowing ({{range-changes}}). An authorization server that
  stops using an IP range set no longer has an accepted IP range set for
  the client.

# Client IP Range Metadata {#metadata}

This specification defines the following client metadata parameters
({{Section 2 of RFC7591}}) for declaring the IP address ranges from
which a client originates requests ({{use}}):

`client_ip_ranges`:
: OPTIONAL. JSON array of strings, each an IPv4 or IPv6 prefix
  ({{prefix-syntax}}). The array is a set. The authorization server MUST
  NOT assign meaning to the order of entries or to duplicate entries. An
  empty array declares an empty IP range set.

`client_ip_ranges_uri`:
: OPTIONAL. URL of a Client IP Range Document ({{document}}) that
  contains the client's IP range set. The URL MUST use the `https`
  scheme and MUST NOT contain a fragment or userinfo component
  {{RFC3986}}. The authorization server retrieves the document as
  described in {{retrieval}}.

`client_ip_ranges_uri` conveys an IP range set by reference, with the
same semantics as `client_ip_ranges`. The `client_ip_ranges` and
`client_ip_ranges_uri` parameters MUST NOT both be present in the same
client metadata. If neither is present, the client makes no declaration,
which differs from declaring an empty IP range set. This specification
sets no maximum number of entries; the authorization server applies its
own bounds ({{resource-exhaustion}}).

## Prefix Syntax {#prefix-syntax}

Each prefix MUST conform to the `ip-prefix` type defined in
{{Section 4 of RFC9911}}, that is, match in full every pattern of either
the `ipv4-prefix` or the `ipv6-prefix` type. A value that does not
conform to the `ip-prefix` type is invalid.

The client, and the party hosting a Client IP Range Document, MUST
represent each prefix in the canonical format of that type, with all
bits outside the prefix length set to zero and, for IPv6, the address
represented as defined in {{Section 4 of RFC5952}}. A value that is not
in canonical format is nonetheless valid and denotes the prefix given by
its network address and prefix length ({{Section 4 of RFC9911}}). For
example, `192.0.2.1/24` denotes `192.0.2.0/24`, and `2001:DB8::1/64`
denotes `2001:db8::/64`.

A prefix within the IPv4-mapped IPv6 address range `::ffff:0:0/96`
({{Section 2.5.5.2 of RFC4291}}) is invalid; IPv4 ranges are expressed
as IPv4 prefixes. Every prefix length that the `ip-prefix` type permits,
including zero, is valid. Whether to accept a broad prefix is local
policy ({{range-changes}}).

## Metadata Validation {#validation}

If a trusted software statement ({{Section 2.3 of RFC7591}}) carries
`client_ip_ranges` or `client_ip_ranges_uri`, the authorization server
ignores both parameters elsewhere in the client metadata, whether in a
registration request or in a Client ID Metadata Document
({{Section 4.3 of CIMD}}). This extends the precedence rule of
{{Section 2 of RFC7591}} to the pair of parameters.

After this precedence is applied, client IP range metadata is invalid if
any of the following is true:

1. Both `client_ip_ranges` and `client_ip_ranges_uri` are present.
2. The value of `client_ip_ranges` is not an array of strings, or any
   entry is an invalid prefix ({{prefix-syntax}}).
3. The value of `client_ip_ranges_uri` does not meet the requirements
   for `client_ip_ranges_uri` in {{metadata}}.
4. The value of `client_ip_ranges` has more entries than the
   authorization server's bound on prefixes ({{resource-exhaustion}}).

If a client registration request contains invalid client IP range
metadata, the authorization server responds with an HTTP 400 (Bad
Request) error response as described in {{Section 3.2.2 of RFC7591}},
using `invalid_client_metadata` as the error code value, or
`invalid_software_statement` if the invalid value comes from a trusted
software statement. If a Client ID Metadata Document {{CIMD}} contains
invalid client IP range metadata, the authorization server treats the
client IP range metadata as a validation failure
({{retrieval-failures}}) and does not treat the rest of the document as
invalid.

An authorization server that does not support these parameters ignores
them ({{Section 2 of RFC7591}}). The client information response
({{Section 3.2.1 of RFC7591}}) shows whether the authorization server
registered them.

# Client IP Range Document {#document}

A Client IP Range Document is a JSON {{RFC8259}} object with the
following member:

`client_ip_ranges`:
: REQUIRED. JSON array of prefixes, with the same syntax and semantics
  as `client_ip_ranges` ({{metadata}}).

The authorization server MUST ignore members that it does not
understand. A document whose top-level object contains duplicate member
names is invalid; {{Section 4 of RFC8259}} notes that receivers handle
such objects unpredictably. The document is self-contained, and the
authorization server MUST NOT retrieve other resources to complete it.

All prefixes in the document apply equally to each client whose metadata
references it. Multiple clients can reference the same document, and the
document can be hosted by a party other than the client, such as its
hosting provider.

The following is a non-normative example of a Client IP Range Document:

~~~ json
{
  "client_ip_ranges": [
    "192.0.2.0/24",
    "198.51.100.10/32",
    "2001:db8:100::/48",
    "2001:db8:200::/48"
  ]
}
~~~

# Retrieval and Acceptance {#retrieval}

## Retrieval {#fetch}

An authorization server retrieves a Client IP Range Document by sending
an HTTP GET request to the `client_ip_ranges_uri` URL, subject to
{{ssrf}}. A response is successful if it has status code 200 (OK), the
`application/json` media type, and a complete body within the
authorization server's bounds ({{resource-exhaustion}}), or if it has
status code 304 (Not Modified) ({{Section 15.4.5 of RFC9110}}) in answer
to a conditional request ({{retrieval-scheduling}}). A 304 response
indicates that the stored response is current.

The party hosting a Client IP Range Document MUST serve it with the
`application/json` media type and MUST NOT require authentication to
retrieve it unless that party and the authorization server have agreed
on an authentication mechanism, which is outside the scope of this
specification. The authorization server MUST NOT send credentials
associated with an OAuth request, such as client credentials, access
tokens, or authorization codes, when retrieving a Client IP Range
Document.

The authorization server MUST NOT treat a failed retrieval, an
unsuccessful response, or an invalid document as an empty IP range set;
each is a failure ({{retrieval-failures}}).

## Caching {#caching}

The party hosting a Client IP Range Document SHOULD support HTTP caching
{{RFC9111}} and SHOULD include the `Cache-Control`
({{Section 5.2 of RFC9111}}) and `ETag` ({{Section 8.8.3 of RFC9110}})
response header fields.

The accepted IP range set is application state, distinct from any HTTP
cache ({{Section 6 of RFC9111}}). A fresh response does not make its IP
range set accepted, and expiry of a cached response does not end use of
the accepted IP range set ({{acceptance}}). The authorization server
SHOULD retrieve the document periodically to obtain changes.

## Acceptance {#acceptance}

Before accepting an IP range set, whether it is conveyed by
`client_ip_ranges` or in a Client IP Range Document, the authorization
server MUST ensure the following:

1. The client IP range metadata that conveys the set is valid
   ({{validation}}), and every prefix in the set is valid
   ({{prefix-syntax}}).
2. If the set was obtained from a Client IP Range Document, the response
   is successful ({{fetch}}) and the complete document is valid
   ({{document}}).

If any of these checks fails, the authorization server MUST reject the
entire set. Acceptance is also subject to local policy
({{range-changes}}). Registering client IP range metadata does not by
itself accept its IP range set.

An accepted IP range set replaces the previously accepted IP range set
in full; this specification defines no incremental update format. The
accepted IP range set remains in effect until the authorization server
accepts a replacement, narrows it, or stops using it
({{range-changes}}). A validated IP range set that has not yet been
accepted under local policy does not replace it.

A client SHOULD add a prefix to its IP range set before it originates
requests from addresses in that prefix and SHOULD keep the prefix in the
set until it no longer originates requests from them.

## Changes to Client Metadata {#metadata-changes}

A client's metadata can change the value of `client_ip_ranges_uri`,
switch between `client_ip_ranges` and `client_ip_ranges_uri`, or remove
both. The new metadata replaces the previous declaration.

When the metadata changes representation or URI, the accepted IP range
set remains in effect until the authorization server accepts a set under
the new metadata or stops using it ({{acceptance}}). An invalid new
declaration or an unretrievable new document is a failure
({{retrieval-failures}}). If the previous metadata contained
`client_ip_ranges_uri`, the authorization server does not retrieve that
URI again for the client.

Removal of both parameters withdraws the client's declaration
({{retrieval-failures}}). In a client update request
({{Section 2.2 of RFC7592}}), omitting both parameters requests their
removal, which the authorization server can ignore. The client
information response ({{Section 3 of RFC7592}}) shows whether they were
removed.

## Failures and Withdrawal {#retrieval-failures}

An authorization server MUST NOT admit a source address that it did not
admit before because of a failure to retrieve a Client IP Range
Document, a failure to validate client IP range metadata or a retrieved
document, or, by itself, withdrawal of the client's declaration
({{metadata-changes}}).

After a failure, the authorization server MAY continue to use the
accepted IP range set, within the retention limit ({{range-changes}}).
Relaxing enforcement after a withdrawal is a local policy decision, not
an automatic consequence of the withdrawal.

If an authorization server requires an accepted IP range set for a
client, it MUST reject requests from the client while it has no accepted
IP range set for the client, whether because the client has made no
declaration, because its declaration is invalid or cannot be retrieved,
or because a validated IP range set has not yet been accepted under
local policy.

Otherwise, what applies while a client has no accepted IP range set is
local policy, except after the retention limit ({{range-changes}}) is
exceeded.

## Lifecycle Summary {#lifecycle}

The following table summarizes the accepted IP range set after each
event, as specified in {{acceptance}}, {{retrieval-failures}}, and
{{range-changes}}. It adds no requirements. Whenever a client has no
accepted IP range set, an authorization server that requires one rejects
requests from the client.

| Event | Accepted IP range set afterward |
|---|---|
| First declaration validated | None, until accepted under local policy |
| Replacement accepted | The replacement |
| Replacement validated, not yet accepted | Unchanged or narrowed to the intersection, at most until the retention limit |
| Retrieval or validation fails | Unchanged, at most until the retention limit |
| Declaration withdrawn | Unchanged, at most until the retention limit |
| Retention limit exceeded | None; an authorization server that was enforcing the set rejects requests from the client until it accepts a new set or local policy removes the client from enforcement |

# Use of Client IP Range Metadata {#use}

The IP range set identifies source addresses from which the client
originates requests directly to authorization servers and resource
servers, such as token requests and requests to protected resources. It
does not describe the network origins of user agents, redirection
endpoints, or requests initiated by the resource owner.

A source address matches an IP range set if it is within any of its
prefixes. Before matching, the authorization server MUST convert an
IPv4-mapped IPv6 source address to the IPv4 address it contains. An
authorization server that enforces a client's accepted IP range set
admits requests from the client only from source addresses that match
it; if that set is empty, it admits none.

An authorization server MAY use the IP range set as an input to local
network access control. The IP range set is a declaration made by or on
behalf of the client, and a match does not authenticate the client or
authorize a request. The authorization server MUST NOT use a source
address match in place of client authentication, access token
validation, or any other security control required by OAuth.

Enforcement by resource servers, including how a resource server
determines the client associated with a request and obtains that
client's metadata, is outside the scope of this specification and could
be addressed by a companion specification.

## Enforcement by Authorization Servers {#as-enforcement}

An authorization server can enforce a client's accepted IP range set at
the token endpoint and at other endpoints that receive direct requests
from the client, once it has identified the client, either through
client authentication or, for a public client
({{Section 2.1 of RFC6749}}), as the client to which a grant presented
in the request was issued. This specification does not define how an
authorization server identifies a public client in a request that
presents no grant, such as a device authorization request
({{Section 3.1 of RFC8628}}).

In a response to a party that has not authenticated as the client, an
authorization server SHOULD NOT indicate that a request was rejected
because of its source address. This specification does not yet define an
error response for such a rejection ({{open-issues}}).

# Implementation Considerations {#implementation}

## Enforcement Points {#enforcement-points}

Enforcing the IP range set of a specific client requires knowing which
client sent a request. A network enforcement point that operates below
HTTP, such as a firewall, observes the source address but not the
client, so it can enforce only the aggregate of the IP range sets of the
clients permitted to reach an endpoint. An authorization server can
enforce each client's IP range set once it has identified the client
({{as-enforcement}}).

For example, an enterprise administrator approves a hosted client for
use with the enterprise's authorization server and reviews the IP range
set in its metadata. The enterprise adds the approved ranges to the
network allowlist in front of its token endpoint, and the authorization
server rejects token requests from that client that originate outside
them. When the client's IP range set later changes, local policy
determines whether the change is applied automatically, such as an
addition within an approved scope, or is held for administrator approval
({{range-changes}}).

Requests from a client are rejected if its egress addresses change
before the authorization server accepts an updated declaration.
Deployments that require an accepted IP range set need an administrative
path to restore access, such as approving a replacement set. Enforcing a
client's IP range set at the client configuration endpoint
({{Section 2 of RFC7592}}) can prevent the client from correcting a
declaration made by value.

## Retrieval Scheduling {#retrieval-scheduling}

This specification does not require retrieval during the processing of
an OAuth request; an authorization server can retrieve documents in the
background. An authorization server typically schedules the next
retrieval when the stored response's freshness lifetime
({{Section 4.2 of RFC9111}}) ends, within local minimum and maximum
intervals. A client needs to allow for these intervals, and for approval
delays, when adding a prefix ({{acceptance}}).

To revalidate, an authorization server sends a conditional GET request
carrying the `If-None-Match` header field
({{Section 13.1.2 of RFC9110}}) with the entity tag of the most recent
complete response it stored from that URI ({{Section 4.3 of RFC9111}}).
That response can be newer than the one from which the accepted IP range
set was derived. Without a stored complete response, the authorization
server sends an unconditional request.

## Document Size {#document-size}

{{Section 8.7 of CIMD}} recommends that authorization servers read at
most 5 kilobytes of a Client ID Metadata Document. A client whose IP
range set does not fit within such a limit can use
`client_ip_ranges_uri`. An authorization server sets its bounds on
Client IP Range Documents ({{resource-exhaustion}}) independently of
bounds on client metadata, because such documents are expected to be
larger. The gzip content coding ({{Section 8.4.1.3 of RFC9110}}) can
reduce the transfer size of large documents.

# Security Considerations {#security}

The security considerations of the mechanism that conveys the client
metadata, such as {{Section 5 of RFC7591}} or {{Section 8 of CIMD}},
also apply.

## Trust in Declared Ranges {#trust}

Client IP range metadata is a declaration made by or on behalf of the
client. Inclusion of a prefix does not establish that the client
exclusively owns or controls the addresses it contains. Prefixes of
shared infrastructure, such as cloud provider or carrier-grade NAT
egress ranges, also carry traffic from other parties.

Whether to use declared ranges in network access control is a local
trust decision. A software statement ({{Section 2.3 of RFC7591}})
vouches only for the values it contains: for `client_ip_ranges`, the
prefixes; for `client_ip_ranges_uri`, the URL but not the referenced
document. The party hosting the document asserts its contents, and
accepting them automatically is a separate decision to trust that party
within an approved scope ({{range-changes}}). An authorization server
can require that `client_ip_ranges_uri` share an origin with the client
identifier URL of a Client ID Metadata Document {{CIMD}}.

## Changes to Declared Ranges {#range-changes}

A change to a client's IP range set can broaden or narrow the source
addresses admitted by a network policy derived from it. An authorization
server MUST NOT treat the addition of a prefix to a client's IP range
set as granting network access by itself. When local policy permits
automatic updates for the client and the change falls within an approved
scope, such as prefixes contained in supernets that the authorization
server has configured for the client, the authorization server MAY
accept the change automatically. Detecting unexpected expansion of a
client's IP range set, including the addition of overly broad prefixes,
helps an authorization server decide which changes to hold for approval.
The authorization server SHOULD apply changes atomically, so that no
intermediate state admits a source address that neither the previous nor
the new IP range set contains.

Continued use of an accepted IP range set that may no longer match the
client's current declaration admits addresses that may have been
reassigned to another party. An authorization server SHOULD set a
retention limit on how long it continues to use an accepted IP range set
while retrieval or validation of the client's current declaration fails,
after the client withdraws its declaration, and while a validated
replacement has not been accepted under local policy.

Before accepting a validated replacement, an authorization server MAY
narrow the accepted IP range set to its intersection with the
replacement, so that it admits only source addresses that both sets
match. The narrowed set remains the accepted IP range set; the
replacement is still not accepted.

When the retention limit is exceeded, the authorization server stops
using the set and has no accepted IP range set for the client. An
authorization server that was enforcing the set rejects requests from
the client until it accepts a new IP range set or an explicit local
policy decision removes the client from enforcement.

## Server-Side Request Forgery {#ssrf}

Retrieval of `client_ip_ranges_uri` causes an authorization server to
send a request to a URL supplied in client metadata, which creates a
risk of Server-Side Request Forgery (SSRF) attacks. When retrieving a
Client IP Range Document, the following requirements apply:

1. The authorization server MUST NOT retrieve a URL that does not meet
   the requirements for `client_ip_ranges_uri` in {{metadata}}.
2. The authorization server MUST verify the server certificate as
   described in {{Section 4.3.4 of RFC9110}} and MUST NOT accept a
   document retrieved over a connection that fails this verification.
   This strengthens the requirement of that section for automated
   clients.
3. The authorization server MUST NOT automatically follow HTTP redirects
   ({{Section 15.4 of RFC9110}}).
4. The authorization server MUST NOT connect to an address that the IANA
   special-purpose address registries ({{RFC6890}}, {{RFC8190}}) list as
   not globally reachable, unless local policy explicitly permits that
   destination. This check applies to the address of each connection
   after DNS resolution. An authorization server that retrieves through
   a configured forward proxy relies on the proxy to apply this check.

An authorization server retrieving a URL obtained from a Client ID
Metadata Document is also subject to {{Section 8.6 of CIMD}}.

## Resource Exhaustion {#resource-exhaustion}

A malicious or compromised party hosting a Client IP Range Document can
return an oversized document, a highly compressed response, or a large
number of overlapping prefixes. An authorization server MUST bound
retrieval time, response size, decompressed size, and the number of
prefixes in an IP range set, whether conveyed by value or in a document.
If a document exceeds any bound, the authorization server MUST reject
the entire document. This specification sets no minimum value for any of
these bounds.

## Source Address Interpretation {#source-address}

Proxies, load balancers, and network address translation change the
source address that an authorization server observes. The authorization
server MUST determine the source address from a trusted source of
connection-origin information. It MUST NOT use a forwarding header
field, such as `Forwarded` {{RFC7239}} or `X-Forwarded-For`, as the
source address unless it was set by an intermediary the authorization
server trusts. Because a trusted intermediary can append its own
observation to a field that already carries entries supplied by an
attacker ({{Section 8.1 of RFC7239}}), the authorization server MUST
select the source address from such a field using its configuration of
trusted proxies and MUST NOT trust an entry merely because a trusted
intermediary forwarded or appended to the field.

# Privacy Considerations {#privacy}

Client IP range metadata describes network infrastructure rather than
end users. However, when a client runs on infrastructure associated with
an individual, such as a residential network, its declared ranges can
reveal that individual's network location. A publicly retrievable
Client IP Range Document discloses the client's egress infrastructure to
any party.

Retrieval of `client_ip_ranges_uri` reveals to the party hosting the
document the addresses and retrieval timing of authorization servers
that evaluate the client, which can indicate the authorization servers
and resource servers with which the client is used (see also
{{Section 9.1 of CIMD}}).

# IANA Considerations {#iana}

This specification requests registration of the following client
metadata in the "OAuth Dynamic Client Registration Metadata" registry
established by {{RFC7591}}.

* Client Metadata Name: `client_ip_ranges`
* Client Metadata Description: Array of IPv4 and IPv6 prefixes from which
  the client originates requests
* Change Controller: IETF
* Specification Document(s): {{metadata}} of this specification
{: spacing="compact"}

* Client Metadata Name: `client_ip_ranges_uri`
* Client Metadata Description: HTTPS URL of a document containing the
  IPv4 and IPv6 prefixes from which the client originates requests
* Change Controller: IETF
* Specification Document(s): {{metadata}} of this
  specification
{: spacing="compact"}

--- back

# Examples {#examples}

## Dynamic Client Registration by Value

The following is a non-normative example of a client registration
request ({{Section 3.1 of RFC7591}}) that declares an IP range set by
value:

~~~ http-message
POST /register HTTP/1.1
Host: server.example.com
Content-Type: application/json

{
  "client_name": "Example Client",
  "grant_types": ["client_credentials"],
  "token_endpoint_auth_method": "private_key_jwt",
  "jwks_uri": "https://client.example.com/jwks.json",
  "client_ip_ranges": [
    "192.0.2.0/24",
    "2001:db8:100::/48"
  ]
}
~~~

## Client ID Metadata Document by Reference

The following is a non-normative example of a Client ID Metadata
Document {{CIMD}} that declares an IP range set by reference:

~~~ json
{
  "client_id": "https://client.example.com/oauth/client.json",
  "client_name": "Example Client",
  "redirect_uris": ["https://client.example.com/callback"],
  "token_endpoint_auth_method": "private_key_jwt",
  "jwks_uri": "https://client.example.com/jwks.json",
  "client_ip_ranges_uri": "https://client.example.com/ip-ranges.json"
}
~~~

## Document Retrieval

The following is a non-normative example of a request for the Client IP
Range Document referenced above:

~~~ http-message
GET /ip-ranges.json HTTP/1.1
Host: client.example.com
Accept: application/json
Accept-Encoding: gzip
~~~

The following is a non-normative example of a successful response:

~~~ http-message
HTTP/1.1 200 OK
Content-Type: application/json
Cache-Control: public, max-age=3600
ETag: "ranges-v42"

{
  "client_ip_ranges": [
    "192.0.2.0/24",
    "198.51.100.0/24",
    "2001:db8:100::/48"
  ]
}
~~~

The following non-normative example shows a later conditional request
and a response indicating that the document is unchanged:

~~~ http-message
GET /ip-ranges.json HTTP/1.1
Host: client.example.com
Accept: application/json
If-None-Match: "ranges-v42"
~~~

~~~ http-message
HTTP/1.1 304 Not Modified
Cache-Control: public, max-age=3600
ETag: "ranges-v42"
~~~

# Related Work {#related-work}

{{JAFAR}} defines a format for publishing the IP address ranges of
automated HTTP clients. Its prefixes are objects with separate IPv4 and
IPv6 members and optional service identifiers, the document carries a
creation time, and a prefix object that has both or neither address
member is ignored. This specification instead uses one prefix array for
both `client_ip_ranges` and the Client IP Range Document, so that both
forms share one syntax and one validation procedure; rejects the whole
set when any prefix is invalid ({{acceptance}}), so that a document is
applied completely or not at all; and relies on client metadata to
associate the set with a client, so that the document needs no service
identifiers.

Clients can announce planned changes to their IP range sets through
a separately specified notification mechanism, such as event types
delivered using the Shared Signals Framework {{SSF}}. This specification
does not define advance announcement events or scheduled activation of
ranges.

# Open Issues {#open-issues}

1. Which error response indicates that an authorization server rejected
   a request under a client's source address policy? The
   `invalid_client` error code signals failed client authentication
   ({{Section 5.2 of RFC6749}}) and could prompt credential rotation.

# Document History {#history}

-00

* Initial version.
