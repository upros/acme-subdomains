---
stand_alone: yes
title: "ACME for Subdomains"
abbrev: ACME-SUBDOMAINS
docname: draft-friel-acme-subdomains-04
cat: std
coding: utf-8
pi:
  strict: 'yes'
  toc: 'yes'
  tocdepth: '2'
  symrefs: 'yes'
  sortrefs: 'yes'
  compact: 'no'
  subcompact: 'no'
  comments: 'yes'
  inline: 'yes'
author:
 -  ins: O. Friel
    name: Owen Friel
    org: Cisco
    email: ofriel@cisco.com
 -  ins: R. Barnes
    name: Richard Barnes
    org: Cisco
    email: rlb@ipv.sx
 -  ins: T. Hollebeek
    name: Tim Hollebeek
    org: DigiCert
    email: tim.hollebeek@digicert.com
 -  ins: M. Richardson
    name: Michael Richardson
    org: Sandelman Software Works
    email: mcr+ietf@sandelman.ca
informative:
  CAB:
    author:
      org: CA/Browser Forum
    title: Baseline Requirements for the Issuance and Management of Publicly-Trusted Certificates
    target: https://cabforum.org/wp-content/uploads/CA-Browser-Forum-BR-1.7.1.pdf

--- abstract

This document outlines how ACME can be used by a client to obtain a certificate for a subdomain identifier from a certification authority. The client has fulfilled a challenge against a parent domain but does not need to fulfill a challenge against the explicit subdomain as certificate policy allows issuance of the subdomain certificate without explicit subdomain ownership proof.

--- middle


# Introduction

ACME {{?RFC8555}} defines a protocol that a certification authority (CA) and an applicant can use to automate the process of domain name ownership validation and X.509v3 (PKIX) {{?RFC5280}} certificate issuance. This document outlines how ACME can be used to issue subdomain certificates, without requiring the ACME client to explicitly fulfill an ownership challenge against the subdomain identifiers - the ACME client need only fulfill an ownership challenge against a parent domain identifier.

# Terminology

   The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT",
   "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and
   "OPTIONAL" in this document are to be interpreted as described in BCP
   14 {{?RFC2119}} {{?RFC8174}} when, and only when, they appear in all
   capitals, as shown here.
   
The following terms are defined in the CA/Browser Forum Baseline Requirements [CAB] and are reproduced here:

- Base Domain Name: The portion of an applied-for FQDN that is the first domain name node left of a registry-controlled or public suffix plus the registry-controlled or public suffix (e.g. “example.co.uk” or “example.com”). For FQDNs where the right-most domain name node is a gTLD having ICANN Specification 13 in its registry agreement, the gTLD itself may be used as the Base Domain Name.

- ADN: Authorization Domain Name. The Domain Name used to obtain authorization for certificate issuance for a given FQDN.

- Domain Name: The label assigned to a node in the Domain Name System

- Domain Namespace: The set of all possible Domain Names that are subordinate to a single node in the Domain Name System

The following terms are used in this document:

- CA: Certification Authority

- CSR: Certificate Signing Request

- FQDN: Fully Qualified Domain Name

- Parent Domain: a node in the Domain Name System that has a Domain Name

- Subdomain: a Domain Name that is in the Domain Namespace of a given Parent Domain

# ACME Workflow and Identifier Requirements

A typical ACME workflow for issuance of certificates is as follows:

1. client POSTs a newOrder request that contains a set of "identifiers"

2. server replies with a set of "authorizations" and a "finalize" URI

3. client sends POST-as-GET requests to retrieve the "authorizations", with the downloaded "authorization" object(s) containing the "identifier" that the client must prove that they control

4. client proves control over the "identifier" in the "authorization" object by completing the specified challenge, for example, by publishing a DNS TXT record

5. client POSTs a CSR to the "finalize" API

6. server replies with an updated order object that includes a "certificate" URI

7. client sends POST-as-GET request to the "certificate" URI to download the certificate

ACME places the following restrictions on "identifiers":

- section 7.1.3: The authorizations required are dictated by server policy; there may not be a 1:1 relationship between the order identifiers and the authorizations required.

- section 7.1.4: the only type of "identifier" defined by the ACME specification is a fully qualified domain name: "The only type of identifier defined by this specification is a fully qualified domain name (type: "dns"). The domain name MUST be encoded in the form in which it would appear in a certificate."

- Section 7.4: the "identifier" in the CSR request must match the "identifier" in the newOrder request: "The CSR MUST indicate the exact same set of requested identifiers as the initial newOrder request."

- Sections 8.3: the "identifier", or FQDN, in the "authorization" object must be used when fulfilling challenges via HTTP: "Construct a URL by populating the URL template ... where the domain field is set to the domain name being verified"

