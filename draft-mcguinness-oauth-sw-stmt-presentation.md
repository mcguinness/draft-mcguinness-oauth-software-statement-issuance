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
  AU-CDR:
    target: https://consumerdatastandardsaustralia.github.io/standards/
    title: "Consumer Data Standards (Australia)"
  OPENID-FED:
    target: https://openid.net/specs/openid-federation-1_0.html
    title: "OpenID Federation 1.0"
  STATUS-LIST:
    target: https://datatracker.ietf.org/doc/draft-ietf-oauth-status-list
    title: "Token Status List"
  TRUST-FRAMEWORK:
    target: https://datatracker.ietf.org/doc/draft-mcguinness-oauth-id-assertion-framework
    title: "OAuth Identity Assertion Trust Framework"
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

RFC 7591 defines the software statement as input to dynamic client registration but does not define how long the resulting registration remains valid or how a client renews the statement on which it was based. This specification defines two consumption profiles. In the registration profile, a statement bound to an RFC 7591 `software_id` governs the registration until the statement expires and a replacement renews that validity. In the runtime profile, a statement bound to a Client ID Metadata Document URL establishes an otherwise unregistered client for an authorization or token request and the grant state derived from it. Runtime presentation requires proof of a key directly attested by the statement or covered by an accepted endorsement chain, so possession of the statement alone is insufficient. Together, the profiles let an issuer curate approved client software across the authorization servers in a statement's audience while preserving each server's control over trust, metadata authority, grants, and token lifetime.

--- middle

# Introduction

{{RFC7591}} defines no standard expiry or renewal procedure for a dynamic client registration. A software statement ({{RFC7591}}, Section 2.3) can carry a reviewer's approval into a registration request, but the registration can outlive the statement and the review it represents. An organization that reviews client software therefore has no interoperable way to keep that review current at the authorization servers that relied on it.

This specification defines two ways to consume the statement format and validation rules of {{ISSUANCE}}:

* **At registration** ({{dcr-presentation}}): the statement is consumed in an {{RFC7591}} registration request and at renewal, and its `exp` bounds the registration's validity ({{registration-validity}}).
* **At runtime** ({{cimd-presentation}}): the client presents the statement in an authorization or token request, proves a key the statement attests, and is established for the resulting grant without creating a persistent registration ({{runtime-presentation}} and {{grant-lifecycle}}).

Establishment is one layer of the decision to let a client act. An authorization server hosting many customers separates it from tenant approval, the customer's own decision about which established clients may operate in its tenant ({{tenant-approval}}); both sit above the sender-constraint proof that identifies the presenter and the grant that carries a user's authorization. Each layer has its own decider, artifact, and lifecycle, and this specification keeps them apart rather than collapsing them into one decision.

In both profiles, ceasing statement renewal stops new establishment after the applicable expiry. It also stops continued use of an existing registration or grant where this specification requires a current statement. It does not revoke access tokens already issued, and continuation of runtime-established grants is subject to the refresh policy in {{refresh}}. These enforcement bounds are detailed in {{enforcement-bounds}}.

This separation lets one issuer curate approved software across many authorization servers without making the issuer the final policy authority. Each server independently configures issuer trust, subject scope, authoritative metadata, grant policy, and token lifetime. An enterprise operating a statement issuer is the motivating deployment ({{deployment-model}}).

Statement format, issuance, and validation are defined by {{ISSUANCE}} and are not modified here; the profiles in this document state which elements each consumption model requires, and how the client acquired its statement is out of scope. A statement authorizes metadata, not its presenter. Runtime proof of the presenter is what the sender-constraint rules of {{sender-constraint}} supply, and nothing in this specification attests software instances or binaries.

## Protocol Overview

The following non-normative sequences summarize the two models.

Registration governed by a statement (DCR profile):

1. The client registers through {{RFC7591}}, carrying a DCR-profile statement in the `software_statement` member; the statement binds to the registration by `software_id` and, when attested, `software_version` ({{ISSUANCE}}).
2. The authorization server records the statement's identity and `exp` with the registration; the registration is valid until that expiry.
3. The client delivers a replacement statement in an authenticated token request or, where supported, an {{RFC7592}} update request; the registration's validity extends to the replacement's `exp`.
4. If no replacement arrives, the registration expires and requests under it fail. This does not revoke tokens already issued or determine the disposition of outstanding grants.

Runtime presentation (CIMD profile):

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

The two therefore compose. A deployment holding only a client attestation knows what is running but not whether anyone approved it; a deployment holding only a software statement knows the software was reviewed but not that this sender is running it. Runtime presentation always requires both halves: the statement carries the review, and a proof carries the presenter, supplied either by a key the statement itself attests or, where it attests none, by a client attestation or instance assertion ({{sender-constraint}}).

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
: The durable state a successful runtime presentation creates for the grant it opens, enumerated in {{grant-lifecycle}}. This specification uses the term only in that sense; the establishment issuer role of {{ISSUANCE}}, and the layer it names, are the separate sense used when discussing which software may exist as a client.

Proven Key:
: The key for which the presenter demonstrates possession during runtime presentation. The accepted proof path binds this key to the statement as specified in {{sender-constraint}}.

# Statement Requirements {#profiles}

This document consumes the software statement of {{ISSUANCE}}. The syntax, validation, and semantics of every element are defined there; this section states what a statement must carry to be consumed here, and what each element is consumed for. How the client acquired it is out of scope.

`typ` header:
: `software-statement+jwt`; explicit typing prevents confusion with other JWTs from the same issuer.

`iss`:
: Identifies the issuer for trust and scope decisions, and forms part of the statement identity recorded with a registration or establishment.

`sub`:
: The client's Client ID Metadata Document URL {{CIMD}}. In a runtime presentation the request's `client_id` equals it, by the rule of {{runtime-presentation}}.

`aud`:
: Scopes which authorization servers may accept the statement; without it a statement would be usable at every server that trusts the issuer.

