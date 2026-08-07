---
title: "OAuth 2.0 Software Statement Runtime Presentation"
abbrev: oauth-sw-stmt-presentation
docname: draft-mcguinness-oauth-sw-stmt-presentation-latest
category: std

ipr: trust200902
area: "Security"
workgroup: "Web Authorization Protocol"

keyword:
 - OAuth
 - Software Statement
 - Client ID Metadata Document
 - Runtime Presentation
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
  RFC8414:
  RFC9126:
  RFC9449:
  CIMD:
    target: https://datatracker.ietf.org/doc/draft-ietf-oauth-client-id-metadata-document
    title: "OAuth Client ID Metadata Document"
  ISSUANCE:
    target: https://datatracker.ietf.org/doc/draft-mcguinness-oauth-software-statement-issuance
    title: "OAuth 2.0 Software Statement Issuance"
  CLIENT-INSTANCE:
    target: https://datatracker.ietf.org/doc/draft-mcguinness-oauth-client-instance-assertion
    title: "OAuth 2.0 Client Instance Assertion"

informative:
  ABCA:
    target: https://datatracker.ietf.org/doc/draft-ietf-oauth-attestation-based-client-auth
    title: "OAuth 2.0 Attestation-Based Client Authentication"

--- abstract

RFC 7591 defines the software statement and how a dynamic client registration endpoint consumes it. This specification defines a second consumption path, runtime presentation: a client identified by a Client ID Metadata Document URL presents an already-issued software statement in an authorization or token request through the `software_statement` parameter, and the authorization server applies the attested metadata to that request without creating a persistent registration. Every presentation is sender-constrained by a proof that chains to the statement, so a stolen statement cannot be presented.

--- middle

# Introduction

A software statement ({{RFC7591}}, Section 2.3) is a signed JWT in which an issuer attests client metadata. {{ISSUANCE}} defines how a client obtains one for a client identified by a Client ID Metadata Document URL {{CIMD}}, and hardens the artifact with a subject bound to that URL, a digest of the reviewed document, an explicit audience, and a bounded lifetime. Until now the statement has had one consumption path: an {{RFC7591}} registration request, after which the client's standing at the server is carried by a persistent registration.

{{CIMD}} established that a client can be resolved at request time, with no registration ceremony, from metadata it publishes itself. What resolution alone cannot carry is anyone else's review of that metadata. Runtime presentation closes the gap between the two models: the client presents its statement inside an ordinary authorization or token request, the authorization server verifies the issuer's review and applies the attested metadata to that request, and no client record is created.

This specification defines the `software_statement` request parameter for authorization and token requests ({{runtime-presentation}}), the processing rules a trusting authorization server applies ({{processing}}), the lifecycle of a grant opened by a presentation ({{grant-lifecycle}}), error reporting ({{errors}}), and a discovery signal ({{authorization-server-metadata}}). Statement format, issuance, and validation are defined by {{ISSUANCE}} and are not modified here; {{presentable-statement}} states the claim contract a presentation consumes, and decouples it from how the statement was acquired.

A statement authorizes metadata, not its presenter. Runtime proof of the presenter is what the sender-constraint rules of {{sender-constraint}} supply, and nothing in this specification attests software instances or binaries.

## Protocol Overview

The following non-normative sequence summarizes a presentation:

1. The client includes the `software_statement` parameter in a token request, or in a pushed authorization request for a redirect flow, with its Client ID Metadata Document URL as `client_id`.
2. The authorization server validates the statement ({{presentable-statement}}), verifies the sender-constraint chain ({{sender-constraint}}), and derives the client's effective metadata for the request ({{effective-metadata}}).
3. The request proceeds under the effective metadata. No persistent registration is created.
4. The state the rest of the grant depends on persists as an establishment ({{grant-lifecycle}}).

# Conventions and Definitions

{::boilerplate bcp14-tagged}

OAuth terminology is defined by {{RFC6749}}. Client metadata and software statement terminology is defined by {{RFC7591}}. Client ID Metadata Document terminology is defined by {{CIMD}}. Statement issuance terminology, including Issuing Authorization Server, Trusting Authorization Server, and the statement format and validation rules, is defined by {{ISSUANCE}}.

