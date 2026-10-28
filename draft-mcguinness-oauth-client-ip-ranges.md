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

* `client_ip_ranges` carries the ranges by value.
* `client_ip_ranges_uri` references a Client IP Range Document that
  contains them.

As with the `jwks` and `jwks_uri` parameters of {{RFC7591}}, the two
forms have the same semantics and are mutually exclusive. The reference
form lets a client publish a large or frequently changing set without
updating its registration. The parameters can be used with OAuth 2.0
Dynamic Client Registration {{RFC7591}}, OAuth Client ID Metadata
Documents {{CIMD}}, and other mechanisms that convey client metadata.

Declared ranges are an input to network access control. Enforcing them
confines the use of a client's credentials and grants to the networks
the client declares, which limits the value of stolen credentials.
Declared ranges do not authenticate the client or authorize a request
({{use}}).

This specification defines the metadata and the processing requirements
for consumers that use it. Selecting the clients subject to enforcement,
approving declared ranges, and configuring network enforcement points
are local policy; {{implementation}} describes typical deployments.
Enforcement by resource servers is outside the scope of this
specification.

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
: The IP range set, if any, that a consumer has validated and accepted
  for a client and currently uses for it. A consumer that stops using
  an IP range set no longer has an accepted IP range set for the client.

publisher:
: An entity that publishes a client's IP range set, either by value in
  client metadata or in a Client IP Range Document. The publisher of a
  Client IP Range Document can differ from the entity that provides the
  client metadata referencing it.

consumer:
: An entity that processes client IP range metadata, such as an
  authorization server or a resource server.

# Client IP Range Metadata {#metadata}

The following client metadata parameters convey a client's IP range
set, by value or by reference.

## `client_ip_ranges` {#client-ip-ranges}

OPTIONAL. JSON array of strings, each an IPv4 or IPv6 prefix
({{prefix-syntax}}) identifying source addresses from which the client
originates requests ({{use}}).

The array is a set. Consumers MUST NOT assign meaning to the order of
entries or to duplicate entries. This specification sets no maximum
number of entries; consumers apply their own bounds
({{resource-exhaustion}}).

An empty array declares an empty IP range set. Absence of both
`client_ip_ranges` and `client_ip_ranges_uri` means the client makes no
declaration; it does not imply an empty IP range set.

## `client_ip_ranges_uri` {#client-ip-ranges-uri}

OPTIONAL. URL of a Client IP Range Document ({{document}}), whose IP
range set has the same semantics as a `client_ip_ranges` value. The URL
MUST use the `https` scheme and MUST NOT contain a fragment or userinfo
component {{RFC3986}}. Consumers retrieve the document as described in
{{retrieval}}.

The `client_ip_ranges` and `client_ip_ranges_uri` parameters MUST NOT
both be present in the same client metadata.

## Prefix Syntax {#prefix-syntax}

Each prefix MUST conform to the `ip-prefix` type defined in Section 4 of
{{RFC9911}}, that is, match in full every pattern of either the
`ipv4-prefix` or the `ipv6-prefix` type. A value that does not conform
to the `ip-prefix` type is invalid. In addition:

* Publishers MUST use the canonical format of that type: all bits
  outside the prefix length set to zero and, for IPv6, the address
  represented as defined in Section 4 of {{RFC5952}}.
* A value that is not in canonical format is valid and denotes the
  prefix given by its network address and prefix length (Section 4 of
  {{RFC9911}}). For example, `192.0.2.1/24` denotes `192.0.2.0/24`, and
  `2001:DB8::1/64` denotes `2001:db8::/64`.
* A prefix within the IPv4-mapped IPv6 address range `::ffff:0:0/96`
  ({{Section 2.5.5.2 of RFC4291}}) is invalid. IPv4 ranges are
  expressed as IPv4 prefixes.
* Every prefix length that the `ip-prefix` type permits, including
  zero, is valid. Whether to accept a broad prefix is local policy
  ({{range-changes}}).

## Metadata Validation {#validation}

If a trusted software statement ({{Section 2.3 of RFC7591}}) carries
`client_ip_ranges` or `client_ip_ranges_uri`, the authorization server
ignores both parameters elsewhere in the client metadata, whether in a
registration request or in a Client ID Metadata Document (Section 4.3 of
{{CIMD}}). This extends the precedence rule of {{Section 2 of RFC7591}}
to the pair of parameters.

After this precedence is applied, client IP range metadata is invalid if
any of the following is true:

1. Both `client_ip_ranges` and `client_ip_ranges_uri` are present.
2. The value of `client_ip_ranges` is not an array of strings, or any
   entry is an invalid prefix ({{prefix-syntax}}).
