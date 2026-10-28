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
  RFC7591:
  RFC8259:
  RFC9110:
  RFC9111:
  RFC9911:

informative:
  RFC7239:
  RFC7592:
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
transcribe them into network access controls by hand and track changes
out of band.

Client metadata ({{Section 2 of RFC7591}}) conveys information that an
authorization server uses when interacting with a client, such as its
redirection URIs and public keys. It has no parameter for the network
origins of the client.

This specification defines two client metadata parameters for declaring
the IP address ranges from which a client originates requests:
`client_ip_ranges`, which carries the ranges by value, and
`client_ip_ranges_uri`, which references a Client IP Range Document
containing them. The pair follows the pattern of the `jwks` and
`jwks_uri` parameters of {{RFC7591}}: the two forms have the same
semantics and are mutually exclusive, and the reference form lets a
client publish a large or frequently changing set without updating its
registration. The parameters can be used with OAuth 2.0 Dynamic Client
Registration {{RFC7591}}, OAuth Client ID Metadata Documents {{CIMD}},
and other mechanisms that convey client metadata.

For example, an enterprise administrator approves a hosted client for
use with the enterprise's authorization server. The authorization server
obtains the client's IP range set from its metadata and the
administrator reviews it. The enterprise adds the approved ranges to the
network allowlist in front of its token endpoint, and the authorization
server rejects token requests from that client that originate outside
them ({{as-enforcement}}). When the client operator later publishes a
changed set, local policy determines whether the change is applied
automatically, such as an addition within an approved scope, or is held
for administrator approval ({{range-changes}}).

Declared ranges are an input to network access control. They do not
authenticate the client or authorize a request ({{use}}). Firewall
policy, network access authorization decisions, and the configuration of
network enforcement points are outside the scope of this specification.

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
: The IP range set that a consumer has most recently validated and
  accepted for a client.

publisher:
: An entity that publishes a client's IP range set, either by value in
  client metadata or in a Client IP Range Document. The publisher of a
  Client IP Range Document can differ from the entity that provides the
  client metadata referencing it.

consumer:
: An entity that processes client IP range metadata, such as an
  authorization server or a resource server.

# Client IP Range Metadata {#metadata}

This specification defines the following client metadata parameters.
Both convey the client's IP range set; they differ only in whether the
set is carried by value or by reference.

## `client_ip_ranges` {#client-ip-ranges}

OPTIONAL. JSON array of strings, each an IPv4 or IPv6 prefix as defined
in {{prefix-syntax}}. Each entry identifies a range of source addresses
from which the client originates requests (see {{use}}). The array MAY
contain both IPv4 and IPv6 prefixes.

The array is a set. Consumers MUST NOT assign meaning to the order of
entries or to duplicate entries. This specification does not limit the
number of entries.

An empty array declares an empty IP range set: the client declares no
source address from which it originates requests. Absence of the
parameter means the client makes no declaration; it does not imply an
empty IP range set.

## `client_ip_ranges_uri` {#client-ip-ranges-uri}

OPTIONAL. URL of a Client IP Range Document ({{document}}). The URL MUST
use the `https` scheme and MUST NOT contain a fragment or userinfo
component {{RFC3986}}. Consumers retrieve the document as described in
{{retrieval}}. The IP range set it contains has the same semantics as
the value of the `client_ip_ranges` parameter.

The `client_ip_ranges` and `client_ip_ranges_uri` parameters MUST NOT
both be present in the same client metadata.

Section 8.7 of {{CIMD}} recommends that authorization servers read at
most 5 kilobytes of a Client ID Metadata Document. A client whose IP
range set does not fit within such a limit uses `client_ip_ranges_uri`.

## Prefix Syntax {#prefix-syntax}

Each prefix MUST conform to the `ip-prefix` type defined in Section 4 of
{{RFC9911}}. Publishers MUST use the canonical format of that type: all
bits outside the prefix length set to zero and, for IPv6, the address
represented as defined in Section 4 of {{RFC5952}}.

Consumers MUST interpret each prefix by its network address and prefix
length, not by its textual form. A value that does not conform to the
`ip-prefix` type is invalid. Consumers MUST NOT treat a value as invalid
solely because it is not in canonical format. For example, a consumer
interprets `192.0.2.1/24` as `192.0.2.0/24` and `2001:DB8::1/64` as
`2001:db8::/64`.