`iat`:
: The issuance time, which orders one statement against another when a replacement arrives ({{revalidation}}, {{refresh}}).

`exp`:
: The expiry the issuer chose. It bounds registration validity ({{registration-validity}}), is checked at every presentation, and is the issuer's continuous-governance lever: renewal extends standing, ceased renewal ends it. The effect on registrations, establishments, grants, and tokens is bounded as {{enforcement-bounds}} describes.

`jti`:
: Identifies the statement within the issuer's namespace; inventory and concurrency bounds key on the `iss` and `jti` pair. A replacement statement has its own `jti`, so replacement matching does not key on this value.

`cimd_digest`:
: Binds the issuer's review to the exact document bytes evaluated and feeds the post-issuance change policy of {{ISSUANCE}}.

A statement lacking any of these cannot be consumed under this specification, whatever a registration endpoint might otherwise accept, and one failing validation is rejected as {{errors}} defines.

# Issuer Trust Establishment {#issuer-trust}

A trusting authorization server accepts statements only from configured issuers. Trust is established out of band, for example through a marketplace publisher program or shared enterprise operation. This specification defines no in-band issuer discovery or trust decision.

Configuring trust in an issuer is a one-time act that covers every client that issuer attests, so a trusting authorization server maintains a small, stable set of trusted issuers rather than per-client state. For each, it records at least:

* the exact `iss` identifier it will accept;
* the source of that issuer's signing keys: the `jwks_uri` in the issuer's authorization server metadata {{RFC8414}}, reached from the configured `iss`;
* the signing algorithms it will accept from the issuer;
* the client identifier namespaces the issuer may attest through `sub`;
* the audience identifiers the issuer may name;
* the maximum statement lifetime it will honor, which also caps registration validity where the server implements the registration-validity model of {{registration-validity}};
* the role in which it accepts the issuer ({{issuer-roles}}); and
* its policy on repeated and multiple registration ({{multi-instance}}).

These inputs, not the signature alone, define acceptance. Because the issuer is an authorization server role, key retrieval reuses {{RFC8414}} discovery ({{authorization-server-metadata}}).

A trusting authorization server MUST derive trust from this local configuration and MUST NOT derive it from an `iss`, `jku`, `x5u`, or other key-location value carried in a presented statement. Having established trust, it validates each statement as described in {{ISSUANCE}}.

Issuer trust SHOULD be scoped as well as explicit. An issuer accepted for all values of `sub` can, if compromised or over-broad, mint acceptable statements about any client software; trust configuration SHOULD therefore constrain each issuer to the client identifier namespaces it is expected to attest, for example URLs under the domains of the software publishers it serves, and a statement whose `sub` falls outside that scope MUST be rejected even when its signature verifies ({{ISSUANCE}}). Where an issuer attests software across many publishers, as an enterprise issuer does, the scope is the set of identifiers it is configured for rather than a single domain; the rejection rule is the same.

## Issuer Roles {#issuer-roles}

A trusting authorization server accepts an issuer in one of two roles, and records which when it configures the issuer. The roles answer different questions, are held by different parties, and run on separate lifecycles.

Establishment issuer:
: Its statements decide which software may exist as a client at this authorization server. A marketplace publisher program or an ecosystem directory holds this role. Statements from an establishment issuer are consumed at registration or at runtime establishment, and govern registration validity where this specification applies that model. One such decision serves every tenant the authorization server hosts.

Tenant approval issuer:
: Its statements decide which software a particular tenant permits. The tenant's own review function holds this role, and the authorization server records the tenant the issuer speaks for. An issuer in this role serves its current decisions rather than handing them to clients to carry ({{approvals-endpoint}}), because withdrawing an approval is the point of holding one and the restricted party must not control its conveyance. A tenant approval statement is recorded at the authorization server by the tenant it governs and evaluated in that tenant's request context under this specification; it neither creates nor renews a registration, and it bears on no other tenant.

An issuer MAY hold both roles, and the two consumptions never collide: an establishment statement arrives in a request, a tenant approval is recorded out of band. For the events of {{SIGNALS}}, the event type determines scope.

Where an authorization server hosts multiple tenants, issuer trust is configured per tenant. The authorization server MUST resolve the tenant a request belongs to and MUST evaluate only the issuers configured for that tenant; accepting a tenant approval issuer's statement outside the tenant it speaks for is a cross-tenant escalation ({{security-considerations}}).

## Serving Current Approvals {#approvals-endpoint}

An issuer in the tenant approval role serves the statements it currently has in force to the authorization servers they name. A request carries the requesting authorization server's issuer identifier as the audience of interest; the response is a JSON object whose `software_statements` member is an array of the statements in force naming that audience, and whose HTTP caching headers bound how long the response may be reused. An empty array is a well-formed answer, and means the issuer has no approvals in force for that audience.

Absence is how an approval ends. An issuer withdrawing an approval stops serving it, and a consuming authorization server that no longer receives it treats the approval as it treats an expired one (this specification). This is why the endpoint serves the set in force rather than answering questions about individual subjects: a per-subject query cannot distinguish a withdrawn approval from one the issuer never made.

The endpoint is authenticated. The two parties already exchange issuer identifiers and keys when trust is established ({{issuer-trust}}), and the credential a requesting authorization server presents is part of that configuration; this document defines no new credential type. An issuer MUST serve only the statements naming the requester's own audience, and MUST NOT reveal statements naming other authorization servers.

An issuer supporting this endpoint advertises it as `software_statement_approvals_endpoint` ({{authorization-server-metadata}}). An issuer in the establishment role does not serve statements this way: those are carried by the clients they admit, and serving them on request would disclose which authorization servers a client intends to establish relationships with.

