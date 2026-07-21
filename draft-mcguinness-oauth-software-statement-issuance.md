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
  CIBA:
    target: https://openid.net/specs/openid-client-initiated-backchannel-authentication-core-1_0.html
    title: "OpenID Connect Client-Initiated Backchannel Authentication Flow - Core 1.0"
  CLIENT-INSTANCE:
    target: https://datatracker.ietf.org/doc/draft-mcguinness-oauth-client-instance-assertion
    title: "OAuth 2.0 Client Instance Assertion"
  OPENID-FED:
    target: https://openid.net/specs/openid-federation-1_0.html
    title: "OpenID Federation 1.0"
  PUSHED-DCR:
    target: https://datatracker.ietf.org/doc/draft-richer-oauth-pushed-client-registration
    title: "OAuth 2.0 Pushed Client Registration"
  TRUST-FRAMEWORK:
    target: https://datatracker.ietf.org/doc/draft-mcguinness-oauth-id-assertion-framework
    title: "OAuth Identity Assertion Trust Framework"

--- abstract

This specification defines OAuth 2.0 flows through which a client requests a software statement, as defined in RFC 7591, from an authorization server. The redirect flow uses a new authorization endpoint `response_type` value, `software_statement`; the client redeems the resulting intermediate code with the standard `authorization_code` grant type. The backchannel flow uses a new extension grant. A Client ID Metadata Document identifies a client that has not been registered with the authorization server. An optional `registration` parameter selects a request-specific subset of the metadata to be attested.

When issuance can complete synchronously, the authorization endpoint returns a short-lived code that the client exchanges at the token endpoint for the software statement. When issuance requires additional processing or approval, the authorization server instead returns the direct authorization endpoint response defined by Deferred Token Response. That response contains a `deferral_code`, which the client uses to poll the token endpoint for the eventual software statement. The software statement itself is never placed in an authorization response URL.

A client without access to a user agent can instead send a backchannel software statement request directly to the token endpoint using the extension grant. Issuance that cannot complete immediately then uses the deferred token response defined by Deferred Token Response, without any redirect.

--- middle

# Introduction

Section 2.3 of {{RFC7591}} defines a software statement as a JSON Web Token (JWT) that asserts client metadata. A client can present the software statement to a dynamic client registration endpoint, where the statement's signature enables the authorization server to determine who attested to the metadata. {{RFC7591}} does not define a protocol for obtaining a software statement.

In practice, software statements are commonly issued through manual provisioning, deployment-specific portals, or proprietary federation processes. Those mechanisms do not give clients an interoperable way to request a statement, redirect a user or administrator for interaction, and retrieve the resulting credential without exposing it through the browser.

The client establishment mechanisms adjacent to this specification are bilateral: pre-registration of a client identifier URL as described by {{CIMD}}, pushed registration of ephemeral clients {{PUSHED-DCR}}, and approval-gated registration {{APPROVAL-DCR}} each establish trust at a single authorization server, and each asks that authorization server to decide on the client's own assertion of its metadata. A software statement changes both properties: the metadata a trusting authorization server evaluates originates from an issuer's signed issuance decision rather than from the client alone ({{what-issuance-attests}}), and one issuance decision is honored at every authorization server in the statement's audience rather than being repeated at each ({{comparison}}).

The consumption side of this design is already standardized and deployed: {{RFC7591}} registration endpoints have accepted the `software_statement` request member since 2015. This specification supplies the missing issuance protocol and hardens the artifact so that independent issuers and consumers interoperate ({{software-statement-format}}). The resulting portability is bounded by trust: a statement is honored only where the trusting authorization server is configured to trust its issuer, so its practical audience is an ecosystem or administrative domain rather than the open web.

The protocol introduces `response_type=software_statement` at the authorization endpoint and one extension grant, `urn:ietf:params:oauth:grant-type:software-statement`, for initiating a backchannel request at the token endpoint; an intermediate code from the redirect flow is redeemed with the standard `authorization_code` grant type. Clients identify themselves with the HTTPS URL and metadata document defined by {{CIMD}}. If issuance is approved without deferral, the authorization server returns an intermediate code that the client redeems for the software statement. If issuance remains pending and the client opted in to deferral, the authorization server returns a `deferral_code` directly from the authorization endpoint and the client polls the token endpoint for the eventual response. A client without access to a user agent instead initiates the request with the software statement grant.

The flow concerns client establishment, not authorization to access a protected resource. Consequently, a software statement request cannot be combined with `scope`, `resource`, `authorization_details`, or an access-token-producing response type.

The redirect flow requires the client to operate a redirection endpoint and to reach the authorization endpoint through a user agent. For software without access to a user agent, such as a daemon or command-line tool, this specification defines a backchannel flow ({{backchannel-request}}) in which the client requests the software statement directly at the token endpoint and any approval completes out of band through deferred processing.

Not every client establishment problem calls for a software statement. When the approving party is the resource owner and the client needs only the authorization server that resource owner uses, the approval is the OAuth authorization grant itself, and identifying the client by its metadata document {{CIMD}} suffices. A software statement earns its cost when the trust decision is made by a party other than the user in a transaction, must be made before any transaction exists, or must be made once and honored at many authorization servers. Issuance is therefore deliberately decoupled from any access-granting transaction and can complete out of band over hours or days.

## Relationship to Other Specifications

This specification uses the following building blocks:

