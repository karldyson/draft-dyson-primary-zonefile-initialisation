---
title: "Initialisation of Zone Files on the DNS Primary Server"
category: std
updates: 9432

docname: draft-dyson-primary-zonefile-initialisation-latest
submissiontype: IETF  # also: "independent", "editorial", "IAB", or "IRTF"
stand_alone: true
number:
date:
consensus: true
v: 3
# area: Operations and Management
# workgroup: Internet Engineering Task Force
keyword:
 - catalog zone
 - zone file initialisation
 - primary server
 - master file
#venue:
#  github: karldyson/draft-dyson-primary-zonefile-initialisation

author:
 -
    fullname: Karl Dyson
    organization: Nominet UK
    street:
        - Minerva House
        - Edmund Halley Road
        - Oxford Science Park
    city: Oxford
    code: OX4 4DQ
    country: UK
    email: karl.dyson@nominet.uk
    uri: https://nominet.uk

normative:
  RFC9432:
  RFC2136:
  RFC2119:
  RFC1035:
  RFC3596:

informative:

--- abstract

This document describes an update to {{RFC9432}} that facilitates a method for the primary server of a DNS zone to create the underlying master file for member zone(s) using information contained within a catalog zone.

--- middle

# Introduction

Once a DNS zone's master file exists on the primary server, there are standards-compliant ways to automate the distribution of that zone to secondary servers in {{RFC9432}} DNS Catalog Zones. Further, there are standards-compliant ways to alter the contents of the zone in {{RFC2136}} Dynamic Updates.

However there is no standards-compliant method of initialising a new master file for the zone, ready for such operations.

Various DNS software products have proprietary mechanisms for acheiving this, some requiring that the zone master file is somehow populated first on the primary servers' filesystem.

However, there is no standards-compliant vendor-independent mechanism of signalling to the primary server that a new file is to be created for the zone and populated with basic minimal initial zone data.

Operators of large scale DNS systems may want to be able to signal the creation of a new file for a new zone without wanting to be tied to a particular vendor's proprietary software. Further, they may want to avoid the need or overhead of engineering a bespoke solution with the ongoing need to support and maintain it.

Having dynamically provisioned a new zone on the primary server, the operator may then manage resource records in the zone via Dynamic Updates {{RFC2136}}. In this scenario, they may also want to distribute the zones to secondary servers via DNS Catalog Zones {{RFC9432}}.

The scope of this document is therefore confined to the initial provisioning of the zone file on the primary server, and optional signalling of initial DNSSEC policy or configuration (see {{dnssecConsideration}}).

Broader provisioning of the base nameserver configuration is beyond the scope of this document.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

The following is in addition to the conventions and definitions as defined in {{RFC9432}}.

Within DNS servers, specifically when transferring zones to other servers, there is the concept of a primary server and a secondary server in each transfer relationship.

Each secondary server will be transferring the zone from a configured upstream primary server, which may, itself, be a secondary server to a further upstream primary server, and so on.

However, within this document, the term "primary server" is used specifically to mean the primary server at the root of the AXFR/IXFR dependency graph. This server is where the resource records that form the zone's content are maintained in the master file. Thus, that server does not transfer the zone from another server.

The term "master file" is as per the description in Section 5 of {{RFC1035}}, noting that some software products offer data stores for the master file that are not an actual file on a filesystem, such as a database.

The use of parenthesis in the examples is as described in Section 5.1 of {{RFC1035}}.

# Catalog Properties

If initialisation of the underlying master file for the member zone is not required or is disabled in configuration, then the various init properties defined in this document MAY be absent, and in that context, their absence DOES NOT constitute a broken catalog zone.

However, if the initialisation of the underlying master file for the member zone is enabled, and the properties and parameters defined below constitute a broken configuration as defined in this document, then the catalog is broken, and MUST NOT be processed (Section 5.1 {{RFC9432}}).

## Zone File Initialisation (init property)

When suitable configuration is activated in the implementation, and a new member zone entry is added to the catalog being served by the primary server, the primary server SHOULD create the underlying master file for the zone with the properties and parameters outlined in the init property.