Pairwise configuration bounds a statement's reach to the issuers a trusting authorization server has configured, which is why a statement's practical audience is an ecosystem or administrative domain rather than the open web. The OAuth Identity Assertion Trust Framework {{TRUST-FRAMEWORK}} generalizes the model: a trusting authorization server publishes the conditions an issuer must satisfy and evaluates published evidence, such as authorization by the owner of a client identifier's namespace, when a statement is presented, replacing enumeration of trusted issuers with open-world policy. OpenID Federation {{OPENID-FED}} provides an alternative through trust chains. Both are out of scope here.

# Presentation at Registration {#dcr-presentation}

Under the DCR profile, the statement is consumed in the `software_statement` member of an {{RFC7591}} registration request. The authorization server validates and binds it as {{ISSUANCE}} requires, with rejections using the {{RFC7591}} error codes defined there. This section defines the registration's relationship to the statement's lifetime.

## Registration Validity {#registration-validity}

An authorization server that advertises `software_statement_registration_validity_supported` as `true` MUST apply this model to every registration it creates from a validated software statement. It MUST record the governing statement's `iss`, `jti`, `sub`, `iat`, and `exp` with the registration, together with any attested `software_version`, which {{version-changes}} compares against a replacement. The registration is valid until that `exp`. After it passes without a replacement ({{revalidation}}), the authorization server MUST reject requests under the registration, using `invalid_client` at the token endpoint and, at the authorization or pushed authorization request endpoint, the error {{RFC6749}} defines for an unauthorized client, except for the revalidation requests {{revalidation}} permits. The server SHOULD retain the expired record so that it can process a later authenticated revalidation, and MAY allow a grace period during which it accepts a replacement without treating the registration as expired. An expired registration is not an unknown client: the difference is observable to a client that authenticates, which is what makes recovery possible ({{oracle-considerations}}). Registration expiry does not revoke tokens already issued; the disposition of outstanding grants is local policy.

The metadata signal in {{authorization-server-metadata}} lets a client determine before registration whether this model applies. The client already holds the statement and therefore learns the initial validity boundary from its `exp`. A server that omits the signal or advertises `false` can still consume a statement as ordinary {{RFC7591}} registration input, but it MUST NOT claim conformance to this registration-validity model.

## Revalidation {#revalidation}

The client renews a statement-governed registration by delivering a replacement statement, in any of these ways:

* in the `software_statement` parameter of a token request under the registration, authenticated as the registered client under the registration's own method;
* in the `software_statement` parameter of an authenticated pushed authorization request {{RFC9126}} under the registration, which is the renewal path available to a client that holds no refresh token and whose only grant type is the authorization code; or
* in the `software_statement` member of an authenticated {{RFC7592}} update request, where the deployment offers registration management. Such a request replaces the registration's metadata in full, as {{RFC7592}} requires, so the client sends its complete current metadata alongside the statement; a renewal-only request omitting other members would reset them.

The replacement MUST validate under {{ISSUANCE}} with this server in its audience, MUST have the governing statement's `iss` and `sub`, MUST be unexpired, and MUST have an `iat` no earlier than the recorded statement's `iat`. Recency rather than a longer lifetime is the ordering rule, so an issuer downgrading a review can deliver a deliberately short-lived narrowed replacement, while a client cannot roll back to an older, broader statement it still holds. On success the authorization server MUST replace the recorded statement identity, `iat`, and `exp` in a single atomic update; concurrent deliveries resolve to the most recently issued statement. Whether attested changes in the replacement update the registration record is local registration policy, and a server that relies on narrowing to take effect applies them.

When an expired registration sends a request containing a replacement, the authorization server MUST authenticate the retained registration and evaluate the replacement before applying the expiry rejection. A valid replacement therefore restores the registration; an omitted or invalid replacement does not.

A request under an expired registration that does not restore it is rejected with `invalid_client`, at every endpoint, including the refresh-token grant. The rejection reports the registration's state, not a defect in the grant, so it does not indicate refresh-token replay and MUST NOT by itself trigger the refresh-token family revocation of {{RFC9700}}; a client recovers by delivering a valid replacement.

A delivery that fails the rules above under a still-valid registration leaves the recorded statement unchanged and does not fail the request it accompanies; the authorization server SHOULD signal the declined renewal in `error_description` on a subsequent rejection or through its registration management interface, and the client can distinguish success by the registration's continued validity past the recorded `exp`. A registration request without an {{RFC7592}} registration access token creates a new registration and never renews an existing one.

## Version Changes {#version-changes}

A review covers the software the issuer evaluated, and `software_version` is the DCR profile's change signal: its value changes on any update to the software. When the statement attests it, the binding rules of {{ISSUANCE}} reject a registration request whose `software_version` differs. A replacement delivered outside a registration request carries no such member, so the authorization server MUST instead compare the replacement's attested `software_version` with the value recorded for the registration: a replacement attesting a different version is a version change, not a renewal. The authorization server MUST reject it as a renewal, and MAY instead treat it as a new registration decision under its registration policy, updating the recorded version if it accepts one. A new version therefore requires a new review and a statement covering it. A version string is a vendor-asserted label, not a byte binding; the guarantee is review-of-what-the-vendor-calls-that-version.

The CIMD profile's change signal is `cimd_digest`, which is byte-exact: a mismatch against the currently published document is a policy input under {{ISSUANCE}}, and re-issuance against the new document is the remedy. In both profiles the statement's bounded lifetime caps how stale a review can get: drift the change signal misses still expires with the statement.

# Runtime Presentation {#cimd-presentation}

Under the CIMD profile, the client presents its statement in the request. The authorization server validates it under policy for a trusted issuer, verifies the presenter, and applies the effective metadata to that request without creating a persistent client registration. This establishes an otherwise unregistered client at request time, as {{CIMD}} resolution does, with the issuer's review carried in the statement.

## Presentation Request {#runtime-presentation}

A client presents a software statement by including the following parameter in a token request or a pushed authorization request:

`software_statement`:
: REQUIRED for presentation. The software statement ({{profiles}}). It is consumed for establishment: a runtime presentation, a refresh replacement ({{refresh}}), or a revalidation delivery ({{revalidation}}). Tenant approvals are recorded rather than carried ({{tenant-approval}}), so one statement parameter serves every request this specification defines. The authorization server MUST verify that it accepts the statement's issuer in the establishment role ({{ISSUANCE}}), and MUST reject a request repeating the parameter with `invalid_request`.

The request's `client_id` is the client's Client ID Metadata Document URL. It MUST exactly equal the statement's `sub`; the authorization server MUST reject a presentation where they differ. The effective `client_id` is the statement's `sub`, and the authorization server assigns none.

The request MUST also carry the proof required by {{sender-constraint}}: client authentication under a method the statement's metadata specifies, or a DPoP proof with an attested key. A successful presentation establishes the client for the request and for the grant state derived from it ({{grant-lifecycle}}).

At the token endpoint, a runtime presentation is valid only on a request that can open a new grant or directly issue access under a grant not already bound to an establishment. Authorization-code redemption and refresh-token use continue an existing grant and follow {{grant-lifecycle}} and {{refresh}} instead. A request that initiates issuance under {{ISSUANCE}} MUST NOT carry the `software_statement` parameter; the authorization server rejects the combination with `invalid_request`. Such a request is recognized before validation by its issuance signals: `response_type=software_statement_code` at the authorization endpoint, and the software statement `requested_token_type` at the token endpoint ({{ISSUANCE}}). A request that carries `software_statement` and is none of a runtime presentation, a refresh replacement ({{refresh}}), or a revalidation delivery ({{revalidation}}) is rejected with `invalid_request`. An issuer holding both roles ({{ISSUANCE}}) creates no ambiguity here, because a tenant approval never arrives on a request.

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

A statement that attests no key material cannot be presented at runtime and is consumable only through registration ({{dcr-presentation}}). Endorsement of a key the statement does not name, by a client attester or by an issuer the statement delegates to, is left to extensions ({{extensions}}): admitting it in the core would make the accepted proof depend on configuration a client cannot observe, and would let a presenter reach for a weaker proof than the one the issuer reviewed.

Verifying a key at the attested `jwks_uri` is a retrieval at presentation time. A fetch failure leaves the key unverified, and the presentation is rejected as `invalid_client`. A server MAY reuse a recently retrieved key set within ordinary HTTP caching bounds, subject to a maximum reuse period of its own choosing; it MUST NOT let the client's cache directives alone determine how long a removed key continues to verify ({{external-retrieval}}).

## Effective Metadata {#effective-metadata}

Having validated the statement and its proof, the authorization server derives the client's effective metadata for the request:

* An attested member has the value the statement gives it, with the precedence {{RFC7591}} defines, the issuer's scope ({{ISSUANCE}}) being what bounds which clients it may attest at all.
* A member the statement does not attest takes its value from the client's current Client ID Metadata Document, the document at the statement's `sub`, or from a default defined by {{RFC7591}} where that default is compatible with {{CIMD}} and this runtime model. Such a member is client-asserted metadata, not part of the review. The authorization server MUST resolve that document when the request depends on an unattested member. In particular, a shared-secret authentication method cannot be inferred because no client secret is assigned by presentation and {{CIMD}} does not support shared-secret client authentication.

The effective metadata is therefore attested member by member, not wholesale. The authorization server MAY require specific members, notably the redirection URIs of a redirect flow, to be attested, rejecting a presentation whose statement omits them.

The request is evaluated against the effective metadata: a `redirect_uri` MUST match an effective redirection URI, and any requested grant type, response type, or scope MUST fall within the effective metadata. A grant or response type supported by the authorization server but not authorized by the effective metadata fails with `unauthorized_client`; a scope outside the effective metadata fails with `invalid_scope`.

A trusting authorization server SHOULD use the statement's `sub`, `iss`, and `jti` to inventory the establishments derived from one statement and SHOULD bound their concurrent number, as {{ISSUANCE}} bounds registrations derived from one statement. A presentation refused because that bound is reached is rejected with `invalid_client`; the statement and its proof are sound, so the authorization server SHOULD say so in `error_description`, and the client can retry once earlier establishments are released.

## Grant Lifecycle {#grant-lifecycle}

A successful presentation creates an establishment comprising the following, which is the state a server persists for the grant:

* the validated `sub`;
* the statement identity, its `iss`, `jti`, `iat`, and expiry;
* the tenant the grant was opened for, where the authorization server hosts more than one;
* the tenant the grant was opened for, where the tenant requires an approval ({{tenant-approval}}), the approval itself being evaluated afresh rather than frozen into the establishment;
* the effective metadata ({{effective-metadata}});
* the issuer trust decision; and
* the sender-constraint mechanism and Proven Key.

The establishment persists for as long as the grant depends on it: it is created by the presentation, referenced by the grant continuation state bound to it, and MAY be discarded once no `request_uri`, authorization code, refresh token, or other continuation state references it. Discarding an establishment ends nothing the grant still holds; it is the removal of state no longer reachable. The authorization server MUST bind the resulting `request_uri`, authorization code, refresh token, and other grant continuation state to the establishment as applicable. A token request that redeems an authorization code MUST have a `client_id` exactly equal to the establishment's `sub`, MUST demonstrate possession of the same Proven Key under the same sender-constraint mechanism, and MUST NOT carry the `software_statement` parameter. A redemption carrying a statement is rejected with `invalid_request`; a wrong client identifier or failed key binding is rejected with `invalid_grant`. This prohibition covers redemption of a code bound to an establishment; a registered client redeeming its own code may deliver a replacement statement under {{revalidation}}, which is a delivery rather than a presentation.

A statement MUST be unexpired when presented. Expiry after presentation does not by itself invalidate an establishment already bound. The authorization server controls continued use through its grant and refresh-token policy; {{refresh}} defines how it can require a current replacement statement.

### Refresh {#refresh}