3. The value of `client_ip_ranges_uri` does not meet the requirements of
   {{client-ip-ranges-uri}}.
4. The value of `client_ip_ranges` has more entries than the
   authorization server's bound on prefixes ({{resource-exhaustion}}).

An authorization server handles invalid client IP range metadata as
follows:

* In a dynamic client registration request, the authorization server
  responds with HTTP status code 400 and the `invalid_client_metadata`
  error code ({{Section 3.2.2 of RFC7591}}), or the
  `invalid_software_statement` error code if the invalid value comes
  from a trusted software statement.
* In a Client ID Metadata Document {{CIMD}}, the authorization server
  handles the metadata as a validation failure ({{retrieval-failures}})
  and does not treat the rest of the document as invalid.

An authorization server that does not support these parameters ignores
them ({{Section 2 of RFC7591}}). The client information response
({{Section 3.2.1 of RFC7591}}) shows whether the authorization server
registered them.

# Client IP Range Document {#document}

A Client IP Range Document is a JSON {{RFC8259}} object with the
following member:

`client_ip_ranges`:
: REQUIRED. JSON array of prefixes, with the syntax and semantics
  defined in {{client-ip-ranges}}.

The following rules apply to the document:

* The document MUST be served with the `application/json` media type.
* Consumers MUST ignore members they do not understand.
* A document whose top-level object contains duplicate member names is
  invalid. Section 4 of {{RFC8259}} notes that receivers handle such
  objects unpredictably.
* The document is self-contained. Consumers MUST NOT retrieve other
  resources to complete it.

All prefixes in the document apply equally to each client whose
metadata references it. Multiple clients can reference the same
document.

The following is an example of a Client IP Range Document:

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

A consumer retrieves a Client IP Range Document by sending an HTTP GET
request to the `client_ip_ranges_uri` URL, subject to {{ssrf}}. A
response is successful if it either:

* has status code 200 (OK), the `application/json` media type, and a
  complete body within the consumer's bounds ({{resource-exhaustion}});
  or
* has status code 304 (Not Modified) ({{Section 15.4.5 of RFC9110}}) in
  response to a conditional request ({{retrieval-scheduling}}), which
  indicates that
  the retained response is current.

Publishers MUST NOT require authentication to retrieve a Client IP Range
Document unless the publisher and consumer have agreed on an
authentication mechanism, which is outside the scope of this
specification. Consumers MUST NOT send credentials associated with an
OAuth request, such as client credentials, access tokens, or
authorization codes, when retrieving a Client IP Range Document.

A failed retrieval, an unsuccessful response, or an invalid document
MUST NOT be treated as an empty IP range set; it is a failure
({{retrieval-failures}}).

## Caching {#caching}

Publishers SHOULD support HTTP caching {{RFC9111}} and SHOULD include
the `Cache-Control` ({{Section 5.2 of RFC9111}}) and `ETag`
({{Section 8.8.3 of RFC9110}}) response header fields.


The accepted IP range set is application state, distinct from any HTTP
cache ({{Section 6 of RFC9111}}):

* A fresh response does not make its IP range set accepted.
* Expiry of a cached response does not end use of the accepted IP range
  set ({{acceptance}}).

Consumers SHOULD retrieve the document periodically to obtain changes.

## Acceptance {#acceptance}

A consumer MUST validate an IP range set completely, including every
prefix, before accepting it, whether the set is conveyed by
`client_ip_ranges` or in a Client IP Range Document, and MUST reject the
entire set if any part of it is invalid. For a set obtained from a
Client IP Range Document, the consumer MUST also validate the response
({{fetch}}) and the complete document ({{document}}) before accepting
the set, and
MUST reject the entire document if either is invalid. Acceptance is also
subject to local policy ({{range-changes}}). Registering client IP range
metadata does not by itself accept its IP range set.

An accepted IP range set replaces the previously accepted IP range set
in full; this specification defines no incremental update format. The
accepted IP range set remains in effect until the consumer accepts a
replacement or stops using it ({{range-changes}}). A validated IP range
set that has not yet been accepted under local policy does not replace
it.

Publishers SHOULD publish a new prefix before the client originates
requests from it and SHOULD continue to publish a prefix until the
client no longer originates requests from it.

## Changes to Client Metadata {#metadata-changes}

A client's metadata can change the value of `client_ip_ranges_uri`,
switch between `client_ip_ranges` and `client_ip_ranges_uri`, or remove
both. The new metadata replaces the previous declaration.

When the metadata changes representation or URI:

* The accepted IP range set remains in effect until the consumer accepts
  a set under the new metadata or stops using it ({{acceptance}}). An
  invalid new declaration or an unretrievable new document is a failure
  ({{retrieval-failures}}).
* If the previous metadata contained `client_ip_ranges_uri`, the
  consumer does not retrieve that URI again for the client.

Removal of both parameters withdraws the client's declaration
({{retrieval-failures}}). In a client update request
({{Section 2.2 of RFC7592}}), omitting both parameters requests their
removal, which the authorization server can ignore. The client
information response ({{Section 3 of RFC7592}}) shows whether they were
removed.

## Failures and Withdrawal {#retrieval-failures}

The following MUST NOT cause a consumer to admit a source address that
it did not admit before:

* a failure to retrieve a Client IP Range Document;
* a failure to validate client IP range metadata or a retrieved
  document;
* withdrawal of the client's declaration ({{metadata-changes}}), by
  itself; and
* ceasing to use an accepted IP range set when the limit in
  {{range-changes}} is exceeded.

After a failure, the consumer MAY continue to use the accepted IP range
set, within the limit described in {{range-changes}}. Whether to relax
enforcement after a withdrawal is local policy.

A consumer can require an accepted IP range set for a client. Such a
consumer MUST reject requests from the client while it has no accepted
IP range set for the client, whether because:

* the client has made no declaration;
* its declaration is invalid or cannot be retrieved; or
* a validated IP range set has not yet been accepted under local
  policy.

For other consumers, what applies while a client has no accepted IP
range set is local policy.

## Lifecycle Summary {#lifecycle}

The following table summarizes the accepted IP range set after each
event, as specified in {{acceptance}}, {{retrieval-failures}}, and
{{range-changes}}. It adds no requirements. Whenever a client has no
accepted IP range set, a consumer that requires one rejects requests
from the client.

| Event | Accepted IP range set afterward |
|---|---|
| First declaration validated | None, until accepted under local policy |
| Replacement accepted | The replacement |
| Replacement validated, not yet accepted | Unchanged or narrowed to the intersection, until the limit |
| Retrieval or validation fails | Unchanged, until the limit |
| Declaration withdrawn | Unchanged, until the limit |
| Limit exceeded | None; no source address is newly admitted |

# Use of Client IP Range Metadata {#use}

The IP range set identifies source addresses from which the client
originates requests directly to authorization servers and resource
servers, such as token requests and requests to protected resources. It
does not describe the network origins of user agents, redirection
endpoints, or requests initiated by the resource owner.

A source address matches an IP range set if it is within any of its
prefixes. Before matching, consumers MUST convert an IPv4-mapped IPv6
source address to the IPv4 address it contains. A consumer that enforces
a client's accepted IP range set admits requests from the client only
from source addresses that match it; if that set is empty, it admits
none.

Consumers MAY use the IP range set as an input to local network access
control. The IP range set is a declaration by its publisher, and a match
does not authenticate the client or authorize a request. Consumers MUST
NOT use a source address match in place of client authentication, access
token validation, or any other security control required by OAuth.

Enforcement by resource servers, including how a resource server
determines the client associated with a request and obtains that
client's metadata, is outside the scope of this specification and could
be addressed by a companion specification.

## Enforcement by Authorization Servers {#as-enforcement}

An authorization server can enforce a client's accepted IP range set at
the token endpoint and at other endpoints that receive direct requests
from the client, once it has identified the client:

* through client authentication; or
* for a public client ({{Section 2.1 of RFC6749}}), through the client
  to which a grant presented in the request was issued.

This specification does not define how an authorization server
identifies a public client in a request that presents no grant, such
as a device authorization request ({{Section 3.1 of RFC8628}}).

An authorization server SHOULD NOT indicate, in a response to a party
that has not authenticated as the client, that a request was rejected
because of its source address. This specification does not yet define
an error response for such a rejection ({{open-issues}}).

# Implementation Considerations {#implementation}

## Enforcement Points {#enforcement-points}

Enforcing the IP range set of a specific client requires knowing which
client sent a request:

* A network enforcement point that operates below HTTP, such as a
  firewall, observes the source address but not the client. It can
  enforce only the aggregate of the IP range sets of the clients
  permitted to reach an endpoint.
* An authorization server can enforce each client's IP range set once
  it has identified the client ({{as-enforcement}}).

For example, an enterprise administrator approves a hosted client for
use with the enterprise's authorization server and reviews the IP range
set in its metadata. The enterprise adds the approved ranges to the
network allowlist in front of its token endpoint, and the authorization
server rejects token requests from that client that originate outside
them. When the publisher later changes the set, local
policy determines whether the change is applied automatically, such as
an addition within an approved scope, or is held for administrator
approval ({{range-changes}}).

