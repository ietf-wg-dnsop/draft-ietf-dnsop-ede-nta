---
title: "Disclosure of Negative Trust Anchors in DNS Responses"
abbrev: "EDE NTA"
docname: draft-ietf-dnsop-ede-nta-latest
category: info
stream: IETF
ipr: trust200902
area: "Operations and Management"
keyword:
  - DNS
  - EDE
  - Extended DNS Errors
  - DNSSEC
  - NTA
  - Negative Trust Anchor
pi: [toc, tocindent, sort refs, symrefs, strict, compact, inline]
updates: 7646
author:
  - ins: B. Farrokhi
    name: Babak Farrokhi
    org: Quad9
    email: babak@farrokhi.net
  - ins: J. Abley
    name: Joe Abley
    org: Cloudflare
    email: jabley@cloudflare.com
  - ins: S. Neuteboom
    name: Sebastiaan Neuteboom
    org: Cloudflare
    email: sebastiaan@cloudflare.com
---

--- abstract

This document describes a mechanism for disclosing that a Negative
Trust Anchor (NTA) was in effect at the time that a DNS response
was generated, using an Extended DNS Error (EDE).

--- middle

# Introduction

{{!RFC8914}} defines the Extended DNS Error (EDE) mechanism, which
allows DNS servers to encode additional information in a DNS response.

{{!RFC7646}} defines the concept of a DNSSEC Negative Trust Anchor
(NTA), an operational mechanism by which a validating resolver can
be configured to temporarily disable DNSSEC validation for a specific
domain to mitigate misconfiguration.

A resolver with an NTA in effect might send a response that ordinarily
would have been suppressed because of validation failures.  This
document defines a new EDE that can be sent within a response to
indicate that the response was subject to an active NTA.

A further goal of this signal is transparency toward end users and
applications.  {{Section 3.1 of !RFC7646}} recommends that operators
disclose the NTAs they have in place, for example on a website, and
notes that no in-band DNS signal exists to indicate that an NTA is in
effect.  This document defines that in-band signal, complementing such
out-of-band disclosure.

# Terminology

{::boilerplate bcp14-tagged}

This document assumes a familiarity with common DNS terminology as
described in {{!RFC9499}}.

# Extended DNS Error Code 33 - Negative Trust Anchor in Effect

A response that includes one or more instances of EDE 33 was
generated with a covering NTA {{!RFC7646}} in effect.

As with all EDEs, the EDE defined in this document is diagnostic;
per {{Section 6 of !RFC8914}} a client MUST NOT use its presence
to alter protocol processing.  The inclusion of this EDE in a
response does not change AD bit processing in DNS messages in any
way. The only purpose of this EDE is to provide additional information
about the response in which it appears.

This EDE is intended for use in DNS responses sent by a DNS resolver
with a configured NTA and MUST NOT be included in other responses.
For example, a DNS response sent by an authoritative-only DNS server,
which does not perform validation and hence has no obvious use for
an NTA, SHOULD NOT include this EDE.

# Operational Considerations

An operator that applies an NTA SHOULD return this EDE in affected
responses, so that end users and applications can tell that the
response may not have been DNSSEC-validated.  This complements, and
does not replace, the disclosure recommended in {{Section 3.1 of
!RFC7646}}.

A response might be affected by an NTA for various reasons, not
just in the case where the QNAME is subordinate to the domain for
which an NTA has been configured. In any case, a resolver MAY include
this EDE on any responses while an NTA is in effect, regardless of
whether the presence of the NTA had a material effect on the contents
of the response.

A resolver with multiple NTAs in place simultaneously MAY include
multiple instances of this EDE in a single response. Multiple
instances of this EDE in a single response might each have different
EXTRA-TEXT fields, for example, and might each describe a different
active, applicable NTA. When multiple instances of this EDE are
included in a response, the EXTRA-TEXT field in each instance MUST
be populated.

The operator MAY use the EXTRA-TEXT field to add context about the
NTA, such as the name at which it was configured, the reason it was
put in place, a reference where more information can be found, or
its expected duration.  As noted in {{Section 2 of !RFC8914}},
EXTRA-TEXT is intended for human consumption; operators SHOULD keep
it readable and SHOULD NOT include private or sensitive information.