On refresh-token use the authorization server MUST verify possession of the establishment's Proven Key under the same sender-constraint mechanism. It MAY, by local policy, additionally require a current unexpired statement, and SHOULD require one once the establishment's recorded statement has expired, since that requirement is what makes ceased renewal end an existing grant ({{enforcement-bounds}}).

Where the grant was opened for a tenant that requires approval ({{tenant-approval}}), the authorization server MUST also evaluate that tenant's current approval on refresh, retrieved or recorded. An approval that has expired, that is absent from the current retrieval, or that a record refuses, ends the grant's continuation with `invalid_grant`. Nothing additional travels on the request, because the approval is state the server holds or fetches.

An authorization server that holds a record refusing a statement, such as one kept under a withdrawal ({{SIGNALS}}), MUST treat that statement as not current wherever this section requires currency, so that a withdrawal ends grant continuation on the same terms as an expiry.

When policy requires one, the client presents the replacement in the `software_statement` parameter of the refresh request. The replacement:

* MUST validate under {{ISSUANCE}} with this server in its audience;
* MUST have the establishment's `iss` and `sub`;
* MUST have an `iat` no earlier than the recorded statement's `iat`, so a client cannot roll back to an older, broader statement; and
* MUST authorize the establishment's Proven Key ({{sender-constraint}}).

The refreshed access MUST fall within the replacement's effective metadata; the grant never widens beyond the original authorization. The authorization server MUST recompute the attested members from the replacement. It MAY re-resolve the Client ID Metadata Document for unattested members, in which case their current values apply; otherwise the recorded values persist for the grant. Because the replacement must authorize the existing Proven Key, this operation does not rotate the establishment's key. A client that needs a new key performs a new presentation and opens a new establishment. On success, the establishment's statement identity, `iat`, expiry, effective metadata, and trust decision are replaced in a single atomic update; concurrent deliveries resolve to the most recently issued statement. A refresh that fails these requirements, or omits a statement that policy requires, is rejected with `invalid_grant` and leaves the establishment unchanged.

## Tenant Approval {#tenant-approval}

An authorization server hosting multiple tenants establishes a client once and decides separately, per tenant, whether that client may operate there. A tenant approval statement carries that decision: it is issued by the tenant's approval issuer ({{ISSUANCE}}), its `sub` identifies the established client, and the tenant records it at the authorization server.

A tenant approval is never carried on a request. An artifact that grants may be conveyed by the party it grants to, whose incentive is to convey it faithfully; an artifact that restricts must not be, because withdrawal is the point of holding it and the restricted party would control whether the server ever sees it. An approval reaches an authorization server in one of two ways.

**Retrieved.** The authorization server retrieves the tenant's current approvals from that tenant's issuer, at the location and with the credential recorded when the tenant configured the issuer ({{ISSUANCE}}). It MAY reuse a retrieved set within the caching bounds of the response and MUST NOT reuse a statement past its own `exp`. An approval absent from a retrieval is withdrawn: the authorization server MUST treat it as it treats an expired one, which is what makes ceasing to serve an approval sufficient to end it. Where a retrieval fails, the authorization server continues on what it last retrieved until those statements expire, so a failure degrades to the expiry behavior specified elsewhere rather than to a new one.

**Recorded.** The tenant supplies the statement through the authorization server's administrative interface, for deployments that cannot retrieve or have not configured it. The authorization server MUST authenticate the supplying party as an administrator of the tenant the approval will govern, and MUST verify that the statement's issuer is one it accepts in the tenant approval role for that same tenant. A statement signed by an issuer configured for another tenant MUST be rejected rather than recorded. The record holds the statement's `iss`, `sub`, `jti`, `iat`, and `exp`, and the tenant it governs.

Where both are configured for a tenant, the retrieved set governs, and a recorded approval covers only subjects the retrieval does not mention.

The authorization server MUST re-verify an approval at each use, whether retrieved or recorded, so that withdrawal of issuer trust, a change of issuer scope, or a record refusing the statement takes effect without waiting for expiry. It MUST NOT record an approval whose `iat` is earlier than that of the approval it replaces, so that a narrower decision cannot be rolled back by re-recording an older one. On expiry the authorization server MUST treat the tenant as having no approval for that client.

Tenant approval constrains the request; it does not establish the client:

* The client's identity and metadata remain those of its establishment: a tenant approval MUST NOT create a registration, alter a registration record, or extend registration validity.
* Where the statement attests metadata, that metadata narrows the request and MUST NOT widen it. The effective policy for the request is the intersection of what the establishment allows and what the tenant approved; a requested scope outside the intersection fails as {{errors}} defines.
* Expiry bears on that tenant alone. A lapsed tenant approval stops that tenant's requests and leaves the client's registration, and every other tenant, untouched.

Whether a tenant requires an approval is that tenant's policy. A tenant that expresses approval through the authorization server's own interface without an artifact needs no statement; the statement is what lets one decision be made once and honored at every authorization server where the tenant has configured its issuer. Retrieval is what keeps it current there without further administrative action.

## Statements from an Established Client {#registered-delivery}

A client already established at an authorization server, whether registered through {{RFC7591}} or registered under its Client ID Metadata Document URL as its `client_id`, can still carry a statement. From the establishment issuer governing its registration, the statement is a delivery: it renews validity under {{registration-validity}} and supplies no effective metadata for the request. Runtime presentation establishes clients the server does not have; it does not reopen metadata for a client it does. A server that wishes attested changes to reach the registration applies them through its registration policy ({{revalidation}}) or through {{RFC7592}}, where a narrower record survives until it does. A statement from an issuer the server accepts in neither role for that client is rejected as {{errors}} defines.

A registration created from a statement is statement-governed when the server advertises `software_statement_registration_validity_supported`. The validity and revalidation model of {{registration-validity}} and {{revalidation}} applies, with the delivered statement's `sub` equal to the registered `client_id`. The request authenticates as the registered client under the registration's own method; the delivered statement renews validity and does not otherwise alter the registration. Where the registration is still valid and the server requires a current statement, a refresh-token request that omits one or delivers one failing these rules is rejected with `invalid_grant`. Where the registration has already expired, {{revalidation}} governs and the rejection is `invalid_client` at every endpoint.

