---
title: "OAuth 2.0 Software Statement Issuance"
abbrev: oauth-software-statement-issuance
docname: draft-mcguinness-oauth-software-statement-issuance-latest
category: std

ipr: trust200902
area: "Security"
workgroup: "Web Authorization Protocol"

keyword:
 - OAuth
 - Software Statement
 - Dynamic Client Registration
 - Deferred Processing
 - Client ID Metadata Document
 - Approval

stand_alone: yes
pi: [toc, sortrefs, symrefs]

author:
 -
    ins: K. McGuinness
    name: Karl McGuinness
    email: public@karlmcguinness.com

normative:
  RFC6234:
  RFC6749:
  RFC6838:
  RFC7009:
  RFC7515:
  RFC7519:
  RFC7591:
  RFC7636:
  RFC8414:
  RFC8693:
  RFC8707:
  RFC8725:
  RFC9126:
  RFC9207:
  RFC9396:
  RFC9449:
  RFC9700:
  DTR:
    target: https://datatracker.ietf.org/doc/draft-gerber-oauth-deferred-token-response
    title: "Deferred Token Response"
  CIMD:
    target: https://datatracker.ietf.org/doc/draft-ietf-oauth-client-id-metadata-document
    title: "OAuth Client ID Metadata Document"
  CLIENT-INSTANCE:
    target: https://datatracker.ietf.org/doc/draft-mcguinness-oauth-client-instance-assertion
    title: "OAuth 2.0 Client Instance Assertion"
  OAUTH-MRT:
    target: https://openid.net/specs/oauth-v2-multiple-response-types-1_0.html
    title: "OAuth 2.0 Multiple Response Type Encoding Practices"
  FORM-POST:
    target: https://openid.net/specs/oauth-v2-form-post-response-mode-1_0.html
    title: "OAuth 2.0 Form Post Response Mode"

informative:
  RFC8252:
  RFC8628:
  ABCA:
    target: https://datatracker.ietf.org/doc/draft-ietf-oauth-attestation-based-client-auth
    title: "OAuth 2.0 Attestation-Based Client Authentication"
  APPROVAL-DCR:
    target: https://datatracker.ietf.org/doc/draft-dellaert-oauth-approval-based-dcr
    title: "OAuth 2.0 Approval-Based Dynamic Client Registration"
  AU-CDR:
    target: https://consumerdatastandardsaustralia.github.io/standards/
    title: "Consumer Data Standards (Australia)"
  OID4VCI:
    target: https://openid.net/specs/openid-4-verifiable-credential-issuance-1_0.html
    title: "OpenID for Verifiable Credential Issuance 1.0"
  OPENID-FED:
    target: https://openid.net/specs/openid-federation-1_0.html
    title: "OpenID Federation 1.0"
  PRESENTATION:
    target: https://datatracker.ietf.org/doc/draft-mcguinness-oauth-sw-stmt-presentation
    title: "OAuth 2.0 Software Statement Consumption and Runtime Presentation"
  SIGNALS:
    target: https://datatracker.ietf.org/doc/draft-mcguinness-oauth-sw-stmt-signals
    title: "Shared Signals Events for OAuth Software Statements"
  PUSHED-DCR:
    target: https://datatracker.ietf.org/doc/draft-richer-oauth-pushed-client-registration
    title: "OAuth 2.0 Pushed Client Registration"
  STATUS-LIST:
    target: https://datatracker.ietf.org/doc/draft-ietf-oauth-status-list
    title: "Token Status List"
  TRUST-FRAMEWORK:
    target: https://datatracker.ietf.org/doc/draft-mcguinness-oauth-id-assertion-framework
    title: "OAuth Identity Assertion Trust Framework"
  UK-OPEN-BANKING:
    target: https://openbankinguk.github.io/dcr-docs-pub/v3.3/dynamic-client-registration.html
    title: "Open Banking UK Dynamic Client Registration Specification v3.3"

--- abstract

RFC 7591 standardizes how a client presents a software statement and how a registration endpoint consumes it, but not how the client obtains one. This specification defines OAuth 2.0 issuance flows for that artifact, in which a Client ID Metadata Document identifies a client that has not been registered with the authorization server.

In the redirect flow, the authorization endpoint returns a short-lived `software_statement_code`, which the client redeems using a new token endpoint grant. A completed decision returns a statement; a pending decision uses Deferred Token Response and polling. The statement never appears in an authorization response URL.

A client that holds an initial access token authorizing issuance instead uses OAuth 2.0 Token Exchange (RFC 8693), without a redirect.

The issued statement is consumed through RFC 7591 dynamic client registration; a companion specification defines sender-constrained runtime presentation in authorization and token requests.

--- middle

# Introduction

Section 2.3 of {{RFC7591}} defines a software statement as a JWT that asserts client metadata. A client presents it to a dynamic client registration endpoint, where the signature identifies who attested to the metadata. {{RFC7591}} standardizes consumption, but not issuance.

Today, issuance relies on manual provisioning, deployment-specific portals, or proprietary federation processes. The UK Open Banking Directory and Australian Consumer Data Right Register each built a central issuer for statements consumed through the same {{RFC7591}} `software_statement` member ({{UK-OPEN-BANKING}}, {{AU-CDR}}). Clients nevertheless need an ecosystem-specific issuance integration for each.

A portal does not provide interoperable submission, deferral, delivery, metadata binding, renewal, or errors. This specification standardizes that protocol while leaving approval workflow, acceptance policy, and issuer trust to deployments.

Pre-registration {{CIMD}}, pushed registration {{PUSHED-DCR}}, and approval-based registration {{APPROVAL-DCR}} each establish trust bilaterally at one authorization server from client-supplied metadata.

A software statement makes an issuer's decision portable. A publisher program, enterprise security function, or ecosystem operator reviews the software once; each authorization server in the audience can rely on the signed decision under its own policy ({{what-issuance-attests}}, {{comparison}}, {{beyond-pre-registration}}).

This specification supplies the missing issuance protocol and hardens the existing artifact for interoperability ({{software-statement-format}}). This document owns the artifact and how a client obtains one. Everything a consuming authorization server does with a statement, configuring which issuers it trusts and in which role, validating and applying one, bounding a registration by its expiry, presenting one at runtime, and serving a tenant's current approvals, is defined by the companion {{PRESENTATION}}. It introduces no new client credential or federation architecture. Portability remains bounded by configured issuer trust, typically within an ecosystem or administrative domain rather than the open web.

A client identified by its {{CIMD}} URL obtains a statement through either:

* a redirect flow, using `response_type=software_statement_code` and the `urn:ietf:params:oauth:grant-type:software-statement` redemption grant; or
* OAuth token exchange, when it already holds an initial access token authorizing issuance ({{token-exchange-profile}}).

The flow concerns client establishment, not authorization to access a protected resource. Consequently, a software statement request cannot be combined with `scope`, `resource`, `authorization_details`, or an access-token-producing response type.

Other mechanisms are preferable when:

* the resource owner is the approver, because the ordinary OAuth grant is the approval;
* the trust decision is local to one authorization server, where an initial access token {{RFC7591}}, pre-registration {{CIMD}}, or approval-based registration {{APPROVAL-DCR}} can establish the client directly; or
* the client cannot host a metadata document at an HTTPS URL and therefore does not fit this specification's identity model ({{client-identity}}).

A software statement earns its cost beyond those boundaries when one or more of the following holds:

1. One approval must be honored at many authorization servers.
2. The party approving the software is not the resource owner in a transaction.
3. Approval must exist before any authorization transaction does.
4. Approval requires asynchronous review that outlives a single protocol exchange.
5. Issuance and renewal must be automated rather than performed through a portal.

Where none applies, this issuance protocol is unnecessary. Issuance is decoupled from access grants and can complete out of band over hours or days. {{deployment-examples}} illustrates the boundaries.

## What Pre-Registration Does Not Solve {#beyond-pre-registration}

Pre-registration {{CIMD}} lets a client enroll its identifier URL before its first authorization request. The authorization server fetches the metadata, reviews it, and records a local decision. This specification does not replace that model.

Pre-registration changes when the trust decision is made, but not:

* **Who decides:** each consuming authorization server still evaluates the client's self-asserted document.
* **What transfers:** the decision remains local state and cannot be presented elsewhere.
* **What was approved:** the subject is a URL whose content can change, with no interoperable binding to the reviewed version.
* **What acceptance depends on:** later evaluation still requires dereferencing the document.

A software statement makes one decision portable, binds it to the exact content evaluated ({{metadata-snapshot}}), and permits offline verification. This is useful where many authorization servers are not staffed to review software: each verifies a trusted issuer's signature instead of running an approval process. The mechanisms compose ({{comparison}}).

## Relationship to Other Specifications

This specification uses the following building blocks:

* {{RFC7591}} defines the software statement and the client metadata carried in it.
* {{CIMD}} defines the client identifier, canonical metadata source, pre-registration, and metadata-change handling. Those mechanisms can serve as approval-carrying enrollment, while the metadata digest ({{metadata-snapshot}}) makes changes precisely detectable.
* {{DTR}} defines client opt-in, token endpoint deferral, polling, cancellation, and sender constraint. This specification profiles it for both issuance flows.
* {{RFC8693}} defines both the response convention for non-access security tokens and the exchange profiled in {{token-exchange-profile}}.

{{APPROVAL-DCR}} creates an authorization-server-specific `client_id` and, when applicable, client credentials after approval. This specification issues a portable statement for later {{RFC7591}} registration. The two compose ({{comparison}}).

{{PUSHED-DCR}} pushes registration alongside an authorization flow at the same authorization server. Such a registration can carry a statement issued under this specification. That composition is out of scope; combining issuance itself with an access-granting response type remains prohibited ({{prohibited-parameters}}).

