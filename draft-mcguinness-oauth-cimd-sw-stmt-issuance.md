---
title: "CIMD Software Statement Issuance"
abbrev: oauth-cimd-sw-stmt-issuance
docname: draft-mcguinness-oauth-cimd-sw-stmt-issuance-latest
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
    organization: Independent
    email: public@karlmcguinness.com

normative:
  STATEMENT:
    target: https://datatracker.ietf.org/doc/draft-mcguinness-oauth-cimd-sw-stmt
    title: "CIMD Software Statement"
  RFC6749:
  RFC7521:
  RFC7523:
  RFC7591:
  RFC7636:
  RFC8414:
  RFC8693:
  RFC8707:
  RFC8725:
  RFC9207:
  RFC9396:
  RFC9449:
  RFC9700:
  STATUSLIST:
    target: https://datatracker.ietf.org/doc/draft-ietf-oauth-status-list
    title: "Token Status List"
  DTR:
    target: https://datatracker.ietf.org/doc/draft-ietf-oauth-deferred-token-response
    title: "Deferred Token Response"
  CIMD:
    target: https://datatracker.ietf.org/doc/draft-ietf-oauth-client-id-metadata-document
    title: "OAuth Client ID Metadata Document"
  OAUTH-MRT:
    target: https://openid.net/specs/oauth-v2-multiple-response-types-1_0.html
    title: "OAuth 2.0 Multiple Response Type Encoding Practices"
  FORM-POST:
    target: https://openid.net/specs/oauth-v2-form-post-response-mode-1_0.html
    title: "OAuth 2.0 Form Post Response Mode"

informative:
  REGISTRATION:
    target: https://datatracker.ietf.org/doc/draft-mcguinness-oauth-cimd-sw-stmt-registration
    title: "CIMD Software Statement Registration"
  RFC8628:
  RFC9126:
  CLIENT-INSTANCE:
    target: https://datatracker.ietf.org/doc/draft-mcguinness-oauth-client-instance-assertion
    title: "OAuth 2.0 Client Instance Assertion"
  RFC8252:
  APPROVAL-DCR:
    target: https://datatracker.ietf.org/doc/draft-dellaert-oauth-approval-based-dcr
    title: "OAuth 2.0 Approval-Based Dynamic Client Registration"
  AU-CDR:
    target: https://consumerdatastandardsaustralia.github.io/standards/
    title: "Consumer Data Standards (Australia)"
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

RFC 7591 defines how a client presents a software statement and how a registration endpoint consumes it, but not how the client obtains one. This specification defines OAuth 2.0 flows for issuing a software statement to a client identified by a Client ID Metadata Document and not registered with the authorization server.

A client holding an initial access token that authorizes issuance, or a statement it can renew, uses OAuth 2.0 Token Exchange (RFC 8693) without a redirect. Any client can instead use a redirect flow: the authorization endpoint returns a short-lived `software_statement_code`, which the client redeems through a new token endpoint grant. In both flows, a completed decision returns a statement, and a pending decision uses Deferred Token Response and polling. The statement never appears in an authorization response URL.

The client presents the statement at registration or at runtime, or publishes it for servers to pull; a companion specification defines the statement, its validation, and its consumption.

--- middle

# Introduction

Section 2.3 of {{RFC7591}} defines a software statement: a JWT asserting client metadata, presented to a registration endpoint, whose signature identifies who attested to the metadata. {{RFC7591}} standardizes its consumption, but not its issuance.

Issuance today relies on manual provisioning, deployment-specific portals, or proprietary federation processes. The UK Open Banking Directory {{UK-OPEN-BANKING}} and the Australian Consumer Data Right Register {{AU-CDR}} each built a central issuer of {{RFC7591}} software statements, yet a client needs a separate integration for each. A portal provides no interoperable protocol for submission, deferral, delivery, metadata binding, renewal, or errors; this specification defines one.

Pre-registration {{CIMD}}, pushed registration {{PUSHED-DCR}}, and approval-based registration {{APPROVAL-DCR}} each establish trust at one authorization server from client-supplied metadata. A software statement makes an issuer's decision portable: a publisher program, enterprise security function, or ecosystem operator reviews the software once, and each authorization server in the audience can rely on the signed decision under its own policy ({{what-issuance-attests}}, {{STATEMENT}}, {{beyond-pre-registration}}).

{{STATEMENT}} defines the statement, its claims, its validation, and its consumption; this document defines how a client obtains one. It introduces no new client credential or federation architecture. Portability remains bounded by configured issuer trust, typically within an ecosystem or administrative domain rather than the open web.

A client identified by its {{CIMD}} URL obtains a statement through OAuth token exchange ({{token-exchange-profile}}) or a redirect flow ({{authorization-request}}) that uses `response_type=software_statement_code` and the `urn:ietf:params:oauth:grant-type:software-statement` redemption grant.

Issuance concerns client establishment, not authorization to access a protected resource, so a software statement request cannot be combined with `scope`, `resource`, `authorization_details`, or an access-token-producing response type ({{prohibited-parameters}}).

Other mechanisms are preferable when:

* the resource owner is the approver, because the ordinary OAuth grant is the approval;
* the trust decision is local to one authorization server, where an initial access token {{RFC7591}}, pre-registration {{CIMD}}, or approval-based registration {{APPROVAL-DCR}} can establish the client directly; or
* the client cannot host a metadata document at an HTTPS URL ({{client-identity}}).

A software statement is warranted only when one or more of the following holds ({{STATEMENT}} illustrates the boundaries):

1. One approval must be honored at many authorization servers.
2. The party approving the software is not the resource owner in a transaction.
3. Approval must exist before any authorization transaction does.
4. Approval requires asynchronous review that outlives a single protocol exchange.
5. Issuance and renewal must be automated rather than performed through a portal.

## What Pre-Registration Does Not Solve {#beyond-pre-registration}

Pre-registration {{CIMD}} lets a client enroll its identifier URL before its first authorization request: the authorization server fetches and reviews the metadata and records a local decision.

Pre-registration changes when the trust decision is made, but not:

* **Who decides:** each consuming authorization server still evaluates the client's self-asserted document.
* **What transfers:** the decision remains local state and cannot be presented elsewhere.
* **What was approved:** the subject is a URL whose content can change, with no interoperable binding to the reviewed version.
* **What acceptance depends on:** later evaluation still requires dereferencing the document.

