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
 - Runtime Presentation
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
  RFC7636:
  RFC9449:
  RFC9700:
  RFC9110:
  CIMD:
    target: https://datatracker.ietf.org/doc/draft-ietf-oauth-client-id-metadata-document
    title: "OAuth Client ID Metadata Document"
  IDJAG:
    target: https://datatracker.ietf.org/doc/draft-ietf-oauth-identity-assertion-authz-grant
    title: "Identity Assertion Authorization Grant"
  STATUSLIST:
    target: https://datatracker.ietf.org/doc/draft-ietf-oauth-status-list
    title: "Token Status List"

informative:
  REGISTRATION:
    target: https://datatracker.ietf.org/doc/draft-mcguinness-oauth-cimd-sw-stmt-registration
    title: "CIMD Software Statement Registration"
  ISSUANCE:
    target: https://datatracker.ietf.org/doc/draft-mcguinness-oauth-cimd-sw-stmt-issuance
    title: "CIMD Software Statement Issuance"
  OPENID-FED:
    target: https://openid.net/specs/openid-federation-1_0.html
    title: "OpenID Federation 1.0"
  UK-OPEN-BANKING:
    target: https://openbankinguk.github.io/dcr-docs-pub/v3.3/dynamic-client-registration.html
    title: "Open Banking UK Dynamic Client Registration Specification v3.3"
  AU-CDR:
    target: https://consumerdatastandardsaustralia.github.io/standards/
    title: "Consumer Data Standards (Australia)"
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

A Client ID Metadata Document carries a client's own claims and can change at any time. This specification defines a software statement in which a trusted party records its review of the exact bytes of such a document, and how an authorization server enforces that review. Presented in a token or pushed authorization request, or published where the reviewed document points, the statement establishes an otherwise unregistered client for that request and the resulting grant. Admission also requires proof of a key the reviewed document carries or, for software distributed to end users, delivery to a redirection URI the document lists, so possession of the statement alone does not complete a grant. A digest binds the statement to the document, which remains the metadata, so one review is portable across the authorization servers in its audience, each of which keeps control of trust, grants, and token lifetime. A companion specification defines consumption of the statement at RFC 7591 registration.

--- middle

# Introduction

A Client ID Metadata Document {{CIMD}} identifies a client by a URL and supplies its metadata from a document the client controls; that metadata is the client's own claim and can change at any time. An organization that reviews client software, such as a publisher program, an enterprise security function, or an ecosystem operator, has no interoperable way to record its review of a particular version of the document, or to have an authorization server it has never dealt with enforce that review.

This specification defines that record and its enforcement. The software statement (Section 2.3 of {{RFC7591}}) profiled here names the client by its document URL and binds the review to the document's exact octets by a digest ({{metadata-digest}}). An authorization server that trusts the statement's issuer admits the client at request time on the reviewed document, and no other, without creating a registration ({{cimd-presentation}}). The document remains the metadata and the statement carries the decision, so one review is portable across every authorization server in its audience, each of which configures its own issuer trust, subject scope, grant policy, and token lifetime. An enterprise operating a statement issuer is the motivating deployment ({{deployment-model}}).

The UK Open Banking Directory and the Australian Consumer Data Right Register each operate a central issuer whose statements many unrelated authorization servers consume at registration through the {{RFC7591}} `software_statement` parameter ({{UK-OPEN-BANKING}}, {{AU-CDR}}). Neither conforms to this specification: neither binds a statement to a document version the consumer retrieves, and both carry client metadata in the statement.

{{OPENID-FED}} conveys attested metadata through trust chains: a consumer resolves an entity to a trust anchor rather than configuring the issuer that vouched for it, and applies metadata derived by policy along that chain. This specification instead keeps the reviewed document as the metadata and leaves issuer trust to local configuration ({{issuer-trust}}). A federation can supply that issuer trust and carry a review as a trust mark, but resolved metadata is not the octets a digest covers, so a client's metadata comes from one source or the other, not both. Ecosystems already operating a federation should consider its registration mechanisms first.

This specification defines the statement, its validation, issuer trust configuration ({{issuer-trust}}), and runtime consumption ({{cimd-presentation}}). {{ISSUANCE}} defines how a client obtains a statement. {{REGISTRATION}} defines consumption of the same statement in an {{RFC7591}} registration request, where the statement's expiry bounds the registration; it builds on this specification, which does not depend on it.

A statement authorizes metadata, not its presenter, and nothing in this specification attests software instances or binaries. How the presenter is bound depends on the kind of client ({{sender-constraint}}):

* A confidential client proves a key the reviewed document carries.
* Software distributed to end users, which can hold no such key, is bound by the reviewed redirection URIs instead.
* Software whose redirection URIs another application could claim is reviewed but not admitted on the strength of the review.

This specification defines only one layer of the decision to let a client act: admission, which sits above the sender-constraint proof that identifies the presenter and the grant that carries a user's authorization. A statement records who reviewed the software and what they attested, not whether a particular customer currently permits the software to operate in its tenant. That question is answered on the customer's schedule rather than the reviewer's; where a customer's identity provider mediates the grant, it is answered continuously by whether that provider issues an assertion ({{identity-assertions}}). A tenant-scoped decision (`aud_tenant`) constrains where a review applies; it does not authorize any particular user or transaction.

Ceasing statement renewal stops new admissions after the applicable expiry, and ends continuation of a grant where the server requires a current statement ({{refresh}}). It does not revoke issued access tokens ({{enforcement-bounds}}).

## Protocol Overview

The following non-normative sequence summarizes runtime admission:

1. Using its Client ID Metadata Document URL as `client_id`, the client presents its statement in a token request or, for a redirect flow, a pushed authorization request, or publishes it at the `software_statements_uri` its document names ({{runtime-presentation}}, {{pulled-statements}}).
2. The authorization server validates the statement ({{profiles}}), resolves the reviewed document ({{effective-metadata}}), and verifies the sender constraint against it ({{sender-constraint}}).
3. The request proceeds under that document's metadata, and no persistent registration is created.
4. The state on which the rest of the grant depends persists as an establishment ({{grant-lifecycle}}).

## Relationship to Client Attestation {#relationship-attestation}

This section is non-normative.

A software statement is an attestation: a signed third-party assertion about a client, accepted by an authorization server that trusts the signer. It is an attributable claim bounded by the signer's process, not proof that what it says is true. It differs from the client attestation of {{ABCA}} and the client instance assertion of {{CLIENT-INSTANCE}} in subject, authority, lifetime, and effect, not in kind.

Subject:
: A software statement attests client software and the metadata a reviewer evaluated. A client attestation attests a running instance and the key it holds.

Authority:
: A software statement is signed by a review authority whose scope is a set of client identifiers. A client attestation is signed by a client attester whose scope is a deployment of the software. An authorization server configures trust in the two independently.

Lifetime and audience:
: A software statement carries a reviewer-chosen expiry and, optionally, an audience, and is consumed at every authorization server that trusts its issuer, within that audience where it names one. A client attestation carries its own expiry and is presented with a proof bound to the request it accompanies.

Effect:
: A software statement supports admission: it says which client this is and what metadata a named reviewer stands behind. A client attestation supplies presenter proof: that the party sending this request holds a key someone vouches for. Neither grants access, and neither substitutes for the other.

A deployment holding only a client attestation knows what is running but not whether anyone approved it; one holding only a software statement knows the software was reviewed but not that this sender is running it. Runtime presentation therefore always requires both the statement and a sender constraint ({{sender-constraint}}).

This specification defines no new attestation format and no new attester role. Where the presenter proves a key the reviewed document carries, it uses ordinary client authentication or DPoP; binding a presenter attestation defined elsewhere is an extension ({{extensions}}).

# Conventions and Definitions

{::boilerplate bcp14-tagged}

This specification uses terminology from {{RFC6749}} (OAuth), {{RFC7591}} (client metadata and software statements), and {{CIMD}} (Client ID Metadata Documents).

This specification defines the following terms.

Issuing Authorization Server:
: The authorization server that makes the issuance decision and signs the software statement.

Trusting Authorization Server:
: An authorization server that consumes a software statement, at runtime under this specification or at registration under {{REGISTRATION}}.