* {{RFC7591}} defines the software statement and the client metadata carried in it.
* {{CIMD}} defines the client identifier and the canonical source of client metadata, including pre-registration with an authorization server and the handling of changed metadata. The flows in this specification can serve as the approval-carrying enrollment step, and the canonical digest ({{metadata-snapshot}}) makes metadata change precisely detectable.
* Section 7.2.1 of {{OIDC-CORE}} defines the syntax of the `registration` authorization request parameter. This specification defines a more constrained use of that parameter for selecting metadata to be attested.
* {{DTR}} defines client opt-in, deferred token responses at the token endpoint, direct deferral at the authorization endpoint, polling, cancellation, callbacks, PKCE verification on the first poll, and sender constraint. This specification profiles the token endpoint deferral for the backchannel flow and the direct-deferral mechanism for the redirect flow, and defines each originating request's successful response.
* {{RFC8693}} establishes the token response convention used to return a security token that is not an OAuth access token.

*Editor's note (remove before publication): The direct authorization endpoint deferral profiled by the redirect flow, including PKCE verification on the first poll and the `deferred_authorization_endpoint_supported` metadata member, currently exists only as an open change proposal against DTR (https://github.com/maxwellgerber/deferred-token-response/pull/47). This specification cannot advance until that proposal is merged into DTR or published as equivalent stable text.*

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
: A request in which a client asks an authorization server to issue a software statement, sent either as an authorization request with `response_type=software_statement` (the redirect flow) or as a `urn:ietf:params:oauth:grant-type:software-statement` grant request to the token endpoint (the backchannel flow).

Issuing Authorization Server:
: The authorization server that processes a software statement request and signs the resulting software statement.

Trusting Authorization Server:
: An authorization server that accepts the issued software statement in a dynamic client registration request.

Registration Overlay:
: A JSON object carried in the `registration` request parameter that selects request-specific values from the client's canonical metadata. The overlay cannot add to or widen that metadata.

Intermediate Code:
: The short-lived, single-use code returned by a synchronous authorization response ({{synchronous-response}}) and redeemed with the `authorization_code` grant type ({{code-redemption}}). It is not an authorization code: redeeming it yields the software statement token response, never an access token.

Metadata Snapshot:
: The validated canonical metadata and accepted registration overlay bound to a request, as defined in {{metadata-snapshot}}.

Canonical Digest:
: The unpadded base64url encoding of the SHA-256 hash of a metadata document in the canonical form of {{RFC8785}}, as defined in {{metadata-snapshot}}.

# Protocol Overview

A client initiates a software statement request through one of two flows. The redirect flow uses the authorization endpoint and a user agent and supports interactive authentication or consent during issuance. The backchannel flow uses only the token endpoint and serves software without access to a user agent. In both flows, the authorization server either completes the issuance decision synchronously or defers it.

## Redirect Flow Overview

~~~
+--------+                                  +----------------------+
| Client |                                  | Authorization Server |
+--------+                                  +----------------------+
    |                                                 |
    | (A) Authorization request                       |
    |     response_type=software_statement            |
    |     client_id=<metadata document URL>           |
    |     code_challenge, [registration],             |
    |     [audience],                                 |
    |     [completion_mode=deferred]                  |
    |------------------------------------------------>|
    |                                                 |
    | (B1) Synchronous response (code)                |
    |<------------------------------------------------|
    |                     or                          |
    | (B2) Deferred response                          |
    |      (deferral_code, expires_in, [interval])    |
    |<------------------------------------------------|
    |                                                 |
    | (C1) Code exchange (code, code_verifier)        |
    |                     or                          |
    | (C2) First poll (deferral_code, code_verifier)  |
    |------------------------------------------------>|
    |                                                 |
    | (D) Software statement response or              |
    |     authorization_pending                       |
    |<------------------------------------------------|
    |                                                 |
    | (E) Subsequent poll(s), if still pending        |
    |------------------------------------------------>|
    |     authorization_pending or final response     |
    |<------------------------------------------------|
~~~

At (A), the client requests a software statement. If a `registration` overlay is present, the client uses PAR {{RFC9126}}. The authorization server validates the client identifier, redirect URI, canonical metadata, overlay, and request binding parameters. It can perform user-agent interaction or initiate an approval workflow.

If issuance can complete without deferral, the authorization server returns a short-lived code at (B1). The client exchanges the code and PKCE verifier at (C1) and receives the software statement at (D).

If issuance remains pending and the client opted in to deferral, the authorization server returns the direct authorization endpoint deferred response at (B2). The client presents the `deferral_code` and PKCE verifier on its first polling request at (C2). That request returns the software statement or `authorization_pending` at (D). If processing is still pending, the client continues polling at (E) without re-presenting the verifier. A successful first or subsequent poll returns the same software statement response as the synchronous path. Authorization endpoint and polling errors follow {{RFC6749}} and {{DTR}}, respectively.

## Backchannel Flow Overview

~~~
+--------+                                  +----------------------+
| Client |                                  | Authorization Server |
+--------+                                  +----------------------+
    |                                                 |
    | (A) Token request                               |
    |     grant_type=...:software-statement           |
    |     client_id=<metadata document URL>           |
    |     [initial_access_token],                     |
    |     [registration],                             |
    |     [audience],                                 |
    |     [completion_mode=deferred]                  |
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

At (A), the client sends the software statement grant request directly to the token endpoint, authenticating according to {{client-identity}} when the Client ID Metadata Document establishes an authentication method, presenting an `initial_access_token` when the authorization server requires one, and optionally including a `registration` overlay in the request body. If issuance completes immediately, the authorization server returns the software statement at (B1). If issuance remains pending and the client opted in to deferral, the authorization server returns the deferred token response of {{DTR}} at (B2), and the client polls at (C) until it receives the final response. {{backchannel-request}} defines this flow.

# Client Identification and Authentication {#client-identity}

