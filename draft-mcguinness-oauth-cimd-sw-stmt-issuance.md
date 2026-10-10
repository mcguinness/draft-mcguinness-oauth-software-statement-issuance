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
  STATUS:
    target: https://datatracker.ietf.org/doc/draft-mcguinness-oauth-cimd-sw-stmt-status
    title: "CIMD Software Statement Status"
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
  UK-OPEN-BANKING:
    target: https://openbankinguk.github.io/dcr-docs-pub/v3.3/dynamic-client-registration.html
    title: "Open Banking UK Dynamic Client Registration Specification v3.3"

--- abstract

RFC 7591 defines how a client presents a software statement and how a registration endpoint consumes it, but not how the client obtains one. This specification defines OAuth 2.0 flows for issuing a software statement to a client identified by a Client ID Metadata Document and not registered with the authorization server.

A client holding an initial access token that authorizes issuance, or a statement it can renew, uses OAuth 2.0 Token Exchange (RFC 8693) without a redirect. Any client can instead use a redirect flow: the authorization endpoint returns a short-lived `software_statement_code`, which the client redeems through a new token endpoint grant. In both flows, a completed decision returns a statement, and a pending decision uses Deferred Token Response and polling. The statement never appears in an authorization response URL.

The client presents the statement at registration or at runtime, or publishes it for servers to pull; a companion specification defines the statement, its validation, and its consumption.

--- middle

# Introduction

Section 2.3 of {{RFC7591}} defines a software statement: a JWT asserting client metadata, presented to a registration endpoint, whose signature identifies who attested to the metadata. {{RFC7591}} standardizes its consumption, but not its issuance. Issuance today relies on manual provisioning, deployment-specific portals, or proprietary processes; the UK Open Banking Directory {{UK-OPEN-BANKING}} and the Australian Consumer Data Right Register {{AU-CDR}} each built a central issuer, yet a client needs a separate integration for each.

{{STATEMENT}} defines the statement, its claims, its validation, and its consumption; this document defines how a client obtains one. A client identified by its {{CIMD}} URL obtains a statement through OAuth token exchange ({{token-exchange-profile}}) or a redirect flow ({{authorization-request}}) that uses `response_type=software_statement_code` and the `urn:ietf:params:oauth:grant-type:software-statement` redemption grant. An issuer whose review outlives a request can defer it under {{DTR}}.

A statement makes one issuer's decision portable to the authorization servers in its audience that trust the issuer ({{STATEMENT}}). Where the trust decision is local to one authorization server, an initial access token {{RFC7591}}, pre-registration {{CIMD}}, or approval-based registration {{APPROVAL-DCR}} can establish the client directly.

Approval workflow, approver identity, external approval integration, acceptance of any particular issuer, and issuer discovery are out of scope.

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

A client that holds an initial access token authorizing issuance, or a statement it can renew, uses token exchange at the token endpoint without a user agent ({{token-exchange-profile}}). Any client can instead initiate the redirect flow at the authorization endpoint ({{authorization-request}}) and redeem the resulting software statement code at the token endpoint. In either flow, the issuer completes the decision synchronously or defers it under {{DTR}}. A publisher's backend acting for a client is not a party this specification authenticates; issuance it authorizes on a client's behalf is out of scope.

The issuing authorization server fetches and snapshots the client's Client ID Metadata Document ({{metadata-snapshot}}), decides whether to issue, and signs the statement. The client carries the statement to trusting authorization servers in its audience, at registration ({{REGISTRATION}}) or at runtime ({{STATEMENT}}), each of which applies its own acceptance policy.

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

In the redirect flow, a client identifier whose metadata document cannot be retrieved or validated, including a document rejected for duplicate member names ({{metadata-snapshot}}), is invalid for the purposes of Section 4.1.2.1 of {{RFC6749}}. The authorization server MUST NOT redirect the user agent when the client identifier or redirection URI is missing or invalid.

The client authenticates to the token endpoint using the `token_endpoint_auth_method` and related key metadata in its Client ID Metadata Document. The authorization server classifies the client from that member:

* `none` (explicit) establishes a public client.
* Any other value establishes a confidential client. The authorization server MUST require exactly that method; {{CIMD}} states this rule for `private_key_jwt`, and this specification applies it to every declared method. If the authorization server does not support the declared method, it MUST reject the request rather than treat the client as public, with `unauthorized_client` at the authorization endpoint and with `invalid_client` (Section 5.2 of {{RFC6749}}) at the token endpoint.
* An omitted value establishes neither, and any request identifying such a client MUST be rejected with `invalid_request`, returned at the authorization endpoint to the redirection URI validated against the document.

