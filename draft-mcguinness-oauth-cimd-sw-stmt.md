---
title: "CIMD Software Statement"
abbrev: oauth-cimd-sw-stmt
docname: draft-mcguinness-oauth-cimd-sw-stmt-latest
category: std

ipr: trust200902
area: "Security"
workgroup: "Web Authorization Protocol"

keyword:
 - OAuth
 - Software Statement
 - Dynamic Client Registration
 - Client ID Metadata Document
 - Registration Validity
 - Sender Constraint

stand_alone: yes
pi: [toc, sortrefs, symrefs]

author:
 -
    ins: K. McGuinness
    name: Karl McGuinness
    organization: Independent
    email: public@karlmcguinness.com

normative:
  RFC6749:
  RFC6234:
  RFC6838:
  RFC7515:
  RFC7519:
  RFC8725:
  RFC7521:
  RFC7591:
  RFC8414:
  RFC9126:
  RFC9449:
  RFC9700:
  CIMD:
    target: https://datatracker.ietf.org/doc/draft-ietf-oauth-client-id-metadata-document
    title: "OAuth Client ID Metadata Document"

informative:
  ISSUANCE:
    target: https://datatracker.ietf.org/doc/draft-mcguinness-oauth-cimd-sw-stmt-issuance
    title: "CIMD Software Statement Issuance"
  RFC7592:
  ABCA:
    target: https://datatracker.ietf.org/doc/draft-ietf-oauth-attestation-based-client-auth
    title: "OAuth 2.0 Attestation-Based Client Authentication"
  CLIENT-INSTANCE:
    target: https://datatracker.ietf.org/doc/draft-mcguinness-oauth-client-instance-assertion
    title: "OAuth 2.0 Client Instance Assertion"
  SIGNALS:
    target: https://datatracker.ietf.org/doc/draft-mcguinness-oauth-cimd-sw-stmt-signals
    title: "Shared Signals Events for CIMD Software Statements"

--- abstract

RFC 7591 defines the software statement as input to dynamic client registration but does not define how long the resulting registration remains valid or how a client renews the statement on which it was based. This specification profiles the software statement of RFC 7591 for clients identified by a Client ID Metadata Document, and defines everything a trusting authorization server does with one: which issuers it trusts, how it validates and applies one, and the two points at which it consumes one. Consumed in a registration request, a statement governs that registration until it expires and a replacement renews it. Presented in an authorization or token request, it establishes an otherwise unregistered client for that request and the grant state derived from it, on proof of a key the reviewed document carries, so possession of the statement alone is insufficient. Together these let an issuer curate approved client software across the authorization servers in a statement's audience while each server keeps control of trust, grants, and token lifetime.

--- middle

# Introduction

{{RFC7591}} defines no standard expiry or renewal procedure for a dynamic client registration. A software statement ({{RFC7591}}, Section 2.3) can carry a reviewer's approval into a registration request, but the registration can outlive the statement and the review it represents. An organization that reviews client software therefore has no interoperable way to keep that review current at the authorization servers that relied on it.

This specification defines the trust configuration a consuming server keeps ({{issuer-trust}}) and two points at which it consumes it:

* **At registration** ({{dcr-presentation}}): the statement is consumed in an {{RFC7591}} registration request and at renewal, and its `exp` bounds the registration's validity ({{registration-validity}}).
* **At runtime** ({{cimd-presentation}}): the client presents the statement in an authorization or token request, proves a key the statement attests, and is established for the resulting grant without creating a persistent registration ({{runtime-presentation}} and {{grant-lifecycle}}).

Establishment is one layer of the decision to let a client act, and this specification defines only that layer. It sits above the sender-constraint proof that identifies the presenter and the grant that carries a user's authorization, and beside a question it deliberately does not answer: whether a particular customer permits this software to operate in its tenant right now. That decision changes on the customer's clock rather than the reviewer's, and where a customer's identity provider mediates the grant it is answered continuously by whether that provider issues an assertion at all ({{identity-assertions}}). A statement answers the durable question instead: who reviewed this software, and what did they attest.

In both profiles, ceasing statement renewal stops new establishment after the applicable expiry. It also stops continued use of an existing registration or grant where this specification requires a current statement. It does not revoke access tokens already issued, and continuation of runtime-established grants is subject to the refresh policy in {{refresh}}. These enforcement bounds are detailed in {{enforcement-bounds}}.

This separation lets one issuer curate approved software across many authorization servers without making the issuer the final policy authority. Each server independently configures issuer trust, subject scope, authoritative metadata, grant policy, and token lifetime. An enterprise operating a statement issuer is the motivating deployment ({{deployment-model}}).

This document defines the statement, its validation, and its consumption; {{ISSUANCE}} defines how a client obtains one, and how a client acquired its statement is out of scope here. A statement authorizes metadata, not its presenter. Runtime proof of the presenter is what the sender-constraint rules of {{sender-constraint}} supply, and nothing in this specification attests software instances or binaries.

## Protocol Overview

The following non-normative sequences summarize the two models.

Registration governed by a statement:

1. The client registers through {{RFC7591}}, carrying a statement in the `software_statement` member.
2. The authorization server records the statement's identity and `exp` with the registration; the registration is valid until that expiry.
3. The client delivers a replacement statement in an authenticated token request or, where supported, an {{RFC7592}} update request; the registration's validity extends to the replacement's `exp`.
4. If no replacement arrives, the registration expires and requests under it fail.

Runtime presentation:

1. The client includes the `software_statement` parameter in a token request, or in a pushed authorization request for a redirect flow, with its Client ID Metadata Document URL as `client_id`.
2. The authorization server validates the statement ({{profiles}}), verifies the sender constraint ({{sender-constraint}}), and resolves the reviewed document ({{effective-metadata}}).
3. The request proceeds under that document's metadata. No persistent registration is created.
4. The state the rest of the grant depends on persists as an establishment ({{grant-lifecycle}}).

## Relationship to Client Attestation {#relationship-attestation}

This section is non-normative.

A software statement is an attestation. It is a signed assertion by a third party about a client, accepted by an authorization server that has configured trust in the signer, and it is no more than that: an attributable claim bounded by the signer's process, not proof that what it says is true. It differs from the client attestation of {{ABCA}} and the client instance assertion of {{CLIENT-INSTANCE}} in subject, authority, lifetime, and effect, not in kind.

Subject:
: A software statement attests client software and the metadata a reviewer evaluated. A client attestation attests a running instance and the key it holds.

