# Expect-Staple Extension for HTTP
abbrev: "Expect-Staple"

docname: draft-helme-expect-staple.md

author:

name: Scott Helme

org: scotthelme.co.uk

email: mail@scotthelme.co.uk

# Abstract

This document defines a new HTTP header, named Expect-Staple, that allows web host
operators to instruct user agents to expect valid Online Certificate Status Protocol 
(OCSP) responses to be served on connections to these hosts. When configured, UAs 
will report the lack of valid OCSP responses to a URI configured by the host, but will 
allow the connection. By turning on Expect-Staple, web host operators can discover 
misconfigurations or unreliability in their OCSP Stapling deployments and ensure 
that they are resolved prior to deploying OCSP Must-Staple certificates.

# Introduction

This document defines a new HTTP header that enables UAs to identify web hosts
that expect the presence of stapled Online Certificate Status Protocol {{!RFC6960}} 
responses in future Transport Layer Security (TLS) {{!RFC5246}} connections.

Web hosts that serve the Expect-Staple HTTP header are noted by the UA as Known
Expect-Staple Hosts. The UA evaluates each connection to a Known Expect-Staple Host for
a valid, stapled OCSP response. If the host does not present a valid, stapled OCSP response, 
the UA sends a report to a URI configured by the Expect-Staple Host.

Expect-Staple provides value by allowing hosts to detect problems with their OCSP Stapling
implementation prior to commiting to OCSP Must-Staple certificates and adverseley affecting 
their availability.

## Requirements Language

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD",
"SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this
document are to be interpreted as described in RFC 2119 {{!RFC2119}}.

## Terminology

Terminology is defined in this section.

Expect-Staple Host
  : See HTTP Expect-Staple Host.

HTTP Expect-Staple
  : is the overall name for the combined UA- and server-side security policy
  defined by this specification.

HTTP Expect-Staple Host
  : is a conformant host implementing the HTTP server aspects of HTTP
  Expect-Staple. This means that an Expect-Staple Host returns the "Expect-Staple"
  HTTP response header field in its HTTP response messages sent over secure
  transport.

Known Expect-Staple Host
  : is an Expect-Staple Host that the UA has noted as such. See
  {{noting-expect-staple}} for particulars.

UA
  : is an acronym for "user agent". For the purposes of this specification, a UA
  is an HTTP client application typically actively manipulated by a user
  {{!RFC2616}}.

Unknown Expect-Staple Host
  : is an Expect-Staple Host that the UA has not noted.

# Server and Client Behavior

## Response Header Field Syntax

The "Expect-Staple" header field is a new response header defined in this
specification. It is used by a server to indicate that UAs should evaluate
connections to the host emitting the header for valid, stapled OCSP responses
({{expect-staple-compliance}}).

{{expect-staple-syntax}} describes the syntax (Augmented Backus-Naur Form) of the header field,
using the grammar defined in RFC 5234 {{!RFC5234}} and the rules defined in
Section 3.2 of RFC 7230 {{!RFC7230}}.