Classification uses the singular `token_endpoint_auth_method`; a list of declared supported methods is not interpreted.

A public client binds its requests with DPoP {{RFC9449}}: in the redirect flow through `dpop_jkt` ({{authorization-request}}), and in the token exchange profile ({{token-exchange-profile}}) by a DPoP proof, which a public client MUST include on the exchange request. Where the request that creates a deferral carries a DPoP proof, {{DTR}} binds the deferral to that key and requires the same key on every polling request. A confidential client MAY use DPoP in addition to its client authentication method.

## Metadata Snapshot {#metadata-snapshot}

Client metadata documents can change while a request is pending. Before returning a software statement code or a deferral code, the authorization server MUST bind it to the validated canonical metadata. The bound values constitute the metadata snapshot for the request.

The authorization server MAY retrieve the document again before issuing the software statement. If it does so and the digest differs, it MUST either re-evaluate the request under the new document, binding it as the snapshot, or reject the request. This applies even if the change touches no member its policy examines, since a statement over a superseded snapshot fails at registration ({{REGISTRATION}}). It MUST NOT silently combine values from different document versions.

An approval recorded against a superseded snapshot does not carry forward without a fresh issuance-policy decision. Replacing the snapshot does not alter the sender-constraint context fixed for a deferral at origination ({{deferred-processing}}). If a replacement snapshot no longer authorizes the key material behind that context, for example because the client-authentication key is absent from the new document, the authorization server MUST invalidate the deferral; the client makes a new request under its current keys.

The digest is the metadata digest of {{STATEMENT}}, and a changed digest marks a new trust state for the same client identifier. The authorization server MUST reject duplicate object member names, because parsers can interpret them differently despite an identical digest.

A client SHOULD publish its keys in its document by reference through `jwks_uri` rather than inline through `jwks`, since inline rotation changes the digest and requires a new statement. The digest then binds only the key location; {{STATEMENT}} weighs that trade-off, and an issuer serving theft-sensitive deployments can require `jwks` inline instead.

{{CIMD}} permits a document to carry a `software_statement` member. An issuing authorization server evaluates the document as served and MUST NOT refuse a document because it carries the member; consumers ignore a statement embedded there ({{STATEMENT}}), and refusing would leave a client that published its statement unable to renew it.

# Token Exchange Profile {#token-exchange-profile}

A client that already holds a token carrying issuance authority MAY exchange that token for a statement using OAuth 2.0 Token Exchange {{RFC8693}} instead of using the redirect flow ({{authorization-request}}).

The client sends a token exchange request as defined in Section 2.1 of {{RFC8693}}, with the following parameters:

`requested_token_type`:
: REQUIRED. The value MUST be `urn:ietf:params:oauth:token-type:software-statement`.

`subject_token` and `subject_token_type`:
: REQUIRED. One of the following:

  * **First issuance.** An initial access token, presented with a `subject_token_type` of `urn:ietf:params:oauth:token-type:access_token`: an authorization credential issued out of band by this authorization server that pre-authorizes software statement issuance, analogous to the initial access token of {{RFC7591}}. This profile standardizes the exchange, not the credential, which is deployment-defined.
  * **Renewal.** A software statement this authorization server previously issued for the same `sub`, presented with a `subject_token_type` of `urn:ietf:params:oauth:token-type:software-statement` ({{renewal}}).

`client_id`:
: REQUIRED. The client identifier URL described in {{client-identity}}.

`audience`:
: OPTIONAL. The requested audience, with the syntax, validation, and narrowing rules of the `audience` authorization request parameter ({{authorization-request}}). On renewal ({{renewal}}), the subject statement's `aud` stands in for the requested audience where the request carries none, and bounds it where it does, so a replacement is never broader than the statement it replaces.

`completion_mode`:
: As defined for software statement code redemption ({{software-statement-code-redemption}}).

The request MUST NOT contain `actor_token` or `actor_token_type`, nor the `scope`, `resource`, or `authorization_details` parameters prohibited by {{prohibited-parameters}}.