- Section 8.4: the "identifier", or FQDN, in the "authorization" object must be used when fulfilling challenges via DNS: "The client constructs the validation domain name by prepending the label "_acme-challenge" to the domain name being validated."

ACME does not mandate that the "identifier" in a newOrder request matches the "identifier" in "authorization" objects.

# ACME Issuance of Subdomain Certificates

As noted in the previous section, ACME does not mandate that the "identifier" in a newOrder request matches the "identifier" in "authorization" objects. This means that the ACME specification does not preclude an ACME server processing newOrder requests and issuing certificates for a subdomain without requiring a challenge to be fulfilled against that explicit subdomain.

ACME server policy could allow issuance of certificates for a subdomain to a client where the client only has to fulfill an authorization challenge for a parent domain of that subdomain. This allows a flow where a client proves ownership of, for example, "example.org" and then successfully obtains a certificate for "sub.example.org".

ACME server policy is out of scope of this document, however some commentary is provided in {{acme-server-policy-considerations}}.

## ACME Challenge Type

ACME for subdomains is restricted for use with "dns-01" challenges. If a server policy allows a client to fulfill a challenge against a parent ADN of a requested certificate FQDN identifier, then the server MUST issue a "dns-01" challenge against that parent ADN.

## Parent Domain Authorization Indication

Clients need a mechanism to optionally indicate to servers whether or not they are authorized to fulfill challenges against parent ADNs for a given identifier FQDN. For example, if a client places an order for an identifier `foo.bar.example.org`, and is authorized to update DNS TXT records against the parent ADNs `bar.example.org` or `example.org`, then the client needs a mechanism to indicate control over the parent ADNs to the ACME server.

This can be achieved by adding an optional field "authorizedNamespace" to the "identifiers" field in the order object. This field specifies the ADN of the Domain Namespace that the client has DNS control over, and is capable of fulfilling challenges against. The server can choose to issue a challenge against any parent domain of the identifier in the Domain Namespace up to and including the specified "authorizedNamespace".

In the following example, the client requests a certificate for identifier `foo.bar.example.org` and indicates that it can fulfill a challenge against the parent ADN and the Domain Namespace under `bar.example.org`. The server can then choose to issue a challenge against against either `foo.bar.example.org` or `bar.example.org`.

~~~
  {
    "identifiers": [
      { "type": "dns", "value": "foo.bar.example.org", "authorizedNamespace": "bar.example.org" }
    ],
    "notBefore": "2016-01-01T00:04:00+04:00",
    "notAfter": "2016-01-08T00:04:00+04:00"
   }
~~~

In the following example, the client requests a certificate for identifier `foo.bar.example.org` and indicates that it can fulfill a challenge against the parent ADN and the Domain Namespace under `example.org`. The server can then choose to issue a challenge against against any one of `foo.bar.example.org`, `bar.example.org` or `example.org`.

~~~
  {
    "identifiers": [
      { "type": "dns", "value": "foo.bar.example.org", "authorizedNamespace": "example.org" }
    ],
    "notBefore": "2016-01-01T00:04:00+04:00",
    "notAfter": "2016-01-08T00:04:00+04:00"
   }
~~~

If the client is unable to fulfill authorizations against parent ADNs, the client should not include the "authorizedNamespace" field.

## Pre-Authorization

The standard ACME workflow has authorization objects created reactively in response to a certificate order. ACME also allows for pre-authorization, where clients obtain authorization for an identifier proactively, outside of the context of a specific issuance. This document allows for both workflows, and {{neworder-and-newauthz-handling}} outlines how the ACME server handles newOrder and newAuthz requests for both workflows.

It may make sense to use the ACME pre-authorization flow for the subdomain use case, however, that is an operator implementation and deployment decision. With the ACME pre-authorization flow, the client could pre-authorize for the parent domain once, and then issue multiple newOrder requests for certificates for multiple subdomains.

## Illustrative Call Flow

The call flow illustrated here uses the ACME pre-authorization flow. The call flow also illustrates the DNS-based proof of ownership mechanism, but the subdomain workflow is equally valid for HTTP based proof of ownership.

~~~