{{CLIENT-INSTANCE}} carries per-instance identity into tokens for runtime instances behind one `client_id`. This specification carries the trust decision about the client software to the trusting authorization server. {{PRESENTATION}} describes their composition.

OpenID Federation {{OPENID-FED}} conveys attested metadata through trust chains, suiting ecosystems prepared to operate federation infrastructure. This specification instead issues the existing {{RFC7591}} artifact under explicit issuer trust. Federation standardizes trust resolution, not enrollment, approval, or deferred completion; it relocates rather than replaces the issuance ceremony defined here.

The OAuth Identity Assertion Trust Framework {{TRUST-FRAMEWORK}} could generalize pairwise configuration by evaluating issuers against published conditions, including authority over the client's identifier namespace. Such a profile is out of scope.

This specification does not define approval workflow, approver identity, external approval integration, acceptance of any particular issuer, or issuer discovery. Acceptance policy and trust establishment are deployment-specific. A future metadata-document member could name issuers authorized for the publisher's namespace and serve as {{TRUST-FRAMEWORK}} evidence.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

OAuth terminology is defined by {{RFC6749}}. Client metadata and software statement terminology is defined by {{RFC7591}}. Client ID Metadata Document terminology is defined by {{CIMD}}. Deferred response terminology is defined by {{DTR}}.

This specification additionally defines the following terms:

Software Statement Request:
: A request in which a client asks an authorization server to issue a software statement, sent as an authorization request with `response_type=software_statement_code`.

Issuing Authorization Server:
: The authorization server that makes the issuance decision and signs the software statement.

Trusting Authorization Server:
: An authorization server that consumes the issued software statement, in a dynamic client registration request ({{PRESENTATION}}), a runtime presentation, or a replacement delivery ({{PRESENTATION}}).

Software Statement Code:
: The short-lived, single-use artifact returned by the software statement code response ({{software-statement-code-response}}) and redeemed at the token endpoint ({{software-statement-code-redemption}}). It is not an authorization code: redeeming it yields the software statement token response or a deferred token response, never an access token.

Originating Request:
: A request that can produce a software statement: a software statement code redemption ({{software-statement-code-redemption}}) or a token exchange ({{token-exchange-profile}}). Each yields a statement, a terminal denial, or, at a deferred issuer, a deferral.

Metadata Snapshot:
: The validated canonical metadata bound to a request, as defined in {{metadata-snapshot}}.

Metadata Digest:
: The unpadded base64url encoding of the SHA-256 hash of the retrieved octets of a metadata document, as defined in {{metadata-snapshot}}.

# Protocol Overview

A client initiates the redirect flow at the authorization endpoint and redeems the resulting software statement code at the token endpoint. The issuer either completes the decision synchronously or defers it under {{DTR}}. A client that already holds an initial access token authorizing issuance instead uses token exchange, without a user agent ({{token-exchange-profile}}).

The flow has four elements:

1. An HTTPS Client ID Metadata Document URL identifies the client, and its content supplies the canonical metadata {{CIMD}}.
2. The issuing authorization server fetches and snapshots that document, decides whether to issue, and signs the statement ({{metadata-snapshot}}).
3. The client presents the statement to consuming servers: in an {{RFC7591}} registration request ({{PRESENTATION}}), a sender-constrained runtime presentation, or a delivery that renews what a server already holds ({{PRESENTATION}}).
4. Trusting authorization servers in its audience apply their local acceptance policies.

The URL remains the client identity when its content changes. One issuance decision can serve many registrations and runtime presentations ({{comparison}}).

~~~
                +--------------------------+
                | Client ID Metadata       |
                | Document (client_id URL) |
                +--------------------------+
                  ^                      ^
            hosts |                      | fetches and
                  |                      |   snapshots
+----------+      |                      |      +---------------+
|          +------+                      +------+    Issuing    |
|  Client  |                                    | Authorization |
|          |--(1) software statement request -->|    Server     |
|          |<-(2) software statement -----------|   (approval)  |
+----------+      (signed JWT)                  +---------------+
     |
     | (3) RFC 7591 registration request
     |     carrying the software statement
     v
+---------------+  +---------------+       +---------------+
|   Trusting    |  |   Trusting    |  ...  |   Trusting    |
| Authorization |  | Authorization |       | Authorization |
|   Server 1    |  |   Server 2    |       |   Server M    |
+---------------+  +---------------+       +---------------+
~~~

The figure shows the registration path. Under runtime presentation ({{PRESENTATION}}), step 3 is instead a sender-constrained authorization or token request carrying the statement, and no registration is created.

## Redirect Flow Overview

~~~
+--------+                                  +----------------------+
| Client |                                  | Authorization Server |
+--------+                                  +----------------------+
    |                                                 |
    | (A) Authorization request                       |
    |     response_type=software_statement_code       |
    |     client_id=<metadata document URL>           |
    |     code_challenge, [audience], [dpop_jkt]      |
    |------------------------------------------------>|
    |                                                 |
    | (B) Software statement code response            |
    |     (software_statement_code, state, iss)       |
    |<------------------------------------------------|
    |                                                 |
    | (C) Redemption                                  |
    |     (software_statement_code, code_verifier,    |
    |      completion_mode=deferred)                  |
    |------------------------------------------------>|
    |                                                 |
    | (D) Software statement response or              |
    |     deferred response (deferral_code)           |
    |<------------------------------------------------|
    |                                                 |
    | (E) Poll(s) (deferral_code), if deferred        |
    |------------------------------------------------>|
    |     authorization_pending or final response     |
    |<------------------------------------------------|
~~~

At (A), the client requests a statement. The authorization server validates the request, performs any user-agent interaction or initiates an approval workflow, and returns a software statement code at (B).

At (C), the client redeems the code with its PKCE verifier. Redemption returns the statement at (D) or a {{DTR}} `deferral_code`; in the latter case, the client polls at (E). Errors follow {{RFC6749}} and {{DTR}}.

# Client Identification and Authentication {#client-identity}

The `client_id` in every request defined by this specification MUST be a client identifier URL conforming to {{CIMD}}. The authorization server MUST obtain and validate the corresponding Client ID Metadata Document according to {{CIMD}}. A non-URL identifier issued by the authorization server is not supported by this specification.

In the redirect flow, the authorization server MUST compare the `redirect_uri` in the request with the `redirect_uris` in the Client ID Metadata Document according to {{CIMD}} and {{RFC9700}}. The authorization server MUST NOT redirect the user agent when the client identifier or redirect URI is missing or invalid. A client identifier whose metadata document cannot be retrieved or validated, including a document rejected for duplicate member names ({{metadata-snapshot}}), is invalid for this purpose; the authorization server reports the failure directly to the user agent rather than to the redirection endpoint.

The client authenticates to the token endpoint using the `token_endpoint_auth_method` and related key metadata in its Client ID Metadata Document. The authorization server classifies the client from that member:

* `none` (explicit) establishes a public client.
* Any other value establishes a confidential client, and the authorization server MUST require exactly that method, as required by {{CIMD}}; a declared method the authorization server does not support MUST cause rejection rather than treatment as public.
* An omitted value establishes neither: the {{RFC7591}} default is a symmetric-secret method that {{CIMD}} prohibits and an unregistered client cannot satisfy, so any request identifying such a client MUST be rejected with `invalid_request`, returned to the redirection URI validated against the document.

Classification uses the singular `token_endpoint_auth_method`; this specification does not interpret a list of methods a client merely declares it supports, and a client that wants a deterministic classification for issuance declares the singular member.

Sender constraint is established per flow:

* A public client using the redirect flow MUST include `dpop_jkt` in the authorization request and MUST present a DPoP proof signed with the corresponding key at redemption and on every polling request.
* A public client using the token exchange profile ({{token-exchange-profile}}) MUST instead include a DPoP proof on the exchange request. The authorization server MUST bind any resulting deferral state to that proof's key, and the client MUST use the same key on every polling request.
* A confidential client MAY use DPoP in addition to its client authentication method.

All uses of DPoP MUST follow {{RFC9449}} and {{DTR}}.

## Metadata Snapshot {#metadata-snapshot}

Client metadata documents can change while a request is pending. Before returning a software statement code or a deferral code, the authorization server MUST bind it to the validated canonical metadata. The bound values constitute the metadata snapshot for the request.

The authorization server MAY retrieve the document again before issuing the software statement. If it does so and detects a security-relevant change (for example, a change to `jwks`, `jwks_uri`, `redirect_uris`, or `token_endpoint_auth_method`), it MUST either re-evaluate the request under the new metadata or reject the request. It MUST NOT silently combine values from different document versions.

Re-evaluation replaces the bound snapshot in full. A statement attests the snapshot in effect when it is signed, and `cimd_digest` is that snapshot's digest, so a client that observes a digest other than the one it expected knows which document content was attested. An approval recorded against a superseded snapshot does not carry forward to its replacement without a fresh issuance-policy decision. Replacing the snapshot does not alter the sender-constraint context recorded for a deferral, which remains as it was fixed at origination ({{deferred-processing}}). If a replacement snapshot no longer authorizes the key material behind that context, for example because the client-authentication key is absent from the new document, the authorization server MUST invalidate the deferral; the client makes a new request under its current keys.

The metadata digest is the unpadded base64url-encoded SHA-256 hash {{RFC6234}} of the retrieved representation body after removal of content coding. No transcoding, normalization, or re-serialization occurs; a byte order mark and trailing newline are included. Retrieval for snapshot purposes SHOULD NOT use content negotiation, and an issuance source SHOULD serve the document without negotiated variants, so that independent fetchers obtain identical bytes.