This specification additionally defines the following terms:

Runtime Presentation:
: The consumption of a validated software statement inside an authorization or token request, applying its attested metadata to that request without creating a persistent client registration.

Establishment:
: The durable state a successful runtime presentation creates for the grant it opens, enumerated in {{grant-lifecycle}}.

# Presentable Statement {#presentable-statement}

Runtime presentation consumes a software statement conforming to the CIMD-anchored format of {{ISSUANCE}}. The registration-only shape for non-CIMD clients defined there cannot be presented; the processing rules of this specification assume the document the subject names. How the client acquired the statement is out of scope: the issuance flows of {{ISSUANCE}} are one way to obtain a presentable statement, not a requirement of presentation.

Each element below is consumed by a specific mechanism of this specification; a statement lacking any of them cannot be presented at runtime, whatever a registration endpoint might still accept:

`typ` header:
: `software-statement+jwt`; explicit typing prevents confusion with other JWTs from the same issuer.

`iss`:
: identifies the issuer for trust, claim-authority, and namespace decisions, and forms half of the establishment identity ({{grant-lifecycle}}).

`sub`:
: the client's Client ID Metadata Document URL; the request's `client_id` equals it, by the rule of {{runtime-presentation}}.

`aud`:
: scopes which authorization servers may accept the presentation; without it a statement would be presentable at every server that trusts the issuer.

`exp`:
: checked at every presentation and at the validity points of {{grant-lifecycle}}.

`jti`:
: the other half of the establishment identity; the inventory, concurrency bounds, and replacement matching of this specification key on it.

`cimd_digest`:
: binds the issuer's review to the exact document bytes evaluated and feeds the post-issuance change policy of {{ISSUANCE}}.

The syntax, validation, and semantics of every element are those of {{ISSUANCE}}; this section adds no claim rules. A presented statement failing this contract is rejected as {{errors}} defines for an invalid statement.

# Presentation Request {#runtime-presentation}

A client presents a software statement by including the following parameter in a token request or a pushed authorization request:

`software_statement`:
: REQUIRED for presentation. The software statement ({{presentable-statement}}).

The request's `client_id` is the client's Client ID Metadata Document URL. It MUST exactly equal the statement's `sub`; the authorization server MUST reject a presentation where they differ. The effective `client_id` is the statement's `sub`, and the authorization server assigns none.

A presentation is compatible with an access-granting request: it is client establishment carried alongside the request, not a request for issuance. A request that initiates issuance under {{ISSUANCE}} MUST NOT carry the `software_statement` parameter; the authorization server rejects the combination with `invalid_request`.

An authorization server advertises support through `software_statement_presentation_supported` ({{authorization-server-metadata}}).

## Authorization Endpoint {#authorization-requests}

Presentation in an authorization request MUST use a pushed authorization request {{RFC9126}}. The statement and its proof are sent to the pushed authorization request endpoint, where the processing rules of {{processing}} apply. A statement never appears in a front-channel URL, just as {{ISSUANCE}} keeps the issued statement out of authorization responses.

## Token Endpoint

Presentation at the token endpoint requires no additional profile: the client includes the parameter in the token request alongside the grant.

# Presentation Processing {#processing}

On receiving a presentation, the authorization server proceeds as follows, rejecting as {{errors}} defines at the first failure:

1. Validate the statement under {{ISSUANCE}}: format, issuer trust, audience, lifetime, and namespace, with digest comparison as the policy input defined there.
2. Verify the contract of {{presentable-statement}} and the `client_id` rule of {{runtime-presentation}}.
3. Verify the sender-constraint chain ({{sender-constraint}}).
4. Derive the effective metadata and evaluate the request against it ({{effective-metadata}}).
5. On success, create the establishment ({{grant-lifecycle}}).

## Sender Constraint {#sender-constraint}

A runtime presentation MUST be sender-constrained, and the constraint MUST chain to the statement. The presenter proves possession of a key through a client authentication method, a DPoP proof {{RFC9449}}, or the proof-of-possession mechanism of a client attestation scheme such as {{ABCA}}.

Which chain the authorization server accepts depends on what the statement attests:

* When the statement attests key material, the proven key MUST appear in the attested `jwks` or at the attested `jwks_uri`, and the presenter MUST authenticate with the effective `token_endpoint_auth_method` where one applies. No other chain is accepted.
* When the statement attests no key material, the authorization server MUST verify one of two endorsement chains: an attestation of the proven key whose subject is the statement's `sub`, from a client attester the server trusts for that `sub` or its namespace ({{ABCA}}); or an endorsement of the proven key from an issuer named by the attested `instance_issuers` delegation, in the assertion format that delegation's specification defines ({{CLIENT-INSTANCE}}; {{ISSUANCE}} defines how the delegation is attested).

Each chain is gated by the authority behind it: an attested member counts only where the server treats the statement issuer as authoritative for that member ({{ISSUANCE}}), and attester trust counts only within its configured `sub` scope. The authorization server MUST reject a presentation without such a proof, or whose proven key no accepted chain covers.

A statement that attests neither key material nor an instance-key delegation, presented at a server whose attester trust does not cover its `sub`, cannot satisfy this section and is consumable only through registration.

Verifying a key at the attested `jwks_uri` is a retrieval at presentation time. A fetch failure leaves the chain unverified, and the presentation is rejected as `invalid_client`. A server MAY reuse a recently retrieved key set within ordinary HTTP caching bounds ({{security-considerations}}).

## Effective Metadata {#effective-metadata}

Having validated the statement and its proof, the authorization server derives the client's effective metadata for the request:

* An attested member for which the server treats the issuer as authoritative ({{ISSUANCE}}) has the value the statement gives it, with the precedence {{RFC7591}} defines.
* An attested member outside that set does not take attested precedence; per the policy rule of {{ISSUANCE}}, the server ignores it or treats it as client-asserted.
* A member the statement does not attest takes its value from the client's current Client ID Metadata Document, the document at the statement's `sub`, or from the defaults of {{RFC7591}}. Such a member is client-asserted metadata, not part of the review. The authorization server MUST resolve that document when the request depends on an unattested member.

The effective metadata is therefore attested member by member, not wholesale. The authorization server MAY require specific members, notably the redirection URIs of a redirect flow, to be attested, rejecting a presentation whose statement omits them.

The request is evaluated against the effective metadata: a `redirect_uri` MUST match an effective redirection URI, and any requested grant type, response type, or scope MUST fall within the effective metadata.

A server that also holds a persistent registration for the same software decides by local policy whether to accept a runtime presentation alongside it; the statement's attested members govern the presented request only.

A trusting authorization server SHOULD use the statement's `sub`, `iss`, and `jti` to inventory the establishments derived from one statement and MAY bound their concurrent number.

# Grant Lifecycle {#grant-lifecycle}

A successful presentation creates an establishment comprising:

* the validated `sub`;
* the statement identity, its `iss` and `jti`, and its expiry;
* the effective metadata ({{effective-metadata}});
* the issuer trust decision; and
* the sender-constraint mechanism and proven key.

The establishment persists for the grant it opens. The authorization server MUST bind the resulting `request_uri` and any authorization code to the establishment. The token request that redeems the code MUST demonstrate possession of the same proven key under the same sender-constraint mechanism, and MUST NOT carry the `software_statement` parameter; a redemption carrying one is rejected with `invalid_request`.

A statement MUST be unexpired when presented. Expiry after presentation does not invalidate an establishment already bound, just as expiry does not undo an {{RFC7591}} registration.

## Refresh {#refresh}

On refresh-token use the authorization server MUST verify possession of the establishment's proven key under the same sender-constraint mechanism. It MAY, by local policy, additionally require a current unexpired statement.

When policy requires one, the client presents the replacement in the `software_statement` parameter of the refresh request. The replacement:

* MUST validate under {{ISSUANCE}} with this server in its audience;
* MUST have the establishment's `iss` and `sub`; and
* MUST authorize the establishment's proven key ({{sender-constraint}}).

The refreshed access MUST fall within the replacement's effective metadata; the grant never widens beyond the original authorization. On success, the establishment's statement identity, expiry, effective metadata, and trust decision are replaced atomically. A refresh that fails these requirements, or omits a statement that policy requires, is rejected with `invalid_grant` and leaves the establishment unchanged; the authorization server SHOULD state the reason in `error_description`.

