---
ipr: trust200902
stand_alone: yes
title: "Automated Certificate Management Environment (ACME) for Subdomains"
ipr: trust200902
abbrev: ACME-SUBDOMAINS
area: Security
wg: ACME Working Group
cat: std
consensus: 'yes'
docname: draft-ietf-acme-subdomains-latest
coding: utf-8
pi:
  rfcedstyle: 'yes'
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
    target: https://cabforum.org/baseline-requirements-documents/

ACME Identifier Types:
    author:
      org: IANA
    title: ACME Identifier Types
    target: https://www.iana.org/assignments/acme/acme.xhtml#acme-identifier-types

ACME Validation Methods:
    author:
      org: IANA
    title: ACME Validation Methods
    target: https://www.iana.org/assignments/acme/acme.xhtml#acme-validation-methods

--- abstract

This document specifies how Automated Certificate Management Environment (ACME) {{!RFC8555}} can be used by a client to obtain a certificate for a subdomain identifier from a certification authority. This document specifies how a client can fulfill a challenge against a parent domain but may not need to fulfill a challenge against the explicit subdomain if certification authority policy allows issuance of the subdomain certificate without explicit subdomain ownership proof.

--- middle


# Introduction

ACME {{!RFC8555}} defines a protocol that a certification authority (CA) and an applicant can use to automate the process of domain name ownership validation and X.509v3 (PKIX) {{?RFC5280}} certificate issuance. The CA is the ACME server and the applicant is the ACME client, and the client uses the ACME protocol to request certificate issuance from the server. This document outlines how ACME can be used to issue subdomain certificates, without requiring the ACME client to explicitly fulfill an ownership challenge against the subdomain identifiers - the ACME client need only fulfill an ownership challenge against a parent domain identifier.

# Terminology

{::boilerplate bcp14-tagged}

The following terms are defined in DNS Terminology {{!RFC8499}} and are reproduced here:

- Label: An ordered list of zero or more octets that makes up a
      portion of a domain name.  Using graph theory, a label identifies
      one node in a portion of the graph of all possible domain names.

- Domain Name: An ordered list of one or more labels.

- Subdomain: "A domain is a subdomain of another domain if it is
      contained within that domain.  This relationship can be tested by
      seeing if the subdomain's name ends with the containing domain's
      name."  (Quoted from {{Section 3.1 of ?RFC1034}}.) For example, in the
      host name "nnn.mmm.example.com", both "mmm.example.com" and
      "nnn.mmm.example.com" are subdomains of "example.com".  Note that
      the comparisons here are done on whole labels; that is,
      "ooo.example.com" is not a subdomain of "oo.example.com".

- Fully-Qualified Domain Name (FQDN):  This is often just a clear way
      of saying the same thing as "domain name of a node", as outlined
      above.  However, the term is ambiguous.  Strictly speaking, a
      fully-qualified domain name would include every label, including
      the zero-length label of the root: such a name would be written
      "www.example.net." (note the terminating dot).  But, because every
      name eventually shares the common root, names are often written
      relative to the root (such as "www.example.net") and are still
      called "fully qualified".  This term first appeared in {{?RFC0819}}.
      In this document, names are often written relative to the root.

The following additional terms are used in this document:

- Certification Authority (CA): An organization that is responsible for the creation, issuance, revocation, and management of Certificates. The term applies equally to both Root CAs and Subordinate CAs. Refer to {{?RFC5280}} for detailed information on Certification Authorities.

- CSR: Certificate Signing Request as defined in {{?RFC2986}}

- Parent Domain: a domain is a parent domain of a subdomain if it contains that subdomain, as per the {{!RFC8499}} definition of subdomain. For example, for the host name "nnn.mmm.example.com", both "mmm.example.com" and "example.com" are parent domains of "nnn.mmm.example.com". Note that the comparisons here are done on whole labels; that is, "oo.example.com" is not a parent domain of "ooo.example.com"

ACME {!RFC8555}} defines the following object types which are used in this document:

- Order Object: An ACME order object represents a client's request for a certificate and is used to track the progress of that order through to issuance.

- Authorization Object: An ACME authorization object represents a server's authorization for an account to represent an identifier.

