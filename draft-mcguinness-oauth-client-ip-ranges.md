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
  RFC6454:
  RFC7239:
  RFC7592:
  RFC8628:
  RFC8705:
  RFC9449:
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
an HTTPS resource containing the set, so that large and frequently
changing sets can use standard HTTP retrieval and caching. The
specification also defines how authorization servers validate, accept,
and retain declared ranges.
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
transcribe them into network access controls by hand. Clients that use
OAuth Client ID Metadata Documents {{CIMD}} can reach authorization
servers without prior registration, leaving no opportunity to exchange
ranges by hand in advance. Client metadata ({{Section 2 of RFC7591}})
has no parameter for the network origins of a client.

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
authorize a request ({{use}}). They complement client authentication and
sender-constrained tokens ({{RFC9449}}, {{RFC8705}}), which bind
credentials and tokens to keys rather than to network locations.

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
"resource owner", "resource server", and "token
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

client IP range metadata:
: The `client_ip_ranges` and `client_ip_ranges_uri` client metadata
  parameters.

accepted IP range set:
: The IP range set, if any, that an authorization server has validated,
  accepted, and currently uses for a client, possibly narrowed
  ({{acceptance}}).

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

The `client_ip_ranges` and
`client_ip_ranges_uri` parameters MUST NOT both be present in the same
client metadata. If neither is present, the client makes no declaration,
which differs from declaring an empty IP range set. This specification
sets no maximum number of entries; the authorization server applies its
own bounds ({{resource-exhaustion}}).

## Prefix Syntax {#prefix-syntax}

Each prefix MUST conform to the `ip-prefix` type defined in
{{Section 4 of RFC9911}}, that is, match in full every pattern of either
the `ipv4-prefix` or the `ipv6-prefix` type. A value that does not
conform is invalid.

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
3. The value of `client_ip_ranges_uri` does not meet the requirements of
   {{metadata}}.
4. The value of `client_ip_ranges` has more entries than the
   authorization server's bound on prefixes ({{resource-exhaustion}}).

If a client registration request, or a client update request
({{Section 2.2 of RFC7592}}), contains invalid client IP range
metadata, the authorization server responds with an HTTP 400 (Bad
Request) error response as described in {{Section 3.2.2 of RFC7591}},
using `invalid_client_metadata` as the error code value, or
`invalid_software_statement` if the invalid value comes from a trusted
software statement. The authorization server MUST NOT substitute a
default value for invalid client IP range metadata. If a Client ID
Metadata Document {{CIMD}} contains
invalid client IP range metadata, the authorization server handles this
as a validation failure ({{failures}}) without treating the rest of the
document as invalid.

An authorization server that does not support these parameters ignores
them ({{Section 2 of RFC7591}}). The client information response
({{Section 3.2.1 of RFC7591}}) shows whether the authorization server
registered them.

# Client IP Range Document {#document}

A Client IP Range Document is a JSON {{RFC8259}} object with the
following member:

`client_ip_ranges`:
: REQUIRED. JSON array of prefixes, with the same syntax and semantics
  as the `client_ip_ranges` client metadata parameter ({{metadata}}).

The authorization server MUST ignore members that it does not
understand. A document whose top-level object contains duplicate member
names is invalid; {{Section 4 of RFC8259}} notes that receivers handle
such objects unpredictably. The document is self-contained, and the
authorization server MUST NOT retrieve other resources to complete it.

Multiple clients can reference the same document; all its prefixes apply
equally to each such client. The document can be hosted by a party other
than the client, such as its hosting provider.

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
to a conditional request ({{Section 13 of RFC9110}}). Following a 304
response, the authorization server MUST apply the same response and
document validation requirements to the corresponding stored response,
with its header fields updated as described in
{{Section 4.3.4 of RFC9111}}.

The party hosting a Client IP Range Document MUST serve it with the
`application/json` media type and MUST NOT require authentication to
retrieve it unless that party and the authorization server have agreed
on an authentication mechanism, which is outside the scope of this
specification. The authorization server MUST NOT send credentials
associated with an OAuth request, such as client credentials, access
tokens, or authorization codes, when retrieving the document.

The authorization server MUST NOT treat a failed retrieval, an
unsuccessful response, or an invalid document as an empty IP range set;
each is a failure ({{failures}}).

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

Before accepting an IP range set, the authorization server MUST ensure
the following:

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
accepts a replacement, narrows it, or stops using it ({{retention}}).