A software statement makes one decision portable and binds it to the exact content evaluated ({{metadata-snapshot}}), so a consumer can check the review against the document it retrieves, and an authorization server not staffed to review software can rely on a trusted issuer's signature instead. The two mechanisms compose.

## Relationship to Other Specifications

This specification builds on the following:

* {{RFC7591}} defines the software statement; {{STATEMENT}} profiles it for clients identified by a Client ID Metadata Document.
* {{CIMD}} defines the client identifier, canonical metadata source, pre-registration, and metadata-change handling; the metadata digest ({{metadata-snapshot}}) makes changes precisely detectable.
* {{DTR}} defines client opt-in, token endpoint deferral, polling, cancellation, and sender constraint; this specification uses it unchanged for asynchronous issuance, adding only a delivery restriction ({{deferred-processing}}).
* {{RFC8693}} defines the response convention for non-access security tokens and the exchange profiled in {{token-exchange-profile}}.

{{APPROVAL-DCR}} creates an authorization-server-specific `client_id`, and client credentials where applicable, after approval; this specification issues a portable statement for later {{RFC7591}} registration. The two compose.

{{PUSHED-DCR}} pushes registration alongside an authorization flow at the same authorization server, and such a registration can carry a statement issued under this specification. That composition is out of scope; combining issuance itself with an access-granting response type remains prohibited ({{prohibited-parameters}}).

{{CLIENT-INSTANCE}} carries per-instance identity into tokens for runtime instances behind one `client_id`; this specification carries the trust decision about the client software to the trusting authorization server. {{STATEMENT}} describes their composition.

OpenID Federation {{OPENID-FED}} conveys attested metadata through trust chains; {{STATEMENT}} compares the two trust models. Federation standardizes trust resolution, not enrollment, approval, or deferred completion, so it does not replace this document's issuance process.

A profile of the OAuth Identity Assertion Trust Framework {{TRUST-FRAMEWORK}}, which could generalize pairwise configuration by evaluating issuers against published conditions such as authority over the client's identifier namespace, is out of scope.

Approval workflow, approver identity, external approval integration, acceptance of any particular issuer, and issuer discovery are out of scope. Acceptance policy and issuer trust establishment are deployment-specific.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

This specification uses terminology defined in {{RFC6749}}, {{RFC7591}}, {{CIMD}}, and {{DTR}}. The terms Issuing Authorization Server, Trusting Authorization Server, and Metadata Digest are defined by {{STATEMENT}}.

This specification defines the following terms:

Software Statement Request:
: An authorization request with `response_type=software_statement_code`, asking the authorization server to issue a software statement ({{authorization-request}}).

Software Statement Code:
: The short-lived, single-use artifact returned by the software statement code response ({{software-statement-code-response}}) and redeemed at the token endpoint ({{software-statement-code-redemption}}).

Originating Request:
: A request that can produce a software statement: a software statement code redemption ({{software-statement-code-redemption}}) or a token exchange ({{token-exchange-profile}}). Each yields a statement, a terminal denial, or, at a deferred issuer, a deferral.

Metadata Snapshot:
: The validated canonical metadata bound to a request, as defined in {{metadata-snapshot}}.

# Protocol Overview

A client that holds an initial access token authorizing issuance, or a statement it can renew, uses token exchange at the token endpoint without a user agent ({{token-exchange-profile}}). Renewal by prior statement ({{renewal}}) requires the client itself to authenticate with a key its reviewed document carries, which a public client does not hold. A publisher's backend acting for a client is not a party this specification authenticates; issuance it authorizes on a client's behalf is out of scope.

Any client can instead initiate the redirect flow at the authorization endpoint ({{authorization-request}}) and redeem the resulting software statement code at the token endpoint. In either flow, the issuer completes the decision synchronously or defers it under {{DTR}}.

The protocol has four elements:

1. An HTTPS Client ID Metadata Document URL identifies the client, and its content supplies the canonical metadata {{CIMD}}.
2. The issuing authorization server fetches and snapshots that document, decides whether to issue, and signs the statement ({{metadata-snapshot}}).
3. The client presents the statement to consuming servers: in an {{RFC7591}} registration request ({{REGISTRATION}}), a sender-constrained runtime presentation ({{STATEMENT}}), or a delivery that renews what a server already holds ({{STATEMENT}}, {{REGISTRATION}}).
4. Trusting authorization servers in its audience apply their local acceptance policies.

The URL remains the client identity when its content changes, and one issuance decision can serve many registrations and runtime presentations.

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

At (A), the client requests a statement; the authorization server validates the request, performs any user-agent interaction or initiates an approval workflow, and returns a software statement code at (B).

At (C), the client redeems the code with its PKCE verifier. The response at (D) carries the statement or a {{DTR}} `deferral_code`, with which the client polls at (E). Errors follow {{RFC6749}} and {{DTR}}.

# Client Identification and Authentication {#client-identity}

The `client_id` in every request defined by this specification MUST be a client identifier URL conforming to {{CIMD}}. The authorization server MUST obtain and validate the corresponding Client ID Metadata Document according to {{CIMD}}.

In the redirect flow, the authorization server MUST compare the `redirect_uri` in the request with the `redirect_uris` in the Client ID Metadata Document according to {{CIMD}} and {{RFC9700}}. The authorization server MUST NOT redirect the user agent when the client identifier or redirect URI is missing or invalid. A client identifier is invalid for this purpose if its metadata document cannot be retrieved or validated, including a document rejected for duplicate member names ({{metadata-snapshot}}); the authorization server reports such a failure to the user agent.

The client authenticates to the token endpoint using the `token_endpoint_auth_method` and related key metadata in its Client ID Metadata Document. The authorization server classifies the client from that member:

* `none` (explicit) establishes a public client.
* Any other value establishes a confidential client. The authorization server MUST require exactly that method; {{CIMD}} states this rule for `private_key_jwt`, and this specification applies it to every declared method. If the authorization server does not support the declared method, it MUST reject the request rather than treat the client as public, with `unauthorized_client` at the authorization endpoint and with `invalid_client` at the token endpoint, which Section 5.2 of {{RFC6749}} assigns to an unsupported authentication method.
* An omitted value establishes neither, and any request identifying such a client MUST be rejected with `invalid_request`, returned at the authorization endpoint to the redirection URI validated against the document.

Classification uses the singular `token_endpoint_auth_method`; a list of declared supported methods is not interpreted.

Sender constraint depends on the flow:

* A public client using the redirect flow MUST include `dpop_jkt` in the authorization request and MUST present a DPoP proof signed with the corresponding key at redemption and on every polling request.
* A public client using the token exchange profile ({{token-exchange-profile}}) MUST instead include a DPoP proof on the exchange request. The authorization server MUST bind any resulting deferral state to that proof's key, and the client MUST use the same key on every polling request.
* A confidential client MAY use DPoP in addition to its client authentication method, and does so wherever {{authorization-request}} requires `dpop_jkt`.

All uses of DPoP MUST follow {{RFC9449}} and {{DTR}}.

## Metadata Snapshot {#metadata-snapshot}

Client metadata documents can change while a request is pending. Before returning a software statement code or a deferral code, the authorization server MUST bind it to the validated canonical metadata. The bound values constitute the metadata snapshot for the request.

The authorization server MAY retrieve the document again before issuing the software statement. If it does so and the digest differs, it MUST either re-evaluate the request under the new document, binding it as the snapshot, or reject the request. This applies even if the change touches no member its policy examines, since a statement over a superseded snapshot fails at registration ({{REGISTRATION}}). It MUST NOT silently combine values from different document versions.

An approval recorded against a superseded snapshot does not carry forward without a fresh issuance-policy decision. Replacing the snapshot does not alter the sender-constraint context fixed for a deferral at origination ({{deferred-processing}}). If a replacement snapshot no longer authorizes the key material behind that context, for example because the client-authentication key is absent from the new document, the authorization server MUST invalidate the deferral; the client makes a new request under its current keys.

The metadata digest, defined in {{STATEMENT}}, is computed over the retrieved representation of the canonical document. For this specification, equal digests identify the same document, and a changed digest marks a new trust state for the same client identifier and is the signal the re-evaluation rule above acts on. The digest also supplies the `cimd_digest` claim ({{STATEMENT}}) and audit guidance ({{security-considerations}}). The authorization server MUST reject duplicate object member names, because parsers can interpret them differently despite an identical digest.

Byte identity detects serialization-only changes by design. A digest mismatch is fatal at registration ({{REGISTRATION}}) and an input to policy at runtime ({{STATEMENT}}). A document rendered dynamically or served through content negotiation can change digest without any metadata change, and a statement over it then stops matching. This is a consequence of binding to octets, not a requirement on what a publisher may serve.

An issuance source SHOULD publish keys by reference through `jwks_uri` rather than inline through `jwks`. Rotation behind a stable URI leaves the document and digest unchanged; inline rotation changes both and requires a new statement. Because the digest of a document naming `jwks_uri` binds only the key location, key-host compromise is equally invisible to it, and where that key is the runtime proof under {{STATEMENT}}, the compromise also substitutes the presenter; {{STATEMENT}} weighs the trade-off, and an issuer serving theft-sensitive deployments attests `jwks` inline instead.

{{CIMD}} permits a document to carry a `software_statement` member, and this specification neither requires nor forbids it. An issuing authorization server evaluates the document as served and MUST NOT refuse a document because it carries the member. A consumer ignores any statement embedded in the reviewed document, and refusing to issue over such a document would leave a client that published its statement unable to renew it for the life of the identifier.

# Token Exchange Profile {#token-exchange-profile}

A client that already holds a token carrying issuance authority MAY exchange that token for a statement using OAuth 2.0 Token Exchange {{RFC8693}} instead of using the redirect flow ({{authorization-request}}). Existing ecosystems use this pattern ({{UK-OPEN-BANKING}}, {{AU-CDR}}). An authorization server advertises support, and the subject token types it accepts, through `software_statement_subject_token_types_supported` ({{authorization-server-metadata}}).

The client sends a token exchange request as defined in Section 2.1 of {{RFC8693}}, with the following parameters:

`grant_type`:
: REQUIRED. The value MUST be `urn:ietf:params:oauth:grant-type:token-exchange`.

`requested_token_type`:
: REQUIRED. The value MUST be `urn:ietf:params:oauth:token-type:software-statement`.

`subject_token` and `subject_token_type`:
: REQUIRED. One of the following:

  * **First issuance.** An initial access token, presented with a `subject_token_type` of `urn:ietf:params:oauth:token-type:access_token`: an authorization credential issued out of band by this authorization server that pre-authorizes software statement issuance, analogous to the initial access token of {{RFC7591}}. The credential MUST be bound to the request's `client_id`, exactly or through a client identifier namespace, as below. This profile standardizes the exchange, not the credential, which is deployment-defined.
  * **Renewal.** A software statement this authorization server previously issued for the same `sub`, presented with a `subject_token_type` of `urn:ietf:params:oauth:token-type:software-statement` ({{renewal}}).

`client_id`:
: REQUIRED. The client identifier URL described in {{client-identity}}.

`audience`:
: OPTIONAL. The requested audience, with the syntax, validation, and narrowing rules of the `audience` authorization request parameter ({{authorization-request}}). On renewal ({{renewal}}), the subject statement's `aud` stands in for the requested audience where the request carries none, and bounds it where it does, so a replacement is never broader than the statement it replaces.

`completion_mode`:
: As defined for software statement code redemption ({{software-statement-code-redemption}}).

The request MUST NOT contain `actor_token` or `actor_token_type`, nor the `scope`, `resource`, or `authorization_details` parameters prohibited by {{prohibited-parameters}}.

The client authenticates as described in {{client-identity}}, using an assertion-based method such as `private_key_jwt` {{RFC7521}} {{RFC7523}} where its document specifies one. The sender-constraint rules there apply to the exchange, and {{deferred-processing}} governs polling for any resulting deferral.

An initial access token presented under this profile MUST be:

* time limited;
* limited to the issuing authorization server;
* bound to an exact client identifier URL or an explicitly authorized client identifier namespace; and
* of at least 128 bits of entropy, when opaque.

The initial access token is subject to the following:

* A reusable one MUST be sender-constrained to a client key, for example through DPoP or mTLS, and the authorization server MUST verify that binding against the key the exchange request proves; a bearer one MUST be single-use.
* It SHOULD be integrity protected and kept confidential in transit and at rest, and MAY further restrict audiences or metadata.
* The authorization server MUST enforce every restriction the credential carries and MUST prevent replay beyond its permitted number of uses.

A DPoP proof on the exchange constrains any resulting deferral but neither authenticates the presenter nor protects the subject token (Section 3 of {{RFC9449}}), so the initial access token needs its own sender constraint or single-use restriction.