A prefix within the IPv4-mapped IPv6 address range `::ffff:0:0/96`
({{Section 2.5.5.2 of RFC4291}}) is invalid; IPv4 ranges are expressed
as IPv4 prefixes. Before matching, consumers MUST convert an IPv4-mapped
IPv6 source address to the IPv4 address it contains.

Every prefix length that the `ip-prefix` type permits, including zero,
is valid. Whether to accept a broad prefix is local policy
({{range-changes}}).

## Metadata Validation {#validation}

When client metadata contains a trusted software statement
({{Section 2.3 of RFC7591}}) that carries `client_ip_ranges` or
`client_ip_ranges_uri`, the authorization server ignores both
parameters elsewhere in that client metadata, whether in a registration
request or in a Client ID Metadata Document (Section 4.3 of {{CIMD}}).
This extends the precedence rule of {{Section 2 of RFC7591}} to the pair
of parameters. The authorization server resolves this precedence before
applying the rules below. For example, if a Client ID Metadata Document
contains `client_ip_ranges_uri` and a trusted software statement within
it contains `client_ip_ranges`, the effective metadata is the software
statement's `client_ip_ranges`.

Client IP range metadata is invalid if any of the following is true of
the effective metadata:

1. Both `client_ip_ranges` and `client_ip_ranges_uri` are present.
2. The value of `client_ip_ranges` is not an array of strings, or any
   entry is an invalid prefix ({{prefix-syntax}}).
3. The value of `client_ip_ranges_uri` does not meet the requirements of
   {{client-ip-ranges-uri}}.

In a dynamic client registration request, the authorization server
responds to invalid client IP range metadata with HTTP status code 400
and the `invalid_client_metadata` error code
({{Section 3.2.2 of RFC7591}}). An authorization server that obtains
invalid client IP range metadata from a Client ID Metadata Document
{{CIMD}} handles it as a validation failure ({{retrieval-failures}}) and
does not treat the rest of the document as invalid.

An authorization server that does not support these parameters ignores
them ({{Section 2 of RFC7591}}). A client can determine whether the
authorization server registered them from the client information
response ({{Section 3.2.1 of RFC7591}}).

# Client IP Range Document {#document}

A Client IP Range Document is a JSON {{RFC8259}} object with the
following member:

`client_ip_ranges`:
: REQUIRED. JSON array of prefixes, with the syntax and semantics
  defined in {{client-ip-ranges}}.

Consumers MUST ignore members they do not understand. A document whose
top-level object contains duplicate member names is invalid, since
parsers can read such an object differently ({{Section 4 of RFC8259}}).

The document MUST be served with the `application/json` media type.

The document is self-contained. Consumers MUST NOT retrieve other
resources to complete it. All prefixes in the document apply equally to
the client identified by the referencing metadata; the document carries
no service, region, or network policy information. Multiple clients MAY
reference the same document.

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
successful response has status code 200 (OK) and contains the complete
document.

Publishers MUST NOT require authentication to retrieve a Client IP Range
Document unless the publisher and consumer have agreed on an
authentication mechanism, which is outside the scope of this
specification. Otherwise, consumers retrieve the document without
credentials. Consumers MUST NOT send credentials associated with an
OAuth request, such as client credentials, access tokens, or
authorization codes, when retrieving a Client IP Range Document.

The consumer MUST validate the response and the complete document,
including every prefix, before accepting it. If any part is invalid, the
consumer MUST reject the entire document. A failed retrieval, a
malformed response, or a document that is incomplete or exceeds the
consumer's bounds ({{resource-exhaustion}}) MUST NOT be treated as an
empty IP range set; it is handled as described in
{{retrieval-failures}}.

This specification does not require retrieval during the processing of
any OAuth request.

## Caching {#caching}

Publishers SHOULD support HTTP caching {{RFC9111}} and SHOULD include
the `Cache-Control` ({{Section 5.2 of RFC9111}}) and `ETag`
({{Section 8.8.3 of RFC9110}}) response header fields.

Consumers SHOULD revalidate a cached document with a conditional GET
request carrying the `If-None-Match` header field
({{Section 13.1.2 of RFC9110}}) and the entity tag of the document from
which the accepted IP range set was obtained. A 304 (Not Modified)
response ({{Section 15.4.5 of RFC9110}}) indicates that this document
remains current.

Cached responses and their validators are specific to the URI from which
they were retrieved ({{Section 2 of RFC9111}}). A consumer MUST NOT use
an entity tag obtained from one `client_ip_ranges_uri` in a request to
another.