Equal digests identify the same document for this specification. A changed digest marks a new trust state for the same client identifier and is the signal used by the re-evaluation rule above. The digest also supplies the `cimd_digest` claim ({{software-statement-format}}) and audit guidance ({{security-considerations}}).

Byte identity deliberately detects serialization-only changes. A digest mismatch is an input to policy, not a validation failure. A publisher SHOULD serve a stable byte artifact whose octets change only with its metadata; a document rendered dynamically or served through content negotiation produces digest changes unrelated to its metadata. The authorization server MUST reject duplicate object member names, because parsers can interpret them differently despite an identical digest.

An issuance source SHOULD publish keys by reference through `jwks_uri` rather than inline through `jwks`. Rotation behind a stable URI leaves the document and digest unchanged; inline rotation changes both, so the attested keys no longer match the current document and a new statement is needed. The statement attests either the key location or the inline keys. The convenience cuts both ways: rotation invisible to the digest means key-host compromise is also invisible to it, and where the attested key is the runtime proof under {{PRESENTATION}} the compromise substitutes the presenter as well; {{PRESENTATION}} weighs the trade, and an issuer serving theft-sensitive deployments attests `jwks` inline instead.

A metadata document used as an issuance source MUST NOT contain a `software_statement` member, although {{CIMD}} otherwise permits one. Embedding a prior statement would change the digest on every issuance; an issuing authorization server treats a document containing one as failing validation for issuance. A statement is presented in a registration request ({{PRESENTATION}}) or at runtime ({{PRESENTATION}}), never embedded in the source document.

# Software Statement Authorization Request {#authorization-request}

The client sends an authorization request as described in Section 4.1.1 of {{RFC6749}}, with the following parameters:

`response_type`:
: REQUIRED. The value MUST be `software_statement_code`.

`client_id`:
: REQUIRED. The client identifier URL described in {{client-identity}}.

`redirect_uri`:
: REQUIRED. The value MUST exactly match one of the redirection URIs in the Client ID Metadata Document, subject to the redirect URI rules of {{CIMD}}. A public client MUST use an HTTPS redirection URI and MUST NOT use a private-use URI scheme or a loopback interface redirection URI in a software statement request. A confidential client MAY use a loopback or private-use redirection URI, because redemption is bound to its client authentication ({{authorization-response-security}}); such a request MUST also include `dpop_jkt`, since a declared confidential method proves key possession, not that the key is absent from distributed software.

`state`:
: REQUIRED. An opaque value used by the client to bind the authorization response to its request. The authorization server MUST return the exact value in the authorization response.

`code_challenge`:
: REQUIRED. A PKCE challenge as defined by {{RFC7636}}.

`code_challenge_method`:
: REQUIRED. The value MUST be `S256`.

`response_mode`:
: OPTIONAL. The mechanism for returning authorization response parameters, as defined by {{OAUTH-MRT}}. The default response mode for `response_type=software_statement_code` is `query`. Response mode considerations are given in {{software-statement-code-response}}.

`audience`:
: OPTIONAL. A target service at which the client intends to use the statement, with the semantics defined in Section 2.1 of {{RFC8693}}. The parameter MAY be repeated to request multiple audiences. Each value MUST be an authorization server issuer identifier as defined by {{RFC8414}}; values MUST NOT be repeated, and order is insignificant. The authorization server selects the final audience according to policy. When this parameter is present, every value in the statement's `aud` claim MUST have appeared in the request. If no requested audience is acceptable, the authorization server MUST reject the request with `invalid_target` {{RFC8693}}, following the authorization-request precedent of {{RFC8707}}. These semantics apply only to software statement requests and do not affect proprietary uses of `audience` for access-token targeting.

`completion_mode`:
: OPTIONAL. A value that includes `deferred`, sent as the advance hint {{DTR}} defines for an endpoint preceding a token request. It lets the authorization server choose a review path suited to out-of-band completion before it begins work, and does not replace the opt-in required at redemption ({{deferred-processing}}).

`dpop_jkt`:
: REQUIRED for a public client and for any request whose `redirect_uri` is a loopback or private-use URI; OPTIONAL otherwise. The parameter has the semantics defined in Section 10 of {{RFC9449}}. When present, its value MUST be associated with the resulting software statement code and with any deferral state derived from its redemption.

The authorization server MUST reject with `invalid_request` a request that omits a required PKCE parameter or a required `dpop_jkt`.

The following is a non-normative example of an authorization request from a confidential client; a public client would additionally include `dpop_jkt` (line breaks are for display purposes only):

~~~ http
GET /authorize?response_type=software_statement_code
  &client_id=https%3A%2F%2Fclient.example.org%2Fmetadata.json
  &redirect_uri=https%3A%2F%2Fclient.example.org%2Fcb
  &state=4L7xQ2mN9pR6sT1vW8yZ3aB5cD0fG2hJ
  &code_challenge=E9Melhoa2OwvFrEMTJguCHaoeK1t8URWbuGJSstw-cM
  &code_challenge_method=S256 HTTP/1.1
Host: server.example.com
~~~

## Prohibited Parameters {#prohibited-parameters}

A software statement request does not grant access to a protected resource. The following authorization request parameters MUST NOT be present:

* `scope`;
* `resource`, as defined by {{RFC8707}}; and
* `authorization_details`, as defined by {{RFC9396}}.

An authorization server MUST reject a request containing any of these parameters with `invalid_request`. The same prohibition and error apply to a software statement code redemption ({{software-statement-code-redemption}}). The `audience` parameter ({{authorization-request}}) names the authorization servers at which the issued statement will be presented, a property of the requested artifact rather than a request for access, and is not prohibited by this section.

Hybrid response types that combine `software_statement_code` with `code`, `token`, `id_token`, or any other response type are not defined. An authorization server MUST reject such a request with `unsupported_response_type`.

# Authorization Response {#authorization-response}

After validating the request and performing any immediate interaction, the authorization server returns the software statement code response or an error. The authorization server MUST NOT place the software statement or approval-sensitive information in any authorization response.

A denial is never signaled in the authorization response. The authorization server returns a software statement code whether the issuance decision is complete, pending, or already a denial, and a decision to deny is delivered at redemption ({{terminal-denial}}). A client therefore does not implement a redirect-side `access_denied` handler for issuance outcomes.

## Software Statement Code Response {#software-statement-code-response}

The authorization server returns the following parameters to the client's redirection endpoint through the user agent, using the selected response mode, whether or not the issuance decision has already completed:

`software_statement_code`:
: REQUIRED. A short-lived, single-use artifact redeemed at the token endpoint ({{software-statement-code-redemption}}). It MUST be bound to the client identifier, redirect URI, PKCE challenge, metadata snapshot, requested audience, and, when `dpop_jkt` was present, that JWK thumbprint. The value MUST contain at least 128 bits of entropy from a cryptographically secure random source, MUST be opaque to the client, MUST expire shortly after issuance, and MUST NOT be accepted more than once.

`state`:
: REQUIRED. The exact value received in the authorization request.

`iss`:
: REQUIRED. The authorization server issuer identification parameter defined by {{RFC9207}}.

A software statement code is not an authorization code and MUST NOT be redeemable as one. Redemption requires the PKCE verifier and, for a public client, a DPoP proof with the `dpop_jkt` key.

The fragment response mode SHOULD NOT be used, because scripts at the redirection endpoint can access it. A client MAY request `form_post` {{FORM-POST}} to keep the code out of URLs, browser history, and Referer headers.

An OAuth library that recognizes only `code` needs a callback adapter for `software_statement_code`. The adapter applies the same `state`, `iss`, and PKCE handling.

For example, using the default query response mode (line breaks are for display purposes only):

~~~ http
HTTP/1.1 302 Found
Location: https://client.example.org/cb?
  software_statement_code=V7e1gP8zT2mN4qR6sW9xY3aB5cD7fH0jK2pL4uQ6vX8&
  state=4L7xQ2mN9pR6sT1vW8yZ3aB5cD0fG2hJ&
  iss=https%3A%2F%2Fserver.example.com
~~~

Request validation errors, such as a prohibited parameter or an unsupported response type, follow Section 4.1.2.1 of {{RFC6749}}. An issuance denial is not a request validation error.

# Software Statement Code Redemption {#software-statement-code-redemption}

The client redeems a software statement code by sending an HTTP `POST` request to the token endpoint using the `application/x-www-form-urlencoded` format with:

`grant_type`:
: REQUIRED. The value MUST be `urn:ietf:params:oauth:grant-type:software-statement`.

`software_statement_code`:
: REQUIRED. The software statement code returned by the authorization endpoint.

`redirect_uri`:
: REQUIRED. The same redirection URI used in the authorization request.

`client_id`:
: REQUIRED. The same client identifier URL used in the authorization request.

`code_verifier`:
: REQUIRED. The PKCE verifier corresponding to the `code_challenge` in the authorization request.

`completion_mode`:
: REQUIRED when the authorization server advertises `deferred_token_response_supported` ({{authorization-server-metadata}}); otherwise not used, and a synchronous issuer ignores it ({{deferred-processing}}). When present, the value MUST include `deferred`, per the deferral opt-in rule of {{deferred-processing}}.

The request MUST NOT contain `audience`, which was bound at the authorization endpoint; a request containing it is rejected with `invalid_request`. The client authenticates according to {{client-identity}}, and when `dpop_jkt` was included in the authorization request, the client MUST send a DPoP proof for the token endpoint using the same key.

The authorization server MUST validate the software statement code and all of its bindings before processing the request. An invalid, expired, previously used, or incorrectly bound code MUST result in an `invalid_grant` error; a PKCE or DPoP binding failure is handled according to {{RFC7636}} or {{RFC9449}}, respectively.