+--------+             +------+     +-----+
| Client |             | ACME |     | DNS |
+--------+             +------+     +-----+
    |                      |           |
 STEP 1: Pre-Authorization of parent domain
    |                      |           |
    | POST /newAuthz       |           |
    | "example.org"        |           |
    |--------------------->|           |
    |                      |           |
    | 201 authorizations   |           |
    |<---------------------|           |
    |                      |           |
    | Publish DNS TXT      |           |
    | "example.org"        |           |
    |--------------------------------->|
    |                      |           |
    | POST /challenge      |           |
    |--------------------->|           |
    |                      | Verify    |
    |                      |---------->|
    | 200 status=valid     |           |
    |<---------------------|           |
    |                      |           |
    | Delete DNS TXT       |           |
    | "example.org"        |           |
    |--------------------------------->|
    |                      |           |
 STEP 2: Place order for subdomain
    |                      |           |
    | POST /newOrder       |           |
    | "sub.example.org"    |           |
    |--------------------->|           |
    |                      |           |
    | 201 status=ready     |           |
    |<---------------------|           |
    |                      |           |
    | POST /finalize       |           |
    | CSR "sub.example.org"|           |
    |--------------------->|           |
    |                      |           |
    | 200 OK status=valid  |           |
    |<---------------------|           |
    |                      |           |
    | POST /certificate    |           |
    |--------------------->|           |
    |                      |           |
    | 200 OK               |           |
    | PKI "sub.example.org"|           |
    |<---------------------|           |

~~~

## newOrder and newAuthz Handling

Servers may consider validation of a parent domain sufficient authorization for a subdomain. If a server has such a policy and a client has already fulfilled an authorization challenge for the parent domain then:

- If the client submits a newAuthz request for a subdomain: The server MUST return status 200 (OK) response. The response body is the existing authorization object for the parent domain with status set to "valid".

- If the client submits a newOrder request for a subdomain: The server MUST return a 201 (Created) response. The response body is an order object with status set to "ready" and links to the unexpired authorizations against the parent domain.

If a server has such a policy and a client has not fulfilled an authorization challenge for the parent domain then:

- If the client submits a newAuthz request for a subdomain: The server MUST return a status 201 (Created) response. If the client indicates that it has control over the parent domains by including the "parentDomainAuthorization" value of `true`, then the response body is a newly created authorization object, and server policy dictates whether the authorization object is for the subdomain identifier, or one of the parent domains. If the client indicates that it does not have control over the parent domain by including the "parentDomainAuthorization" value of `false`, then server MUST return an authorization object for the specified identifier, and not for a parent domain.

- If the client submits a newOrder request for a subdomain: The server MUST return a status 201 (Created) response. If the client indicates that it has control over the parent domains by including the "parentDomainAuthorization" value of `true`, then the response body is an order object with status set to "pending" and links to newly created authorizations object, and server policy dictates whether the authorization object is for the subdomain identifier, or one of the parent domains. If the client indicates that it does not have control over the parent domain by including the "parentDomainAuthorization" value of `false`, then server MUST return an authorization object for the specified identifier, and not for a parent domain.

## Examples

In order to illustrate subdomain behaviour, let us assume that a client wishes to get certificates for subdomain identifiers "sub0.example.org", "sub1.example.org" and "sub2.example.org" under parent domain "example.org", and CA policy allows certificate issuance of these subdomain identifiers while only requiring the client to fulfill an ownership challenge for parent domain "example.org". Let us also assume that the client has not yet proven ownership of parent domain "example.org".

1. The client POSTs a newOrder request for identifier "sub0.example.org" and includes a "parentDomainAuthorization" value of `true`

    The server creates an authorization object for identifier "example.org". The server replies with a 201 (Created) response. The response body is an order object with status set to "pending" and a link to newly created authorization object against the parent domain "example.org". Therefore, the server is instructing the client to fulfill a challenge against domain identifier "example.org" in order to obtain a certificate including identifier "sub0.example.org".

    The client completes the challenge for "example.org", POSTs a CSR to the order finalize URI, and downloads the certificate.

2. The client POSTs a newOrder request for identifier "sub1.example.org"

    The server replies with a 201 (Created) response. The response body is an order object with status set to "ready" and a link to the unexpired authorization against the parent domain "example.org". 

    The client POSTs a CSR to the order finalize URI, and downloads the certificate.

3. The client POSTs a newAuthz request for identifier "sub2.example.org"

    The server replies with a 200 (OK) response. The response body is the previously created authorization object for "example.org" with status set to "valid".

# Resource Enhancements

This document defines enhancements to the authorization and directory objects.

## Authorization Object

If an ACME server allows issuance of certificates for subdomains of a parent domain, then the authorization object for the parent domain MUST include the optional "includeSubDomains" field, with a value of true.

The structure of an ACME authorization resource is enhanced to include the following optional field:

   includeSubDomains (optional, boolean):  This field MUST be present and true
      for authorizations where ACME server policy allows certificates
      to be issued for subdomains of the identifier in the authorization
      object without explicit authorization of the subdomain

## Identifier Object

[TODO]: NEED TO UPDATE