~~~
Expect-Staple-Directives = directive *( OWS ";" OWS directive )
directive                = directive-name [ "=" directive-value ]
directive-name           = token
directive-value          = token / quoted-string
~~~
{: #expect-staple-syntax title="Syntax of the Expect-Staple header field"}

Optional white space (`OWS`) is used as defined in Section 3.2.3 of RFC 7230
{{!RFC7230}}. `token` and `quoted-string` are used as defined in Section 3.2.6
of RFC 7230 {{!RFC7230}}.

The directives defined in this specification are described below. The overall
requirements for directives are:

1. The order of appearance of directives is not significant.

2. A given directive MUST NOT appear more than once in a given header
   field. Directives are either optional or required, as stipulated in their
   definitions.

3.  Directive names are case insensitive.

4.  UAs MUST ignore any header fields containing directives, or other header
    field value data, that do not conform to the syntax defined in this
    specification.  In particular, UAs must not attempt to fix malformed header
    fields.

5.  If a header field contains any directive(s) the UA does not recognize, the
    UA MUST ignore those directives.

6.  If the Expect-Staple header field otherwise satisfies the above requirements (1
    through 5), the UA MUST process the directives it recognizes.

### The report-uri Directive

The OPTIONAL `report-uri` directive indicates the URI to which the UA SHOULD
report Expect-Staple failures ({{expect-staple-compliance}}). The UA POSTs the reports
to the given URI as described in {{reporting-expect-staple-failure}}.

The `report-uri` directive is REQUIRED to have a directive value, for which the
syntax is defined in {{reporturi-syntax}}.

~~~
report-uri-value = absolute-URI
~~~
{: #reporturi-syntax title="Syntax of the report-uri directive value"}

`absolute-URI` is defined in Section 4.3 of RFC 3986 {{!RFC3986}}.

Hosts may set `report-uri`s that use HTTP or HTTPS. If the scheme in the
`report-uri` is one that uses TLS (e.g., HTTPS), UAs MUST check Expect-Staple
compliance when the host in the `report-uri` is a Known Expect-Staple Host;
similarly, UAs MUST apply HSTS if the host in the `report-uri` is a Known HSTS
Host.

Note that the report-uri need not necessarily be in the same Internet
domain or web origin as the host being reported about.

UAs SHOULD make their best effort to report Expect-Staple failures to the
`report-uri`, but they may fail to report in exceptional conditions.  For
example, if connecting the `report-uri` itself incurs an Expect-Staple failure or
other certificate validation failure, the UA MUST cancel the connection.
Similarly, if Expect-Staple Host A sets a `report-uri` referring to Expect-Staple Host
B, and if B sets a `report-uri` referring to A, and if both hosts fail to comply
to the UA's Expect-Staple Policy, the UA SHOULD detect and break the loop by failing to
send reports to and about those hosts.

UAs SHOULD limit the rate at which they send reports. For example, it is
unnecessary to send the same report to the same `report-uri` more than once.

### The max-age Directive

The `max-age` directive specifies the number of seconds after the reception of
the Expect-Staple header field during which the UA SHOULD regard the host from whom
the message was received as a Known Expect-Staple Host.

The `max-age` directive is REQUIRED to be present within an "Expect-Staple" header
field. The `max-age` directive is REQUIRED to have a directive value, for which
the syntax (after quoted-string unescaping, if necessary) is defined in
{{maxage-syntax}}.

~~~
max-age-value = delta-seconds
delta-seconds = 1*DIGIT
~~~
{: #maxage-syntax title="Syntax of the max-age directive value"}

`delta-seconds` is used as defined in Section 1.2.1 of RFC 7234 {{!RFC7234}}.

### The preload Directive

The optional `preload` directive indicates whether the host gives consent to be included in
client-side preload lists similar to the HSTS Preload List.

The `preload` directive does not have a directive value.

### The includeSubDomains Directive

The optional `includeSubDomains` directive signals to the UA that the Expect-Staple Policy
applies to this Expect-Staple Host as well as any subdomains of the host's domain name.

The `includeSubDomains` directive does not have a directive value.

### Examples

Some example Expect-Staple response header fields.

    Expect-Staple: max-age=0; report-uri https://example.com/report/ct
    Expect-Staple: max-age=3600; report-uri https://example.com/report/ct
    Expect-Staple: max-age=31536000; report-uri https://example.com/report/ct; includSubDomains
    Expect-Staple: max-age=31536000; report-uri https://example.com/report/ct; includSubDomains; preload

## Server Processing Model

This section describes the processing model that Expect-Staple Hosts implement.  The
model has 2 parts: (1) the processing rules for HTTP request messages received
over a secure transport (e.g., authenticated, non-anonymous TLS); and (2) the
processing rules for HTTP request messages received over non-secure transports,
such as TCP.

### HTTP-over-Secure-Transport Request Type

When replying to an HTTP request that was conveyed over a secure transport, an
Expect-Staple Host SHOULD include in its response exactly one Expect-Staple header
field. The header field MUST satisfy the grammar specified in
{{response-header-field-syntax}}.

Establishing a given host as an Expect-Staple Host, in the context of a given UA,
is accomplished as follows:

1.  Over the HTTP protocol running over secure transport, by correctly returning
    (per this specification) at least one valid Expect-Staple header field to the
    UA.

2.  Through other mechanisms, such as a client-side preloaded Expect-Staple Host
    list.

### HTTP Request Type

Expect-Staple Hosts SHOULD NOT include the Expect-Staple header field in HTTP responses
conveyed over non-secure transport.  UAs MUST ignore any Expect-Staple header
received in an HTTP response conveyed over non-secure transport.

## User Agent Processing Model

The UA processing model relies on parsing domain names. Note that
internationalized domain names SHALL be canonicalized according to
the scheme in Section 10 of {{!RFC6797}}.

### Expect-Staple Header Field Processing

If the UA receives, over a secure transport, an HTTP response that includes an
Expect-Staple header field conforming to the grammar specified in
{{response-header-field-syntax}}, the UA MUST evaluate the connection on which
the header was received for compliance with the UA's Expect-Staple Policy, and then
process the Expect-Staple header field as follows.

If the connection complies with the UA's Expect-Staple Policy (i.e. the connection
included a valid, stapled OCSP response), then the UA MUST either:

- Note the host as a Known Expect-Staple Host if it is not already so noted (see
  {{noting-expect-staple}}), or
- Update the UA's cached information for the Known Expect-Staple Host if the
  `max-age`, or `report-uri` header field value directives convey
  information different from that already maintained by the UA. If the `max-age`
  directive has a value of 0, the UA MUST remove its cached Expect-Staple
  information if the host was previously noted as a Known Expect-Staple Host, and
  MUST NOT note this host as a Known Expect-Staple Host if it is not already noted.

If the connection does not comply with the UA's Expect-Staple Policy (i.e. the 
connection did not include a valid, stapled OCSP response), then the UA MUST NOT 
note this host as a Known Expect-Staple Host.

If the header field includes a `report-uri` directive, and the connection does
not comply with the UA's Expect-Staple Policy (i.e. the connection did not include a 
valid, stapled OCSP response), and the UA has not already sent an Expect-Staple 
report for this connection, then the UA SHOULD send a report to the specified
`report-uri` as specified in {{reporting-expect-staple-failure}}.