- Challenge Object: An ACME challenge object represents a server's offer to validate a client's possession of an identifier in a specific way.

ACME {!RFC8555}} Section 6.3 introduces the following term which is used in this document:

- POST-as-GET Request: When a client wishes to fetch a resource from the server, then it MUST send a POST request with a signed JWS body, where the JWS body is specified in ACME {!RFC8555}} Section 6.2. ACME refers to these as "POST-as-GET" requests.

# ACME Workflow and Identifier Requirements

A typical ACME workflow for issuance of certificates is as follows:

1. client POSTs a newOrder request that contains a set of "identifiers"

2. server replies with an order object that contains a set of links to authorization object(s) and a "finalize" URI

3. client sends POST-as-GET requests to retrieve the authorization object(s), with the downloaded authorization object(s) containing the "identifier" that the client must prove that they control, and a set of links to associated challenges objects, one of which the client must fulfill

4. client proves control over the "identifier" in the authorization object by completing one of the specified challenges, for example, by publishing a DNS TXT record

5. client POSTs a CSR to the "finalize" API

6. server replies with an updated order object that includes a "certificate" URI

7. client sends POST-as-GET request to the "certificate" URI to download the certificate

ACME {{!RFC8555}} places the following restrictions on "identifiers":

- {{!RFC8555, Section 7.1.3}}: The authorizations required are dictated by server policy; there may not be a 1:1 relationship between the order identifiers and the authorizations required.

- {{!RFC8555, Section 7.1.4}}: the only type of "identifier" defined by the ACME specification is an FQDN: "The only type of identifier defined by this specification is a fully qualified domain name (type: "dns"). The domain name MUST be encoded in the form in which it would appear in a certificate."

- {{!RFC8555, Section 7.4}}: the "identifier" in the CSR request must match the "identifier" in the newOrder request: "The CSR MUST indicate the exact same set of requested identifiers as the initial newOrder request."

- {{!RFC8555, Section 8.3}}: the "identifier", or FQDN, in the authorization object must be used when fulfilling challenges via HTTP: "Construct a URL by populating the URL template ... where the domain field is set to the domain name being verified"

- {{!RFC8555, Section 8.4}}: the "identifier", or FQDN, in the authorization object must be used when fulfilling challenges via DNS: "The client constructs the validation domain name by prepending the label "_acme-challenge" to the domain name being validated."

ACME does not mandate that the "identifier" in a newOrder request matches the "identifier" in authorization objects.

The base ACME {{!RFC8555}} document only specifies the "dns" identifier type. Additional identifiers may be defined and registered in the IANA [ACME Identifier Types] registry. For example, {{?RFC8738}} specifies the "ip" identifier type. This document is only relevant for the "dns" identifier type.

Note also that ACME supports multiple different validation methods that can be used to fulfill challenges and prove ownership of identifiers. Validation methods are registered in the IANA [ACME Validation Methods] registry. This document does not mandate use of any particular validation method or methods. ACME server policy dictates which validation methods are supported. See {{acme-server-policy-considerations}} for more information on ACME server policy.

# ACME Issuance of Subdomain Certificates

As noted in the previous section, ACME does not mandate that the "identifier" in a newOrder request matches the "identifier" in authorization objects. This means that the ACME specification does not preclude an ACME server processing newOrder requests and issuing certificates for a subdomain without requiring a challenge to be fulfilled against that explicit subdomain.

ACME server policy could allow issuance of certificates for a subdomain to a client where the client only has to fulfill an authorization challenge for a parent domain of that subdomain. This allows a flow where a client proves ownership of, for example, "example.org" and then successfully obtains a certificate for "sub.example.org".

ACME server policy is out of scope of this document, however, some commentary is provided in {{acme-server-policy-considerations}}.

Clients need a mechanism to instruct the ACME server that they are requesting authorization for all subdomains subordinate to the specified domain, as opposed to just requesting authorization for an explicit domain identifier. Clients need a mechanism to do this in both newAuthz and newOrder requests. ACME servers need a mechanism to indicate to clients that authorization objects are valid for all subdomains under the specified domain. These are described in this section.

## Authorization Object