While a validated replacement awaits acceptance, an authorization server
MAY
narrow the accepted IP range set to its intersection with the
replacement, so that it admits only source addresses that both sets
match. The narrowed set remains the accepted IP range set; the
replacement is still not accepted.

A client SHOULD add a prefix to its IP range set before it originates
requests from addresses in that prefix and SHOULD keep the prefix in the
set until it no longer originates requests from them.

## Changes to Client Metadata {#metadata-changes}

A client's metadata can change the value of `client_ip_ranges_uri`,
switch between `client_ip_ranges` and `client_ip_ranges_uri`, or remove
both. The new metadata replaces the previous declaration.

When the metadata changes representation or URL, the accepted IP range
set remains in effect as described in {{acceptance}}. An invalid new
declaration in a Client ID Metadata Document, or an unretrievable or
invalid new document, is a failure ({{failures}}). If the previous
metadata contained
`client_ip_ranges_uri`, the authorization server MUST NOT retrieve that
URL again for the client.

Removal of both parameters withdraws the client's declaration
({{withdrawal}}). In a client update request
({{Section 2.2 of RFC7592}}), omitting both parameters requests their
removal, which the authorization server can ignore. An empty
`client_ip_ranges` array in an update request declares an empty IP
range set; it is not a request to delete the parameter. The client
information response ({{Section 3 of RFC7592}}) shows whether the
parameters were removed.

## Retention {#continued-use}

### After a Failure {#failures}

A failure to retrieve a Client IP Range Document, or to validate client
IP range metadata or a retrieved document, MUST NOT cause an
authorization server to admit a source address that it did not admit
before the failure. After a failure, the authorization server MAY
continue to use the accepted IP range set, within its retention limit,
if it sets one ({{retention}}).

### After a Withdrawal {#withdrawal}

Withdrawal of the client's declaration ({{metadata-changes}}) MUST NOT
by itself cause an authorization server to admit a source address that
it did not admit before the withdrawal. Enforcement is relaxed after a
withdrawal only by an explicit local policy decision.

### Retention Limit {#retention}

An authorization server SHOULD set a retention limit on how long it
continues to use an accepted IP range set in any of the following cases
({{range-changes}}):

* while retrieval or validation of the client's current declaration
  fails;
* after the client withdraws its declaration; or
* while a validated replacement has not been accepted under local
  policy.

The limit runs from the first failed retrieval or validation, from the
withdrawal, or from the first validation of the replacement, and ends
when the authorization server accepts a set under the client's current
declaration. The third case does not apply to an accepted IP range set
that the authorization server has narrowed to its intersection with
the replacement ({{acceptance}}), because every source address that set
admits is also in the replacement.

When the retention limit is exceeded, the authorization server stops
using the set and has no accepted IP range set for the client
({{use}}).

# Enforcement {#use}

The IP range set identifies source addresses from which the client
originates requests directly to authorization servers and resource
servers, such as token requests and requests to protected resources. It
does not describe the network origins of user agents or of requests
initiated by the resource owner.

A source address matches an IP range set if it is within any of its
prefixes. Before matching, the authorization server MUST convert an
IPv4-mapped IPv6 source address to the IPv4 address it contains.

An authorization server that enforces a client's accepted IP range set
admits requests from the client only from source addresses that match
it; if that set is empty, it admits none. While a client has no
accepted IP range set:

* an authorization server that requires an accepted IP range set for
  the client MUST reject requests from the client;
* after the retention limit is exceeded ({{retention}}), an
  authorization server that was enforcing a set for the client MUST
  reject requests from the client until it accepts a new IP range set
  or an explicit local policy decision removes the client from
  enforcement; and
* in other cases, what applies is local policy.

An authorization server MAY use the IP range set as an input to local
network access control. A match does not authenticate the client or
authorize a request. The authorization server MUST NOT use a source
address match in place of client authentication, access token
validation, or any other security control required by OAuth.

Enforcement by resource servers, including how a resource server
determines the client associated with a request and obtains that
client's metadata, is outside the scope of this specification and could
be addressed by a companion specification.

## Client Identification and Error Responses {#as-enforcement}