HTTP caching applies only to retrieving the document. A consumer that
uses an HTTP cache follows {{RFC9111}}, including the `no-cache`,
`no-store`, and `must-revalidate` cache directives. The accepted IP
range set is application state, distinct from any HTTP cache
({{Section 6 of RFC9111}}). It stays in use until the consumer accepts a
replacement or local policy ends its use ({{retrieval-failures}}); cache
freshness does not determine when enforcement of a prefix starts or
stops. Consumers SHOULD retrieve the document periodically, within local
minimum and maximum intervals, to obtain changes.

Publishers and consumers SHOULD support the gzip content coding
({{Section 8.4.1.3 of RFC9110}}).

## Acceptance {#acceptance}

A consumer MUST validate an IP range set completely before accepting it,
whether the set is conveyed by `client_ip_ranges` or in a Client IP
Range Document. Acceptance is subject to local policy
({{range-changes}}).

An accepted IP range set replaces the previously accepted IP range set
in full. This specification defines no incremental update format.
Consumers can compare the two sets to identify added and removed
prefixes. The accepted IP range set remains in effect until the consumer
accepts a replacement or stops using it ({{retrieval-failures}}). A
validated IP range set that awaits approval does not replace it.

Publishers SHOULD publish a new prefix before the client originates
requests from it, allowing for consumer retrieval intervals and approval
delays, and SHOULD continue to publish a prefix until the client no
longer originates requests from it.

Publishers can use a separately specified notification mechanism to
announce planned changes to their IP range sets. Such a mechanism could
define event types delivered using the Shared Signals Framework {{SSF}}.
This specification does not define advance announcement events or
scheduled activation of ranges.

## Changes to Client Metadata {#metadata-changes}

A client's metadata can change the value of `client_ip_ranges_uri`,
switch between `client_ip_ranges` and `client_ip_ranges_uri`, or remove
both. The new metadata replaces the previous declaration.

When the metadata changes representation or URI, the accepted IP range
set remains in effect until the consumer accepts a set under the new
metadata ({{acceptance}}). If the new declaration is invalid or the new
document cannot be retrieved, the consumer handles this as a failure
({{retrieval-failures}}). The consumer does not retrieve the previous
URI again for that client or reuse its cached responses or validators
({{caching}}).

Removal of both parameters withdraws the client's declaration. In a
client update request ({{Section 2.2 of RFC7592}}), omitted metadata
fields are treated as a request to delete them, so an update that omits
both parameters withdraws the declaration.
Withdrawal MUST NOT by itself cause a consumer to admit a source address
that it did not admit before the withdrawal. Whether to relax
enforcement after a withdrawal is local policy.

## Retrieval Failures {#retrieval-failures}

A retrieval or validation failure MUST NOT cause a consumer to admit a
source address that it did not admit before the failure.

The consumer MAY continue to use the accepted IP range set, subject to
local freshness and security policy. How long an accepted IP range set
remains usable, and what applies when no IP range set has been accepted
for the client, is local policy.

# Use of Client IP Range Metadata {#use}

The IP range set identifies source addresses from which the client
originates requests directly to authorization servers and resource
servers, such as token requests, other back-channel requests, and
requests to protected resources. It does not describe the network
origins of user agents, redirection endpoints, or requests initiated by
the resource owner.

Consumers MAY use the IP range set as an input to local network access
control. Matching a source address against the IP range set of a
specific client requires knowing which client sent the request:

* A network enforcement point that operates below HTTP, such as a
  firewall, observes the source address but not the client. It can
  enforce only an aggregate of the IP range sets of the clients
  permitted to reach an endpoint.
* An authorization server can match the source address against the
  client's IP range set once it has identified the client
  ({{as-enforcement}}).
* A resource server can match the source address against the IP range
  set of the client to which an access token was issued, if it can
  obtain that client's metadata. How a resource server obtains
  a client's metadata is outside the scope of this specification and
  could be addressed by a companion specification.

An empty IP range set declares that the client originates no direct
requests. A consumer that enforces it for a client admits no source
address for requests from that client.

The IP range set is a declaration by its publisher. A source address
match does not authenticate the client or authorize a request. Consumers
MUST NOT use a source address match in place of client authentication,
access token validation, or any other security control required by
OAuth.

## Enforcement by Authorization Servers {#as-enforcement}

An authorization server can enforce a client's IP range set at the token
endpoint and at other endpoints that receive direct requests from the
client. Enforcement requires identifying the client: through client
authentication or, for a public client ({{Section 2.1 of RFC6749}}),
through the client to which the grant presented in the request was
issued.