Runtime Presentation:
: The consumption of a validated software statement presented in a token request or pushed authorization request, or pulled from where the client's document points ({{pulled-statements}}), applying the metadata of the document it vouches for without creating a persistent client registration.

Establishment:
: The durable state a successful runtime presentation creates for the grant it opens, enumerated in {{grant-lifecycle}}.

Proven Key:
: The key for which the presenter demonstrates possession during runtime presentation. The accepted proof path binds this key to the statement as specified in {{sender-constraint}}.

Refusal Record:
: State an authorization server holds when it has learned that a statement ceased to be acceptable before its expiry, whether from a status it resolved ({{validation}}) or from its own operator. Wherever this specification requires a statement to be current, a statement matching a refusal record is not current. A refusal derived from a resolved status lasts only as long as that status: a later resolution supersedes it, and a server MUST NOT retain such a refusal once a later resolution no longer supports it. A refusal its operator entered persists on the operator's own terms.

# The Software Statement {#profiles}

The software statement is a compact JWT {{RFC7519}} protected by JWS {{RFC7515}}. Although {{RFC7591}} permits a MAC, a statement issued under this specification MUST use an asymmetric digital signature so trusting servers need not receive an issuer-held symmetric key. The issuer and trusting authorization server MUST follow {{RFC8725}} algorithm-verification guidance. The `none` algorithm and symmetric algorithms MUST NOT be used. An issuer advertises its signing algorithms in its authorization server metadata ({{ISSUANCE}}).

The JOSE header MUST include the `kid` header parameter, identifying the signing key within the issuer's statement key set ({{authorization-server-metadata}}), and MUST include the `typ` header parameter with the value `software-statement+jwt`, applying Section 3.11 of {{RFC8725}}. This value is the media type `application/software-statement+jwt` ({{media-type}}) with the `application/` prefix omitted (Section 4.1.9 of {{RFC7515}}). Explicit typing prevents confusion with other JWTs from the same issuer. Extensions can add claims; an incompatible revision would use a new `typ` value.

A statement asserts that its issuer evaluated the Client ID Metadata Document whose content the digest identifies, and vouches for that document to any party that accepts the issuer. The JWT payload MUST contain the following claims.

`iss`:
: REQUIRED. The issuer identifier of the issuing authorization server, as defined by {{RFC8414}}.

`sub`:
: REQUIRED. The exact client identifier URL presented in the request that produced the statement.

`aud`:
: OPTIONAL. One or more authorization server issuer identifiers ({{RFC8414}}) restricting which authorization servers may accept the statement. Where the claim is present, a trusting authorization server MUST reject the statement unless one of its locally configured audience identifiers exactly matches a value in it. {{ISSUANCE}} states the corresponding constraint on an issuer: a requested audience bounds what it may name. Where the claim is absent, the statement is unrestricted, and acceptance rests on the consumer's configured trust in the issuer and its identifier scope ({{issuer-trust}}). Omitting the claim lets one review serve every authorization server that trusts the issuer, but also lets any holder of a copy use it at any of them. Where `consumable_at` permits registration, a holder can always choose registration, which requires no key. An issuer SHOULD therefore name an audience, and omit it only where it accepts that any server trusting it may register the software on the strength of a copy ({{REGISTRATION}}).

`aud_tenant`:
: OPTIONAL. A tenant identifier at a trusting authorization server, as {{IDJAG}} defines the claim, naming the tenant in which this statement's decision applies. A statement whose decision is confined to one tenant MUST carry it, and a trusting authorization server MUST reject a statement carrying it unless the value identifies the tenant the request belongs to. A consumer MUST NOT read its absence as meaning the statement applies in every tenant, since an issuer also omits it where the consumer is single-tenant or where the issuer does not know the identifier; a consumer that requires a tenant-scoped decision and finds no `aud_tenant` rejects the statement rather than choosing between those readings. A listing review that applies wherever its `aud` reaches carries no `aud_tenant`.

`consumable_at`:
: OPTIONAL. A JSON array naming the points at which the statement may be consumed: `registration` ({{REGISTRATION}}) and `presentation` ({{cimd-presentation}}). A trusting authorization server MUST reject a statement at a point the array does not name, treating a member it does not recognize as naming no point it serves. Where the claim is absent, the statement may be consumed only by presentation; an issuer that intends a statement for registration includes `registration`. Because registration requires no key and a statement published for pulling is readable by anyone ({{pulled-statements}}), a statement is usable at registration only where its issuer says so ({{REGISTRATION}}). A delivery renewing a registration ({{REGISTRATION}}) is consumption at registration; a refresh replacement ({{refresh}}) and a pulled statement are presentation.

`iat`:
: REQUIRED. A NumericDate value representing the time at which the software statement was issued. An issuer MUST NOT issue two statements for a given `iss` and `sub` pair with the same `iat`, and MUST ensure the value increases strictly across its signing nodes, so that the order in which it made its decisions is recoverable from the statements themselves. Consumers rely on that order when one statement replaces another ({{refresh}}, {{REGISTRATION}}), so a statement back-dated for clock skew cannot serve as a replacement.

`exp`:
: REQUIRED. A NumericDate value representing the expiration time. A trusting authorization server MUST reject an expired statement.

`jti`:
: REQUIRED. A unique identifier for the statement within the issuer's namespace.

`cimd_digest`:
: REQUIRED. The metadata digest ({{metadata-digest}}) of the Client ID Metadata Document the issuer evaluated. It binds the statement to the exact document content evaluated, so any party can determine whether the client's currently published metadata still matches it.

`status`:
: OPTIONAL. The `status` claim as {{STATUSLIST}} defines it, locating this statement in the issuer's Status List Token. It lets an issuer end a decision before `exp` without a trusting authorization server contacting the issuer for each statement. {{ISSUANCE}} requires an issuer that publishes status to carry the claim in every statement it issues from then on, since a consumer cannot otherwise distinguish a statement that omits it from one whose issuer publishes no status.

A statement issued under this specification MUST NOT contain client metadata claims. The reviewed metadata is the document the digest names; metadata copied into the statement would duplicate what the digest binds and reopen questions of precedence and partial review. Vouching for particular members without binding the whole document requires an extension defining what a partial review asserts and how a consumer applies it ({{extensions}}).

A trusting authorization server MUST NOT consume a statement of this profile from a `software_statement` member of a Client ID Metadata Document, which {{CIMD}} permits the document to carry, and MUST NOT treat that member as satisfying any requirement of this specification. A statement of this profile cannot describe the document that carries it, since its digest would have to cover its own bytes; one found there names an earlier version. The member may also carry statements of other kinds, which this specification does not evaluate.

The issuer sets the lifetime, and any audience restriction, by policy. A short lifetime bounds exposure and keeps the review fresh. At an authorization server implementing the registration-validity model of {{REGISTRATION}}, however, the lifetime is also the registration's validity and so sets the renewal cadence both parties must sustain.

The `sub` claim is the client identifier URL, not the local `client_id` assigned through {{RFC7591}} registration. A trusting authorization server MAY use `sub` to correlate registrations and apply per-client policy ({{multi-instance}}).

## Metadata Digest {#metadata-digest}

The metadata digest is the unpadded base64url-encoded SHA-256 hash {{RFC6234}} of the retrieved representation body after removal of content coding. The body is not transcoded, normalized, or re-serialized; any byte order mark or trailing newline is included. Retrieval for digest purposes MUST NOT use content negotiation: the fetcher sends `Accept: application/json` and no other variant selection, and decodes any content coding before digesting. When retrieving a document for digest purposes, the fetcher MUST follow the retrieval rules of {{CIMD}}, including its prohibition on automatically following redirects.

A document served as negotiated variants therefore cannot carry a usable statement, since independent fetchers obtain different octets; this follows from the digest and is not a requirement on what a publisher may serve.

An issuer computes the digest over the document it evaluated, and a consumer computes it over the document it retrieves; equal digests show the two are the same octets ({{effective-metadata}}).

Three retrieval conditions affect the comparison:

* {{CIMD}} recommends reading no more than a bounded number of octets and treating a longer response as an error. A digest computed over a truncated read is not the digest of the document, so a trusting authorization server MUST treat a response exceeding its configured bound as a retrieval failure rather than digesting what it read.
* A shared cache may hold a variant selected for some other request. A server MUST NOT compare a digest against a representation it did not itself retrieve, or store under this section, or retain as {{REGISTRATION}} permits. A `304 Not Modified` response confirms that the stored octets are still current, and that their digest still applies, only where it carries a strong entity tag equal to the one stored with those octets. A conditional request with `If-None-Match` is evaluated with weak comparison (Section 13.1.2 of {{RFC9110}}), so a `304` without such a tag does not show the octets are unchanged, and the server retrieves the body again.
* Issuer and consumer must obtain identical octets; any condition that makes retrieval depend on who is asking defeats the comparison.

## Validating a Statement {#validation}

Before accepting a statement, a trusting authorization server MUST:

* verify that the `typ` JOSE header parameter carries the value `software-statement+jwt`;
* verify the signature under a key trusted for the exact `iss` value, obtained as {{issuer-trust}} requires;
* verify that every required claim of {{profiles}} is present and of the correct type, rejecting a statement that omits one;
* validate `iat` and `exp`, rejecting an expired statement and one whose `iat` is unreasonably far in the future according to its clock-skew policy;
* where the statement carries `aud`, verify that one of its own audience identifiers appears in it;
* where the statement carries `aud_tenant`, verify that its value identifies the tenant this request belongs to, resolved before the statement is evaluated, by the server's own means or from a signed assertion it has validated, and never from a value the client chooses;
* verify that `consumable_at` names the point at which the statement is being consumed or, where the claim is absent, that the point is presentation;
* verify that `sub` is a client identifier URL conforming to {{CIMD}}, and that it falls within the identifier scope for which this server accepts the issuer ({{issuer-trust}});
* reject a statement carrying any claim registered in the IANA "OAuth Dynamic Client Registration Metadata" registry, which {{profiles}} forbids, and ignore any other claim it does not recognize; and
* apply the JWT validation guidance in {{RFC8725}}.

Where the statement carries `status` and the trusting authorization server resolves statuses for that issuer, it MUST reject a statement whose status is `INVALID`, and MUST apply its configured policy for that issuer to a statement whose status is `SUSPENDED`. A rejection on status uses the refusal-record row of {{errors}}, because re-presenting the same statement cannot succeed while that status stands.

Where a trusting authorization server resolves statuses for an issuer, resolution follows {{STATUSLIST}}, including its caching rules and its requirement to reject where the referenced index lies outside the list, so a server MAY decide from a Status List Token it already holds within that token's validity. A server MUST NOT replace a Status List Token it holds with one whose `iat` is earlier, and MUST treat one whose `iat` is equal but whose contents differ as a resolution failure, since a host serving an older token could otherwise restore a status the issuer has withdrawn. An issuer gives each token it publishes at a list URI a later `iat` than the last ({{ISSUANCE}}).

Status constrains and never relaxes. A statement carrying no `status`, or whose status the server cannot resolve, is bounded by `exp` as it would be otherwise, and a server MUST NOT treat status as grounds to accept a statement past `exp`. What a server does when resolution fails is local policy. Refusing makes issuer availability a precondition for every request the statement governs; proceeding leaves a withdrawn statement acceptable for the rest of its lifetime.

A trusting authorization server accepts only configured issuers ({{issuer-trust}}) and obtains their statement signing keys from the `software_statement_jwks_uri` value in the issuer's authorization server metadata ({{authorization-server-metadata}}), never from the statement.

A trusting authorization server resolving the reviewed document MUST reject a document containing duplicate object member names, since parsers interpret them differently despite an identical digest.

{{REGISTRATION}} defines rejections at a registration endpoint; rejections elsewhere use {{errors}}.

## Change After Review {#version-changes}

A review covers the document that `cimd_digest` identifies. At registration, a digest mismatch is fatal ({{REGISTRATION}}), because the registration would otherwise record metadata no issuer reviewed. At runtime, a mismatch is a policy input rather than an automatic failure: the statement still names a document the issuer reviewed, and the authorization server decides how to treat the change. Either way, the remedy is re-issuance against the current document.

Because an issuer reviews the document it retrieves from the client identifier URL, a changed document is always served before any statement over it exists; a publisher shortens that gap by arranging prompt review. The statement's bounded lifetime limits how stale a review can become: drift that the digest comparison never observes still expires with the statement.

# Issuer Trust Establishment {#issuer-trust}

A trusting authorization server accepts statements only from configured issuers. Trust is established out of band, for example through a marketplace publisher program or shared enterprise operation; this specification defines no in-band issuer discovery or trust decision.

For each trusted issuer, a trusting authorization server records at least:

* the exact `iss` identifier it will accept;
* the source of that issuer's statement signing keys: the `software_statement_jwks_uri` in the issuer's authorization server metadata ({{authorization-server-metadata}}), discovered from the configured `iss` as {{RFC8414}} describes, and never its `jwks_uri`, whose keys sign other artifacts;
* the signing algorithms it will accept from the issuer;
* the client identifier namespaces the issuer may attest through `sub`;
* the audience identifiers the issuer may name;
* the maximum statement lifetime it will honor, which also caps registration validity where the server implements the registration-validity model of {{REGISTRATION}};
* its policy on repeated and multiple registration ({{multi-instance}}).

These inputs, not the signature alone, define acceptance. A trusting authorization server MUST derive trust from this local configuration and MUST NOT derive it from an `iss`, `jku`, `x5u`, or other key-location value carried in a presented statement.

Issuer trust SHOULD be scoped as well as explicit. Trust configuration SHOULD constrain each issuer to the client identifier namespaces it is expected to attest, for example URLs under the domains of the software publishers it serves. A statement whose `sub` falls outside the issuer's configured scope MUST be rejected even when its signature verifies ({{validation}}). Without such a scope, a compromised or over-broad issuer can mint acceptable statements about any client software. For an issuer attesting software across many publishers, as an enterprise issuer does, the scope is the set of identifiers it is configured for rather than a single domain.

Where a trusting authorization server configures an issuer whose decisions are confined to one of its tenants, as a customer's own issuer is, it SHOULD record with that configuration the client identifier namespaces for which the issuer's decision is required in that tenant. For a client in such a namespace, only a current statement from that issuer establishes reviewed software in that tenant, whatever other issuers' statements say.

A server that records such a requirement MUST NOT admit software in a covered namespace in that tenant without a current statement from the required issuer, whatever its policy for clients it has not reviewed, since otherwise withholding the statement would restore access the customer withdrew. The requirement is configured rather than inferred from the statements the server sees, so a holder cannot evade a customer's withdrawal by presenting or publishing only a broader listing ({{deployment-model}}). A required issuer configured for software whose documents are presented review-only ({{public-client-presentation}}) excludes that software from the tenant, since no statement admits it there.

## Issuer and Consumer as One Server {#same-server}

An authorization server that issued a statement already holds the decision that statement records. Where it is also the trusting authorization server, it MAY admit the client on that record rather than require the statement to be presented back to it, at registration or at runtime.

Such a server MUST apply the conditions it would apply to a presentation, since a decision it made is not a document it has re-read:

* the recorded decision is unexpired, and so is the registration where the validity model of {{REGISTRATION}} governs it;
* its status is current, where the server publishes status ({{validation}});
* the document at `sub` has been obtained; and
* that document's digest equals the one recorded ({{metadata-digest}}).

A client dealing only with the server that reviewed it therefore presents nothing; portability matters only to other servers.

## Multi-Tenant Issuers {#multi-tenant-issuers}

Where the tenants of a reviewer serving several customers are independently trusted reviewing authorities, the reviewer MUST issue under a distinct `iss` for each. Each tenant's decisions, keys, statuses, and watermark then stay separate, and a consumer configures trust per tenant as {{issuer-trust}} describes. A statement carries no claim naming the issuer's tenant.

If several such authorities issue under one `iss`, their statements about the same software are indistinguishable by `iss` and `sub`, so the watermark of {{multi-instance}} treats the later as superseding the earlier: one customer's renewal invalidates the other's review for no reason that customer can observe.

# Runtime Presentation {#cimd-presentation}

