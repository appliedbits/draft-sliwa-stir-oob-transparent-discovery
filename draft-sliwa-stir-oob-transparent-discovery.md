---
###
# Discovery of STIR Out‑of‑Band Call Placement Services
###
title: "Transparent Discovery of STIR Out‑of‑Band Call Placement Services"
abbrev: "Discovery of STIR OOB CPS"
category: std

docname: draft-sliwa-stir-oob-transparent-discovery-00
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
  github: "appliedbits/draft-sliwa-stir-oob-transparent-discovery"
  latest: "https://github.com/appliedbits/draft-sliwa-stir-oob-transparent-discovery"

author:
 -
    fullname: Rob Sliwa
    organization: Somos Inc.
    email: robjsliwa@gmail.com
 -
    fullname: Chris Wendt
    organization: Somos Inc.
    email: chris@appliedbits.com

normative:
  I-D.sliwa-stir-cert-cps-ext:
  I-D.wendt-stir-certificate-transparency:
  RFC8224:
  RFC8225:
  RFC8226:
  RFC8816:
  RFC9060:

informative:


--- abstract

This document defines a Discovery Service for STIR Out-of-Band (OOB) Call Placement Services (CPS). The Discovery Service enables Authentication Services (AS) and Verification Services (VS) to quickly determine which CPS is responsible for a given telephone number (TN) or Service Provider Code (SPC), allowing retrieval of PASSporTs even when SIP Identity headers are removed by non-IP or hybrid network segments. The Discovery Service leverages a CPS URI certificate extension, which allows STIR Certificates or Delegate Certificates to embed an HTTPS URI for the CPS serving the TNs or SPCs covered by the certificate. 

--- middle

# Introduction

In a STIR ecosystem, defined primarily by {{RFC8224}}, {{RFC8225}}, and {{RFC8226}}, and specifically when enabling Out-of-Band (OOB) delivery of PASSporTs defined in {{RFC8816}}, a Call Placement Service (CPS) plays a vital role, particularly when SIP Identity headers are lost or removed in non-IP or hybrid network environments. While the role of CPS was well established in {{RFC8816}}, the challenge remained for the definition of discovering which CPS is responsible for a specific telephone number (TN) or Service Provider Code (SPC) in a secure, scalable, and interoperable way.  

This document introduces a CPS Discovery Service designed to solve that challenge. The CPS Discovery Service provides a transparent and cryptographically verifiable method for identifying the correct CPS for any given TN or SPC identified in the TNAuthList of a STIR certificate defined in {{RFC8226}}, supporting OOB call authentication without requiring static configuration or bilateral agreements between service providers.

The CPS Discovery Service operates by leveraging the CPS URI certificate extension defined in {{I-D.sliwa-stir-cert-cps-ext}}. This extension allows STIR Certificates {{RFC8226}} or Delegate Certificates {{RFC9060}} to encode an HTTPS URI pointing to the CPS responsible for the TNs or SPCs listed in the certificate's TNAuthList. Once such certificates are published to STIR Certificate Transparency (STI-CT) logs defined in {{I-D.wendt-stir-certificate-transparency}}, the CPS URI becomes immediately visible, auditable, and publicly verifiable by relying parties.

To facilitate CPS discovery, the Discovery Service continuously monitors these CT logs, extracts CPS URIs from newly issued or updated certificates, and registers mappings from TNs or SPCs to CPS endpoints. These mappings are exposed via a simple REST API that supports fast, automated lookups by Authentication Services (AS) and Verification Services (VS). 

By combining CPS URI certificate extensions with CT monitoring and a queryable discovery interface, this approach enables automatic OOB CPS discovery. It can integrate with existing STIR framework components and requires only an additional lookup step to improve the robustness and scalability of Out-of-Band PASSporT delivery.

# Architectural Overview

This document defines a mechanism for discovering Call Placement Services (CPS) responsible for specific telephone numbers (TNs) or Service Provider Codes (SPCs) by monitoring STI Certificate Transparency (STI-CT) logs for certificates that include a CPS URI extension.

Entities that operate a CPS obtain delegate certificates with TNAuthList entries covering the TNs or SPCs they serve. These certificates include a CPS URI extension as defined in {{I-D.sliwa-stir-cert-cps-ext}}, indicating the HTTPS endpoint of the CPS. Once issued, these certificates are submitted to STI-CT logs.

Any relying party may monitor STI-CT logs for new or updated certificates containing a CPS URI extension. Upon detection, the CPS URI and its associated TNAuthList entries are extracted and recorded. This enables parties to associate TNs or SPCs with corresponding CPS URIs based on the content of logged certificates.

This approach provides the following properties:

- Transparency: CPS endpoint declarations are logged and auditable through STI-CT.
- Verifiability: Mappings are derived from signed certificates anchored in existing STIR trust infrastructure and supported by Signed Certificate Timestamps (SCTs).
- No bilateral provisioning: Originating and terminating parties do not require prior agreement or static configuration to determine the correct CPS.
- Compatibility: The mechanism is fully compatible with the STIR Out-of-Band (OOB) architecture as defined in {{RFC8816}} and requires no modification to existing CPS publish or retrieve interfaces.

This architecture enables a scalable and interoperable means of CPS discovery using existing STIR certificate issuance and transparency mechanisms.

# Components

The discovery mechanism relies on existing STIR framework components and Certificate Transparency (CT) logs. No new interfaces are required between entities. CPS operators signal the location of their service using a certificate extension, and relying parties extract and verify these declarations by monitoring CT logs.

## Call Placement Service (CPS)