Structured data MAY be included in the EXTRA-TEXT field, as described
in {{!I-D.ietf-dnsop-structured-dns-error}}. The JSON Names "d" and
"t" have been registered for this purpose (see {{iana}}) and should
be used and interpreted when used with this EDE as follows:

| JSON Name  | Use and interpretation for this EDE  |
| d          | The domain name at which an active NTA has been configured  |
| t          | An indicative time at which this NTA might be expected to remain in place until  |

Note that it is usual for NTAs to be configured in response to
unplanned events that have afflicted unaffiliated third parties,
and hence consumers of structured EXTRA-TEXT should be interpret
timestamps with generous flexibility.


# IANA Considerations {#iana}

The IANA has made the following allocation in the "Extended DNS
Error Codes" registry under the "Domain Name System (DNS) Parameters"
registry group:

| INFO-CODE  | Purpose               | Reference     |
|:-----------|:--------------------- |:------------- |
| 33         | Negative Trust Anchor | This document |

The IANA is directed to update the "EXTRA-TEXT JSON Names" registry under
the "Domain Name System (DNS) Parameters) registry group by adding the
following entries:

| JSON Name | Field Meaning | Description | Reference |
|:--------- |:------------- |:----------- |:--------- |
| d         | domain-name   | A fully-qualified domain name (in the case of an IDN, containing only A-labels) with no trailing period  | This document  |
| t         | timestamp     | A timestamp in {{!RFC3339}} format  | This document  |


# Security Considerations

An NTA represents an intentional, operator-driven suspension of
DNSSEC validation for a specific domain. See {{!RFC7646}} for further
discussion of the operational and security implications of NTAs.

The presence of the EDE defined in this document does not modify
AD bit processing in DNS messages, and its only purpose is to provide
additional information.

EDEs encoded in DNS messages carry no cryptographic signatures and
hence enjoy no inherent integrity protection.  An on-path attacker
could add, remove, or modify an EDE.  Clients that require integrity
protection of these signals should use a suitable mechanism, such
as TSIG {{?RFC8945}}, SIG(0) {{?RFC2931}} or an authenticated and
encrypted transport protocol such as DNS over TLS {{?RFC7858}} or
DNS over HTTPS {{?RFC8484}}.  See {{Section 6 of !RFC8914}} for
more discussion.

--- back

# Acknowledgements

The authors acknowledge review and ideas from Carlos Horowicz,
Mukund Sivaraman, Ralf Weber, Warren Kumari, Robert Edmonds and
Petr Spacek.

# Examples

The following examples use configurations that exist in the DNS at
the time of writing. Readers from the future may need to create
analogous configurations in order to reproduce the results illustrated
here.

These examples were constructed using the well-known diagnostic tool
dig, as shipped with BIND9. The output was hand-edited slightly to
fit the formatting constraints of the document, but is otherwise
consistent with the original output.

## NTA in Effect, Valid DNSSEC Configuration

The zone NTAMUCH.ORG is signed and an intact and functional path
of trust exists back to the root zone. An NTA is applied to the
domain NTAMUCH.ORG at the security-aware, validating resolver
1.1.1.1, but not at 9.9.9.9.

~~~
jabley@manta ~ % dig @1.1.1.1 NTAMUCH.ORG SOA +nocd +rec +multiline

; <<>> DiG 9.20.24 <<>> @1.1.1.1 NTAMUCH.ORG SOA +nocd +rec +multiline
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 46491
;; flags: qr rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 1232
; EDE: 33: (a Negative Trust Anchor has been applied for this query
;   (see RFC 7646))
;; QUESTION SECTION:
;NTAMUCH.ORG.		IN SOA

;; ANSWER SECTION:
NTAMUCH.ORG.  1800  IN  SOA barbara.ns.cloudflare.com. dns.cloudflare.com. (
                          2407706086 ; serial
                          10000      ; refresh (2 hours 46 minutes 40 seconds)
                          2400       ; retry (40 minutes)
                          604800     ; expire (1 week)
                          1800       ; minimum (30 minutes)
                        )