A use is consumed when the authorization server commits to an outcome: issuing a statement, creating a deferral, or denying issuance. Concurrent presentations of a single-use credential MUST NOT both be committed. Once a deferral exists, the client recovers the outcome by polling with the deferral code, not by presenting the credential again.

The authorization server MUST validate the subject token before retrieving client-controlled metadata or enqueueing any processing. A subject token MUST result in `invalid_request` if it:

* is invalid or revoked;
* has expired, unless it is a prior software statement accepted under {{renewal}}; or
* does not authorize issuance for the presented `client_id`.

Section 2.2.2 of {{RFC8693}} requires this error for a subject token that is invalid or unacceptable under policy. An unacceptable requested audience results in `invalid_target` {{RFC8693}}.

The authorization server MUST obtain and validate the Client ID Metadata Document and MUST bind a fresh metadata snapshot ({{metadata-snapshot}}) and the requested audience before returning either the software statement or a deferral code. The statement's claims derive from that snapshot. An authorization server that recorded the metadata digest of a prior issuance for this `client_id` MAY compare it against the fresh snapshot and treat a change as an input to issuance policy.

Issuance policy determines whether an initial access token authorizes only the request or issuance itself. The exchange results in:

* a software statement token response ({{software-statement-response}}) on success;
* a {{DTR}} deferred token response from a deferred issuer, followed by polling, when processing cannot complete immediately ({{deferred-processing}}); or
* the terminal denial of {{terminal-denial}} when issuance policy denies the exchange.

## Renewal {#renewal}

A client renews by presenting its current or most recent software statement as the subject token. The authorization server MUST verify that:

* it issued the statement;
* the statement's `sub` equals the request's `client_id`; and
* the client authenticated with a key carried both by the document the statement's `cimd_digest` names and by the current document.

That authentication is the holder binding: a statement is otherwise a bearer artifact, and without it whoever held a copy could renew. To check the reviewed document's keys, an issuer offering renewal retains the octets of each document it issues a statement over.

A key removed or added since review cannot renew; a client rotating keys inline renews while its document carries both the old and the new key. For a document naming `jwks_uri`, both carry the same location, so the binding is to whatever that location serves ({{STATEMENT}}). A public client holds no such key and cannot renew this way; it obtains a replacement through the redirect flow ({{authorization-request}}).

A replacement carries the subject statement's `aud_tenant` while the decision is confined to that tenant. An issuer whose tenants are independently trusted reviewing authorities issues under a distinct `iss` for each ({{STATEMENT}}), so a replacement is matched to the statement it replaces by `iss` and `sub`.

An issuer that publishes status MUST NOT accept as subject token a statement whose own published status is other than `VALID`. Renewing a statement it has withdrawn would reissue the decision that withdrawal ended.

The authorization server MAY accept a statement that has expired, and SHOULD bound how long after expiry it will do so, since a client absent for an extended period is asking to be re-established rather than renewed.

Where the current document's digest equals the subject statement's `cimd_digest`, whether renewal requires fresh review is issuer policy. Where the digests differ, the issuer MUST apply the decision it would apply to a first issuance for that document, and MUST NOT renew on the strength of the prior statement alone: a changed document is a new trust state ({{metadata-snapshot}}), and otherwise whoever can change the document could obtain the issuer's signature over an unreviewed change.

# Software Statement Authorization Request {#authorization-request}

The client sends an authorization request as described in Section 4.1.1 of {{RFC6749}}, with the following parameters:

`response_type`:
: REQUIRED. The value MUST be `software_statement_code`.

`client_id`:
: REQUIRED. The client identifier URL described in {{client-identity}}.

`redirect_uri`:
: REQUIRED. The value MUST exactly match one of the redirection URIs in the Client ID Metadata Document, subject to the redirect URI rules of {{CIMD}}. A public client MUST use an HTTPS redirection URI and MUST NOT use a private-use URI scheme or a loopback interface redirection URI in a software statement request. A confidential client MAY use a loopback or private-use redirection URI ({{authorization-response-security}}).

`state`:
: REQUIRED. An opaque value used by the client to bind the authorization response to its request. The authorization server MUST return the exact value in the authorization response.

`code_challenge`:
: REQUIRED. A PKCE challenge as defined by {{RFC7636}}.

`code_challenge_method`:
: REQUIRED. The value MUST be `S256`.

`response_mode`:
: OPTIONAL. The response mode, as defined by {{OAUTH-MRT}}. The default for `response_type=software_statement_code` is `query`; {{software-statement-code-response}} gives response mode considerations.

`audience`:
: OPTIONAL. A target service at which the client intends to use the statement, with the semantics of Section 2.1 of {{RFC8693}}; the parameter can be repeated to request several. Each value MUST be an authorization server issuer identifier as defined by {{RFC8414}}; values MUST NOT be repeated, and order is insignificant.

The authorization server selects the final audience according to policy. It MUST NOT place in the statement's `aud` claim any value the request did not carry, except that a renewal request carrying no `audience` counts as carrying the subject statement's `aud` ({{token-exchange-profile}}); an issuer narrows a requested audience and never widens it. Where no requested audience is acceptable, the authorization server MUST reject the request with `invalid_target` {{RFC8693}}, which {{RFC8707}} uses at the authorization endpoint for a missing or invalid target. An authorization server whose policy requires a restricted audience rejects a request carrying none with the same error. These semantics apply only to software statement requests and do not affect proprietary uses of `audience` for access-token targeting.

`completion_mode`:
: OPTIONAL. A value that includes `deferred`, sent as the advance hint {{DTR}} defines for an endpoint preceding a token request, so that the authorization server can choose a review path suited to out-of-band completion before it begins work. It does not replace the opt-in required at redemption ({{deferred-processing}}).

`dpop_jkt`:
: REQUIRED for a public client, and for a confidential client whose `redirect_uri` is a loopback or private-use URI; OPTIONAL otherwise. A declared confidential method proves key possession, not that the key is absent from distributed software, which is why the confidential-client exception for loopback and private-use redirection URIs carries this condition. The parameter has the semantics of Section 10 of {{RFC9449}}. When present, its value MUST be associated with the resulting software statement code and with any deferral state derived from its redemption.

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

An authorization server MUST reject a request containing any of these parameters with `invalid_request`. The same prohibition and error apply to a software statement code redemption ({{software-statement-code-redemption}}). The `audience` parameter ({{authorization-request}}) is not prohibited: it describes the requested artifact rather than requesting access.

Hybrid response types that combine `software_statement_code` with `code`, `token`, `id_token`, or any other response type are not defined. An authorization server MUST reject such a request with `unsupported_response_type`.

