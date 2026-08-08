---
title: "OAuth 2.0 Software Statement Consumption and Runtime Presentation"
abbrev: oauth-sw-stmt-presentation
docname: draft-mcguinness-oauth-sw-stmt-presentation-latest
category: std

ipr: trust200902
area: "Security"
workgroup: "Web Authorization Protocol"

keyword:
 - OAuth
 - Software Statement
 - Software Statement Consumption
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
    email: public@karlmcguinness.com

normative:
  RFC6749:
  RFC7591:
  RFC7592:
  RFC8414:
  RFC9126:
  RFC9449:
  RFC9700:
  CIMD:
    target: https://datatracker.ietf.org/doc/draft-ietf-oauth-client-id-metadata-document
    title: "OAuth Client ID Metadata Document"
  ISSUANCE:
    target: https://datatracker.ietf.org/doc/draft-mcguinness-oauth-software-statement-issuance
    title: "OAuth 2.0 Software Statement Issuance"

informative:
  ABCA:
    target: https://datatracker.ietf.org/doc/draft-ietf-oauth-attestation-based-client-auth
    title: "OAuth 2.0 Attestation-Based Client Authentication"
  CLIENT-INSTANCE:
    target: https://datatracker.ietf.org/doc/draft-mcguinness-oauth-client-instance-assertion
    title: "OAuth 2.0 Client Instance Assertion"
  SIGNALS:
    target: https://datatracker.ietf.org/doc/draft-mcguinness-oauth-sw-stmt-signals
    title: "Shared Signals Events for OAuth Software Statements"

--- abstract

RFC 7591 defines the software statement as input to dynamic client registration but does not define how long the resulting registration remains valid or how a client renews the statement on which it was based. This specification defines everything a trusting authorization server does with a software statement: which issuers it trusts, how it validates and applies one, and the two points at which it consumes one. Consumed in a registration request, a statement governs that registration until it expires and a replacement renews it. Presented in an authorization or token request, it establishes an otherwise unregistered client for that request and the grant state derived from it, on proof of a key the statement itself attests, so possession of the statement alone is insufficient. Together these let an issuer curate approved client software across the authorization servers in a statement's audience while each server keeps control of trust, grants, and token lifetime.

--- middle

# Introduction

{{RFC7591}} defines no standard expiry or renewal procedure for a dynamic client registration. A software statement ({{RFC7591}}, Section 2.3) can carry a reviewer's approval into a registration request, but the registration can outlive the statement and the review it represents. An organization that reviews client software therefore has no interoperable way to keep that review current at the authorization servers that relied on it.

This specification defines the trust configuration a consuming server keeps ({{issuer-trust}}) and two points at which it consumes the artifact {{ISSUANCE}} defines:

* **At registration** ({{dcr-presentation}}): the statement is consumed in an {{RFC7591}} registration request and at renewal, and its `exp` bounds the registration's validity ({{registration-validity}}).
* **At runtime** ({{cimd-presentation}}): the client presents the statement in an authorization or token request, proves a key the statement attests, and is established for the resulting grant without creating a persistent registration ({{runtime-presentation}} and {{grant-lifecycle}}).

Establishment is one layer of the decision to let a client act, and this specification defines only that layer. It sits above the sender-constraint proof that identifies the presenter and the grant that carries a user's authorization, and beside a question it deliberately does not answer: whether a particular customer permits this software to operate in its tenant right now. That decision changes on the customer's clock rather than the reviewer's, and where a customer's identity provider mediates the grant it is answered continuously by whether that provider issues an assertion at all ({{identity-assertions}}). A statement answers the durable question instead: who reviewed this software, and what did they attest.

In both profiles, ceasing statement renewal stops new establishment after the applicable expiry. It also stops continued use of an existing registration or grant where this specification requires a current statement. It does not revoke access tokens already issued, and continuation of runtime-established grants is subject to the refresh policy in {{refresh}}. These enforcement bounds are detailed in {{enforcement-bounds}}.

This separation lets one issuer curate approved software across many authorization servers without making the issuer the final policy authority. Each server independently configures issuer trust, subject scope, authoritative metadata, grant policy, and token lifetime. An enterprise operating a statement issuer is the motivating deployment ({{deployment-model}}).

Statement format, issuance, and validation are defined by {{ISSUANCE}} and are not modified here; this document states which elements each consumption model requires, and how the client acquired its statement is out of scope. A statement authorizes metadata, not its presenter. Runtime proof of the presenter is what the sender-constraint rules of {{sender-constraint}} supply, and nothing in this specification attests software instances or binaries.

## Protocol Overview