The client presents its statement in the request, or publishes it where its document points ({{pulled-statements}}). The authorization server validates it under policy for a trusted issuer, verifies the presenter, and applies the reviewed document's metadata to that request without creating a persistent client registration, as {{CIMD}} resolution does.

## Presentation Request {#runtime-presentation}

A client presents a software statement by including the following parameter in a token request or a pushed authorization request:

`software_statement`:
: REQUIRED for presentation. The software statement ({{profiles}}). It is consumed as a runtime presentation, a refresh replacement ({{refresh}}), or, where {{REGISTRATION}} is implemented, a revalidation delivery under it; a request carrying the parameter in any other role is rejected with `invalid_request`. The authorization server MUST verify that it accepts the statement's issuer for the subject ({{issuer-trust}}), and MUST reject a request repeating the parameter with `invalid_request`.

The request's `client_id` is the client's Client ID Metadata Document URL. It MUST exactly equal the statement's `sub`; the authorization server MUST reject a presentation where they differ, with `invalid_client`. The effective `client_id` is the statement's `sub`, and the authorization server assigns none.

The request MUST also carry the proof required by {{sender-constraint}}: client authentication under a method the reviewed document specifies, a DPoP proof with a key that document carries, or, for a public client, the PKCE and `dpop_jkt` binding of {{public-client-presentation}}. A successful presentation establishes the client for the request and for the grant state derived from it ({{grant-lifecycle}}).

At the token endpoint, a runtime presentation is valid on a request using:

* the `client_credentials` grant;
* the JWT or SAML assertion grants of {{RFC7521}}; or
* another grant an authorization server names in `software_statement_presentation_grant_types_supported` ({{authorization-server-metadata}}).

Authorization-code redemption and refresh-token use continue an existing grant and follow {{grant-lifecycle}} and {{refresh}} instead. A request that asks an authorization server to issue a software statement, rather than to consume one, MUST NOT carry the `software_statement` parameter; {{ISSUANCE}} defines the response type, grant type, and requested token type that identify such a request.

An authorization server advertises support through `software_statement_presentation_supported` ({{authorization-server-metadata}}).

### Authorization Endpoint {#authorization-requests}

Presentation in an authorization request MUST use a pushed authorization request {{RFC9126}}; a pulled statement, which never travels in a request, needs none ({{pulled-statements}}). The pushed authorization request endpoint processes the statement and its proof as {{processing}} describes. The subsequent authorization request MUST use a `client_id` exactly equal to the establishment's `sub` and MUST NOT include the `software_statement` parameter. A statement therefore never appears in a front-channel URL.

### Token Endpoint

The client includes the parameter in an eligible token request ({{runtime-presentation}}).

## Pulled Statements {#pulled-statements}

A client can publish its statements rather than present them. Its Client ID Metadata Document then carries the following client metadata member:

`software_statements_uri`:
: OPTIONAL. URL, in a Client ID Metadata Document, at which the client publishes software statements about itself; the member has no meaning in a registration request. It MUST use the `https` scheme. A request to it returns a JSON object whose `software_statements` member is an array of statements ({{profiles}}), each a string in JWS compact serialization. The metadata digest covers this member ({{metadata-digest}}), so changing where statements are found changes the document every statement names.

An authorization server that supports pulled statements MAY retrieve this URL when it resolves the document of a client that presented no statement. In retrieving it, the server:

* sends a `GET` request with `Accept: application/json`, does not automatically follow redirects, and accepts only a `200` response with a JSON media type;
* applies the protections {{CIMD}} sets for URLs a document contains and the bounds of {{external-retrieval}};
* bounds the response size and the number of statements it examines separately from the document's, since several issuers' statements outgrow a document-sized bound and a host can pad the array;
* discards statements from issuers it has not configured before verifying any signature; and
* caches a successful response no longer than it caches the document, and does not cache a failure, as {{CIMD}} requires of the document.

The authorization server MAY revalidate a cached response from that URL with a conditional request, as it may the document ({{metadata-digest}}). Both the document and this URL are retrieved before any statement exists to validate, so the ordering of {{processing}} applies from the first statement verified onward.

The server considers only statements that:

* have a `sub` equal to the document's client identifier;
* validate under {{validation}};
* permit presentation ({{profiles}}); and
* have a `cimd_digest` equal to the digest of the document it resolved.

It ignores the rest, since anyone able to publish at the URL can place anything there.

Where its trust configuration requires an issuer for this client ({{issuer-trust}}), the server applies only that issuer's statement, and where none remains, the client is not reviewed. Otherwise, where more than one statement remains, it applies any of them, taking the latest `iat` from each issuer, subject to the watermark of {{multi-instance}}. A pulled statement the server does not apply does not advance the watermark. The server then continues from step 4 of {{processing}}, with the pulled statement in place of a presented one and the document already resolved; the selection above satisfies the `client_id` rule of step 2.

The presenter of a pulled statement is bound as for a presented statement ({{sender-constraint}}), at the point the request allows:

* At the token endpoint, the request's client authentication or DPoP proof binds it, as for a presentation there.
* At the authorization endpoint, binding completes at code redemption, before any token is issued. A confidential client authenticates there with a key carried by the octets the server digested, not one that only a later retrieval of the document carries, and that key becomes the establishment's proven key. A public client's authorization request is subject to {{public-client-presentation}}, including PKCE and `dpop_jkt`, and redemption proves the `dpop_jkt` key.
* A public client opening a new grant at the token endpoint has nothing to bind, so a statement pulled for it is review-only ({{public-client-presentation}}), as is one pulled for a document whose redirection URIs that section does not accept. Code redemption and refresh continue an establishment and keep the binding it already has ({{grant-lifecycle}}, {{refresh}}).

Where retrieval does not complete or no statement remains, the server applies its policy for Client ID Metadata Document clients it has not reviewed. Where that policy requires reviewed software, it rejects the request with `temporarily_unavailable` if retrieval did not complete and with `statement_required` if no statement remains. A retrieval failure is never a withdrawal.

On refresh, where {{refresh}} needs a replacement for an establishment created from a pulled statement, the server pulls again and applies, as the replacement under that section, the newest statement with the recorded statement's `iss` and `sub`. Where the newest is the recorded statement itself, nothing is replaced, and currency rests on that statement as {{refresh}} describes. If the re-pull does not complete or yields no replacement, the refresh fails as {{refresh}} provides; a reviewed grant never continues under the policy for clients the server has not reviewed.

An authorization server advertises support through `software_statement_pull_supported` ({{authorization-server-metadata}}).

## Presentation Processing {#processing}

On receiving a presentation, the authorization server proceeds as follows, rejecting as {{errors}} defines at the first failure:

1. Validate the statement ({{validation}}), completing every check that does not require a client-controlled network retrieval before performing any such retrieval.
2. Verify the statement requirements of {{profiles}} and the `client_id` rule of {{runtime-presentation}}.
3. Resolve the reviewed document ({{effective-metadata}}), which is the first step requiring a client-controlled retrieval.
4. Verify the sender constraint against that document ({{sender-constraint}}), then evaluate the request against its metadata ({{effective-metadata}}).
5. On success, create the establishment ({{grant-lifecycle}}).

## Sender Constraint {#sender-constraint}

A runtime presentation MUST be sender-constrained by a key the statement attests or, under {{public-client-presentation}}, by the key the presenter binds through `dpop_jkt`. The presenter proves possession of that key through the applicable client authentication method or a DPoP proof {{RFC9449}}, and the proof MUST be bound to the current request and validated with the replay protections of that mechanism.

Except as {{public-client-presentation}} provides, the proven key MUST appear in the `jwks` or at the `jwks_uri` of the reviewed document ({{effective-metadata}}). Where the reviewed document specifies a client authentication method, the presenter MUST use it, and where the grant type requires client authentication a DPoP proof does not satisfy that requirement ({{RFC9449}}). The authorization server MUST reject a presentation without such a proof, or whose proven key the reviewed document does not carry. A document carrying a redirection URI another application could claim is presented review-only whatever its authentication method ({{public-client-presentation}}).

A statement whose reviewed document carries no key material can be consumed at registration where its `consumable_at` claim permits ({{REGISTRATION}}) and, where the document declares `token_endpoint_auth_method` of `none`, at the pushed authorization request endpoint under {{public-client-presentation}}. Endorsement of a key the statement does not name, by a client attester or by an issuer the statement delegates to, is left to extensions ({{extensions}}).