The `client_id` in a software statement request MUST be a client identifier URL conforming to {{CIMD}}. The authorization server MUST obtain and validate the corresponding Client ID Metadata Document according to {{CIMD}}. A non-URL identifier issued by the authorization server is not supported by this specification.

In the redirect flow, the authorization server MUST compare the `redirect_uri` in the request with the `redirect_uris` in the Client ID Metadata Document according to {{CIMD}} and {{RFC9700}}. The authorization server MUST NOT redirect the user agent when the client identifier or redirect URI is missing or invalid.

The client uses the `token_endpoint_auth_method` and related key metadata in its Client ID Metadata Document when authenticating to the PAR and token endpoints. If the metadata does not establish a client authentication method usable at the authorization server, the client is treated as a public client.

A public client that opts in to deferral in the redirect flow MUST include `dpop_jkt` in the authorization request and MUST present a DPoP proof signed with the corresponding key on every polling request. A public client that opts in to deferral in the backchannel flow ({{backchannel-request}}) MUST instead include a DPoP proof on the initiating token request; the authorization server MUST bind any resulting deferral state to that proof's key, and the client MUST use the same key on every polling request. A confidential client MAY use `dpop_jkt` in addition to its client authentication method. When `dpop_jkt` is present and the authorization server returns a synchronous code, the client MUST also present a DPoP proof signed with the corresponding key when exchanging that code. All uses of DPoP MUST follow {{RFC9449}} and {{DTR}}.

## Metadata Snapshot {#metadata-snapshot}

Client metadata documents can change while a request is pending. Before returning either an intermediate code or a deferral code, the authorization server MUST bind the returned credential to the validated canonical metadata and registration overlay. The bound values constitute the metadata snapshot for the request.

The authorization server MAY retrieve the document again before issuing the software statement. If it does so and detects a security-relevant change (for example, a change to `jwks`, `jwks_uri`, `redirect_uris`, `token_endpoint_auth_method`, or any metadata selected by the overlay), it MUST either re-evaluate the request under the new metadata or reject the request. It MUST NOT silently combine values from different document versions.

Because JSON member ordering and serialization details carry no meaning, comparing retrieved documents byte for byte over-detects change. Before calculating a canonical digest, the authorization server MUST verify that the document can be represented as the I-JSON input required by {{RFC8785}}. In particular, it MUST reject a document containing duplicate object member names or a number that cannot be represented as an IEEE 754 double-precision value without loss.

The canonical digest of a metadata document is the unpadded base64url encoding of the SHA-256 hash {{RFC6234}} of the UTF-8 bytes produced by applying {{RFC8785}} to the complete document. Two retrievals with the same canonical digest contain the same metadata for purposes of this specification; a changed digest is the concrete signal for the re-evaluation requirement above. The canonical digest also serves the `cimd_digest` claim ({{software-statement-format}}) and the audit guidance in {{security-considerations}}. Deployments MAY use other deterministic encodings internally for change detection, but interoperable uses of the canonical digest MUST use the form defined here.

# Software Statement Authorization Request {#authorization-request}

The client sends an authorization request as described in Section 4.1.1 of {{RFC6749}}, with the following parameters:

`response_type`:
: REQUIRED. The value MUST be `software_statement`.

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
: OPTIONAL. The mechanism for returning authorization response parameters, as defined by {{OAUTH-MRT}}. The default response mode for `response_type=software_statement` is `query`. Response mode requirements for deferred responses are given in {{deferred-authorization-response}}.

`registration`:
: OPTIONAL. A JSON object containing the registration overlay described in {{registration-overlay}}. A request containing this parameter MUST be submitted using PAR {{RFC9126}}.

`audience`:
: OPTIONAL. A target service at which the client intends to use the issued software statement, with the semantics of the `audience` parameter defined in Section 2.1 of {{RFC8693}}. The parameter MAY be repeated to request multiple audiences. Each value MUST be an authorization server issuer identifier as defined by {{RFC8414}}, values MUST NOT be repeated, and order is not significant. The authorization server determines the final audience according to policy, but when this parameter is present every value in the statement's `aud` claim MUST have appeared in the request. If none of the requested audiences is acceptable, the authorization server MUST reject the request with `invalid_target` {{RFC8693}}. Some deployments use a proprietary `audience` authorization request parameter to select an API for access tokens; the semantics defined here apply only to software statement requests.

`completion_mode`:
: OPTIONAL. The client MAY include `deferred` among the values of this parameter to opt in to direct authorization endpoint deferral as defined by {{DTR}}. If the parameter is absent or does not include `deferred`, the authorization server MUST NOT return a direct deferred response.

`dpop_jkt`:
: REQUIRED for a public client when `completion_mode` includes `deferred`, and OPTIONAL otherwise. The parameter has the semantics defined in Section 10 of {{RFC9449}}. When present, its value MUST be associated with the resulting intermediate code or the direct-deferral state, whichever is returned.

The authorization server MUST reject with `invalid_request` a request that omits a required PKCE parameter or, for a public client opting in to deferral, omits `dpop_jkt`.

The following is a non-normative example of an authorization request (line breaks are for display purposes only):

~~~ http
GET /authorize?response_type=software_statement
  &client_id=https%3A%2F%2Fclient.example.org%2Fmetadata.json
  &redirect_uri=https%3A%2F%2Fclient.example.org%2Fcb
  &state=af0ifjsldkj
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

Hybrid response types that combine `software_statement` with `code`, `token`, `id_token`, or any other response type are not defined. An authorization server MUST reject such a request with `unsupported_response_type`.

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

After validating the request and performing any immediate interaction, the authorization server returns a synchronous response, a direct deferred response, or an error. The authorization server MUST NOT place the software statement or approval-sensitive information in any authorization response.