The client authenticates, and a public client constrains the request, as described in {{client-identity}}; an assertion-based method such as `private_key_jwt` {{RFC7521}} {{RFC7523}} serves where the document specifies one.

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

A use is consumed when the authorization server commits to an outcome: issuing a statement, creating a deferral, or denying issuance. Concurrent presentations of a single-use credential MUST NOT both be committed.

The authorization server MUST validate the subject token before retrieving client-controlled metadata or enqueueing any processing. A subject token MUST result in `invalid_request` (Section 2.2.2 of {{RFC8693}}) if it:

* is invalid or revoked;
* has expired, unless it is a prior software statement accepted under {{renewal}}; or
* does not authorize issuance for the presented `client_id`.

An unacceptable requested audience results in `invalid_target` {{RFC8693}}.

The authorization server MUST bind a fresh metadata snapshot ({{metadata-snapshot}}) and the requested audience before returning either the software statement or a deferral code.

Issuance policy determines whether an initial access token authorizes only the request or issuance itself. The exchange results in a software statement token response ({{software-statement-response}}), a deferral at a deferred issuer ({{deferred-processing}}), or the terminal denial of {{terminal-denial}}.

## Renewal {#renewal}

A client renews by presenting its current or most recent software statement as the subject token. The authorization server MUST verify that:

* it issued the statement;
* the statement's `sub` equals the request's `client_id`; and
* the client authenticated with a key carried both by the document the statement's `cimd_digest` names and by the current document.

That authentication is the holder binding: a statement is otherwise a bearer artifact, and without it whoever held a copy could renew. To check the reviewed document's keys, an issuer offering renewal retains the octets of each document it issues a statement over.

A client rotating keys inline renews while its document carries both the old and the new key; for a document naming `jwks_uri`, the binding is to whatever that location serves. A public client holds no such key and obtains a replacement through the redirect flow ({{authorization-request}}).

A replacement carries the subject statement's `aud_tenant` while the decision is confined to that tenant, and is matched to the statement it replaces by `iss` and `sub` ({{STATEMENT}}).

An issuer that publishes status MUST NOT accept as subject token a statement whose own published status is other than `VALID`. Renewing a statement it has withdrawn would reissue the decision that withdrawal ended.

The authorization server MAY accept a statement that has expired, and SHOULD bound how long after expiry it will do so, since a client absent for an extended period is asking to be re-established rather than renewed.

Where the current document's digest equals the subject statement's `cimd_digest`, whether renewal requires fresh review is issuer policy. Where the digests differ, the issuer MUST apply the decision it would apply to a first issuance for that document, and MUST NOT renew on the strength of the prior statement alone, since otherwise whoever can change the document could obtain the issuer's signature over an unreviewed change.

# Software Statement Authorization Request {#authorization-request}

The client sends an authorization request as described in Section 4.1.1 of {{RFC6749}}, with the following parameters:

`response_type`:
: REQUIRED. The value MUST be `software_statement_code`.

`client_id`:
: REQUIRED. The client identifier URL described in {{client-identity}}.

`redirect_uri`:
: REQUIRED. The value MUST exactly match one of the redirection URIs in the Client ID Metadata Document, subject to the redirect URI rules of {{CIMD}}. A public client MUST use an HTTPS redirection URI and MUST NOT use a private-use URI scheme or a loopback interface redirection URI in a software statement request. A confidential client MAY use a loopback or private-use redirection URI ({{authorization-response-security}}).

`state`:
: REQUIRED. An opaque value used by the client to bind the authorization response to its request.

`code_challenge`:
: REQUIRED. A PKCE challenge as defined by {{RFC7636}}.

`code_challenge_method`:
: REQUIRED. The value MUST be `S256`.

`response_mode`:
: OPTIONAL. The response mode, as defined by {{OAUTH-MRT}}. The default for `response_type=software_statement_code` is `query`; {{software-statement-code-response}} gives response mode considerations.

`audience`:
: OPTIONAL. A target service at which the client intends to use the statement, with the semantics of Section 2.1 of {{RFC8693}}; the parameter can be repeated to request several. Each value MUST be an authorization server issuer identifier as defined by {{RFC8414}}; values MUST NOT be repeated, and order is insignificant.

`completion_mode`:
: OPTIONAL. A value that includes `deferred`, sent as the advance hint of Section 4.2 of {{DTR}}.