Authority:
: A software statement is signed by a review authority whose scope is a set of client identifiers. A client attestation is signed by a client attester whose scope is a deployment of the software; the two are configured independently by the authorization server.

Lifetime and audience:
: A software statement carries an audience and an expiry the reviewer chose, and the same statement is consumed at every server in that audience. A client attestation is short-lived and bound to the request it accompanies.

Effect:
: A software statement supplies establishment: which client this is, and what metadata a named reviewer stands behind. A client attestation supplies presenter proof: that the party sending this request holds a key someone vouches for. Neither grants access, and neither substitutes for the other.

The two therefore compose. A deployment holding only a client attestation knows what is running but not whether anyone approved it; a deployment holding only a software statement knows the software was reviewed but not that this sender is running it. Runtime presentation always requires both halves: the statement carries the review, and possession of a key the statement attests carries the presenter ({{sender-constraint}}).

This specification defines no new attestation format and no new attester role. The presenter proves a key the reviewed document carries, using ordinary client authentication or DPoP; binding a presenter attestation defined elsewhere is an extension ({{extensions}}).

# Conventions and Definitions

{::boilerplate bcp14-tagged}

OAuth terminology is defined by {{RFC6749}}. Client metadata and software statement terminology is defined by {{RFC7591}}. Client ID Metadata Document terminology is defined by {{CIMD}}. This specification additionally defines the following terms:

Issuing Authorization Server:
: The authorization server that makes the issuance decision and signs the software statement.

Trusting Authorization Server:
: An authorization server that consumes a software statement, at registration or at runtime.

This specification additionally defines the following terms:

Runtime Presentation:
: The consumption of a validated software statement inside an authorization or token request, applying the metadata of the document it vouches for without creating a persistent client registration.

Statement-Governed Registration:
: An {{RFC7591}} client registration, at a server advertising `software_statement_registration_validity_supported`, whose validity is bound to a software statement's `exp` and renewed by replacement statements ({{registration-validity}}).

Establishment:
: The durable state a successful runtime presentation creates for the grant it opens, enumerated in {{grant-lifecycle}}.

Proven Key:
: The key for which the presenter demonstrates possession during runtime presentation. The accepted proof path binds this key to the statement as specified in {{sender-constraint}}.

# The Software Statement {#profiles}

The software statement is a compact JWT {{RFC7519}} protected by JWS {{RFC7515}}. Although {{RFC7591}} permits a MAC, a statement issued under this specification MUST use an asymmetric digital signature so trusting servers need not receive an issuer-held symmetric key. The issuer and trusting authorization server MUST follow {{RFC8725}} algorithm-verification guidance. The `none` algorithm and symmetric algorithms MUST NOT be used.

The JOSE header MUST include `kid`, identifying the signing key within the issuer's JWK Set, and MUST include `typ` with the value `software-statement+jwt`, applying Section 3.11 of {{RFC8725}}. This value names `application/software-statement+jwt` ({{media-type}}) with the `application/` prefix omitted, as described in Section 4.1.9 of {{RFC7515}}. Explicit typing prevents confusion with other JWTs from the same issuer.

Extensions can add claims; an incompatible revision would use a new type value. An issuer advertises the algorithms it signs with in its own authorization server metadata ({{ISSUANCE}}).

A statement says that its issuer evaluated the Client ID Metadata Document whose content the digest names, and vouches for it to whoever accepts that issuer. The document carries the metadata; the statement carries the decision. The JWT payload MUST contain the following claims.

`iss`:
: REQUIRED. The issuer identifier of the issuing authorization server, as defined by {{RFC8414}}.

`sub`:
: REQUIRED. The exact client identifier URL presented in the request that produced the statement.

`aud`:
: OPTIONAL. One or more audience identifiers restricting which authorization servers may accept the statement, each an authorization server issuer identifier as defined by {{RFC8414}}. Where the claim is present, a trusting authorization server MUST reject the statement unless one of its locally configured audience identifiers exactly matches a value in it, and where the request carried `audience` values every value in the claim MUST have appeared among them. Where the claim is absent, the statement is unrestricted and acceptance rests on the consumer's configured trust in the issuer and its identifier scope ({{issuer-trust}}). Omitting the claim lets one review serve every server that trusts the issuer, which is the portability the artifact exists for; it also lets whoever holds a copy use it at any of them. At a registration endpoint that is the whole exposure, since registration requires no key, so an issuer SHOULD name an audience wherever it expects consumption at registration, and MAY omit it where consumption is by runtime presentation, which requires a key the reviewed document carries ({{statement-validation}}).

`iat`:
: REQUIRED. A NumericDate value representing the time at which the software statement was issued. An issuer MUST NOT issue two statements for a given `iss` and `sub` pair with the same `iat`, and MUST ensure the value increases strictly across its signing nodes, so that the order in which it made its decisions is recoverable from the statements themselves. Consumers rely on that order when one statement replaces another ({{revalidation}}, {{refresh}}) and when a withdrawal separates statements issued before it from those issued after ({{SIGNALS}}). Back-dating a statement to allow for clock skew makes it unusable as a replacement.

`exp`:
: REQUIRED. A NumericDate value representing the expiration time. A trusting authorization server MUST reject an expired statement.

`jti`:
: REQUIRED. A unique identifier for the statement within the issuer's namespace.

`cimd_digest`:
: REQUIRED. The metadata digest ({{metadata-digest}}) of the Client ID Metadata Document the issuer evaluated. This claim binds the statement to the exact document content evaluated during issuance and lets any party determine whether the client's currently published metadata still matches what was attested.

A statement carries no client metadata. The reviewed metadata is the document the digest names, which a consuming authorization server retrieves and verifies for itself, so copying members into the statement would duplicate what the digest already binds and reopen questions of precedence and partial review that the digest settles. A statement issued under this specification MUST NOT contain client metadata claims.

An issuer that means to vouch for particular members without binding the whole document needs an extension defining what a partial review asserts and how a consumer applies it ({{extensions}}).

The issuer determines the lifetime, and any audience restriction, according to policy. Lifetime pulls in two directions: a short one bounds exposure and keeps the review fresh, while at a server implementing the registration-validity model of this specification the same value is the registration's validity, and so the renewal cadence both parties must sustain.

The `sub` claim is the client identifier URL, not the local `client_id` assigned through {{RFC7591}} registration. A trusting authorization server MAY use `sub` to correlate registrations and apply per-client policy ({{multi-instance}}).



## Metadata Digest {#metadata-digest}