# Multi-Instance Client Software {#multi-instance}

A software statement attests client software, identified by `sub`; it does not attest or identify the runtime instances of that software. This specification defines no instance identifier, and instances do not obtain per-instance statements. This section concerns the registration path ({{ISSUANCE}}); under runtime presentation (this specification) the presenter proves a key that chains to the statement, an instance is identified only where that key material is per-instance, and no persistent registration is created.

A single unexpired statement is therefore intended to be presented more than once: at each trusting authorization server in its audience and, where local policy permits, in more than one registration at the same authorization server, for example one registration per deployment or tenant. Where those servers implement the registration-validity model of {{registration-validity}}, every registration derived from one statement inherits its `exp` and lapses at the same moment, which is what the staggering guidance of {{ISSUANCE}} addresses. A trusting authorization server SHOULD use the statement's `sub` and `jti` to inventory the registrations derived from a statement and to enforce any local bound on their number.

A trusting authorization server SHOULD bound the number of registrations derived from one statement at one local audience. The safe default is one registration per (`iss`, `sub`, `jti`, local audience), the same key this specification uses to inventory establishments. On repeated presentation, local policy can reject the request, treat it as idempotent, or create another registration; {{RFC7591}} defines no duplicate-registration protocol.

A server that treats a repeat as idempotent MUST NOT disclose the credentials or registration access token of the prior registration to a later presenter unless it independently authorizes that presenter.

Deployments whose client software runs many concurrent instances SHOULD register the logical client once per authorization server and differentiate instances at the token endpoint, for example with {{CLIENT-INSTANCE}} or attestation-based client authentication {{ABCA}}, rather than minting a registration per instance. The statement's key material determines which registration models it can support:

* Shared client key: if the statement contains `jwks` or `jwks_uri`, that attested value takes precedence under {{RFC7591}}, and every registration derived from the statement MUST use the attested key material rather than an instance-supplied replacement. It also fixes the runtime proof, as {{statement-validation}} describes.
* Per-instance keys: where authorization server policy is keyed on `client_id` and genuinely requires per-instance registrations, the same statement supports that model only if it omits `jwks` and `jwks_uri`; each registration then supplies its own instance key as plain metadata, subject to trusting authorization server policy and the presenter-proof guidance of {{statement-validation}}.
* Attested delegation: the statement omits key material but attests the `instance_issuers` delegation ({{attesting-instance-issuers}}), so instance keys are endorsed by an attested authority at the token endpoint instead of appearing unattested in registration metadata.

## Attesting Instance Issuers {#attesting-instance-issuers}

{{CLIENT-INSTANCE}} defines the `instance_issuers` client metadata parameter, through which a client delegates attestation of its runtime instances to named authorities. For the purposes of this specification, `instance_issuers` is client metadata like any other: it can appear in the canonical Client ID Metadata Document and be carried as an attested claim in the software statement.

A statement containing `instance_issuers` attests the instance-attestation delegation instead of presenting a self-asserted list. This avoids dependence on document availability or locally configured lists. An issuing authorization server SHOULD include `instance_issuers` in a statement only when it recognizes the member and its approval process covered the delegation the member expresses.

For example, the following claims fragment attests one instance issuer for the client:

~~~
"instance_issuers": [
  {
    "issuer": "https://workload.client.example.org",
    "jwks_uri": "https://workload.client.example.org/jwks.json"
  }
]
~~~

# Deployment Model: Centrally Curated Software {#deployment-model}

This section is non-normative.

Two review functions operate here, and they belong to different parties. A provider's marketplace decides which software may exist as a client on its platform, and a customer decides which of that software may operate in its tenant. The provider configures the marketplace's issuer in the establishment role and each customer's issuer in the tenant approval role ({{ISSUANCE}}).

A marketplace application registers once. Its listing is an establishment statement whose renewal keeps the registration valid, and that single registration serves every tenant, which is what lets a vendor onboard a customer without provisioning anything per customer. Software hosted by its vendor rather than deployed by the customer works the same way: one client, many tenants, one listing lifecycle.

An enterprise that reviews and approves client software operates a statement issuer of its own, in the tenant approval role at each provider where it has configured it. Approval of an application is the issuance of a short-lived statement whose `sub` names the application and whose `aud` names the providers where the approval should hold. Renewal is automatic while the approval stands, so approved applications keep working.

The controls follow from the lifetime machinery, and the two lifecycles never have to be synchronized. Onboarding a provider is one trust configuration covering the issuer, its role, identifier scope, accepted metadata authority, and lifetime policy. Approved applications then carry that review to the provider instead of being copied into a separate per-application allowlist. Ceasing renewal at the customer's issuer lapses the application in that customer's tenant at every provider, and leaves the vendor's listing and every other customer untouched; ceasing renewal at the marketplace expires the listing itself. Either prevents new runtime presentations after `exp`, and the second expires statement-governed registrations at their recorded boundary. It also ends refresh-based continuation where the provider requires a current statement under {{refresh}}. Already-issued access tokens remain governed by their own lifetime, and providers retain local control over grants and emergency deprovisioning. Narrowing an approval takes effect when the narrower replacement is next consumed. The result is one issuance policy enforced by multiple trusting authorization servers, subject to their explicit local policy.

# Error Responses {#errors}

A statement consumed at registration is rejected with the {{RFC7591}} error codes as {{ISSUANCE}} defines. A rejected presentation or delivery uses the error responses of {{RFC6749}} for the endpoint at which it was presented. At the token endpoint:

`invalid_client`:
: the statement or its proof fails to establish the client, including a failed profile ({{profiles}}), chain ({{sender-constraint}}), or `jwks_uri` retrieval; also any request under an expired statement-governed registration that does not restore it ({{revalidation}}), at every endpoint including the refresh-token grant.