An authorization server can enforce a client's accepted IP range set at
the token endpoint and at other endpoints that receive direct requests
from the client, once it has identified the client, either through
client authentication or, for a public client
({{Section 2.1 of RFC6749}}), as the client to which the authorization
code or refresh token presented in the request was issued. This
specification does not define enforcement for a public client in a
request that identifies the client only by an unauthenticated
`client_id`, such as a device authorization request
({{Section 3.1 of RFC8628}}). A request rejected because of its source
address does not count as a use of the authorization code it presents
({{Section 4.1.2 of RFC6749}}).

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
enforce each client's accepted IP range set once it has identified the
client
({{as-enforcement}}).

For example, an enterprise adds a hosted client's approved ranges to the
network allowlist in front of its token endpoint, and the authorization
server also enforces the client's accepted IP range set. Local policy
determines whether later changes are accepted automatically or require
administrator approval ({{range-changes}}).

Deployments that require an accepted IP range set need an administrative
recovery path for a client whose source addresses move outside that set,
such as approving a replacement set. Enforcement at the client
configuration endpoint ({{Section 2 of RFC7592}}) can prevent the client
from updating a declaration made by value.

## Retrieval Scheduling {#retrieval-scheduling}

An authorization server can retrieve documents in the background rather
than while processing an OAuth request, and typically schedules the next
retrieval when the stored response's freshness lifetime
({{Section 4.2 of RFC9111}}) ends, within local minimum and maximum
intervals. A client needs to allow for these intervals, and for approval
delays, when adding a prefix ({{acceptance}}).

To revalidate with `If-None-Match` ({{Section 13.1.2 of RFC9110}}), an
authorization server uses the entity tag of the most recent complete
response it stored from that URL ({{Section 4.3 of RFC9111}}), which can
be newer than the one from which the accepted IP range set was derived.

## Document Size {#document-size}

{{Section 8.7 of CIMD}} recommends that authorization servers read at
most 5 kilobytes of a Client ID Metadata Document. A client whose IP
range set does not fit within such a limit can use
`client_ip_ranges_uri`. An authorization server sets its bounds on
Client IP Range Documents ({{resource-exhaustion}}) independently of
bounds on client metadata. The gzip content coding
({{Section 8.4.1.3 of RFC9110}}) can reduce the transfer size of large
documents.

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
within an approved scope ({{range-changes}}). A malicious or compromised
party hosting a document can also deny service to every client that
references it, by publishing an empty or unrelated IP range set or by
keeping the document unavailable beyond the retention limit.

A client cannot observe whether an authorization server enforces its
declaration, so declaring an IP range set does not by itself protect
the client's credentials.

An authorization server can require that `client_ip_ranges_uri` share an
origin ({{RFC6454}}) with the client identifier URL of a Client ID
Metadata Document {{CIMD}}.

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

An accepted IP range set that no longer matches the client's current
declaration can include addresses that have since been reassigned to
another party. The retention limit ({{retention}}) and
narrowing ({{acceptance}}) limit how long such addresses remain
admitted.

## Rejection Responses {#rejection-responses}

A rejection that a requesting party can attribute to its source address
tells that party that its other inputs were acceptable. If the
authorization server checks the source address only after client
authentication succeeds, such a rejection confirms to a party holding
stolen or guessed client credentials that they are valid.
{{as-enforcement}} limits this disclosure for parties that have not
authenticated as the client; the error response for authenticated parties is
an open issue ({{open-issues}}).

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
Metadata Document is also subject to {{Section 8.6 of CIMD}}; the local
policy exception in item 4 does not override that section.

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
connection-origin information. A source address that the authorization
server cannot determine in this way matches no IP range set. It MUST NOT
use a forwarding header
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
Client IP Range Document, or a publicly retrievable Client ID Metadata
Document that carries `client_ip_ranges`, discloses the client's egress
infrastructure to any party.

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
member is ignored. This specification instead reuses the
`client_ip_ranges` array, rejects the whole set when any prefix is
invalid ({{acceptance}}), and relies on client metadata to associate the
set with a client.

Clients can announce planned changes to their IP range sets through
a separately specified notification mechanism, such as event types
delivered using the Shared Signals Framework {{SSF}}. This specification
does not define advance announcement events or scheduled activation of
ranges.

# Open Issues {#open-issues}

1. Which error response indicates that an authorization server rejected
   a request under a client's source address policy? The
   `invalid_client` error code signals failed client authentication
   ({{Section 5.2 of RFC6749}}) and could prompt credential rotation,
   while an error that identifies the source address confirms to a
   party holding stolen credentials that they are valid
   ({{rejection-responses}}).

# Document History {#history}

-00

* Initial version.