## Synchronous Response {#synchronous-response}

If issuance has been approved and no further processing is required, the authorization server redirects the user agent to the client's redirection endpoint with:

`code`:
: REQUIRED. A short-lived, single-use intermediate code. The code MUST be bound to the client identifier, redirect URI, PKCE challenge, metadata snapshot (which incorporates any accepted registration overlay), and requested audience. If the authorization request was bound to a DPoP key, the code MUST also be bound to that key's thumbprint.

`state`:
: REQUIRED. The exact value received in the authorization request.

`iss`:
: REQUIRED. The authorization server issuer identification parameter defined by {{RFC9207}}.

The code MUST contain sufficient entropy to prevent guessing, MUST expire shortly after issuance, and MUST NOT be used more than once. The requirements for authorization codes in Section 4.1.2 of {{RFC6749}} otherwise apply.

The synchronous response MUST NOT contain a `deferral_code`.

## Direct Deferred Response {#deferred-authorization-response}

If issuance remains pending and the request included `deferred` in `completion_mode`, the authorization server MAY use the direct authorization endpoint deferral mechanism of {{DTR}}. It returns the following parameters to the client's redirection endpoint through the user agent, using the selected response mode:

`deferral_code`:
: REQUIRED. The deferral code defined by {{DTR}}. It MUST be bound to the client identifier, redirect URI, PKCE challenge, metadata snapshot (which incorporates any accepted registration overlay), and requested audience. If `dpop_jkt` was present, it MUST also be bound to that JWK thumbprint.

`state`:
: REQUIRED. The exact value received in the authorization request.

`expires_in`:
: REQUIRED. The remaining lifetime of the deferral code in seconds.

`interval`:
: OPTIONAL. The minimum number of seconds the client SHOULD wait before making its first polling request.

`iss`:
: REQUIRED. The authorization server issuer identification parameter defined by {{RFC9207}}.

The deferred response MUST NOT contain `code` or any other parameter that indicates synchronous completion. Because a deferral code is longer-lived than an intermediate code, a client that opts in to deferral SHOULD request the `form_post` response mode {{FORM-POST}}, which delivers the response parameters in the body of an HTTP POST rather than in a URL, and an authorization server that supports deferral SHOULD support that response mode. The fragment response mode SHOULD NOT be used, because scripts at the redirection endpoint can access the fragment; the default query response mode exposes the deferral code to URL-based logging, browser history, and Referer headers.

For example, using the form_post response mode:

~~~ http
HTTP/1.1 200 OK
Content-Type: text/html;charset=UTF-8
Cache-Control: no-store

<html>
 <body onload="javascript:document.forms[0].submit()">
  <form method="post" action="https://client.example.org/cb">
   <input type="hidden" name="deferral_code"
          value="8d67dc78-7faa-4d41-aabd-67707b374255"/>
   <input type="hidden" name="state" value="af0ifjsldkj"/>
   <input type="hidden" name="expires_in" value="900"/>
   <input type="hidden" name="interval" value="30"/>
   <input type="hidden" name="iss"
          value="https://server.example.com"/>
  </form>
 </body>
</html>
~~~

If the authorization server cannot complete synchronously and cannot or will not use direct deferral, it returns an authorization error response as described in Section 4.1.2.1 of {{RFC6749}}. When issuance could have been deferred had the client opted in, the error code SHOULD be `deferral_required`:

`deferral_required`:
: The authorization server cannot complete the software statement request synchronously, and the request did not include `deferred` in `completion_mode`. The client MAY repeat the request with `deferred` included in `completion_mode`.

This error code lets a client distinguish a request that can be retried with deferral from one that was denied.

# Intermediate Code Redemption {#code-redemption}

The client redeems an intermediate code with the `authorization_code` grant type of {{RFC6749}}. The code, not the grant type, determines what the token endpoint returns, mirroring how OpenID Connect returns artifacts other than access tokens through the same grant. Backchannel initiation uses the extension grant defined in {{backchannel-request}} and never involves an intermediate code; a redemption request missing `code` is an ordinary `invalid_request` under {{RFC6749}} and cannot initiate issuance work.

The client redeems the intermediate code by sending an HTTP `POST` request to the token endpoint using the `application/x-www-form-urlencoded` format. The request contains:

`grant_type`:
: REQUIRED. The value MUST be `authorization_code`.

`code`:
: REQUIRED. The intermediate code returned by the authorization endpoint.

`redirect_uri`:
: REQUIRED. The same redirection URI used in the authorization request.

`client_id`:
: REQUIRED. The same client identifier URL used in the authorization request.

`code_verifier`:
: REQUIRED. The PKCE verifier corresponding to the `code_challenge` in the authorization request.

The client authenticates according to {{client-identity}}. When `dpop_jkt` was included in the authorization request, the client MUST send a DPoP proof for the token endpoint using the same key.

The authorization server MUST validate the code and all of its bindings before processing the request. An invalid, expired, previously used, or incorrectly bound code MUST result in an `invalid_grant` error. A PKCE or DPoP binding failure is handled according to {{RFC7636}} or {{RFC9449}}, respectively.

Because the intermediate code was issued for `response_type=software_statement`, a valid redemption returns the software statement response in {{software-statement-response}}; it MUST NOT return an access token or refresh token. Conversely, an authorization code issued for any other response type MUST NOT be redeemed for a software statement. Because the synchronous authorization response indicates that issuance has already been approved, redemption of an intermediate code never returns a deferred response.

# Backchannel Software Statement Request {#backchannel-request}

