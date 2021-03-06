---
title: EDNS(0) Private Option
abbrev: EDNS(0) Private Option
docname: draft-wood-dprive-edns0-privacy-latest
date:
category: info
ipr: trust200902
keyword: Internet-Draft

stand_alone: yes
pi: [toc, sortrefs, symrefs]

author:
  -
    ins: O. Gudmundsson
    name: Olafur Gudmundsson
    org: Cloudflare Inc.
    email: olafur+ietf@cloudflare.com
  -
    ins: N. Sullivan
    name: Nick Sullivan
    org: Cloudflare Inc.
    email: nick@cloudflare.com
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
    RFC4033:
    RFC6891:
    RFC7858:
    Acs17:
        title: Privacy-Aware Caching in Information-Centric Networking
        target: http://ieeexplore.ieee.org/document/7874168/
        date: false

--- abstract

This document specifies the EDNS(0) "Private" option, which allows
DNS clients to signal to servers (recursive resolvers) that query
answers should not be cached. This option is intended to mitigate
cache probing attacks on DNS query privacy.

--- middle

# Introduction {#introduction}

The Domain Name System (DNS) {{RFC1035}} was designed to transport all messages
in cleartext. This provides no privacy, confidentiality, or integrity
(in the absence of DNSSEC {{RFC4033}}) for all client queries and responses
against a network adversary.
{{RFC7858}} standardizes running DNS over (D)TLS to protect
data in transit.

Even if data is protected in transit, the current caching behavior of
recursive DNS resolvers introduces a side channel for an active network
adversary to learn queries made by other clients.
Specifically, in response to a client's query, a recursive resolver will
fetch the corresponding record by traversing the DNS hierarchy and cache
it to more efficiently respond to the same query in the future.
Returning a cached record is expected to be faster than making network
requests to fetch it and that timing difference is observable by the
client making the query.
Because the cache is shared by all clients, if a client observes a fast
response time for a given query it can infer that some other client has
already made that query in the past.
Malicious clients may exploit this caching behavior to use the cache
as an oracle for the network activity of other clients.

In this document, we propose a new EDNS(0) option that allows clients to mark
queries as privacy sensitive. We specify recursive behavior when processing
private queries and caching their responses. Usage of this option SHOULD only
be enabled when running over a spoofing-resistant transport such as TCP, as
recursive resolvers may choose to cache responses in per-user caches identified by
network information such as the source port and IP.

# Terminology

The terms "Requestor" and "Responder" are to be interpreted as
specified in {{RFC6891}}.

- Private query: A query carrying the PRIVATE EDNS(0) option.

# Cache Probe Privacy Attack {#attacks}

Securing DNS traffic in transit via DNS-over-TLS has many benefits. Minimally, it prevents
eavesdroppers observing queries and responses in transit. However, as this only protects the
data channel, other privacy attacks remain possible. In this document, we focus solely
on the cache probe attack outlined in Section {{introduction}}. We describe the attack
in more detail here.

Let S be a benign client (requestor) using recursive resolver (responder) R.
In answering S's queries, R speaks to several authoritative servers A1,...,An.
Let RTT(requestor, responder) be a function that computes the average RTT for a given
query and response between requestor and responder. For example, RTT(R,A1) is the
average RTT for answering a query sent by R and answered by A1. Let Delay(q, r)
be a function that measures the delay in answering request q with response r.
Finally, let Adv be a malicious client who wishes to learn what queries were asked by S.
Adv proceeds as follows. Assume Adv wants to learn if S queried for q = example.com.
To do so, Adv requests q from R, yielding response r, and measures d = delay(q, r). Then,
Adv sends a random query q' in the same top-level domain (e.g., .com), yielding response r',
that S is unlikely to have requested, and measures d' = delay(q', r').
If |d - d'| ~ 0, then Adv concludes q was not requested by S (or someone else). Otherwise,
Adv concludes that, with some probability q was requested.

Severity of this attack depends on several factors:

- How many clients (stubs) are served by R.
- What types of queries are requested by other clients of R. For example, do they require more
recursive resolution steps to resolve? If so, this may inflate time required to answer
the query.

While learning the existence of a certain query may not always constitute a privacy violation,
it may for some clients.

## Recursive Resolver Deployments

Large-scale recursive resolvers, such as those operated by Quad9, Cloudflare, and Google,
traditionally deploy several servers behind well known anycast IP addresses, such
as 8.8.8.8 or 9.9.9.9. Thus, stub queries to such well known resolvers may not be
serviced by the same logical recursive resolver. UDP queries and TCP connections
may be routed to different resolvers based on distance and connection cost.
This increases the difficulty of cache probing attacks. Recursive resolvers that are not
deployed at such large scales are more susceptible to this type of attack.

# PRIVATE Option

EDNS(0) {{RFC6891}} specifies a mechanism to include new options in DNS messages.
This document specifies a new "PRIVATE" option that allows clients to express
the maximum desired cache lifetime for DNS query answers.

~~~
0                       8                      16
+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
|                  OPTION-CODE                  |
+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
|                 OPTION-LENGTH                 |
+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
|    PRIVATE Option     |
+--+--+--+--+--+--+--+--+
~~~

The OPTION-CODE for the "PRIVATE" option is TBD.