An authorization server SHOULD NOT indicate, in a response to a party
that has not authenticated as the client, that a request was rejected
because of its source address. This specification does not yet define
the error response for this case ({{open-issues}}).

# Security Considerations {#security}

## Trust in Declared Ranges {#trust}

Client IP range metadata is a declaration by its publisher. Inclusion of
a prefix does not establish that the client exclusively owns or controls
the addresses it contains. Prefixes of shared infrastructure, such as
cloud provider or carrier-grade NAT egress ranges, also carry traffic
from other parties.

Whether to use declared ranges in network access control is a local
trust decision. A software statement ({{Section 2.3 of RFC7591}})
vouches for the values it contains: for `client_ip_ranges`, the
prefixes; for `client_ip_ranges_uri`, only the URL. The publisher of the
referenced document asserts its contents. Trust in the statement's
issuer therefore does not extend to those contents; accepting them
automatically is a separate decision to trust the document's publisher
within an approved scope ({{range-changes}}). A consumer MAY require
that `client_ip_ranges_uri` share an origin with the client identifier
URL of a Client ID Metadata Document {{CIMD}}.

## Changes to Declared Ranges {#range-changes}

A change to a published IP range set can broaden or narrow the source
addresses admitted by a network policy derived from it. Consumers MUST
NOT treat publication of a new prefix as granting network access by
itself. Consumers MAY accept updates automatically when local policy
authorizes the publisher and the change falls within an approved scope,
such as prefixes contained in supernets that the consumer has configured
for the client.

Consumers SHOULD detect unexpected expansion of a declared IP range set,
including the addition of overly broad IPv4 or IPv6 prefixes. Consumers
SHOULD apply changes atomically, so that no intermediate state admits a
source address that neither the previous nor the new IP range set
contains.

Continued use of an accepted IP range set after the publisher has
removed a prefix admits addresses that may have been reassigned to
another party.

## Server-Side Request Forgery {#ssrf}

Retrieval of `client_ip_ranges_uri` causes a consumer to send a request
to a URL chosen by the publisher, which is a server-side request forgery
vector. When retrieving a Client IP
Range Document, consumers:

1. MUST NOT retrieve a URL that does not meet the requirements of
   {{client-ip-ranges-uri}}.
2. MUST verify the server certificate as described in
   {{Section 4.3.4 of RFC9110}}.
3. MUST NOT automatically follow HTTP redirects
   ({{Section 15.4 of RFC9110}}).
4. MUST NOT connect to an address that the IANA special-purpose address
   registries {{RFC6890}} list as not globally reachable, unless local
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
and prefix count. If a document exceeds any bound, the consumer MUST
reject the entire document and MUST NOT accept part of it.

These bounds are set independently of any bound on client metadata,
such as the limit recommended in Section 8.7 of {{CIMD}}, because a
Client IP Range Document is expected to be larger.

## Source Address Interpretation {#source-address}

Proxies, load balancers, and network address translation change the
source address that a consumer observes. Consumers MUST determine the
source address from a trusted source of connection-origin information.
Consumers MUST NOT use forwarding header fields, such as `Forwarded`
{{RFC7239}} or `X-Forwarded-For`, as the source address unless they
were set by an intermediary the consumer trusts.

A trusted intermediary can append its own observation to a forwarding
header field that already carries entries supplied by an attacker
({{Section 8.1 of RFC7239}}). Consumers MUST select the source address
from such a field using their configuration of trusted proxies, and MUST
NOT trust an entry merely because a trusted intermediary forwarded or
appended to the field.

# Privacy Considerations {#privacy}

Client IP range metadata describes network infrastructure rather than
end users. When a client runs on infrastructure associated with an
individual, such as a residential network, its declared ranges can
reveal that individual's network location. A publicly retrievable
Client IP Range Document discloses the client's egress infrastructure
to any party.

Retrieval of `client_ip_ranges_uri` reveals to the publisher the
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

# Open Issues {#open-issues}

1. Which error response indicates that an authorization server rejected
   a request under a client's source address policy? The
   `invalid_client` error code signals failed client authentication
   ({{Section 5.2 of RFC6749}}) and could prompt credential rotation.
2. Should every consumer support a minimum number of prefixes, so that
   independently chosen limits do not prevent interoperability?
3. Should authorization server metadata advertise support for these
   parameters, and what would that signal? Recognizing the metadata,
   retrieving Client IP Range Documents, and enforcing IP range sets are
   distinct capabilities.

# Document History {#history}

-00

* Initial version.