A client without access to a user agent can request a software statement directly at the token endpoint. An authorization server advertises support for this flow by listing `urn:ietf:params:oauth:grant-type:software-statement` in `grant_types_supported` ({{authorization-server-metadata}}); a client MUST NOT send a backchannel request to an authorization server that does not advertise that value.

The client sends an HTTP `POST` request to the token endpoint using the `application/x-www-form-urlencoded` format with:

`grant_type`:
: REQUIRED. The value MUST be `urn:ietf:params:oauth:grant-type:software-statement`.

`client_id`:
: REQUIRED. The client identifier URL described in {{client-identity}}.

`registration`:
: OPTIONAL. A JSON object containing the registration overlay described in {{registration-overlay}}, subject to the same validation rules. Rule 6 does not apply, because the request uses no `redirect_uri`.

`audience`:
: OPTIONAL. The requested audience described in {{authorization-request}}. The same syntax, validation, and narrowing rules apply.

`completion_mode`:
: OPTIONAL. The client MAY include `deferred` among the values of this parameter to opt in to a deferred token response as defined by {{DTR}}. If the parameter is absent or does not include `deferred`, the authorization server MUST NOT return a deferred token response.

`initial_access_token`:
: OPTIONAL. An authorization credential issued out of band by the authorization server that pre-authorizes software statement requests, analogous to the initial access token of {{RFC7591}}. An authorization server MAY require this parameter for backchannel requests. The credential is distinct from OAuth client authentication and is used together with client authentication when the Client ID Metadata Document establishes an authentication method.

`client_notification_token`:
: OPTIONAL. The callback authentication credential defined by {{DTR}}. It MUST be used only when `completion_mode` includes `deferred`. If the client has a `deferred_client_notification_endpoint` and intends to accept callbacks, it SHOULD include this parameter.

The request MUST NOT contain `code`, `redirect_uri`, `code_verifier`, `state`, or `dpop_jkt`. The prohibitions of {{prohibited-parameters}} apply: the request MUST NOT contain `scope`, `resource`, or `authorization_details`.

The client authenticates according to {{client-identity}} when its metadata establishes an authentication method and presents an `initial_access_token` when the authorization server requires one. A public client that opts in to deferral MUST additionally include a DPoP proof {{RFC9449}} for the token endpoint on the initiating request; the authorization server MUST bind any resulting deferral state to the proof's public key. A confidential client MAY additionally use DPoP. A DPoP proof establishes sender constraint but, by itself, does not authorize the presenter to request a statement for the `client_id` URL.

When an `initial_access_token` is presented, the authorization server MUST validate it before retrieving client-controlled metadata or enqueueing approval work. The credential SHOULD be integrity protected, confidential in transit and at rest, limited to the issuing authorization server, time limited, and bound to an exact client identifier URL or an explicitly authorized client identifier namespace; it MAY further restrict audiences, metadata, overlays, or the number of uses. An opaque bearer credential SHOULD contain at least 128 bits of entropy. The authorization server MUST enforce every restriction the credential carries and MUST prevent replay beyond its permitted number of uses.

The authorization server MUST obtain and validate the Client ID Metadata Document and any overlay exactly as in the redirect flow and MUST bind the metadata snapshot ({{metadata-snapshot}}) and the requested audience before returning either the software statement or a deferral code. Because no user agent is present, approval and any verification of the requesting party occur out of band. The authorization server MUST apply the same issuance policy to backchannel requests as to redirect flow requests.

If issuance completes immediately, the authorization server returns the software statement token response ({{software-statement-response}}). If issuance remains pending and the request opted in to deferral, the authorization server returns the deferred token response defined by {{DTR}}, and the client polls according to {{deferred-processing}}. If issuance cannot complete immediately and the request did not opt in, the authorization server returns a token error response with HTTP status code 400 and the error code `deferral_required` ({{deferred-authorization-response}}).

Other backchannel errors use the token error response format of Section 5.2 of {{RFC6749}}:

* A malformed request, invalid Client ID Metadata Document, invalid overlay, or unacceptable requested audience results in `invalid_request` with HTTP status code 400.
* Failed client authentication, or a missing, invalid, expired, or insufficient `initial_access_token` where the authorization server requires one, results in `invalid_client` with HTTP status code 401.
* A request rejected by issuance policy results in `access_denied` with HTTP status code 400.
* A server that does not support the backchannel grant returns `unsupported_grant_type` with HTTP status code 400.

These errors are terminal for the submitted request, except that a client receiving `deferral_required` MAY submit a new request that opts in to deferral. The authorization server MUST include `Cache-Control: no-store` on every backchannel error response.

The following is a non-normative example of an initiating backchannel request (line breaks are for display purposes only):

~~~ http
POST /token HTTP/1.1
Host: server.example.com
Content-Type: application/x-www-form-urlencoded
DPoP: eyJ0eXAiOiJkcG9wK2p3dCIsImFsZyI6IkVTMjU2IiwiandrIjp7Li4ufX0...

grant_type=urn%3Aietf%3Aparams%3Aoauth%3Agrant-type%3A
  software-statement
  &client_id=https%3A%2F%2Fclient.example.org%2Fmetadata.json
  &completion_mode=deferred
  &initial_access_token=iat_2f7e9c1a6d...
~~~

# Deferred Processing {#deferred-processing}

A deferral originates either from the direct deferred authorization response in {{deferred-authorization-response}} or from a deferred token response to a backchannel request ({{backchannel-request}}). In both cases the client polls the token endpoint using the polling grant defined by {{DTR}}.

