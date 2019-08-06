---

title: "ACME for Subdomains"
abbrev: ACME-SUBDOMAINS
docname: draft-friel-acme-subdomains-latest
category: std

stand_alone: yes
pi: [toc, sortrefs, symrefs]

author:
 -
    ins: O. Friel
    name: Owen Friel
    org: Cisco
    email: ofriel@cisco.com
 -
    ins: R. Barnes
    name: Richard Barnes
    org: Cisco
    email: rlb@ipv.sx

--- abstract

This document outlines how ACME can be used by a client to obtain a certificate for a subdomain identifier from a certificate authority. The client has fulfilled a challenge against a parent domain but does not need to fulfill a challenge against the explicit subdomain, and certificate authority policy allows issuance of the subdomain certificate without explicit subdomain ownership proof.

--- middle


# Introduction

ACME {{?RFC8555}} defines a protocol that a certificate authority (CA) and an applicant can use to automate the process of domain name ownership validation and X.509 (PKIX) certificate issuance. The protocol is rich and flexible and enables multiple use cases that are not immediately obvious from reading the specification.

This document explicitly outlines how ACME can be used to issue subdomain certificates, without requiring the ACME client to explicitly fulfill an ownership challenge against the subdomain identifiers - the ACME client need only fulfill an ownership challenge against a parent domain identifier.

# Terminology

   The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT",
   "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and
   "OPTIONAL" in this document are to be interpreted as described in BCP
   14 {{?RFC2119}} {{?RFC8174}} when, and only when, they appear in all
   capitals, as shown here.
   

The following terms are used in this document:

- CA: Certificate Authority

- CSR: Certificate Signing Request

- FQDN: Fully Qualified Domain Name

# ACME Workflow and Identifier Requirements

A typical ACME workflow for issuance of certificates is as follows:

1. client POSTs a newOrder request that contains a set of "identifiers"

2. server replies with a set of "authorizations" and a "finalize" URI

3. client sends POST-as-GET requests to retrieve the "authorizations", with the downloaded "authorization" object(s) containing the "identifier" that the client must prove control of

4. client proves control over the "identifier" in the "authorization" object by completing the specified challenge, for example, by publishing a DNS TXT record

5. client POSTs a CSR to the "finalize" API

ACME places the following restrictions on "identifiers":

- section 7.1.4: the only type of "identifier" defined by the ACME specification is a fully qualified domain name: "The only type of identifier defined by this specification is a fully qualified domain name (type: "dns"). The domain name MUST be encoded in the form in which it would appear in a certificate."

- Section 7.4: the "identifier" in the CSR request must match the "identifier" in the newOrder request: "The CSR MUST indicate the exact same set of requested identifiers as the initial newOrder request."

- Sections 8.3: the "identifier", or FQDN, in the "authorization" object must be used when fulfilling challenges via HTTP: "Construct a URL by populating the URL template ... where the domain field is set to the domain name being verified"

- Section 8.4: the "identifier", or FQDN, in the "authorization" object must be used when fulfilling challenges via DNS: "The client constructs the validation domain name by prepending the label "_acme-challenge" to the domain name being validated."

ACME does not mandate that the "identifier" in a newOrder request matches the "identifier" in "authorization" objects.

# ACME Issuance of Subdomain Certificates

As noted in the previous section, ACME does not mandate that the "identifier" in a newOrder request matches the "identifier" in "authorization" objects. This means that the ACME specification does not preclude an ACME server processing newOrder requests and issuing certificates for a subdomain without requiring a challenge to be fulfilled against that explicit subdomain. ACME server policy could allow issuance of certificates for a subdomain to a client where the client only has to fulfill an authorization challenge for the parent domain.

This allows a flow where a client proves ownership of, for example, "example.com" and then successfully obtains a certificate for "sub.example.com". The ACME pre-authorization flow makes most sense for this use case, and that is what is illustrated in the following call flow.

The client could pre-authorize for the parent domain once, and then issue multiple newOrder requests for certificates for multiple subdomains. This call flow illustrates the client only placing one newOrder request.

The call flow illustrates the DNS-based proof of ownershp mechanism, but the subdomain workflow is equally valid for HTTP based proof of ownership.

~~~

+--------+             +------+     +-----+
| Client |             | ACME |     | DNS |
+--------+             +------+     +-----+
    |                      |           |
 STEP 1: Pre-Authorization of parent domain
    |                      |           |
    | POST /newAuthz       |           |
    | "example.com"        |           |
    |--------------------->|           |
    |                      |           |
    | 201 authorizations   |           |
    |<---------------------|           |
    |                      |           |
    | Publish DNS TXT      |           |
    | "example.com"        |           |
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
    | "example.com"        |           |
    |--------------------------------->|
    |                      |           |
 STEP 2: Place order for subdomain
    |                      |           |
    | POST /newOrder       |           |
    | "sub.example.com"    |           |
    |--------------------->|           |
    |                      |           |
    | 201 status=ready     |           |
    |<---------------------|           |
    |                      |           |
    | POST /finalize       |           |
    | CSR "sub.example.com"|           |
    |--------------------->|           |
    |                      |           |
    | 200 OK status=valid  |           |
    |<---------------------|           |
    |                      |           |
    | POST /certificate    |           |
    |--------------------->|           |
    |                      |           |
    | 200 OK               |           |
    | PKI "sub.example.com"|           |
    |<---------------------|           |

~~~

[[ TODO ]]

What happens if client sends newAuthz for a subdomain and the CA doesnt require it?
What does existing ACME do here? e.g. what happens if client sends an authz req multiple times?
What is the non pre-authz flow?

Include the baseline text.



# IANA Considerations

None.

# Security Considerations 

[todo]

--- back

# Comments
