---
###
# Discovery of STIR Out‑of‑Band Call Placement Services
###
title: "Discovery of STIR Out‑of‑Band Call Placement Services"
abbrev: "Discovery of STIR OOB CPS"
category: std

docname: draft-stir-oob-discovery
submissiontype: IETF  # also: "independent", "editorial", "IAB", or "IRTF"
number:
date:
consensus: true
v: 3
area: AREA
workgroup: WG Working Group
keyword:
- stir
- certificates
- delegate certificates
- oob
venue:
  group: "Secure Telephone Identity Revisited"
  type: "Working Group"
  mail: "stir@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/stir/"
  github: "appliedbits/draft-stir-oob-discovery"
  latest: "https://github.com/appliedbits/draft-stir-oob-discovery"

author:
 -
    fullname: Your Name Here
    organization: Your Organization Here
    email: your.email@example.com

normative:

informative:

...

--- abstract

This document defines a Discovery Service for STIR Out-of-Band (OOB) Call Placement Services (CPS).

The Discovery Service enables Authentication Services (AS) and Verification Services (VS) to quickly determine which CPS is responsible for a given telephone number (TN) or Service Provider Code (SPC), allowing retrieval of PASSporTs even when SPI Identity headers are removed by non-IP or hybrid network segments.

The Discovery Service leverages the CPS URI certificate extension defined in [I-D.draft-stir-cert-cps]](https://github.com/appliedbits/draft-stir-cert-cps/blob/main/draft-stir-cert-cps.md), which allows Delegate Certificates to embed an HTTPS URI for the CPS serving the TNs or SPCs covered by the certificate. A built-in monitoring component continuously watches STI Certificate Transparency (STI-CT) logs for new or updated certificates containing this extension, automatically registering TN->CPS and SPC->CPS mappings. These mappings are exposed via a simple REST API for fast lookups, eliminating the need for bilateral provisioning and enabling transparent, cryptographically verifiable CPS discovery.


--- middle

# Introduction

The Discovery Service specified in this document provides a scalable, cryptographically verifiable way to determine which CPS is responsible for a given TN or SPC when performing STIR/SHAKEN Out‑of‑Band call authentication as described in {{RFC8816}}.

The Discovery Service leverages the CPS URI certificate extension defined in [I-D.draft-stir-cert-cps]](https://github.com/appliedbits/draft-stir-cert-cps/blob/main/draft-stir-cert-cps.md). This extension allows Delegate Certificates to embed an HTTPS URI that points to the CPS serving the TNs or SPCs covered by that certificate’s TNAuthList. When these certificates are published to the STI‑CT logs, their CPS URIs become immediately visible and auditable by the ecosystem.

The Discovery Service continuously monitors STI‑CT logs to learn new and updated certificate registrations, automatically extracting the TN → CPS and SPC → CPS mappings from any CPS URIs present. These mappings are stored in the Discovery Service and exposed via a simple REST API that supports fast, one call lookups by Authentication Services and Verification Services. This means that:

- Originating AS implementations can find the correct CPS endpoint with a single lookup instead of relying on bilateral provisioning or static configuration.
- Terminating VS implementations can verify PASSporTs published to the correct CPS, even when SIP Identity headers are stripped by non‑IP or hybrid network segments.
- The ecosystem gains continuous monitoring of certificate issuance and CPS endpoint registrations, ensuring rapid detection of misconfigurations or mis‑issuance.

Combination of CPS URI certificate extensions with STI‑CT log monitoring and a simple REST query interface, the Discovery Service enables zero‑touch OOB CPS discovery while remaining fully backward‑compatible with existing OOB publish and retrieve APIs. Verifiers and originators only need to add one new lookup step to gain reliable and scale discovery of OOB PASSporTs.

# Architectural Overview

## Design Goals

- Open registration - Any authenticated holder of RTU evidence can publish a CPS for TNs it controls.
- Zero manual peering - Originators and receivers discover the CPS with one lookup and no bilateral agreement.
- Cryptographic binding - The discovery record and the CPS‑URI certificate extension are both verifiable against the same RTU trust anchor.
- Transparency - All CPS bindings are logged in STI‑CT and continuously monitored for mis‑issuance.
- Backward compatibility - No changes to OOB publish/retrieve APIs. Discovery is purely additive.

## Components

The Discovery Service architecture provides simple and reliable mechanisms for any STIR/SHAKEN entity to find the correct CPS for a TN or SPC.

Following diagram shows the high‑level components and their relationships.

~~~~~~~~~~~~~
                +--------------+     Cert w/ CPS‑URI
          +---->|  STI‑CT Logs |<=================+
          |     +------+-------+                  |
          |            ^                          |
     1. Submit DC      |   CT Monitor             |
        w/SPC URIs     |  (auto‑registers CPS)    |
          |            |                          v
   +-----------+       |   +-------------------+  |  2. Signed
   | Enterprise|       +---| Discovery Service |<-+--- Advertisement
   +-----------+           +---------+---------+
                                     |
                                     | 3. Lookup
                                     v
                             +-------+-------+
                             |  Originating  |
                             |      AS       |
                             +-------+-------+
                                     |
                              1. Publish PASSporT
                                     |
                                     v
                             +-------+-------+
                             |      CPS      |
                             +-------+-------+
                                     |
                              2. Retrieve PASSporT
                                     |
                                     v
                             +-------+-------+
                             |  Terminating  |
                             |      VS       |
                             +---------------+
~~~~~~~~~~~~~

### Call Placement Service (CPS)

A CPS is a service endpoint defined by {{RFC8816}} that stores PASSporTs for later retrieval when SIP Identity headers are removed by intermediate networks.  Originating networks publish signed PASSporTs to the destination’s CPS, and terminating networks retrieve them during call verification.

This specification does not modify the CPS publish/retrieve APIs, it only defines how to reliably find the correct CPS endpoint for any TN or SPC.

### Discovery Service

The Discovery Service provides REST API that allows quick, cryptographically verifiable lookups of CPS endpoints.  It supports two main functions:

- Registration of CPS endpoints – Authorized entities can post signed CPS advertisements that map TN ranges or SPCs to CPS URIs.
- Lookup of CPS endpoints – Any client (e.g., an Authentication Service or Verification Service) can query the Discovery Service to obtain the correct CPS URI for a given TN or SPC, along with supporting evidence such as certificate fingerprints and Signed Certificate Timestamps (SCTs).

### Discovery Service STI-CT Monitor Component

The Discovery Service includes an automated monitoring component that continuously watches STI‑CT logs for new or updated certificates.  When a certificate includes the CPS URI extension, the monitor:

- Verifies the certificate and its SCTs.
- Extracts the TNAuthList and CPS URI.
- Automatically registers or updates the mapping in the Discovery Service database.

This ensures that CPS endpoints referenced in Delegate Certificates become immediately discoverable, even if the operator has not explicitly posted an advertisement.

## CPS Discovery Mechanisms

The Discovery Service provides two complementary paths for finding the correct CPS for a TN or SPC.  Both paths are cryptographically verifiable and can be used independently or together.

### Certificate‑Based Discovery via STI‑CT

The first path leverages the CPS URI certificate extension defined in [I-D.draft-stir-cert-cps]](https://github.com/appliedbits/draft-stir-cert-cps/blob/main/draft-stir-cert-cps.md).  When an entity (for example, an enterprise or carrier) obtains a STIR/SHAKEN delegate certificate that covers one or more TNs, it can include an HTTPS CPS URI in that certificate.  The certificate is then logged in the STI Certificate Transparency (STI‑CT) ecosystem, producing a Signed Certificate Timestamp (SCT) that proves the certificate’s presence in the transparency log.

Any client (such as an Authentication Service, Verification Service, or directory operator) can query the STI‑CT log to:

- Find certificates whose TNAuthList matches the destination TN or SPC.
- Read the CPS URI extension to learn where Out‑of‑Band PASSporTs should be published or retrieved.
- Validate the certificate and its SCTs to ensure that the mapping is trusted and transparently auditable.

This approach requires no pre‑existing registry because the CPS location information travels with the same cryptographic credentials already used for STIR/SHAKEN call authentication.

### Discovery Service with STI‑CT Monitoring

The second path uses the Discovery Service API defined in this document. Instead of every client parsing Certificate Transparency logs directly, the Discovery Service operates as an always‑on STI‑CT monitor.  It continuously observes new or updated certificates and automatically extracts CPS URIs from any certificate containing the CPS URI extension.  For each certificate, it verifies SCTs, checks TNAuthList coverage, and records TN→CPS and SPC→CPS mappings in its internal database.

This means that as soon as a CPS URI is published in a certificate and logged to STI‑CT, it becomes discoverable through a simple REST query:

~~~~~~~~~~~~~
GET /.well-known/stir-cps-registry/discover?tn=+12025550123
~~~~~~~~~~~~~

A client receives a response with:

- The CPS URI (e.g., https://cps.example.net/oob/v1)
- Evidence such as the certificate fingerprint and SCTs

### End‑to‑End Flow

1. Certificate Issuance – An operator obtains a delegate certificate containing a CPS URI and submits it to STI‑CT.
2. Auto‑Registration – The Discovery Service detects the new certificate via its STI‑CT monitoring component and automatically creates the TN->CPS and SPC->CPS mappings.
3. Lookup – An originating Authentication Service placing a call queries the Discovery Service for the destination TN.
4. Publish – The Authentication Service publishes the PASSporT to the returned CPS URI.
5. Retrieve and Verify – The terminating Verification Service retrieves the PASSporT from that CPS and verifies it.

This flow provides zero‑touch provisioning for Out‑of‑Band PASSporT delivery: operators simply issue certificates containing their CPS URIs, and the Discovery Service handles everything else automatically, providing quick, verifiable lookups for any participant in the STIR/SHAKEN ecosystem.

# Discovery Service API

This section defines the HTTP interface for registering and discovering CPS endpoints. All endpoints MUST be served over HTTPS (TLS 1.2+). The API SHOULD be exposed at a stable base path; this document uses the following well-known locations:

~~~~~~~~~~~~~
POST /.well-known/stir-cps-registry/advertisements
GET  /.well-known/stir-cps-registry/discover?tn={+E164}
GET  /.well-known/stir-cps-registry/discover?spc={ID}
~~~~~~~~~~~~~

## Common Conventions

- Content Types
  - Registration requests use application/stir-cps-advertisement+json.
  - Discovery responses use application/stir-cps-discovery+json.
- Normalization
  - tn: parameters MUST be E.164 with leading “+” and no whitespace.
  - spc: values follow {{RFC8226}} conventions for SPC identifiers.
- Time: Timestamps are RFC 3339/ISO 8601 UTC.
- Caching: Discovery responses SHOULD include Cache-Control: max-age=NN and ETag. Clients MAY cache per policy but MUST honor certificate notAfter.
- Authn/Authz: The server MUST require HTTPS. It SHOULD support mTLS and/or OAuth 2.0 (mTLS or JWT-based client auth). However, acceptance of an Advertisement is ultimately gated by cryptographic proof of control, bearer auth alone is insufficient.  TODO: Provide more details for proof of control via RTU (TNAuthList, SCT)

## Registration API

The Registration API allows authorized entities to assert mappings from TN ranges or SPCs to CPS URIs. Each assertion is an Advertisement signed with the private key corresponding to a STIR certificate whose TNAuthList covers the asserted resources.

## Advertisement Object

Media type: application/stir-cps-advertisement+json

~~~~~~~~~~~~~
{
  "version": 1,
  "jti": "c0a8012e-9b38-4d6a-bfdf-2d3a1a2fe0b7",
  "resources": [
    { "tn": "+12025550100/20" },
    { "spc": "1234" }
  ],
  "cps": "https://cps.example.net/oob/v1",
  "validity": {
    "notBefore": "2025-08-06T00:00:00Z",
    "notAfter":  "2026-08-05T23:59:59Z"
  },
  "evidence": {
    "cert": "<base64-DER>"
  },
  "signature": "<Detached-JWS-compact>"
}
~~~~~~~~~~~~~

- version - Protocol version, MUST be 1.
- jti - Unique identifier for idempotency and replay protection.
- resources - Array of objects: either { "tn": "+E164[/prefix]" } or { "spc": "ID" }.
  - TN ranges **SHOULD** use prefix notation (`+1202555/24`); exact TN use `/15` equivalent (or omit suffix if you prefer exact-match semantics—server **MUST** treat omitted as exact).
- cps - HTTPS URI for the CPS root as defined in \[I-D.stir-cert-cps].
- validity - Optional server-enforced bounds. notAfter MUST NOT exceed certificate expiry.
- evidence.cert - Base64 DER of the certificate containing the TNAuthList. MAY be a delegate or provider certificate.
- signature - Detached JWS over the UTF-8 bytes of the Advertisement WITHOUT the signature field.
  - alg MUST correspond to evidence.cert public key.
  - The JWS header MUST include either x5c (preferred), or kid with a server-resolvable reference to the certificate.
  - Servers MUST verify the JWS and that the signing cert equals evidence.cert.

## POST /…/advertisements

Request body: Advertisement

Success:

~~~~~~~~~~~~~
201 Created
Location: /.well-known/stir-cps-registry/advertisements/{id}
Content-Type: application/json

{ "id": "{id}", "status": "accepted", "source": "registry" }
~~~~~~~~~~~~~

Errors:

- 400 Bad Request - Invalid JSON, unknown fields, malformed TN/SPC/CPS URI.
- 401/403 - Authentication/authorization failure (TLS/OAuth).
- 409 Conflict - Overlaps with an existing mapping having equal or higher specificity from the SAME certificate and a later notBefore.
- 422 Unprocessable Entity - JWS invalid, cert chain invalid, TNAuthList does not cover claimed resources, SCT missing/invalid, notAfter beyond cert expiry.

Idempotency: Servers MUST treat identical (jti, cert fingerprint) pairs as idempotent and SHOULD return the same {id}.

## Server Validation

Upon POST, the server MUST:

1. Validate TLS client auth policy (if configured).
2. Parse and validate the Advertisement schema.
3. Validate JWS signature, ensure the signing cert equals evidence.cert.
4. Validate the evidence.cert chain to an accepted STI trust root.
5. Validate SCTs (structure, signature, and consistency with the cert).
6. Validate TNAuthList coverage for all resources.
7. Validate cps is an HTTPS URI per [I-D.draft-stir-cert-cps]](https://github.com/appliedbits/draft-stir-cert-cps/blob/main/draft-stir-cert-cps.md).
8. Enforce validity window; clamp notAfter to certificate expiry.
9. Store mappings with appropriate indexes (TN prefix trie, SPC map).

# Discovery API

Lookup returns the CPS URI plus cryptographic evidence. Either tn or spc MUST be specified (not both).

Request

~~~~~~~~~~~~~
GET /.well-known/stir-cps-registry/discover?tn=+12025550123
GET /.well-known/stir-cps-registry/discover?spc=1234
~~~~~~~~~~~~~

Response (success)

~~~~~~~~~~~~~
{
  "cps": "https://cps.example.net/oob/v1",
  "source": "registry",              // or "ct"
  "matched": {
    "type": "tn",                    // "tn" or "spc"
    "value": "+12025550123",
    "scope": "+1202555/24"           // scope that matched
  },
  "evidence": {
    "cert_fingerprint": "SHA256:4F:..:A9",
    "notBefore": "2025-08-06T00:00:00Z",
    "notAfter":  "2026-08-05T23:59:59Z"
  }
}
~~~~~~~~~~~~~

Errors

- 404 Not Found - No valid mapping.
- 400 Bad Request - Invalid tn/spc.
- 406 Not Acceptable - accept filter excludes all candidates.
- 410 Gone - Mapping existed but is now revoked/expired (useful for caches).

Servers SHOULD include Cache-Control and ETag headers. Clients MAY use conditional requests.

# CT <-> Registration Synchronization

The Discovery Service contains a continuous STI-CT monitor that auto-registers CPS URIs from certificates containing the CPS-URI extension.

## Monitoring Behavior

- Inputs - One or more CT logs recognized by the STI ecosystem.
- Processing -  For each new/updated certificate:
  1. Verify chain to STI trust roots.
  2. Verify SCT(s) and inclusion (per CT client policy).
  3. If the certificate contains a CPS-URI extension, extract the URI.
  4. Extract TNAuthList entries (TNs, ranges, SPCs).
  5. Create/update mappings in the registry with source: "ct".
- Expiry - CT-derived mappings **MUST** be considered invalid after the certificate notAfter (plus an optional grace period ≤ 48h).
- Revocation - If OCSP/CRL indicates revocation (when available), the service SHOULD immediately mark mappings as inactive.

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