;; Query time: 21 msec
;; SERVER: 1.1.1.1#53(1.1.1.1) (UDP)
;; WHEN: Fri Jun 26 10:49:10 CEST 2026
;; MSG SIZE  rcvd: 181

jabley@manta ~ % dig @9.9.9.9 NTAMUCH.ORG SOA +nocd +rec +multiline

; <<>> DiG 9.20.24 <<>> @9.9.9.9 NTAMUCH.ORG SOA +nocd +rec +multiline
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 7960
;; flags: qr rd ra ad; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 1232
;; QUESTION SECTION:
;NTAMUCH.ORG.		IN SOA

;; ANSWER SECTION:
NTAMUCH.ORG.  1767  IN  SOA barbara.ns.cloudflare.com. dns.cloudflare.com. (
                          2407706086 ; serial
                          10000      ; refresh (2 hours 46 minutes 40 seconds)
                          2400       ; retry (40 minutes)
                          604800     ; expire (1 week)
                          1800       ; minimum (30 minutes)
                        )

;; Query time: 11 msec
;; SERVER: 9.9.9.9#53(9.9.9.9) (UDP)
;; WHEN: Fri Jun 26 10:49:20 CEST 2026
;; MSG SIZE  rcvd: 105

jabley@manta ~ %
~~~

## NTA in Effect, Validation Problems Expected

The zone BROKEN.NTAMUCH.ORG is delegated from the NTAMUCH.ORG zone
using an accurate NS RRSet both sides of the zone cut, but a
deliberately-fabricated DS RRSet at on the parent side. Records in
the child zone are not signed, and no child zone DNSKEY RRSet exists.
The effect is a secure delegation to an unsigned zone, a broken
configuration that we should expect to trigger validation failures.

The same NTA as described in the previous example is in place for
the domain NTAMUCH.ORG on the resolver 1.1.1.1, but, again, not
on the resolver 9.9.9.9.

Both resolvers include different EDEs in the response which illustrate
the problem, but note the different RCODEs: with the NTA in place, the
1.1.1.1 resolver returns an answer with NOERROR while the 9.9.9.9
resolver with no applicable NTA correctly returns RCODE SERVFAIL.

~~~
jabley@manta ~ % dig @1.1.1.1 BROKEN.NTAMUCH.ORG SOA +nocd +rec +multiline

; <<>> DiG 9.20.24 <<>> @1.1.1.1 BROKEN.NTAMUCH.ORG SOA +nocd +rec +multiline
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 25660
;; flags: qr rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 1232
; EDE: 9 (DNSKEY Missing): (no SEP matching the DS found for
;   broken.ntamuch.org.)
; EDE: 33: (a Negative Trust Anchor has been applied for this
;   query (see RFC 7646))
;; QUESTION SECTION:
;BROKEN.NTAMUCH.ORG.	IN SOA

;; ANSWER SECTION:
BROKEN.NTAMUCH.ORG.  300  IN  SOA maleah.ns.cloudflare.com. dns.cloudflare.com. (
                                2407705944 ; serial
                                10000      ; refresh (2 hours 46 minutes 40 seconds)
				2400       ; retry (40 minutes)
				604800     ; expire (1 week)
				1800       ; minimum (30 minutes)
				)

;; Query time: 20 msec
;; SERVER: 1.1.1.1#53(1.1.1.1) (UDP)
;; WHEN: Fri Jun 26 10:52:52 CEST 2026
;; MSG SIZE  rcvd: 245

jabley@manta ~ % dig @9.9.9.9 BROKEN.NTAMUCH.ORG SOA +nocd +rec +multiline

; <<>> DiG 9.20.24 <<>> @9.9.9.9 BROKEN.NTAMUCH.ORG SOA +nocd +rec +multiline
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: SERVFAIL, id: 20047
;; flags: qr rd ra; QUERY: 1, ANSWER: 0, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 1232
; EDE: 9 (DNSKEY Missing)
;; QUESTION SECTION:
;BROKEN.NTAMUCH.ORG.	IN SOA

;; Query time: 116 msec
;; SERVER: 9.9.9.9#53(9.9.9.9) (UDP)
;; WHEN: Fri Jun 26 10:53:05 CEST 2026
;; MSG SIZE  rcvd: 53

jabley@manta ~ %
~~~