A key at the document's `jwks_uri` is retrieved at presentation time; if retrieval fails, the key is unverified and the presentation is rejected with `temporarily_unavailable`. A server MAY reuse a recently retrieved key set within ordinary HTTP caching bounds, subject to a maximum reuse period of its own choosing; it MUST NOT let the client's cache directives alone determine how long a removed key continues to verify ({{external-retrieval}}).

### Public Clients {#public-client-presentation}

Software distributed to end users cannot hold a key its reviewed document carries, since a key in a distributed binary is in every copy and identifies the software, not the installation. Such a document declares `token_endpoint_auth_method` of `none` and carries no key material.

A presentation at the pushed authorization request endpoint, or a pulled statement at the authorization endpoint ({{pulled-statements}}), is bound instead by its reviewed redirection URIs where the client's reviewed document declares `token_endpoint_auth_method` of `none` and none of its redirection URIs is one another application could claim (see below). For such a presentation, the authorization server:

* MUST require PKCE {{RFC7636}} with the `S256` method;
* MUST require the presenter to bind a key it holds, through the `dpop_jkt` parameter {{RFC9449}}, and records that key as the proven key of the establishment ({{grant-lifecycle}}); and
* MUST NOT require that key to appear in the reviewed document.

The reviewed document, not the proven key, admits the statement here: an authorization code opened by such a presentation is delivered only to a redirection URI the issuer reviewed, so a holder of a copied statement cannot receive it ({{public-client-security}}). The proven key identifies the installation the grant belongs to; code redemption and refresh demonstrate possession of that same key ({{grant-lifecycle}}, {{refresh}}).

A presentation is review-only where the reviewed document carries any redirection URI that another application on the same device could claim: a non-`https` URI, such as a private-use scheme or an `http` loopback redirection URI, or an `https` URI whose host is a loopback address or `localhost`. Desktop software commonly redirects this way. One such URI makes the whole document review-only, whatever authentication method it declares: the application that claims the URI receives the code ({{public-client-security}}), so neither the reviewed redirection URIs nor a key shipped in every copy of the software can bind the presenter.

A review-only presentation creates no establishment, admits nothing, and does not advance the watermark of {{multi-instance}}, since it changes nothing for the software's other instances; the authorization server proceeds as it would for the same Client ID Metadata Document client presenting no statement. The statement of a review-only presentation MUST NOT satisfy a policy requiring reviewed software, and the authorization server SHOULD NOT present that review to the user as an assurance about the presenter. Where the server's policy requires reviewed software, it rejects a review-only client with `unauthorized_client`, since no statement can make such a client reviewed.

For a review-only presentation, the authorization server MAY record the statement's issuer for audit and inventory, and MAY refuse the request where the statement's status shows a withdrawal, since status constrains and never relaxes ({{validation}}), subject to the bounds of {{external-retrieval}}.

A presentation at the token endpoint under {{runtime-presentation}} opens no redirect and has nothing to bind it, so an authorization server MUST reject one from a client whose reviewed document carries no key material. A statement pulled for such a client at the token endpoint is instead review-only ({{pulled-statements}}).

## Reviewed Metadata {#effective-metadata}

Having validated the statement, and before verifying the proof ({{processing}}), the authorization server MUST obtain the Client ID Metadata Document at the statement's `sub`, by retrieval or from octets it has already digested for that identifier, and compare its digest with `cimd_digest`. Where octets it already holds do not match, it retrieves the document once more before treating the mismatch as a change, since a statement over an updated document would otherwise fail at a server still holding older bytes.

A match means the served document is the reviewed one, and its members are the client's metadata for the request, since a statement vouches for a document rather than a set of claims ({{profiles}}). A mismatch means the document changed after review ({{version-changes}}), and the authorization server MUST NOT treat the changed document as reviewed. It rejects the presentation with `invalid_client` or, by local policy, proceeds without the statement, applying its policy for Client ID Metadata Document clients it has not reviewed to the current document and creating no establishment for the statement.

The request is evaluated against that metadata: a `redirect_uri` MUST match a redirection URI in the document, and any requested grant type, response type, or scope MUST fall within it. A document that omits `scope` sets no ceiling on scope beyond the server's own policy. A grant or response type the authorization server supports but the document does not authorize fails with `unauthorized_client`; a scope outside it fails with `invalid_scope`.

A presentation refused because a bound of {{multi-instance}} is reached is rejected with `invalid_client`; the statement and its proof are sound, so the authorization server SHOULD say so in `error_description`, and the client can retry once earlier establishments are released.

## Grant Lifecycle {#grant-lifecycle}

A successful presentation creates an establishment, the state a server persists for the grant, comprising:

* the validated `sub`;
* the statement identity, its `iss`, `jti`, `iat`, and expiry, and its `status` claim where it carries one;
* the authorization server's own tenant the grant was opened for, where it hosts more than one;
* the reviewed metadata and the digest it matched ({{effective-metadata}});
* the issuer trust decision; and
* the sender-constraint mechanism and proven key.

An authorization server MAY reuse an establishment across presentations that resolve to the same `sub`, statement, and proven key, rather than creating one per request; the bounds of {{multi-instance}} count distinct establishments, not presentations.

The establishment persists while the grant depends on it. The authorization server MUST bind the resulting `request_uri`, authorization code, refresh token, and other grant continuation state to it, as applicable, and MAY discard it once no such state references it.

A token request that redeems an authorization code opened by a presentation:

* MUST have a `client_id` exactly equal to the establishment's `sub`;
* MUST demonstrate possession of the same proven key under the same sender-constraint mechanism; and
* MUST NOT carry the `software_statement` parameter.

A redemption carrying a statement is rejected with `invalid_request`; a wrong client identifier or failed key binding is rejected with `invalid_grant`. This prohibition covers redemption of a code bound to an establishment; a registered client redeeming its own code may deliver a replacement statement under {{REGISTRATION}}, which is a delivery, not a presentation.

A statement MUST be unexpired when presented. Expiry after presentation does not by itself invalidate an establishment already bound. The authorization server controls continued use through its grant and refresh-token policy and can require a current replacement statement as {{refresh}} defines.

### Refresh {#refresh}

On refresh-token use the authorization server MUST verify possession of the establishment's proven key under the same sender-constraint mechanism. It MAY, by local policy, additionally require a current unexpired statement, and SHOULD require one once the establishment's recorded statement has expired ({{enforcement-bounds}}). The recorded statement satisfies that requirement while it is unexpired and no refusal record covers it; a replacement is needed only once it no longer does.

Where the authorization server holds a refusal record for the establishment's recorded statement, it MUST reject a refresh that is not accompanied by a replacement satisfying this section, presented or pulled, whatever its policy on currency otherwise: a withdrawal ends grant continuation at once rather than waiting on local policy. A server that resolves status for the recorded statement's issuer SHOULD check that statement at each refresh against a Status List Token it holds within that token's validity, so that a withdrawal it has resolved reaches open grants and not only new ones.

When a replacement is needed, the client presents it in the `software_statement` parameter of the refresh request, or, for an establishment created from a pulled statement, the server pulls one ({{pulled-statements}}). A statement with the recorded statement's `iss` and `jti` is not a replacement: offered again, it is rechecked as above. The replacement:

* MUST validate under {{validation}}, including its audience where it carries one;
* MUST have the recorded statement's `iss` and `sub`;
* MUST have an `iat` later than the recorded statement's `iat`;
* MUST name, by its `cimd_digest`, the document currently served at its `sub`, which the server confirms by retrieval or by revalidating octets it holds; and
* MUST authorize the establishment's proven key ({{sender-constraint}}) or, for an establishment created under {{public-client-presentation}}, name a document that still satisfies that section.

The refreshed access MUST fall within the metadata of the document the replacement names.

On success, the establishment's statement identity, `iat`, expiry, reviewed metadata, and trust decision are replaced in a single atomic update, and concurrent replacements resolve to the most recently issued statement. A refresh that fails these requirements, or omits a replacement that is needed, is rejected with `statement_required` and leaves the establishment unchanged. Refresh never rotates the establishment's key; a client that needs a new key performs a new presentation.