The following non-normative sequences summarize the two models.

Registration governed by a statement:

1. The client registers through {{RFC7591}}, carrying a statement in the `software_statement` member, which binds to the registration by its `iss` and `sub` ({{ISSUANCE}}).
2. The authorization server records the statement's identity and `exp` with the registration; the registration is valid until that expiry.
3. The client delivers a replacement statement in an authenticated token request or, where supported, an {{RFC7592}} update request; the registration's validity extends to the replacement's `exp`.
4. If no replacement arrives, the registration expires and requests under it fail.

Runtime presentation:

1. The client includes the `software_statement` parameter in a token request, or in a pushed authorization request for a redirect flow, with its Client ID Metadata Document URL as `client_id`.
2. The authorization server validates the statement ({{profiles}}), verifies the sender constraint ({{sender-constraint}}), and derives the client's effective metadata for the request ({{effective-metadata}}).
3. The request proceeds under the effective metadata. No persistent registration is created.
4. The state the rest of the grant depends on persists as an establishment ({{grant-lifecycle}}).

## Relationship to Client Attestation {#relationship-attestation}

This section is non-normative.

A software statement is an attestation. It is a signed assertion by a third party about a client, accepted by an authorization server that has configured trust in the signer, and it is no more than that: an attributable claim bounded by the signer's process, not proof that what it says is true ({{ISSUANCE}}). It differs from the client attestation of {{ABCA}} and the client instance assertion of {{CLIENT-INSTANCE}} in subject, authority, lifetime, and effect, not in kind.

Subject:
: A software statement attests client software and the metadata a reviewer evaluated. A client attestation attests a running instance and the key it holds.

Authority:
: A software statement is signed by a review authority whose scope is a set of client identifiers. A client attestation is signed by a client attester whose scope is a deployment of the software; the two are configured independently by the authorization server.

Lifetime and audience:
: A software statement carries an audience and an expiry the reviewer chose, and the same statement is consumed at every server in that audience. A client attestation is short-lived and bound to the request it accompanies.

Effect:
: A software statement supplies establishment: which client this is, and what metadata a named reviewer stands behind. A client attestation supplies presenter proof: that the party sending this request holds a key someone vouches for. Neither grants access, and neither substitutes for the other.

The two therefore compose. A deployment holding only a client attestation knows what is running but not whether anyone approved it; a deployment holding only a software statement knows the software was reviewed but not that this sender is running it. Runtime presentation always requires both halves: the statement carries the review, and possession of a key the statement attests carries the presenter ({{sender-constraint}}).

This specification defines no new attestation format and no new attester role. It defines how a review attestation is consumed, and how a presenter attestation defined elsewhere is bound to it.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

OAuth terminology is defined by {{RFC6749}}. Client metadata and software statement terminology is defined by {{RFC7591}}. Client ID Metadata Document terminology is defined by {{CIMD}}. Statement issuance terminology, including Issuing Authorization Server, Trusting Authorization Server, and the statement format and validation rules, is defined by {{ISSUANCE}}.

This specification additionally defines the following terms:

Runtime Presentation:
: The consumption of a validated software statement inside an authorization or token request, applying its attested metadata to that request without creating a persistent client registration.

Statement-Governed Registration:
: An {{RFC7591}} client registration, at a server advertising `software_statement_registration_validity_supported`, whose validity is bound to a software statement's `exp` and renewed by replacement statements ({{registration-validity}}).

Establishment:
: The durable state a successful runtime presentation creates for the grant it opens, enumerated in {{grant-lifecycle}}.

Proven Key:
: The key for which the presenter demonstrates possession during runtime presentation. The accepted proof path binds this key to the statement as specified in {{sender-constraint}}.

# Statement Requirements {#profiles}

This document consumes the software statement of {{ISSUANCE}}. The syntax, validation, and semantics of every element are defined there; this section states what a statement must carry to be consumed here, and what each element is consumed for. How the client acquired it is out of scope.

`typ` header:
: `software-statement+jwt`.

`iss`:
: Identifies the issuer for trust and scope decisions, and forms part of the statement identity recorded with a registration or establishment.

`sub`:
: The client's Client ID Metadata Document URL {{CIMD}}. In a runtime presentation the request's `client_id` equals it, by the rule of {{runtime-presentation}}.

`aud`:
: Scopes which authorization servers may accept the statement.

`iat`:
: The issuance time, which orders one statement against another when a replacement arrives ({{revalidation}}, {{refresh}}).

`exp`:
: The expiry the issuer chose. It bounds registration validity ({{registration-validity}}), is checked at every presentation ({{enforcement-bounds}}).