Approval of a software statement request can take hours or days rather than the seconds typical of user authentication, for example when it involves reviewing the client's policy or compliance documentation. Issuers SHOULD set deferral code lifetimes that reflect their actual approval latency. A backchannel client with a registered notification endpoint SHOULD use the callback mechanism of {{DTR}} together with `client_notification_token`, while continuing to poll as required by {{DTR}}. This specification does not enable callbacks for direct authorization endpoint deferrals because their originating authorization request does not establish a `client_notification_token`.

## First Polling Request {#first-polling-request}

The first polling request is an HTTP `POST` request to the token endpoint using the `application/x-www-form-urlencoded` format. It contains:

`grant_type`:
: REQUIRED. The value MUST be `urn:ietf:params:oauth:grant-type:deferred`.

`deferral_code`:
: REQUIRED. The deferral code returned in the deferred response.

`code_verifier`:
: REQUIRED when the deferral originated from a deferred authorization response; the PKCE verifier corresponding to the `code_challenge` in the authorization request. The parameter MUST NOT be present when the deferral originated from a backchannel request, which has no PKCE state.

For a deferral that originated from a deferred authorization response, the authorization server MUST verify the `code_verifier` against the stored PKCE challenge. If verification fails, it MUST return `invalid_grant` and MUST terminate the deferral. After successful verification, it records that PKCE verification has completed for the deferral state. For a deferral that originated from a backchannel request, the authorization server MUST instead verify that the request presents the client authentication or DPoP key bound at initiation.

The client authenticates according to {{client-identity}}. If the deferral is bound to a DPoP key, through `dpop_jkt` in the redirect flow or the initiating DPoP proof in the backchannel flow, the client MUST send a DPoP proof signed with the corresponding key. The authorization server MUST reject a proof whose key does not match the stored thumbprint.

The first polling request MUST NOT contain an intermediate `code`, `redirect_uri`, `registration`, `audience`, `completion_mode`, `initial_access_token`, or `client_notification_token` parameter.

## Subsequent Polling Requests

If the request remains pending, the client continues polling according to {{DTR}} using only the polling grant's normal parameters. It MUST NOT re-present the `code_verifier`. The authorization server preserves the successful PKCE verification in the deferral state.

Each subsequent polling request MUST use the client authentication or DPoP sender constraint established for the deferral. In particular, a public client MUST send a DPoP proof on every poll, using the key identified by `dpop_jkt` in the redirect flow or the key of the initiating DPoP proof in the backchannel flow.

Pending, denied, expired, cancelled, and polling-rate behavior follows {{DTR}}. Callback behavior follows {{DTR}} only for a backchannel request that supplied `client_notification_token`. A successful first or subsequent polling response is the software statement token response defined in {{software-statement-response}}.

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
  "access_token": "eyJhbGciOiJFUzI1NiIsImtpZCI6IjEyMyJ9...",
  "issued_token_type":
    "urn:ietf:params:oauth:token-type:software-statement",
  "token_type": "N_A",
  "expires_in": 3600
}
~~~

To use the issued statement for dynamic client registration, the client supplies the value of `access_token` as the `software_statement` member of an {{RFC7591}} client registration request.

This specification defines no renewal mechanism. A client obtains a replacement for an expiring or expired software statement by performing a new software statement request; whether replacement requires new approval is determined by issuer policy.

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

The JWT payload SHOULD also contain:

`cimd_digest`:
: The canonical digest ({{metadata-snapshot}}) of the Client ID Metadata Document from which the metadata snapshot was derived. This claim binds the statement to the exact document content evaluated during issuance and lets any party determine whether the client's currently published metadata still matches what was attested.

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

Before accepting the statement, a trusting authorization server MUST verify that the `typ` header carries the value `software-statement+jwt`; verify the signature under a key trusted for the exact `iss` value; validate the presence and type of every required claim; validate `iss`, `aud`, `iat`, and `exp`; verify that `sub` is a Client Identifier URL conforming to {{CIMD}}; reject every metadata claim prohibited by {{registration-overlay}}; and apply the JWT validation guidance in {{RFC8725}}. In particular, it MUST reject a statement whose `iat` is unreasonably far in the future according to its clock-skew policy. Publishing an issuer URL or a JWK Set does not by itself establish trust: signature keys for a trusted issuer are conventionally obtained from the `jwks_uri` in the issuer's authorization server metadata {{RFC8414}}, but the decision to trust the issuer MUST come from explicit local configuration. When the statement contains `cimd_digest`, a trusting authorization server MAY retrieve the client's current metadata document and compare canonical digests to learn whether the attested content is still published; if it does so, it MUST also verify that the document's `client_id` is exactly equal to `sub` as required by {{CIMD}}. A digest mismatch indicates post-issuance change and is an input to registration policy rather than a validation failure. The trusting authorization server then processes the software statement according to {{RFC7591}} and its local registration policy.

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

An authorization server that issues software statements under this specification advertises `true` for `client_id_metadata_document_supported`, as defined by {{CIMD}}, and publishes `software_statement_signing_alg_values_supported` below.

An authorization server supporting the redirect flow additionally advertises:

* `software_statement` in `response_types_supported`;
* `authorization_code` in `grant_types_supported`, used to redeem intermediate codes ({{code-redemption}}); and
* `true` for `authorization_response_iss_parameter_supported`, as defined by {{RFC9207}}.

If it can defer redirect-flow requests directly at the authorization endpoint, it also advertises `true` for both `deferred_token_response_supported` and `deferred_authorization_endpoint_supported`, as defined by {{DTR}}.

An authorization server supporting the backchannel flow advertises `urn:ietf:params:oauth:grant-type:software-statement` in `grant_types_supported`. If it can defer backchannel requests, it also advertises `true` for `deferred_token_response_supported`. A backchannel-only implementation does not advertise `software_statement` in `response_types_supported` and need not advertise `authorization_response_iss_parameter_supported`.