# Error Responses {#errors}

A rejected presentation uses the error responses of {{RFC6749}} for the endpoint at which it was presented, not the registration error codes of {{RFC7591}}. At the token endpoint:

`invalid_client`:
: the statement or its proof fails to establish the client, including a failed contract ({{presentable-statement}}), chain ({{sender-constraint}}), or `jwks_uri` retrieval.

`invalid_scope`:
: a requested scope falls outside the effective metadata.

`unsupported_grant_type`:
: the requested grant type falls outside the effective metadata.

`invalid_request`:
: any other rejection, including a redemption or issuance request carrying the `software_statement` parameter ({{runtime-presentation}}, {{grant-lifecycle}}).

At the pushed authorization request endpoint, the corresponding {{RFC9126}} error responses apply. On refresh-token use, {{refresh}} takes precedence: statement and proof failures there are `invalid_grant`, the grant's continuation failing rather than client establishment.

This surface is coarser than the registration codes and does not tell a client whether to seek a corrected statement or a different issuer; the authorization server SHOULD carry that distinction in `error_description`.

# Example {#example}

The following non-normative example presents an already-issued statement at the token endpoint, sender-constrained by a client attestation {{ABCA}}. The `OAuth-Client-Attestation` header carries an attestation of the instance key from a client attester the server trusts for the statement's `sub`, with that `sub` as its subject; the `OAuth-Client-Attestation-PoP` header proves that key; the `software_statement` parameter carries the issuer's review; and `client_id` is the Client ID Metadata Document URL named by the statement's `sub`. No client record exists at this server.

~~~ http
POST /token HTTP/1.1
Host: as.example
Content-Type: application/x-www-form-urlencoded
OAuth-Client-Attestation: eyJhbGciOiJFUzI1NiIsInR5cCI6Im9hdXRoLWNsaWVu...
OAuth-Client-Attestation-PoP: eyJhbGciOiJFUzI1NiIsInR5cCI6Im9hdXRoLWNs...

grant_type=client_credentials
&scope=tools.read
&client_id=https%3A%2F%2Fclient.example%2Fapp
&software_statement=eyJ0eXAiOiJzb2Z0d2FyZS1zdGF0ZW1l...
~~~

The authorization server validates the statement, verifies that its trust in the attester covers the statement's `sub`, that the attestation's subject is that `sub`, and that the proof-of-possession header proves the attested instance key, and applies the effective metadata to the request: the requested `scope` falls within it, and the request authenticates as the client the statement describes. It keeps no registration; the effective `client_id` is the statement's `sub`. Here {{ABCA}} supplies the sender constraint while the parameter carries the statement, distinct from a profile that carries the statement within the attestation itself ({{extensions}}).

# Authorization Server Metadata {#authorization-server-metadata}

This specification defines the following authorization server metadata {{RFC8414}} value:

`software_statement_presentation_supported`:
: OPTIONAL. Boolean value indicating whether the authorization server accepts a software statement presented at runtime in an authorization or token request ({{runtime-presentation}}), rather than only through {{RFC7591}} registration. If omitted, the default value is `false`. This member describes the consuming role. A value of `true` advertises the path as this specification defines it, at every endpoint the server's supported grant types make applicable; there is no per-endpoint signal.

# Extension Points {#extensions}

This specification defines runtime presentation through the `software_statement` request parameter. Two profiles of the same path are left to extensions: carrying the statement within a client attestation {{ABCA}}, and a Client ID Metadata Document that references where the client publishes its current statements, so a resolving server fetches the review out of band. The latter differs from embedding a statement in the document, which the digest rule of {{ISSUANCE}} forbids. A discovery signal finer than `software_statement_presentation_supported`, advertising which chains a server accepts, is likewise left to an extension.

# Security Considerations {#security-considerations}

## Statement Theft

A runtime presentation is safe for an unregistered presenter because the proof chains to the statement. Possession of an arbitrary key constrains only the request; the chain to the statement is what makes a stolen statement inert to every party outside the issuer's review. The registration path cannot offer this property; {{ISSUANCE}} bounds a stolen statement's use there through audience, lifetime, and registration limits instead.