`dpop_jkt`:
: REQUIRED for a public client, and for a confidential client whose `redirect_uri` is a loopback or private-use URI; OPTIONAL otherwise. A declared confidential method proves key possession, not that the key is absent from distributed software, which is why the confidential-client exception for loopback and private-use redirection URIs carries this condition. The parameter has the semantics of Section 10 of {{RFC9449}}.

The authorization server selects the final audience according to policy. It MUST NOT place in the statement's `aud` claim any value the request did not carry, except that a renewal request carrying no `audience` counts as carrying the subject statement's `aud` ({{token-exchange-profile}}); an issuer narrows a requested audience and never widens it. Where no requested audience is acceptable, the authorization server MUST reject the request with `invalid_target` {{RFC8707}}. An authorization server whose policy requires a restricted audience rejects a request carrying none with the same error. These semantics apply only to software statement requests and do not affect proprietary uses of `audience` for access-token targeting.

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

An authorization server MUST reject a request containing any of these parameters with `invalid_request`. The same prohibition and error apply to a software statement code redemption ({{software-statement-code-redemption}}) and to a token exchange under {{token-exchange-profile}}. The `audience` parameter ({{authorization-request}}) is not prohibited: it describes the requested artifact rather than requesting access.

Hybrid response types that combine `software_statement_code` with `code`, `token`, `id_token`, or any other response type are not defined. An authorization server MUST reject such a request with `unsupported_response_type`.

# Authorization Response {#authorization-response}

After validating the request and performing any immediate interaction, the authorization server returns the software statement code response or an error. The authorization server MUST NOT place the software statement or approval-sensitive information in any response from the authorization endpoint.

A denial is never signaled in the authorization response: the authorization server returns a software statement code whether the issuance decision is complete, pending, or already a denial, and delivers any denial at redemption ({{terminal-denial}}).

## Software Statement Code Response {#software-statement-code-response}

The authorization server returns the following parameters to the client's redirection endpoint through the user agent, using the selected response mode. The response applies the authorization response protections of {{RFC9700}}.

The parameters are:

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
: REQUIRED when the authorization server advertises `deferred_token_response_supported` ({{authorization-server-metadata}}); otherwise not used, and an authorization server that does not defer ignores it. When present, the value MUST include `deferred`. A deferral-capable issuer rejects a redemption that omits it with `invalid_request`.

The request MUST NOT contain `audience`, which was bound at the authorization endpoint; a request containing it is rejected with `invalid_request`. The client authenticates according to {{client-identity}}. When `dpop_jkt` was included in the authorization request, the client MUST send a DPoP proof for the token endpoint using the same key.

The authorization server MUST validate the software statement code and all of its bindings before processing the request. An invalid, expired, previously used, or incorrectly bound code MUST result in an `invalid_grant` error; a PKCE or DPoP binding failure is handled according to {{RFC7636}} or {{RFC9449}}, respectively.

A redemption attempt consumes the software statement code whenever the presented code value is valid, including when its PKCE, DPoP, or client-authentication bindings fail ({{authorization-response-security}}). A DPoP nonce challenge {{RFC9449}} is not a binding failure: an authorization server requiring a nonce issues the `use_dpop_nonce` challenge before evaluating the code, which remains unconsumed.

When a previously consumed code is presented again, the authorization server SHOULD revoke any deferral derived from it ({{RFC9700}}). Because anyone who observed the code in a URL can trigger that revocation, an authorization server MAY limit revocation to replays that pass the PKCE check.

For a valid, unconsumed code, the result depends on the issuance decision:

* **Approved:** the authorization server returns the software statement token response ({{software-statement-response}}).
* **Denied:** it returns the terminal denial of {{terminal-denial}}.
* **Pending:** it returns the deferred token response of {{DTR}}, binding the deferral to the code's metadata snapshot and audience in addition to the bindings {{DTR}} requires; the client then polls ({{deferred-processing}}).

An authorization server that does not defer completes the decision before responding.

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

An authorization server MAY answer an originating request with a deferred token response {{DTR}} instead of a statement.

Deferral follows {{DTR}}, including its opt-in, polling, sender constraint, and cancellation, with the following additions:

* A deferral created under this specification MUST be delivered by polling. A client MUST NOT send the `client_notification_token` parameter of {{DTR}}; an authorization server rejects a request carrying it with `invalid_request` and MUST NOT deliver a callback, whatever the client's metadata says. Callback delivery is a deferred capability ({{design-rationale}}).
* A successful polling response is the software statement token response of {{software-statement-response}}, not an access token response.
* Issuers SHOULD set deferral code lifetimes that reflect their actual approval latency, which for a review involving human judgment can be hours or days.

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
: REQUIRED. The software statement issued by the authorization server. It MUST conform to {{STATEMENT}}, with `cimd_digest` the digest of the bound metadata snapshot and `aud` the selected audience where one is restricted.

`issued_token_type`:
: REQUIRED. The value MUST be `urn:ietf:params:oauth:token-type:software-statement`.

`token_type`:
: REQUIRED. The value MUST be `N_A`, indicating that an OAuth access token type does not apply.

`expires_in`:
: RECOMMENDED. The remaining lifetime of the software statement in seconds. If present, it MUST be consistent with the statement's `exp` claim.

The response MUST NOT contain `refresh_token` or `scope`. The authorization server MUST include `Cache-Control: no-store`; it SHOULD also include `Pragma: no-cache`.

DPoP under this specification binds requests and deferral state, not the issued artifact; {{DTR}}'s requirement that a final access token inherit the originating DPoP binding does not apply, since no access token is issued.

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

The `access_token` member is a security-token container ({{RFC8693}}), not an OAuth access token: the software statement is consumed only as a software statement ({{STATEMENT}}), MUST NOT be attached to a request as an `Authorization: Bearer` credential, and is not subject to refresh. Implementations that cache issued tokens by type SHOULD key this artifact on its `issued_token_type` so that generic access-token handling does not apply to it, and SHOULD treat it as a sensitive credential in logs.

## Terminal Denial {#terminal-denial}

When the authorization server decides not to issue the requested software statement, it returns a token error response as defined in Section 5.2 of {{RFC6749}} with the error code `access_denied` and HTTP status code 400, and MUST include the `Cache-Control: no-store` response header field. This applies to both originating requests, including where the decision completes during deferral and the denial answers a polling request ({{deferred-processing}}).

The denial is terminal for the request; a denied deferral behaves as {{DTR}} specifies for a request resolved with an error. A denial does not preclude a later issuance request; whether to accept one is issuance policy.

# Status Publication {#status-publication}

An issuing authorization server that ends decisions before their expiry publishes statement status as {{STATUSLIST}} defines, and carries the `status` claim in the statements it issues, as {{STATUS}} defines. Withdrawing a statement does not withdraw a replacement obtained through {{renewal}}, and withdrawing a replacement does not restore its predecessor.

# Authorization Server Metadata {#authorization-server-metadata}

An authorization server that issues software statements under this specification advertises `true` for `client_id_metadata_document_supported` ({{CIMD}}) and publishes the issuer metadata of {{STATEMENT}}.

An issuer that may defer a request ({{deferred-processing}}) publishes the metadata {{DTR}} requires, including `true` for `deferred_token_response_supported`. A synchronous issuer advertises neither that value nor deferral-code revocation, and need not implement {{DTR}}.

An authorization server supporting the redirect flow advertises:

* `urn:ietf:params:oauth:grant-type:software-statement` in `grant_types_supported`, the grant through which software statement codes are redeemed;
* `software_statement_code` in `response_types_supported`; and
* `true` for `authorization_response_iss_parameter_supported`, as defined by {{RFC9207}}.

An authorization server supporting the token exchange profile ({{token-exchange-profile}}) advertises `urn:ietf:params:oauth:grant-type:token-exchange` in `grant_types_supported` and publishes `software_statement_subject_token_types_supported` (below); general token exchange support does not by itself imply support for this profile. An implementation supporting only that profile advertises neither the software statement grant nor the `software_statement_code` response type.

A client that requires deferral MUST NOT send a request to an authorization server that does not advertise `deferred_token_response_supported`, since such an issuer answers with a statement or a terminal denial, never a deferral.

This specification defines the following additional authorization server metadata members:

`software_statement_subject_token_types_supported`:
: REQUIRED for an authorization server that supports the token exchange profile ({{token-exchange-profile}}), and absent otherwise. A JSON array of the `subject_token_type` values the authorization server accepts when `requested_token_type` is `urn:ietf:params:oauth:token-type:software-statement`. Publishing this member signals support for the profile; listing `urn:ietf:params:oauth:token-type:software-statement` among its values advertises renewal by prior statement ({{renewal}}).

# Security Considerations {#security-considerations}