The metadata digest is the unpadded base64url-encoded SHA-256 hash {{RFC6234}} of the retrieved representation body after removal of content coding. No transcoding, normalization, or re-serialization occurs; a byte order mark and trailing newline are included. Retrieval for snapshot purposes SHOULD NOT use content negotiation, and an issuance source SHOULD serve the document without negotiated variants, so that independent fetchers obtain identical bytes.

An issuer computes the digest over the document it evaluated; a consumer computes it over the document it retrieves, and equality is what says the two are the same bytes ({{effective-metadata}}).

## Validating a Statement {#validation}

Before accepting a statement, a trusting authorization server MUST:

* verify that the `typ` header carries the value `software-statement+jwt`;
* verify the signature under a key trusted for the exact `iss` value, obtained as {{issuer-trust}} requires;
* verify that every required claim of {{profiles}} is present and of the correct type, rejecting a statement that omits one;
* validate `iat` and `exp`, rejecting an expired statement and one whose `iat` is unreasonably far in the future according to its clock-skew policy;
* where the statement carries `aud`, verify that one of its own audience identifiers appears in it;
* verify that `sub` is a client identifier URL conforming to {{CIMD}}, and that it falls within the identifier scope for which this server accepts the issuer ({{issuer-trust}});
* reject a statement carrying client metadata claims, which {{profiles}} forbids; and
* apply the JWT validation guidance in {{RFC8725}}.

An issuer URL or JWK Set does not establish trust. A trusting authorization server accepts only configured issuers ({{issuer-trust}}) and obtains their keys from that issuer's authorization server metadata {{RFC8414}}, never from the statement.

Rejections at a registration endpoint use the error codes of Section 3.2.2 of {{RFC7591}}: `invalid_software_statement` where the statement is malformed, expired, or fails signature or claim validation, and `unapproved_software_statement` where it validates but is not acceptable here, because its issuer is not configured, its `aud` excludes this server, or its `sub` falls outside the issuer's scope. Rejections elsewhere use {{errors}}.

A trusting authorization server resolving the reviewed document MUST reject a document containing duplicate object member names, since parsers interpret them differently despite an identical digest.

# Issuer Trust Establishment {#issuer-trust}

A trusting authorization server accepts statements only from configured issuers. Trust is established out of band, for example through a marketplace publisher program or shared enterprise operation. This specification defines no in-band issuer discovery or trust decision.

Configuring trust in an issuer is a one-time act that covers every client that issuer attests, so a trusting authorization server maintains a small, stable set of trusted issuers rather than per-client state. For each, it records at least:

* the exact `iss` identifier it will accept;
* the source of that issuer's signing keys: the `jwks_uri` in the issuer's authorization server metadata {{RFC8414}}, reached from the configured `iss`;
* the signing algorithms it will accept from the issuer;
* the client identifier namespaces the issuer may attest through `sub`;
* the audience identifiers the issuer may name;
* the maximum statement lifetime it will honor, which also caps registration validity where the server implements the registration-validity model of {{registration-validity}};
* its policy on repeated and multiple registration ({{multi-instance}}).

These inputs, not the signature alone, define acceptance. Because the issuer is an authorization server role, key retrieval reuses {{RFC8414}} discovery.

A trusting authorization server MUST derive trust from this local configuration and MUST NOT derive it from an `iss`, `jku`, `x5u`, or other key-location value carried in a presented statement. Having established trust, it validates each statement as {{validation}} describes.

Issuer trust SHOULD be scoped as well as explicit. Trust configuration SHOULD constrain each issuer to the client identifier namespaces it is expected to attest, for example URLs under the domains of the software publishers it serves, and a statement whose `sub` falls outside that scope MUST be rejected even when its signature verifies ({{validation}}). An issuer accepted for all values of `sub` can, if compromised or over-broad, mint acceptable statements about any client software.

Where an issuer attests software across many publishers, as an enterprise issuer does, the scope is the set of identifiers it is configured for rather than a single domain. The rejection rule is the same.

# Consumption at Registration {#dcr-presentation}

The statement is consumed in the `software_statement` member of an {{RFC7591}} registration request. The authorization server validates it as {{validation}} requires, and registers the client under its ordinary registration policy. This section defines what the statement's lifetime does to the resulting registration.

Because a statement carries no client metadata ({{profiles}}), the binding that {{RFC7591}} obtained from attested claims taking precedence over the request comes from the document instead. The authorization server MUST resolve the Client ID Metadata Document at the statement's `sub`, MUST verify that its digest equals `cimd_digest`, and MUST register the client with the metadata that document carries. It MUST NOT take a value from another member of the registration request where the document carries that member, and MUST reject with `invalid_client_metadata` a request whose members contradict the document. Without this, a client could present a statement over one document and register the redirect URIs, keys, or branding of another, which at a registration endpoint is a bearer-artifact substitution.

Where the digest does not match, the document has changed since review and {{version-changes}} governs; the authorization server MUST NOT register the changed document as reviewed.

## Registration Validity {#registration-validity}

An authorization server that advertises `software_statement_registration_validity_supported` as `true` MUST apply this model to every registration it creates from a statement of an issuer it has configured for the model:

* It MUST record the governing statement's `iss`, `jti`, `sub`, `iat`, and `exp` with the registration.
* The registration is valid until that `exp`.
* Once that time passes without a replacement ({{revalidation}}), it MUST reject requests under the registration: `invalid_client` at the token endpoint, and at the authorization or pushed authorization request endpoint the error {{RFC6749}} defines for an unauthorized client. The revalidation requests {{revalidation}} permits are the exception.
* It SHOULD retain the expired record so that it can process a later authenticated revalidation ({{oracle-considerations}}), and MAY allow a grace period during which it accepts a replacement without treating the registration as expired.

The disposition of outstanding grants is local policy ({{enforcement-bounds}}).

The metadata signal in {{authorization-server-metadata}} lets a client determine before registration whether this model applies. Because an authorization server may honor less than a statement's full lifetime ({{issuer-trust}}), a client cannot compute the boundary from the statement alone: a server applying this model MUST return a `registration_expires_at` member, a NumericDate giving the time the registration ceases to be valid, in the {{RFC7591}} registration response, and SHOULD return it in any {{RFC7592}} read or update response it supports. A server that omits the signal or advertises `false` can still consume a statement as ordinary {{RFC7591}} registration input, but it MUST NOT claim conformance to this registration-validity model.

## Revalidation {#revalidation}

The client renews a statement-governed registration by delivering a replacement statement, in any of these ways:

* in the `software_statement` parameter of a token request under the registration, authenticated as the registered client under the registration's own method;
* in the `software_statement` parameter of an authenticated pushed authorization request {{RFC9126}} under the registration, which is the renewal path available to a client that holds no refresh token and whose only grant type is the authorization code; or
* in the `software_statement` member of an authenticated {{RFC7592}} update request, where the deployment offers registration management. Such a request replaces the registration's metadata in full, as {{RFC7592}} requires, so the client sends its complete current metadata alongside the statement; a renewal-only request omitting other members would reset them.

The replacement MUST:

* validate under {{validation}} with this server in its audience;
* carry the governing statement's `iss` and `sub`;
* be unexpired; and
* have an `iat` no earlier than the recorded statement's `iat`.

On success the authorization server MUST replace the recorded statement identity, `iat`, and `exp` in a single atomic update, and concurrent deliveries resolve to the most recently issued statement. Whether a replacement over a changed document updates the registration record is local registration policy, and a server that relies on narrowing to take effect applies it.

When an expired registration sends a request containing a replacement, the authorization server MUST authenticate the retained registration and evaluate the replacement before applying the expiry rejection. A valid replacement therefore restores the registration; an omitted or invalid replacement does not.

A request under an expired registration that does not restore it is rejected with `invalid_client`, at every endpoint, including the refresh-token grant. Such a rejection MUST NOT by itself trigger the refresh-token family revocation of {{RFC9700}}; a client recovers by delivering a valid replacement.

A delivery that fails the rules above is rejected with `statement_required`, whatever the state of the registration, so that a client learns immediately rather than by a later outage. A rejected delivery leaves the recorded statement unchanged. A registration request without an {{RFC7592}} registration access token creates a new registration and never renews an existing one.

## Change After Review {#version-changes}

A review covers the document the issuer evaluated, and `cimd_digest` is what says which one. A mismatch against the currently published document is a policy input rather than an automatic failure, and re-issuance against the new document is the remedy. The statement's bounded lifetime caps how stale a review can get: drift the digest comparison never observes still expires with the statement.

# Runtime Presentation {#cimd-presentation}

The client presents its statement in the request. The authorization server validates it under policy for a trusted issuer, verifies the presenter, and applies the reviewed document's metadata to that request without creating a persistent client registration. This establishes an otherwise unregistered client at request time, as {{CIMD}} resolution does, with the issuer's review carried in the statement.

## Presentation Request {#runtime-presentation}

A client presents a software statement by including the following parameter in a token request or a pushed authorization request:

`software_statement`:
: REQUIRED for presentation. The software statement ({{profiles}}). It is consumed as a runtime presentation, a refresh replacement ({{refresh}}), or a revalidation delivery ({{revalidation}}). The authorization server MUST verify that it accepts the statement's issuer for the subject ({{issuer-trust}}), and MUST reject a request repeating the parameter with `invalid_request`.

The request's `client_id` is the client's Client ID Metadata Document URL. It MUST exactly equal the statement's `sub`; the authorization server MUST reject a presentation where they differ. The effective `client_id` is the statement's `sub`, and the authorization server assigns none.

The request MUST also carry the proof required by {{sender-constraint}}: client authentication under a method the reviewed document specifies, or a DPoP proof with a key that document carries. A successful presentation establishes the client for the request and for the grant state derived from it ({{grant-lifecycle}}).

At the token endpoint, a runtime presentation is valid on a request using the `client_credentials` grant, the JWT or SAML assertion grants of {{RFC7521}}, or another grant an authorization server names in `software_statement_presentation_grant_types_supported` ({{authorization-server-metadata}}). Authorization-code redemption and refresh-token use continue an existing grant and follow {{grant-lifecycle}} and {{refresh}} instead. A request that initiates issuance under {{ISSUANCE}}, recognized by `response_type=software_statement_code` or the software statement `requested_token_type`, MUST NOT carry the `software_statement` parameter. A request that carries `software_statement` and is none of a runtime presentation, a refresh replacement ({{refresh}}), or a revalidation delivery ({{revalidation}}) is rejected with `invalid_request`.

An authorization server advertises support through `software_statement_presentation_supported` ({{authorization-server-metadata}}).

### Authorization Endpoint {#authorization-requests}

Presentation in an authorization request MUST use a pushed authorization request {{RFC9126}}. The statement and its proof are sent to the pushed authorization request endpoint, where the processing rules of {{processing}} apply. The subsequent authorization request MUST use a `client_id` exactly equal to the establishment's `sub` and MUST NOT include the `software_statement` parameter. A statement never appears in a front-channel URL, just as {{ISSUANCE}} keeps an issued statement out of authorization responses.

### Token Endpoint

At the token endpoint, the client includes the parameter in an eligible token request as defined by {{runtime-presentation}}.

## Presentation Processing {#processing}

On receiving a presentation, the authorization server proceeds as follows, rejecting as {{errors}} defines at the first failure:

1. Validate the statement ({{validation}}), completing every check that needs no client-controlled retrieval. Complete all validation that does not require a client-controlled network retrieval before performing such a retrieval.
2. Verify the statement requirements of {{profiles}} and the `client_id` rule of {{runtime-presentation}}.
3. Resolve the reviewed document ({{effective-metadata}}), which is the first step requiring a client-controlled retrieval.
4. Verify the sender constraint against that document ({{sender-constraint}}), then evaluate the request against its metadata ({{effective-metadata}}).
5. On success, create the establishment ({{grant-lifecycle}}).

## Sender Constraint {#sender-constraint}

A runtime presentation MUST be sender-constrained by a key the statement attests. The presenter proves possession of that key through the applicable client authentication method or a DPoP proof {{RFC9449}}, and the proof MUST be bound to the current request and validated with the replay protections of that mechanism.

The proven key MUST appear in the `jwks` or at the `jwks_uri` of the reviewed document ({{effective-metadata}}). Where that document specifies a client authentication method, the presenter MUST use it, and where the grant type requires client authentication a DPoP proof does not satisfy that requirement ({{RFC9449}}). The authorization server MUST reject a presentation without such a proof, or whose proven key the reviewed document does not carry.

A statement whose reviewed document carries no key material cannot be presented at runtime and is consumable only through registration ({{dcr-presentation}}). Endorsement of a key the statement does not name, by a client attester or by an issuer the statement delegates to, is left to extensions ({{extensions}}).

Verifying a key at the document's `jwks_uri` is a retrieval at presentation time. A fetch failure leaves the key unverified, and the presentation is rejected as `temporarily_unavailable`. A server MAY reuse a recently retrieved key set within ordinary HTTP caching bounds, subject to a maximum reuse period of its own choosing; it MUST NOT let the client's cache directives alone determine how long a removed key continues to verify ({{external-retrieval}}).