A redemption attempt consumes the software statement code whenever the presented code value is valid, including when its PKCE, DPoP, or client-authentication bindings fail, so a live code cannot be ground down by repeated guessing; the trade, that an observer of the code value can burn it and force a restart, is accepted ({{authorization-response-security}}). A previously consumed code presented again MUST be rejected, and the authorization server SHOULD revoke any deferral derived from it ({{RFC9700}}). For a valid, unconsumed code, the result depends on the issuance decision:

* **Approved:** the authorization server returns the software statement token response ({{software-statement-response}}).
* **Denied:** it returns the terminal denial of {{terminal-denial}}.
* **Pending:** a deferred issuer returns the deferred token response of {{DTR}}. It binds the deferral to the code's client identifier, metadata snapshot, audience, and client authentication or DPoP key. The client then polls according to {{deferred-processing}}.

A synchronous issuer never reaches the pending branch, having decided before it returned the code ({{deferred-processing}}).

The following is a non-normative example of a redemption request from a confidential client at a deferred issuer (line breaks are for display purposes only):

~~~ http
POST /token HTTP/1.1
Host: server.example.com
Content-Type: application/x-www-form-urlencoded

grant_type=urn%3Aietf%3Aparams%3Aoauth%3Agrant-type%3A
  software-statement
  &software_statement_code=V7e1gP8zT2mN4qR6sW9xY3aB5cD7fH0jK2pL4uQ6vX8
  &redirect_uri=https%3A%2F%2Fclient.example.org%2Fcb
  &client_id=https%3A%2F%2Fclient.example.org%2Fmetadata.json
  &code_verifier=dBjftJeZ4CVP-mB92K27uhbUJU1p1r_wW1gFWFOEjXk
  &completion_mode=deferred
  &client_assertion_type=urn%3Aietf%3Aparams%3Aoauth%3A
  client-assertion-type%3Ajwt-bearer
  &client_assertion=eyJhbGciOiJFUzI1NiIsImtpZCI6ImNsaWVudC0xIn0...
~~~

# Token Exchange Profile {#token-exchange-profile}

A software statement request asks the authorization server to make a new issuance decision. A client that already holds a token carrying issuance authority MAY instead exchange that token for a statement using OAuth 2.0 Token Exchange {{RFC8693}}, a pattern used by existing ecosystems ({{UK-OPEN-BANKING}}, {{AU-CDR}}). Support is advertised through `software_statement_subject_token_types_supported` ({{authorization-server-metadata}}).

The client sends a token exchange request as defined in Section 2.1 of {{RFC8693}} with:

`grant_type`:
: REQUIRED. The value MUST be `urn:ietf:params:oauth:grant-type:token-exchange`.

`requested_token_type`:
: REQUIRED. The value MUST be `urn:ietf:params:oauth:token-type:software-statement`.

`subject_token` and `subject_token_type`:
: REQUIRED. The subject token is an initial access token: an authorization credential issued out of band by this authorization server that pre-authorizes software statement issuance, analogous to the initial access token of {{RFC7591}}, presented with a `subject_token_type` of `urn:ietf:params:oauth:token-type:access_token`. The credential MUST be bound to the `client_id` of the request. This version defines no other subject token type; renewal from a previously issued software statement is a deferred capability ({{deferred-capabilities}}).

`client_id`:
: REQUIRED. The client identifier URL described in {{client-identity}}.

`audience`:
: OPTIONAL. The requested audience described in {{authorization-request}}. The same syntax, validation, and narrowing rules apply.

`completion_mode`:
: REQUIRED when the authorization server advertises `deferred_token_response_supported` ({{authorization-server-metadata}}); otherwise not used, and a synchronous issuer ignores it ({{deferred-processing}}). When present, the value MUST include `deferred`, per the deferral opt-in rule of {{deferred-processing}}.

The request MUST NOT contain `actor_token` or `actor_token_type`, nor the `scope`, `resource`, or `authorization_details` parameters prohibited by {{prohibited-parameters}}.

The client authenticates according to {{client-identity}}, whose sender-constraint rules apply to the exchange; the polling rules of {{deferred-processing}} govern any resulting deferral.

The authorization server MUST validate the subject token before retrieving client-controlled metadata or enqueueing any processing. An invalid, expired, or revoked subject token, or one that does not authorize issuance for the presented `client_id`, MUST result in `invalid_request`, as Section 2.2.2 of {{RFC8693}} requires for a subject token that is invalid or unacceptable under policy. An unacceptable requested audience results in `invalid_target` {{RFC8693}}.

An initial access token presented under this profile MUST be:

* time limited;
* limited to the issuing authorization server;
* bound to an exact client identifier URL or an explicitly authorized client identifier namespace; and
* of at least 128 bits of entropy, when opaque.

A reusable initial access token MUST be sender-constrained, for example bound to a client key through DPoP or mTLS; a bearer initial access token MUST be single-use. The credential SHOULD be integrity protected and kept confidential in transit and at rest, and MAY further restrict audiences or metadata. The authorization server MUST enforce every restriction the credential carries and MUST prevent replay beyond its permitted number of uses. A use is consumed when the authorization server commits to an outcome for the request, whether it issues a statement, creates a deferral, or denies issuance; concurrent presentations of a single-use credential MUST NOT both be committed. Once a deferral exists, the client recovers the outcome by polling with the deferral code rather than by presenting the credential again.

A DPoP proof on the exchange constrains any resulting deferral; it does not authenticate the presenter or protect the subject token (Section 3 of {{RFC9449}}). The initial access token therefore needs its own sender constraint or single-use restriction.

The exchange is evaluated against current metadata. The authorization server MUST obtain and validate the Client ID Metadata Document and MUST bind a fresh metadata snapshot ({{metadata-snapshot}}) and the requested audience before returning either the software statement or a deferral code. The statement's claims derive from that snapshot. An authorization server that recorded the metadata digest of a prior issuance for this `client_id` MAY compare it against the fresh snapshot and treat a change as an input to issuance policy.

Issuance policy determines whether an initial access token authorizes only the request or issuance itself. The result is:

* a software statement token response ({{software-statement-response}}) on success;
* a {{DTR}} deferred token response from a deferred issuer, followed by polling, when processing cannot complete immediately ({{deferred-processing}}); or
* the terminal denial of {{terminal-denial}} when issuance policy denies the exchange.

# Deferred Processing {#deferred-processing}

An authorization server supports one of two conformance levels:

* A **synchronous issuer** answers every originating request with a statement or terminal denial ({{terminal-denial}}), so it MUST reach the issuance decision before responding, which in the redirect flow means before returning the software statement code. It never defers, need not implement {{DTR}}, and does not advertise `deferred_token_response_supported`.
* A **deferred issuer** returns a deferred token response when a decision needs out-of-band processing. It implements {{DTR}} and advertises `deferred_token_response_supported` ({{authorization-server-metadata}}).

Against a deferred issuer, every deferral originates from a deferred token response of {{DTR}}, issued for a software statement code redemption ({{software-statement-code-redemption}}) or a token exchange ({{token-exchange-profile}}), and the client polls the token endpoint using the polling grant defined by {{DTR}}.

A client contacting a deferred issuer MUST include the `completion_mode` parameter of {{DTR}} with a value that includes `deferred` on the originating request, and MUST support the polling grant; the issuer MAY still complete synchronously. A deferred issuer MUST reject an originating request that does not carry that opt-in with `invalid_request`, because {{DTR}} treats an absent or non-`deferred` value as a requirement for synchronous handling, which an issuer whose decisions can outlive a request cannot guarantee. This deliberately raises the parameter from OPTIONAL in {{DTR}} to REQUIRED here. A synchronous issuer neither requires nor processes `completion_mode`.

This specification profiles polling delivery only: a client MUST NOT include the `client_notification_token` parameter of {{DTR}} on any request under this specification, a request containing it is rejected with `invalid_request`, and an authorization server MUST NOT deliver callback notifications for a deferral created under this specification, regardless of any `deferred_client_notification_endpoint` in the client's metadata. Polling over the authenticated token endpoint retrieves the statement without an outbound channel, so this version does not require an issuer to operate one; a future version can adopt the callback mechanism of {{DTR}} ({{deferred-capabilities}}).

When an issuer defers a request, it MUST record the sender-constraint context established at origination and bind it to the deferral:

* the client identifier;
* the client authentication method; and
* for a method that binds a key, that specific key: the DPoP JWK thumbprint, or the authenticated key for a method such as `private_key_jwt` or mTLS.

A method that does not bind a key freezes only the client identity and method. Every polling request MUST match the recorded context. The authorization server MUST NOT re-derive that context from the current metadata document. A client that loses the origination key cannot complete the deferral and instead makes a new request.

Approval of software statement issuance can take hours or days rather than the seconds typical of user authentication, for example when it involves reviewing the client's policy or compliance documentation. Issuers SHOULD set deferral code lifetimes that reflect their actual approval latency.

## First Polling Request {#first-polling-request}

The first polling request is an HTTP `POST` request to the token endpoint using the `application/x-www-form-urlencoded` format. It contains:

`grant_type`:
: REQUIRED. The value MUST be `urn:ietf:params:oauth:grant-type:deferred`.

`deferral_code`:
: REQUIRED. The deferral code returned in the deferred response.

The polling request carries no PKCE parameter: for a redirect-flow deferral, the verifier was consumed when the software statement code was redeemed ({{software-statement-code-redemption}}). The polling request MUST NOT contain a `software_statement_code`, `code_verifier`, `redirect_uri`, `audience`, `subject_token`, `subject_token_type`, or `requested_token_type` parameter; a polling request containing any of them is rejected with `invalid_request`.

