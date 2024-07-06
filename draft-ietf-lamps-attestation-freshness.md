---
title: Nonce-based Freshness for Remote Attestation in Certificate Signing Requests (CSRs) for the Certification Management Protocol (CMP) and for Enrollment over Secure Transport (EST)

abbrev: Nonce Extension for CMP/EST
docname:  draft-ietf-lamps-attestation-freshness-latest
category: std

ipr: trust200902
area: Security
workgroup: LAMPS
keyword: Internet-Draft

stand_alone: yes
pi:
  rfcedstyle: yes
  toc: yes
  tocindent: yes
  sortrefs: yes
  symrefs: yes
  strict: yes
  comments: yes
  inline: yes
  text-list-symbols: -o*+
  docmapping: yes
author:
 -    ins: H. Tschofenig
      name: Hannes Tschofenig
      org: Siemens
      country: Germany
      email: hannes.tschofenig@siemens.com
      uri: https://www.siemens.com
 -    ins: H. Brockhaus
      name: Hendrik Brockhaus
      org: Siemens
      abbrev: Siemens
      street: Werner-von-Siemens-Strasse 1
      code: '80333'
      city: Munich
      country: Germany
      email: hendrik.brockhaus@siemens.com
      uri: https://www.siemens.com


normative:
  RFC2119:
  I-D.ietf-lamps-csr-attestation:
  I-D.ietf-lamps-rfc4210bis:
  RFC8295:
  RFC7030:
  RFC5280:
  RFC5785:
  RFC8615:
  RFC7159:
  RFC9482:
informative:
  RFC2986:
  RFC4211:
  RFC9334:
  I-D.tschofenig-rats-psa-token:
  I-D.ietf-rats-eat:
  TPM20:
     author:
        org: Trusted Computing Group
     title: Trusted Platform Module Library Specification, Family 2.0, Level 00, Revision 01.59
     target: https://trustedcomputinggroup.org/resource/tpm-library-specification/
     date: November 2019

--- abstract

When an end entity places attestation Evidence in a Certificate Signing
Request (CSR) it may need to demonstrate freshness of the provided
Evidence. Attestation technology today often accomplishes this task via
the help of nonces.

This document specifies how nonces are provided by an RA/CA to
the end entity for inclusion in Evidence by using the Certificate
Management Protocol (CMP) and Enrollment over Secure Transport (EST).

--- middle

#  Introduction

The management of certificates, including issuance, CA certificate provisioning, renewal, and revocation, has been automated through several standardized protocols.

The Certificate Management Protocol (CMP) {{I-D.ietf-lamps-rfc4210bis}} defines messages for X.509v3 certificate creation and management. CMP facilitates interactions between end entities and PKI management entities, such as a Registration Authority (RA) and a Certification Authority (CA). For Certificate Signing Requests (CSRs), CMP primarily uses the Certificate Request Message Format (CRMF) {{RFC4211}} but also supports PKCS#10 {{RFC2986}}.

Enrollment over Secure Transport (EST) {{RFC7030}}{{RFC8295}} is another certificate management protocol that offers a subset of CMP's features, mainly utilizing PKCS#10 for CSRs.

When an end entity requests a certificate from a Certification Authority (CA), it may need to present credible claims about the protections of the corresponding private key, such as the use of a hardware security module or the protection capabilities provided by the hardware, as well as claims about the platform itself.

To include these claims as Evidence in remote attestation, the remote attestation extension {{I-D.ietf-lamps-csr-attestation}} has been defined. It specifies how to encode Evidence produced by an Attester for inclusion in CSRs, and any certificates necessary for validating it, into CRMF or PKCS#10.
A Verifier or Relying Party might need to know the exact time when Evidence was produced to determine if the Claims are fresh and reflect the latest state of the Attester.

Current attestation technologies, such as {{TPM20}} and {{I-D.tschofenig-rats-psa-token}}, often ensure the freshness of Evidence using nonces. More details about ensuring the freshness of Evidence can be found in Section 10 of {{RFC9334}}.

Since an end entity needs to obtain a nonce from the Verifier via the Relying Party, an additional roundtrip is necessary. However, a CSR is a one-shot message. Therefore, CMP and EST are used by the end entity to obtain the nonce from the RA/CA. CMP and EST conveniently offer mechanisms to request information from the RA/CA before submitting a certification request.

Once the nonce is obtained, the end entity can call an API to the Attester, passing the nonce as an input parameter. The Attester then returns the Evidence, which is embedded into a CSR and sent back to the RA/CA in a certification request message.

