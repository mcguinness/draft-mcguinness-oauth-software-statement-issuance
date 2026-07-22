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
  RFC7515:
  RFC7519:
  RFC7591:
  RFC7636:
  RFC8414:
  RFC8693:
  RFC8707:
  RFC8725:
  RFC8785:
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
  OIDC-CORE:
    target: https://openid.net/specs/openid-connect-core-1_0.html
    title: "OpenID Connect Core 1.0"
  OAUTH-MRT:
    target: https://openid.net/specs/oauth-v2-multiple-response-types-1_0.html
    title: "OAuth 2.0 Multiple Response Type Encoding Practices"
  FORM-POST:
    target: https://openid.net/specs/oauth-v2-form-post-response-mode-1_0.html
    title: "OAuth 2.0 Form Post Response Mode"

informative:
  RFC8628:
  APPROVAL-DCR:
    target: https://datatracker.ietf.org/doc/draft-dellaert-oauth-approval-based-dcr
    title: "OAuth 2.0 Approval-Based Dynamic Client Registration"
  AU-CDR:
    target: https://consumerdatastandardsaustralia.github.io/standards/
    title: "Consumer Data Standards (Australia)"
  CIBA:
    target: https://openid.net/specs/openid-client-initiated-backchannel-authentication-core-1_0.html
    title: "OpenID Connect Client-Initiated Backchannel Authentication Flow - Core 1.0"
  CLIENT-INSTANCE:
    target: https://datatracker.ietf.org/doc/draft-mcguinness-oauth-client-instance-assertion
    title: "OAuth 2.0 Client Instance Assertion"
  OID4VCI:
    target: https://openid.net/specs/openid-4-verifiable-credential-issuance-1_0.html
    title: "OpenID for Verifiable Credential Issuance 1.0"
  OPENID-FED:
    target: https://openid.net/specs/openid-federation-1_0.html
    title: "OpenID Federation 1.0"
  PUSHED-DCR:
    target: https://datatracker.ietf.org/doc/draft-richer-oauth-pushed-client-registration
    title: "OAuth 2.0 Pushed Client Registration"
  TRUST-FRAMEWORK:
    target: https://datatracker.ietf.org/doc/draft-mcguinness-oauth-id-assertion-framework
    title: "OAuth Identity Assertion Trust Framework"
  UK-OPEN-BANKING:
    target: https://openbankinguk.github.io/dcr-docs-pub/v3.3/dynamic-client-registration.html
    title: "Open Banking UK Dynamic Client Registration Specification v3.3"

--- abstract

This specification defines OAuth 2.0 flows through which a client requests a software statement, as defined in RFC 7591, from an authorization server. The redirect flow uses a new authorization endpoint `response_type` value, `software_statement_code`. The backchannel flow uses a new extension grant. A Client ID Metadata Document identifies a client that has not been registered with the authorization server. An optional `registration` parameter selects a request-specific subset of the metadata to be attested.

The authorization endpoint returns a short-lived software statement code that the client redeems at the token endpoint. A decision that has already completed yields the software statement directly; one that requires additional processing or approval is deferred using the mechanism of Deferred Token Response, and the client polls for the eventual statement. The software statement itself is never placed in an authorization response URL.

A client without access to a user agent can instead send a backchannel software statement request directly to the token endpoint using the extension grant. Issuance that cannot complete immediately then uses the deferred token response defined by Deferred Token Response, without any redirect. A client that already holds a pre-authorized credential or an unexpired statement can instead obtain a statement through OAuth 2.0 Token Exchange (RFC 8693).

--- middle

# Introduction

Section 2.3 of {{RFC7591}} defines a software statement as a JSON Web Token (JWT) that asserts client metadata. A client can present the software statement to a dynamic client registration endpoint, where the statement's signature enables the authorization server to determine who attested to the metadata. {{RFC7591}} does not define a protocol for obtaining a software statement.

In practice, software statements are commonly issued through manual provisioning, deployment-specific portals, or proprietary federation processes. Regulated ecosystems have each built this function independently: the UK Open Banking Directory and the Australian Consumer Data Right Register both operate central authorities that issue software statements, which participants must present to member authorization servers under ecosystem profiles of {{RFC7591}} registration ({{UK-OPEN-BANKING}}, {{AU-CDR}}). Each defined its own issuance interface because no standard one exists. These mechanisms do not give clients an interoperable way to request a statement, redirect a user or administrator for interaction, and retrieve the resulting credential without exposing it through the browser.

The client establishment mechanisms adjacent to this specification are bilateral: pre-registration of a client identifier URL as described by {{CIMD}}, pushed registration of ephemeral clients {{PUSHED-DCR}}, and approval-gated registration {{APPROVAL-DCR}} each establish trust at a single authorization server, and each asks that authorization server to decide on the client's own assertion of its metadata. A software statement changes both properties: the metadata a trusting authorization server evaluates originates from an issuer's signed issuance decision rather than from the client alone ({{what-issuance-attests}}), and one issuance decision is honored at every authorization server in the statement's audience rather than being repeated at each ({{comparison}}). {{beyond-pre-registration}} examines the closest of these mechanisms in detail.

The consumption side of this design is already standardized and deployed: {{RFC7591}} registration endpoints have accepted the `software_statement` request member since 2015. This specification supplies the missing issuance protocol and hardens the artifact so that independent issuers and consumers interoperate ({{software-statement-format}}). The resulting portability is bounded by trust: a statement is honored only where the trusting authorization server is configured to trust its issuer, so its practical audience is an ecosystem or administrative domain rather than the open web.

The protocol introduces `response_type=software_statement_code` at the authorization endpoint and one extension grant, `urn:ietf:params:oauth:grant-type:software-statement`, for initiating a backchannel request at the token endpoint. Clients identify themselves with the HTTPS URL and metadata document defined by {{CIMD}}. In the redirect flow, the authorization server returns a short-lived software statement code that the client redeems at the token endpoint; a request whose decision is still pending is deferred there under {{DTR}} and resolved by polling. A client without access to a user agent instead initiates the request with the software statement grant. A client that already holds a token carrying issuance authority, such as a qualifying initial access token or an unexpired statement, can instead use OAuth token exchange ({{token-exchange-profile}}).

The flow concerns client establishment, not authorization to access a protected resource. Consequently, a software statement request cannot be combined with `scope`, `resource`, `authorization_details`, or an access-token-producing response type.

The redirect flow requires the client to operate a redirection endpoint and to reach the authorization endpoint through a user agent. For software without access to a user agent, such as a daemon or command-line tool, this specification defines a backchannel flow ({{backchannel-request}}) in which the client requests the software statement directly at the token endpoint and any approval completes out of band through deferred processing.

Not every client establishment problem calls for a software statement. When the approving party is the resource owner and the client needs only the authorization server that resource owner uses, the approval is the OAuth authorization grant itself, and identifying the client by its metadata document {{CIMD}} suffices. A software statement earns its cost when the trust decision is made by a party other than the user in a transaction, must be made before any transaction exists, or must be made once and honored at many authorization servers. Issuance is therefore deliberately decoupled from any access-granting transaction and can complete out of band over hours or days. {{deployment-examples}} illustrates these boundaries with end-to-end scenarios.

## What Pre-Registration Does Not Solve {#beyond-pre-registration}

Pre-registration, as described by {{CIMD}}, lets a client enroll its client identifier URL with an authorization server before its first authorization request: the authorization server fetches the metadata document, applies policy or human review, and records a local decision. For a client and an authorization server that already know each other, that is the right tool, and this specification does not replace it.

Pre-registration changes when the trust decision is made. It does not change anything else about the decision:

* The decider is still each consuming authorization server, and its only input is still the client's self-asserted document. An enrollment reviewed by a human is a human reading the client's own claims about itself; repeating that review at M authorization servers produces M independent judgments of the same self-assertion, by parties with no particular knowledge of the software.
* The output is still local state. Nothing transfers: an approval at one authorization server is invisible to every other, cannot be presented elsewhere, and cannot amortize the cost of a careful review.
* The subject of the decision is a URL, not content. The document behind the URL changes; what exactly was approved, and whether the currently published metadata is still what was reviewed, has no interoperable answer.
* Every later evaluation still depends on dereferencing the document, with the availability and retrieval risks that implies.

