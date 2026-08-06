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

{{CIMD}} established that a client can be resolved at request time, with no registration ceremony, from metadata it publishes itself. What resolution alone cannot carry is anyone else's review of that metadata. Runtime presentation closes the gap between the two models: the client presents its statement inside an ordinary authorization or token request, the authorization server verifies the issuer's review and applies the attested metadata to that request, and no client record is created. The review travels inline, at the moment it is needed, to servers that may never have seen the client before.

This specification defines the `software_statement` request parameter for authorization and token requests, the sender-constraint chain that makes presentation safe, the lifecycle of a grant opened by a presentation, the derivation of the client's effective metadata, error reporting, and a discovery signal. Statement format, issuance, and validation are defined by {{ISSUANCE}} and are not modified here. A statement authorizes metadata, not its presenter; runtime proof of the presenter is exactly what the sender-constraint chain supplies, and nothing in this specification attests software instances or binaries.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

OAuth terminology is defined by {{RFC6749}}. Client metadata and software statement terminology is defined by {{RFC7591}}. Client ID Metadata Document terminology is defined by {{CIMD}}. Statement issuance terminology, including Issuing Authorization Server, Trusting Authorization Server, and the statement format and validation rules, is defined by {{ISSUANCE}}.

This specification additionally defines the following term:

Runtime Presentation:
: The consumption of a validated software statement inside an authorization or token request, applying its attested metadata to that request without creating a persistent client registration.

# Runtime Presentation {#runtime-presentation}

The client presents an already-issued statement in an ordinary authorization request or token request, as the `software_statement` request parameter, identified by its Client ID Metadata Document URL as `client_id`. The authorization server validates the statement under the format, issuer trust, audience, lifetime, namespace, and digest rules of {{ISSUANCE}} and uses the attested metadata as the client's effective metadata for that request, creating no persistent registration. This establishes an otherwise unregistered client at request time, as {{CIMD}} resolution does, with the issuer's review carried inline instead of fetched. Unlike a statement issuance request, which cannot ask for access ({{ISSUANCE}}), presenting a statement is compatible with an access-granting request: it is client establishment carried alongside the request, not a request for issuance.

An authorization server advertises support through `software_statement_presentation_supported` ({{authorization-server-metadata}}).

## Sender Constraint {#sender-constraint}

A runtime presentation MUST be sender-constrained, and the constraint MUST chain to the statement. The presenter proves possession of a key: through a client authentication method, a DPoP proof {{RFC9449}}, or the proof-of-possession mechanism of a client attestation scheme such as {{ABCA}}. The authorization server MUST verify the chain in one of three forms: the proven key appears in the attested `jwks` or at the attested `jwks_uri`; a client attester the server trusts under that scheme's own trust establishment ({{ABCA}}) has attested the proven key with the statement's `sub` as subject; or an issuer named by the attested `instance_issuers` delegation has endorsed the proven key in the assertion format that delegation's specification defines ({{CLIENT-INSTANCE}}; {{ISSUANCE}} defines how the delegation is attested). A presentation without such a proof, or whose proven key no chain covers, MUST be rejected. Possession of an arbitrary key constrains only the request; the chain to the statement is what removes the theft exposure that {{ISSUANCE}} manages for the registration path, because a stolen statement cannot be presented with a key outside these chains. A statement that attests neither key material nor an instance-key delegation, at a server that trusts no attester for its `sub`, cannot satisfy this section and is consumable only through registration.

## Authorization Requests {#authorization-requests}

Presentation in an authorization request MUST use a pushed authorization request {{RFC9126}}: the statement and its proof travel to the pushed authorization request endpoint, where the mechanisms of {{sender-constraint}} apply, and no statement appears in a front-channel URL, matching the front-channel discipline of {{ISSUANCE}}. Presentation at the token endpoint requires no additional profile.

## Grant Lifecycle {#grant-lifecycle}

A successful presentation creates an establishment: the validated `sub`, the statement identity and expiry, its `iss` and `jti`, the effective metadata ({{effective-metadata}}), the issuer trust decision, and the sender-constraint mechanism and proven key. The establishment persists for the grant it opens. The authorization server MUST bind the resulting `request_uri` and any authorization code to the establishment; the token request that redeems the code MUST demonstrate possession of the same proven key under the same sender-constraint mechanism, and the statement is not presented again between presentation and redemption. A statement MUST be unexpired when presented; expiry after presentation does not invalidate an establishment already bound, just as expiry does not undo an {{RFC7591}} registration.

On refresh-token use the authorization server MUST verify possession of the establishment's proven key under the same sender-constraint mechanism. It MAY, by local policy, additionally require a current unexpired statement, the control that lets an issuer's ceased renewal wind down access. When policy requires one, the client presents the replacement in the `software_statement` parameter of the refresh request. The replacement MUST validate under {{ISSUANCE}} with this server in its audience, MUST have the establishment's `sub`, and MUST authorize the establishment's proven key ({{sender-constraint}}); the refreshed access MUST fall within the replacement's effective metadata, which replaces the establishment's, so a narrowed re-review takes effect mid-grant. A refresh that fails these requirements, or omits a statement that policy requires, is rejected with `invalid_grant`; the authorization server SHOULD state the reason in `error_description`.