The OPTION-LENGTH for the "PRIVATE" option is one octet.

The PRIVATE octet MUST be set to 0x01. Presence of this
option indicates that a query is private. Other values MUST
NOT be used. Responders, such as recursive resolvers, MUST drop
the query if another value is set, as discussed in Section {{usage-recursive}}.

This option does not affect intermediate queries performed
to produce an answer to a query. Responders MUST only
honor it for the final answer. For example, when answering a
query for a.b.c.d.example.com, only the final answer for
a.b.c.d.example.com is affected. All intermediate queries,
e.g., for .com, .example.com, etc., MAY be cached as needed.

# Requestor Usage and Behavior {#usage-stub}

Requestors MAY include a "PRIVATE" option for any query deemed private or
sensitive. This may include queries for explicitly known-to-be private
domains or for all domains when operating in a "private" mode. Specific
recommendations for when to include this option are outside the scope
of this document.

# Responder Usage and Behavior {#usage-recursive}

There are two ways responders can handle a "PRIVATE" option.
If present, responders MUST not cache any final answer for longer
than N seconds. N is a deployment- and instance-specific variable
that changes based on responder load and client pool. Larger values
of N make probing attacks more feasible.

((OPEN ISSUE: determine if N > 0 is an acceptable trade-off))

Responders MAY cache any intermediate answer used to produce the final response.

In cases where responders need to reduce upstream redundant queries, e.g.,
because they are expensive or otherwise reveal more information to authoritative
servers, there are (at least) responders MAY adjust their cache policy to handle
private queries with per-client, segmented caches. This approach works as follows:

Responders that receive a query Q without the "Private" options may treat
it as normal, i.e., by serving a response from cache if available. For a query Q
with a "Private" option sent from client S, responders MUST do the following:

- Index S's private cache using Q, if it exists. If response R is present, serve it to S.
- If S's private cache does not exist, attempt to resolve Q as normal. Cache the response
in S's private cache.

When S's connection to R is torn down, S's private cache MUST be flushed and memory released.
Responders SHOULD apply whatever per-client caching policy is sensible to balance utility
amongst shared (non-private) and private caches. Segmented caches introduce more memory
requirements for resolvers at the cost of improving client privacy.

This approach is only viable when clients connect to resolvers over a mechanism that
provides the server with a way to uniquely identify clients in a validated way.
For transports such as DNS-over-HTTPS (DOH) {{?I-D.ietf-doh-dns-over-https}},
this can be the incoming IP and port pair, or a client certificate. Otherwise,
malicious clients may flood resolvers with private queries and induce cache fragmentation.

# DNS-over-HTTPS Application

A similar per-query "no-cache" flag may be implemented with DNS-over-HTTPS {{?I-D.ietf-doh-dns-over-https}} by
appending a random nonce to each request. Specifically, given a random N-byte nonce
R, the following query parameter can be appended to a DOH query:

~~~
?rand=R
~~~

DOH recursive resolvers that use an HTTP caching layer to satisfy duplicate queries
SHOULD not satisfy cached queries with the same "dns" parameter yet different (or no)
"rand" parameter.

# IANA Considerations

((TODO: codepoint for option type))

# Security Considerations

Clients that malicious mark all queries as private may induce excessive upstream responder
traffic while responding said queries. However, as malicious clients may do this today by issuing
queries for nonsensical or nonexistent domains, this does not introduce a new attack vector.

# Privacy Considerations

Selectively sending a private DNS query based on user behavior leaks information about user behavior
or system policy. For example, if clients only send private queries while in a certain system
configuration, then presence of a "Private" option indicates, with high probability, that the origin
user's device is in such a configuration. This could be addressed by marking queries as private
-- at random or unilaterally -- such that system configurations are indistinguishable under observation
of the "Private" option.

# Acknowledgments

The authors thank Georgios Kontaxis for feedback on earlier versions of this document.

--- back

# Artificial Delays

An alternative mechanism to support caching without inducing upstream
traffic is for responders to "ignore" cached responses and emulate fetch
delays for queries marked private. This variation may work as follows:

Responders that receive a query Q with "Private" options MUST do one
of the following to satisfy Q:

1. If response R is not cached, resolve Q using upstream
authoritative or recursive resolvers.
2. If response R is cached, ignore R and attempt to resolve Q
using upstream authoritative or recursive resolvers.
3. If response R is cached, delay sending R to the requestor
until some time T has passed. T is a random variable with distribution
equal to the resolution distribution time of R.

## Delay Distributions

T measurement and distribution estimation is critical for masking the
timing side channel described in Section {{introduction}}. Both should
be chosen to maximize requestor utility while minimizing responder overhead.
We draw on observations and experiments by {{Acs17}} in deciding these factors.

Responders SHOULD measure query resolution time T' and use this in selecting
T's distribution. There are two recommended delay strategies for responders
assuming knowledge of T', including:

1. Uniform: Delay for some time T* that is selected uniformly at random
in the range \[0, T'\], i.e,. with probability distribution function Pr\[T = t\] = 1/T'.
2. Geometric: Delay for some time T* that is selected according to a geometric
distribution, i.e., with probability distribution function Pr\[T = t\] = (1 - t) * t^k,
where k is a parameter for the distribution.