# Repeated Consumption of One Statement {#multi-instance}

A statement attests client software, identified by `sub`, not its runtime instances, and this specification defines no instance identifier. One unexpired statement is therefore consumable more than once: at each trusting authorization server in its audience and, where the server registers clients ({{REGISTRATION}}) and its policy permits, in more than one registration at the same authorization server.

An authorization server hosting multiple tenants resolves a request's tenant by its own means, such as a per-tenant issuer identifier or endpoint; this specification defines no tenant parameter, and a Client ID Metadata Document URL is the same `client_id` in every tenant. Bounds, inventories, and the watermark below are kept within what the authorization server treats as one deployment (at a multi-tenant server, one tenant), so that an issuer's statements for different tenants do not supersede one another.

A trusting authorization server MUST retain, per `iss` and `sub`, the `iat` of the most recent statement it has accepted, and MUST reject a statement whose `iat` is earlier, whether it arrives as a replacement, a new registration, or a new presentation. Because this floor spans every registration and establishment, a client holding a superseded, broader statement cannot defer a narrowing by opening a fresh registration or establishment with it. The server retains the watermark at least as long as the maximum statement lifetime it honors for that issuer.

For the same reason, the first presentation of a replacement that establishes the client retires its predecessor for every instance of the software at that server. Instances SHOULD therefore obtain the current statement from a source they share, such as an endpoint the publisher operates or a store their deployment shares, rather than each holding a copy fixed when it was built or installed.

A trusting authorization server SHOULD use the statement's `sub` and `iss` to inventory the establishments, and any registrations ({{REGISTRATION}}), derived from that issuer's statements about that software, and SHOULD bound their number. An establishment counts toward a bound only once an access token has been issued from it (in a redirect flow, once its authorization code is redeemed). State created by a pushed authorization request does not count, even where its code was issued but never redeemed, since anyone holding a copy of a public client's statement can create such state and approve it with their own account. A bound is counted across replacements; one keyed on `jti` would reset at every renewal.

Where the reviewed document carries `jwks` or `jwks_uri`, a runtime presentation proves a key from it ({{sender-constraint}}). Software distributed to end users presents at the pushed authorization request endpoint under {{public-client-presentation}}, which creates an establishment per grant rather than a registration per installation. Software whose instances hold their own keys can neither prove nor register those keys ({{sender-constraint}}, {{REGISTRATION}}) and depends on the endorsement extension of {{extensions}}.

# Deployment Model: Centrally Curated Software {#deployment-model}

This section is non-normative.

A provider's marketplace decides which software may exist as a client on its platform, and the provider configures the marketplace's issuer for the identifier namespaces the marketplace speaks for ({{issuer-trust}}). Whether that software may operate in a particular customer's tenant is a separate decision, usually made by the customer's identity provider ({{identity-assertions}}); this specification carries it only where the customer operates its own issuer.

A marketplace application is reviewed once. Its listing is a statement whose renewal keeps the application admissible, and one listing serves every tenant, whether the customer deploys the software or the vendor hosts it, so a vendor onboards customers without per-customer provisioning.

An enterprise that reviews software itself can also operate an issuer, so that its review travels to the providers its statements name in `aud` as an auditable artifact rather than remaining a policy decision inside its identity provider.

The two lifecycles need no synchronization. Onboarding a provider is one trust configuration covering the issuer, its identifier scope, and its lifetime policy, with no per-application allowlist. A request rests on one statement: in a tenant where a provider requires the customer's tenant-scoped decision, that statement is presented; elsewhere, the listing is.

When the customer's issuer stops renewing, the application lapses in that customer's tenant at every provider that requires the customer's decision, leaving the listing and other customers unaffected. When the marketplace stops renewing, the listing expires in every tenant that relies on it, along with registrations a provider bounds by it ({{REGISTRATION}}). Either lapse stops new runtime presentations of the lapsed statements after `exp` and ends refresh-based continuation where the provider requires a current statement ({{refresh}}). Already-issued access tokens remain governed by their own lifetime, and providers retain local control over grants and emergency deprovisioning.

# Relationship to Identity Assertions {#identity-assertions}

A statement records that a named party reviewed software and vouched for its metadata, not that a particular customer currently permits that software to act in its tenant. The two change on different clocks.

Where a customer's identity provider mediates a grant, as when a cross-domain identity assertion carries the customer's users to a provider, the identity provider already answers the second question per grant by issuing assertions only for clients the customer permits: withdrawal takes effect on the next grant, and no artifact outlives the decision. A deployment that wants that property should rely on the identity provider rather than reproduce it with statements.

This specification addresses the durable question, which a statement answers where the reviewer is not in the request path:

* a publisher program or ecosystem directory vouching to servers it will never see;
* a provider admitting software it has never registered; or
* a customer that wants an auditable, portable record of what its review covered, rather than an inference from an assertion having been issued.

# Error Responses {#errors}

A statement consumed at registration is rejected as {{REGISTRATION}} defines. A rejected presentation or replacement uses the error responses of {{RFC6749}} for the endpoint at which it was presented. At the token endpoint:

`invalid_client`:
: the statement or its proof fails to establish the client, including a failed sender constraint ({{sender-constraint}}) and a failed statement requirement of {{profiles}} other than expiry; and, under {{REGISTRATION}}, a request under an expired registration that carries no replacement.

`statement_required`:
: a current statement is required and none was supplied, or the one supplied is expired or refused. The client recovers by obtaining a newer statement; re-sending the same one cannot succeed. Wherever this specification requires a current statement on a refresh-token request, this code replaces `invalid_grant`, so that a client does not read the rejection as an invalid refresh token ({{RFC9700}}).

`temporarily_unavailable`:
: a retrieval this specification requires, of the reviewed document or of a key set it names, did not complete. The client retries; the statement is not in question.

`unauthorized_client`:
: the reviewed document does not authorize the requested grant type or response type, although the authorization server supports it.

`invalid_scope`:
: a requested scope falls outside the reviewed document.

`invalid_request`:
: any other rejection, including a redemption or issuance request carrying the `software_statement` parameter, and a request repeating the parameter ({{runtime-presentation}}, {{grant-lifecycle}}).

The codes apply as follows:

| Condition | Pushed authorization request | Token, including refresh |
| --- | --- | --- |
| Malformed, or failing signature or claim validation | `invalid_client` | `invalid_client` |
| Valid but not acceptable here: issuer not configured, `aud` excludes this server, `sub` outside the issuer's scope, `aud_tenant` not this request's tenant or absent where required, `consumable_at` excludes this point | `invalid_client` | `invalid_client` |
| Expired, or refused by a refusal record, including a status resolved as `INVALID`, or as `SUSPENDED` where policy refuses it, or superseded under the `iat` floor of {{multi-instance}} | `statement_required` | `statement_required` |
| Required statement absent | `statement_required` | `statement_required` |
| Digest does not match the retrieved document | see {{effective-metadata}} | see {{effective-metadata}} |
| Document carries metadata this server's policy refuses | `unauthorized_client` or `invalid_scope` | `unauthorized_client` or `invalid_scope` |
| Retrieval did not complete | `temporarily_unavailable` | `temporarily_unavailable` |
| Review-only client where reviewed software is required ({{public-client-presentation}}) | `unauthorized_client` | `unauthorized_client` |

An authorization server SHOULD use HTTP status code 503 with `temporarily_unavailable` and 400 with the others, so that a client can distinguish a condition worth retrying from one that needs a new statement.

At the pushed authorization request endpoint, these codes are carried in the error response of {{RFC9126}}. On refresh-token use, {{refresh}} takes precedence, and registration validity is reported as {{REGISTRATION}} defines; neither rejection indicates refresh-token replay ({{RFC9700}}).

A statement is never presented at the authorization endpoint ({{authorization-requests}}), though one may be pulled there ({{pulled-statements}}). There, a review-only client is rejected with `unauthorized_client` where reviewed software is required ({{public-client-presentation}}), and a client whose pulled statements could not be retrieved with `temporarily_unavailable` ({{pulled-statements}}).