{{fig-arch}} illustrates this interaction. The nonce is obtained in step (1) using the extension to CMP/EST defined in this document. The CSR extension of {{I-D.ietf-lamps-csr-attestation}} is used to convey Evidence to the RA/CA in step (2). The Verifier processes the received information and returns an Attestation Result to the Relying Party in step (3).

~~~ aasvg
                              .---------------.
                              |               |
                              |   Verifier    |
                              |               |
                              '---------------'
                                   |    ^  |    (3)
                                   |    |  | Attestation
                                   |    |  |   Result
                    (1)            |    |  v
 .------------.   Nonce in    .----|----|-----.
 |            |   CMP or EST  |    |    |     |
 |  End       |<-------------------+    |     |
 |  Entity    |               |         |     |
 |    ^       |-------------->|---------'     |
 |    |       |   Evidence    | Relying       |
 |    v       |   in CSR      | Party (RA/CA) |
 |  Attester  |     (2)       |               |
 |            |               |               |
 '------------'               '---------------'
~~~
{: #fig-arch title="Architecture with Background Check Model."}

The functionality described in this document is divided into two sections:

- {{CMP}} describes how to convey the nonce using CMP.
- {{EST}} describes the equivalent functionality for EST.

# Terminology and Requirements Language

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT",
"SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this
document are to be interpreted as described in RFC 2119 {{RFC2119}}.

The terms Attester, Relying Party, Verifier and Evidence are defined
in {{RFC9334}}. The terms end entity, certification authority (CA),
and registration authority (RA) are defined in {{RFC5280}}.

We use the terms Certificate Signing Request (CSR) and certification
request interchangeably.

# Conveying a Nonce in CMP {#CMP}

Section 5.3.19 of {{I-D.ietf-lamps-rfc4210bis}} defines the
general request message (genm) and general response (genp).
The NonceRequest payload of the genm message, which is
send by the end entity to request a nonce, contains optional
details on the length of nonce the Attester requires.  The
NonceResponse payload of the genp message, which is
sent by the CA/RA in response to a request message by the end
entity, contains the nonce.

~~~
 GenMsg:    {id-it TBD1}, NonceRequestValue
 GenRep:    {id-it TBD2}, NonceResponseValue | < absent >

 id-it-nonceRequest OBJECT IDENTIFIER ::= { id-it TBD1 }
 NonceRequestValue ::= SEQUENCE {
    len INTEGER OPTIONAL,
    -- indicates the required length of the requested nonce
    hint EvidenceHint OPTIONAL
    -- indicates which Verifier to request a nonce from
 }

 id-it-nonceResponse OBJECT IDENTIFIER ::= { id-it TBD2 }
 NonceResponseValue ::= SEQUENCE {
    nonce OCTET STRING
    -- contains the nonce of length len
    -- provided by the Verifier indicated with hint
    expiry Time OPTIONAL
    -- indicates how long the Verifier considers the
    -- nonce valid
 }
~~~

Note: The EvidenceHint structure is defined in {{I-D.ietf-lamps-csr-attestation}}.
The hint is intended for an Attester to indicate to the Relying Party
which Verifier should be invoked to request a nonce.

The use of the general request/response message exchange leads to an
extra roundtrip to convey the nonce from the CA/RA to the end entity
(and ultimately to the Attester inside the end entity).

The end entity MUST construct a NonceRequest request message to
trigger the RA/CA to transmit a nonce in the response.

[Open Issue: Should the request message indicate the remote attestation
capability of the Attester rather than relying on "policy
information"?  This may also allow the Attester (and the end entity)
to inform the the CA/RA about the type
of attestation technology/technologies available.]

If the end entity supports remote attestation and the policy requires
Evidence in a CSR to be provided, the RA/CA issues an NonceResponse
response message containing a nonce.

{{fig-cmp-msg}} showns the interaction graphically.

~~~
End Entity                                          RA/CA
==========                                      =============

              -->>-- NonceRequest -->>--
                                                Verify request
                                                Generate nonce*
                                                Create response
              --<<-- NonceResponse --<<--
                    (nonce, expiry)

Generate key pair
Generate Evidence*
Generate certification request message
              -->>-- certification request -->>--
                   +Evidence including nonce)
                                               Verify request
                                               Verify Evidence*
                                               Check for replay*
                                               Issue certificate
                                               Create response
              --<<-- certification response --<<--