A Call Placement Service (CPS), as defined in {{RFC8816}}, is a network-accessible endpoint that stores PASSporTs for later retrieval during call verification. CPSs are used when SIP Identity headers are removed or unavailable, such as in non-IP or hybrid telephony environments. Originating entities publish PASSporTs to the CPS associated with the called party's number, and terminating entities retrieve them during call processing.

This specification does not modify the behavior or interfaces of CPS endpoints. It only describes how CPS endpoint information is published and discovered using STIR certificates and delegate certificates and Certificate Transparency.

## Certificate Holders

Entities responsible for telephone numbers or SPCs, such as service providers or enterprises, obtain STIR certificates or delegate certificates with TNAuthList entries covering those resources. These certificates may optionally as defined in this document include a CPS URI extension indicating the HTTPS endpoint of the CPS that serves those numbers to facilitate discovery.

When these certificates are submitted to a recognized STI Certificate Transparency log, the CPS URI becomes visible to relying parties monitoring the log. This allows third parties to discover the CPS associated with a number without requiring pre-provisioning or bilateral configuration.

## Transparency Log Monitors

Any party may monitor STI Certificate Transparency logs for certificates containing a CPS URI extension. Upon detecting a new or updated certificate, the monitor performs the following:

- Verifies the certificate chain to a trusted STI root
- Validates the certificate's Signed Certificate Timestamps (SCTs)
- Extracts the TNAuthList and CPS URI from the certificate
- Associates the covered TNs or SPCs with the indicated CPS URI

Monitors may use this data to maintain a local registry or cache of TN→CPS and SPC→CPS mappings. These mappings can be used by Authentication Services or Verification Services during call processing.

# CPS Discovery Mechanisms

This section describes the mechanism by which Call Placement Service (CPS) information is discovered through monitoring of STI Certificate Transparency (STI-CT) logs. This method provides a decentralized, cryptographically verifiable, and interoperable means of associating telephone numbers (TNs) or Service Provider Codes (SPCs) with CPS endpoints.

## Certificate-Based CPS Publication

CPS operators often associated with entities authorized for telephone numbers or SPCs obtain STIR certificates or delegate certificates containing a TNAuthList and the CPS URI certificate extension as defined in [I-D.sliwa-stir-cert-cps-ext]. The CPS URI identifies the CPS HTTPS endpoint where Out-of-Band (OOB) PASSporTs can be published or retrieved for the associated identifiers.

These certificates are submitted to one or more recognized STI-CT logs. Each submission yields a Signed Certificate Timestamp (SCT), which proves inclusion in the log and enables public verification.

## Discovery via Log Monitoring

Relying parties (e.g., Authentication Services, Verification Services, or intermediary discovery components) monitor STI-CT logs for new or updated certificates that include a CPS URI extension. Upon detection, the monitor performs the following steps:
	
1.	Verifies the certificate chain to a trusted STI root.
2.	Validates the SCT(s) associated with the certificate.
3.	Extracts the TNAuthList and the CPS URI extension.
4.	Associates the covered TNs or SPCs with the CPS URI.

The resulting mappings are used to determine the appropriate CPS endpoint during call placement and verification. This process allows discovery to be performed in real-time or asynchronously using CT log clients, without requiring direct interaction with the certificate holder or CPS operator.

# End-to-End Process Summary

1.	A STIR certificate or delegate certificate is issued containing TNAuthList and CPS URI.
2.	The certificate is logged in an STI-CT log, generating SCTs.
3.	A monitoring system observes the log and extracts TN→CPS and SPC→CPS mappings.
4.	Authentication or Verification Services consult the mappings to identify the CPS endpoint corresponding to a given TN or SPC.
5.	PASSporTs are published or retrieved using the discovered CPS URI as part of the OOB authentication process.

This approach removes the need for centralized registries, REST-based discovery interfaces, or bilateral provisioning agreements. All discovery is based on cryptographic credentials and public transparency logs, ensuring an auditable and interoperable discovery process.

# Expected Monitor Behavior

Parties that wish to perform CPS discovery may monitor one or more STI Certificate Transparency (STI-CT) logs for delegate certificates containing the CPS URI extension. Monitors are expected to:

- Validate certificate chains to STI trust anchors.
- Verify inclusion of certificates via Signed Certificate Timestamps (SCTs).
- Extract the CPS URI and TNAuthList from certificates.
- Associate telephone numbers or SPCs with the declared CPS URI.

Monitors may use this information to enable Authentication Services (AS) or Verification Services (VS) to locate the appropriate CPS endpoint. Implementations may maintain local caches or registries based on the extracted data, respecting certificate validity periods and revocation status when available.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

# Security Considerations

The CPS discovery mechanism relies on the integrity of STIR delegate certificates and STI Certificate Transparency (STI-CT) logs. Several security considerations apply:

- Misissued Certificates: A CPS URI could be falsely claimed in a certificate. Relying parties must validate that certificates are issued by trusted STIR Certification Authorities and verify TNAuthList accuracy during issuance.
- Log Misbehavior: CT logs may omit or backdate entries. Implementations should monitor multiple CT logs and validate Signed Certificate Timestamps (SCTs) to ensure visibility and integrity.
- Stale or Revoked Certificates: CPS mappings derived from expired or revoked certificates may lead to incorrect routing. Implementations should check certificate validity and revocation status before using extracted mappings.
- Information Exposure: CPS URIs and TNAuthList entries are publicly logged. This may reveal operational details but does not introduce new exposures beyond existing STIR practices.

This mechanism inherits the security properties of the STIR certificate infrastructure and CT logging, offering verifiable, auditable discovery without requiring bilateral trust.


# IANA Considerations

This document has no IANA actions.


--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