## Reviewed Metadata {#effective-metadata}

A statement vouches for a document, not for a set of claims ({{profiles}}), so the client's metadata for the request is the document the statement names.

Having validated the statement and its proof, the authorization server MUST retrieve the Client ID Metadata Document at the statement's `sub` and compare its digest with `cimd_digest`. A match means the served document is the reviewed one, and its members are the client's metadata for the request. A mismatch means the document changed after review; the authorization server applies the change policy of {{version-changes}}, which MAY accept the current document, reject the presentation, or apply a narrower policy to it, and MUST NOT treat the changed document as reviewed.

The request is evaluated against that metadata: a `redirect_uri` MUST match a redirection URI in the document, and any requested grant type, response type, or scope MUST fall within it. A grant or response type the authorization server supports but the document does not authorize fails with `unauthorized_client`; a scope outside it fails with `invalid_scope`.

A presentation refused because a bound of {{multi-instance}} is reached is rejected with `invalid_client`; the statement and its proof are sound, so the authorization server SHOULD say so in `error_description`, and the client can retry once earlier establishments are released.

## Grant Lifecycle {#grant-lifecycle}

A successful presentation creates an establishment comprising the following, which is the state a server persists for the grant:

* the validated `sub`;
* the statement identity, its `iss`, `jti`, `iat`, and expiry;
* the tenant the grant was opened for, where the authorization server hosts more than one;
* the reviewed metadata and the digest it matched ({{effective-metadata}});
* the issuer trust decision; and
* the sender-constraint mechanism and Proven Key.

The establishment persists for as long as the grant depends on it. The authorization server MUST bind the resulting `request_uri`, authorization code, refresh token, and other grant continuation state to it, as applicable, and MAY discard it once no such state references it.

A token request that redeems an authorization code opened by a presentation:

* MUST have a `client_id` exactly equal to the establishment's `sub`;
* MUST demonstrate possession of the same Proven Key under the same sender-constraint mechanism; and
* MUST NOT carry the `software_statement` parameter.

A redemption carrying a statement is rejected with `invalid_request`; a wrong client identifier or failed key binding is rejected with `invalid_grant`. This prohibition covers redemption of a code bound to an establishment; a registered client redeeming its own code may deliver a replacement statement under {{revalidation}}, which is a delivery rather than a presentation.

A statement MUST be unexpired when presented. Expiry after presentation does not by itself invalidate an establishment already bound. The authorization server controls continued use through its grant and refresh-token policy; {{refresh}} defines how it can require a current replacement statement.

### Refresh {#refresh}

On refresh-token use the authorization server MUST verify possession of the establishment's Proven Key under the same sender-constraint mechanism. It MAY, by local policy, additionally require a current unexpired statement, and SHOULD require one once the establishment's recorded statement has expired ({{enforcement-bounds}}).

An authorization server that holds a record refusing a statement, such as one kept under a withdrawal ({{SIGNALS}}), MUST treat that statement as not current wherever this section requires currency, so that a withdrawal ends grant continuation on the same terms as an expiry.

When policy requires one, the client presents the replacement in the `software_statement` parameter of the refresh request. The replacement:

* MUST validate under {{validation}}, including its audience where it carries one;
* MUST have the establishment's `iss` and `sub`;
* MUST have an `iat` later than the recorded statement's `iat`; and
* MUST authorize the establishment's Proven Key ({{sender-constraint}}).

The refreshed access MUST fall within the metadata of the document the replacement names.

On success, the establishment's statement identity, `iat`, expiry, reviewed metadata, and trust decision are replaced in a single atomic update, and concurrent deliveries resolve to the most recently issued statement. A refresh that fails these requirements, or omits a statement that policy requires, is rejected with `statement_required` and leaves the establishment unchanged. The operation never rotates the establishment's key; a client that needs a new key performs a new presentation.

## Statements from an Established Client {#registered-delivery}

A client already established at an authorization server, whether registered through {{RFC7591}} or registered under its Client ID Metadata Document URL as its `client_id`, can still carry a statement. From the issuer governing its registration, the statement is a delivery: it renews validity under {{registration-validity}} and does not change the client's metadata for the request. Runtime presentation establishes clients the server does not have; it does not reopen metadata for a client it does. A server that wishes a reviewed change to reach the registration applies it through its registration policy ({{revalidation}}) or through {{RFC7592}}, where a narrower record survives until it does. A statement from an issuer the server does not accept for that client is rejected as {{errors}} defines.

A registration created from a statement is statement-governed when the server advertises `software_statement_registration_validity_supported`. The validity and revalidation model of {{registration-validity}} and {{revalidation}} applies, with the delivered statement's `sub` equal to the registered `client_id`. The request authenticates as the registered client under the registration's own method; the delivered statement renews validity and does not otherwise alter the registration. Where the registration is still valid and the server requires a current statement, a refresh-token request that omits one or delivers one failing these rules is rejected with `statement_required`. Where the registration has already expired, {{revalidation}} governs and the rejection is `invalid_client` at every endpoint.

# Registrations Derived from One Statement {#multi-instance}

A statement attests client software, identified by `sub`; it does not identify the runtime instances of that software, and this specification defines no instance identifier.

One unexpired statement is therefore consumable more than once: at each trusting authorization server in its audience and, where local policy permits, in more than one registration at the same authorization server, for example one registration per deployment or tenant. An authorization server hosting multiple tenants resolves which tenant a request belongs to by its own means, such as a per-tenant issuer identifier or a per-tenant endpoint. This specification defines no tenant parameter, and a `client_id` that is a Client ID Metadata Document URL is the same value in every tenant. Bounds and inventories below are counted within whatever scope the authorization server treats as one deployment, which at a multi-tenant server is one tenant.

A trusting authorization server SHOULD use the statement's `sub` and `iss` to inventory the registrations and establishments derived from that issuer's statements about that software, and SHOULD bound their number. The safe default is one registration per (`iss`, `sub`) at one authorization server, counted across replacements, since a replacement statement carries a new `jti` and a bound keyed on it would reset at every renewal. On repeated consumption, local policy can reject the request, treat it as idempotent, or create another registration; {{RFC7591}} defines no duplicate-registration protocol.

Where the reviewed document carries `jwks` or `jwks_uri`, every registration derived from the statement uses that key material rather than an instance-supplied replacement, and the same key is what a runtime presentation proves ({{sender-constraint}}). Software whose instances hold their own keys registers per instance, or waits on the endorsement extension of {{extensions}}.