## Client Establishment Is Not an Access Grant

A software statement attests client metadata; it grants no resource access or consent on behalf of the software's users. A software statement returned in the `access_token` member ({{software-statement-response}}) MUST NOT be accepted as an access token at a protected resource.

When an approval interface is shown, it SHOULD state that the decision concerns attestation to client metadata. It MUST NOT imply that the approver is granting the client access to resources.

An erroneous approval affects every authorization server in the statement's audience until expiry. The approval interface therefore SHOULD present:

* the client identifier URL, with its origin shown as the client's identity and display metadata such as `client_name` and `logo_uri` marked as asserted by the client, since a look-alike document can carry another vendor's name;
* the document content it will vouch for, identified by its digest;
* the audience the issuer intends to place in the statement;
* the tenant the decision is confined to, where the statement will carry `aud_tenant` ({{STATEMENT}}); and
* the intended lifetime.

The interface SHOULD make narrowing visible when the client requested a different or broader audience. A document naming instance-attestation authorities ({{CLIENT-INSTANCE}}) endorses those authorities for the software under review and deserves particular scrutiny.

## What Issuance Attests {#what-issuance-attests}

A software statement means only that its issuer evaluated the document captured in the metadata snapshot ({{metadata-snapshot}}) under its issuance policy and decided to vouch for it; it is not proof that the document's contents are true. The client authors the metadata document, so an issuer SHOULD corroborate security-relevant metadata through evidence beyond the document itself; verification depth is part of the trust relationship.

## Client Metadata Retrieval

Fetching a Client ID Metadata Document and resources referenced by it exposes the authorization server to server-side request forgery, resource exhaustion, malicious content, and client impersonation risks; the considerations of {{CIMD}} apply.

## Authorization Response Security {#authorization-response-security}

The software statement is a signed credential and can contain sensitive deployment information, which is why it is never returned in an authorization response ({{authorization-response}}).

Before redeeming a software statement code, the client MUST verify `state` and MUST validate the authorization response `iss` parameter according to {{RFC9207}}.

A public client presents no client authentication, so its association with the software depends on delivery to a metadata-listed redirection endpoint. Another application can claim a private-use or loopback endpoint ({{RFC8252}}); PKCE and DPoP bind the code to the initiator but cannot stop an attacker from initiating under another party's `client_id`. Only HTTPS demonstrates control of the publisher's origin, which is why {{authorization-request}} requires it of a public client. A native public client can host such an endpoint or use the token exchange profile ({{token-exchange-profile}}).

A confidential client's authentication protects redemption, which is why {{authorization-request}} permits it a loopback or private-use redirection URI. An authorization server SHOULD additionally relate the redirection URI's origin to the client identifier URL or the metadata document's `client_uri` according to policy. Endpoint or key control informs issuance policy but does not determine issuance.

A software statement code in a URL is visible to browser history, referrer fields, logs, and other observers. It is short lived, single use, and unredeemable without the PKCE verifier and, where one was bound, the `dpop_jkt` key; `form_post` ({{software-statement-code-response}}) keeps it out of URLs.

## Token Exchange Considerations {#te-considerations}

Validating the subject token before metadata retrieval or enqueueing ({{token-exchange-profile}}) limits resource consumption by unauthorized requesters. Authorization servers SHOULD still rate-limit these exchanges, cache successful retrieval results and back off after failures rather than caching them, which {{CIMD}} forbids, and bound pending deferrals per client identifier and requester. The redirect flow likewise reaches the approval queue before any client-authenticated step; the same rate limits and pending-approval bounds SHOULD apply per client identifier there.

A subject token is an authorization credential, not a client identifier or a substitute for client authentication where the Client ID Metadata Document establishes a method. It appears in a form body, so any component recording request bodies can expose it. Authorization servers MUST exclude subject tokens from logs, traces, error messages, and audit records; clients and authorization servers MUST protect the credential as a bearer credential unless its format provides proof of possession.

A token exchange carries no in-band evidence of user participation. An authorization server MUST NOT treat a token exchange as implying prior user consent and MUST apply the same issuance and approval policy as for the redirect flow.

## Renewal by Prior Statement

Where both documents carry their keys inline, a party able to change the current document cannot supply a key that satisfies the holder binding of {{renewal}}; where they name a `jwks_uri`, whoever controls that location can, which is why a changed document receives the decision a first issuance receives. Binding renewal to the client's key also spares automated renewal a long-lived reusable initial access token, which would be a standing credential to mint statements.