ACME ({{!RFC8555, Section 7.1.4}}) defines the authorization object. When ACME server policy allows authorization for subdomains subordinate to a domain, the server indicates this by including the "subdomainAuthAllowed" flag in the authorization object for that domain identifier:

~~~
subdomainAuthAllowed  (optional, boolean):  This field MUST be
   present and true for authorizations where ACME server policy
   allows certificates to be issued for any subdomain subordinate
   to the domain specified in the 'identifier' field of the
   authorization object.
~~~

The following example shows an authorization object for the domain `example.org` where the authorization covers the subdomains subordinate to `example.org`.

~~~
   {
     "status": "valid",
     "expires": "2015-03-01T14:09:07.99Z",

     "identifier": {
       "type": "dns",
       "value": "example.org"
     },

     "challenges": [
       {
         "url": "https://example.com/acme/chall/prV_B7yEyA4",
         "type": "http-01",
         "status": "valid",
         "token": "DGyRejmCefe7v4NfDGDKfA",
         "validated": "2014-12-01T12:05:58.16Z"
       }
     ],

     "subdomainAuthAllowed": true
   }
~~~

If the "subdomainAuthAllowed" field is not included, then the assumed default value is false.

## Pre-Authorization

The basic ACME workflow has authorization objects created reactively in response to a certificate order. ACME also allows for pre-authorization, where clients obtain authorization for an identifier proactively, outside of the context of a specific issuance. With the ACME pre-authorization flow, a client can pre-authorize for a domain once, and then issue multiple newOrder requests for certificates with identifiers in the subdomains subordinate to that domain.

ACME {{RFC8555, Section 7.4.1}} defines the "identifier" object for newAuthz requests. One additional field for the "identifier" object is defined:

~~~
subdomainAuthAllowed (optional, boolean): An ACME client sets
   this flag to indicate to the server that it is requesting an 
   authorization for the subdomains subordinate to the specified
   domain identifier value
~~~

Clients include the flag in the "identifier" object of newAuthz requests to indicate that they are requesting a subdomain authorization. In the following example newAuthz payload, the client is requesting pre-authorization for the subdomains subordinate to `example.org`.

~~~
     "payload": base64url({
       "identifier": {
         "type": "dns",
         "value": "example.org",
         "subdomainAuthAllowed": true
       }
     })
~~~

If the server is willing to allow a single authorization for the subdomains, and there is not an existing authorization object for the identifier, then it will create an authorization object and include the "subdomainAuthAllowed" flag with value of true. If the server policy does not allow creation of subdomain authorizations subordinate to that domain, the server can create an authorization object for the indicated identifier, and include the "subdomainAuthAllowed" flag with value of false. In both scenarios, handling of the pre-authorization follows the process documented in ACME {{RFC8555, Section 7.4.1}}.

## New Orders

Clients need a mechanism to optionally indicate to servers whether or not they are authorized to fulfill challenges against a parent domain for a given identifier. For example, if a client places an order for an identifier `foo.bar.example.org`, and is authorized to fulfill a challenge against the parent domains `bar.example.org` or `example.org`, then the client needs a mechanism to indicate control over the parent domains to the ACME server.

This can be achieved by adding an optional field "parentDomain" to the "identifiers" field in the order object:

~~~
parentDomain (optional, string): This is a parent domain of
   the requested identifier. The client MUST be able to fulfill
   a challenge against the parent domain.
~~~

This field specifies a parent domain of the identifier that the client has DNS control over, and is capable of fulfilling challenges against. Based on server policy, the server can choose to issue a challenge against any parent domain of the identifier up to and including the specified "parentDomain", and create a corresponding authorization object against the chosen identifier.

In the following example newOrder payload, the client requests a certificate for identifier `foo.bar.example.org` and indicates that it can fulfill a challenge against the parent domain `bar.example.org`. The server can then choose to issue a challenge against either `foo.bar.example.org` or `bar.example.org` identifiers.

~~~
"payload": base64url({
       "identifiers": [
         { "type": "dns",
           "value": "foo.bar.example.org",
           "parentDomain": "bar.example.org"  }
       ],
       "notBefore": "2016-01-01T00:04:00+04:00",
       "notAfter": "2016-01-08T00:04:00+04:00"
     })
~~~