Otherwise, at the authorization endpoint, where this server requires a statement for a client that has none established, or where the client's statement-governed registration has expired without a replacement ({{REGISTRATION}}), the authorization server MUST return `statement_required` in the authorization error response {{RFC6749}}. The code tells the client to obtain a statement and return through the pushed authorization request endpoint; without it, a client cannot tell a missing review from a policy it will never satisfy.

The authorization server redirects with the error only after resolving the client's Client ID Metadata Document, or for an expired registration consulting the retained registration ({{REGISTRATION}}), and validating the request's `redirect_uri` against it. Where it cannot resolve either, and so cannot validate the redirection URI, the authorization server MUST NOT redirect and reports the error to the resource owner instead.

Where the proof mechanism defines a recoverable error of its own, such as a DPoP nonce challenge {{RFC9449}}, that error takes precedence over the generic errors above.

These codes do not tell a client whether to seek a corrected statement or a different issuer. The authorization server MAY return non-sensitive diagnostics in `error_description` to a client it has authenticated, but MUST NOT reveal issuer-trust, subject-namespace, or attester-policy details to an unauthenticated requester. Where a statement is refused by a refusal record rather than merely expired, and the client is authenticated, the authorization server SHOULD indicate that a statement issued more recently is required, since re-delivering the same statement can never succeed.

# Examples {#example}

Both examples are non-normative.

## Presenting at the Token Endpoint

The following example shows a client presenting an already-issued statement at the token endpoint of a server holding no record for it. The reviewed document, which the statement's digest covers, names a `jwks_uri` and `private_key_jwt` as its authentication method, so the client authenticates with a key that document carries; `client_id` is the Client ID Metadata Document URL named by the statement's `sub` (line breaks are for display purposes only).

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

The authorization server validates the statement, retrieves the document its digest names, verifies the client assertion against a key from that document's `jwks_uri`, and checks that the requested `scope` falls within the document's `scope`. It keeps no registration; the effective `client_id` is the statement's `sub`.

## Publishing a Statement for Servers to Pull

The following example shows a Client ID Metadata Document naming where its statements are published:

~~~ json
{
  "client_id": "https://client.example/app",
  "redirect_uris": ["https://client.example/cb"],
  "token_endpoint_auth_method": "private_key_jwt",
  "jwks_uri": "https://client.example/jwks.json",
  "software_statements_uri":
    "https://client.example/statements.json"
}
~~~

A request to that URL returns the statements, here one:

~~~ json
{
  "software_statements": [
    "eyJ0eXAiOiJzb2Z0d2FyZS1zdGF0ZW1lbnQrand0Iiwi..."
  ]
}
~~~

An authorization server receiving an ordinary authorization request with this `client_id` resolves the document, retrieves the statements, keeps the one from a trusted issuer whose `cimd_digest` matches the document it fetched, and completes the binding when the client authenticates with `private_key_jwt` at code redemption ({{pulled-statements}}).

# Authorization Server Metadata {#authorization-server-metadata}

This specification defines the following authorization server metadata {{RFC8414}} values:

`software_statement_jwks_uri`:
: REQUIRED for an authorization server that issues software statements. URL of a JWK Set containing only the keys with which it signs software statements. The URL MUST use the `https` scheme, as {{RFC8414}} requires of `jwks_uri`, and a trusting authorization server MUST validate the server's certificate when retrieving it, since a substituted key set would let a network attacker sign statements. The keys in it MUST NOT appear in the JWK Set at the server's `jwks_uri`, and the server MUST NOT sign anything other than software statements with them. A trusting authorization server verifies statements only against this set ({{issuer-trust}}), so a key that signs another artifact, such as a Status List Token, cannot be taken for a statement signing key. This member describes the issuing role.

`software_statement_presentation_supported`:
: OPTIONAL. A JSON array naming the endpoints at which the authorization server accepts a software statement presented at runtime ({{runtime-presentation}}). Defined values are `token` and `pushed_authorization_request`; a client ignores a value it does not recognize. Omission, or an empty array, means runtime presentation is not offered; advertising only `token` offers it to clients that need no redirect without the front-channel path. This member describes the consuming role and implies neither acceptance of any particular statement issuer or subject namespace nor support for statement-governed registrations ({{REGISTRATION}}). An authorization server advertising presentation at the pushed authorization request endpoint MUST publish `pushed_authorization_request_endpoint`, since {{authorization-requests}} makes presentation there the only front-channel path. A client also examines the ordinary client-authentication and DPoP metadata for the proof it intends to use. Admitting a public client's presentation at that endpoint ({{public-client-presentation}}) is local policy; a refusal uses {{errors}}.

`software_statement_presentation_grant_types_supported`:
: OPTIONAL. A JSON array of grant type identifiers on which the authorization server accepts a runtime presentation, in addition to those {{runtime-presentation}} names. Omission means only those.

`software_statement_pull_supported`:
: OPTIONAL. Boolean value indicating whether the authorization server retrieves statements from a client's `software_statements_uri` when establishing a client that presented none ({{pulled-statements}}). If omitted, the default value is false. A client whose document names a location needs nothing further from the server; the member tells a client whether it also needs to present.

# Extension Points {#extensions}

The following extensions are left to separate specifications:

* Endorsed keys: a client attestation {{ABCA}}, or an assertion from an issuer named by an `instance_issuers` delegation in the reviewed document {{CLIENT-INSTANCE}}, vouching for a key that document does not carry, which would admit software whose instances hold their own keys.
* Statement conveyance within a client attestation, rather than as a request parameter.
* Partial review, by which an issuer vouches for particular members rather than a whole document ({{profiles}}).

# Security Considerations {#security-considerations}

## Statement Theft and Replay {#statement-validation}

A runtime presentation resists statement theft because the presenter proves a key the reviewed document carries ({{sender-constraint}}), which a thief holding only the statement lacks. Admitting only that key also prevents downgrade: every server in the audience either binds the presenter to it or refuses the presentation. An extension admitting endorsed keys ({{extensions}}) has to address downgrade on its own terms.

Attesting a `jwks_uri` attests the location, not its contents: a compromised key host can add keys that satisfy the proof without a digest change. Where that matters, an issuer attests `jwks` inline and accepts that rotation changes the digest. A server reusing a cached key set ({{sender-constraint}}) also accepts that a just-removed key can briefly continue to verify.

The sender constraint, the grant bindings of {{grant-lifecycle}}, and the registration-validity model of {{REGISTRATION}} add to the validation of {{validation}}; nothing relaxes it.

## Servers That Do Not Implement This Specification {#legacy-servers}

A statement carries no client metadata, so an {{RFC7591}} server that verified one without implementing this specification would take every metadata value from the registration request, attaching the issuer's approval to an attacker's redirection URIs and keys, the substitution {{REGISTRATION}} exists to prevent. Keeping statement signing keys at `software_statement_jwks_uri` and out of `jwks_uri` ({{authorization-server-metadata}}) prevents this. A server that discovers an issuer's keys through {{RFC8414}} finds none that verify a statement, and a server configured with the statement key set has been configured for this specification.

## Copied Statements and Public Clients {#public-client-security}

A public client's presentation ({{public-client-presentation}}) admits a statement without proof of a key the reviewed document carries, so a party holding a copied statement can open a pushed authorization request in the reviewed software's name. It cannot complete the grant, because the authorization code is delivered only to a redirection URI the issuer reviewed. That holds only for redirection URIs no other application on the device can claim, so a document carrying a private-use scheme or loopback redirection URI is presented review-only ({{public-client-presentation}}).

The residual exposure is a consent prompt carrying the reviewed software's name and branding, raised by a party that cannot receive what the user approves. An authorization server SHOULD rate-limit presentations per statement identity and per subject ({{external-retrieval}}), and an issuer bounds the exposure through audience, lifetime, and status ({{profiles}}).

This binding is weaker than a confidential client's, but a key shipped in every copy of software distributed to end users would bind no installation ({{public-client-presentation}}). An endorsement extension ({{extensions}}) would strengthen it by giving the instance a key a third party vouches for.

## Tenant Confusion at a Multi-Tenant Consumer {#tenant-confusion}