`unauthorized_client`:
: the tenant has not approved this client, because a required tenant approval is absent, expired, or issued by an issuer configured for another tenant ({{tenant-approval}}); also the grant or response type case below.

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

A registered client, `client_id` `s6BhdRkqt3`, renews its registration's validity by delivering a replacement DCR-profile statement on an ordinary refresh, authenticated under its registered method:

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

## Statement Theft

A runtime presentation resists statement theft because the proof chains to the statement. Possession of an arbitrary key constrains only the request; the chain to the statement makes possession of a stolen statement insufficient unless the attacker also controls a key covered by a mode the authorization server accepts. The registration path cannot offer this property. A DCR-profile statement is a reusable bearer artifact until it expires, so {{ISSUANCE}} relies on narrow audience and lifetime plus local limits on registrations derived from one statement.

Admitting only a key the statement attests is what prevents downgrade: every server in the audience either binds the presenter to that key or refuses the presentation, so a presenter cannot shop the statement to a server with weaker configuration and there prove a key the issuer never reviewed. An extension admitting endorsed keys ({{extensions}}) reopens that question and needs to answer it in its own terms.

## Renewal Authenticates the Credential

Registration renewal proves possession of the registration's own credential and the currency of a statement sharing the governing `iss` and `sub`. It does not prove that the renewing party is the reviewed software: an attacker holding a stolen client credential can renew indefinitely with any current statement for that software, which circulates by design to every deployment of it. Renewal keeps the review current, not the credential honest. Deployments SHOULD pair statement-governed registrations with credential rotation, sender-constrained client authentication, and the registration limits {{ISSUANCE}} recommends, and SHOULD treat a credential compromise as requiring re-registration rather than renewal. The runtime profile does not share this gap, because the sender-constraint chain binds the presenter to the review at every presentation.

## Validation Scope

A consumed statement is validated under the full ruleset of {{ISSUANCE}}: configured issuer trust with keys obtained from authorization server metadata and never from the statement, audience and lifetime checks, and identifier-scope authorization for the `sub`; digest comparison remains the policy input {{ISSUANCE}} defines. Nothing in this specification relaxes those rules; it adds the sender-constraint chain, the grant bindings of {{grant-lifecycle}}, and the registration-validity model on top of them.

## Key Location versus Keys

The `jwks_uri` chain verifies keys at an attested location, not attested keys: `cimd_digest` binds the metadata document's bytes and never the key set served at the URI, so compromise of the client's key host adds keys that satisfy the chain with no digest signal. Deployments for which that exposure is unacceptable prefer statements that attest `jwks` inline, at the cost of digest-visible rotation, or pin observed keys and alert on change. A server reusing a cached key set under {{sender-constraint}} additionally accepts that a just-removed key can briefly continue to verify.

## External Retrieval and Resource Exhaustion {#external-retrieval}

Runtime presentation can cause the authorization server to retrieve the Client ID Metadata Document, an attested `jwks_uri`, or keys and metadata associated with an attested instance issuer. Every such retrieval inherits the server-side request forgery protections of {{CIMD}}. A trusted signature does not make a URL safe: the authorization server MUST apply its URL, redirect, address-range, transport, and content-type policy independently to every referenced location.

Presentation reaches these retrievals before any client is registered or any user has interacted, so the work is available to an unauthenticated requester holding one acceptable statement. An authorization server SHOULD rate-limit presentations per statement identity, per subject, and per source, and SHOULD bound the establishments it will create from one statement ({{effective-metadata}}), before spending retrieval or storage on a new presentation. Before initiating a client-controlled retrieval, the authorization server MUST complete the statement checks that do not depend on that retrieval, including signature, issuer, audience, lifetime, subject, and claim-contract validation. It SHOULD bound JWT size and parsing work, concurrent retrievals, response size, and response time, and SHOULD cache successful and failed retrieval results for an appropriate period. A retrieval failure leaves the relevant metadata or proof chain unverified; the authorization server MUST reject the request and MUST NOT fall back to a weaker sender-constraint mode.

## Statement Validation and Replay {#statement-validation}

A software statement is reusable ({{multi-instance}}) and, until expiry, so is a stolen copy at every registration endpoint in its audience. Issuers SHOULD use the narrowest practical audience and lifetime, and the registration bounds of {{multi-instance}} limit what a stolen statement can create. This replay exposure is specific to the registration path; this specification requires a runtime presentation to prove a key that chains to the statement, which makes a stolen statement inert there to every party outside the issuer's review.

When registrations derived from a statement are intended to share a client key, and the canonical metadata provides `jwks` or `jwks_uri`, the issuer SHOULD include that member in the attested metadata. A trusting authorization server can then require proof of the corresponding private key during or after registration, making a stolen statement unusable without that key. Attesting `jwks_uri` attests the location, not its contents: a compromised key host can add keys that satisfy such proofs with no digest change, so where that exposure matters, attest `jwks` inline and accept digest-visible rotation. When each registration is expected to supply a distinct instance key, the issuer MUST omit `jwks` and `jwks_uri` so that plain registration metadata can carry that key without conflicting with {{RFC7591}} precedence. Omitting key material also forecloses runtime presentation: this specification admits only a key the statement itself attests, so an issuer whose software will be presented at runtime attests `jwks` or `jwks_uri`.

A statement authorizes metadata, not its presenter. If it omits attested key material for per-instance keys ({{multi-instance}}), any holder can register its own key. A trusting authorization server SHOULD require independent proof that the presenter is an authorized instance of the software before accepting such a registration, such as {{ABCA}}, an attested `instance_issuers` chain ({{attesting-instance-issuers}}), or a registration credential the trusting server issued. Without such proof, the one-registration-per-statement default of {{multi-instance}} bounds exposure.