On this and every subsequent polling request, the client MUST satisfy the sender-constraint context recorded for the deferral ({{deferred-processing}}): it uses the origination client authentication method and, when that method or a DPoP sender constraint binds a key, signs with the origination key. The authorization server verifies the request against the stored context, not against the current metadata document, and MUST reject a request that does not match, including a DPoP proof whose key differs from the recorded thumbprint.

## Subsequent Polling Requests

If the request remains pending, the client continues polling according to {{DTR}} using only the polling grant's normal parameters and the sender constraint described in {{first-polling-request}}.

Pending, denied, expired, canceled, and polling-rate behavior follows {{DTR}}. A successful first or subsequent polling response is the software statement token response defined in {{software-statement-response}}.

Cancellation of a deferral follows the revocation mechanism of {{DTR}}: the client presents the deferral code to the authorization server's revocation endpoint {{RFC7009}} with a `token_type_hint` of `urn:ietf:params:oauth:token-type:deferral-code`. For a deferral created under this specification, the authorization server MUST require the deferral's sender constraint on the revocation request: client authentication where the deferral is bound to client authentication, or a DPoP proof with the origination-bound key otherwise. A revocation request that does not present the bound constraint MUST be treated as presenting an unrecognized token per {{DTR}}.

# Software Statement Token Response {#software-statement-response}

A successful response has HTTP status code 200, a media type of `application/json`, and the following members:

`access_token`:
: REQUIRED. The software statement issued by the authorization server. As in {{RFC8693}}, the historically named `access_token` member carries the issued security token.

`issued_token_type`:
: REQUIRED. The value MUST be `urn:ietf:params:oauth:token-type:software-statement`.

`token_type`:
: REQUIRED. The value MUST be `N_A`, indicating that an OAuth access token type does not apply.

`expires_in`:
: RECOMMENDED. The remaining lifetime of the software statement in seconds. If present, it MUST be consistent with the statement's `exp` claim.

The response MUST NOT contain `refresh_token` or `scope`. The authorization server MUST include `Cache-Control: no-store`; it SHOULD also include `Pragma: no-cache`.

DPoP under this specification binds requests and deferral state, not the issued artifact: the software statement is not an OAuth access token, and {{DTR}}'s requirement that a final access token inherit the originating DPoP binding does not apply because none is issued. The statement's replay and theft properties are those described in {{PRESENTATION}}, including key binding through attested `jwks` or `jwks_uri` where a deployment wants proof of possession at registration, and, for runtime presentation, the sender constraint {{PRESENTATION}} mandates.

For example:

~~~ http
HTTP/1.1 200 OK
Content-Type: application/json
Cache-Control: no-store
Pragma: no-cache

{
  "access_token":
    "eyJ0eXAiOiJzb2Z0d2FyZS1zdGF0ZW1lbnQrand0...",
  "issued_token_type":
    "urn:ietf:params:oauth:token-type:software-statement",
  "token_type": "N_A",
  "expires_in": 3600
}
~~~

The client consumes the issued statement, the value of `access_token`, as described in {{PRESENTATION}}: through an {{RFC7591}} registration request or by runtime presentation ({{PRESENTATION}}).

The `access_token` member is a security-token container ({{RFC8693}}), not an OAuth access token: the software statement is consumed only as a software statement ({{PRESENTATION}}), MUST NOT be attached to a request as an `Authorization: Bearer` credential, and is not subject to refresh. Implementations that cache issued tokens by type SHOULD key this artifact on its `issued_token_type` so that generic access-token handling does not apply to it, and SHOULD treat it as a sensitive credential in logs.

A client obtains a replacement for an expiring or expired software statement by performing a new software statement request, or, if it holds an initial access token, through an exchange under {{token-exchange-profile}}. Whether replacement requires new approval is determined by issuer policy. This document defines how a client obtains a replacement; {{PRESENTATION}} defines how it delivers one to a trusting authorization server, and orders replacements by `iat` ({{software-statement-format}}).

## Terminal Denial {#terminal-denial}

When the authorization server decides not to issue the requested software statement, whether that decision is already complete when the originating request arrives or completes during deferred processing, it returns a token error response per Section 5.2 of {{RFC6749}} with the error code `access_denied` and HTTP status code 400, and MUST include the `Cache-Control: no-store` response header field. The same rule applies to both originating requests: a software statement code redemption ({{software-statement-code-redemption}}) and a token exchange ({{token-exchange-profile}}). A decision that completes as a denial during deferred processing is delivered in response to a polling request ({{deferred-processing}}).

The denial is terminal for the request. A deferral resolves to the denied state, in which subsequent polling requests return the same `access_denied` response for the remainder of the deferral code's lifetime, as {{DTR}} requires for a request that has resolved with an error. A denial does not preclude a later issuance request; whether to accept one is issuance policy.

# Software Statement Format {#software-statement-format}

The software statement is a compact JWT {{RFC7519}} protected by JWS {{RFC7515}}. Although {{RFC7591}} permits a MAC, a statement issued under this specification MUST use an asymmetric digital signature so trusting servers need not receive an issuer-held symmetric key. The issuer and trusting authorization server MUST follow {{RFC8725}} algorithm-verification guidance. The `none` algorithm and symmetric algorithms MUST NOT be used.

The JOSE header MUST include `kid`, identifying the signing key within the issuer's JWK Set, and MUST include `typ` with the value `software-statement+jwt`, applying Section 3.11 of {{RFC8725}}. This value names `application/software-statement+jwt` ({{media-type}}) with the `application/` prefix omitted, as described in Section 4.1.9 of {{RFC7515}}. Explicit typing prevents confusion with other JWTs from the same issuer.

Extensions can add claims; an incompatible revision would use a new type value. Supported algorithms appear in `software_statement_signing_alg_values_supported` ({{authorization-server-metadata}}).

The JWT payload MUST contain the following claims in addition to the approved client metadata.

`iss`:
: REQUIRED. The issuer identifier of the issuing authorization server, as defined by {{RFC8414}}.

`sub`:
: REQUIRED. The exact client identifier URL presented in the request that produced the statement.

`aud`:
: REQUIRED. One or more audience identifiers for the authorization servers permitted to accept the statement. Each value MUST be an authorization server issuer identifier as defined by {{RFC8414}}. A trusting authorization server MUST reject the statement unless one of its locally configured audience identifiers exactly matches a value in this claim. When the request contained `audience` values, every value in this claim MUST have appeared among them.

`iat`:
: REQUIRED. A NumericDate value representing the time at which the software statement was issued. An issuer MUST NOT issue a statement for a given `iss` and `sub` pair with an `iat` earlier than one it has already issued for that pair, and SHOULD ensure the value is strictly increasing across its signing nodes, so that the order in which it made its decisions is recoverable from the statements themselves. Consumers rely on that order when one statement replaces another ({{PRESENTATION}}) and when a withdrawal separates statements issued before it from those issued after ({{SIGNALS}}). Back-dating a statement to allow for clock skew makes it unusable as a replacement.

`exp`:
: REQUIRED. A NumericDate value representing the expiration time. A trusting authorization server MUST reject an expired statement.

`jti`:
: REQUIRED. A unique identifier for the statement within the issuer's namespace.

`cimd_digest`:
: REQUIRED. The metadata digest ({{metadata-snapshot}}) of the Client ID Metadata Document from which the metadata snapshot was derived. This claim binds the statement to the exact document content evaluated during issuance and lets any party determine whether the client's currently published metadata still matches what was attested.

Each metadata claim MUST be client metadata registered in the IANA "OAuth Dynamic Client Registration Metadata" registry, or otherwise recognized by the authorization server, such as `instance_issuers` ({{PRESENTATION}}). The following authorization-server-assigned, credential, and recursive metadata members are not eligible for attestation and MUST NOT appear in a software statement issued under this specification: `client_id`, `client_secret`, `client_id_issued_at`, `client_secret_expires_at`, `registration_access_token`, `registration_client_uri`, and `software_statement`. An authorization server MAY exclude additional metadata according to policy. The `client_id` member required in the Client ID Metadata Document by {{CIMD}} identifies the canonical document during issuance; the statement represents that identifier in `sub` and MUST NOT copy it as client metadata.

The issuer determines the audience and lifetime according to policy. Lifetime carries two consequences that pull in opposite directions: a short lifetime bounds exposure and keeps the review fresh, while at a trusting authorization server implementing the registration-validity model of {{PRESENTATION}} the same value is the registration's validity and therefore the renewal cadence the client and issuer must sustain. An issuer SHOULD choose a lifetime it can renew reliably for the deployments it serves, and SHOULD stagger expiries across the statements it issues, or renew ahead of the boundary, so that a fleet issued together does not lapse together. A client can request an audience using the `audience` parameter, but the issuer MAY narrow that request and MUST NOT widen it. If the parameter is omitted, the issuer selects an audience entirely according to policy. Every client metadata claim MUST correspond to a member present in the metadata snapshot, carrying either that member's value or, for a set-valued member such as `redirect_uris`, `grant_types`, or `scope`, a subset of it. The issuer MAY omit members but MUST NOT introduce a member absent from the snapshot, alter a value that is not set-valued, or otherwise contradict or widen the snapshot.

The `sub` claim is the client identifier URL, not the local `client_id` assigned through {{RFC7591}} registration. A trusting authorization server MAY use `sub` to correlate registrations and apply per-client policy ({{PRESENTATION}}).


# Authorization Server Metadata {#authorization-server-metadata}

The issuing authorization server is a role, not necessarily a general-purpose OAuth deployment. An issuer offering only token exchange can consist of a token endpoint, an authorization server metadata document, and signing keys. It can operate solely as an attestation service, without an authorization endpoint, access tokens, or protected resources.

An authorization server that issues software statements under this specification advertises `true` for `client_id_metadata_document_supported`, as defined by {{CIMD}}, and publishes `software_statement_signing_alg_values_supported` below.

