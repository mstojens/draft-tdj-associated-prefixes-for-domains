---
###
# Internet-Draft Markdown Template
#
# Rename this file from draft-todo-yourname-protocol.md to get started.
# Draft name format is "draft-<yourname>-<workgroup>-<name>.md".
#
# For initial setup, you only need to edit the first block of fields.
# Only "title" needs to be changed; delete "abbrev" if your title is short.
# Any other content can be edited, but be careful not to introduce errors.
# Some fields will be set automatically during setup if they are unchanged.
#
# Don't include "-00" or "-latest" in the filename.
# Labels in the form draft-<yourname>-<workgroup>-<name>-latest are used by
# the tools to refer to the current version; see "docname" for example.
#
# This template uses kramdown-rfc: https://github.com/cabo/kramdown-rfc
# You can replace the entire file if you prefer a different format.
# Change the file extension to match the format (.xml for XML, etc...)
#
###
title: "Associated IP Prefixes for Domain Names"
abbrev: "associated-prefixes"
category: info

docname: draft-tdj-associated-prefixes-for-domains
submissiontype: IETF  # also: "independent", "editorial", "IAB", or "IRTF"
number:
date:
consensus: true
v: 3
area: AREA
workgroup: WG Working Group
keyword:
 - DNS
 - IP prefixes
 - SVCB
venue:
  group: WG
  type: Working Group
  mail: WG@example.com
  arch: https://example.com/WG
  github: USER/REPO
  latest: https://example.com/LATEST

author:
 -
    fullname: Tommy Jensen
    email: tojens.ietf@gmail.com
 -
    fullname: David Redekop
    organization: Adam Networks Inc.
    email: david.ietf@adamnet.works
 -
    fullname: John Todd
    organization: Quad9
    email: jtodd@quad9.net

normative:

informative:


--- abstract

RFC9000 defines {alternate IP addresses} which enable discovery of IP addresses
to be used in association with a service with a domain name that are potentially
not discoverable as A/AAAA records for that domain name. This draft defines a
mechanism that allows a client to get advance notice of associated IP addresses
for a domain name as part of the DNS query.


--- middle

# Introduction

(TODO: slightly more detailed version of abstract)

# Terminology

TBD


# Mechanism

When a client wishes to discover the IP prefixes that are associated with a
given domain name, it SHOULD issue a SVCB query for the domain name (which it
may already be doing for other reasons). If the "cidrs" key is present, then
it SHOULD issue another query of type CIDRS to retrieve the associated IP
prefixes.

When a server wishes to provide the associated IP prefixes for a given name, it
SHOULD create CIDRS records as well as a SVCB record with the "cidrs" key set.
While the mechanism is simple, the Operational Considerations section contains
further normative implementation guidance.

## CIDRS record format

The CIDRS RR type is designed to convey an IP prefix, an associated port range,
and a protocol number as defined in the Assigned Internet Protocol Numbers IANA
registry. The port range can be defined as 0-65535 and the protocol number as
255 if these fields are not applicable for the record owner (because only the
IP prefix is interesting).

### Wire format

CIDRS RR data on the wire uses the following format (TODO: ASCII art of the fields below):

- IP family (4 bits): indicates which version of the Internet Protocol the prefix belongs to

- Reserved for future use (4 bits)

- Length of Prefix (8 bits): number of bits of the IP prefix.

- Prefix (ceil(Length of Prefix / 8) bytes): the IP prefix itself. The length of this field is always an integer of bytes (begins and ends on octent boundaries) just large enough to contain a prefix of length equal to the Length of Prefix

- Port Range Begin (16 bits): the lowest port number in the included range

- Port Range End (16 bits): the highest port number in the included range

- Protocol Number (8 bits): a value from the Assigned Internet Protocol Numbers IANA registry (255 if no associated protocol should be defined)


### Zone file format

TODO: will do after the idea survives initial contact with the WG


### Justification for new RR type