A client MUST NOT rely on direct authorization endpoint deferral unless both `deferred_token_response_supported` and `deferred_authorization_endpoint_supported` are `true`.

This specification defines the following additional authorization server metadata members:

`software_statement_signing_alg_values_supported`:
: REQUIRED for an authorization server that issues software statements under this specification. A JSON array containing the asymmetric JWS `alg` values that the authorization server can use to sign software statements. The array MUST NOT contain `none` or a symmetric algorithm. This member describes the issuing role; an authorization server that only accepts software statements does not publish it.

`software_statement_audiences_supported`:
: OPTIONAL. A JSON array containing authorization server issuer identifiers that the issuer is prepared to place in a software statement's `aud` claim. Omission means that the complete set is not publicly enumerable; it does not mean that no audience is supported. Publication does not guarantee that every client is eligible for every listed audience.

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

The software statement is a signed credential and can contain sensitive deployment information. It MUST NOT be returned through the authorization endpoint. The synchronous path returns only a short-lived code and applies exact redirect URI matching, PKCE with `S256`, and the other applicable authorization code protections in {{RFC9700}}.

Before using either a code or deferral code, the client MUST verify `state` and MUST validate the authorization response `iss` parameter according to {{RFC9207}}. Although PKCE with `S256` already provides the cross-site request forgery protection described in {{RFC9700}}, this specification requires `state` because a client that opts in to deferral must correlate a response that can arrive on either the synchronous or the deferred path with its originating request before deciding how to redeem it.

A direct deferred response returned in a URL exposes the deferral code to browser history, referrer fields, redirection-target logs, and other front-channel observers, and deferral codes can remain valid substantially longer than intermediate codes. The `form_post` response mode {{FORM-POST}} keeps the deferral code out of URLs and SHOULD be used for deferred responses as described in {{deferred-authorization-response}}; the fragment response mode SHOULD NOT be used.

## Direct Deferral Sender Constraint

All software statement requests made through the authorization endpoint use PKCE. For a direct deferred response, the authorization server stores the PKCE challenge and verifies the corresponding verifier on the first polling request. The verifier MUST NOT be accepted on subsequent polls.

After the first poll consumes the verifier, each subsequent poll is protected by client authentication or DPoP as required by {{DTR}}. A public client that opts in to deferral is therefore required to supply `dpop_jkt` in the authorization request and use that key on every poll. This preserves sender-constraint continuity across the authorization response and polling sequence.

All additional sender-constraint, polling-rate, replay, cancellation, and logging requirements of {{DTR}} apply. Callback behavior applies only to backchannel deferrals as described in {{backchannel-request}}; this specification does not establish callback authentication for a direct authorization endpoint deferral.

## Backchannel Request Considerations

A backchannel request reaches the token endpoint without prior user-agent interaction. An unauthenticated request from a public client can therefore trigger metadata document retrieval and enqueue approval work at little cost to the requester. Authorization servers SHOULD rate-limit backchannel requests, SHOULD cache metadata retrieval results and failures, and MAY require an `initial_access_token`; when one is presented, it MUST be validated before client-controlled metadata is retrieved or approval work is enqueued. Client authentication remains mandatory when established by the Client ID Metadata Document.

An `initial_access_token` is an authorization credential, not a client identifier and not a substitute for client authentication when the Client ID Metadata Document establishes an authentication method. It is sent in a form body and is therefore exposed to any component that records request bodies. Authorization servers MUST exclude it from logs, traces, error messages, and audit records; clients and authorization servers MUST protect it as a bearer credential unless its format provides proof of possession. The binding, lifetime, entropy, and replay requirements of {{backchannel-request}} limit the effect of disclosure.

The absence of a user agent removes front-channel exposure: neither the deferral code nor any other response parameter transits a browser. It also removes any in-band evidence of user participation. An authorization server MUST NOT treat a backchannel request as implying prior user consent and MUST apply the same issuance and approval policy as for the redirect flow.

Sender constraint for a backchannel deferral is established at initiation through client authentication or a DPoP proof. There is no PKCE state, so the `code_verifier` check in {{first-polling-request}} is replaced by verification of the initiating key or credentials on every poll. If a callback is configured, the client uses `client_notification_token` as defined by {{DTR}}; that token authenticates the callback but does not authorize issuance.

## Statement Validation and Replay

A software statement is intended to be presented more than once ({{multi-instance}}); until it expires, it can equally be presented by a party that steals it, at every registration endpoint included in its audience. Issuers SHOULD use the narrowest practical audience and lifetime. Trusting authorization servers SHOULD track `jti` values to inventory the registrations derived from a statement and to enforce any local bound on their number.

When registrations derived from a statement are intended to share a client key, and the canonical metadata provides `jwks` or `jwks_uri`, the issuer SHOULD include that member in the attested metadata. A trusting authorization server can then require proof of possession of a corresponding private key during or after registration, which renders a stolen statement unusable to a party that does not also hold the client's key. When each registration is expected to supply a distinct instance key, the issuer MUST omit `jwks` and `jwks_uri` so that the plain registration metadata can carry that key without conflicting with the precedence rule of {{RFC7591}}.

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
: `software_statement`

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
: This specification, {{software-statement-response}}

## OAuth Parameters Registry

This specification requests that IANA add this specification, {{registration-overlay}} and {{backchannel-request}}, as an additional reference for the existing `registration` parameter and extend its usage location to include token requests. The parameter name and change controller are unchanged.

This specification further requests that IANA add this specification, {{authorization-request}} and {{backchannel-request}}, as an additional reference for the existing `audience` parameter registered by {{RFC8693}} and extend its usage location to include authorization requests. The parameter name and change controller are unchanged.