An issuer that may defer a request advertises `true` for `deferred_token_response_supported`, as defined by {{DTR}} ({{deferred-processing}}), and, because deferrals are cancellable, lists `urn:ietf:params:oauth:token-type:deferral-code` in `revocation_endpoint_token_type_values_supported` as {{DTR}} requires. A synchronous issuer, which answers every request with a statement or a terminal denial, advertises neither and need not implement {{DTR}}.

An authorization server supporting the redirect flow advertises:

* `urn:ietf:params:oauth:grant-type:software-statement` in `grant_types_supported`, the grant through which software statement codes are redeemed;
* `software_statement_code` in `response_types_supported`; and
* `true` for `authorization_response_iss_parameter_supported`, as defined by {{RFC9207}}.

An authorization server supporting the token exchange profile ({{token-exchange-profile}}) advertises `urn:ietf:params:oauth:grant-type:token-exchange` in `grant_types_supported` and publishes `software_statement_subject_token_types_supported` below; general token exchange support does not by itself imply support for this profile. An implementation supporting only that profile advertises neither the software statement grant nor the `software_statement_code` response type.

A client that requires deferral MUST NOT send a request to an authorization server that does not advertise `deferred_token_response_supported`; a synchronous issuer answers such a request with a statement or a terminal denial rather than a deferral.

This specification defines the following additional authorization server metadata members:

`software_statement_signing_alg_values_supported`:
: REQUIRED for an authorization server that issues software statements under this specification. A JSON array containing the asymmetric JWS `alg` values that the authorization server can use to sign software statements. The array MUST NOT contain `none` or a symmetric algorithm. This member describes the issuing role; an authorization server that only accepts software statements does not publish it.

`software_statement_audiences_supported`:
: OPTIONAL. A JSON array containing authorization server issuer identifiers that the issuer is prepared to place in a software statement's `aud` claim. Omission means that the complete set is not publicly enumerable; it does not mean that no audience is supported. Publication does not guarantee that every client is eligible for every listed audience.

`software_statement_subject_token_types_supported`:
: REQUIRED for an authorization server that supports the token exchange profile ({{token-exchange-profile}}), and absent otherwise. A JSON array of the `subject_token_type` values the authorization server accepts when `requested_token_type` is `urn:ietf:params:oauth:token-type:software-statement`. Publication of this member is the discovery signal for the profile.

# Security Considerations {#security-considerations}

## Client Establishment Is Not an Access Grant

A software statement attests client metadata; it grants no resource access or consent on behalf of the software's users. Each user still authorizes access through the established client.

Authorization servers MUST enforce the prohibited-parameter and response-type rules in {{prohibited-parameters}}. The successful response uses `access_token` only as the generic security-token container defined by {{RFC8693}}; the contained software statement MUST NOT be accepted as an access token at a protected resource.

When an approval interface is shown, it MUST clearly describe that the decision concerns attestation to client metadata. It MUST NOT imply that the approver is granting the client access to resources.

An erroneous approval affects every authorization server in the statement's audience until expiry. The approval interface therefore MUST present:

* the client identifier URL;
* the metadata to be attested; and
* the audience the issuer intends to place in the statement.

It SHOULD present the intended lifetime, and SHOULD make narrowing visible when the client requested a different or broader audience. Attesting `instance_issuers` ({{PRESENTATION}}) endorses the listed authorities to attest runtime instances and deserves particular scrutiny.

## What Issuance Attests {#what-issuance-attests}

A software statement is an attestation about software: a signed, attributable claim by a named issuer, bounded by that issuer's process rather than proof that its contents are true. {{PRESENTATION}} sets it beside the client attestation and instance assertion that attest a presenter, which differ in subject, authority, lifetime, and effect and compose with it rather than replace it.

A software statement means one thing: the issuer evaluated the exact document content captured in the metadata snapshot ({{metadata-snapshot}}), under its issuance policy, at the time recorded in `iat`, and decided to attest the contained metadata for the named audience.

The client authors the metadata document, so issuance does not make every value an independently verified fact. It records an accountable evaluation of a deterministic, digest-bound input ({{metadata-snapshot}}). An issuer SHOULD corroborate security-relevant metadata through evidence beyond the document itself. Verification depth is part of the trust relationship.

A trusting authorization server can conclude only what the issuer decided; local client-establishment policy determines what that decision is worth.

## Client Metadata Retrieval

Fetching a Client ID Metadata Document and resources referenced by it exposes the authorization server to server-side request forgery, resource exhaustion, malicious content, and client impersonation risks. The validation, address filtering, response-size limits, redirect handling, caching, logo handling, and domain-trust considerations of {{CIMD}} apply.

The metadata snapshot requirements in {{metadata-snapshot}} prevent a time-of-check/time-of-use change from silently altering the attested metadata after approval; members the statement does not attest remain live, and runtime presentation ({{PRESENTATION}}) sources them from the current document at each presentation. Authorization servers SHOULD record the metadata digest ({{metadata-snapshot}}) and retain the exact retrieved octets of the approved document for audit purposes; a re-serialized copy cannot reproduce the digest.

## Authorization Response Security {#authorization-response-security}

The software statement is a signed credential and can contain sensitive deployment information. It MUST NOT be returned through the authorization endpoint. The software statement code response applies exact redirect URI matching, PKCE with `S256`, and the other applicable authorization response protections in {{RFC9700}}.

Before redeeming a software statement code, the client MUST verify `state` and MUST validate the authorization response `iss` parameter according to {{RFC9207}}. PKCE with `S256` provides the cross-site request forgery protection described in {{RFC9700}}; `state` also correlates concurrent requests with their responses.

A public client presents no client authentication, so its association with the software depends on delivery to a metadata-listed redirection endpoint. Only HTTPS demonstrates control of the publisher's origin. Another application can claim a private-use or loopback endpoint ({{RFC8252}}); PKCE and DPoP bind the code to the initiator but cannot stop an attacker from initiating under another party's `client_id`.

A public client therefore MUST use an HTTPS redirection URI ({{authorization-request}}). A native public client can host such an endpoint or use the redirect-free token exchange profile ({{token-exchange-profile}}).

A confidential client's authentication protects redemption, so it MAY use a loopback or private-use redirection URI. An authorization server SHOULD additionally relate the redirection URI's origin to the client identifier URL or the metadata document's `client_uri` according to policy. Endpoint or key control informs issuance policy but does not determine issuance.

A software statement code in a URL is visible to browser history, referrer fields, logs, and other observers. It is short lived, single use, and unredeemable without the PKCE verifier and, for a public client, the `dpop_jkt` key.

Deferral codes travel only over a direct TLS connection and are protected by sender-constrained polling and cancellation ({{deferred-processing}}). Deployments sensitive to URL disclosure can use `form_post` {{FORM-POST}}; the fragment response mode SHOULD NOT be used.

## Deferral Sender Constraint

Every deferral is created by a token request, and every poll is bound to the client authentication or DPoP key fixed at origination ({{deferred-processing}}), so a public redirect-flow client's `dpop_jkt` carries sender constraint continuously from the authorization response through polling. All additional sender-constraint, polling-rate, replay, cancellation, and logging requirements of {{DTR}} apply.

## Token Exchange Considerations {#te-considerations}

A token exchange reaches the token endpoint without prior user-agent interaction. Validating the subject token before metadata retrieval or enqueueing ({{token-exchange-profile}}) limits resource consumption by unauthorized requesters. Authorization servers SHOULD still rate-limit these exchanges, cache retrieval results and failures, and bound pending deferrals per client identifier and requester. Client authentication remains mandatory when established by the Client ID Metadata Document. The redirect flow likewise reaches the approval queue before any client-authenticated step; the same rate limits and pending-approval bounds SHOULD apply per client identifier there.

A subject token is an authorization credential, not a client identifier or substitute for client authentication when the Client ID Metadata Document establishes a method. Because it appears in a form body, any component recording request bodies can expose it. Authorization servers MUST exclude subject tokens from logs, traces, error messages, and audit records; clients and authorization servers MUST protect the credential as a bearer credential unless its format provides proof of possession. The binding, lifetime, entropy, and replay requirements of {{token-exchange-profile}} limit disclosure impact.

No response parameter transits a browser, but there is also no in-band evidence of user participation. An authorization server MUST NOT treat a token exchange as implying prior user consent and MUST apply the same issuance and approval policy as for the redirect flow.

## Signing Keys and Algorithms

Compromise of a software-statement signing key enables an attacker to mint statements for every audience that trusts that key. Issuers MUST protect signing keys according to the scope of their trust relationships and SHOULD support controlled key rotation. Trusting authorization servers MUST restrict algorithms to those allowed for the issuer and MUST follow {{RFC8725}} when selecting keys and validating JWTs. Issuers SHOULD prefer signature algorithms with modern security properties, such as `PS256`, `ES256`, or `EdDSA`, over RSASSA-PKCS1-v1_5 (`RS256`); ecosystem profiles that consume software statements commonly restrict signing to such algorithms.

## Approver Identity and Audit

The software statement attests to metadata; it does not identify the human or system that approved issuance. Deployments that require approver attribution MUST retain it in an authorization server audit record or define an explicit statement claim and its privacy semantics. They MUST NOT infer approver identity from the signature alone.

Approval authority is a policy decision with audience-wide effect: an approved statement is accepted at every authorization server in its audience, not only within the approver's own scope. The policy governing who may approve issuance MUST be at least as restrictive as the policy governing manual client establishment at the issuing authorization server, and approval by a party authorized only for a personal or organizational scope MUST NOT produce a statement whose audience exceeds that scope.