# Deployment Model: Centrally Curated Software {#deployment-model}

This section is non-normative.

A provider's marketplace decides which software may exist as a client on its platform, and it configures that reviewer's issuer for the identifier namespaces the reviewer speaks for ({{issuer-trust}}). A customer's separate question, which of that software may operate in its own tenant, is answered on the customer's clock and is out of scope here ({{identity-assertions}}).

A marketplace application registers once. Its listing is a statement whose renewal keeps the registration valid, and that single registration serves every tenant, which is what lets a vendor onboard a customer without provisioning anything per customer. Software hosted by its vendor rather than deployed by the customer works the same way: one client, many tenants, one listing lifecycle.

An enterprise that performs its own software review can operate an issuer too, where it wants that review to travel as an auditable artifact rather than as a policy decision inside its identity provider. Its statements name the application as `sub` and the providers where the review should hold as `aud`, and renewal keeps them current.

The controls follow from the lifetime machinery, and the two lifecycles never have to be synchronized. Onboarding a provider is one trust configuration covering the issuer, its role, identifier scope, accepted metadata authority, and lifetime policy. Approved applications then carry that review to the provider instead of being copied into a separate per-application allowlist. Ceasing renewal at the customer's issuer lapses the application in that customer's tenant at every provider, and leaves the vendor's listing and every other customer untouched; ceasing renewal at the marketplace expires the listing itself. Either prevents new runtime presentations after `exp`, and the second expires statement-governed registrations at their recorded boundary. It also ends refresh-based continuation where the provider requires a current statement under {{refresh}}. Already-issued access tokens remain governed by their own lifetime, and providers retain local control over grants and emergency deprovisioning. Narrowing an approval takes effect when the narrower replacement is next consumed. The result is one issuance policy enforced by multiple trusting authorization servers, subject to their explicit local policy.

# Relationship to Identity Assertions {#identity-assertions}

A statement records that a named party reviewed software and vouched for its metadata. It does not record that a particular customer permits that software to act in its tenant at this moment. The two answer different questions on different clocks, and conflating them is what makes an allowlist expensive to keep current.

Where a customer's identity provider mediates a grant, as it does when a cross-domain identity assertion carries the customer's users to a provider, the second question is already answered continuously: the identity provider issues an assertion for the clients the customer permits, and issues none for the rest. Enforcement is per grant, withdrawal takes effect on the next one, and no artifact outlives the decision. A deployment wanting that property should use it rather than reproduce it here.

This specification is for the durable question. A statement is worth its lifecycle where the reviewer is not in the request path: a publisher program or ecosystem directory vouching to servers it will never see, a provider admitting software it has never registered, or a customer that wants an auditable, portable record of what its review covered rather than an inference from an assertion having been issued.

# Error Responses {#errors}

A statement consumed at registration is rejected with the {{RFC7591}} error codes {{validation}} names. A rejected presentation or delivery uses the error responses of {{RFC6749}} for the endpoint at which it was presented. At the token endpoint:

`invalid_client`:
: the statement or its proof fails to establish the client, including failed statement requirements ({{profiles}}) or sender constraint ({{sender-constraint}}); also any request under an expired statement-governed registration that does not restore it ({{revalidation}}), at every endpoint including the refresh-token grant.

`statement_required`:
: a current statement is required and none was supplied, or the one supplied is expired or refused. A client recovers by obtaining a newer statement and retrying; re-sending the same one cannot succeed. This code is returned in place of `invalid_grant` wherever this specification requires a current statement on a refresh-token request, so that a client does not read the rejection as a dead refresh token ({{RFC9700}}).

`temporarily_unavailable`:
: a retrieval this specification requires, of the reviewed document or of a key set it names, did not complete. The client retries; nothing about the statement is in question.

`unauthorized_client`:
: the reviewed document does not authorize the requested grant type or response type, although the authorization server supports it.

`invalid_scope`:
: a requested scope falls outside the reviewed document.

`invalid_request`:
: any other rejection, including a redemption or issuance request carrying the `software_statement` parameter, and a request repeating the parameter ({{runtime-presentation}}, {{grant-lifecycle}}).

Which code applies where:

| Endpoint | Statement invalid or untrusted | Required statement missing, expired, or refused | Retrieval failed |
| --- | --- | --- | --- |
| Registration ({{RFC7591}}) | `invalid_software_statement` or `unapproved_software_statement` ({{validation}}) | `unapproved_software_statement` | `invalid_software_statement` |
| Pushed authorization request | `invalid_client` | `statement_required` | `temporarily_unavailable` |
| Token, including refresh | `invalid_client` | `statement_required` | `temporarily_unavailable` |

At the pushed authorization request endpoint these are carried in the error response {{RFC9126}} defines. On refresh-token use, {{refresh}} takes precedence: a missing or failing replacement statement is `statement_required`. Registration validity is reported differently, as `invalid_client` under {{revalidation}}, because the registration rather than the grant is what lapsed; neither rejection indicates refresh-token replay ({{RFC9700}}).

Where the proof mechanism defines a recoverable error of its own, such as a DPoP nonce challenge {{RFC9449}}, that error takes precedence over the generic errors above.

This surface is coarser than the registration codes and does not tell a client whether to seek a corrected statement or a different issuer. The authorization server MAY return non-sensitive diagnostics in `error_description` to a client it has authenticated, but MUST NOT reveal issuer-trust, subject-namespace, or attester-policy details to an unauthenticated requester. Where a statement is refused by a record rather than merely expired, and the client is authenticated, the authorization server SHOULD indicate that a statement issued more recently is required, since re-delivering the same statement can never succeed.

# Examples {#example}

Both examples are non-normative.

## Revalidating a Statement-Governed Registration

A registered client, `client_id` `s6BhdRkqt3`, renews its registration's validity by delivering a replacement statement on an ordinary refresh, authenticated under its registered method:

~~~ http
POST /token HTTP/1.1
Host: as.example
Content-Type: application/x-www-form-urlencoded
Authorization: Basic czZCaGRSa3F0Mzo3RmpmcDBaQnIxS3REUmJuZlZkbUl3

grant_type=refresh_token
&refresh_token=tGzv3JOkF0XG5Qx2TlKWIA
&software_statement=eyJ0eXAiOiJzb2Z0d2FyZS1zdGF0ZW1l...
~~~

The authorization server validates the replacement, matches its `iss` and `sub` to the registration's governing statement, and atomically extends the registration's validity to the replacement's `exp`. Had the request arrived after expiry without a replacement, it would have failed with `invalid_client` ({{revalidation}}).

