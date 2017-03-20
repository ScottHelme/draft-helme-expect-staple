# draft-helme-expect-staple

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