A client can be locked out if its egress addresses change before the
consumer accepts an updated declaration. Deployments that require an
accepted IP range set need an administrative path to restore access,
such as approving a replacement set. Enforcing a client's IP range set
at the client configuration endpoint ({{Section 2 of RFC7592}}) can
prevent the client from correcting an inline declaration.

## Retrieval Scheduling {#retrieval-scheduling}

This specification does not require retrieval during the processing of
an OAuth request; consumers can retrieve documents in the background.
Consumers typically schedule the next retrieval when the retained
response's freshness lifetime ({{Section 4.2 of RFC9111}}) ends, within
local minimum and maximum intervals. Publishers allow for these
intervals, and for approval delays, when publishing a new prefix
({{acceptance}}).

To revalidate, a consumer sends a conditional GET request carrying the
`If-None-Match` header field ({{Section 13.1.2 of RFC9110}}) with the
entity tag of the most recent complete response it retained from that
URI ({{Section 4.3 of RFC9111}}). That response can be newer than the
one from which the accepted IP range set was obtained. Without a
retained complete response, the consumer sends an unconditional
request.

## Document Size {#document-size}

Section 8.7 of {{CIMD}} recommends that authorization servers read at
most 5 kilobytes of a Client ID Metadata Document. A client whose IP
range set does not fit within such a limit uses `client_ip_ranges_uri`.
Consumers set their bounds on Client IP Range Documents
({{resource-exhaustion}}) independently of bounds on client metadata,
because such documents are expected to be larger. Publishers and
consumers can use the gzip content coding
({{Section 8.4.1.3 of RFC9110}}) to reduce the transfer size of large
documents.

# Security Considerations {#security}

The security considerations of the mechanism that conveys the client
metadata, such as {{Section 5 of RFC7591}} or Section 8 of {{CIMD}},
also apply.

## Trust in Declared Ranges {#trust}

Client IP range metadata is a declaration by its publisher. Inclusion of
a prefix does not establish that the client exclusively owns or controls
the addresses it contains. Prefixes of shared infrastructure, such as
cloud provider or carrier-grade NAT egress ranges, also carry traffic
from other parties.

Whether to use declared ranges in network access control is a local
trust decision. A software statement ({{Section 2.3 of RFC7591}})
vouches only for the values it contains:

* For `client_ip_ranges`, it vouches for the prefixes.
* For `client_ip_ranges_uri`, it vouches for the URL, not the referenced
  document. The document's publisher asserts its contents, and accepting
  them automatically is a separate decision to trust that publisher
  within an approved scope ({{range-changes}}).

A consumer can require that `client_ip_ranges_uri` share an origin with
the client identifier URL of a Client ID Metadata Document {{CIMD}}.

## Changes to Declared Ranges {#range-changes}

A change to a published IP range set can broaden or narrow the source
addresses admitted by a network policy derived from it. When processing
such changes:

* Consumers MUST NOT treat publication of a new prefix as granting
  network access by itself.
* Consumers MAY accept updates automatically when local policy
  authorizes the publisher and the change falls within an approved
  scope, such as prefixes contained in supernets that the consumer has
  configured for the client.
* Consumers SHOULD apply changes atomically, so that no intermediate
  state admits a source address that neither the previous nor the new
  IP range set contains.

Detecting unexpected expansion of a declared IP range set, including
the addition of overly broad prefixes, helps a consumer decide which
changes to hold for approval.

Continued use of an accepted IP range set that no longer matches the
client's current declaration admits addresses that may have been
reassigned to another party. Consumers SHOULD limit how long they
continue to use such a set, including:

* while retrieval or validation of the client's current declaration
  fails;
* after the client withdraws its declaration; and
* while a validated replacement has not been accepted under local
  policy.

Before accepting a validated replacement, a consumer MAY narrow the
accepted IP range set to its intersection with the replacement, so that
it admits only source addresses that both sets match. When the limit is
exceeded, the consumer stops using the set and has no accepted IP range
set for the client ({{retrieval-failures}}). The limit trades
availability for safety: once it is exceeded, requests from the client
are rejected until the consumer accepts a new IP range set.

## Server-Side Request Forgery {#ssrf}

Retrieval of `client_ip_ranges_uri` causes a consumer to send a request
to a URL supplied in client metadata, which is a server-side request
forgery vector. When retrieving a Client IP Range Document, consumers:

1. MUST NOT retrieve a URL that does not meet the requirements of
   {{client-ip-ranges-uri}}.