An audit record SHOULD bind each decision, whether approval or denial, to the metadata digest ({{metadata-snapshot}}) of the document the deciding party evaluated, the policy under which the decision was made, the identity of that party, and the time of decision. A recorded denial carrying its grounds has the same audit value as a recorded approval. Portable, independently verifiable decision records are out of scope for this specification.

# Privacy Considerations

The authorization server learns the client identifier URL, the canonical metadata document, and information about the party interacting with the authorization endpoint. It SHOULD collect and retain only the information required for issuance, security monitoring, and audit obligations.

A software statement distributes attested client metadata to every authorization server at which the client presents it. Issuers SHOULD omit metadata that is not required by the intended audience and SHOULD choose the narrowest practical audience. Clients SHOULD NOT present statements outside their intended deployment context. Requested `audience` values reveal the authorization servers with which the client plans to establish relationships. A redirect-flow client SHOULD use Pushed Authorization Requests {{RFC9126}} when that relationship is sensitive. Authorization servers SHOULD avoid logging issued statements.

Approval records can link a person to a client and deployment. Such records SHOULD be access-controlled and retained only as long as required.

# IANA Considerations {#iana}

## OAuth Authorization Endpoint Response Types Registry

This specification requests registration of the following value in the IANA "OAuth Authorization Endpoint Response Types" registry established by {{RFC6749}}:

Response Type Name:
: `software_statement_code`

Change Controller:
: IESG

Specification Document(s):
: This specification, {{authorization-request}} and {{authorization-response}}

## OAuth URI Registry

This specification requests registration of the following values in the IANA "OAuth URI" registry:

URN:
: `urn:ietf:params:oauth:grant-type:software-statement`

Common Name:
: OAuth Software Statement Grant Type

Change Controller:
: IESG

Specification Document(s):
: This specification, {{software-statement-code-redemption}}

URN:
: `urn:ietf:params:oauth:token-type:software-statement`

Common Name:
: OAuth Software Statement Token Type

Change Controller:
: IESG

Specification Document(s):
: This specification, {{software-statement-response}} and {{token-exchange-profile}}

## OAuth Parameters Registry

This specification requests that IANA add this specification, {{authorization-request}} and {{token-exchange-profile}}, as an additional reference for the existing `audience` parameter registered by {{RFC8693}} and extend its usage location to include authorization requests. The parameter name and change controller are unchanged.

This specification also requests registration of the following value in the IANA "OAuth Parameters" registry established by {{RFC6749}}:

Parameter Name:
: `software_statement_code`

Parameter Usage Location:
: authorization response, token request

Change Controller:
: IESG

Specification Document(s):
: This specification, {{software-statement-code-response}} and {{software-statement-code-redemption}}

## OAuth Extensions Error Registry

This specification requests registration of the following error usage in the IANA "OAuth Extensions Error Registry" established by {{RFC6749}}:

Error name:
: `access_denied`

Error usage location:
: token error response

Related protocol extension:
: The software statement grant ({{software-statement-code-redemption}}) and token exchange profile ({{token-exchange-profile}}) of this specification

Change controller:
: IESG

Specification document(s):
: This specification, {{terminal-denial}}

It further requests registration of the following error usage:

Error name:
: `invalid_target`

Error usage location:
: authorization error response, token error response

Related protocol extension:
: The `software_statement_code` response type and the `audience` parameter usage of this specification

Change controller:
: IESG

Specification document(s):
: This specification, {{authorization-request}} and {{token-exchange-profile}}

## OAuth Authorization Server Metadata Registry

This specification requests registration of the following values in the IANA "OAuth Authorization Server Metadata" registry established by {{RFC8414}}:

Metadata Name:
: `software_statement_signing_alg_values_supported`

Metadata Description:
: JSON array containing the asymmetric JWS algorithms supported by the authorization server for signing software statements.

Change Controller:
: IESG

Specification Document(s):
: This specification, {{authorization-server-metadata}}

Metadata Name:
: `software_statement_audiences_supported`

Metadata Description:
: JSON array containing authorization server issuer identifiers that the issuer is prepared to place in software statement audience claims.

Change Controller:
: IESG

Specification Document(s):
: This specification, {{authorization-server-metadata}}

Metadata Name:
: `software_statement_subject_token_types_supported`

Metadata Description:
: JSON array of subject token type values accepted for exchanging into software statements; signals support for the token exchange profile.

Change Controller:
: IESG

Specification Document(s):
: This specification, {{authorization-server-metadata}} and {{token-exchange-profile}}

## Media Type Registration {#media-type}

This specification requests registration of the `application/software-statement+jwt` media type in the IANA "Media Types" registry {{RFC6838}}.

Type name:
: application

Subtype name:
: software-statement+jwt

Required parameters:
: n/a

Optional parameters:
: n/a

Encoding considerations:
: 8bit. A software statement is a JWT; JWT values are encoded as a series of base64url-encoded values separated by period ('.') characters, as registered for `application/jwt` in Section 10.3.1 of {{RFC7519}}.

Security considerations:
: See {{security-considerations}} of this specification and Section 11 of {{RFC7519}}.

Interoperability considerations:
: n/a

Published specification:
: This specification

Applications that use this media type:
: Authorization servers and clients that issue, request, or accept OAuth 2.0 software statements

Fragment identifier considerations:
: n/a

Additional information:
: File extension(s): n/a. Macintosh file type code(s): n/a.

Person & email address to contact for further information:
: Karl McGuinness, public@karlmcguinness.com

Intended usage:
: COMMON

Restrictions on usage:
: none

Author:
: Karl McGuinness, public@karlmcguinness.com

Change controller:
: IETF

## JSON Web Token Claims Registry

This specification requests registration of the following value in the IANA "JSON Web Token Claims" registry established by {{RFC7519}}:

Claim Name:
: `cimd_digest`

Claim Description:
: Unpadded base64url-encoded SHA-256 digest of the retrieved octets of the Client ID Metadata Document evaluated during software statement issuance

Change Controller:
: IESG

Specification Document(s):
: This specification, {{software-statement-format}}

--- back

# Deployment Examples {#deployment-examples}

These non-normative scenarios show two end-to-end deployments and one case outside the specification's applicability.

## Publisher Program: One Review, Many Customer Authorization Servers {#example-publisher}

TaskFlow is a workflow client used by marketplace customers. Its metadata at `https://taskflow.example/oauth/metadata.json` defines a production redirect URI and `private_key_jwt` authentication through `jwks_uri`. The marketplace's issuer is `https://issuer.marketplace.example`. Two customers trust it for TaskFlow's namespace at `https://as.customer-a.example` and `https://as.customer-b.example`.

1. TaskFlow requests `software_statement_code` and repeats `audience` for both customers ({{authorization-request}}).
2. After the developer authenticates, TaskFlow validates `state` and `iss`, then redeems the code with PKCE, `completion_mode=deferred`, and `private_key_jwt`. A two-day review defers redemption, so TaskFlow polls with the same authentication ({{software-statement-code-redemption}}, {{deferred-processing}}).
3. Approval returns a statement whose `sub` identifies TaskFlow, whose `aud` contains both customers, and whose `cimd_digest` binds the reviewed document.
4. TaskFlow presents the statement at both {{RFC7591}} registration endpoints. Each validates it and assigns a local `client_id`. Both registrations use the attested `jwks_uri` ({{software-statement-format}}, {{PRESENTATION}}).
5. Out of band, the publisher program also issued TaskFlow an initial access token scoped to its client identifier and both audiences. Before expiry, the release pipeline exchanges it for a new statement ({{token-exchange-profile}}); line breaks are for display purposes only:

~~~ http
POST /token HTTP/1.1
Host: issuer.marketplace.example
Content-Type: application/x-www-form-urlencoded

grant_type=urn%3Aietf%3Aparams%3Aoauth%3Agrant-type%3A
  token-exchange
  &requested_token_type=urn%3Aietf%3Aparams%3Aoauth%3A
  token-type%3Asoftware-statement
  &subject_token=SplxlOBeZQQYbYS6WxSbIA...
  &subject_token_type=urn%3Aietf%3Aparams%3Aoauth%3A
  token-type%3Aaccess_token
  &client_id=https%3A%2F%2Ftaskflow.example%2Foauth%2F
  metadata.json
  &audience=https%3A%2F%2Fas.customer-a.example
  &audience=https%3A%2F%2Fas.customer-b.example
  &completion_mode=deferred
  &client_assertion_type=urn%3Aietf%3Aparams%3Aoauth%3A
  client-assertion-type%3Ajwt-bearer
  &client_assertion=eyJhbGciOiJFUzI1NiIsImtpZCI6InRhc2tmbG93LTEifQ...
~~~

The marketplace completes renewal synchronously when policy permits, using the unchanged audience and matching fresh metadata digest as inputs. The digest match alone does not determine the result.

## Enterprise Deployment of a Third-Party Agent {#example-enterprise}

ACME's AI agent uses one client identifier URL, `https://acme.example/agent`, across installations. An enterprise identity platform at `https://idp.enterprise.example` operates a token-exchange-only issuer. Internal authorization servers at `https://tools-as.enterprise.example` and `https://data-as.enterprise.example` trust it for the ACME agent.

1. At onboarding, the identity platform gives its daemon an initial access token authorizing issuance for ACME. The daemon exchanges it while requesting both internal authorization servers as repeated `audience` values, with `completion_mode=deferred` and a DPoP proof ({{token-exchange-profile}}).
2. The identity platform validates the token and returns a three-day {{DTR}} deferral. The daemon polls at the advertised interval with fresh proofs from the same DPoP key and no PKCE verifier ({{first-polling-request}}). Security review occurs out of band.
3. The final poll returns the statement. The daemon registers once at each internal authorization server and supplies the enterprise's instance issuer as plain metadata ({{PRESENTATION}}). Instances authenticate through that issuer, allowing shared registrations with per-instance token identity ({{CLIENT-INSTANCE}}). Because the statement omits `jwks` and `jwks_uri`, policy could instead permit per-deployment registrations with distinct keys.