Because the list of associated IP addresses for a given name is likely to be
somewhat large, at least relative to the typical A, AAAA, and SVCB/HTTPS queries
a client is likely to make, it makes sense to create a new type that ensures
DNS query issuers can opt into receiving this payload. Additionally, inclusion
within SVCB or HTTPS records is a dubious reuse given there is no guarantee that
services at associated IP addresses can authenticate a name (if the resulting
protocol does not perform name verification).


### Verification

CIDRS records MUST be DNSSEC signed. This is because unlike A and AAAA records,
there is no expectation that the resulting traffic the querying client will
send to these IP addresses will be able to prevent attacker impersonation via
secure peer validation such as that provided by TLS certificates. By definition,
the addresses in these CIDRs are used in association with services that use the
domain name but cannot validate claims of the domain name. DNSSEC validation
will provide assurance that the IP addresses are those expected by the valid
owner of the domain name.


# Security Considerations

## Suspiciously large IP address sets

Servers SHOULD avoid claiming very short prefixes in CIDRS records. It is not
expected that a single domain name is legitimately associated with a short
prefix. Clients MAY choose to ignore claims of association by CIDRS records
with prefixes shorter than a preconfigured minimum length per IP version.
This version of the text does not suggest defined values for minimum prefix
lengths, though as a reasonable rule of thumb, domain name owners SHOULD NOT
have prefixes in CIDRS records that include multiple ASNs to avoid clients
considering their prefixes to be too short.

Note that associated IP addresses SHOULD be restricted to IP addresses which a
server reasonably expects a client will need to interact with the functionality
provided by the service which uses the domain name. For example, name owners
SHOULD NOT create CIDRS records that include all IP ranges owned by a company
for the company's primarily recognizable domain name (example-company.example.
having a CIDRS record listing every IP address owned by Example Company would
be inappropriate).

## Large payload sizes

It is expected that in most common use cases, CIDRS records will need more than
one CIDR value, possibly many (balancing this against guidance given in
{{complexity}}). Servers SHOULD truncate responses to avoid creating risk of
effective DDoS attacks, even if the CIDRS record would fit in a single UDP
packet. This means in effect that CIDRS records SHOULD NOT ever be sent using
unencrypted DNS over UDP.

# Operational Considerations

## TTLs for SVCB and CIDRS records

Servers SHOULD NOT have a TTL value for SVCB records containing the "%%%%%%%%%"
key that are less than the TTL values for CIDRS records for the same domain
name. This would cause the client to re-check for the presence of CIDRS records
that it will still have cached.

The TTL values of CIDRS records SHOULD NOT be any shorter than the expected
lifetime of traffic flows of typical service usage. Doing so would encourage
DNS stub resolvers and the processes calling DNS sutb resolver APIs to ignore
TTL values in favor of supporting performant user experiences.

## Complex messages {complexity}

{{?RFC3123}} defines an experimental mechanism by which the APL RR type can both
define CIDRs that are somehow associated with a domain name as well as negate
subsets of the CIDR. This is specifically not supported for the CIDRS RR type
because of the added complexity this creates for implementors and the support
for defining non-consecutive subnets for the same purpose. Implementors SHOULD
reduce the number of CIDRs needed for a given domain name rather than have
many long prefixes that cannot be grouped under fewer CIDRs without needing to
define the non-included gaps within them.

## Transition period workarounds

Before deployment of this document is common, clients will frequently run into
the problem of wanting to discover associated IP addresses for a given zone, but
the zone owner does not yet support this document. During this ramp up of
deployment, administrators might turn to workarounds, such as creating a zone
they control to distribute CIDRs the administrator knows to be associated with
services they depend on but do not yet support this document.

How to accomplish such a mapping is left to implementors as a non-standard
mechanism. This is out of scope for this document, which only defines
advertisement of IP addresses associated with a given name directly.

{closing}




# Conventions and Definitions

{::boilerplate bcp14-tagged}


# Security Considerations

TODO Security


# IANA Considerations

This document has no IANA actions.


--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