Restricting the accepted chain by what the statement attests prevents downgrade: when key material is attested, no other chain is accepted, so a reviewed key cannot be bypassed by a presenter selecting a weaker proof form. The authority gates of {{sender-constraint}} serve the same end: an issuer trusted for other metadata, or an attester trusted for other subjects, does not authorize runtime keys.

## Validation Scope

The presented statement is validated under the full ruleset of {{ISSUANCE}}: configured issuer trust with keys obtained from authorization server metadata and never from the statement, audience and lifetime checks, and namespace authorization for the `sub`; digest comparison remains the policy input {{ISSUANCE}} defines. Nothing in runtime presentation relaxes those rules; this specification adds the sender-constraint chain and the grant bindings of {{grant-lifecycle}} on top of them.

## Key Location versus Keys

The `jwks_uri` chain verifies keys at an attested location, not attested keys: `cimd_digest` binds the metadata document's bytes and never the key set served at the URI, so compromise of the client's key host adds keys that satisfy the chain with no digest signal. Deployments for which that exposure is unacceptable prefer statements that attest `jwks` inline, at the cost of digest-visible rotation, or pin observed keys and alert on change. A server reusing a cached key set under {{sender-constraint}} additionally accepts that a just-removed key can briefly continue to verify.

## Enforcement Bounds

Expiry is enforced at every presentation, so a lapsed statement stops new grants at once, a continuous-enforcement property a persistent registration does not provide by itself. Grants already open continue under {{grant-lifecycle}}; the refresh-time statement requirement of {{refresh}} is the policy lever that winds down existing access when an issuer ceases renewal, and a narrowed re-review takes effect mid-grant through the replacement's effective metadata. Detection of post-issuance metadata change depends on the server's retrieval policy: it stops requests only where the server retrieves the current document and rejects the change, and change detection on every grant costs a retrieval per presentation and an availability dependence on the client's metadata host.

## Client-Asserted Metadata

Members the statement does not attest are client-asserted, and omission can mean the issuer declined to attest a value. The attested-members-required policy of {{effective-metadata}} is the control for deployments that do not want client-asserted values, notably redirection URIs, entering effective metadata. Resolving a Client ID Metadata Document for unattested members inherits the resolution considerations of {{CIMD}}, including server-side request forgery and availability; a server MAY cache resolution results within the document's caching directives, and the statement's digest binds the review to specific bytes regardless of cache state.

## Statement Handling

A software statement remains a sensitive artifact in transit and at rest: possession alone does not enable presentation, but statements reveal attested metadata and audience relationships, and servers SHOULD avoid logging them.

# Privacy Considerations

A presentation reveals to the authorization server the client's issuer relationship and the full attested metadata, including the audience list, which names the other authorization servers the client intends to establish relationships with. Issuers can bound the disclosure by keeping audiences narrow, as {{ISSUANCE}} recommends. The pushed authorization request requirement of {{authorization-requests}} keeps statements out of browser history, referrers, and front-channel logs.

# IANA Considerations {#iana}

## OAuth Parameters Registry

This specification requests registration of the `software_statement` parameter in the IANA "OAuth Parameters" registry established by {{RFC6749}}, for runtime presentation of the artifact defined by {{RFC7591}}:

Parameter Name:
: `software_statement`

Parameter Usage Location:
: authorization request, token request

Change Controller:
: IESG

Specification Document(s):
: This specification, {{runtime-presentation}}

## OAuth Authorization Server Metadata Registry

This specification requests registration of the following value in the IANA "OAuth Authorization Server Metadata" registry established by {{RFC8414}}:

Metadata Name:
: `software_statement_presentation_supported`

Metadata Description:
: Boolean value indicating whether the authorization server accepts a software statement presented at runtime in an authorization or token request.

Change Controller:
: IESG

Specification Document(s):
: This specification, {{authorization-server-metadata}}

--- back

# Acknowledgments
{:numbered="false"}

This mechanism began as a consumption section of {{ISSUANCE}} and was split into its own specification so that presentation can evolve separately from issuance. It draws on the working-group discussion around Client ID Metadata Documents and attestation-based client authentication.