The deferred response in step 2 is shown below. The HTTP 400 status and `authorization_pending` error indicate a successfully established deferral, not rejection:

~~~ http
HTTP/1.1 400 Bad Request
Content-Type: application/json
Cache-Control: no-store

{
  "error": "authorization_pending",
  "deferral_code":
    "m9K2pL4uQ6vX8cD0fG2hJ5nR7sT1wY3aB6eP8zN4qV0",
  "expires_in": 259200,
  "interval": 300
}
~~~

The registration request at the tools authorization server in step 3 is:

~~~ http
POST /register HTTP/1.1
Host: tools-as.enterprise.example
Content-Type: application/json

{
  "software_statement": "eyJ0eXAiOiJzb2Z0d2FyZS1zdGF0ZW1l...",
  "instance_issuers": [
    {
      "issuer": "https://workload.enterprise.example",
      "jwks_uri": "https://workload.enterprise.example/jwks.json"
    }
  ]
}
~~~

Because the statement omits `instance_issuers`, the local value does not conflict with {{RFC7591}} precedence. It is unattested, so the internal authorization servers accept it under local policy, having authenticated the enterprise's own deployment tooling as the presenter ({{PRESENTATION}}). Had the agent been deployed by a party the internal servers did not already authenticate, the attested variant of {{PRESENTATION}} would carry that delegation instead.

Before expiry, the daemon renews with the same initial access token and audiences; this version does not accept a prior statement as subject token ({{token-exchange-profile}}). If an ACME update changes the metadata digest, policy defers renewal for another security review.

## The Resource Owner as Approver: A Case This Specification Does Not Serve {#example-resource-owner}

Now consider an individual running the ACME agent against their own authorization server. The resource owner is the approver, so the Introduction's applicability guidance excludes this case.

The OAuth authorization grant already records the user's approval. The generic URL identifies the client {{CIMD}}; instance assertions identify installations at the token endpoint ({{CLIENT-INSTANCE}}). Tokens can then identify the user as subject and the instance as actor, sender-constrained to the installation key.

A statement adds nothing: there is a single authorization server, a single approver already present in the transaction, and no review whose cost needs amortizing. {{example-enterprise}} reverses each condition.

# Issuance Across the Software Delivery Lifecycle {#delivery-lifecycle}

This non-normative appendix places review, attestation, renewal, and expiry in the software-delivery lifecycle. Renewal here means re-issuing a statement; renewing the registration a statement governs is the consumption-side operation {{PRESENTATION}} defines. A publisher or enterprise reviews once and issues a statement ({{example-publisher}}, {{example-enterprise}}). A release pipeline renews through token exchange; a changed digest prompts a carry-forward-or-re-review decision ({{token-exchange-profile}}). Sunset begins with the absence of renewal, which stops further statement use; registrations already derived from a statement expire with it at a server implementing the registration-validity model of {{PRESENTATION}}, and elsewhere their retirement remains that server's own lifecycle work ({{PRESENTATION}}).

The statement attests the software, not its instances ({{PRESENTATION}}). It does not attest binaries or build provenance, identify running instances, or grant access; identifying the presenter is the work of a client attestation or instance assertion ({{ABCA}}, {{CLIENT-INSTANCE}}), which composes with the statement rather than substituting for it ({{what-issuance-attests}}). It standardizes the vouching layer of delivery, which no other layer of the stack defines.

# Design Rationale

## Comparison with Related Establishment Mechanisms {#comparison}

The table compares where each establishment mechanism decides trust, its input, and its cost across M authorization servers.

| Mechanism | Decision made at | Input evaluated | Cost for M authorization servers |
| --- | --- | --- | --- |
| Pre-registration of a client identifier URL ({{CIMD}}) | each authorization server | self-asserted metadata document | M decisions |
| Pushed client registration ({{PUSHED-DCR}}) | each authorization server | self-asserted pushed metadata, per transaction | M decisions |
| Approval-based registration ({{APPROVAL-DCR}}) | each authorization server | self-asserted registration request | M approvals |
| This specification | the issuing authorization server | canonical metadata document | one issuance decision, M policy evaluations of an attested artifact |
| OpenID Federation ({{OPENID-FED}}) | resolved along a trust chain | entity statements | federation infrastructure |

The bilateral mechanisms compose with statements: pre-registration or pushed registration can carry attested metadata, and approval-based registration can evaluate an issuer's decision. For one authorization server, they remain sufficient alone; portability justifies issuance. OpenID Federation supplies trust chains when pairwise issuer configuration no longer scales.

OpenID for Verifiable Credential Issuance {{OID4VCI}} has a similar flow but serves wallet-held credentials presented to verifiers. A software statement describes software and is consumed by {{RFC7591}} registration or runtime presentation ({{PRESENTATION}}). An OID4VCI profile would still need this specification's format, metadata binding, and consumption rules, although an existing ecosystem could carry the statement as a credential payload.

## Why Use the Authorization Endpoint

The authorization endpoint supplies redirect URI validation, request correlation, PKCE, and issuer identification for browser-mediated interaction. A short-lived, sender-constrained code keeps the statement out of the browser; the token endpoint delivers it over a direct TLS connection. Without in-band interaction, a pre-authorized client instead uses token exchange ({{token-exchange-profile}}).

## Why Not the Device Authorization Grant

The device authorization grant {{RFC8628}} assumes an interactive user co-present with a constrained device. Software-statement approval can instead involve a remote administrator and a long-running out-of-band workflow. Token endpoint deferral covers that case without a verification URI or user code; the redirect flow covers interactive authentication or consent.

## Why the Issuer Is an Authorization Server Role

An alternative design defines a standalone issuer and leaves the minting interface out of scope, as {{CLIENT-INSTANCE}} does for instance issuers. That works for intra-domain instance attestation. It does not work here: {{RFC7591}} already standardized registration-time presentation, so minting is the missing interface.

The authorization server role reuses token endpoint authentication, {{DTR}} deferral, {{RFC8693}} token exchange, and {{RFC8414}} discovery. A minimal issuer needs only a token endpoint, metadata, and signing keys ({{authorization-server-metadata}}). Explicit `typ` values separate statement and instance issuers.

## Why the Software Statement Code Is Not an Authorization Code

The code is distinct because successful `authorization_code` redemption issues an access token under {{RFC6749}}. OpenID Connect returns an ID Token alongside, not instead of, an access token. A separate code preserves a code-shaped front channel while keeping deferral at the token endpoint.

## Why Token Exchange Is a Profile, Not the Only Flow

Token exchange requires a `subject_token` {{RFC8693}}, which a first-time client lacks. It derives authority from that token; the redirect flow derives authority from a new issuance decision. Thus token exchange serves pre-authorized clients, while the redirect flow serves those without a token.

## Why Deferral Happens Only at the Token Endpoint

A short-lived code keeps statements and long-lived credentials out of the browser. Token endpoint redemption returns the statement or defers through {{DTR}}, so every deferral originates from a token request.

An {{RFC7591}} registration endpoint creates a local client, and {{APPROVAL-DCR}} defers that operation there. This specification instead mints a portable artifact for registration elsewhere; placing issuance at registration would validate metadata without creating a client. The token endpoint already mints security tokens and hosts the pre-authorized path ({{token-exchange-profile}}). Using it for both flows reuses its authentication and {{DTR}} polling, cancellation, sender constraint, and rate limiting.

## Why the Response Uses `access_token`

The response follows {{RFC8693}}: `access_token` carries the artifact, `issued_token_type` identifies it, and `token_type=N_A` excludes access-token semantics. Immediate and deferred responses therefore use existing token endpoint processing without implying resource access.

## Deliberately Deferred Capabilities {#deferred-capabilities}

This version omits seven capabilities, each with an extension point:

* **Token-endpoint initiation:** a client without a user agent or pre-authorizing credential cannot initiate issuance. An extension can use the software statement grant without `software_statement_code` and advertise that mode in metadata.
* **Statement-as-subject renewal:** renewal uses an initial access token or a new request ({{token-exchange-profile}}), not a prior statement. `software_statement_subject_token_types_supported` can advertise a future holder-bound subject-token profile.
* **Request-time metadata selection:** only issuer-side narrowing is supported ({{software-statement-format}}). An extension can define a selection parameter.
* **Callback delivery:** this version permits polling only ({{deferred-processing}}). A future version can adopt {{DTR}} callbacks.
* **Canonicalized digests:** serialization changes alter the octet digest ({{metadata-snapshot}}). An extension can define a canonicalized digest claim or parameter.
* **Acceptance-time status:** lifetime is the only in-band revocation control ({{PRESENTATION}}); under {{PRESENTATION}} it expires a statement-governed registration outright and, as a matter of local policy, ends a runtime-established grant at refresh. An extension can define a status claim, for example over a Token Status List {{STATUS-LIST}}, with the processing rules a trusting authorization server applies.
* **CIMD-native conveyance:** runtime presentation ({{PRESENTATION}}) carries the statement in the request, and the registration path carries it in a registration request. A further profile could let a Client ID Metadata Document reference where a client publishes its current statements, so a resolving server fetches the review out of band with neither; {{PRESENTATION}} lists it as an extension point. This differs from embedding a statement in the document, which the digest rule of {{metadata-snapshot}} forbids.

# Acknowledgments

This specification builds on {{DTR}}, {{CIMD}}, and {{APPROVAL-DCR}}. The author thanks the authors and contributors to those specifications and the OAuth working group participants whose discussions informed this work.