The init property is the parent label to the other labels that facilitate the adding of the various properties and parameters.

The implementation may permit the following on a global, or per catalog basis, by way of suitable configuration parameters:

  * The master file is ONLY created for the zone if the master file does not already exist
  * The master file is NEVER created (effectively, the initialisation capability is disabled for this catalog or primary server, and the master file would be expected to exist as is the case before this document)
  * The master file is ALWAYS created, overwriting any existing master file for the zone

The second of the above options can facilitate the initialisation of a downstream signer configuration without necessarily being used to signal the initial state of the zone's master file on the primary server.

A number of sub-properties, expressed as labels within the bailiwick of the "init" label, define the initialisation parameters.

## Start Of Authority (soa property)

The soa property is used to specify the SOA that will be applied to the created zone file for the member zone.

There MUST be one and ONLY one soa property record with the exception that a member zone record can be specified to override the default (see {{memberZoneSection}} below).

Multiple soa property records within a given scope constitutes a broken catalog zone.

Absence of a soa property similarly constitues a broken catalog zone.

### Parameters

With the exception of the serial number, the SOA record parameters are supplied as key=value pairs in the RDATA of a TXT resource record, with the pairs separated by whitespace.

The keys are as per the field names expected in a SOA record as defined in Section 3.3.13 of {{RFC1035}}. The values supplied MUST be compliant with the definitions therein.

### Example

~~~~
soa.init.$CATZ 0 TXT ( "mname=<mname> rname=<rname> "
      "refresh=<refresh> retry=<retry> expire=<expire> "
      "minimum=<minimum>" )
~~~~

## Nameservers (ns property)

Actual NS records cannot be used, as we do not want to actually delegate outside of this catalog zone.

The nameserver parameters are supplied as key=value pairs in the RDATA of a TXT resource record, with the pairs separated by whitespace.

If the nameservers are in-bailiwick and address records are therefore required, suitable address records MUST be created in the member zone's master file from the parameters specified.

If the nameservers are in-bailiwick of a zone in the catalog, and an address is not specified, this would result in a zone that will not load; This denotes a broken catalog zone.

There MUST be at least one ns property record that can be applied to a member zone, either specified directly as per {{memberZoneSection}} or in the catalog section to be inherited.

Therefore, a catalog zone that contains no nameserver entries constitutes a broken catalog zone.

The ns property can be specified multiple times, with one nameserver specified per entry.

### name Parameter

The "name" parameter MUST be present, and contains the name of the nameserver as it should appear in the zone's master file. To avoid ambiguity in behaviour, this SHOULD be a fully qualified and dot-terminated name.

The value of the "name" parameter MUST be compliant with Section 3.3.11 of {{RFC1035}}.

An ns property record that does not contain a "name" parameter consistutes a broken catalog zone.

### ipv4 and ipv6 Parameters

The "ipv4" and "ipv6" parameters are OPTIONAL. They contain the IP address(es) of the hostname specified in the "name" parameter.

If the value in the "name" parameter is in-bailwick, and hence requires that the relevant address enties are also created in the zone, at least one of either the "ipv4" or "ipv6" parameters MUST be specified.

The value of the "ipv4" parameter, if present, MUST be a valid IPv4 address, compliant with Section 3.4.1 of {{RFC1035}}.

The value of the "ipv6" parameter, if present, MUST be a valid IPv6 address, compliant with Section 2.1 of {{RFC3596}}.

An ns property record that contains an in-bailiwick name, but does not contain at least one address parameter constitutes a broken catalog zone.

### Example

~~~~
ns.init.$CATZ 0 TXT ( "name=some.name.server. "
      "ipv4=192.0.2.1 ipv6=2001:db8::1" )
ns.init.$CATZ 0 TXT ( "name=another.name.server. "
      "ipv4=192.0.2.129 ipv6=2001:db8:44::1" )
~~~~

## DNSSEC (dnssec Property)

Placeholder for definition of how to signal to a zone's signer(s) that we wish to initialise keys and sign the newly created zone...