Renewal in this version does not accept a prior software statement as a subject token; a client renews by presenting an initial access token to the token exchange profile ({{ISSUANCE}}) or by making a new software statement request. A stolen statement therefore cannot be exchanged for a fresh one that outlives it. The consumption side carries a different exposure: delivering a replacement to an existing registration proves that registration's own credential and the currency of any statement for the software, not that the deliverer is the reviewed software, so a stolen client credential can sustain a registration indefinitely. this specification states that limit and the credential-rotation and re-registration practices that answer it. Statement-as-subject renewal, and the holder binding against current metadata it requires, is deferred to a future version ({{ISSUANCE}}).

This specification does not define online revocation, and defines no status claim for it. Short lifetimes bound exposure; the Australian Consumer Data Right Register uses ten minutes ({{AU-CDR}}). Scoped trust supports emergency removal of a compromised issuer or namespace ({{issuer-trust}}). An extension could add acceptance-time status, for example a claim referencing a Token Status List {{STATUS-LIST}}, along with the processing rules a trusting authorization server would apply ({{ISSUANCE}}). {{SIGNALS}} defines the event-driven alternative, carrying withdrawal over the pairwise relationship that already exists between issuer and trusting authorization server, shortening the interval between a decision and its effect while leaving expiry as the floor.

At a server that does not implement the registration-validity model of {{registration-validity}}, revocation does not undo registrations already derived from a statement, and responding to malicious software after registration is client lifecycle management at that server. Where that model is in force, a registration expires at the recorded `exp` unless a replacement renews it, so ceasing renewal retires the registration without per-server action.

The explicit, scoped issuer trust configuration that acceptance depends on, and the requirement never to derive trust from key-location values in the statement itself, are specified in {{issuer-trust}}.

At an authorization server hosting multiple tenants, the tenant a statement is evaluated in is part of its meaning. A tenant approval issuer speaks only for its own tenant ({{issuer-roles}}), so accepting its statement in another tenant's context grants one customer's approval authority over another customer's data. An authorization server MUST bind each configured tenant approval issuer to its tenant and resolve the tenant from the request before evaluating any statement.


## Registration Fraud and Impersonation {#registration-fraud}

Open registration permits `client_name`, `logo_uri`, and `client_uri` values that imitate trusted software on consent screens. Requiring a statement replaces self-asserted branding with issuer-reviewed values. Servers that render registration-supplied values on consent screens SHOULD prefer attested values and SHOULD apply heightened scrutiny to unattested registrations that claim user-visible branding.

Statement-gated registration also makes each rotated identity require another issuer decision, rather than letting a discarded client return at no cost; `sub` and `jti` tracking bounds registrations ({{multi-instance}}). Neither control makes metadata true: a client that misleads review can obtain a genuine statement for fraudulent metadata, so issuer verification depth remains decisive ({{ISSUANCE}}).

## Approval Availability

Retrieving approvals puts the tenant's issuer in the path of that tenant's requests, bounded by the caching rules of {{tenant-approval}}: an unreachable issuer costs nothing until the last retrieved statements expire, and then costs that tenant its approvals. This is the same trade the family already takes with statement lifetimes, and the same mitigation applies, which is to choose lifetimes the issuer can sustain and renew ahead of the boundary. Retrieval also discloses to each authorization server the approvals naming it, which is the estate relevant to that server and no more; an issuer serving statements naming other audiences would disclose where else its software is approved ({{ISSUANCE}}).

## Enforcement Bounds {#enforcement-bounds}

Expiry is enforced at every presentation and every statement-governed registration, so a lapsed statement stops new runtime establishment and causes registration-backed requests to fail at the recorded `exp`. It does not retroactively invalidate an establishment, revoke an access token, or terminate an outstanding grant. Requiring a current statement on refresh under {{refresh}} is the control that makes issuer non-renewal end runtime-established grants, and the tenant approval evaluation in that section is what makes a customer's ceased renewal end them; a deployment that adopts neither retains grants for the life of their refresh tokens whatever the statement lifetime; registration-backed grants remain subject to the server's grant policy after the registration expires. A narrowed re-review takes effect through replacement effective metadata or an updated registration. Detection of post-issuance metadata change depends on the signal each profile carries: `software_version` is a vendor-asserted label, while `cimd_digest` covers exact bytes but requires retrieval for comparison. The bounded statement lifetime limits what either signal can miss for new establishment. {{SIGNALS}} defines an optional event mechanism by which an issuer ends a decision before its expiry; expiry remains the floor, and this specification does not depend on it.

Renewal cadence is a deployment trade: short lifetimes tighten the issuer's control loop and increase issuance and delivery traffic, and a fleet of registrations issued together expires together, so issuers SHOULD stagger expiries or renew ahead of the boundary to avoid synchronized lapses.

## Client-Asserted Metadata

Members the statement does not attest are client-asserted, and omission can mean the issuer declined to attest a value. The attested-members-required policy of {{effective-metadata}} is the control for deployments that do not want client-asserted values, notably redirection URIs, entering effective metadata. Resolving a Client ID Metadata Document for unattested members inherits the resolution considerations of {{CIMD}}, including server-side request forgery and availability; a server MAY cache resolution results within the document's caching directives, and the statement's digest binds the review to specific bytes regardless of cache state.

## Observable State {#oracle-considerations}

A retained expired registration is distinguishable from an unknown client, because recovery requires the server to authenticate the registration and evaluate a replacement before rejecting. That disclosure is deliberate and bounded: it is available only to a requester that authenticates as the registration, so it reveals to the legitimate client the state it must act on. Servers publishing `software_statement_registration_validity_supported` additionally disclose that statement-derived registrations expire there, which is configuration a client needs before registering. Neither discloses issuer trust, subject scope, or attester policy, which {{errors}} keeps from unauthenticated requesters.

## Statement Handling

A software statement remains a sensitive artifact in transit and at rest: possession alone does not enable presentation, but statements reveal attested metadata and audience relationships, and servers SHOULD avoid logging them.

Error responses can also disclose trust configuration. The restrictions in {{errors}} prevent unauthenticated probing of issuer, namespace, and attester policy.

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