## Effective Metadata {#effective-metadata}

Having validated the statement, the authorization server determines the client's effective metadata for the request. An attested member has the value the statement gives it, with the precedence {{RFC7591}} defines. A member the statement does not attest takes its value from the client's current Client ID Metadata Document, which the authorization server MUST resolve when the request depends on an unattested member, or from the defaults of {{RFC7591}}. Such a member is client-asserted metadata, not part of the review, and omission can mean the issuer declined to attest it: the server applies the same local policy it applies to plain registration metadata, and MAY require specific members, notably the redirection URIs of a redirect flow, to be attested, rejecting a presentation whose statement omits them. The effective metadata is attested member by member, not wholesale. A `redirect_uri` MUST match an effective redirection URI, and any requested grant type, response type, or scope MUST fall within the effective metadata. The `client_id` is the statement's `sub`, and the authorization server assigns none. A server that also holds a persistent registration for the same software decides by local policy whether to accept a runtime presentation alongside it; the statement's attested members govern the presented request only.

Because every presentation consumes the statement afresh, expiry applies per request: a lapsed statement stops new grants at once, a continuous-enforcement property that a persistent registration does not provide by itself. Enforcement of post-issuance change remains a policy input under the digest rules of {{ISSUANCE}}: it stops requests only where the server retrieves the current document and rejects the change.

## Errors {#errors}

Rejections use the error responses of {{RFC6749}} for the endpoint, not the registration error codes: at the token endpoint, `invalid_client` when the statement or its proof fails to establish the client, otherwise `invalid_request`; at the pushed authorization request endpoint, the corresponding {{RFC9126}} responses. This surface is coarser than the registration codes of {{RFC7591}} and does not tell a client whether to seek a corrected statement or a different issuer; the authorization server SHOULD carry that distinction in `error_description`. A request is either an issuance request or a presentation: a request that initiates issuance under {{ISSUANCE}} MUST NOT carry the `software_statement` parameter, and the authorization server rejects the combination with `invalid_request`.

## Example {#example}

The following non-normative example presents an already-issued statement at the token endpoint, sender-constrained by a client attestation {{ABCA}}. The `OAuth-Client-Attestation` header carries an attestation of the instance key from a client attester the server trusts, with the statement's `sub` as its subject; the `OAuth-Client-Attestation-PoP` header proves that key; the `software_statement` parameter carries the issuer's review; and `client_id` is the Client ID Metadata Document URL named by the statement's `sub`. No client record exists at this server.

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

The authorization server validates the statement, verifies that it trusts the attester, that the attestation's subject is the statement's `sub`, and that the proof-of-possession header proves the attested instance key, and applies the effective metadata to the request: the requested `scope` falls within it, and the request authenticates as the client the statement describes. It keeps no registration; the effective `client_id` is the statement's `sub`. Here {{ABCA}} supplies the sender constraint while the parameter carries the statement, distinct from a profile that carries the statement within the attestation itself ({{extensions}}).

# Authorization Server Metadata {#authorization-server-metadata}

This specification defines the following authorization server metadata {{RFC8414}} value:

`software_statement_presentation_supported`:
: OPTIONAL. Boolean value indicating whether the authorization server accepts a software statement presented at runtime in an authorization or token request ({{runtime-presentation}}), rather than only through {{RFC7591}} registration. If omitted, the default value is `false`. This member describes the consuming role. A value of `true` advertises the path as this specification defines it, at every endpoint the server's supported grant types make applicable; there is no per-endpoint signal.

# Extension Points {#extensions}

This specification defines runtime presentation through the `software_statement` request parameter. Two profiles of the same path are left to extensions: carrying the statement within a client attestation {{ABCA}}, and a Client ID Metadata Document that references where the client publishes its current statements, so a resolving server fetches the review out of band. The latter differs from embedding a statement in the document, which the digest rule of {{ISSUANCE}} forbids.

# Security Considerations

The theft and replay analysis follows from {{sender-constraint}}: a stolen statement is inert to every party outside the issuer's review, because presentation requires a key that chains to the statement. This is the property the registration path cannot offer, where {{ISSUANCE}} bounds a stolen statement's use through audience, lifetime, and registration limits instead.

The presented statement is validated under the full ruleset of {{ISSUANCE}}: configured issuer trust with keys obtained from authorization server metadata and never from the statement, audience and lifetime checks, namespace authorization for the `sub`, and digest semantics. Nothing in runtime presentation relaxes those rules; this specification adds the sender-constraint chain and the grant bindings of {{grant-lifecycle}} on top of them.

The per-request enforcement property of {{effective-metadata}} is bounded: expiry is enforced on every presentation, but detection of post-issuance metadata change depends on the server's retrieval policy. A deployment that wants change detection on every grant must retrieve and compare on every presentation and accept the availability dependence that creates on the client's metadata host.

Resolving a Client ID Metadata Document for unattested members inherits the resolution considerations of {{CIMD}}, including server-side request forgery and availability. A server MAY cache resolution results within the document's caching directives; the statement's digest binds the review to specific bytes regardless of cache state.

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

This mechanism began as a consumption section of {{ISSUANCE}} and was split into its own specification so that presentation can progress independently of issuance. It draws on the working-group discussion around Client ID Metadata Documents and attestation-based client authentication.