## Presenting at the Token Endpoint

The following presents an already-issued statement at the token endpoint. The statement attests the client's `jwks_uri` and `private_key_jwt` as its authentication method, so the client authenticates with a key the statement covers; the `software_statement` parameter carries the issuer's review; and `client_id` is the Client ID Metadata Document URL named by the statement's `sub`. No client record exists at this server.

~~~ http
POST /token HTTP/1.1
Host: as.example
Content-Type: application/x-www-form-urlencoded

grant_type=client_credentials
&scope=tools.read
&client_id=https%3A%2F%2Fclient.example%2Fapp
&client_assertion_type=urn%3Aietf%3Aparams%3Aoauth%3A
client-assertion-type%3Ajwt-bearer
&client_assertion=eyJhbGciOiJFUzI1NiIsImtpZCI6IjIwMjYtMDgifQ...
&software_statement=eyJ0eXAiOiJzb2Z0d2FyZS1zdGF0ZW1l...
~~~

The authorization server validates the statement, retrieves the document its digest names, verifies the client assertion against a key from that document's `jwks_uri`, and evaluates the request against it: the requested `scope` falls within the document's `scope`, and the request authenticates as the client the statement vouches for. It keeps no registration; the effective `client_id` is the statement's `sub`.

# Authorization Server Metadata {#authorization-server-metadata}

This specification defines the following authorization server metadata {{RFC8414}} values:

`software_statement_presentation_supported`:
: OPTIONAL. A JSON array naming the endpoints at which the authorization server accepts a software statement presented at runtime ({{runtime-presentation}}). Defined members are `token` and `pushed_authorization_request`; a member a client does not recognize is ignored. Omission, or an empty array, means the path is not offered. Advertising only `token` is what lets a server offer presentation to clients that need no redirect, without also promising the front-channel path. This member describes the consuming role. It does not imply acceptance of any particular statement issuer or subject namespace, and it does not imply support for statement-governed registrations. An authorization server advertising presentation at the pushed authorization request endpoint MUST publish `pushed_authorization_request_endpoint`, since {{authorization-requests}} makes presentation there the only front-channel path. A client also examines the ordinary client-authentication and DPoP metadata for the proof it intends to use.

`software_statement_presentation_grant_types_supported`:
: OPTIONAL. A JSON array of grant type identifiers on which the authorization server accepts a runtime presentation, in addition to those {{runtime-presentation}} names. Omission means only those.

`software_statement_registration_validity_supported`:
: OPTIONAL. Boolean value indicating whether every registration the authorization server creates from a validated software statement is governed by the validity and revalidation rules of {{registration-validity}} and {{revalidation}}. If omitted, the default value is `false`. A value of `true` does not imply runtime-presentation support. It tells a client that the statement's `exp` will bound the registration and that the server accepts replacement delivery through an authenticated token request or pushed authorization request and, if the server supports {{RFC7592}}, a registration update request. The model is configured per issuer, so a client learns whether its own registration is governed from the `registration_expires_at` member the response carries.

# Extension Points {#extensions}

Three extensions of this path are left to separate specifications. Endorsed keys: a client attestation {{ABCA}}, or an assertion from an issuer named by an `instance_issuers` delegation in the reviewed document {{CLIENT-INSTANCE}}, vouching for a key that document does not carry, which would admit software whose instances hold their own keys. Statement conveyance within a client attestation, rather than as a request parameter. And a Client ID Metadata Document that references where the client publishes its current statements, so a resolving server fetches the review out of band, which differs from embedding a statement in the document as the digest rule of {{metadata-digest}} forbids.

# Security Considerations {#security-considerations}

## Statement Theft and Replay {#statement-validation}

A runtime presentation resists statement theft because the proof is a key the reviewed document carries. Possession of a stolen statement is insufficient unless the attacker also controls that key, and admitting only that key is what prevents downgrade: every server in the audience either binds the presenter to it or refuses the presentation. An extension admitting endorsed keys ({{extensions}}) reopens that question and needs to answer it in its own terms.

A statement consumed at registration is a reusable bearer artifact until it expires, so an issuer relies on narrow audience and lifetime, and the bounds of {{multi-instance}} limit what a stolen statement can create.

Attesting `jwks_uri` attests the location, not its contents: a compromised key host can add keys that satisfy the proof with no digest change, so where that exposure matters an issuer attests `jwks` inline and accepts digest-visible rotation. A server reusing a cached key set under {{sender-constraint}} additionally accepts that a just-removed key can briefly continue to verify.

Nothing elsewhere relaxes these validation rules; it adds the sender constraint, the grant bindings of {{grant-lifecycle}}, and the registration-validity model on top of them.

## Renewal Authenticates the Credential

Renewing a registration proves possession of the registration's own credential and the currency of a statement sharing the governing `iss` and `sub`. It does not prove that the renewing party is the reviewed software: an attacker holding a stolen client credential can renew indefinitely with any current statement for that software, which circulates by design to every deployment of it. Renewal keeps the review current, not the credential honest.

Deployments SHOULD pair statement-governed registrations with credential rotation, sender-constrained client authentication, and the limits of {{multi-instance}}, and SHOULD treat a credential compromise as requiring re-registration rather than renewal. Runtime presentation does not share this gap, because the proof binds the presenter to the reviewed document at every presentation.

## External Retrieval and Resource Exhaustion {#external-retrieval}

Runtime presentation can cause the authorization server to retrieve the Client ID Metadata Document, a `jwks_uri` it names, or keys and metadata associated with an instance issuer it names. Every such retrieval inherits the server-side request forgery protections of {{CIMD}}. A trusted signature does not make a URL safe: the authorization server MUST apply its URL, redirect, address-range, transport, and content-type policy independently to every referenced location.

Presentation reaches these retrievals before any client is registered or any user has interacted, so the work is available to an unauthenticated requester holding one acceptable statement. An authorization server SHOULD therefore bound it:

* rate-limit presentations per statement identity, per subject, and per source, and bound the establishments it will create from one statement ({{multi-instance}}), before spending retrieval or storage on a new presentation;
* bound JWT size and parsing work, concurrent retrievals, response size, and response time; and
* cache successful and failed retrieval results for an appropriate period.

A retrieval failure leaves the relevant metadata or proof unverified; the authorization server MUST reject the request and MUST NOT fall back to a weaker proof.

## Registration Fraud and Impersonation {#registration-fraud}