In the following example newOrder payload, the client requests a certificate for identifier `foo.bar.example.org` and indicates that it can fulfill a challenge against the parent domain `example.org`. The server can then choose to issue a challenge against any one of `foo.bar.example.org`, `bar.example.org` or `example.org` identifiers.

~~~
"payload": base64url({
       "identifiers": [
         { "type": "dns",
           "value": "foo.bar.example.org",
           "parentDomain": "example.org"  }
       ],
       "notBefore": "2016-01-01T00:04:00+04:00",
       "notAfter": "2016-01-08T00:04:00+04:00"
     })
~~~

If the client is unable to fulfill authorizations against a parent domain, the client should not include the "parentDomain" field.

Server newOrder handling generally follows the process documented in ACME, {{Section 7.4 of RFC8555}}. If the server is willing to allow subdomain authorizations for the domain specified in "parentDomain", then it creates an authorization object against that parent domain and includes the "subdomainAuthAllowed" flag with a value of true. If the server policy does not allow creation of subdomain authorizations against that parent domain, then it can create an authorization object for the indicated identifier value, and includes the "subdomainAuthAllowed" flag with value of false.

## Directory Object Metadata

An ACME server can advertise support for authorization of subdomains by including the following boolean flag in its "ACME Directory Metadata Fields" registry:

~~~
subdomainAuthAllowed (optional, bool): Indicates if an ACME 
   server supports authorization of subdomains.
~~~

If not specified, then no default value is assumed. If an ACME server supports authorization of subdomains, it can indicate this by including this field with a value of "true".

# Illustrative Call Flow

The call flow illustrated here uses the ACME pre-authorization flow using DNS-based proof of ownership.

~~~

+--------+                   +------+     +-----+
| Client |                   | ACME |     | DNS |
+--------+                   +------+     +-----+
    |                            |           |
 STEP 1: Pre-Authorization of parent domain
    |                            |           |
    | POST /newAuthz             |           |
    | "example.org"              |           |
    |--------------------------->|           |
    |                            |           |
    | 201 authorizations         |           |
    |<---------------------------|           |
    |                            |           |
    | Publish DNS TXT            |           |
    | "example.org"              |           |
    |--------------------------------------->|
    |                            |           |
    | POST /challenge            |           |
    |--------------------------->|           |
    |                            | Verify    |
    |                            |---------->|
    | 200 status=valid           |           |
    |<---------------------------|           |
    |                            |           |
    | Delete DNS TXT             |           |
    | "example.org"              |           |
    |--------------------------------------->|
    |                            |           |
 STEP 2: Place order for sub1.example.org
    |                            |           |
    | POST /newOrder             |           |
    | "sub1.example.org"         |           |
    |--------------------------->|           |
    |                            |           |
    | 201 status=ready           |           |
    |<---------------------------|           |
    |                            |           |
    | POST /finalize             |           |
    | CSR SAN "sub1.example.org" |           |
    |--------------------------->|           |
    |                            |           |
    | 200 OK status=valid        |           |
    |<---------------------------|           |
    |                            |           |
    | POST /certificate          |           |
    |--------------------------->|           |
    |                            |           |
    | 200 OK                     |           |
    | PEM SAN "sub1.example.org" |           |
    |<---------------------------|           |
    |                            |           |
 STEP 3: Place order for sub2.example.org
    |                            |           |
    | POST /newOrder             |           |
    | "sub2.example.org"         |           |
    |--------------------------->|           |
    |                            |           |
    | 201 status=ready           |           |
    |<---------------------------|           |
    |                            |           |
    | POST /finalize             |           |
    | CSR SAN "sub2.example.org" |           |
    |--------------------------->|           |
    |                            |           |
    | 200 OK status=valid        |           |
    |<---------------------------|           |
    |                            |           |
    | POST /certificate          |           |
    |--------------------------->|           |
    |                            |           |
    | 200 OK                     |           |
    | PEM SAN "sub2.example.org" |           |
    |<---------------------------|           |
~~~

- STEP 1: Pre-authorization of parent domain

   The client sends a newAuthz request for the parent domain including the "subdomainAuthAllowed" flag in the identifier object.

