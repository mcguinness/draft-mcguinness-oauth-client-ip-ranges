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

normative:
  RFC3986:
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
client metadata. Authorization servers and resource servers can use the
declared ranges as an input to network access control.

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
obtains the client's IP range set from its metadata, the administrator
reviews it, and the enterprise configures its network enforcement points
to admit requests from the approved ranges. When the client operator
later publishes a changed set, local policy determines whether the
change is applied automatically, such as an addition within an approved
scope, or is held for administrator approval ({{range-changes}}).

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
endpoint" defined by OAuth 2.0 {{RFC6749}}, and the terms "client metadata" and "software
statement" defined by OAuth 2.0 Dynamic Client Registration {{RFC7591}}.

This specification defines the following terms:

IP range set:
: The set of IPv4 and IPv6 prefixes that a client declares, either by
  value with `client_ip_ranges` or by reference with
  `client_ip_ranges_uri`.

Client IP Range Document:
: A JSON document, defined in {{document}}, that contains an IP range
  set and is referenced by `client_ip_ranges_uri`.

publisher:
: The entity that publishes a client's metadata or Client IP Range
  Document, typically the client operator.

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

## Metadata Validation {#validation}

Client IP range metadata is invalid if any of the following is true:

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
{{CIMD}} treats the document as invalid.

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

Consumers MUST ignore members they do not understand.

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

# Retrieval and Synchronization {#retrieval}

## Retrieval {#fetch}

A consumer retrieves a Client IP Range Document by sending an HTTP GET
request to the `client_ip_ranges_uri` URL, subject to {{ssrf}}. A
successful response has status code 200 (OK) and contains the complete
document.

Unless a deployment profile specifies otherwise, publishers MUST NOT
require authentication to retrieve a Client IP Range Document, and
consumers retrieve it without credentials. Authenticated retrieval,
including the provisioning of credentials, is outside the scope of this
specification. Consumers MUST NOT send credentials associated with an
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
({{Section 13.1.2 of RFC9110}}) and the entity tag of the last accepted
document. A 304 (Not Modified) response ({{Section 15.4.5 of RFC9110}})
indicates that the last accepted document remains current.

Cached responses and their validators are specific to the URI from which
they were retrieved ({{Section 2 of RFC9111}}). A consumer MUST NOT use
an entity tag obtained from one `client_ip_ranges_uri` in a request to
another.

HTTP caching governs retrieval only. A consumer that uses an HTTP cache
follows {{RFC9111}}, including the `no-cache`, `no-store`, and
`must-revalidate` cache directives. The previously accepted IP range set
is application state, distinct from any HTTP cache
({{Section 6 of RFC9111}}). How long a consumer continues to use it is
local policy ({{retrieval-failures}}); a freshness lifetime such as
`max-age` is not a deadline for adding a prefix to, or removing a prefix
from, enforcement. Consumers MAY schedule retrievals within local
minimum and maximum intervals.

Publishers and consumers SHOULD support the gzip content coding
({{Section 8.4.1.3 of RFC9110}}).

## Updates {#updates}

Each accepted document replaces the previously accepted IP range set in
full. This specification defines no incremental update format.

A consumer MUST validate a new document completely before replacing the
previously accepted IP range set. Consumers can compare the two sets to
identify added and removed prefixes. Activation of changes is subject to
local policy ({{range-changes}}).

## Changes to Client Metadata {#metadata-changes}

A client's metadata can change the value of `client_ip_ranges_uri`,
switch between `client_ip_ranges` and `client_ip_ranges_uri`, or remove
both. The new metadata replaces the previous declaration.

When the representation or URI changes, the set accepted under the
previous metadata remains the previously accepted IP range set until the
consumer accepts a set under the new metadata. If the new document
cannot be retrieved or validated, the consumer handles this as a
retrieval failure ({{retrieval-failures}}). A document retrieved from
the previous URI is no longer a declaration for the client, and cached
responses and validators from it do not apply to the new URI
({{caching}}).

Removal of both parameters withdraws the client's declaration. Consumers
MUST NOT treat withdrawal as permitting requests from any source address
or as authorizing removal of a network restriction configured under
local policy.