If a UA receives more than one Expect-Staple header field in an HTTP response
message over secure transport, then the UA MUST process only the first Expect-Staple
header field.

The UA MUST ignore any Expect-Staple header field not conforming to the grammar
specified in {{response-header-field-syntax}}.

### Noting an Expect-Staple Host - Storage Model

The "effective Expect-Staple date" of a Known Expect-Staple Host is the time that the UA
observed a valid Expect-Staple header for the host. The "effective expiration date"
of a Known Expect-Staple Host is the effective Expect-Staple date plus the max-age. An
Expect-Staple Host is "expired" if the effective expiration date refers to a date in
the past. The UA MUST ignore any expired Expect-Staple Hosts in its cache.

Known Expect-Staple Hosts are identified only by domain names, and never IP
addresses. If the substring matching the host production from the Request-URI
(of the message to which the host responded) syntactically matches the
IP-literal or IPv4address productions from Section 3.2.2 of {{!RFC3986}}, then
the UA MUST NOT note this host as a Known Expect-Staple Host.

Otherwise, if the substring does not congruently match an existing Known
Expect-Staple Host's domain name, per the matching procedure specified in Section
8.2 of {{!RFC6797}}, then the UA MUST add this host to the Known Expect-Staple Host
cache. The UA caches:

- the Expect-Staple Host's domain name,
- the effective expiration date, or enough information to calculate it (the
  effective Expect-Staple date and the value of the `max-age` directive),
- the value of the `report-uri` directive, if present.

If any other metadata from optional or future Expect-Staple header directives are
present in the Expect-Staple header, and the UA understands them, the UA MAY note
them as well.

UAs MAY set an upper limit on the value of max-age, so that UAs that have noted
an erroneous Expect-Staple Host (whether by accident or due to attack) have some
chance of recovering over time. If the server sets a max-age greater than the
UA's upper limit, the UA MAY behave as if the server set the max-age to the UA's
upper limit. For example, if the UA caps max-age at 5,184,000 seconds (60
days), and an Expect-Staple Host sets a max-age directive of 90 days in its Expect-Staple
header, the UA MAY behave as if the max-age were effectively 60 days. (One way
to achieve this behavior is for the UA to simply store a value of 60 days
instead of the 90-day value provided by the Expect-Staple Host.)

### HTTP-Equiv \<meta\> Element Attribute

UAs MUST NOT heed `http-equiv="Expect-Staple"` attribute settings on `<meta>`
elements {{!W3C.REC-html401-19991224}} in received content.

## Noting Expect-Staple

Upon receipt of the Expect-Staple response header field, the UA notes the host as a
Known Expect-Staple Host, storing the host's domain name and its associated
Expect-Staple directives in non-volatile storage. The domain name and associated
Expect-Staple directives are collectively known as "Expect-Staple metadata".