~~~
   POST /acme/new-authz HTTP/1.1
   Host: example.com
   Content-Type: application/jose+json

   {
     "protected": base64url({
       "alg": "ES256",
       "kid": "https://example.com/acme/acct/evOfKhNU60wg",
       "nonce": "uQpSjlRb4vQVCjVYAyyUWg",
       "url": "https://example.com/acme/new-authz"
     }),
     "payload": base64url({
       "identifier": {
         "type": "dns",
         "value": "example.org",
         "subdomainAuthAllowed": true
       }
     }),
     "signature": "nuSDISbWG8mMgE7H...QyVUL68yzf3Zawps"
   }
~~~

   The server creates and returns an authorization object for the identifier including the "subdomainAuthAllowed" flag. The object is initially in "pending" state.

~~~
   {
     "status": "pending",
     "expires": "2015-03-01T14:09:07.99Z",

     "identifier": {
       "type": "dns",
       "value": "example.org"
     },

     "challenges": [
       {
         "url": "https://example.com/acme/chall/prV_B7yEyA4",
         "type": "dns-01",
         "status": "pending",
         "token": "DGyRejmCefe7v4NfDGDKfA",
         "validated": "2014-12-01T12:05:58.16Z"
       }
     ],

     "subdomainAuthAllowed": true
   }
~~~

   The example illustrates the client completing a DNS challenge by publishing a DNS TXT record. The client then posts to the challenge resource to inform the server that it can validate the challenge.
   
   Once the server validates the challenge by checking the DNS TXT record, the server will transition the authorization object and associated challenge object status to "valid".
   
   The call flow above illustrates the ACME server replying to the client's challenge with status of "valid" after the ACME server has validated the DNS challenge. However, the validation flow may take some time. If this is the case, the ACME server may reply to the client's challenge immediately with a status of "processing", and the client will then need to poll the authorization resource to see when it is finalized. Refer to ACME {{RFC8555, Section 7.5.1}} for more details.

- STEP 2: The client places a newOrder for `sub1.example.org`

   The client sends a newOrder request to the server and includes the subdomain identifier. Note that the identifier is a subdomain of the parent domain that has been pre-authorized in step 1. The client does not need to include the "subdomainAuthAllowed" field in the "identifier" object as it has already pre-authorized the parent domain.

~~~
   POST /acme/new-order HTTP/1.1
   Host: example.com
   Content-Type: application/jose+json

   {
     "protected": base64url({
       "alg": "ES256",
       "kid": "https://example.com/acme/acct/evOfKhNU60wg",
       "nonce": "5XJ1L3lEkMG7tR6pA00clA",
       "url": "https://example.com/acme/new-order"
     }),
     "payload": base64url({
       "identifiers": [
         { "type": "dns", "value": "sub1.example.org" }
       ],
       "notBefore": "2016-01-01T00:04:00+04:00",
       "notAfter": "2016-01-08T00:04:00+04:00"
     }),
     "signature": "H6ZXtGjTZyUnPeKn...wEA4TklBdh3e454g"
   }
~~~

As an authorization object already exists for the parent domain, the server replies with an order object with a status of "ready" that includes a link to the existing "valid" authorization object.

~~~
   HTTP/1.1 201 Created
   Replay-Nonce: MYAuvOpaoIiywTezizk5vw
   Link: <https://example.com/acme/directory>;rel="index"
   Location: https://example.com/acme/order/TOlocE8rfgo

   {
     "status": "ready",
     "expires": "2016-01-05T14:09:07.99Z",

     "notBefore": "2016-01-01T00:00:00Z",
     "notAfter": "2016-01-08T00:00:00Z",

     "identifiers": [
       { "type": "dns", "value": "sub1.example.org" }
     ],

     "authorizations": [
       "https://example.com/acme/authz/PAniVnsZcis"
     ],

     "finalize": "https://example.com/acme/order/TOlocrfgo/finalize"
   }
~~~

The client can proceed to finalize the order by posting a CSR to the "finalize" resource. The client can then download the certificate for `sub1.example.org`.

- STEP 3: The client places a newOrder for `sub2.example.org`

   The client sends a newOrder request to the server and includes the subdomain identifier. Note that the identifier is a subdomain of the parent domain that has been pre-authorized in step 1. The client does not need to include the "subdomainAuthAllowed" field in the "identifier" object as it has already pre-authorized the parent domain.