# Authorization Response {#authorization-response}

After validating the request and performing any immediate interaction, the authorization server returns the software statement code response or an error. The authorization server MUST NOT place the software statement or approval-sensitive information in any response from the authorization endpoint.

A denial is never signaled in the authorization response: the authorization server returns a software statement code whether the issuance decision is complete, pending, or already a denial, and delivers any denial at redemption ({{terminal-denial}}). A client therefore needs no redirect-side `access_denied` handler for issuance outcomes.

## Software Statement Code Response {#software-statement-code-response}

The authorization server returns the following parameters to the client's redirection endpoint through the user agent, using the selected response mode:

`software_statement_code`:
: REQUIRED. A short-lived, single-use artifact redeemed at the token endpoint ({{software-statement-code-redemption}}). It MUST be bound to the client identifier, redirect URI, PKCE challenge, metadata snapshot, requested audience, and, when `dpop_jkt` was present, that JWK thumbprint. The value MUST:

  * contain at least 128 bits of entropy from a cryptographically secure random source;
  * be opaque to the client;
  * expire shortly after issuance; and
  * not be accepted more than once.

`state`:
: REQUIRED. The exact value received in the authorization request.

`iss`:
: REQUIRED. The authorization server issuer identification parameter defined by {{RFC9207}}.

A software statement code is not an authorization code and MUST NOT be redeemable as one.

The fragment response mode SHOULD NOT be used, because scripts at the redirection endpoint can access it. A client MAY request `form_post` {{FORM-POST}} to keep the code out of URLs, browser history, and Referer headers.

The following is an example of a software statement code response using the default `query` response mode (line breaks are for display purposes only):

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
: REQUIRED when the authorization server advertises `deferred_token_response_supported` ({{authorization-server-metadata}}); otherwise not used, and a synchronous issuer ignores it ({{deferred-processing}}). When present, the value MUST include `deferred`. A deferral-capable issuer rejects a redemption that omits it with `invalid_request`. A client that cannot poll therefore cannot redeem at such an issuer, which cannot promise a synchronous answer.

The request MUST NOT contain `audience`, which was bound at the authorization endpoint; a request containing it is rejected with `invalid_request`. The client authenticates according to {{client-identity}}. When `dpop_jkt` was included in the authorization request, the client MUST send a DPoP proof for the token endpoint using the same key.

The authorization server MUST validate the software statement code and all of its bindings before processing the request. An invalid, expired, previously used, or incorrectly bound code MUST result in an `invalid_grant` error; a PKCE or DPoP binding failure is handled according to {{RFC7636}} or {{RFC9449}}, respectively.

A redemption attempt consumes the software statement code whenever the presented code value is valid, including when its PKCE, DPoP, or client-authentication bindings fail ({{authorization-response-security}}). A DPoP nonce challenge {{RFC9449}} is not a binding failure: an authorization server requiring a nonce issues the `use_dpop_nonce` challenge before evaluating the code, which remains unconsumed.

A previously consumed code presented again MUST be rejected, and the authorization server SHOULD revoke any deferral derived from it ({{RFC9700}}). Because the code travels in a URL, anyone who observed it can trigger that revocation and cancel a decision that may take days. An authorization server MAY limit revocation to replays that pass the PKCE check, which only the original requester can produce; a client that wants the code kept out of URLs uses `form_post` {{FORM-POST}}.

For a valid, unconsumed code, the result depends on the issuance decision:

* **Approved:** the authorization server returns the software statement token response ({{software-statement-response}}).
* **Denied:** it returns the terminal denial of {{terminal-denial}}.
* **Pending:** it returns the deferred token response of {{DTR}}, binding the deferral to the code's metadata snapshot and audience in addition to the bindings {{DTR}} requires; the client then polls ({{deferred-processing}}).

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

# Deferred Processing {#deferred-processing}

An authorization server MAY answer an originating request, a software statement code redemption ({{software-statement-code-redemption}}) or a token exchange ({{token-exchange-profile}}), with a deferred token response {{DTR}} instead of a statement.

An authorization server that does so implements {{DTR}} and advertises `deferred_token_response_supported` ({{authorization-server-metadata}}); one that does not, does neither. Clients and authorization servers MUST follow {{DTR}} for every aspect of the deferral: the `completion_mode` opt-in, the polling grant and its parameters, pending, denied, and expired behavior, polling rate, sender constraint across polls, and cancellation. This document adds three constraints and one recommendation:

* A client redeeming at a deferral-capable issuer accepts deferral by including `completion_mode`; the parameter records that acceptance rather than negotiating it.
* A deferral created under this specification MUST be delivered by polling. A client MUST NOT send the `client_notification_token` parameter of {{DTR}}; an authorization server rejects a request carrying it with `invalid_request` and MUST NOT deliver a callback, whatever the client's metadata says. Callback delivery is a deferred capability ({{design-rationale}}).
* A successful polling response is the software statement token response of {{software-statement-response}}, not an access token response.
* Issuers SHOULD set deferral code lifetimes that reflect their actual approval latency, which for a review involving human judgment can be hours or days.

{{DTR}} specifies the binding between a deferral and the requests that poll it; this document neither relaxes nor restates it.

The following is a non-normative example of a first polling request for a deferral created by a token exchange, from a confidential client authenticating with the same `private_key_jwt` method as the exchange:

~~~ http
POST /token HTTP/1.1
Host: issuer.example
Content-Type: application/x-www-form-urlencoded

grant_type=urn%3Aietf%3Aparams%3Aoauth%3Agrant-type%3Adeferred
&deferral_code=8xLOxBtZp8
&client_id=https%3A%2F%2Fclient.example.org%2Fmetadata.json
&client_assertion_type=urn%3Aietf%3Aparams%3Aoauth%3A
client-assertion-type%3Ajwt-bearer
&client_assertion=eyJhbGciOiJFUzI1NiIsImtpZCI6ImNsaWVudC0xIn0...
~~~

# Software Statement Token Response {#software-statement-response}

A successful response has HTTP status code 200, a media type of `application/json`, and the following members:

`access_token`:
: REQUIRED. The software statement issued by the authorization server. It MUST conform to {{STATEMENT}}: `sub` is the client identifier URL of the request, `cimd_digest` is the digest of the bound metadata snapshot, `aud` is the selected audience where one is restricted, and the statement carries no claim registered as client metadata. An issuer that publishes status carries the `status` claim locating this statement in its Status List Token ({{status-publication}}).