Where a trusting authorization server serves several tenants and evaluates anything tenant-specific, the tenant it decides against and the tenant that scopes what it issues MUST be the same, and neither may be selected by a value the client chooses; a tenant asserted in a signed assertion the server has validated, such as an identity assertion authorization grant {{IDJAG}}, is the server's own resolution. A server that derives the tenant one way to check a decision and another way to scope a token lets a client reach one tenant's resources on another tenant's decision. The `aud_tenant` claim makes that failure detectable: a statement naming a tenant cannot be used in another.

Tenant resolution remains the server's responsibility. With no tenant parameter and one client identifier URL across tenants ({{multi-instance}}), neither a client nor an issuer can verify that a server honored this rule.

The `aud_tenant` claim confines a decision to a tenant at the consumer; it does not identify which of the client's own customers is asking. Software that acts for many customers under one client identifier and key presents the same statement and key for each, so a statement naming a tenant admits it there on behalf of every customer it serves, and any of them can direct it at another's tenant. A statement attests software and cannot prevent this; a deployment that needs that binding records it at the consumer or carries the customer as instance identity ({{CLIENT-INSTANCE}}).

## External Retrieval and Resource Exhaustion {#external-retrieval}

Runtime presentation can cause the authorization server to retrieve the Client ID Metadata Document, a `jwks_uri` it names, or the statements its `software_statements_uri` names. Every such retrieval inherits the server-side request forgery protections of {{CIMD}}. A trusted signature does not make a URL safe: the authorization server MUST apply its URL, redirect, address-range, transport, and content-type policy independently to every referenced location.

Presentation reaches these retrievals before any client is registered or any user has interacted, so an unauthenticated requester holding one acceptable statement can cause this work. An authorization server SHOULD therefore bound it:

* rate-limit presentations per statement identity, per subject, and per source, and pulls ({{pulled-statements}}) per client identifier and per source, and bound the establishments it will create from one statement ({{multi-instance}}), before spending retrieval or storage on a new presentation;
* bound JWT size and parsing work, concurrent retrievals, response size, and response time; and
* cache successful retrieval results within the document's caching directives, and back off after a failure rather than cache it, since {{CIMD}} forbids caching error responses.

A retrieval failure leaves the relevant metadata or proof unverified; the authorization server MUST reject the request and MUST NOT fall back to a weaker proof.

## Enforcement Bounds {#enforcement-bounds}

Expiry is enforced at every presentation, so a lapsed statement prevents new runtime admission ({{REGISTRATION}} defines its effect on registrations). It does not retroactively invalidate an establishment, revoke an access token, or terminate an outstanding grant. Issuer non-renewal ends runtime-established grants only where the server requires a current statement on refresh; otherwise they last for the life of their refresh tokens whatever the statement lifetime, unless the server has resolved a withdrawal ({{refresh}}).

A narrowed re-review takes effect when the client publishes the narrower document and obtains a statement over it. Post-issuance metadata change is detected through `cimd_digest`, which covers exact bytes but requires the server to hold the current ones, retrieved or retained. The bounded statement lifetime limits what either signal can miss for new admissions. The `status` claim of {{profiles}}, resolved through {{STATUSLIST}}, lets an issuer end a decision before its expiry, and `exp` remains the floor where no status resolves ({{validation}}). {{SIGNALS}} defines an optional, earlier notification of status change, on which neither this specification nor the status mechanism depends.

Short statement lifetimes tighten the issuer's control and increase issuance and delivery traffic. A fleet of statements issued together expires together, so issuers SHOULD stagger expiries or renew ahead of the boundary to avoid synchronized lapses.

## Status Resolution

Resolving status adds a dependency on the issuer and a fetch the client does not control. Because {{STATUSLIST}} aggregates many statements into one list, a fetch tells the issuer only that some consumer is checking. A server SHOULD fetch on the list's own schedule rather than once per request, so that its request timing does not disclose the client population it serves; per-request resolution would also make every request the statement governs depend on issuer availability.

A status list is signed by the issuer, and a server MUST obtain its verification keys from the `jwks_uri` of the issuer's authorization server metadata {{RFC8414}}, reached from the configured `iss`, never from the list itself. That key set is separate from the issuer's statement key set ({{authorization-server-metadata}}), so a key that signs the list cannot sign statements.

## Document Resolution

Every presentation resolves the Client ID Metadata Document its statement names ({{effective-metadata}}), and that resolution inherits the considerations of {{CIMD}}, including server-side request forgery and availability. A server MAY cache resolution results within the document's caching directives; the digest tells it whether the bytes it holds, retrieved or retained, are the reviewed ones ({{metadata-digest}}, {{REGISTRATION}}).

## Observable State {#oracle-considerations}

A `statement_required` refusal at the authorization endpoint ({{errors}}) travels through the user agent, so it tells anyone who can see the redirection that this client has no established standing at this server. The `software_statement_presentation_supported` metadata discloses only that presentation is offered. The disclosure is accepted because a client that cannot learn why it was refused cannot act. It carries no issuer-trust, subject-scope, or attester-policy detail, which {{errors}} keeps from unauthenticated requesters.

## Statement Handling

Servers SHOULD avoid logging software statements, which remain sensitive in transit and at rest: possession alone does not enable presentation, but a statement names reviewed software and, where it carries an audience, that software's intended relationships.

# Privacy Considerations

A presentation or delivery reveals to the authorization server the client's issuer relationship and, where the statement carries an `aud` claim, the other authorization servers the client intends to use; omitting that claim discloses nothing beyond the review ({{ISSUANCE}}). Pushed authorization requests ({{authorization-requests}}) keep statements out of browser history, referrers, and front-channel logs.

A central issuer also learns from renewals which of its statements are in active use, so issuance and renewal logs deserve the same care as the statements.

Retrieving the document tells its host that the server is resolving the client; pulling statements ({{pulled-statements}}) tells the host of `software_statements_uri` the same; that host is an additional party only where the URL is on another origin. Published statements are readable by anyone, so their `aud` and `aud_tenant` claims disclose which servers and tenants a review names; a statement whose audience is sensitive is presented rather than published.

# IANA Considerations {#iana}

## OAuth Parameters Registry

This specification requests registration of the `software_statement` parameter in the IANA "OAuth Parameters" registry established by {{RFC6749}}, for runtime presentation and statement delivery of the artifact defined by {{RFC7591}}:

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

The "OAuth Dynamic Client Registration Metadata" registry established by {{RFC7591}} already contains a `software_statement` member, which this registration does not affect. The name is reused because the artifact is the same {{RFC7591}} software statement, carried to the same server for the same purpose; a distinct name would make a client carry one artifact under two names, and a server would not recognize at the token endpoint what it accepts at the registration endpoint.

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

## Claims Defined Elsewhere

This specification uses one claim it does not define: `aud_tenant`, defined and registered by {{IDJAG}}.

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

Claim Name:
: `consumable_at`

Claim Description:
: Points at which a software statement may be consumed

Change Controller:
: IESG

Specification Document(s):
: This specification, {{profiles}}

## OAuth Extensions Error Registry

This specification requests registration of the following error codes in the IANA "OAuth Extensions Error Registry" established by {{RFC6749}}:

Error Name:
: `statement_required`

Error Usage Location:
: authorization error response, token error response, pushed authorization request error response

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

This specification requests registration of the following client metadata member, published by a client in its Client ID Metadata Document:

Client Metadata Name:
: `software_statements_uri`

Client Metadata Description:
: URL, in a Client ID Metadata Document, at which the client publishes software statements about itself for an authorization server to retrieve; not meaningful in a registration request.

Change Controller:
: IESG

Specification Document(s):
: This specification, {{pulled-statements}}

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
: `software_statement_jwks_uri`

Metadata Description:
: URL of the JWK Set containing only the keys with which the authorization server signs software statements.

Change Controller:
: IESG

Specification Document(s):
: This specification, {{authorization-server-metadata}}

Metadata Name:
: `software_statement_pull_supported`

Metadata Description:
: Boolean value indicating whether the authorization server retrieves software statements from a client's `software_statements_uri`.

Change Controller:
: IESG

Specification Document(s):
: This specification, {{authorization-server-metadata}}

--- back

# Acknowledgments
{:numbered="false"}

This mechanism began as a consumption section of {{ISSUANCE}} and was split out so that consumption can evolve separately from issuance. It draws on working-group discussion of Client ID Metadata Documents and attestation-based client authentication.