`jti`:
: Identifies the statement within the issuer's namespace; inventory and concurrency bounds key on the `iss` and `jti` pair. A replacement statement has its own `jti`, so replacement matching does not key on this value.

`cimd_digest`:
: Binds the issuer's review to the exact document bytes evaluated and feeds the post-issuance change policy of {{ISSUANCE}}.

A statement lacking any of these cannot be consumed under this specification, and one failing validation is rejected as {{errors}} defines.

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

These inputs, not the signature alone, define acceptance. Because the issuer is an authorization server role, key retrieval reuses {{RFC8414}} discovery ({{authorization-server-metadata}}).

A trusting authorization server MUST derive trust from this local configuration and MUST NOT derive it from an `iss`, `jku`, `x5u`, or other key-location value carried in a presented statement. Having established trust, it validates each statement as described in {{ISSUANCE}}.

Issuer trust SHOULD be scoped as well as explicit. An issuer accepted for all values of `sub` can, if compromised or over-broad, mint acceptable statements about any client software; trust configuration SHOULD therefore constrain each issuer to the client identifier namespaces it is expected to attest, for example URLs under the domains of the software publishers it serves, and a statement whose `sub` falls outside that scope MUST be rejected even when its signature verifies ({{ISSUANCE}}). Where an issuer attests software across many publishers, as an enterprise issuer does, the scope is the set of identifiers it is configured for rather than a single domain; the rejection rule is the same.

# Consumption at Registration {#dcr-presentation}

The statement is consumed in the `software_statement` member of an {{RFC7591}} registration request. The authorization server validates it as {{ISSUANCE}} requires, with rejections using the {{RFC7591}} error codes defined there, and registers the client under its ordinary registration policy. This section defines what the statement's lifetime does to the resulting registration.

## Registration Validity {#registration-validity}

An authorization server that advertises `software_statement_registration_validity_supported` as `true` MUST apply this model to every registration it creates from a validated software statement. It MUST record the governing statement's `iss`, `jti`, `sub`, `iat`, and `exp` with the registration. The registration is valid until that `exp`. After it passes without a replacement ({{revalidation}}), the authorization server MUST reject requests under the registration, using `invalid_client` at the token endpoint and, at the authorization or pushed authorization request endpoint, the error {{RFC6749}} defines for an unauthorized client, except for the revalidation requests {{revalidation}} permits. The server SHOULD retain the expired record so that it can process a later authenticated revalidation ({{oracle-considerations}}), and MAY allow a grace period during which it accepts a replacement without treating the registration as expired. The disposition of outstanding grants is local policy ({{enforcement-bounds}}).

The metadata signal in {{authorization-server-metadata}} lets a client determine before registration whether this model applies. The client already holds the statement and therefore learns the initial validity boundary from its `exp`. A server that omits the signal or advertises `false` can still consume a statement as ordinary {{RFC7591}} registration input, but it MUST NOT claim conformance to this registration-validity model.

## Revalidation {#revalidation}

The client renews a statement-governed registration by delivering a replacement statement, in any of these ways:

* in the `software_statement` parameter of a token request under the registration, authenticated as the registered client under the registration's own method;
* in the `software_statement` parameter of an authenticated pushed authorization request {{RFC9126}} under the registration, which is the renewal path available to a client that holds no refresh token and whose only grant type is the authorization code; or
* in the `software_statement` member of an authenticated {{RFC7592}} update request, where the deployment offers registration management. Such a request replaces the registration's metadata in full, as {{RFC7592}} requires, so the client sends its complete current metadata alongside the statement; a renewal-only request omitting other members would reset them.

The replacement MUST validate under {{ISSUANCE}} with this server in its audience, MUST have the governing statement's `iss` and `sub`, MUST be unexpired, and MUST have an `iat` no earlier than the recorded statement's `iat`. On success the authorization server MUST replace the recorded statement identity, `iat`, and `exp` in a single atomic update; concurrent deliveries resolve to the most recently issued statement. Whether attested changes in the replacement update the registration record is local registration policy, and a server that relies on narrowing to take effect applies them.

When an expired registration sends a request containing a replacement, the authorization server MUST authenticate the retained registration and evaluate the replacement before applying the expiry rejection. A valid replacement therefore restores the registration; an omitted or invalid replacement does not.

A request under an expired registration that does not restore it is rejected with `invalid_client`, at every endpoint, including the refresh-token grant. Such a rejection MUST NOT by itself trigger the refresh-token family revocation of {{RFC9700}}; a client recovers by delivering a valid replacement.

