---
title: EDNS(0) Privacy Options
abbrev: EDNS(0) Privacy Options
docname: draft-wood-dprive-edns0-privacy-latest
date:
category: info

ipr: trust200902
keyword: Internet-Draft

stand_alone: yes
pi: [toc, sortrefs, symrefs]

author:
  -
    ins: C. A. Wood
    name: Christopher A. Wood
    org: Apple Inc.
    street: 1 Infinite Loop
    city: Cupertino, California 95014
    country: United States of America
    email: cawood@apple.com

normative:
    RFC1035:
    RFC2104:
    RFC4033:
    RFC5077:
    RFC6891:
    RFC7830:
    I-D.ietf-dprive-padding-policy:
    I-D.ietf-tls-tls13:

--- abstract

TODO

--- middle

# Introduction

The Domain Name System (DNS) {{RFC1035}} was designed to transport all messages
in cleartext. By default, this provides no privacy, confidentiality, or 
integrity (in the absence of DNSSEC {{RFC4033}}) for all client queries
and responses. {{RFC7858}} standardizes running DNS over (D)TLS to protect
data in transit. This prevents passive eavesdroppers from seeing cleartext
data, though it does not mitigate traffic analysis of data based on packet
sizing, for example, which can be an adequate side channel for learning 
underlying queries and responses ((CITE:SHULMAN)). {{RFC7830}} and
{{I-D.ietf-dprive-padding-policy}} offer mechanisms and advice for clients
and servers to pad padding to queries and responses to mask real message
size information. Padding profiles in {{I-D.ietf-dprive-padding-policy}}
are informed by analyis of DNS message datasets ((TODO:CITE)), though more
analysis could and should be done to better understand this mechanism. 

One privacy-sensitive side channel introduced by DNS recursive resolvers
and not covered by existing mechanisms is caused by caching. Specifically, 
recursive resolvers that cache responses and use them in response to queries
make the observed latency to clients less than that for non-cached
or stale records. Malicious clients may exploit this side channel as an 
oracle to learn what records were previously requested by nearby clients.
Source code for an attack exploiting this side channel is given in Appendix A.

In this document, we propose a new EDNS(0) option that allows clients to mark
queries as privacy sensitive. We specify recursive behavior when processing
private queries and caching their responses. Usage of this option SHOULD only
be enabled when running over a secure transport such as TLS {{RFC7858}}, as
otherwise recursive resolvers or other responders are subject to denial-of-service
attacks.

# Terminology

The terms "Requestor" and "Responder" are to be interpreted as
specified in {{RFC6891}}.

# "Private" Option

EDNS(0) {{RFC6891}} specifies a mechanism to include new options in DNS messages.
This document specifies a new "Private" option that allows clients to indicate 

~~~
0                       8                      16
+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
|                  OPTION-CODE                  |
+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
|                 OPTION-LENGTH                 |
+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
|     PRIVATE Option    |
+--+--+--+--+--+--+--+--+
~~~

The OPTION-CODE for the "Private" option is TBD.

The OPTION-LENGTH for the "Private" option is one octet.

The PRIVATE octet MUST be set to 0x01. Presence of this
option indicates that a query is private. Other values MUST
NOT be used. Recursives MUST drop the query if another value
is set, as discussed in Section {{usage-recursive}}.

# Stub Usage and Behavior {#usage-stub}

TODO

# Recursive Usage and Behavior {#usage-recursive}

TODO

# IANA Considerations

((TODO: codepoint for handshake message type))

# Security Considerations

TODO

# Privacy Considerations

TODO

# Acknowledgments

TODO

--- back

# Timing Oracle Attack Code

TODO: include bash script that connects to recursive and performs correlation attack