Handle response
Store certificate

*: These steps require interactions with the Attester
(on the EE side) and with the Verifier (on the RA/CA side).
~~~
{: #fig-cmp-msg title="CMP Exchange with Nonce and Evidence."}

If HTTP transfer of the NonceRequest and Nonce Response message is
used, the OPTIONAL \<operation> path segment defined in Section 3.6
of {{I-D.ietf-lamps-rfc4210bis}} MAY be used.

~~~
 +------------------------+-----------------+-------------------+
 | Operation              |Operation path   | Details           |
 +========================+=================+===================+
 | Get Attestation        | getnonce        | {{CMP}}           |
 | Freshness Nonce        |                 |                   |
 +------------------------+-----------------+-------------------+
~~~

If CoAP transfer of the NonceRequest and Nonce Response message is
used, the OPTIONAL \<operation> path segment defined in Section 2.1
of {{RFC9482}} MAY be used.

~~~
 +------------------------+-----------------+-------------------+
 | Operation              |Operation path   | Details           |
 +========================+=================+===================+
 | Get Attestation        | nonce           | {{CMP}}           |
 | Freshness Nonce        |                 |                   |
 +------------------------+-----------------+-------------------+
~~~

# Conveying a Nonce in EST {#EST}

The EST client can request a nonce for its Attester from the EST
server.  This function is generally performed after requesting CA
certificates and before other EST functions.

The EST server MUST support the use of the path-prefix of "/.well-
known/" as defined in {{RFC5785}} and the registered name of "est".
Thus, a valid EST server URI path begins with
"https://www.example.com/.well-known/est".  Each EST operation is
indicated by a path-suffix that indicates the intended operation.

The following operation is defined by this specification:

~~~
 +------------------------+-----------------+-------------------+
 | Operation              |Operation path   | Details           |
 +========================+=================+===================+
 | Retrieval of a nonce   | /nonce          | {{EST}}           |
 +------------------------+-----------------+-------------------+
~~~

The operation path is appended to the path-prefix to form
the URI used with HTTP GET or POST to perform the desired EST
operation.  An example valid URI absolute path for the "/nonce"
operation is "/.well-known/est/nonce".

An EST client uses a GET or a POST depending on whether parameters
are included:

- A GET request MUST be used when the EST client does not want to
convey extra parameters.
- A POST request MUST be used when parameters, like nonce length
or a hint about the verification service, are included in the request.

~~~
 +-------------------+---------------------------------+---------------+
 | Message type      | Media type(s)                   | Reference     |
 | (per operation)   |                                 |               |
 +===================+=================================+===============+
 | Nonce Request     | N/A (for GET) or                | This section  |
 |                   | application/json (for POST)     |               |
 +===================+=================================+===============+
 | Nonce Response    | application/json                | This section  |
 |                   |                                 |               |
 +===================+=================================+===============+
~~~

To retrieve a nonce using a GET, the EST client would use the following
HTTP request-line:

~~~
GET /.well-known/est/nonce HTTP/1.1
~~~

To retrieve a nonce by specifying the size of the requested nonce
(and/or by including a hint about the Verification service) a POST
message is used, as shown below:

~~~
POST /.well-known/est/nonce HTTP/1.1
Content-Type: application/json
{
  "len": 8,
  "hint": "https://example.com"
}
~~~

The payload in a POST request MUST be of content-type of "application/json"
and MUST contain a JSON object {{RFC7159}} with the member "len" and/or
"hint". The value of the "len" member indicates the length of the requested
nonce value in bytes. The "hint" member contains either be a rfc822Name, a
dNSName, a uri, or a test value (based on the definition in the EvidenceHint
structure from {{I-D.ietf-lamps-csr-attestation}}).

The EST server MAY request HTTP-based client authentication, as
explained in Section 3.2.3 of {{RFC7030}}.

If the request is successful, the EST server response MUST contain
a HTTP 200 response code with a content-type of "application/json"
and a JSON object  {{RFC7159}} with the member nonce. The expiry
member is optional and indicates the time the nonce is considered
valid. After the expiry time is expired, the session is likely
garbage collected.

Below is a non-normative example of the response given by the EST server:

~~~
HTTP/1.1 200 OK
Content-Type: application/json

{
    "nonce": "MTIzNDU2Nzg5MDEyMzQ1Njc4OTAxMjM0NTY3ODkwMTI=",
    "expiry": "2031-10-12T07:20:50.52Z"
}
~~~

[Open Issue: Should we also register a content type for use with
EST over CoAP where the nonce and the expiry field is encoded in
a CBOR structure?]

# Nonce Processing Guidelines

When the RA/CA is requested to transmit a nonce to an
end entity then it interacts with the Verifier.
The Verifier is, according to the IETF RATS architecture {{RFC9334}}, "a role
performed by an entity that appraises the validity of Evidence about
an Attester and produces Attestation Results to be used by a Relying
Party." Since the Verifier validates Evidence it is also the source
of the nonce to check for replay.

[Open Issue: Who generates the nonce? According to Mike St.John, the Relying Party can generate the nonce and provide it along with the evidence to the Verifier. However, according to the RATS architecture, the nonce is generated by the Verifier.]

The nonce value MUST contain a random byte sequence whereby the length
depends on the used remote attestation technology as specific nonce
length may be required by the end entity.
Since the nonce is relayed with the RA/CA, it MUST be copied to
the respective structure, as described in {{EST}} and {{CMP}}, for
transmission to the Attester.

For example, the PSA attestation token {{I-D.tschofenig-rats-psa-token}}
supports nonces of length 32, 48 and 64 bytes. Other attestation
technologies use nonces of similar length. The assumption in this
specification is that the RA/CA have either out-of-band knowledge or
in-band knowledge by using the len field in the NonceRequest, knowledge
about the nonce length required for the attestation technology used by
the end entity. The nonces of incorrect length will cause the remote
attestation protocol to fail.

When the end entity requests a nonce, the RA/CA SHOULD send a nonce in the
response. If a specific lengths of the nonce was requested, the RA/CA
should provide a nonce of the requested size. The end entity MUST use the
received nonce if remote attestation is available and supports the
received nonce length. The end entity may truncate or pad the received
nonce to the required length.

While the semantics of the attestation API and the software/hardware
architecture is out-of-scope of this specification, the API will return
Evidence from the Attester in a format specific to the attestation technology
utilized. The encoding of the returned evidence varies but will be placed
inside the CSR, as specified in {{I-D.ietf-lamps-csr-attestation}}. The
software creating the CSR will not have to interpret the Evidence format
- it is treated as an opaque blob. It is important to note that the
nonce is carried in the Evidence, either implicitly or explicitly, and
it MUST NOT be conveyed in CSR structures outside the Evidence payload.

The processing of the CSR containing Evidence is described in
{{I-D.ietf-lamps-csr-attestation}}. Note that the issued certificates
do not contain the nonce, as explained in
{{I-D.ietf-lamps-csr-attestation}}.

#  IANA Considerations

This document adds new entries to the "CMP Well-Known URI Path Segments"
registry defined in {{RFC8615}}.

~~~
 +----------------+---------------------------+-----------------+
 | Path Segment   | Description               | Reference       |
 +================+===========================+=================+
 | getnonce       | Get Attestation Freshness | {{cmp}}         |
 |                | Nonce over HTTP           |                 |
 +----------------+---------------------------+-----------------+
 | nonce          | Get Attestation Freshness | {{cmp}}         |
 |                | Nonce over CoAP           |                 |
 +----------------+---------------------------+-----------------+
~~~

[Open Issue: Register path segments for EST]

#  Security Considerations

This specification defines how to obtain a nonce via CMP and EST and assumes that the nonce does not require confidentiality protection, without impacting the security property of the remote attestation protocol. {{RFC9334}} defines the IETF remote attestation architecture and discusses nonce-based freshness in great detail.

Section 8.4 of {{I-D.ietf-rats-eat}} places requirements on the randomness and privacy of the nonce generation for use with an Entity Attestation Token (EAT). These requirements, adopted by attestation technologies such as the PSA attestation token {{I-D.tschofenig-rats-psa-token}}, are of general utility:

The nonce MUST have at least 64 bits of entropy.
To avoid conveying privacy-related information in the nonce, it should be derived using a salt from a true and reliable random number generator or another source of randomness.
Each specification of an attestation technology provides guidance for replay protection with nonces (and other techniques). This document defers specific guidance to the respective specifications.

For the use of Evidence in a CSR, the security considerations of {{I-D.ietf-lamps-csr-attestation}} are relevant to this document.

--- back

# Acknowledgments

We would like to thank Russ Housley, Thomas Fossati, Watson Ladd, Ionut Mihalcea,
Carl Wallace, and Michael StJohns for their review comments.