A delivery that fails the rules above under a still-valid registration leaves the recorded statement unchanged and does not fail the request it accompanies; the authorization server SHOULD signal the declined renewal in `error_description` on a subsequent rejection or through its registration management interface, and the client can distinguish success by the registration's continued validity past the recorded `exp`. A registration request without an {{RFC7592}} registration access token creates a new registration and never renews an existing one.

## Change After Review {#version-changes}

A review covers the document the issuer evaluated, and `cimd_digest` is what says which one. A mismatch against the currently published document is a policy input under {{ISSUANCE}} rather than an automatic failure, and re-issuance against the new document is the remedy. The statement's bounded lifetime caps how stale a review can get: drift the digest comparison never observes still expires with the statement.

# Runtime Presentation {#cimd-presentation}

The client presents its statement in the request. The authorization server validates it under policy for a trusted issuer, verifies the presenter, and applies the effective metadata to that request without creating a persistent client registration. This establishes an otherwise unregistered client at request time, as {{CIMD}} resolution does, with the issuer's review carried in the statement.

## Presentation Request {#runtime-presentation}

A client presents a software statement by including the following parameter in a token request or a pushed authorization request:

`software_statement`:
: REQUIRED for presentation. The software statement ({{profiles}}). It is consumed as a runtime presentation, a refresh replacement ({{refresh}}), or a revalidation delivery ({{revalidation}}). The authorization server MUST verify that it accepts the statement's issuer for the subject ({{issuer-trust}}), and MUST reject a request repeating the parameter with `invalid_request`.

The request's `client_id` is the client's Client ID Metadata Document URL. It MUST exactly equal the statement's `sub`; the authorization server MUST reject a presentation where they differ. The effective `client_id` is the statement's `sub`, and the authorization server assigns none.

The request MUST also carry the proof required by {{sender-constraint}}: client authentication under a method the statement's metadata specifies, or a DPoP proof with an attested key. A successful presentation establishes the client for the request and for the grant state derived from it ({{grant-lifecycle}}).

At the token endpoint, a runtime presentation is valid only on a request that can open a new grant or directly issue access under a grant not already bound to an establishment. Authorization-code redemption and refresh-token use continue an existing grant and follow {{grant-lifecycle}} and {{refresh}} instead. A request that initiates issuance under {{ISSUANCE}}, recognized by `response_type=software_statement_code` or the software statement `requested_token_type`, MUST NOT carry the `software_statement` parameter. A request that carries `software_statement` and is none of a runtime presentation, a refresh replacement ({{refresh}}), or a revalidation delivery ({{revalidation}}) is rejected with `invalid_request`.

An authorization server advertises support through `software_statement_presentation_supported` ({{authorization-server-metadata}}).

### Authorization Endpoint {#authorization-requests}

Presentation in an authorization request MUST use a pushed authorization request {{RFC9126}}. The statement and its proof are sent to the pushed authorization request endpoint, where the processing rules of {{processing}} apply. The subsequent authorization request MUST use a `client_id` exactly equal to the establishment's `sub` and MUST NOT include the `software_statement` parameter. A statement never appears in a front-channel URL, just as {{ISSUANCE}} keeps an issued statement out of authorization responses.

### Token Endpoint

At the token endpoint, the client includes the parameter in an eligible token request as defined by {{runtime-presentation}}.

## Presentation Processing {#processing}

On receiving a presentation, the authorization server proceeds as follows, rejecting as {{errors}} defines at the first failure:

1. Validate the statement under {{ISSUANCE}}: format, issuer trust, audience, lifetime, and namespace, with digest comparison as the policy input defined there. Complete all validation that does not require a client-controlled network retrieval before performing such a retrieval.
2. Verify the statement requirements of {{profiles}} and the `client_id` rule of {{runtime-presentation}}.
3. Verify the sender-constraint chain ({{sender-constraint}}).
4. Derive the effective metadata and evaluate the request against it ({{effective-metadata}}).
5. On success, create the establishment ({{grant-lifecycle}}).

## Sender Constraint {#sender-constraint}

A runtime presentation MUST be sender-constrained by a key the statement attests. The presenter proves possession of that key through the applicable client authentication method or a DPoP proof {{RFC9449}}, and the proof MUST be bound to the current request and validated with the replay protections of that mechanism.

The proven key MUST appear in the attested `jwks` or at the attested `jwks_uri`. Where the effective metadata specifies a client authentication method, the presenter MUST use it, and where the grant type requires client authentication a DPoP proof does not satisfy that requirement ({{RFC9449}}). The authorization server MUST reject a presentation without such a proof, or whose proven key the statement does not attest.