A software statement changes each of these properties. The decision is made once, by a party positioned to evaluate the software (an enterprise identity function, a software publisher's program, an ecosystem operator). The output is a signed artifact honored at every authorization server in its audience. The decision is bound to the exact document content evaluated ({{metadata-snapshot}}). And the artifact verifies offline against issuer keys, with no fetch at acceptance time. Deployments where each protected service fronts its own small authorization server, none staffed to review client software, are the motivating case: pre-registration asks each of them to operate an approval process, while a statement asks each of them to verify a signature under a configured issuer.

The two mechanisms compose rather than compete: a pre-registration push that carries a software statement gives the enrolling authorization server an issuer's signed decision to evaluate instead of self-asserted values alone ({{comparison}}).

## Relationship to Other Specifications

This specification uses the following building blocks:

* {{RFC7591}} defines the software statement and the client metadata carried in it.
* {{CIMD}} defines the client identifier and the canonical source of client metadata, including pre-registration with an authorization server and the handling of changed metadata. The flows in this specification can serve as the approval-carrying enrollment step, and the canonical digest ({{metadata-snapshot}}) makes metadata change precisely detectable.
* Section 7.2.1 of {{OIDC-CORE}} defines the syntax of the `registration` authorization request parameter. This specification defines a more constrained use of that parameter for selecting metadata to be attested.
* {{DTR}} defines client opt-in, deferred token responses at the token endpoint, polling, cancellation, callbacks, and sender constraint. This specification profiles that token endpoint deferral for every flow and defines each originating request's successful response.
* {{RFC8693}} establishes the token response convention used to return a security token that is not an OAuth access token. Its token exchange grant is profiled in {{token-exchange-profile}} for clients that already hold a token carrying issuance authority.

{{APPROVAL-DCR}} defines an approval-based extension to the dynamic client registration endpoint that ultimately creates an authorization-server-specific `client_id` and, when applicable, client credentials. This specification instead issues a signed software statement that can be presented in a later {{RFC7591}} registration request. The two specifications address different deployment models, do not depend on each other, and compose: an approval-gated registration whose request carries a software statement gives its approver an issuer's signed decision to evaluate instead of only client-asserted values.

{{PUSHED-DCR}} explores pushing client registration alongside the start of an authorization flow at the same authorization server. A trusting authorization server that supports such a mechanism can accept a software statement issued under this specification within that pushed registration, sparing the user consecutive interactive flows. That composition occurs at the trusting authorization server and is out of scope here; combining issuance itself with an access-granting response type remains prohibited ({{prohibited-parameters}}), because the party that attests client metadata is not necessarily a party from which the client will request access.

{{CLIENT-INSTANCE}} addresses the instance layer: a single `client_id`, including one established by presenting a software statement, abstracts over many runtime instances, which prove themselves at the token endpoint through assertions from instance issuers endorsed in the client's metadata. The two specifications compose as layers: this specification carries the trust decision about the client software to the trusting authorization server, and {{CLIENT-INSTANCE}} carries per-instance identity into issued tokens. {{multi-instance}} describes the composition, including attestation of the `instance_issuers` metadata itself.

OpenID Federation {{OPENID-FED}} also conveys third-party-attested client metadata across authorization servers, through entity statements resolved along trust chains anchored in a federation operator. That architecture suits ecosystems prepared to operate trust anchors and intermediates and to adopt its resolution model. This specification targets deployments below that threshold: it issues the {{RFC7591}} artifact that existing registration endpoints already accept, with issuer trust established by explicit pairwise configuration rather than chain resolution. The two address different scales of trust establishment and do not depend on each other.

The pairwise issuer configuration this specification assumes can itself be generalized. The OAuth Identity Assertion Trust Framework {{TRUST-FRAMEWORK}} defines an authority delegation model in which a verifier publishes the conditions an assertion issuer must satisfy, instead of enumerating acceptable issuers, and evaluates published evidence when an assertion is presented; membership in an OpenID Federation is one such form of evidence. Applied to software statements, that model would let a trusting authorization server evaluate a previously unconfigured issuer and verify that the owner of the client identifier's namespace has authorized that issuer to attest software under it. Such a profile, including a subject identifier extraction for client identifier URLs and a binding at the registration endpoint, is out of scope for this specification.

This specification does not define the approval workflow, the identity of an approver, or a protocol between the authorization server and an external approval system. It also does not require a trusting authorization server to accept any particular issuer. Acceptance policy and trust establishment remain deployment-specific.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

OAuth terminology is defined by {{RFC6749}}. Client metadata and software statement terminology is defined by {{RFC7591}}. Client ID Metadata Document terminology is defined by {{CIMD}}. Deferred response terminology is defined by {{DTR}}.

This specification additionally defines the following terms:

Software Statement Request:
: A request in which a client asks an authorization server to issue a software statement, sent either as an authorization request with `response_type=software_statement_code` (the redirect flow) or as a `urn:ietf:params:oauth:grant-type:software-statement` grant request without a `software_statement_code` (the backchannel flow).

Issuing Authorization Server:
: The authorization server that processes a software statement request and signs the resulting software statement.

Trusting Authorization Server:
: An authorization server that accepts the issued software statement in a dynamic client registration request.

Registration Overlay:
: A JSON object carried in the `registration` request parameter that selects request-specific values from the client's canonical metadata. The overlay cannot add to or widen that metadata.

Software Statement Code:
: The short-lived, single-use artifact returned by the software statement code response ({{software-statement-code-response}}) and redeemed at the token endpoint ({{software-statement-code-redemption}}). It is not an authorization code: redeeming it yields the software statement token response or a deferred token response, never an access token.

Metadata Snapshot:
: The validated canonical metadata and accepted registration overlay bound to a request, as defined in {{metadata-snapshot}}.

Canonical Digest:
: The unpadded base64url encoding of the SHA-256 hash of a metadata document in the canonical form of {{RFC8785}}, as defined in {{metadata-snapshot}}.

# Protocol Overview

A client initiates a software statement request through one of two flows. The redirect flow uses the authorization endpoint and a user agent and supports interactive authentication or consent during issuance. The backchannel flow uses only the token endpoint and serves software without access to a user agent. In every flow, the authorization server either completes the issuance decision synchronously or defers it at the token endpoint under {{DTR}}. A client that already holds a qualifying credential or a prior statement obtains a statement through token exchange instead ({{token-exchange-profile}}).

## Redirect Flow Overview

~~~
+--------+                                  +----------------------+
| Client |                                  | Authorization Server |
+--------+                                  +----------------------+
    |                                                 |
    | (A) Authorization request                       |
    |     response_type=software_statement_code       |
    |     client_id=<metadata document URL>           |
    |     code_challenge, [registration],             |
    |     [audience], [dpop_jkt]                      |
    |------------------------------------------------>|
    |                                                 |
    | (B) Software statement code response            |
    |     (software_statement_code, state, iss)       |
    |<------------------------------------------------|
    |                                                 |
    | (C) Redemption                                  |
    |     (software_statement_code, code_verifier)    |
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

At (A), the client requests a software statement. If a `registration` overlay is present, the client uses PAR {{RFC9126}}. The authorization server validates the client identifier, redirect URI, canonical metadata, overlay, and request binding parameters; it can perform user-agent interaction or initiate an approval workflow, and returns the software statement code response at (B).

At (C), the client redeems the `software_statement_code` at the token endpoint with the software statement grant and its PKCE verifier. If the issuance decision has already completed, redemption returns the software statement at (D); otherwise the redemption is deferred under {{DTR}}, returning a `deferral_code`, and the client polls at (E) until it receives the final response. Authorization endpoint and polling errors follow {{RFC6749}} and {{DTR}}, respectively.

## Backchannel Flow Overview

~~~
+--------+                                  +----------------------+
| Client |                                  | Authorization Server |
+--------+                                  +----------------------+
    |                                                 |
    | (A) Token request                               |
    |     grant_type=...:software-statement           |
    |     client_id=<metadata document URL>           |
    |     [registration],                             |
    |     [audience]                                  |
    |------------------------------------------------>|
    |                                                 |
    | (B1) Software statement response                |
    |<------------------------------------------------|
    |                     or                          |
    | (B2) Deferred token response                    |
    |      (authorization_pending, deferral_code,     |
    |       expires_in, interval)                     |
    |<------------------------------------------------|
    |                                                 |
    | (C) Poll(s) (deferral_code)                     |
    |------------------------------------------------>|
    |     authorization_pending or final response     |
    |<------------------------------------------------|
~~~

At (A), the client sends the software statement grant request directly to the token endpoint, authenticating according to {{client-identity}} when the Client ID Metadata Document establishes an authentication method and optionally including a `registration` overlay in the request body. If issuance completes immediately, the authorization server returns the software statement at (B1). If issuance remains pending, the authorization server returns the deferred token response of {{DTR}} at (B2), and the client polls at (C) until it receives the final response. {{backchannel-request}} defines this flow.

# Client Identification and Authentication {#client-identity}

The `client_id` in a software statement request MUST be a client identifier URL conforming to {{CIMD}}. The authorization server MUST obtain and validate the corresponding Client ID Metadata Document according to {{CIMD}}. A non-URL identifier issued by the authorization server is not supported by this specification.

In the redirect flow, the authorization server MUST compare the `redirect_uri` in the request with the `redirect_uris` in the Client ID Metadata Document according to {{CIMD}} and {{RFC9700}}. The authorization server MUST NOT redirect the user agent when the client identifier or redirect URI is missing or invalid.

The client uses the `token_endpoint_auth_method` and related key metadata in its Client ID Metadata Document when authenticating to the PAR and token endpoints. If the metadata does not establish a client authentication method usable at the authorization server, the client is treated as a public client.

A public client using the redirect flow MUST include `dpop_jkt` in the authorization request and MUST present a DPoP proof signed with the corresponding key at redemption and on every polling request. A public client using the backchannel flow ({{backchannel-request}}) MUST instead include a DPoP proof on the initiating token request; the authorization server MUST bind any resulting deferral state to that proof's key, and the client MUST use the same key on every polling request. A confidential client MAY use `dpop_jkt` in addition to its client authentication method. All uses of DPoP MUST follow {{RFC9449}} and {{DTR}}.

## Metadata Snapshot {#metadata-snapshot}

Client metadata documents can change while a request is pending. Before returning a software statement code or a deferral code, the authorization server MUST bind it to the validated canonical metadata and registration overlay. The bound values constitute the metadata snapshot for the request.

The authorization server MAY retrieve the document again before issuing the software statement. If it does so and detects a security-relevant change (for example, a change to `jwks`, `jwks_uri`, `redirect_uris`, `token_endpoint_auth_method`, or any metadata selected by the overlay), it MUST either re-evaluate the request under the new metadata or reject the request. It MUST NOT silently combine values from different document versions.

Because JSON member ordering and serialization details carry no meaning, comparing retrieved documents byte for byte over-detects change. Before calculating a canonical digest, the authorization server MUST verify that the document can be represented as the I-JSON input required by {{RFC8785}}. In particular, it MUST reject a document containing duplicate object member names or a number that cannot be represented as an IEEE 754 double-precision value without loss.

The canonical digest of a metadata document is the unpadded base64url encoding of the SHA-256 hash {{RFC6234}} of the UTF-8 bytes produced by applying {{RFC8785}} to the complete document. Two retrievals with the same canonical digest contain the same metadata for purposes of this specification; a changed digest is the concrete signal for the re-evaluation requirement above. The canonical digest also serves the `cimd_digest` claim ({{software-statement-format}}) and the audit guidance in {{security-considerations}}. Deployments MAY use other deterministic encodings internally for change detection, but interoperable uses of the canonical digest MUST use the form defined here.

# Software Statement Authorization Request {#authorization-request}

The client sends an authorization request as described in Section 4.1.1 of {{RFC6749}}, with the following parameters:

`response_type`:
: REQUIRED. The value MUST be `software_statement_code`.

`client_id`:
: REQUIRED. The client identifier URL described in {{client-identity}}.

`redirect_uri`:
: REQUIRED. The value MUST exactly match one of the redirection URIs in the Client ID Metadata Document, subject to the redirect URI rules of {{CIMD}}.

`state`:
: REQUIRED. An opaque value used by the client to bind the authorization response to its request. The authorization server MUST return the exact value in the authorization response.

`code_challenge`:
: REQUIRED. A PKCE challenge as defined by {{RFC7636}}.

`code_challenge_method`:
: REQUIRED. The value MUST be `S256`.

`response_mode`:
: OPTIONAL. The mechanism for returning authorization response parameters, as defined by {{OAUTH-MRT}}. The default response mode for `response_type=software_statement_code` is `query`. Response mode considerations are given in {{software-statement-code-response}}.

`registration`:
: OPTIONAL. A JSON object containing the registration overlay described in {{registration-overlay}}. A request containing this parameter MUST be submitted using PAR {{RFC9126}}.

`audience`:
: OPTIONAL. A target service at which the client intends to use the issued software statement, with the semantics of the `audience` parameter defined in Section 2.1 of {{RFC8693}}. The parameter MAY be repeated to request multiple audiences. Each value MUST be an authorization server issuer identifier as defined by {{RFC8414}}, values MUST NOT be repeated, and order is not significant. The authorization server determines the final audience according to policy, but when this parameter is present every value in the statement's `aud` claim MUST have appeared in the request. If none of the requested audiences is acceptable, the authorization server MUST reject the request with `invalid_target`, defined by {{RFC8693}} and used on authorization requests following the precedent of {{RFC8707}}. Some deployments use a proprietary `audience` authorization request parameter to select an API for access tokens; the semantics defined here apply only to software statement requests.

`dpop_jkt`:
: REQUIRED for a public client and OPTIONAL for a confidential client. The parameter has the semantics defined in Section 10 of {{RFC9449}}. When present, its value MUST be associated with the resulting software statement code and with any deferral state derived from its redemption.

The authorization server MUST reject with `invalid_request` a request that omits a required PKCE parameter or, for a public client, omits `dpop_jkt`.

The following is a non-normative example of an authorization request (line breaks are for display purposes only):

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

A software statement request does not grant access to a protected resource. The following top-level authorization request parameters MUST NOT be present:

* `scope`;
* `resource`, as defined by {{RFC8707}}; and
* `authorization_details`, as defined by {{RFC9396}}.

An authorization server MUST reject a request containing any of these parameters with `invalid_request`. A `scope` member inside the `registration` JSON object is client metadata and is not the OAuth authorization request parameter prohibited by this section. The `audience` parameter ({{authorization-request}}) names the authorization servers at which the issued statement will be presented, a property of the requested artifact rather than a request for access, and is not prohibited by this section.

Hybrid response types that combine `software_statement_code` with `code`, `token`, `id_token`, or any other response type are not defined. An authorization server MUST reject such a request with `unsupported_response_type`.

## Registration Overlay {#registration-overlay}

The `registration` parameter syntax is defined in Section 7.2.1 of {{OIDC-CORE}}. Its value is a JSON object. In this specification, each member MUST be client metadata registered in the IANA "OAuth Dynamic Client Registration Metadata" registry, recognized by the authorization server, and eligible for attestation under this section.

The following authorization-server-assigned, credential, and recursive metadata members are not eligible for attestation and MUST NOT appear in an overlay or in a software statement issued under this specification: `client_id`, `client_secret`, `client_id_issued_at`, `client_secret_expires_at`, `registration_access_token`, `registration_client_uri`, and `software_statement`. An authorization server MAY exclude additional metadata according to policy. The `client_id` member required in the Client ID Metadata Document by {{CIMD}} identifies the canonical document during issuance; the statement represents that identifier in `sub` and MUST NOT copy it as client metadata.

The overlay selects values from the canonical metadata in the Client ID Metadata Document. It MUST satisfy the same I-JSON input requirements used for the canonical digest in {{metadata-snapshot}}. The authorization server MUST apply the following rules:

1. For metadata whose value is an array, every overlay value MUST occur in the corresponding canonical array. Two JSON values are equal for this comparison when their {{RFC8785}} serializations are byte-for-byte identical.
2. For space-delimited metadata, every space-delimited value in the overlay MUST occur among the space-delimited values of the corresponding canonical value.
3. For all other metadata, the {{RFC8785}} serialization of the overlay value MUST be byte-for-byte identical to the serialization of the corresponding canonical value.
4. A member absent from the canonical metadata MUST NOT be introduced by the overlay.
5. An unrecognized member or a member that violates these rules MUST cause the request to be rejected with `invalid_request`.
6. If the overlay contains `redirect_uris`, that array MUST contain the `redirect_uri` used for the authorization request, so that the redirection endpoint delivering the response remains within the attested metadata.

For example, the following overlay selects one redirection URI and a subset of the grant types and scope values present in the client's canonical metadata:

~~~ json
{
  "redirect_uris": ["https://client.example.org/cb"],
  "grant_types": ["authorization_code"],
  "scope": "read"
}
~~~

When an overlay is accepted, its values replace the corresponding values in the metadata snapshot. The authorization server MAY omit additional metadata from the software statement according to policy, but it MUST NOT add a value that contradicts or widens the snapshot.

The `registration` parameter is not a container for approval notes, business justification, requested JWT claims, or deployment-specific context. A profile that needs such information MUST define separate parameters, including their validation, privacy, and IANA requirements.

# Authorization Response {#authorization-response}

After validating the request and performing any immediate interaction, the authorization server returns the software statement code response or an error. The authorization server MUST NOT place the software statement or approval-sensitive information in any authorization response.

## Software Statement Code Response {#software-statement-code-response}

The authorization server returns the following parameters to the client's redirection endpoint through the user agent, using the selected response mode, whether or not the issuance decision has already completed:

`software_statement_code`:
: REQUIRED. A short-lived, single-use artifact redeemed at the token endpoint ({{software-statement-code-redemption}}). It MUST be bound to the client identifier, redirect URI, PKCE challenge, metadata snapshot (which incorporates any accepted registration overlay), and requested audience. If `dpop_jkt` was present, it MUST also be bound to that JWK thumbprint. The value MUST contain at least 128 bits of entropy from a cryptographically secure random source, MUST be opaque to the client, MUST expire shortly after issuance, and MUST NOT be accepted more than once.

`state`:
: REQUIRED. The exact value received in the authorization request.

`iss`:
: REQUIRED. The authorization server issuer identification parameter defined by {{RFC9207}}.

A software statement code is not an authorization code and MUST NOT be redeemable as one. Possession of one confers nothing by itself: redemption requires the PKCE verifier and, for a public client, a DPoP proof with the `dpop_jkt` key, so the default query response mode is acceptable exactly as it is for authorization codes. The fragment response mode SHOULD NOT be used, because scripts at the redirection endpoint can access the fragment for no benefit. A client MAY request the `form_post` response mode {{FORM-POST}} to keep the code out of URLs, browser history, and Referer headers where logging hygiene warrants it.

For example, using the default query response mode (line breaks are for display purposes only):

~~~ http
HTTP/1.1 302 Found
Location: https://client.example.org/cb?
  software_statement_code=V7e1gP8zT2mN4qR6sW9xY3aB5cD7fH0jK2pL4uQ6vX8&
  state=4L7xQ2mN9pR6sT1vW8yZ3aB5cD0fG2hJ&
  iss=https%3A%2F%2Fserver.example.com
~~~

Errors follow Section 4.1.2.1 of {{RFC6749}}.

# Software Statement Code Redemption {#software-statement-code-redemption}

The client redeems a software statement code by sending an HTTP `POST` request to the token endpoint using the `application/x-www-form-urlencoded` format with:

`grant_type`:
: REQUIRED. The value MUST be `urn:ietf:params:oauth:grant-type:software-statement`.

`software_statement_code`:
: REQUIRED. The software statement code returned by the authorization endpoint. Its presence distinguishes a redemption from a backchannel initiation ({{backchannel-request}}).

`redirect_uri`:
: REQUIRED. The same redirection URI used in the authorization request.

`client_id`:
: REQUIRED. The same client identifier URL used in the authorization request.

`code_verifier`:
: REQUIRED. The PKCE verifier corresponding to the `code_challenge` in the authorization request.

`client_notification_token`:
: OPTIONAL. The callback authentication credential defined by {{DTR}}, with the same requirements as in {{backchannel-request}}.

The request MUST NOT contain `registration` or `audience`; both were bound at the authorization endpoint. The client authenticates according to {{client-identity}}, and when `dpop_jkt` was included in the authorization request, the client MUST send a DPoP proof for the token endpoint using the same key.

The authorization server MUST validate the software statement code and all of its bindings before processing the request. An invalid, expired, previously used, or incorrectly bound code MUST result in an `invalid_grant` error; a PKCE or DPoP binding failure is handled according to {{RFC7636}} or {{RFC9449}}, respectively.

If the issuance decision has completed, the authorization server returns the software statement token response ({{software-statement-response}}). If it remains pending, the authorization server defers the redemption with the deferred token response of {{DTR}}, binding the deferral state to the same client identifier, metadata snapshot, requested audience, and client authentication or DPoP key as the software statement code, and the client polls according to {{deferred-processing}}.

# Backchannel Software Statement Request {#backchannel-request}

A client without access to a user agent can request a software statement directly at the token endpoint. An authorization server advertises support for this flow with the `software_statement_backchannel_supported` metadata member ({{authorization-server-metadata}}); advertising the grant alone does not signal initiation support, because the same grant redeems software statement codes. A client MUST NOT send a backchannel initiation to an authorization server that does not advertise support. This grant carries no credential beyond client authentication; a client that holds a pre-authorization credential or a prior statement presents it through the token exchange profile ({{token-exchange-profile}}) instead.

Every request under this grant, whether a backchannel initiation or a software statement code redemption ({{software-statement-code-redemption}}), is deferral-willing by definition: sending it constitutes the client opt-in that {{DTR}} requires, and a client implementing this specification MUST support the polling grant of {{deferred-processing}}. The `completion_mode` parameter of {{DTR}} is not used: a client MUST NOT send it, and an authorization server MUST ignore it if present, which preserves {{DTR}}'s requirement that the parameter never cause rejection.

The client sends an HTTP `POST` request to the token endpoint using the `application/x-www-form-urlencoded` format with:

`grant_type`:
: REQUIRED. The value MUST be `urn:ietf:params:oauth:grant-type:software-statement`.

`client_id`:
: REQUIRED. The client identifier URL described in {{client-identity}}.

`registration`:
: OPTIONAL. A JSON object containing the registration overlay described in {{registration-overlay}}, subject to the same validation rules. Rule 6 does not apply, because the request uses no `redirect_uri`.

`audience`:
: OPTIONAL. The requested audience described in {{authorization-request}}. The same syntax, validation, and narrowing rules apply.

`client_notification_token`:
: OPTIONAL. The callback authentication credential defined by {{DTR}}. If the client has a `deferred_client_notification_endpoint` and intends to accept callbacks, it SHOULD include this parameter.

The request MUST NOT contain `software_statement_code`, `redirect_uri`, `code_verifier`, `state`, or `dpop_jkt`. The prohibitions of {{prohibited-parameters}} apply: the request MUST NOT contain `scope`, `resource`, or `authorization_details`.

The client authenticates according to {{client-identity}} when its metadata establishes an authentication method. A public client MUST additionally include a DPoP proof {{RFC9449}} for the token endpoint on the initiating request; the authorization server MUST bind any resulting deferral state to the proof's public key. A confidential client MAY additionally use DPoP. A DPoP proof establishes sender constraint but, by itself, does not authorize the presenter to request a statement for the `client_id` URL.

A backchannel request from a public client is unauthenticated, and a DPoP proof does not change that. Such a request MUST NOT complete synchronously: the authorization server MUST defer it and MUST NOT issue the statement until it has verified out of band that the requesting party is authorized to act for the client identifier. That verification MUST be bound to the specific request and its DPoP key. The redirect flow needs no such rule because its response is delivered to a redirection endpoint registered in the client's own metadata document, which demonstrates the requester's association with the client software.

The authorization server MUST obtain and validate the Client ID Metadata Document and any overlay exactly as in the redirect flow and MUST bind the metadata snapshot ({{metadata-snapshot}}) and the requested audience before returning either the software statement or a deferral code. Because no user agent is present, approval and any verification of the requesting party occur out of band. The authorization server MUST apply the same issuance policy to backchannel requests as to redirect flow requests.

If issuance completes immediately, the authorization server returns the software statement token response ({{software-statement-response}}). If issuance remains pending, the authorization server returns the deferred token response defined by {{DTR}}, and the client polls according to {{deferred-processing}}.

Other backchannel errors use the token error response format of Section 5.2 of {{RFC6749}}:

* A malformed request, invalid Client ID Metadata Document, or invalid overlay results in `invalid_request` with HTTP status code 400.
* An unacceptable requested audience results in `invalid_target` {{RFC8693}} with HTTP status code 400.
* Failed client authentication results in `invalid_client` with HTTP status code 401.
* A request rejected by issuance policy results in `access_denied` with HTTP status code 400.
* A server that does not support the grant at all returns `unsupported_grant_type`; one that supports it only for redemption rejects an initiation with `invalid_request`. Both use HTTP status code 400.

These errors are terminal for the submitted request. The authorization server MUST include `Cache-Control: no-store` on every backchannel error response.

The following is a non-normative example of an initiating backchannel request (line breaks are for display purposes only):

~~~ http
POST /token HTTP/1.1
Host: server.example.com
Content-Type: application/x-www-form-urlencoded
DPoP: eyJ0eXAiOiJkcG9wK2p3dCIsImFsZyI6IkVTMjU2IiwiandrIjp7Li4ufX0...

grant_type=urn%3Aietf%3Aparams%3Aoauth%3Agrant-type%3A
  software-statement
  &client_id=https%3A%2F%2Fclient.example.org%2Fmetadata.json
~~~

# Token Exchange Profile {#token-exchange-profile}

A software statement request asks the authorization server to make an issuance decision that has not yet been made. When the client already holds a token that carries the authority to issue, it MAY instead exchange that token for a software statement using OAuth 2.0 Token Exchange {{RFC8693}}. The rule of thumb is wire-visible: a client that holds a token for issuance exchanges it; a client that holds nothing makes a software statement request. An authorization server advertises support for this profile through `software_statement_subject_token_types_supported` ({{authorization-server-metadata}}).

The client sends a token exchange request as defined in Section 2.1 of {{RFC8693}} with:

`grant_type`:
: REQUIRED. The value MUST be `urn:ietf:params:oauth:grant-type:token-exchange`.

`requested_token_type`:
: REQUIRED. The value MUST be `urn:ietf:params:oauth:token-type:software-statement`.

`subject_token` and `subject_token_type`:
: REQUIRED. This profile defines two acceptable subject tokens. The first is an initial access token: an authorization credential issued out of band by this authorization server that pre-authorizes software statement issuance, analogous to the initial access token of {{RFC7591}}, presented with a `subject_token_type` of `urn:ietf:params:oauth:token-type:access_token`; the credential MUST be bound to the `client_id` of the request. The second is an unexpired software statement previously issued by this authorization server, presented with a `subject_token_type` of `urn:ietf:params:oauth:token-type:software-statement`; its `sub` MUST exactly equal the `client_id` of the request.

`client_id`:
: REQUIRED. The client identifier URL described in {{client-identity}}.

`audience`:
: OPTIONAL. The requested audience described in {{authorization-request}}. The same syntax, validation, and narrowing rules apply.

`registration`:
: OPTIONAL. A JSON object containing the registration overlay described in {{registration-overlay}}, subject to the same validation rules. Rule 6 does not apply.

`client_notification_token`:
: OPTIONAL. The callback authentication credential defined by {{DTR}}, with the same requirements as in {{backchannel-request}}.

The request MUST NOT contain `actor_token` or `actor_token_type`. The prohibitions of {{prohibited-parameters}} apply: the request MUST NOT contain `scope`, `resource`, or `authorization_details`. An exchange requesting `urn:ietf:params:oauth:token-type:software-statement` is deferral-willing by definition, constituting the client opt-in that {{DTR}} requires; the token exchange grant is within {{DTR}}'s scope, and the `completion_mode` parameter is not used: a client MUST NOT send it, and it MUST be ignored if present.

The client authenticates according to {{client-identity}}. A public client MUST include a DPoP proof {{RFC9449}} on the exchange request; the authorization server MUST bind any resulting deferral state to the proof's public key, and the polling rules of {{deferred-processing}} apply as for a backchannel deferral.

The authorization server MUST validate the subject token before retrieving client-controlled metadata or enqueueing any processing. An invalid, expired, or revoked subject token, or one that does not authorize issuance for the presented `client_id`, MUST result in `invalid_grant`. An unacceptable requested audience results in `invalid_target` {{RFC8693}}.

When the subject token is a software statement, the request MUST be holder-bound: the client MUST authenticate according to {{client-identity}}, or present a DPoP proof whose key is attested by the statement (present in its `jwks` or resolvable through its attested `jwks_uri`). A statement that attests no usable key material and belongs to a client without an authentication method MUST NOT be accepted for renewal; such a client performs a new software statement request instead.

An initial access token presented under this profile SHOULD be integrity protected, confidential in transit and at rest, limited to the issuing authorization server, time limited, and bound to an exact client identifier URL or an explicitly authorized client identifier namespace; it MAY further restrict audiences, metadata, overlays, or the number of uses. An opaque credential SHOULD contain at least 128 bits of entropy. The authorization server MUST enforce every restriction the credential carries and MUST prevent replay beyond its permitted number of uses.

The exchange is evaluated against current metadata, not the subject token's contents. The authorization server MUST obtain and validate the Client ID Metadata Document and MUST bind a fresh metadata snapshot ({{metadata-snapshot}}) and the requested audience before returning either the software statement or a deferral code. The statement's claims derive from that snapshot; the authorization server MUST NOT copy metadata claims from a subject statement. When the current canonical digest differs from a subject statement's `cimd_digest`, the metadata has changed since the prior issuance; the change is an input to issuance policy and MAY cause the exchange to be deferred or rejected. The issued statement's `aud` MUST NOT contain a value absent from a subject statement's `aud` unless issuance policy independently approves the addition.

Whether an exchange completes synchronously or defers for approval is issuance policy: a subject token can pre-authorize only the making of the request, or the issuance itself. A successful exchange returns the software statement token response ({{software-statement-response}}), which already carries the {{RFC8693}} response members. If processing cannot complete immediately, the authorization server returns the deferred token response of {{DTR}} and the client polls according to {{deferred-processing}}.

# Deferred Processing {#deferred-processing}

Every deferral originates from a deferred token response of {{DTR}}, issued for a software statement code redemption ({{software-statement-code-redemption}}), a backchannel request ({{backchannel-request}}), or a token exchange ({{token-exchange-profile}}). The client polls the token endpoint using the polling grant defined by {{DTR}}.

Approval of a software statement request can take hours or days rather than the seconds typical of user authentication, for example when it involves reviewing the client's policy or compliance documentation. Issuers SHOULD set deferral code lifetimes that reflect their actual approval latency. A client with a registered notification endpoint SHOULD use the callback mechanism of {{DTR}} together with `client_notification_token` on the originating token request, while continuing to poll as required by {{DTR}}.

## First Polling Request {#first-polling-request}

The first polling request is an HTTP `POST` request to the token endpoint using the `application/x-www-form-urlencoded` format. It contains:

`grant_type`:
: REQUIRED. The value MUST be `urn:ietf:params:oauth:grant-type:deferred`.

`deferral_code`:
: REQUIRED. The deferral code returned in the deferred response.

The polling request carries no PKCE parameter: for a redirect-flow deferral, the verifier was consumed when the software statement code was redeemed ({{software-statement-code-redemption}}). On every polling request the authorization server MUST verify that the request presents the client authentication or DPoP key bound to the deferral at its origination.

The client authenticates according to {{client-identity}}. If the deferral is bound to a DPoP key, through `dpop_jkt` in the redirect flow or the initiating DPoP proof otherwise, the client MUST send a DPoP proof signed with the corresponding key. The authorization server MUST reject a proof whose key does not match the stored thumbprint.

The first polling request MUST NOT contain a `software_statement_code`, `code_verifier`, `redirect_uri`, `registration`, `audience`, `client_notification_token`, `subject_token`, `subject_token_type`, or `requested_token_type` parameter.

## Subsequent Polling Requests

If the request remains pending, the client continues polling according to {{DTR}} using only the polling grant's normal parameters.

Each subsequent polling request MUST use the client authentication or DPoP sender constraint established for the deferral. In particular, a public client MUST send a DPoP proof on every poll, using the key identified by `dpop_jkt` in the redirect flow or the key of the initiating DPoP proof otherwise.

Pending, denied, expired, cancelled, and polling-rate behavior follows {{DTR}}. Callback behavior follows {{DTR}} only for a deferral whose initiating request supplied `client_notification_token`. A successful first or subsequent polling response is the software statement token response defined in {{software-statement-response}}.

Cancellation of a deferral follows the revocation mechanism of {{DTR}}. For a deferral created under this specification, the authorization server MUST require the deferral's sender constraint on the revocation request: client authentication where the deferral is bound to client authentication, or a DPoP proof with the initiation-bound key otherwise. A revocation request that does not present the bound constraint MUST be treated as presenting an unrecognized token per {{DTR}}.

# Software Statement Token Response {#software-statement-response}

A successful response has HTTP status code 200, a media type of `application/json`, and the following members:

`access_token`:
: REQUIRED. The software statement issued by the authorization server. As in {{RFC8693}}, the historically named `access_token` member carries the issued security token; its presence does not make the software statement an OAuth access token.

`issued_token_type`:
: REQUIRED. The value MUST be `urn:ietf:params:oauth:token-type:software-statement`.

`token_type`:
: REQUIRED. The value MUST be `N_A`, indicating that an OAuth access token type does not apply.

`expires_in`:
: RECOMMENDED. The remaining lifetime of the software statement in seconds. If present, it MUST be consistent with the statement's `exp` claim.

The response MUST NOT contain `refresh_token` or `scope`. The authorization server MUST include `Cache-Control: no-store`; it SHOULD also include `Pragma: no-cache`.

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

To use the issued statement for dynamic client registration, the client supplies the value of `access_token` as the `software_statement` member of an {{RFC7591}} client registration request.

A client obtains a replacement for an expiring or expired software statement by performing a new software statement request or, while the statement is unexpired, by exchanging it under {{token-exchange-profile}}. Whether replacement requires new approval is determined by issuer policy.

# Software Statement Format {#software-statement-format}

The software statement is a compact JWT {{RFC7519}} protected by a digital signature using JWS {{RFC7515}}. Although {{RFC7591}} also permits a MAC, a statement issued under this specification MUST use an asymmetric digital signature so that it can be validated without distributing an issuer-held symmetric key. The issuer and trusting authorization server MUST follow the algorithm verification guidance in {{RFC8725}}. The `none` algorithm and symmetric algorithms MUST NOT be used.

The JOSE header MUST include a `typ` (type) header parameter with the value `software-statement+jwt`, applying the explicit typing recommendation in Section 3.11 of {{RFC8725}}. As described in Section 4.1.9 of {{RFC7515}}, this value names the media type `application/software-statement+jwt` ({{media-type}}) with the `application/` prefix omitted. Explicit typing prevents other JWTs signed by the same issuer, such as JWT access tokens or ID Tokens, from being accepted as software statements.

The JWT payload MUST contain the following claims in addition to the approved client metadata that is eligible for attestation under {{registration-overlay}}. It MUST NOT contain any metadata prohibited by that section.

`iss`:
: REQUIRED. The issuer identifier of the issuing authorization server, as defined by {{RFC8414}}.

`sub`:
: REQUIRED. The exact client identifier URL from the software statement request.

`aud`:
: REQUIRED. One or more audience identifiers for the authorization servers permitted to accept the statement. Each value SHOULD be an authorization server issuer identifier as defined by {{RFC8414}}. A trusting authorization server MUST reject the statement unless one of its locally configured audience identifiers exactly matches a value in this claim. When the request contained `audience` values, every value in this claim MUST have appeared among them.

`iat`:
: REQUIRED. A NumericDate value representing the time at which the software statement was issued.

`exp`:
: REQUIRED. A NumericDate value representing the expiration time. A trusting authorization server MUST reject an expired statement.

`jti`:
: REQUIRED. A unique identifier for the statement within the issuer's namespace.

`cimd_digest`:
: REQUIRED. The canonical digest ({{metadata-snapshot}}) of the Client ID Metadata Document from which the metadata snapshot was derived. This claim binds the statement to the exact document content evaluated during issuance and lets any party determine whether the client's currently published metadata still matches what was attested.

The issuer determines the audience and lifetime according to policy; neither is supplied in the `registration` overlay. A client can request an audience using the `audience` parameter, but the issuer MAY narrow that request and MUST NOT widen it. If the parameter is omitted, the issuer selects an audience entirely according to policy. The client metadata claims MUST reflect the metadata snapshot and any further narrowing performed by the issuing authorization server. They MUST NOT contradict or widen the snapshot.

The `sub` claim identifies the client software by its client identifier URL; it does not name the `client_id` that a trusting authorization server will assign. Registration with the software statement proceeds under {{RFC7591}} and yields an authorization-server-specific `client_id` that need not be related to `sub`. A trusting authorization server MAY use `sub` to correlate registrations of the same client software and to apply per-client policy, such as bounding the number of active registrations ({{multi-instance}}). When the metadata snapshot includes `software_id`, that value identifies the software as defined by {{RFC7591}}; this specification does not derive either identifier from the other.

The following is a non-normative example of the JOSE header and JWT payload of a software statement:

~~~
{
  "typ": "software-statement+jwt",
  "alg": "ES256",
  "kid": "issuer-key-1"
}
.
{
  "iss": "https://server.example.com",
  "sub": "https://client.example.org/metadata.json",
  "aud": ["https://server.example.net"],
  "iat": 1767225600,
  "exp": 1767229200,
  "jti": "0b1f8a3e-6d2c-4c58-9a0f-6b2f6c3d9e21",
  "cimd_digest": "3W6cWfLXi0mZbUIhk8N4Zt2v9Qq7oT1xJdKe5RgYs0A",
  "client_name": "Example App",
  "redirect_uris": ["https://client.example.org/cb"],
  "grant_types": ["authorization_code"],
  "scope": "read",
  "token_endpoint_auth_method": "private_key_jwt",
  "jwks_uri": "https://client.example.org/jwks.json"
}
~~~

Before accepting the statement, a trusting authorization server MUST verify that the `typ` header carries the value `software-statement+jwt`; verify the signature under a key trusted for the exact `iss` value; validate the presence and type of every required claim; validate `iss`, `aud`, `iat`, and `exp`; verify that `sub` is a Client Identifier URL conforming to {{CIMD}}; reject every metadata claim prohibited by {{registration-overlay}}; and apply the JWT validation guidance in {{RFC8725}}. In particular, it MUST reject a statement whose `iat` is unreasonably far in the future according to its clock-skew policy. Publishing an issuer URL or a JWK Set does not by itself establish trust: signature keys for a trusted issuer are conventionally obtained from the `jwks_uri` in the issuer's authorization server metadata {{RFC8414}}, but the decision to trust the issuer MUST come from explicit local configuration. A trusting authorization server MAY retrieve the client's current metadata document and compare canonical digests against `cimd_digest` to learn whether the attested content is still published; if it does so, it MUST also verify that the document's `client_id` is exactly equal to `sub` as required by {{CIMD}}. A digest mismatch indicates post-issuance change and is an input to registration policy rather than a validation failure. The trusting authorization server then processes the software statement according to {{RFC7591}} and its local registration policy.

# Multi-Instance Client Software {#multi-instance}

A software statement attests client software, identified by `sub`; it does not attest or identify the runtime instances of that software. This specification defines no instance identifier, and instances do not obtain per-instance statements.

A single unexpired statement is therefore intended to be presented more than once: at each trusting authorization server in its audience and, where local policy permits, in more than one registration at the same authorization server, for example one registration per deployment or tenant. A trusting authorization server SHOULD use the statement's `sub` and `jti` to inventory the registrations derived from a statement and to enforce any local bound on their number.

Deployments whose client software runs as many concurrent instances SHOULD register the logical client once per authorization server and differentiate instances at the token endpoint, for example with {{CLIENT-INSTANCE}}, rather than minting a registration per instance. Where authorization server policy is keyed on `client_id` and genuinely requires per-instance registrations, the same statement can support that model only if it omits `jwks` and `jwks_uri`: each registration can then supply its own instance key as plain metadata, subject to trusting authorization server policy. If the statement contains `jwks` or `jwks_uri`, that attested value takes precedence under {{RFC7591}} and every registration derived from the statement MUST use the attested key material rather than an instance-supplied replacement.

## Attesting Instance Issuers {#attesting-instance-issuers}

{{CLIENT-INSTANCE}} defines the `instance_issuers` client metadata parameter, through which a client delegates attestation of its runtime instances to named authorities. For the purposes of this specification, `instance_issuers` is client metadata like any other: it can appear in the canonical Client ID Metadata Document, be selected by a registration overlay, and be carried as an attested claim in the software statement.

When `instance_issuers` appears in an overlay, rule 1 of {{registration-overlay}} applies at the granularity of whole descriptors: the overlay selects complete descriptor objects that occur in the canonical array. A descriptor with any altered member does not occur in the canonical array and causes rejection under rule 5.

A statement whose claims include `instance_issuers` gives the trusting authorization server a third-party attestation of the instance-attestation delegation itself rather than a self-asserted list. This is particularly useful where the trusting authorization server does not dereference client metadata documents: the registration carries an attested issuer list with no dependence on document availability or on locally configured, unattested lists. An issuing authorization server SHOULD include `instance_issuers` in a statement only when it recognizes the member and its approval process covered the delegation the member expresses.

For example, the following claims fragment attests one instance issuer for the client:

~~~
"instance_issuers": [
  {
    "issuer": "https://workload.client.example.org",
    "jwks_uri": "https://workload.client.example.org/jwks.json"
  }
]
~~~

# Authorization Server Metadata {#authorization-server-metadata}

The issuing authorization server is a role, named the way "trusting authorization server" is: this specification does not require it to be a general-purpose OAuth deployment. A conforming issuer that offers only the backchannel flow and the token exchange profile consists of a token endpoint, an authorization server metadata document, and signing keys. It operates no authorization endpoint, never issues access tokens, and serves no protected resources; a deployment can operate such an issuer purely as an attestation service.

An authorization server that issues software statements under this specification advertises `true` for `client_id_metadata_document_supported`, as defined by {{CIMD}}, and publishes `software_statement_signing_alg_values_supported` below.

Every issuer advertises `urn:ietf:params:oauth:grant-type:software-statement` in `grant_types_supported` and `true` for `deferred_token_response_supported`, as defined by {{DTR}}: the grant serves both software statement code redemption and backchannel initiation, and deferred processing is integral to every flow. An authorization server that accepts backchannel initiations additionally advertises `true` for `software_statement_backchannel_supported` below.

An authorization server supporting the redirect flow additionally advertises:

* `software_statement_code` in `response_types_supported`; and
* `true` for `authorization_response_iss_parameter_supported`, as defined by {{RFC9207}}.

A backchannel-only implementation does not advertise `software_statement_code` in `response_types_supported` and need not advertise `authorization_response_iss_parameter_supported`.

An authorization server supporting the token exchange profile ({{token-exchange-profile}}) advertises `urn:ietf:params:oauth:grant-type:token-exchange` in `grant_types_supported` and publishes `software_statement_subject_token_types_supported` below; general token exchange support does not by itself imply support for this profile.

A client MUST NOT send a software statement request to an authorization server that does not advertise `deferred_token_response_supported` as `true`.

This specification defines the following additional authorization server metadata members:

`software_statement_backchannel_supported`:
: OPTIONAL. Boolean value indicating whether the authorization server accepts backchannel initiations of the software statement grant ({{backchannel-request}}). If omitted, the default value is `false`.

`software_statement_signing_alg_values_supported`:
: REQUIRED for an authorization server that issues software statements under this specification. A JSON array containing the asymmetric JWS `alg` values that the authorization server can use to sign software statements. The array MUST NOT contain `none` or a symmetric algorithm. This member describes the issuing role; an authorization server that only accepts software statements does not publish it.

`software_statement_audiences_supported`:
: OPTIONAL. A JSON array containing authorization server issuer identifiers that the issuer is prepared to place in a software statement's `aud` claim. Omission means that the complete set is not publicly enumerable; it does not mean that no audience is supported. Publication does not guarantee that every client is eligible for every listed audience.

`software_statement_subject_token_types_supported`:
: REQUIRED for an authorization server that supports the token exchange profile ({{token-exchange-profile}}), and absent otherwise. A JSON array of the `subject_token_type` values the authorization server accepts when `requested_token_type` is `urn:ietf:params:oauth:token-type:software-statement`. Publication of this member is the discovery signal for the profile.

# Security Considerations {#security-considerations}

## Client Establishment Is Not an Access Grant

A software statement represents an issuer's attestation to client metadata. It grants no access to a protected resource. In particular, approval of issuance authorizes no user's access: each user's access through a subsequently registered client still requires that user's own authorization at the trusting authorization server. Approving issuance for client software is therefore never a consent decision made on behalf of that software's users. Authorization servers MUST enforce the prohibited-parameter and response-type rules in {{prohibited-parameters}}. The successful token response uses the `access_token` member only as a generic security-token container following {{RFC8693}}; the contained software statement MUST NOT be accepted as an access token at a protected resource.

When an approval interface is shown, it MUST clearly describe that the decision concerns attestation to client metadata. It MUST NOT imply that the approver is granting the client access to resources.

An erroneous approval is amplified by the statement's audience: every authorization server in the audience will accept the resulting credential until it expires. The approval interface therefore MUST present the client identifier URL, the effective metadata being attested, and the audience the issuer intends to place in the statement; when the client requested a different or broader audience, the interface SHOULD also make the narrowing visible. It SHOULD present the intended lifetime of the statement. Metadata that itself delegates authority deserves particular scrutiny: attesting the `instance_issuers` member ({{multi-instance}}) endorses the listed authorities to attest runtime instances of the client.

## What Issuance Attests {#what-issuance-attests}

A software statement means one thing: the issuer evaluated the exact document content captured in the metadata snapshot ({{metadata-snapshot}}), under its issuance policy, at the time recorded in `iat`, and decided to attest the contained metadata for the named audience.

It does not mean that every attested value has been independently verified. The canonical metadata document is authored by the client, and issuance does not convert self-assertion into fact. What issuance changes is who judges the assertion and how that judgment is recorded: one accountable evaluation of a deterministic, digest-bound input ({{metadata-snapshot}}, {{registration-overlay}}) replaces an independent ad-hoc evaluation of raw self-asserted values at every trusting authorization server. An issuer SHOULD corroborate security-relevant metadata through evidence beyond the document itself; the depth of verification an issuer performs before signing is a core element of the trust relationship a trusting authorization server accepts when it configures that issuer.

A trusting authorization server is accordingly entitled to conclude what the issuer decided, and no more. Its own registration policy determines what that decision is worth.

## Client Metadata Retrieval

Fetching a Client ID Metadata Document and resources referenced by it exposes the authorization server to server-side request forgery, resource exhaustion, malicious content, and client impersonation risks. The validation, address filtering, response-size limits, redirect handling, caching, logo handling, and domain-trust considerations of {{CIMD}} apply.

The metadata snapshot requirements in {{metadata-snapshot}} prevent a time-of-check/time-of-use change from silently altering the metadata after approval. Authorization servers SHOULD record the canonical digest ({{metadata-snapshot}}) or an immutable copy of the approved document and snapshot for audit purposes.

## Overlay Validation

An overlay that adds or widens metadata would let the request obtain an attestation for values not present in its canonical document. Authorization servers MUST perform the member-by-member validation in {{registration-overlay}}. When seeking approval, they MUST present the effective metadata, not merely the canonical document, to the approver.

Because the overlay can contain client metadata that should not traverse the browser, requests containing `registration` are required to use PAR. The confidentiality and integrity considerations of {{RFC9126}} apply.

## Authorization Response Security

The software statement is a signed credential and can contain sensitive deployment information. It MUST NOT be returned through the authorization endpoint. The software statement code response applies exact redirect URI matching, PKCE with `S256`, and the other applicable authorization response protections in {{RFC9700}}.

Before redeeming a software statement code, the client MUST verify `state` and MUST validate the authorization response `iss` parameter according to {{RFC9207}}. Although PKCE with `S256` already provides the cross-site request forgery protection described in {{RFC9700}}, this specification requires `state` so that a client running concurrent software statement requests can correlate each authorization response with its originating request before redemption.

A software statement code returned in a URL is visible to browser history, referrer fields, redirection-target logs, and other front-channel observers, like an authorization code, and is protected the same way: it is short lived, single use, and unredeemable without the PKCE verifier and, for a public client, the `dpop_jkt` key. Deferral codes never transit the front channel under this specification; they are issued only in deferred token responses over the direct TLS connection, and their theft is contained by sender-constrained polling and cancellation ({{deferred-processing}}). Deployments sensitive to URL disclosure of the software statement code can use the `form_post` response mode {{FORM-POST}} as described in {{software-statement-code-response}}; the fragment response mode SHOULD NOT be used.

## Deferral Sender Constraint

All software statement requests made through the authorization endpoint use PKCE, verified when the software statement code is redeemed. Every deferral in this specification is created by a token request, and every polling request is protected by the client authentication or DPoP key bound at that origination, as required by {{DTR}}. A public client using the redirect flow is therefore required to supply `dpop_jkt` in the authorization request and use that key at redemption and on every poll; this preserves sender-constraint continuity from the authorization response through the polling sequence.

All additional sender-constraint, polling-rate, replay, cancellation, and logging requirements of {{DTR}} apply. Callback behavior follows {{DTR}} for any deferral whose originating token request supplied `client_notification_token`.

## Backchannel and Token Exchange Considerations

A backchannel request reaches the token endpoint without prior user-agent interaction. An unauthenticated request from a public client can therefore trigger metadata document retrieval and enqueue approval work at little cost to the requester. Authorization servers SHOULD rate-limit backchannel requests and SHOULD cache metadata retrieval results and failures. An unauthenticated public-client request can never complete synchronously ({{backchannel-request}}): issuance to an anonymous requester would deliver a bearer attestation for someone else's software identity, so the out-of-band verification of the requester's authority is the gate that replaces the redirect flow's delivery to a registered redirection endpoint. An authorization server that requires pre-authorization for token endpoint issuance supports only the token exchange profile ({{token-exchange-profile}}), which places subject token validation in front of any processing. Client authentication remains mandatory when established by the Client ID Metadata Document.

A pre-authorization credential presented as a subject token ({{token-exchange-profile}}) is an authorization credential, not a client identifier and not a substitute for client authentication when the Client ID Metadata Document establishes an authentication method. It is sent in a form body and is therefore exposed to any component that records request bodies. Authorization servers MUST exclude subject tokens from logs, traces, error messages, and audit records; clients and authorization servers MUST protect the credential as a bearer credential unless its format provides proof of possession. The binding, lifetime, entropy, and replay requirements of {{token-exchange-profile}} limit the effect of disclosure.

The absence of a user agent removes front-channel exposure: neither the deferral code nor any other response parameter transits a browser. It also removes any in-band evidence of user participation. An authorization server MUST NOT treat a backchannel request or token exchange as implying prior user consent and MUST apply the same issuance and approval policy as for the redirect flow.

Sender constraint for every deferral is established at its originating token request through client authentication or a DPoP proof; polling carries no PKCE state, the redirect flow's `code_verifier` having been consumed at redemption ({{software-statement-code-redemption}}). If a callback is configured, the client uses `client_notification_token` as defined by {{DTR}}; that token authenticates the callback but does not authorize issuance.

## Statement Validation and Replay

A software statement is intended to be presented more than once ({{multi-instance}}); until it expires, it can equally be presented by a party that steals it, at every registration endpoint included in its audience. Issuers SHOULD use the narrowest practical audience and lifetime. Trusting authorization servers SHOULD track `jti` values to inventory the registrations derived from a statement and to enforce any local bound on their number.

When registrations derived from a statement are intended to share a client key, and the canonical metadata provides `jwks` or `jwks_uri`, the issuer SHOULD include that member in the attested metadata. A trusting authorization server can then require proof of possession of a corresponding private key during or after registration, which renders a stolen statement unusable to a party that does not also hold the client's key. When each registration is expected to supply a distinct instance key, the issuer MUST omit `jwks` and `jwks_uri` so that the plain registration metadata can carry that key without conflicting with the precedence rule of {{RFC7591}}.

Renewal through {{token-exchange-profile}} would otherwise let an unexpired stolen statement be exchanged for a fresh one, outliving its intended lifetime. That profile therefore requires holder binding for statement subjects, and a statement without usable key binding cannot be renewed at all; the issuer-scoping guidance of this section applies to the exchange as it does to registration.

This specification does not define online revocation of an issued software statement. A deployment that cannot tolerate acceptance of a stolen or mistakenly issued statement for its full lifetime needs short statement lifetimes, a deployment-specific mechanism through which trusting authorization servers obtain revocation status, or both.

Trusting authorization servers MUST use an explicit issuer trust configuration and MUST verify the signature and claims described in {{software-statement-format}}. They MUST NOT derive trust solely from attacker-controlled `iss`, `jku`, `x5u`, or other key-location values in the JWT.

Issuer trust SHOULD be scoped as well as explicit. A trusting authorization server that accepts an issuer for all values of `sub` allows that issuer, if compromised or over-broad, to mint acceptable statements about any client software. Trust configuration SHOULD therefore constrain each issuer to the client identifier namespaces it is expected to attest, for example to URLs under the domains of the software publishers it serves, and statements whose `sub` falls outside that scope SHOULD be rejected. {{TRUST-FRAMEWORK}} generalizes this namespace-authorization check into an open-world policy evaluation.

## Signing Keys and Algorithms

Compromise of a software-statement signing key enables an attacker to mint statements for every audience that trusts that key. Issuers MUST protect signing keys according to the scope of their trust relationships and SHOULD support controlled key rotation. Trusting authorization servers MUST restrict algorithms to those allowed for the issuer and MUST follow {{RFC8725}} when selecting keys and validating JWTs.

## Approver Identity and Audit

The software statement attests to metadata; it does not identify the human or system that approved issuance. Deployments that require approver attribution MUST retain it in an authorization server audit record or define an explicit statement claim and its privacy semantics. They MUST NOT infer approver identity from the signature alone.

Approval authority is a policy decision with audience-wide effect: an approved statement is accepted at every authorization server in its audience, not only within the approver's own scope. The policy governing who may approve issuance MUST be at least as restrictive as the policy governing manual client establishment at the issuing authorization server, and approval by a party authorized only for a personal or organizational scope MUST NOT produce a statement whose audience exceeds that scope.

An audit record SHOULD bind each decision, whether approval or denial, to the canonical digest ({{metadata-snapshot}}) of the document the deciding party evaluated, the policy under which the decision was made, the identity of that party, and the time of decision. A recorded denial carrying its grounds has the same audit value as a recorded approval. Portable, independently verifiable decision records are out of scope for this specification.

# Privacy Considerations

The authorization server learns the client identifier URL, the canonical metadata document, the selected overlay, and information about the party interacting with the authorization endpoint. It SHOULD collect and retain only the information required for issuance, security monitoring, and audit obligations.

A software statement distributes attested client metadata to every authorization server at which the client presents it. Issuers SHOULD omit metadata that is not required by the intended audience and SHOULD choose the narrowest practical audience. Clients SHOULD NOT present statements outside their intended deployment context. Requested `audience` values reveal the authorization servers with which the client plans to establish relationships. A redirect-flow client SHOULD use PAR when that relationship is sensitive, even when no `registration` overlay is present.

The `registration` parameter can reveal request-specific metadata. Requiring PAR prevents that object from appearing directly in the browser-visible authorization request URL, but it does not make the data invisible to the authorization server or its operational logs. Authorization servers SHOULD avoid logging full overlays and issued statements.

Approval records can link a person to a client and deployment. Such records SHOULD be access-controlled and retained only as long as required.

# IANA Considerations {#iana}

## OAuth Authorization Endpoint Response Types Registry

This specification requests registration of the following value in the IANA "OAuth Authorization Endpoint Response Types" registry established by {{RFC6749}}:

Response Type Name:
: `software_statement_code`

Change Controller:
: IETF

Specification Document(s):
: This specification, {{authorization-request}} and {{authorization-response}}

## OAuth URI Registry

This specification requests registration of the following values in the IANA "OAuth URI" registry:

URN:
: `urn:ietf:params:oauth:grant-type:software-statement`

Common Name:
: OAuth Software Statement Grant Type

Change Controller:
: IETF

Specification Document(s):
: This specification, {{backchannel-request}}

URN:
: `urn:ietf:params:oauth:token-type:software-statement`

Common Name:
: OAuth Software Statement Token Type

Change Controller:
: IETF

Specification Document(s):
: This specification, {{software-statement-response}} and {{token-exchange-profile}}

## OAuth Parameters Registry

This specification requests that IANA add this specification, {{registration-overlay}}, {{backchannel-request}}, and {{token-exchange-profile}}, as an additional reference for the existing `registration` parameter and extend its usage location to include token requests. The parameter name and change controller are unchanged.

This specification further requests that IANA add this specification, {{authorization-request}}, {{backchannel-request}}, and {{token-exchange-profile}}, as an additional reference for the existing `audience` parameter registered by {{RFC8693}} and extend its usage location to include authorization requests. The parameter name and change controller are unchanged.

This specification also requests registration of the following value in the IANA "OAuth Parameters" registry established by {{RFC6749}}:

Parameter Name:
: `software_statement_code`

Parameter Usage Location:
: authorization response, token request

Change Controller:
: IETF

Specification Document(s):
: This specification, {{software-statement-code-response}} and {{software-statement-code-redemption}}

## OAuth Authorization Server Metadata Registry

This specification requests registration of the following values in the IANA "OAuth Authorization Server Metadata" registry established by {{RFC8414}}:

Metadata Name:
: `software_statement_backchannel_supported`

Metadata Description:
: Boolean value indicating whether the authorization server accepts backchannel initiations of the software statement grant.

Change Controller:
: IESG

Specification Document(s):
: This specification, {{backchannel-request}}

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
: binary. A software statement is a JWT; JWT values are encoded as a series of base64url-encoded values separated by period ('.') characters.

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
: Unpadded base64url-encoded SHA-256 digest of the JCS-serialized Client ID Metadata Document evaluated during software statement issuance

Change Controller:
: IETF

Specification Document(s):
: This specification, {{software-statement-format}}

--- back

# Deployment Examples {#deployment-examples}

The scenarios in this appendix are non-normative. The first two connect the flows of this specification end to end and reference the normative sections that govern each step; the third marks the boundary where this specification deliberately does not apply.

## Publisher Program: One Review, Many Customer Authorization Servers {#example-publisher}

A developer builds TaskFlow, a workflow client used by customers of a marketplace. TaskFlow publishes its Client ID Metadata Document at `https://taskflow.example/oauth/metadata.json`; the document contains its production redirect URI and establishes `private_key_jwt` client authentication using a published `jwks_uri`. The marketplace operates a publisher program whose issuing authorization server is `https://issuer.marketplace.example`. Two participating customers operate authorization servers with issuer identifiers `https://as.customer-a.example` and `https://as.customer-b.example`. Each customer configures the publisher program as a trusted statement issuer, scoped to TaskFlow's client identifier namespace.

1. TaskFlow's publisher tooling submits an authorization request through PAR ({{RFC9126}}). The request uses `response_type=software_statement_code`, requests both customer issuer identifiers by repeating `audience`. Its `registration` overlay selects the production redirect URI and only the metadata needed for the reviewed deployment. The tooling then opens the resulting authorization request in the developer's user agent ({{authorization-request}}).
2. The developer authenticates to the publisher program, which records the developer's identity in its audit trail. The authorization endpoint returns the software statement code response ({{software-statement-code-response}}). The tooling validates `state` and `iss` and redeems the software statement code with its PKCE verifier, authenticating with `private_key_jwt` ({{software-statement-code-redemption}}); because the compliance review takes two days, the redemption is deferred, and the tooling polls with the same authentication ({{deferred-processing}}).
3. On approval, polling returns the software statement token response. The statement's `sub` is the TaskFlow metadata URL, its `aud` array contains the two requested authorization server issuer identifiers, and its `cimd_digest` binds the review to the exact metadata document evaluated.
4. TaskFlow presents the statement in an {{RFC7591}} registration request at each customer authorization server. Each server verifies the `typ` header, signature, issuer, audience, lifetime, and other claims ({{software-statement-format}}), then assigns its own local `client_id`. Because the statement attests TaskFlow's `jwks_uri`, each registration uses the same attested client key material ({{multi-instance}}). One review is thereby honored at both authorization servers without weakening audience validation.
5. Before the statement expires, TaskFlow's release pipeline renews it with an authenticated token exchange ({{token-exchange-profile}}), presenting the unexpired statement as the subject token and requesting the same audiences. Line breaks and indentation in the form body are for display only:

~~~ http
POST /token HTTP/1.1
Host: issuer.marketplace.example
Content-Type: application/x-www-form-urlencoded

grant_type=urn%3Aietf%3Aparams%3Aoauth%3Agrant-type%3A
  token-exchange
  &requested_token_type=urn%3Aietf%3Aparams%3Aoauth%3A
  token-type%3Asoftware-statement
  &subject_token=eyJ0eXAiOiJzb2Z0d2FyZS1zdGF0ZW1lbnQrand0...
  &subject_token_type=urn%3Aietf%3Aparams%3Aoauth%3A
  token-type%3Asoftware-statement
  &client_id=https%3A%2F%2Ftaskflow.example%2Foauth%2F
  metadata.json
  &audience=https%3A%2F%2Fas.customer-a.example
  &audience=https%3A%2F%2Fas.customer-b.example
  &client_assertion_type=urn%3Aietf%3Aparams%3Aoauth%3A
  client-assertion-type%3Ajwt-bearer
  &client_assertion=eyJhbGciOiJFUzI1NiIsImtpZCI6InRhc2tmbG93LTEifQ...
~~~

Under the marketplace's renewal policy, the matching canonical digest and unchanged audience allow the exchange to complete synchronously without a new review. The policy decision, rather than the digest match alone, determines that result.

## Enterprise Deployment of a Third-Party Agent {#example-enterprise}

A vendor, ACME, ships an AI agent whose installations all share the client identifier URL `https://acme.example/agent`; the document at that URL describes the logical client software, so no per-installation metadata document exists. An enterprise deploys the agent for its workforce. Internal services each front a small authorization server, none staffed to review client software. The enterprise's identity platform, `https://idp.enterprise.example`, acts as the issuing authorization server in the attestation-service role of {{authorization-server-metadata}}: a token endpoint, metadata, and signing keys. The internal authorization servers `https://tools-as.enterprise.example` and `https://data-as.enterprise.example` each configure it as a trusted issuer for the ACME agent.

1. The agent is a public client. The enterprise's platform tooling, a daemon with no user agent, sends a backchannel request ({{backchannel-request}}) with `client_id=https://acme.example/agent`, a `registration` overlay selecting a subset of ACME's canonical metadata, both internal authorization server issuer identifiers as repeated `audience` parameters. It includes a DPoP proof and retains the private key for every polling request. The proof sender-constrains any resulting deferral code; it does not by itself authorize issuance for ACME's client identifier.
2. The identity platform accepts the request for review and returns a DTR deferred response with a three-day lifetime. The tooling waits for the advertised interval, then polls with the deferred grant, the deferral code, and a fresh DPoP proof signed by the same key. Polling carries no PKCE verifier ({{first-polling-request}}). The security team reviews the vendor assessment out of band; responses to pending polls repeat neither the deferral code nor the interval.
3. On approval, the final poll returns the software statement token response. The tooling registers the agent once at each internal authorization server, supplying the statement together with plain registration metadata naming the enterprise's own instance issuer ({{multi-instance}}). At runtime, each agent instance proves itself at the token endpoints through assertions from that instance issuer ({{CLIENT-INSTANCE}}), so every installation shares one registration per server while tokens carry per-instance identity.

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

Because the statement does not contain `instance_issuers`, the locally supplied value does not conflict with the precedence rule of {{RFC7591}}; it is not covered by the issuer's attestation, and accepting it is the registering server's policy.

Finally, before the statement expires, the tooling submits a renewal exchange with the statement as subject token and the same two audiences. When ACME has shipped an update that changes the canonical metadata document, the identity platform computes a current canonical digest that no longer matches the subject statement's `cimd_digest` ({{token-exchange-profile}}). Its issuance policy therefore defers the exchange for a fresh security review instead of copying claims from the old statement or renewing it silently.

## The Resource Owner as Approver: A Case This Specification Does Not Serve {#example-resource-owner}

Consider the same ACME agent in a different setting: an individual user runs an installation against the user's own authorization server, and the goal is to bind that installation to that user. The approving party is the resource owner. This is the case the applicability guidance in the Introduction deliberately excludes, and it is instructive to see why no software statement appears in its solution.

The user's approval is the OAuth authorization grant itself: when the agent initiates an authorization code flow, the resource owner authenticates and consents, which is precisely the decision that a registration-time or issuance-time approval ceremony would duplicate. In a deployment that accepts Client ID Metadata Document identifiers without pre-registration, the agent can be identified directly by its generic client identifier URL {{CIMD}}. The per-installation binding lives in the tokens rather than in establishment state: the instance proves itself at the token endpoint through an instance assertion ({{CLIENT-INSTANCE}}), and the issued tokens carry the user as subject and the instance as actor, sender-constrained to a key that installation holds.

A software statement would add nothing here. There is no second authorization server that needs to honor the decision, no approver other than the user already present in the transaction, and no review whose cost needs amortizing. The statement earns its place only when those conditions invert, as in {{example-enterprise}}, where the same agent software is approved once, by a party other than its users, for use across many authorization servers whose operators never review clients.

# Design Rationale

## Comparison with Related Establishment Mechanisms {#comparison}

Several contemporaneous mechanisms address client establishment. The table summarizes where each makes its trust decision, what input that decision evaluates, and what establishing trust with M authorization servers costs.

| Mechanism | Decision made at | Input evaluated | Cost for M authorization servers |
| --- | --- | --- | --- |
| Pre-registration of a client identifier URL ({{CIMD}}) | each authorization server | self-asserted metadata document | M decisions |
| Pushed client registration ({{PUSHED-DCR}}) | each authorization server | self-asserted pushed metadata, per transaction | M decisions |
| Approval-based registration ({{APPROVAL-DCR}}) | each authorization server | self-asserted registration request | M approvals |
| This specification | the issuing authorization server | canonical metadata document and overlay | one issuance decision, M policy evaluations of an attested artifact |
| OpenID Federation ({{OPENID-FED}}) | resolved along a trust chain | entity statements | federation infrastructure |

The bilateral mechanisms are not competitors; each composes with a software statement. A pre-registration or pushed registration that carries a statement presents attested metadata instead of self-asserted values. An approval-based registration that receives a statement gives its approver an issuer's signed decision to evaluate, addressing the long-standing problem that a registration approver otherwise has only the client's word. Where a deployment outgrows pairwise issuer configuration, OpenID Federation provides the trust-chain infrastructure this specification deliberately omits.

OpenID for Verifiable Credential Issuance {{OID4VCI}} shares this specification's issuance shape: an authorization code flow, a pre-authorized path for decisions made in advance, and deferred issuance for decisions that take time. It is nevertheless not an alternative rail for this artifact. {{OID4VCI}} issues credentials to a wallet, ordinarily bound to a key the wallet holds, for later presentation to verifiers; a software statement is deliberately software-level rather than holder-bound, covers every instance of the software, and is consumed not by presentation but by the `software_statement` member of an {{RFC7591}} registration request. A profile of {{OID4VCI}} that issued software statements would still have to define what this specification defines, namely the statement format, the canonical-metadata binding, and the registration-endpoint consumption, while adding credential issuer metadata, proof and nonce handling, and wallet machinery that a registration workflow does not use. The two therefore compose rather than compete: an ecosystem that already operates {{OID4VCI}} infrastructure can carry a software statement as a credential payload, and the payload's meaning at a registration endpoint remains as this specification defines it.

## Why Use the Authorization Endpoint

Issuance commonly requires browser-mediated interaction with a user or administrator. The authorization endpoint supplies established redirect URI, request correlation, PKCE, and issuer-identification mechanisms for that interaction. Returning a short-lived, sender-constrained software statement code keeps the software statement itself out of the browser and delivers it over a direct TLS-protected connection. When no in-band interaction is possible, the backchannel flow in {{backchannel-request}} substitutes direct client authentication and out-of-band approval for the front channel, and a pre-authorized client exchanges its credential instead ({{token-exchange-profile}}).

## Why Not the Device Authorization Grant

The device authorization grant {{RFC8628}} serves an interactive user who is co-present with a constrained device and can transcribe a short code into a secondary browser. Software statement issuance for non-interactive software has a different shape: the approving party is typically an administrator who is not co-present, acts through an out-of-band workflow, and may respond long after the request. The backchannel flow combined with deferred processing covers that case without a verification URI or user code, while the redirect flow covers issuance that does require interactive authentication or consent.

## Why the Backchannel Flow Uses the Token Endpoint

Two registration-endpoint designs were considered for the backchannel flow. The first extends the {{RFC7591}} registration endpoint with a mode that validates client metadata but returns a software statement instead of creating a client. It offers a JSON-native request body and a response member that mirrors the `software_statement` member of a later registration request. It also inverts the registration contract by deliberately creating no client, provides no standard client authentication beyond initial access tokens, and has no deferral mechanism: {{DTR}} defers token endpoint responses, so pending issuance would require either a second, bespoke polling protocol or a normative dependency on the pending-registration machinery of {{APPROVAL-DCR}}.

The second design follows the split used by {{CIBA}}: a dedicated initiation endpoint accepts a JSON request and, when issuance is pending, returns a deferral code that the client redeems through the same {{DTR}} polling grant used elsewhere in this specification. That design preserves a single deferral model and could be defined later as a compatible extension. It was not adopted because it introduces a new endpoint, new capability metadata, and a new authentication surface, while the token endpoint already provides the client authentication methods, sender constraint, and deferral behavior the flow requires.

The one capability unique to the registration endpoint model, pre-authorization through an out-of-band credential, is available at the token endpoint as well: the credential is presented as the subject token of a token exchange ({{token-exchange-profile}}).

## Why the Issuer Is an Authorization Server Role

An alternative design defines a standalone statement issuer, analogous to the instance issuer of {{CLIENT-INSTANCE}}: an identifier, keys, and issuance obligations, with the minting interface out of scope. That model works for instance attestation because its minting is intra-domain: a platform hands its own workloads their assertions, and interoperability is needed only at presentation. Software statements invert the situation. Presentation has been interoperable since {{RFC7591}}, and the minting interface is precisely the gap this specification exists to fill; leaving it out of scope would reconstruct the proprietary portals and manual provisioning described in the introduction.

Framing the issuer as an authorization server role is what keeps this specification thin. The token endpoint carries the client authentication methods that {{CIMD}} metadata already binds to it, the deferral machinery of {{DTR}}, the token exchange grant of {{RFC8693}}, and discovery through {{RFC8414}}. Each alternative examined in this appendix reinvents some of that machinery to avoid the framing; none of it needs reinventing. The role is also small in practice ({{authorization-server-metadata}}): an issuer that supports only the backchannel flow and the token exchange profile is a token endpoint, a metadata document, and signing keys. An existing attestation authority, such as a workload identity provider, a software publisher's portal, or an enrollment service, can front itself with that single endpoint, mirroring the adapter pattern of {{CLIENT-INSTANCE}} one layer up.

The instance issuer and the statement issuer remain distinct roles even when one party holds both: they attest different subjects (a runtime versus client software), on different lifetimes, through different decision processes, under differently anchored trust (client-delegated descriptors versus verifier-configured issuers). The explicit `typ` values of the two artifact formats keep that boundary verifiable.

## Why One Grant Type, and Why the Software Statement Code Is Not an Authorization Code

One new grant type suffices for both of this specification's token endpoint operations: a request that carries an `software_statement_code` completes a redirect flow, and one that does not initiates a backchannel request. The ambiguity a shared grant once implied is gone, because an initiation from an unauthenticated public client can no longer mint anything synchronously ({{backchannel-request}}).

The software statement code is deliberately not an authorization code. Reusing `authorization_code` for redemption would contradict {{RFC6749}}, whose successful code redemption issues an access token; the OpenID Connect precedent does not apply, because OpenID Connect returns an ID Token alongside a real access token rather than instead of one. An earlier design avoided redemption entirely by returning deferral codes directly from the authorization endpoint, but that coupled the whole redirect flow to an unpublished extension of {{DTR}} and made the response type return an artifact defined by another specification. The software statement code restores the conventional code-shaped front channel, named and defined here, while keeping deferral exclusively at the token endpoint on {{DTR}}'s base mechanism.

## Why Token Exchange Is a Profile, Not the Backchannel

Modeling every backchannel request as token exchange fails on the grant's own terms: {{RFC8693}} requires a `subject_token`, and a client approaching an issuer for the first time in an open ecosystem holds no token to exchange. Requiring one would reintroduce the bilateral pre-arrangement this specification exists to remove, and synthesizing one would satisfy the parameter without satisfying its semantics.

The deeper distinction is provenance. Token exchange derives a token's authority from the token presented; a software statement request derives it from an issuance decision, possibly a human approval completed days later. {{token-exchange-profile}} therefore applies exactly where exchange semantics hold: the client already possesses a token that pre-authorizes issuance, and the exchange records that derivation. The resulting partition is wire-visible: a client that holds such a token exchanges it; a client that holds nothing makes a software statement request.

## Why Deferral Happens Only at the Token Endpoint

Neither the software statement nor any long-lived credential should transit the front channel, and a pending decision must not imply a completion point that has not been reached. Redeeming a short-lived software statement code at the token endpoint satisfies both: an already-completed decision returns the statement immediately, and a pending one defers through {{DTR}} exactly as a backchannel request or token exchange does. Every deferral in this specification therefore originates from a token request, PKCE is consumed by ordinary redemption rather than a special first poll, and no extension to {{DTR}}'s authorization endpoint behavior is required.

## Why the Response Uses `access_token`

OAuth token endpoint infrastructure expects a token response to contain `access_token` and `token_type`, and {{DTR}} returns the originating grant's successful token response after polling. {{RFC8693}} establishes how the same response structure can carry a security token that is not an access token: `access_token` contains the issued artifact, `issued_token_type` identifies its representation, and `token_type=N_A` states that access-token usage semantics do not apply.

Using that convention keeps immediate and deferred responses compatible with existing token endpoint processing while unambiguously stating that a software statement grants no resource access.

## Why Reuse the `registration` Parameter

The `registration` parameter from Section 7.2.1 of {{OIDC-CORE}} is an existing authorization request parameter whose value is a JSON object of client metadata. Reuse avoids defining another container with the same wire format.

This specification deliberately narrows its semantics. The object can only select values already present in the Client ID Metadata Document; it cannot carry issuer claims, approval notes, or arbitrary request context. This restriction makes the attested input deterministic and prevents the overlay from bypassing the canonical metadata document.

# Acknowledgments

This specification builds on {{DTR}}, {{CIMD}}, {{APPROVAL-DCR}}, and {{OIDC-CORE}}. The author thanks the authors and contributors to those specifications and the OAuth working group participants whose discussions informed this work.