This specification also requests registration of the following value in the IANA "OAuth Parameters" registry established by {{RFC6749}}:

Parameter Name:
: `initial_access_token`

Parameter Usage Location:
: token request

Change Controller:
: IETF

Specification Document(s):
: This specification, {{backchannel-request}}

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

## OAuth Extensions Error Registry

This specification requests registration of the following value in the IANA "OAuth Extensions Error" registry established by {{RFC6749}}:

Error Name:
: `deferral_required`

Error Usage Location:
: Authorization endpoint response, token endpoint response

Related Protocol Extension:
: OAuth 2.0 Software Statement Issuance

Change Controller:
: IETF

Specification Document(s):
: This specification, {{deferred-authorization-response}} and {{backchannel-request}}

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

## Why Use the Authorization Endpoint

Issuance commonly requires browser-mediated interaction with a user or administrator. The authorization endpoint supplies established redirect URI, request correlation, PKCE, and issuer-identification mechanisms for that interaction. Returning either an intermediate code or a sender-constrained deferral code keeps the software statement itself out of the browser and delivers it over a direct TLS-protected connection. When no in-band interaction is possible, the backchannel flow in {{backchannel-request}} substitutes direct client authentication, optional pre-authorization through an initial access token, and out-of-band approval for the front channel.

## Why Not the Device Authorization Grant

The device authorization grant {{RFC8628}} serves an interactive user who is co-present with a constrained device and can transcribe a short code into a secondary browser. Software statement issuance for non-interactive software has a different shape: the approving party is typically an administrator who is not co-present, acts through an out-of-band workflow, and may respond long after the request. The backchannel flow combined with deferred processing covers that case without a verification URI or user code, while the redirect flow covers issuance that does require interactive authentication or consent.

## Why the Backchannel Flow Uses the Token Endpoint

Two registration-endpoint designs were considered for the backchannel flow. The first extends the {{RFC7591}} registration endpoint with a mode that validates client metadata but returns a software statement instead of creating a client. It offers a JSON-native request body and a response member that mirrors the `software_statement` member of a later registration request. It also inverts the registration contract by deliberately creating no client, provides no standard client authentication beyond initial access tokens, and has no deferral mechanism: {{DTR}} defers token endpoint responses, so pending issuance would require either a second, bespoke polling protocol or a normative dependency on the pending-registration machinery of {{APPROVAL-DCR}}.

The second design follows the split used by {{CIBA}}: a dedicated initiation endpoint accepts a JSON request and, when issuance is pending, returns a deferral code that the client redeems through the same {{DTR}} polling grant used elsewhere in this specification. That design preserves a single deferral model and could be defined later as a compatible extension. It was not adopted because it introduces a new endpoint, new capability metadata, and a new authentication surface, while the token endpoint already provides the client authentication methods, sender constraint, and deferral behavior the flow requires.

The one capability unique to the registration endpoint model, pre-authorization of requests through an out-of-band credential, is available to the backchannel flow through the `initial_access_token` parameter ({{backchannel-request}}).

## Why One Grant Type, and Why Redemption Uses `authorization_code`

Only the backchannel request needs a new grant type: it initiates issuance work at the token endpoint, which no existing grant expresses. `urn:ietf:params:oauth:grant-type:software-statement` therefore means exactly one thing.

Redemption of an intermediate code reuses `authorization_code`, following the OpenID Connect precedent in which the request that produced a code, not the grant type, determines what its redemption yields. The intermediate code is bound to `response_type=software_statement`, so redeeming it returns the software statement token response and never an access token, and an authorization code issued for any other response type cannot be redeemed for a statement ({{code-redemption}}). This split also removes two failure modes at once: a malformed redemption missing `code` is an ordinary `authorization_code` error and cannot initiate new issuance work, and no special-purpose grant exists whose behavior changes based on which parameters happen to be present.

## Why Use Direct Authorization Endpoint Deferral

Issuing an intermediate code before approval completes would imply that the software statement request had reached its synchronous completion point. It would also require the client to make a token request solely to learn that processing was pending.

The direct preceding-endpoint extension of {{DTR}} lets the authorization server return a deferral code instead. PKCE is verified on the first poll, and client authentication or DPoP protects subsequent polls. This gives the authorization server time to complete approval without prematurely issuing a code while preserving the same successful software statement response for both paths.

## Why the Response Uses `access_token`

OAuth token endpoint infrastructure expects a token response to contain `access_token` and `token_type`, and {{DTR}} returns the originating grant's successful token response after polling. {{RFC8693}} establishes how the same response structure can carry a security token that is not an access token: `access_token` contains the issued artifact, `issued_token_type` identifies its representation, and `token_type=N_A` states that access-token usage semantics do not apply.

Using that convention keeps immediate and deferred responses compatible with existing token endpoint processing while unambiguously stating that a software statement grants no resource access.

## Why Reuse the `registration` Parameter

The `registration` parameter from Section 7.2.1 of {{OIDC-CORE}} is an existing authorization request parameter whose value is a JSON object of client metadata. Reuse avoids defining another container with the same wire format.

This specification deliberately narrows its semantics. The object can only select values already present in the Client ID Metadata Document; it cannot carry issuer claims, approval notes, or arbitrary request context. This restriction makes the attested input deterministic and prevents the overlay from bypassing the canonical metadata document.

# Acknowledgments

This specification builds on {{DTR}}, {{CIMD}}, {{APPROVAL-DCR}}, and {{OIDC-CORE}}. The author thanks the authors and contributors to those specifications and the OAuth working group participants whose discussions informed this work.