`issued_token_type`:
: REQUIRED. The value MUST be `urn:ietf:params:oauth:token-type:software-statement`.

`token_type`:
: REQUIRED. The value MUST be `N_A`, indicating that an OAuth access token type does not apply.

`expires_in`:
: RECOMMENDED. The remaining lifetime of the software statement in seconds. If present, it MUST be consistent with the statement's `exp` claim.

The response MUST NOT contain `refresh_token` or `scope`. The authorization server MUST include `Cache-Control: no-store`; it SHOULD also include `Pragma: no-cache`.

DPoP under this specification binds requests and deferral state, not the issued artifact; {{DTR}}'s requirement that a final access token inherit the originating DPoP binding does not apply, since no access token is issued. {{STATEMENT}} describes the statement's replay and theft properties.

The following is an example of a software statement token response:

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

The client uses the issued statement in an {{RFC7591}} registration request ({{REGISTRATION}}), by runtime presentation, or by publishing it at its `software_statements_uri` for servers to pull ({{STATEMENT}}).

The `access_token` member is a security-token container ({{RFC8693}}), not an OAuth access token: the software statement is consumed only as a software statement ({{STATEMENT}}), MUST NOT be attached to a request as an `Authorization: Bearer` credential, and is not subject to refresh. Implementations that cache issued tokens by type SHOULD key this artifact on its `issued_token_type` so that generic access-token handling does not apply to it, and SHOULD treat it as a sensitive credential in logs.

A client replaces an expiring or expired statement by renewal ({{renewal}}), by a new software statement request, or, if it holds an initial access token, through the token exchange profile ({{token-exchange-profile}}); issuer policy determines whether a replacement requires new approval. {{STATEMENT}} and {{REGISTRATION}} define how a replacement is delivered to a trusting authorization server, and {{STATEMENT}} orders replacements by `iat`.

## Terminal Denial {#terminal-denial}

When the authorization server decides not to issue the requested software statement, it returns a token error response as defined in Section 5.2 of {{RFC6749}} with the error code `access_denied` and HTTP status code 400, and MUST include the `Cache-Control: no-store` response header field. This applies to both originating requests, a software statement code redemption ({{software-statement-code-redemption}}) and a token exchange ({{token-exchange-profile}}), whether the decision is already complete when the request arrives or completes during deferred processing; in the latter case the denial is returned in response to a polling request ({{deferred-processing}}).

The denial is terminal for the request. A deferral resolves to the denied state, and subsequent polling requests return the same `access_denied` response for the remainder of the deferral code's lifetime, as {{DTR}} requires for a request resolved with an error. A denial does not preclude a later issuance request; whether to accept one is issuance policy.

# Status Publication {#status-publication}

An issuing authorization server that ends decisions before their expiry publishes statement status as {{STATUSLIST}} defines and carries the `status` claim in the statements it issues ({{STATEMENT}}). The statement records what was decided; the status list records whether that decision still stands.

An issuer that publishes status:

* MUST publish it for every statement it issues under a given `iss` from the point it begins publishing, not for a subset, so that a consumer can read an absent claim as meaning the issuer publishes no status. Statements issued before that point carry no claim, so the inference holds once those have expired;
* MUST assign each statement its own index and MUST NOT reuse an index across statements. Withdrawing a statement then affects no other, including a replacement obtained through {{renewal}}: withdrawing a superseded statement does not withdraw its replacement, and withdrawing a replacement does not restore its predecessor;
* MUST sign the Status List Token with a key published at the `jwks_uri` of its authorization server metadata {{RFC8414}}, never with a statement signing key ({{STATEMENT}});
* MUST include `exp` in every Status List Token it publishes, so that an older token cannot stand in for a newer one indefinitely; and
* MUST give each Status List Token it publishes at a given list URI an `iat` later than that of any token it published there before, so that a consumer can tell the newer of two apart.

An issuer that issues a replacement narrower than the statement it replaces SHOULD withdraw the earlier statement. A replacement is narrower when it:

* is over a document that no longer carries a key, redirection URI, or scope the earlier document carried;
* has a narrower `aud`, `aud_tenant`, or `consumable_at`; or
* has a shorter remaining lifetime.

What was removed may be why the replacement was issued; a consumer that has not yet seen the replacement, or at which it does not validate, would otherwise accept the earlier statement until it expires.

Status does not replace lifetime. A consumer is not obliged to resolve status, so an issuer chooses `exp` assuming none does; status shortens a decision but does not bound it.

# Authorization Server Metadata {#authorization-server-metadata}

The issuing authorization server is a role, not necessarily a general-purpose OAuth deployment: an issuer offering only token exchange can consist of a token endpoint, an authorization server metadata document, and signing keys, with no authorization endpoint, access tokens, or protected resources.

An authorization server that issues software statements under this specification advertises `true` for `client_id_metadata_document_supported` ({{CIMD}}), publishes `software_statement_signing_alg_values_supported` (below), and publishes its statement signing keys only at `software_statement_jwks_uri` ({{STATEMENT}}).

An issuer that may defer a request ({{deferred-processing}}) advertises `true` for `deferred_token_response_supported` ({{DTR}}) and, because deferrals are cancellable, lists `urn:ietf:params:oauth:token-type:deferral-code` in `revocation_endpoint_token_type_values_supported`, as {{DTR}} requires. A synchronous issuer advertises neither and need not implement {{DTR}}.

An authorization server supporting the redirect flow advertises:

* `urn:ietf:params:oauth:grant-type:software-statement` in `grant_types_supported`, the grant through which software statement codes are redeemed;
* `software_statement_code` in `response_types_supported`; and
* `true` for `authorization_response_iss_parameter_supported`, as defined by {{RFC9207}}.

An authorization server supporting the token exchange profile ({{token-exchange-profile}}) advertises `urn:ietf:params:oauth:grant-type:token-exchange` in `grant_types_supported` and publishes `software_statement_subject_token_types_supported` (below); general token exchange support does not by itself imply support for this profile. An implementation supporting only that profile advertises neither the software statement grant nor the `software_statement_code` response type.

A client that requires deferral MUST NOT send a request to an authorization server that does not advertise `deferred_token_response_supported`, since such an issuer answers with a statement or a terminal denial, never a deferral.

This specification defines the following additional authorization server metadata members:

`software_statement_signing_alg_values_supported`:
: REQUIRED for an authorization server that issues software statements under this specification. A JSON array containing the asymmetric JWS `alg` values that the authorization server can use to sign software statements. The array MUST NOT contain `none` or a symmetric algorithm. This member describes the issuing role; an authorization server that only accepts software statements does not publish it.