~~~
   POST /acme/new-order HTTP/1.1
   Host: example.com
   Content-Type: application/jose+json

   {
     "protected": base64url({
       "alg": "ES256",
       "kid": "https://example.com/acme/acct/evOfKhNU60wg",
       "nonce": "5XJ1L3lEkMG7tR6pA00clA",
       "url": "https://example.com/acme/new-order"
     }),
     "payload": base64url({
       "identifiers": [
         { "type": "dns", "value": "sub2.example.org" }
       ],
       "notBefore": "2016-01-01T00:04:00+04:00",
       "notAfter": "2016-01-08T00:04:00+04:00"
     }),
     "signature": "H6ZXtGjTZyUnPeKn...wEA4TklBdh3e454g"
   }
~~~

As an authorization object already exists for the parent domain, the server replies with an order object with a status of "ready" that includes a link to the existing "valid" authorization object.

~~~
   HTTP/1.1 201 Created
   Replay-Nonce: MYAuvOpaoIiywTezizk5vw
   Link: <https://example.com/acme/directory>;rel="index"
   Location: https://example.com/acme/order/TOlocE8rfgo

   {
     "status": "ready",
     "expires": "2016-01-05T14:09:07.99Z",

     "notBefore": "2016-01-01T00:00:00Z",
     "notAfter": "2016-01-08T00:00:00Z",

     "identifiers": [
       { "type": "dns", "value": "sub2.example.org" }
     ],

     "authorizations": [
       "https://example.com/acme/authz/PAniVnsZcis"
     ],

     "finalize": "https://example.com/acme/order/ROni7rdde/finalize"
   }
~~~

The client can proceed to finalize the order by posting a CSR to the "finalize" resource. The client can then download the certificate for `sub2.example.org`.

# IANA Considerations

## Authorization Object Fields Registry

The following field is added to the "ACME Authorization Object Fields" registry defined in ACME {{!RFC8555}}.

        +----------------------+------------+--------------+-----------+
        | Field Name           | Field Type | Configurable | Reference |
        +----------------------+------------+--------------+-----------+
        | subdomainAuthAllowed | boolean    | false        | RFC XXXX  |
        +----------------------+------------+--------------+-----------+

## Directory Object Metadata Fields Registry

The following field is added to the "ACME Directory Metadata Fields" registry defined in ACME {{!RFC8555}}.

         +----------------------+------------+-----------+
         | Field Name           | Field Type | Reference |
         +----------------------+------------+-----------+
         | subdomainAuthAllowed | boolean    | RFC XXXX  |
         +----------------------+------------+-----------+

# Security Considerations

This document documents enhancements to ACME {{!RFC8555}} that optimize the protocol flows for issuance of certificates for subdomains. The underlying goal of ACME for Subdomains remains the same as that of ACME: managing certificates that attest to identifier/key bindings for these subdomains. Thus, ACME for Subdomains has the same two security goals as ACME:

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

## Subdomain Determination

The {{!RFC8499}} definition of a subdomain is reproduced in {{terminology}}. When comparing domains to determine if one is a subdomain of the other, it is important to compare entire labels, and not rely on a string prefix match. Relying on string prefix matches may yield incorrect results.

## ACME Server Policy Considerations

The ACME for Subdomains and the ACME specifications do not mandate any specific ACME server or CA policies, or any specific use cases for issuance of certificates. For example, an ACME server could be used:

- to issue Web PKI certificates where the ACME server must comply with CA/Browser Forum [CAB] Baseline Requirements.

- as a Private CA for issuance of certificates within an organization. The organization could enforce whatever policies they desire on the ACME server.

- for issuance of IoT device certificates. There are currently no IoT device certificate policies that are generally enforced across the industry. Organizations issuing IoT device certificates can enforce whatever policies they desire on the ACME server.

ACME server policy could specify whether:

- issuance of subdomain certificates is allowed based on proof of ownership of a parent domain

- issuance of subdomain certificates is allowed, but only for a specific set of parent domains

- DNS based proof of ownership, or HTTP based proof of ownership, or both, are allowed

ACME server policy specification is explicitly out of scope of this document.