The "Identifier" object which can be included in requests to newAuthz resource, and in order objects, is enhanced to include the following optional field:

   parentDomainAuthorization (optional, boolean):  Clients include this
      field to indicate if they have control over parent domains for the
      specified identifier and are able to fulfill challenges against
      parent domains of the identifier. If not specified, then no default
      value is assumed
      
## Directory Object Metadata

An ACME server can advertise support of issuance of subdomain certificates by including the boolean field "includeSubDomainsAuthorization" in its "ACME Directory Metadata Fields" registry. If not specified, then no default value is assumed. If an ACME server supports issuance of subdomain certificates, it can indicate this by including this field with a value of "true".

# IANA Considerations

## Authorization Object Fields Registry 

The following field is added to the "ACME Authorization Object Fields" registry defined in ACME {{?RFC8555}}.

        +-------------------+------------+--------------+-----------+
        | Field Name        | Field Type | Configurable | Reference |
        +-------------------+------------+--------------+-----------+
        | includeSubDomains | boolean    | false        | RFC XXXX  |
        +-------------------+------------+--------------+-----------+

## Directory Object Metadata Fields Registry

The following field is added to the "ACME Directory Metadata Fields" registry defined in ACME {{?RFC8555}}.

         +--------------------------------+------------+-----------+
         | Field Name                     | Field Type | Reference |
         +--------------------------------+------------+-----------+
         | includeSubDomainsAuthorization | boolean    | RFC XXXX  |
         +--------------------------------+------------+-----------+

# Security Considerations 

This document documents enhancements to ACME {{?RFC8555}} that optimize the protocol flows for issuance of certificates for subdomains. The underlying goal of ACME for Subdomains remains the same as that of ACME: managing certificates that attest to identifier/key bindings for these subdomains. Thus, ACME for Subdomains has the same two security goals as ACME:

1. Only an entity that controls an identifier can get an authorization for that identifier

2. Once authorized, an account key's authorizations cannot be improperly used by another account

ACME for Subdomains makes no changes to:

- account or account key management

- ACME channel establishment, security mechanisms or threat model

- Validation channel establishment, security mechanisms or threat model

Therefore, all Security Considerations in ACME in the following areas are equally applicable to ACME for Subdomains:

- Threat Model

- Integrity of Authorizations

- Denial-of-Service Considerations

- Server-Side Request Forgery

- CA Policy Considerations 

Some additional comments on ACME server policy are given in the following section.

## ACME Server Policy Considerations

The ACME for Subdomains and the ACME specifications do not mandate any specific ACME server or CA policies, or any specific use cases for issuance of certificates. For example, an ACME server could be used:

- to issue Web PKI certificates where the ACME server must comply with CA/Browser Forum [CAB] Baseline Requirements. 

- as a Private CA for issuance of certificates within an organisation. The organisation could enforce whatever policies they desire on the ACME server.

- for issuance of IoT device certificates. There are currently no IoT device certificate policies that are generally enforced across the industry. Organizations issuing IoT device certificates can enforce whatever policies they desire on the ACME server.

ACME server policy could specify whether:

- issuance of subdomain certificates is allowed based on proof of ownership of a parent domain

- issuance of subdomain certificates is allowed, but only for a specific set of parent domains

- whether DNS based proof of ownership, or HTTP based proof of ownership, or both, are allowed

ACME server policy specification is explicitly out of scope of this document. For reference, extracts from CA/Browser Forum Baseline Requirements are given in the appendices.

--- back

# CA Browser Forum Baseline Requirements Extracts

The CA/Browser Forum Baseline Requirements [CAB] allow issuance of subdomain certificates where authorization is only required for a parent domain. Baseline Requirements version 1.7.1 states:

- Section: "1.6.1 Definitions": Authorization Domain Name: The Domain Name used to obtain authorization for certificate issuance for a given FQDN. The CA may use the FQDN returned from a DNS CNAME lookup as the FQDN for the purposes of domain validation. If the FQDN contains a wildcard character, then the CA MUST remove all wildcard labels from the left most portion of requested FQDN. The CA may prune zero or more labels from left to right until encountering a Base Domain Name and may use any one of the intermediate values for the purpose of domain validation.

- Section: "3.2.2.4.6 Agreed-Upon Change to Website": Once the FQDN has been validated using this method, the CA MAY also issue Certificates for other FQDNs that end with all the labels of the validated FQDN. This method is suitable for validating Wildcard Domain Names.

- Section: "3.2.2.4.7 DNS Change": Once the FQDN has been validated using this method, the CA MAY also issue Certificates for other FQDNs that end with all the labels of the validated FQDN. This method is suitable for validating Wildcard Domain Names.