The UA MUST note a host as a Known Expect-Staple Host if and only if it received the
Expect-Staple response header field over an error-free TLS connection, including the
validation added in {{expect-staple-compliance}}.

To note a host as a Known Expect-Staple Host, the UA MUST set its Expect-Staple metadata
given in the most recently received valid Expect-Staple header.

For forward compatibility, the UA MUST ignore any unrecognized Expect-Staple header
directives, while still processing those directives it does
recognize. {{response-header-field-syntax}} specifies the directives `max-age` and 
`report-uri` but future specifications and implementations might use additional directives.

## Evaluating Expect-Staple Connections for OCSP Stapling Compliance {#expect-staple-compliance}

When a UA connects to a Known Expect-Staple Host using a TLS connection, if the TLS
connection has errors, the UA MUST terminate the connection without allowing the
user to proceed anyway. (This behavior is the same as that required by
{{!RFC6797}}.)

If the connection has no errors, then the UA will apply an additional
correctness check: compliance with an OCSP Stapling Policy. A UA should evaluate
compliance with its OCSP Stapling Policy whenever connecting to a Known Expect-Staple
Host, as soon as possible. It is acceptable to skip this OCSP Stapling Policy check 
for some hosts according to local policy. For example, a UA may disable OCSP Stapling 
Policy compliance checks for hosts whose validated certificate chain terminates at a 
user-defined trust anchor, rather than a trust anchor built-in to the UA (or 
underlying platform).

If a connection to a Known Expect-Staple Host violates the UA's OCSP Stapling Policy,
and if the Known Expect-Staple Host's Expect-Staple metadata includes a `report-uri`, 
the UA SHOULD send an Expect-Staple report to that `report-uri` ({{reporting-expect-staple-failure}}).

A UA that has previously noted a host as a Known Expect-Staple Host MUST evaluate OCSP 
Stapling Policy compliance when setting up the TLS session, before beginning an HTTP
conversation over the TLS channel.

If the UA does not evaluate OCSP Stapling Policy compliance, e.g. because the user has elected to
disable it, or because a presented certificate chain chains up to a user-defined
trust anchor, UAs SHOULD NOT send Expect-Staple reports.

# Reporting Expect-Staple Failure

When the UA attempts to connect to a Known Expect-Staple Host and the connection is
not OCSP-Staple-Qualified, the UA SHOULD report Expect-Staple failures to the `report-uri`,
if any, in the Known Expect-Staple Host's Expect-Staple metadata.

When the UA receives an Expect-Staple response header field over a connection that
is not OCSP-Staple-Qualified, if the UA has not already sent an Expect-Staple report for this
connection, then the UA SHOULD report Expect-Staple failures to the configured
`report-uri`, if any.

## Generating a violation report

To generate a violation report object, the UA constructs a JSON message of the
following form:

~~~
{
  "date-time": date-time,
  "hostname": hostname,
  "port": port,
  "effective-expiration-date": date-time,
  "response-status": ResponseStatus,
  "ocsp-response": ocsp,
  "cert-status": CertStatus,
  "served-certificate-chain": [pem1, ... pemN],(MUST be in the order served)
  "validated-certificate-chain": [pem1, ... pemN](MUST be in the order served)
}

enum ResponseStatus {
"MISSING",
"PROVIDED",
"ERROR_RESPONSE",
"BAD_PRODUCED_AT",
"NO_MATCHING_RESPONSE",
"INVALID_DATE",
"PARSE_RESPONSE_ERROR",
"PARSE_RESPONSE_DATA_ERROR"
};

enum CertStatus {
"GOOD",
"REVOKED",
"UNKNOWN"
};