A statement that attests no key material cannot be presented at runtime and is consumable only through registration ({{dcr-presentation}}). Endorsement of a key the statement does not name, by a client attester or by an issuer the statement delegates to, is left to extensions ({{extensions}}).

Verifying a key at the attested `jwks_uri` is a retrieval at presentation time. A fetch failure leaves the key unverified, and the presentation is rejected as `invalid_client`. A server MAY reuse a recently retrieved key set within ordinary HTTP caching bounds, subject to a maximum reuse period of its own choosing; it MUST NOT let the client's cache directives alone determine how long a removed key continues to verify ({{external-retrieval}}).

## Effective Metadata {#effective-metadata}

Having validated the statement and its proof, the authorization server derives the client's effective metadata for the request:

* An attested member has the value the statement gives it, with the precedence {{RFC7591}} defines, the issuer's scope ({{ISSUANCE}}) being what bounds which clients it may attest at all.
* A member the statement does not attest takes its value from the client's current Client ID Metadata Document, the document at the statement's `sub`, or from a default defined by {{RFC7591}} where that default is compatible with {{CIMD}} and this runtime model. Such a member is client-asserted metadata, not part of the review. The authorization server MUST resolve that document when the request depends on an unattested member. In particular, a shared-secret authentication method cannot be inferred because no client secret is assigned by presentation and {{CIMD}} does not support shared-secret client authentication.

The effective metadata is therefore attested member by member, not wholesale. The authorization server MAY require specific members, notably the redirection URIs of a redirect flow, to be attested, rejecting a presentation whose statement omits them.

The request is evaluated against the effective metadata: a `redirect_uri` MUST match an effective redirection URI, and any requested grant type, response type, or scope MUST fall within the effective metadata. A grant or response type supported by the authorization server but not authorized by the effective metadata fails with `unauthorized_client`; a scope outside the effective metadata fails with `invalid_scope`.

A presentation refused because a bound of {{multi-instance}} is reached is rejected with `invalid_client`; the statement and its proof are sound, so the authorization server SHOULD say so in `error_description`, and the client can retry once earlier establishments are released.

## Grant Lifecycle {#grant-lifecycle}

A successful presentation creates an establishment comprising the following, which is the state a server persists for the grant:

* the validated `sub`;
* the statement identity, its `iss`, `jti`, `iat`, and expiry;
* the tenant the grant was opened for, where the authorization server hosts more than one;
* the effective metadata ({{effective-metadata}});
* the issuer trust decision; and
* the sender-constraint mechanism and Proven Key.

The establishment persists for as long as the grant depends on it: it is created by the presentation, referenced by the grant continuation state bound to it, and MAY be discarded once no `request_uri`, authorization code, refresh token, or other continuation state references it. Discarding an establishment ends nothing the grant still holds; it is the removal of state no longer reachable. The authorization server MUST bind the resulting `request_uri`, authorization code, refresh token, and other grant continuation state to the establishment as applicable. A token request that redeems an authorization code MUST have a `client_id` exactly equal to the establishment's `sub`, MUST demonstrate possession of the same Proven Key under the same sender-constraint mechanism, and MUST NOT carry the `software_statement` parameter. A redemption carrying a statement is rejected with `invalid_request`; a wrong client identifier or failed key binding is rejected with `invalid_grant`. This prohibition covers redemption of a code bound to an establishment; a registered client redeeming its own code may deliver a replacement statement under {{revalidation}}, which is a delivery rather than a presentation.

A statement MUST be unexpired when presented. Expiry after presentation does not by itself invalidate an establishment already bound. The authorization server controls continued use through its grant and refresh-token policy; {{refresh}} defines how it can require a current replacement statement.

### Refresh {#refresh}

On refresh-token use the authorization server MUST verify possession of the establishment's Proven Key under the same sender-constraint mechanism. It MAY, by local policy, additionally require a current unexpired statement, and SHOULD require one once the establishment's recorded statement has expired ({{enforcement-bounds}}).

An authorization server that holds a record refusing a statement, such as one kept under a withdrawal ({{SIGNALS}}), MUST treat that statement as not current wherever this section requires currency, so that a withdrawal ends grant continuation on the same terms as an expiry.

When policy requires one, the client presents the replacement in the `software_statement` parameter of the refresh request. The replacement:

* MUST validate under {{ISSUANCE}} with this server in its audience;
* MUST have the establishment's `iss` and `sub`;
* MUST have an `iat` no earlier than the recorded statement's `iat`; and
* MUST authorize the establishment's Proven Key ({{sender-constraint}}).

