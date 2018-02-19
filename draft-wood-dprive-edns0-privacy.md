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
    RFC7858:
    I-D.ietf-dprive-padding-policy:
    I-D.ietf-tls-tls13:
    Acs17:
        title: Privacy-Aware Caching in Information-Centric Networking
        target: http://ieeexplore.ieee.org/document/7874168/

--- abstract

TODO

--- middle

# Introduction {#introduction}

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

<!--
TODO: comment that this is not a problem without TLS since you can see queries in the clear
The attack here is to obviate the privacy gains of opportunistic TLS connections
-->

In this document, we propose a new EDNS(0) option that allows clients to mark
queries as privacy sensitive. We specify recursive behavior when processing
private queries and caching their responses. Usage of this option SHOULD only
be enabled when running over a secure transport such as TLS {{RFC7858}}, as
otherwise recursive resolvers or other responders are subject to denial-of-service
attacks.

# Terminology

The terms "Requestor" and "Responder" are to be interpreted as
specified in {{RFC6891}}.

# Privacy Attacks {#attacks}

TODO: attack description(s)

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
NOT be used. Responders, such as recursive resolvers, MUST drop 
the query if another value is set, as discussed in Section {{usage-recursive}}.

# Requestor Usage and Behavior {#usage-stub}

Requestors MAY include a "Private" option for any query deemed private or 
sensitive. This may include queries for explicitly known-to-be private 
domains or for all domains when operating in a "private" mode. Specific
recommendations for when to include this option are outside the scope
of this document. 

# Responder Usage and Behavior {#usage-recursive}

Responders that receive a query Q with "Private" options MUST do one
of the following to satisfy Q:

1. If response R is not cached, resolve Q using upstream 
authoritative or recursive resolvers.
2. If response R is cached, ignore R and attempt to resolve Q
using upstream authoritative or recursive resolvers.
3. If response R is cached, delay sending R to the requestor
until some time T has passed. T is a random variable with distribution
equal to the resolution distribution time of R.

TODO: summarize rules for privacy bits in queries and different attacks

## Delay Distributions

T measurement and distribution estimation is critical for masking the 
timing side channel described in Section {{introduction}}. Both should
be chosen to maximize requestor utility while minimizing responder overhead.
We draw on observations and experiments by {{Acs17}} in deciding these
factors.

Responders SHOULD measure query resolution time T' and use this in selecting 
T's distribution. There are two recommended delay strategies for responders
assuming knowledge of T', including:

1. Uniform: Delay for some time T* that is selected uniformly at random
in the range [0, T'], i.e,. with probability distribution function Pr[T = t] = 1/T'.
2. Geometric: Delay for some time T* that is selected according to a geometric
distribution, i.e., with probability distribution function Pr[T = t] = (1 - t) * t^k,
where k is a parameter for the distribution.

# IANA Considerations

((TODO: codepoint for option type))

# Security Considerations

TODO

# Privacy Considerations

TODO

# Acknowledgments

TODO

--- back

# Timing Oracle Attack Code

TODO: include bash script that connects to recursive and performs correlation attack