~~~
{: #violation-report-object title="JSON format of a violation report object"}

Whitespace outside of quoted strings is not significant. The key/value pairs
may appear in any order, but each MUST appear only once.

The `date-time` indicates the time the UA observed the OCSP Stapling Policy compliance failure.
It is provided as a string formatted according to Section 5.6, "Internet
Date/Time Format", of RFC 3339 {{!RFC3339}}.

The `hostname` is the hostname to which the UA made the original request that
failed the OCSP Stapling Policy compliance check. It is provided as a string.

The `port` is the port to which the UA made the original request that failed the
OCSP Stapling Policy compliance check. It is provided as an integer.

The `effective-expiration-date` is the Effective Expiration Date for the
Expect-Staple Host that failed the OCSP Stapling Policy compliance check.
It is provided as a string formatted according to Section 5.6, 
"Internet Date/Time Format", of RFC 3339 {{!RFC3339}}.

The `response-status` provides information about the stapled OCSP response that 
the UA received, whether one was was received or not, for the Expect-Staple Host.

The `ocsp-response` is the stapled OCSP response that the UA received, if one was
was received, for the Expect-Staple Host. The format of `ocsp` is a PEM encoded string.
Only present when `response-status` is not MISSING.

The `cert-status` indicates the status of the certificate indicated in the stapled OCSP
response. Only present when `response-status` is PROVIDED.

The `served-certificate-chain` is the certificate chain, as served by the
Expect-Staple Host during TLS session setup.  It is provided as an array of strings,
which MUST appear in the order that the certificates were served; each string
`pem1`, ... `pemN` is the Privacy-Enhanced Mail (PEM) representation of each
X.509 certificate as described in RFC 7468 {{!RFC7468}}.

The `validated-certificate-chain` is the certificate chain, as constructed by
the UA during certificate chain verification. (This may differ from the
`served-certificate-chain`.) It is provided as an array of strings, which MUST
appear in the order matching the chain that the UA validated; each string
`pem1`, ... `pemN` is the Privacy-Enhanced Mail (PEM) representation of each
X.509 certificate as described in RFC 7468 {{!RFC7468}}.

## Sending a violation report

When an Expect-Staple header field contains the `report-uri` directive, and the
connection does not comply with the UA's OCSP Stapling Policy, or when the UA
connects to a Known Expect-Staple Host with Expect-Staple metadata that contains 
a `report-uri`, the UA SHOULD report the failure as follows:

1. Prepare a JSON object `report object` with the single key `expect-staple-report`,
   whose value is the result of generating a violation report object as
   described in {{violation-report-object}}.
2. Let `report body` by the JSON stringification of `report object`.
3. Let `report-uri` be the value of the `report-uri` directive in the Expect-Staple
   header field.
3. [Queue a task](https://html.spec.whatwg.org/#queue-a-task) to
   [fetch](https://fetch.spec.whatwg.org/#fetching) `report-uri`, with the
   synchronous flag not set, using HTTP method `POST`, with a `Content-Type`
   header field of `application/expect-staple-report`, and an entity body consisting
   of `report body`.

# Security Considerations

This is efectively a 'report-only' mechanism, are there any security considerations?

## Maximum max-age

Since Expect-Staple is primarily a policy-expansion and investigation technology
rather than an end-user protection, a value on the order of 30 days (2,592,000 seconds)
may be considered a good balance.

## Avoiding amplification attacks

Another kind of hostile header attack uses the `report-uri` mechanism on many
hosts not currently OCSP Stapling as a method to cause a denial-of-service to
the host receiving the reports. If some highly-trafficked websites emitted
a non-enforcing Expect-Staple header with a `report-uri`, implementing UAs' reports
could flood the reporting host. It is noted in {{the-report-uri-directive}} that UAs
should limit the rate at which they emit reports, but an attacker may alter the
Expect-Staple header's fields to induce UAs to submit different reports to different
URIs to still cause the same effect.

# Privacy Considerations

Reports submitted to the `report-uri` could reveal information to a third party
about which webpage is being accessed and by which IP address, by using individual
`report-uri` values for individually-tracked pages. This information could be leaked
even if client-side scripting were disabled.

Implementations must store state about Known Expect-Staple Hosts, and hence which
domains, the UA has contacted.

Violation reports, as noted in {{reporting-expect-staple-failure}}, contain
information about the certificate chain that has violated the OCSP Stapling policy. 
In some cases, such as organization-wide compromise of the end-to-end security of TLS,
this may include information about the interception tools and design used by the
organization that the organization would otherwise prefer not be disclosed.

Because Expect-Staple causes remotely-detectable behavior, it's advisable that UAs
offer a way for privacy-sensitive users to clear currently noted Expect-Staple
hosts, and allow users to query the current state of Known Expect-Staple Hosts.

# IANA Considerations

TBD

# Usability Considerations

When the UA detects a Known Expect-Staple Host in violation of the UA's OCSP Stapling 
Policy, users will experience denials of service. It is advisable for UAs to explain the
reason why.