2. MUST verify the server certificate as described in
   {{Section 4.3.4 of RFC9110}} and MUST NOT accept a document retrieved
   over a connection that fails this verification, strengthening the
   requirement of that section for automated clients.
3. MUST NOT automatically follow HTTP redirects
   ({{Section 15.4 of RFC9110}}).
4. MUST NOT connect to an address that the IANA special-purpose address
   registries ({{RFC6890}}, {{RFC8190}}) list as not globally reachable,
   unless local
   policy explicitly permits that destination. This check applies to the
   address of each connection after DNS resolution. A consumer that
   retrieves through a configured forward proxy relies on the proxy to
   apply this check.

An authorization server retrieving a URL obtained from a Client ID
Metadata Document is also subject to Section 8.6 of {{CIMD}}.

## Resource Exhaustion {#resource-exhaustion}

A malicious or compromised publisher can return an oversized document, a
highly compressed response, or a large number of overlapping prefixes.
Consumers MUST bound retrieval time, response size, decompressed size,
and the number of prefixes in an IP range set, whether conveyed by value
or in a document. If a document exceeds any bound, the consumer MUST
reject the entire document. This specification sets no minimum value for
any of these bounds.

## Source Address Interpretation {#source-address}

Proxies, load balancers, and network address translation change the
source address that a consumer observes. Consumers MUST determine the
source address from a trusted source of connection-origin information.
For forwarding header fields, such as `Forwarded` {{RFC7239}} or
`X-Forwarded-For`:

* Consumers MUST NOT use such a field as the source address unless it
  was set by an intermediary the consumer trusts.
* A trusted intermediary can append its own observation to a field that
  already carries entries supplied by an attacker
  ({{Section 8.1 of RFC7239}}). Consumers MUST select the source address
  from such a field using their configuration of trusted proxies, and
  MUST NOT trust an entry merely because a trusted intermediary
  forwarded or appended to the field.

# Privacy Considerations {#privacy}

Client IP range metadata describes network infrastructure rather than
end users, but it can still disclose information:

* When a client runs on infrastructure associated with an individual,
  such as a residential network, its declared ranges can reveal that
  individual's network location.
* A publicly retrievable Client IP Range Document discloses the client's
  egress infrastructure to any party.
* Retrieval of `client_ip_ranges_uri` reveals to the publisher the
  addresses and retrieval timing of consumers that evaluate the client,
  which can indicate the authorization servers and resource servers with
  which the client is used (see also Section 9.1 of {{CIMD}}).

# IANA Considerations {#iana}

This specification requests registration of the following client
metadata in the "OAuth Dynamic Client Registration Metadata" registry
established by {{RFC7591}}.

* Client Metadata Name: `client_ip_ranges`
* Client Metadata Description: Array of IPv4 and IPv6 prefixes from which
  the client originates requests
* Change Controller: IETF
* Specification Document(s): {{client-ip-ranges}} of this specification
{: spacing="compact"}

* Client Metadata Name: `client_ip_ranges_uri`
* Client Metadata Description: HTTPS URL of a document containing the
  IPv4 and IPv6 prefixes from which the client originates requests
* Change Controller: IETF
* Specification Document(s): {{client-ip-ranges-uri}} of this
  specification
{: spacing="compact"}

# Examples {#examples}

## Dynamic Client Registration by Value

The following is an example of a client registration request
({{Section 3.1 of RFC7591}}) that declares an IP range set by value:

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

The following is an example of a Client ID Metadata Document {{CIMD}}
that declares an IP range set by reference:

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

The following is an example of a request for the Client IP Range
Document referenced above:

~~~ http-message
GET /ip-ranges.json HTTP/1.1
Host: client.example.com
Accept: application/json
Accept-Encoding: gzip
~~~

The following is an example of a successful response:

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

The following example shows a later conditional request and a response
indicating that the document is unchanged:

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

--- back

# Related Work {#related-work}

{{JAFAR}} defines a format for publishing the IP address ranges of
automated HTTP clients. Its prefixes are objects with separate IPv4 and
IPv6 members and optional service identifiers, the document carries a
creation time, and consumers ignore a prefix object that has both or
neither address member. This specification uses a different model:

* The Client IP Range Document uses the same prefix array as
  `client_ip_ranges`, so both forms share one syntax and one validation
  procedure.
* An invalid prefix invalidates the whole set ({{acceptance}}), so a
  partly invalid document does not change what a consumer admits.
* Client metadata associates the set with a client, so the document
  needs no service identifiers.

Publishers can announce planned changes to their IP range sets through
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