# Member Zone Properties {#memberZoneSection}

The default properties outlined above can be overridden on a per member zone basis. Where per member zone entries are specified in the catalog, they MUST be used in preference to the default properties specified at the catalog level.

A subset MAY be specified here; for example, the SOA could be omitted here and just the NS records or DNSSEC parameters specified, with the omitted properties inherited from the catalog level values.

~~~~
<unique-N>.zones.$CATZ          0 PTR example.com.
soa.init.<unique-N>.zones.$CATZ 0 TXT ( "mname=<mname> "
      "rname=<rname> refresh=<refresh> retry=<retry> "
      "expire=<expire> minimum=<minimum>" )
ns.init.<unique-N>.zones.$CATZ  0 TXT ( "name=some.name.server. "
      "ipv4=192.0.2.1 ipv6=2001:db8::1" )
~~~~

## Change Of Ownership (coo Property)

There is no change to the coo property; if the member zone changes ownership to another catalog, fundamentally, the zone's master file already exists.

The scope of this document is solely concerned with the initialisation of a new zone's master file, and so in the case of the zone changing ownership, the initialisation parameters MUST NOT be processed.

# Name Server Behaviour

## General Behaviour

The parameters specified in the init property will contain hostnames, for example in the NS records and in the SOA; these will be replicated verbatim into the zone upon creation, and so it should be noted that if they would be fully qualified in a manually created master file for the zone, they MUST be fully quallified in the parameter specification in the property.

# Security Considerations

This document does not alter the security considerations outlined in {{RFC9432}}.

# IANA Considerations {#IANA}

IANA is requested to add the following entries to the registry:

Registry Name: DNS Catalog Zones Properties

Reference: this document

| Property Prefix | Description        | Status          | Reference     |
| init            | Initialisation     | Standards Track | this document |
| soa             | Start Of Authority | Standards Track | this document |
| ns              | Name Server        | Standards Track | this document |
| dnssec          | DNSSEC             | Standards Track | this document |
{:title="DNS Catalog Zones Properies Registry"}

Field meanings are unchanged from {{RFC9432}}.

--- back

# Examples

## Catalog Zone Example

The following is an example catalog zone showing the additional properties and parameters as outlined in this document.

There are defaults specified for the SOA and NS records, which would be used by the example.com. zone.

The example.net. zone would utilise the default SOA record, but would utilise the more specific NS records.

The default nameservers are in-bailiwick of example.com, which is in the catalog, and so the address record details are supplied in order to facilitate the addition of the address records.

~~~~
catz.invalid.                   0 SOA invalid. invalid. (
      1 3600 600 2419200 3600 )
catz.invalid.                   0 NS invalid.
soa.init.catz.invalid.          0 TXT ( "mname=ns1.example.com. "
      "rname=hostmaster.example.com. refresh=14400 "
      "retry=900 expire=2419200 minimum=3600" )
ns.init.catz.invalid.           0 TXT ( "name=ns1.example.com. "
      "ipv4=192.0.2.1 ipv6=2001:db8::1" )
ns.init.catz.invalid.           0 TXT ( "name=ns2.example.com. "
      "ipv4=192.0.2.2 ipv6=2001:db8::2" )
kahdkh6f.zones.catz.invalid.    0 PTR example.com.
hajhsjha.zones.catz.invalid.    0 PTR example.net.
ns.hajhsjha.zones.catz.invalid. 0 TXT "name=ns1.example.com"
ns.hajhsjha.zones.catz.invalid. 0 TXT ( "name=ns1.example.net "
      "ipv4=192.0.2.250 ipv6=2001:db8:ff::149" )
~~~~

## example.com Master File Example

~~~~
example.com.     3600 SOA ns1.example.com. hostmaster.example.com. (
                           1 14400 900 2419200 3600 )
example.com.     3600 NS   ns1.example.com.
example.com.     3600 NS   ns2.example.com.
ns1.example.com. 3600 A    192.0.2.1
ns1.example.com. 3600 AAAA 2001:db8::1
ns2.example.com. 3600 A    192.0.2.2
ns2.example.com. 3600 AAAA 2001:db8::2
~~~~