The refreshed access MUST fall within the replacement's effective metadata; the grant never widens beyond the original authorization. The authorization server MUST recompute the attested members from the replacement. It MAY re-resolve the Client ID Metadata Document for unattested members, in which case their current values apply; otherwise the recorded values persist for the grant. This operation does not rotate the establishment's key; a client that needs a new key performs a new presentation. On success, the establishment's statement identity, `iat`, expiry, effective metadata, and trust decision are replaced in a single atomic update; concurrent deliveries resolve to the most recently issued statement. A refresh that fails these requirements, or omits a statement that policy requires, is rejected with `invalid_grant` and leaves the establishment unchanged.

## Statements from an Established Client {#registered-delivery}

A client already established at an authorization server, whether registered through {{RFC7591}} or registered under its Client ID Metadata Document URL as its `client_id`, can still carry a statement. From the issuer governing its registration, the statement is a delivery: it renews validity under {{registration-validity}} and supplies no effective metadata for the request. Runtime presentation establishes clients the server does not have; it does not reopen metadata for a client it does. A server that wishes attested changes to reach the registration applies them through its registration policy ({{revalidation}}) or through {{RFC7592}}, where a narrower record survives until it does. A statement from an issuer the server does not accept for that client is rejected as {{errors}} defines.

A registration created from a statement is statement-governed when the server advertises `software_statement_registration_validity_supported`. The validity and revalidation model of {{registration-validity}} and {{revalidation}} applies, with the delivered statement's `sub` equal to the registered `client_id`. The request authenticates as the registered client under the registration's own method; the delivered statement renews validity and does not otherwise alter the registration. Where the registration is still valid and the server requires a current statement, a refresh-token request that omits one or delivers one failing these rules is rejected with `invalid_grant`. Where the registration has already expired, {{revalidation}} governs and the rejection is `invalid_client` at every endpoint.

# Registrations Derived from One Statement {#multi-instance}

A statement attests client software, identified by `sub`; it does not identify the runtime instances of that software, and this specification defines no instance identifier.

One unexpired statement is therefore consumable more than once: at each trusting authorization server in its audience and, where local policy permits, in more than one registration at the same authorization server, for example one registration per deployment or tenant. A trusting authorization server SHOULD use the statement's `sub`, `iss`, and `jti` to inventory the registrations and establishments derived from it, and SHOULD bound their number at one local audience. The safe default is one registration per (`iss`, `sub`, `jti`, local audience). On repeated consumption, local policy can reject the request, treat it as idempotent, or create another registration; {{RFC7591}} defines no duplicate-registration protocol.

Where a statement attests `jwks` or `jwks_uri`, every registration derived from it uses that attested key material rather than an instance-supplied replacement, and that same key is what a runtime presentation proves ({{sender-constraint}}). Software whose instances hold their own keys registers per instance, or waits on the endorsement extension of {{extensions}}.

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

A statement consumed at registration is rejected with the {{RFC7591}} error codes as {{ISSUANCE}} defines. A rejected presentation or delivery uses the error responses of {{RFC6749}} for the endpoint at which it was presented. At the token endpoint:

`invalid_client`:
: the statement or its proof fails to establish the client, including failed statement requirements ({{profiles}}), chain ({{sender-constraint}}), or `jwks_uri` retrieval; also any request under an expired statement-governed registration that does not restore it ({{revalidation}}), at every endpoint including the refresh-token grant.

`unauthorized_client`:
: the client is not permitted by its effective metadata to use the requested grant type or response type, although the authorization server supports it.

`invalid_scope`:
: a requested scope falls outside the effective metadata.

`invalid_request`:
: any other rejection, including a redemption or issuance request carrying the `software_statement` parameter, and a request repeating the parameter ({{runtime-presentation}}, {{grant-lifecycle}}).

At the pushed authorization request endpoint, the corresponding {{RFC9126}} error responses apply. On refresh-token use of a runtime-established grant, {{refresh}} takes precedence: a missing or failing replacement statement is `invalid_grant`, the grant's continuation failing rather than client establishment. Registration validity is reported differently, as `invalid_client` under {{revalidation}}, because the registration rather than the grant is what lapsed; neither rejection indicates refresh-token replay ({{RFC9700}}).

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

The authorization server validates the statement, retrieves the attested `jwks_uri`, verifies the client assertion against a key found there, and applies the effective metadata to the request: the requested `scope` falls within it, and the request authenticates as the client the statement describes. It keeps no registration; the effective `client_id` is the statement's `sub`.

# Authorization Server Metadata {#authorization-server-metadata}

This specification defines the following authorization server metadata {{RFC8414}} values:

`software_statement_presentation_supported`:
: OPTIONAL. Indicates whether the authorization server accepts a software statement presented at runtime ({{runtime-presentation}}). The value is either a boolean or a JSON array of the endpoints at which presentation is accepted, whose members are `token` and `pushed_authorization_request`. A value of `true` is equivalent to an array naming every endpoint the server's supported grant types make applicable, and `false`, or omission, means the path is not offered. Advertising only `token` is what lets a server offer presentation to clients that need no redirect, without also promising the front-channel path. This member describes the consuming role. It does not imply acceptance of any particular statement issuer or subject namespace, and it does not imply support for statement-governed registrations. An authorization server advertising presentation at the pushed authorization request endpoint MUST publish `pushed_authorization_request_endpoint`, since {{authorization-requests}} makes presentation there the only front-channel path. A client also examines the ordinary client-authentication and DPoP metadata for the proof it intends to use.

`software_statement_registration_validity_supported`:
: OPTIONAL. Boolean value indicating whether every registration the authorization server creates from a validated software statement is governed by the validity and revalidation rules of {{registration-validity}} and {{revalidation}}. If omitted, the default value is `false`. A value of `true` does not imply runtime-presentation support. It tells a client that the statement's `exp` will bound the registration and that the server accepts replacement delivery through an authenticated token request or pushed authorization request and, if the server supports {{RFC7592}}, a registration update request. A server MAY apply the model to registrations created from statements of some configured issuers and not others, since a client learns the boundary that applies to its own registration from the `exp` of the statement it delivered; advertising `true` claims conformance for the registrations the model governs, not that every registration at the server is statement-governed.

# Extension Points {#extensions}

Three extensions of this path are left to separate specifications. Endorsed keys: a client attestation {{ABCA}}, or an assertion from an issuer named by an attested `instance_issuers` delegation {{CLIENT-INSTANCE}}, vouching for a key the statement does not itself attest, which would admit software whose instances hold their own keys. Statement conveyance within a client attestation, rather than as a request parameter. And a Client ID Metadata Document that references where the client publishes its current statements, so a resolving server fetches the review out of band, which differs from embedding a statement in the document as the digest rule of {{ISSUANCE}} forbids.

# Security Considerations {#security-considerations}

## Statement Theft and Replay {#statement-validation}

A runtime presentation resists statement theft because the proof is a key the statement attests. Possession of a stolen statement is insufficient unless the attacker also controls that key, and admitting only that key is what prevents downgrade: every server in the audience either binds the presenter to it or refuses the presentation. An extension admitting endorsed keys ({{extensions}}) reopens that question and needs to answer it in its own terms.

A statement consumed at registration is a reusable bearer artifact until it expires, so {{ISSUANCE}} relies on narrow audience and lifetime, and the bounds of {{multi-instance}} limit what a stolen statement can create.

Attesting `jwks_uri` attests the location, not its contents: a compromised key host can add keys that satisfy the proof with no digest change, so where that exposure matters an issuer attests `jwks` inline and accepts digest-visible rotation. A server reusing a cached key set under {{sender-constraint}} additionally accepts that a just-removed key can briefly continue to verify.

Nothing in this specification relaxes the validation rules of {{ISSUANCE}}; it adds the sender constraint, the grant bindings of {{grant-lifecycle}}, and the registration-validity model on top of them.

## Renewal Authenticates the Credential

Registration renewal proves possession of the registration's own credential and the currency of a statement sharing the governing `iss` and `sub`. It does not prove that the renewing party is the reviewed software: an attacker holding a stolen client credential can renew indefinitely with any current statement for that software, which circulates by design to every deployment of it. Renewal keeps the review current, not the credential honest. Deployments SHOULD pair statement-governed registrations with credential rotation, sender-constrained client authentication, and the registration limits {{ISSUANCE}} recommends, and SHOULD treat a credential compromise as requiring re-registration rather than renewal. The runtime profile does not share this gap, because the sender-constraint chain binds the presenter to the review at every presentation.

## External Retrieval and Resource Exhaustion {#external-retrieval}

Runtime presentation can cause the authorization server to retrieve the Client ID Metadata Document, an attested `jwks_uri`, or keys and metadata associated with an attested instance issuer. Every such retrieval inherits the server-side request forgery protections of {{CIMD}}. A trusted signature does not make a URL safe: the authorization server MUST apply its URL, redirect, address-range, transport, and content-type policy independently to every referenced location.