`software_statement_subject_token_types_supported`:
: REQUIRED for an authorization server that supports the token exchange profile ({{token-exchange-profile}}), and absent otherwise. A JSON array of the `subject_token_type` values the authorization server accepts when `requested_token_type` is `urn:ietf:params:oauth:token-type:software-statement`. Publishing this member signals support for the profile; listing `urn:ietf:params:oauth:token-type:software-statement` among its values advertises renewal by prior statement ({{renewal}}).

# Security Considerations {#security-considerations}

## Client Establishment Is Not an Access Grant

A software statement attests client metadata; it grants no resource access or consent on behalf of the software's users. Each user still authorizes access through the established client.

Authorization servers MUST enforce the prohibited-parameter and response-type rules in {{prohibited-parameters}}. A software statement returned in the `access_token` member ({{software-statement-response}}) MUST NOT be accepted as an access token at a protected resource.

When an approval interface is shown, it SHOULD clearly describe that the decision concerns attestation to client metadata. It MUST NOT imply that the approver is granting the client access to resources.

An erroneous approval affects every authorization server in the statement's audience until expiry. The approval interface therefore SHOULD present:

* the client identifier URL, with its origin shown as the client's identity and display metadata such as `client_name` and `logo_uri` marked as asserted by the client, since a look-alike document can carry another vendor's name;
* the document content it will vouch for, identified by its digest;
* the audience the issuer intends to place in the statement;
* the tenant the decision is confined to, where the statement will carry `aud_tenant` ({{STATEMENT}}); and
* the intended lifetime.

The interface SHOULD make narrowing visible when the client requested a different or broader audience. A document naming instance-attestation authorities ({{CLIENT-INSTANCE}}) endorses those authorities for the software under review and deserves particular scrutiny.

## What Issuance Attests {#what-issuance-attests}

A software statement is a signed, attributable claim by a named issuer about software, bounded by that issuer's process; it is not proof that its contents are true. It means only that the issuer evaluated the exact document content captured in the metadata snapshot ({{metadata-snapshot}}), under its issuance policy, at the time recorded in `iat`, and decided to vouch for it to the named audience. It composes with, rather than replaces, the client attestation and instance assertion that attest a presenter ({{STATEMENT}}).

The client authors the metadata document, so issuance does not make every value an independently verified fact. An issuer SHOULD corroborate security-relevant metadata through evidence beyond the document itself; verification depth is part of the trust relationship. A trusting authorization server can conclude only what the issuer decided, and local client-establishment policy determines what that decision is worth.

## Client Metadata Retrieval

Fetching a Client ID Metadata Document and resources referenced by it exposes the authorization server to server-side request forgery, resource exhaustion, malicious content, and client impersonation risks. The considerations of {{CIMD}} on validation, address filtering, response-size limits, redirect handling, caching, logo handling, and domain trust apply.

The metadata snapshot ({{metadata-snapshot}}) prevents a time-of-check/time-of-use change from altering what was reviewed. Authorization servers SHOULD record the metadata digest and retain the exact retrieved octets of the approved document for audit purposes; a re-serialized copy cannot reproduce the digest.

## Authorization Response Security {#authorization-response-security}

The software statement is a signed credential and can contain sensitive deployment information, which is why it is never returned in an authorization response ({{authorization-response}}). The software statement code response applies exact redirect URI matching, PKCE with `S256`, and the other applicable authorization response protections of {{RFC9700}}.

Before redeeming a software statement code, the client MUST verify `state` and MUST validate the authorization response `iss` parameter according to {{RFC9207}}. PKCE with `S256` provides the cross-site request forgery protection described in {{RFC9700}}; `state` also correlates concurrent requests with their responses.

A public client presents no client authentication, so its association with the software depends on delivery to a metadata-listed redirection endpoint. Another application can claim a private-use or loopback endpoint ({{RFC8252}}); PKCE and DPoP bind the code to the initiator but cannot stop an attacker from initiating under another party's `client_id`. Only HTTPS demonstrates control of the publisher's origin, which is why {{authorization-request}} requires it of a public client. A native public client can host such an endpoint or use the redirect-free token exchange profile ({{token-exchange-profile}}).

A confidential client's authentication protects redemption, which is why {{authorization-request}} permits it a loopback or private-use redirection URI. An authorization server SHOULD additionally relate the redirection URI's origin to the client identifier URL or the metadata document's `client_uri` according to policy. Endpoint or key control informs issuance policy but does not determine issuance.

A software statement code in a URL is visible to browser history, referrer fields, logs, and other observers. It is short lived, single use, and unredeemable without the PKCE verifier and, where one was bound, the `dpop_jkt` key; {{software-statement-code-response}} describes `form_post` for deployments sensitive to URL disclosure. Deferral codes travel only over a direct TLS connection and are protected by sender-constrained polling and cancellation ({{deferred-processing}}).

## Token Exchange Considerations {#te-considerations}

A token exchange reaches the token endpoint without prior user-agent interaction. Validating the subject token before metadata retrieval or enqueueing ({{token-exchange-profile}}) limits resource consumption by unauthorized requesters. Authorization servers SHOULD still rate-limit these exchanges, cache successful retrieval results and back off after failures rather than caching them, which {{CIMD}} forbids, and bound pending deferrals per client identifier and requester. The redirect flow likewise reaches the approval queue before any client-authenticated step; the same rate limits and pending-approval bounds SHOULD apply per client identifier there.

A subject token is an authorization credential, not a client identifier or a substitute for client authentication where the Client ID Metadata Document establishes a method. It appears in a form body, so any component recording request bodies can expose it. Authorization servers MUST exclude subject tokens from logs, traces, error messages, and audit records; clients and authorization servers MUST protect the credential as a bearer credential unless its format provides proof of possession. The binding, lifetime, entropy, and replay requirements of {{token-exchange-profile}} limit disclosure impact.

No response parameter transits a browser, but neither is there in-band evidence of user participation. An authorization server MUST NOT treat a token exchange as implying prior user consent and MUST apply the same issuance and approval policy as for the redirect flow.

## Renewal by Prior Statement