## example.net Master File Example

~~~~
example.net.     3600 SOA ns1.example.com. hostmaster.example.com. (
                            1 14400 900 2419200 3600 )
example.net.     3600 NS   ns1.example.com.
example.net.     3600 NS   ns1.example.net.
ns1.example.net. 3600 A    192.0.2.250
ns1.example.net. 3600 AAAA 2001:db8:ff::149
~~~~

# Author Notes/Thoughts

*NB: To be removed by the RFC Editor prior to publication.*

The term "Primary Master" (Section 1 {{RFC2136}} is not applicable as the primary server noted in this document likely is NOT listed in the MNAME or NS.

## Is catalog zones the right place for this?

Much consideration has been given as to whether the primary server should be consuming the/a catalog zone, rather than simply serving it to secondary servers for consumption.

It does feel a little bit like it muddies the waters between zone distribution and zone "provisioning" but:

1. In a catalog zone scenario, the catalog equally feels like the place for zone related parameters
1. It feels less like Dynamic Updates would be the right place for it, for example.
1. An API for *just* zone initialisation feels like a big thing that would likely be overkill, and probably not get implemented, and would likely be a part of a wider implementation's general nameserver configuration and operations API, which is waaaaay beyond the scope of this document, and possibly beyond standardisation.

It may be considered that this is "nameserver configuration", however, it has strong parallels in this regard to the "configuration" on secondary servers, including such considerations as to which entities are allowed to notify and/or transfer the zone, as are conveyed to those secondary servers in {{RFC9432}} DNS Catalog Zones. Indeed, much of the same configuration may be needed by or shared with the primary server for those same zones.

Implementing via an extension of catalog zones feels like it closes the gap in the end-to-end ecosystem whereby catalog zones + dynamic updates gives an end-to-end approach to the creation of a zone, its underlying master file, distribution of that zone to secondary servers, and the ongoing manipulation of records in the zone.

TODO - add more detail explaining the above, reasoning, etc...?

## DNSSEC {#dnssecConsideration}

It seems that it'd be useful to signal initial policy/settings for DNSSEC in a standardised way also, but the primary might, or might not be the signer; the signer may be downstream. But it might be very nice to be able to signal to the signer that it should create some keys, sign the zone...

15/11 - Knot acheives this by use of the group property, setting the RDATA of the group record for a member zone to the name of a pre-defined template name. However, this appears to assume that the primary server is also the signer, which may not be the case. Further reading required to see if this would facilitate a downstream (knot) server signing in-line.

## Properties

### General

Should the properties be listed in the registry as "soa.init" and "ns.init", given "init" itself is a placeholder label, and does not (currently?) take any parameters or records of its own?

TODO: Do we need to signal the initial TTL of the records being added (SOA, NS, A, AAAA)... I think so... Could specifiy with an extra key=value pair, or could leave it to pick up from the SOA

### coo Property

Are there any considerations around change of ownership that need mentioning or documenting here...?

14/11 - section added to address this; is there anything else that needs clarifying or covering?

### soa Property

Consideration was given as to whether things like SOA parameters should be individual records, but it seemed unnecessary to break them out and create the additional records.

Should we permit the property to be made up of multiple TXT records so long as a given parameter is not repeated?

Should we specify a SOA serial format? or an initial soa serial value...? If not, should we specify in the text, or leave it to the implemention, which may have a default, such as BIND's "serial-update-method"

Given that it is pretty much expected that the operator is going to start making changes to the zone via dynamic updates, it'd be reasonable to expect them to be able to set those parameters. Which does beg the question, do we need to specify soa and nameserver values at all, or just specify that the zone file is or is not to be created, and fill some template default values with the expectation that the operator would immediatly overwrite them with "correct" values...?

### ns Property

Is there a circular dependency or race condition issue here...?

# Change Log

*NB: To be removed by the RFC Editor prior to publication.*

## 00 - Initial draft

# Acknowledgments {#Acknowledgements}
{:numbered="false"}

TODO acknowledge.