## Retrieval Failures {#retrieval-failures}

If retrieval or validation fails, the consumer MUST NOT treat the
failure as permitting requests from any source address.

The consumer MAY continue to use the previously accepted IP range set,
subject to local freshness and security policy. How long a previously
accepted set remains usable, and what applies when none is available,
is a matter of local policy.

# Use of Client IP Range Metadata {#use}

The IP range set identifies source addresses from which the client
originates requests directly to authorization servers and resource
servers, such as token requests, other back-channel requests, and
requests to protected resources. It does not describe the network
origins of user agents, redirection endpoints, or requests initiated by
the resource owner.

Consumers MAY use the IP range set as an input to local network access
control. A consumer that enforces an empty IP range set permits no
source address for requests originated by the client, subject to local
policy.

The IP range set is a declaration by its publisher. A source address
match does not authenticate the client or authorize a request. Consumers
MUST NOT use a source address match in place of client authentication,
access token validation, or any other security control required by
OAuth.

How a resource server obtains a client's metadata is outside the scope
of this specification.

# Security Considerations {#security}

## Trust in Declared Ranges {#trust}

Client IP range metadata is a declaration by its publisher. Inclusion of
a prefix does not establish that the client exclusively owns or controls
the addresses it contains. Prefixes of shared infrastructure, such as
cloud provider or carrier-grade NAT egress ranges, also carry traffic
from other parties.

Whether to use declared ranges in network access control is a local
trust decision. A software statement ({{Section 2.3 of RFC7591}})
protects the claims it carries, so a `client_ip_ranges` value in a
trusted software statement is asserted by the statement's issuer. A
`client_ip_ranges_uri` value in a software statement is asserted only as
a reference: the signature covers the URL, not the current or future
contents of the document it identifies. Trust in the statement's issuer
does not extend to ranges later published at that URL. Accepting them
automatically requires a separate local decision to trust the
document's publisher within an approved scope ({{range-changes}}). A
consumer MAY require that `client_ip_ranges_uri` share an
origin with the client identifier URL of a Client ID Metadata Document
{{CIMD}}.

## Changes to Declared Ranges {#range-changes}

A change to a published IP range set can broaden or narrow the source
addresses admitted by a network policy derived from it. Consumers MUST
NOT treat publication of a new prefix as granting network access by
itself. Consumers MAY accept updates automatically when local policy
authorizes the publisher and the permitted scope of change.

Consumers SHOULD detect unexpected expansion of a declared IP range set,
including the addition of overly broad IPv4 or IPv6 prefixes. Consumers
SHOULD apply changes atomically, so that no intermediate state admits a
source address that neither the previous nor the new IP range set
contains.

Continued use of a previously accepted IP range set after the publisher
has removed a prefix admits addresses that may have been reassigned to
another party.

## Server-Side Request Forgery {#ssrf}

Retrieval of `client_ip_ranges_uri` causes a consumer to send a request
to a URL chosen by the publisher, which is a server-side request forgery
vector (see also Section 8.6 of {{CIMD}}). When retrieving a Client IP
Range Document, consumers:

1. MUST NOT retrieve a URL that does not meet the requirements of
   {{client-ip-ranges-uri}}.
2. MUST verify the server certificate as described in
   {{Section 4.3.4 of RFC9110}}.
3. MUST NOT automatically follow HTTP redirects
   ({{Section 15.4 of RFC9110}}).
4. MUST NOT connect to a special-use IP address as defined in
   {{RFC6890}}, unless local policy explicitly permits that destination.
   This check applies to the address of each connection after DNS
   resolution. The local policy exception does not apply to a
   `client_ip_ranges_uri` obtained from a Client ID Metadata Document.

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

1. Whether a dedicated media type should replace `application/json` for
   the Client IP Range Document.
2. Whether authorization server metadata should advertise support for
   these parameters, and which client behavior that would enable.
   Recognition, retrieval, and enforcement are distinct capabilities.

# Document History {#history}

-00

* Initial version.