A statement is a bearer artifact, so anyone holding a stolen copy could renew it. The holder binding of {{renewal}} prevents this: the client authenticates with a key carried by both the reviewed and the current document. Where both documents carry their keys inline, a party able to change the current document cannot supply such a key; where they name a `jwks_uri`, whoever controls that location can, which is why a changed document receives the decision a first issuance receives. Binding renewal to that key also spares automated renewal a long-lived reusable initial access token, which would be a standing credential to mint statements. Without a bound on how long after expiry a statement is accepted ({{renewal}}), a client absent long enough for its review to be meaningless could renew rather than be re-established.

## Signing Keys and Algorithms

Compromise of a software statement signing key lets an attacker mint statements for every audience that trusts that key. Keeping those keys at `software_statement_jwks_uri`, apart from every key that signs anything else ({{STATEMENT}}), confines that power to the keys that need it.

* Issuers SHOULD protect signing keys according to the scope of their trust relationships and support controlled key rotation.
* Issuers SHOULD prefer signature algorithms with modern security properties, such as `PS256`, `ES256`, or `EdDSA`, over RSASSA-PKCS1-v1_5 (`RS256`), and MUST follow {{RFC8725}} when signing.

## Approver Identity and Audit

A software statement does not identify the human or system that approved issuance. Deployments that require approver attribution retain it in an authorization server audit record or define an explicit statement claim and its privacy semantics.

An approved statement is accepted at every authorization server in its audience, not only within the approver's own scope. The policy governing who may approve issuance MUST be at least as restrictive as the policy governing manual client establishment at the issuing authorization server. Approval by a party authorized only for a personal or organizational scope MUST NOT produce a statement whose audience exceeds that scope, or whose `aud_tenant` names a tenant outside it. An issuer SHOULD limit what an approver may approve to client identifiers under publisher namespaces it has enrolled, so that a look-alike document cannot reach approval.

An audit record SHOULD bind each decision, whether approval or denial, to the metadata digest ({{metadata-snapshot}}) of the document the deciding party evaluated, the policy under which the decision was made, the identity of that party, and the time of decision. A recorded denial carrying its grounds has the same audit value as a recorded approval. Portable, independently verifiable decision records are out of scope for this specification.

# Privacy Considerations

The authorization server learns the client identifier URL, the canonical metadata document, and information about the party interacting with the authorization endpoint. It SHOULD collect and retain only the information required for issuance, security monitoring, and audit obligations.

A statement names the software a reviewer evaluated, and an `aud` claim also reveals which authorization servers the client plans to establish relationships with. Omitting the claim avoids that disclosure but lets any holder of a copy register the software wherever the issuer is trusted; {{STATEMENT}} weighs the two and has an issuer name an audience by default.

Clients SHOULD NOT present statements outside their intended deployment context, and a redirect-flow client SHOULD use Pushed Authorization Requests {{RFC9126}} where the relationship is sensitive. Authorization servers SHOULD avoid logging issued statements.

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

This specification likewise requests that IANA add it as an additional reference for the `completion_mode` parameter ({{authorization-request}}). The parameter is defined by {{DTR}}, whose registration already includes authorization requests; the name, usage location, and change controller are unchanged.

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

Both error codes this specification uses at new locations are already registered. This specification requests that IANA add it as an additional reference for each, and extend the usage location of `invalid_target` to authorization error responses:

Error Name:
: `access_denied`

Existing Registration:
: {{RFC8628}}

Change Controller:
: IESG

Specification Document(s):
: {{RFC8628}}, this specification ({{terminal-denial}})

Error Name:
: `invalid_target`

Existing Registration:
: {{RFC8707}}

Error Usage Location:
: authorization error response, in addition to the locations already registered

Change Controller:
: IESG

Specification Document(s):
: {{RFC8707}}, this specification ({{authorization-request}})

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
: `software_statement_subject_token_types_supported`

Metadata Description:
: JSON array of subject token type values accepted for exchanging into software statements; signals support for the token exchange profile.

Change Controller:
: IESG

Specification Document(s):
: This specification, {{authorization-server-metadata}} and {{token-exchange-profile}}


--- back

# Design Rationale {#design-rationale}

**Why the issuer is an authorization server role.** Issuance is an OAuth interaction: it authenticates a client, applies policy, and returns a signed artifact. Reusing the role provides key publication, discovery, and client authentication without new mechanisms.

**Why the authorization endpoint.** A first-time client holds no credential at the issuer, and approval may involve a human. The redirect flow is OAuth's existing mechanism for both. A client holding an initial access token uses the token exchange profile instead.

**Why the software statement code is not an authorization code.** Redeeming an authorization code issues an access token under {{RFC6749}}. This flow issues an artifact that grants nothing; a distinct code keeps implementations that treat any code as redeemable from confusing the two.

**Why the response uses `access_token`.** {{RFC8693}} defines the container, so existing token endpoint machinery carries the artifact; the `issued_token_type` identifies what it is.

**Why not the device authorization grant.** {{RFC8628}} fits a human decision that outlives a request, and an issuer whose approval is always out of band can use it. It assumes a user co-present with a constrained device who enters a user code elsewhere. Issuance approval is made by an administrator or reviewer who does not operate the client and is often not present, and the client already has a browser. The redirect flow covers an approver reachable through that browser; the token exchange profile covers the case with no browser.

**Why not a profile of attestation-based client authentication.** A client attestation and a software statement are both signed third-party assertions presented with a key proof at the token endpoint, but their signers vouch for different things. An attester vouches for a running instance and its key, for as long as it chooses, to the server in front of it; a statement issuer vouches for reviewed software, for days, to every server that trusts it. Profiling one as the other would give the reviewed-software decision an instance-scoped trust model, or give instance attestation unwarranted portability. {{STATEMENT}} composes the two rather than merging them.

**Why not OpenID Federation trust marks.** A trust mark, the closest prior art, is a signed third-party assertion about an entity, with a defined issuer and a status endpoint. It is resolved through a federation, which supplies key discovery, policy, and delegation and requires both parties to enroll. This specification is pairwise: a consumer configures an issuer directly, with no federation above it. Ecosystems already operating a federation are better served by trust marks.

**Deferred capabilities.** This version omits several capabilities, each with an extension point: callback delivery for deferral, a canonicalized digest, and partial review, in which an issuer vouches for particular members rather than a whole document. {{STATEMENT}} defines how a client publishes an issued statement for servers to pull and names the consumption-side extensions, including endorsed instance keys.

# Acknowledgments
{:numbered="false"}

This specification builds on {{DTR}}, {{CIMD}}, and {{APPROVAL-DCR}}. The author thanks the authors and contributors to those specifications and the OAuth working group participants whose discussions informed this work.