## Signing Keys and Algorithms

Compromise of a software statement signing key lets an attacker mint statements for every audience that trusts that key.

* Issuers SHOULD protect signing keys according to the scope of their trust relationships and support controlled key rotation.
* Issuers SHOULD prefer signature algorithms with modern security properties, such as `PS256`, `ES256`, or `EdDSA`, over RSASSA-PKCS1-v1_5 (`RS256`), and MUST follow {{RFC8725}} when signing.

## Approver Identity and Audit

A software statement does not identify the human or system that approved issuance. Deployments that require approver attribution retain it in an authorization server audit record or define an explicit statement claim and its privacy semantics.

An approved statement is accepted at every authorization server in its audience, not only within the approver's own scope. The policy governing who may approve issuance MUST be at least as restrictive as the policy governing manual client establishment at the issuing authorization server. Approval by a party authorized only for a personal or organizational scope MUST NOT produce a statement whose audience exceeds that scope, or whose `aud_tenant` names a tenant outside it. An issuer SHOULD limit what an approver may approve to client identifiers under publisher namespaces it has enrolled, so that a look-alike document cannot reach approval.

Authorization servers SHOULD keep an audit record that binds each decision, whether approval or denial, to the metadata digest ({{metadata-snapshot}}) of the document the deciding party evaluated, the policy under which the decision was made, the identity of that party, and the time of decision. Authorization servers SHOULD retain the exact retrieved octets of the approved document for audit purposes; a re-serialized copy cannot reproduce the digest.

# Privacy Considerations

The authorization server learns the client identifier URL, the canonical metadata document, and information about the party interacting with the authorization endpoint. It SHOULD collect and retain only the information required for issuance, security monitoring, and audit obligations.

An `aud` claim reveals which authorization servers the client plans to establish relationships with. {{STATEMENT}} weighs that disclosure against omitting the claim and recommends that an issuer name an audience, which an issuer does by requiring one in the request ({{authorization-request}}).

Clients SHOULD NOT present statements outside their intended deployment context, and a redirect-flow client SHOULD use Pushed Authorization Requests {{RFC9126}} where the relationship is sensitive.

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
: `software_statement_subject_token_types_supported`

Metadata Description:
: JSON array of subject token type values accepted for exchanging into software statements; signals support for the token exchange profile.

Change Controller:
: IESG

Specification Document(s):
: This specification, {{authorization-server-metadata}} and {{token-exchange-profile}}


--- back

# Design Rationale {#design-rationale}

**Why the software statement code is not an authorization code.** Redeeming an authorization code issues an access token under {{RFC6749}}. This flow issues an artifact that grants nothing; a distinct code keeps implementations that treat any code as redeemable from confusing the two.

**Why the response uses `access_token`.** {{RFC8693}} defines the container, so existing token endpoint machinery carries the artifact; the `issued_token_type` identifies what it is.

**Why not the device authorization grant.** {{RFC8628}} fits a human decision that outlives a request, and an issuer whose approval is always out of band can use it. It assumes a user co-present with a constrained device who enters a user code elsewhere. Issuance approval is made by an administrator or reviewer who does not operate the client and is often not present, and the client already has a browser. The redirect flow covers an approver reachable through that browser; the token exchange profile covers the case with no browser.

**Why not a profile of attestation-based client authentication.** An attester vouches for a running instance and its key, for as long as it chooses, to the server in front of it; a statement issuer vouches for reviewed software, for days, to every server that trusts it. Profiling one as the other would give the reviewed-software decision an instance-scoped trust model, or give instance attestation unwarranted portability.

**Why not OpenID Federation trust marks.** A trust mark {{OPENID-FED}} is a signed third-party assertion about an entity, resolved through a federation that supplies key discovery, policy, and delegation and requires both parties to enroll. This specification is pairwise: a consumer configures an issuer directly. Federation standardizes trust resolution, not enrollment, approval, or deferred completion; ecosystems already operating a federation are better served by trust marks.

**Deferred capabilities.** This version omits callback delivery for deferral.

# Acknowledgments
{:numbered="false"}

This specification builds on {{DTR}}, {{CIMD}}, and {{APPROVAL-DCR}}. The author thanks the authors and contributors to those specifications and the OAuth working group participants whose discussions informed this work.