Presentation reaches these retrievals before any client is registered or any user has interacted, so the work is available to an unauthenticated requester holding one acceptable statement. An authorization server SHOULD rate-limit presentations per statement identity, per subject, and per source, and SHOULD bound the establishments it will create from one statement ({{multi-instance}}), before spending retrieval or storage on a new presentation. It SHOULD bound JWT size and parsing work, concurrent retrievals, response size, and response time, and SHOULD cache successful and failed retrieval results for an appropriate period. A retrieval failure leaves the relevant metadata or proof chain unverified; the authorization server MUST reject the request and MUST NOT fall back to a weaker sender-constraint mode.

## Registration Fraud and Impersonation {#registration-fraud}

Open registration permits `client_name`, `logo_uri`, and `client_uri` values that imitate trusted software on consent screens. Requiring a statement replaces self-asserted branding with issuer-reviewed values. Servers that render registration-supplied values on consent screens SHOULD prefer attested values and SHOULD apply heightened scrutiny to unattested registrations that claim user-visible branding.

Statement-gated registration also makes each rotated identity require another issuer decision, rather than letting a discarded client return at no cost; `sub` and `jti` tracking bounds registrations ({{multi-instance}}). Neither control makes metadata true: a client that misleads review can obtain a genuine statement for fraudulent metadata, so issuer verification depth remains decisive ({{ISSUANCE}}).

## Enforcement Bounds {#enforcement-bounds}

Expiry is enforced at every presentation and every statement-governed registration, so a lapsed statement stops new runtime establishment and causes registration-backed requests to fail at the recorded `exp`. It does not retroactively invalidate an establishment, revoke an access token, or terminate an outstanding grant. Requiring a current statement on refresh under {{refresh}} is the control that makes issuer non-renewal end runtime-established grants, and a deployment that does not adopt it retains grants for the life of their refresh tokens whatever the statement lifetime; registration-backed grants remain subject to the server's grant policy after the registration expires. A narrowed re-review takes effect through replacement effective metadata or an updated registration. Detection of post-issuance metadata change rests on `cimd_digest`, which covers exact bytes but requires retrieval for comparison. The bounded statement lifetime limits what either signal can miss for new establishment. {{SIGNALS}} defines an optional event mechanism by which an issuer ends a decision before its expiry; expiry remains the floor, and this specification does not depend on it.

Renewal cadence is a deployment trade: short lifetimes tighten the issuer's control loop and increase issuance and delivery traffic, and a fleet of registrations issued together expires together, so issuers SHOULD stagger expiries or renew ahead of the boundary to avoid synchronized lapses.

## Client-Asserted Metadata

Members the statement does not attest are client-asserted, and omission can mean the issuer declined to attest a value. The attested-members-required policy of {{effective-metadata}} is the control for deployments that do not want client-asserted values, notably redirection URIs, entering effective metadata. Resolving a Client ID Metadata Document for unattested members inherits the resolution considerations of {{CIMD}}, including server-side request forgery and availability; a server MAY cache resolution results within the document's caching directives, and the statement's digest binds the review to specific bytes regardless of cache state.

## Observable State {#oracle-considerations}

A retained expired registration is distinguishable from an unknown client, because recovery requires the server to authenticate the registration and evaluate a replacement before rejecting. That disclosure is deliberate and bounded: it is available only to a requester that authenticates as the registration, so it reveals to the legitimate client the state it must act on. Servers publishing `software_statement_registration_validity_supported` additionally disclose that statement-derived registrations expire there, which is configuration a client needs before registering. Neither discloses issuer trust, subject scope, or attester policy, which {{errors}} keeps from unauthenticated requesters.

## Statement Handling

A software statement remains a sensitive artifact in transit and at rest: possession alone does not enable presentation, but statements reveal attested metadata and audience relationships, and servers SHOULD avoid logging them.

# Privacy Considerations

A presentation or delivery reveals to the authorization server the client's issuer relationship and the full attested metadata, including the audience list, which names the other authorization servers the client intends to establish relationships with. Issuers can bound the disclosure by keeping audiences narrow, as {{ISSUANCE}} recommends. The pushed authorization request requirement of {{authorization-requests}} keeps statements out of browser history, referrers, and front-channel logs. A central issuer additionally learns, through renewal requests, which of its statements are in active use; issuance and renewal logs deserve the same care as the statements themselves.

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

## OAuth Authorization Server Metadata Registry

This specification requests registration of the following values in the IANA "OAuth Authorization Server Metadata" registry established by {{RFC8414}}.

Metadata Name:
: `software_statement_presentation_supported`

Metadata Description:
: Boolean value, or JSON array of endpoint names, indicating whether and where the authorization server accepts a software statement presented at runtime.

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