Open registration permits `client_name`, `logo_uri`, and `client_uri` values that imitate trusted software on consent screens. Requiring a statement replaces self-asserted branding with issuer-reviewed values. Servers that render such values on consent screens SHOULD prefer those from a reviewed document and SHOULD apply heightened scrutiny to registrations that claim user-visible branding without a statement.

Statement-gated registration also makes each rotated identity require another issuer decision, rather than letting a discarded client return at no cost; `sub` and `jti` tracking bounds registrations ({{multi-instance}}). Neither control makes metadata true: a client that misleads review can obtain a genuine statement for fraudulent metadata, so issuer verification depth remains decisive ({{ISSUANCE}}).

## Enforcement Bounds {#enforcement-bounds}

Expiry is enforced at every presentation and every statement-governed registration, so a lapsed statement stops new runtime establishment and causes registration-backed requests to fail at the recorded `exp`. It does not retroactively invalidate an establishment, revoke an access token, or terminate an outstanding grant. Requiring a current statement on refresh under {{refresh}} is the control that makes issuer non-renewal end runtime-established grants, and a deployment that does not adopt it retains grants for the life of their refresh tokens whatever the statement lifetime; registration-backed grants remain subject to the server's grant policy after the registration expires. A narrowed re-review takes effect when the client publishes the narrower document and obtains a statement over it. Detection of post-issuance metadata change rests on `cimd_digest`, which covers exact bytes but requires retrieval for comparison. The bounded statement lifetime limits what either signal can miss for new establishment. {{SIGNALS}} defines an optional event mechanism by which an issuer ends a decision before its expiry; expiry remains the floor, and this specification does not depend on it.

Renewal cadence is a deployment trade: short lifetimes tighten the issuer's control loop and increase issuance and delivery traffic, and a fleet of registrations issued together expires together, so issuers SHOULD stagger expiries or renew ahead of the boundary to avoid synchronized lapses.

## Document Resolution

Every presentation resolves the Client ID Metadata Document its statement names ({{effective-metadata}}), which inherits the resolution considerations of {{CIMD}}, including server-side request forgery and availability. A server MAY cache resolution results within the document's caching directives; the digest is what tells it whether the bytes it holds are the reviewed ones, whatever their source.

## Observable State {#oracle-considerations}

A retained expired registration is distinguishable from an unknown client, because recovery requires the server to authenticate the registration and evaluate a replacement before rejecting. That disclosure is deliberate and bounded: it is available only to a requester that authenticates as the registration, so it reveals to the legitimate client the state it must act on. Servers publishing `software_statement_registration_validity_supported` additionally disclose that statement-derived registrations expire there, which is configuration a client needs before registering. Neither discloses issuer trust, subject scope, or attester policy, which {{errors}} keeps from unauthenticated requesters.

## Statement Handling

A software statement remains a sensitive artifact in transit and at rest: possession alone does not enable presentation, but statements name reviewed software and, where they carry an audience, its intended relationships, and servers SHOULD avoid logging them.

# Privacy Considerations

A presentation or delivery reveals to the authorization server the client's issuer relationship and, where the statement carries an `aud` claim, the other authorization servers the client intends to establish relationships with. Omitting that claim discloses nothing beyond the review ({{ISSUANCE}}). The pushed authorization request requirement of {{authorization-requests}} keeps statements out of browser history, referrers, and front-channel logs. A central issuer additionally learns, through renewal requests, which of its statements are in active use; issuance and renewal logs deserve the same care as the statements themselves.

# IANA Considerations {#iana}

## OAuth Parameters Registry

This specification requests registration of the `software_statement` parameter in the IANA "OAuth Parameters" registry established by {{RFC6749}}, for runtime presentation and statement delivery of the artifact defined by {{RFC7591}}:

The OAuth Dynamic Client Registration Metadata registry already contains a metadata member with the same name. That entry is in a different registry and is unaffected by this registration.

Parameter Name:
: `software_statement`

Parameter Usage Location:
: authorization request, token request

Note:
: In an authorization request the parameter is conveyed only through the pushed authorization request endpoint {{RFC9126}}, never in a front-channel URL ({{authorization-requests}}).

Change Controller:
: IESG

Specification Document(s):
: This specification, {{runtime-presentation}}

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
: This specification, {{profiles}}

## OAuth Extensions Error Registry

This specification requests registration of the following error codes in the IANA "OAuth Extensions Error Registry" established by {{RFC6749}}:

Error Name:
: `statement_required`

Error Usage Location:
: token error response, pushed authorization request error response

Related Protocol Extension:
: CIMD Software Statement

Change Controller:
: IESG

Specification Document(s):
: This specification, {{errors}}

Error Name:
: `temporarily_unavailable`

Existing Registration:
: {{RFC6749}}

Error Usage Location:
: token error response and pushed authorization request error response, in addition to the locations already registered

Change Controller:
: IESG

Specification Document(s):
: {{RFC6749}}, this specification ({{errors}})

## OAuth Dynamic Client Registration Metadata Registry

This specification requests registration of the following client metadata member, returned by an authorization server in a registration response:

Client Metadata Name:
: `registration_expires_at`

Client Metadata Description:
: Time at which a statement-governed registration ceases to be valid, as a NumericDate.

Change Controller:
: IESG

Specification Document(s):
: This specification, {{registration-validity}}

## OAuth Authorization Server Metadata Registry

This specification requests registration of the following values in the IANA "OAuth Authorization Server Metadata" registry established by {{RFC8414}}.

Metadata Name:
: `software_statement_presentation_supported`

Metadata Description:
: JSON array of endpoint names at which the authorization server accepts a software statement presented at runtime.

Change Controller:
: IESG

Specification Document(s):
: This specification, {{authorization-server-metadata}}

Metadata Name:
: `software_statement_presentation_grant_types_supported`

Metadata Description:
: JSON array of grant type identifiers on which the authorization server accepts a runtime presentation, beyond those the specification names.

Change Controller:
: IESG

Specification Document(s):
: This specification, {{authorization-server-metadata}}

Metadata Name:
: `software_statement_registration_validity_supported`

Metadata Description:
: Boolean value indicating whether registrations created from validated software statements are governed by the statement validity and revalidation rules of this specification.

Change Controller:
: IESG

Specification Document(s):
: This specification, {{authorization-server-metadata}}

--- back

# Acknowledgments
{:numbered="false"}

This mechanism began as a consumption section of {{ISSUANCE}} and was split into its own specification so that consumption can evolve separately from issuance. It draws on the working-group discussion around Client ID Metadata Documents and attestation-based client authentication.
