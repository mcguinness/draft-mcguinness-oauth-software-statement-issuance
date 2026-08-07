---
title: "OAuth 2.0 Software Statement Presentation"
abbrev: oauth-sw-stmt-presentation
docname: draft-mcguinness-oauth-sw-stmt-presentation-latest
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
  ABCA:
    target: https://datatracker.ietf.org/doc/draft-ietf-oauth-attestation-based-client-auth
    title: "OAuth 2.0 Attestation-Based Client Authentication"

informative:
  RFC7592:

--- abstract

RFC 7591 defines the software statement but leaves its consumption underdefined: a statement is evaluated once at registration, and the registration then outlives the review it was based on. This specification profiles the software statement for two client identity models and defines consumption that keeps the review current. A statement bound to an RFC 7591 `software_id` governs a dynamic client registration, giving the registration the statement's expiry as its validity, renewed by delivering a replacement. A statement bound to a Client ID Metadata Document URL is presented at runtime in an authorization or token request, establishing an unregistered client for that request alone; every runtime presentation proves possession of a key directly attested by the statement or covered by an accepted endorsement chain, so possession of the statement alone is insufficient. In both models the statement issuer's expiry and renewal decisions govern the client's standing at every authorization server in the statement's audience, so one issuer can centrally curate the approved software across many servers.

--- middle

# Introduction

An {{RFC7591}} dynamic client registration is permanent by default. A software statement ({{RFC7591}}, Section 2.3) can carry a reviewer's approval into the registration request, but {{RFC7591}} leaves the statement's content and lifecycle largely undefined: nothing standard says which claims it must carry, what binds it to the registering software, or what happens when the review it represents goes stale. The registration outlives the approval. An organization that reviews and approves client software therefore has no standard way to bound that approval in time, or to withdraw it, at the authorization servers that relied on it, and every server it works with accumulates its own drifting allowlist.

This specification closes that gap by making the statement issuer's decisions govern client standing continuously:

* The issuer sets the statement's `exp`, and a registration governed by a statement takes that expiry as its validity ({{registration-validity}}). The client renews by delivering a replacement statement ({{revalidation}}); a registration whose statement lapses expires.
* An issuer that stops renewing a client's statements thereby winds down that client at every authorization server in the statements' audience, with no per-server deprovisioning. One issuer can centrally manage and curate the approved software estate across many authorization servers; an enterprise operating its own issuer is the motivating deployment ({{deployment-model}}).

Two client identity models are profiled ({{profiles}}), because clients arrive at authorization servers in two ways:

* **DCR profile** ({{dcr-profile}}): the client is registered through {{RFC7591}} and identified by its `software_id`. The statement is consumed at registration and at renewal, and it governs the registration's validity.
* **CIMD profile** ({{cimd-profile}}): the client is identified by a Client ID Metadata Document URL {{CIMD}} and may be entirely unregistered. The statement is presented at runtime, inside an authorization or token request, establishing the client for that request alone ({{runtime-presentation}}); no persistent registration is created, though the authorization server retains the grant-bound establishment state defined in {{grant-lifecycle}}.

Statement format, issuance, and validation are defined by {{ISSUANCE}} and are not modified here; the profiles in this document state which elements each consumption model requires, and how the client acquired its statement is out of scope. A statement authorizes metadata, not its presenter. Runtime proof of the presenter is what the sender-constraint rules of {{sender-constraint}} supply, and nothing in this specification attests software instances or binaries.

## Protocol Overview

The following non-normative sequences summarize the two models.

Registration governed by a statement (DCR profile):

1. The client registers through {{RFC7591}}, carrying a DCR-profile statement in the `software_statement` member; the statement binds to the registration by `software_id` and, when attested, `software_version` ({{ISSUANCE}}).
2. The authorization server records the statement's identity and `exp` with the registration; the registration is valid until that expiry.
3. Before expiry, the client delivers a replacement statement, in a token request or a new registration request; the registration's validity extends to the replacement's `exp`.
4. If no replacement arrives, the registration expires and requests under it fail; an issuer that ceases renewal has offboarded the client here and at every other server in its statements' audience.

Runtime presentation (CIMD profile):

1. The client includes the `software_statement` parameter in a token request, or in a pushed authorization request for a redirect flow, with its Client ID Metadata Document URL as `client_id`.
2. The authorization server validates the statement ({{cimd-profile}}), verifies the sender-constraint chain ({{sender-constraint}}), and derives the client's effective metadata for the request ({{effective-metadata}}).
3. The request proceeds under the effective metadata. No persistent registration is created.
4. The state the rest of the grant depends on persists as an establishment ({{grant-lifecycle}}).

# Conventions and Definitions

{::boilerplate bcp14-tagged}

OAuth terminology is defined by {{RFC6749}}. Client metadata and software statement terminology is defined by {{RFC7591}}. Client ID Metadata Document terminology is defined by {{CIMD}}. Statement issuance terminology, including Issuing Authorization Server, Trusting Authorization Server, and the statement format and validation rules, is defined by {{ISSUANCE}}.

This specification additionally defines the following terms:

Runtime Presentation:
: The consumption of a validated software statement inside an authorization or token request, applying its attested metadata to that request without creating a persistent client registration.

Statement-Governed Registration:
: An {{RFC7591}} client registration whose validity is bound to a software statement's `exp` and renewed by replacement statements ({{registration-validity}}).

Establishment:
: The durable state a successful runtime presentation creates for the grant it opens, enumerated in {{grant-lifecycle}}.

Proven Key:
: The key for which the presenter demonstrates possession during runtime presentation. The accepted proof path binds this key to the statement as specified in {{sender-constraint}}.

# Statement Profiles {#profiles}

This document consumes the two statement shapes {{ISSUANCE}} defines, discriminated by the presence of `cimd_digest`: the DCR profile is its non-CIMD shape, the CIMD profile its CIMD-anchored shape. The syntax, validation, and semantics of every element are those of {{ISSUANCE}}; this section states which elements each profile requires and what each is consumed for. How the client acquired the statement is out of scope: the issuance flows of {{ISSUANCE}} produce CIMD-profile statements, DCR-profile statements are issued by out-of-band processes such as an enterprise review console, and neither profile requires a particular acquisition path.

## Common Elements {#profile-common}

Both profiles require the following, in addition to the `typ` header value `software-statement+jwt`, whose explicit typing prevents confusion with other JWTs from the same issuer:

`iss`:
: REQUIRED. Identifies the issuer for trust, claim-authority, and scope decisions, and forms part of the statement identity recorded with a registration or establishment.

`aud`:
: REQUIRED. Scopes which authorization servers may accept the statement; without it a statement would be usable at every server that trusts the issuer.

`iat`:
: REQUIRED. The issuance time.

`exp`:
: REQUIRED. The expiry the issuer chose. Under the DCR profile it is the registration's validity ({{registration-validity}}); under the CIMD profile it is checked at every presentation and at the validity points of {{grant-lifecycle}}. In both profiles it is the issuer's continuous-governance lever: renewal extends standing, ceased renewal ends it.

`jti`:
: REQUIRED. Identifies the statement within the issuer's namespace; inventory and concurrency bounds can key on the `iss` and `jti` pair. A replacement statement has its own `jti`, so replacement matching does not key on this value.

## DCR Profile {#dcr-profile}

For client software registered through {{RFC7591}} and identified by `software_id`:

`sub`:
: REQUIRED. The software's {{RFC7591}} `software_id`. The statement binds to a registration request by the identifier rules of {{ISSUANCE}}: a request whose `software_id` differs from `sub` is rejected.

`software_version`:
: OPTIONAL. The version the review covered; when attested, a request whose `software_version` differs is rejected ({{ISSUANCE}}), and a new version requires re-issuance ({{version-changes}}).

`cimd_digest`:
: MUST be absent; its absence is what marks this profile. Attested metadata is inline only, and the statement's own claims carry the entire reviewed content.

A DCR-profile statement is consumed at registration and at renewal ({{dcr-presentation}}). It cannot be presented at runtime: the runtime rules assume the document a CIMD subject names.

## CIMD Profile {#cimd-profile}

For a client identified by a Client ID Metadata Document URL, each element is consumed by a specific runtime mechanism, and a statement lacking any of them cannot be presented:

`sub`:
: REQUIRED. The client's Client ID Metadata Document URL; the request's `client_id` equals it, by the rule of {{runtime-presentation}}.

`cimd_digest`:
: REQUIRED. Binds the issuer's review to the exact document bytes evaluated and feeds the post-issuance change policy of {{ISSUANCE}}.

A CIMD-profile statement is presented at runtime ({{runtime-presentation}}), and also serves a registered client whose `client_id` is its CIMD URL ({{registered-delivery}}). A statement failing this profile is rejected as {{errors}} defines for an invalid statement.

# Presentation at Registration {#dcr-presentation}

Under the DCR profile, the statement is consumed where {{RFC7591}} always put it: the `software_statement` member of a registration request, validated and bound as {{ISSUANCE}} requires, with rejections using the {{RFC7591}} error codes given there. This section adds what {{RFC7591}} left undefined: the registration's relationship to the statement's lifetime.

## Registration Validity {#registration-validity}

An authorization server operating statement-governed registrations MUST record the governing statement's identity, its `iss`, `jti`, and `sub`, and its `exp` with the registration. The registration is valid until that `exp`: when it passes without a replacement ({{revalidation}}), the authorization server MUST treat the registration as expired, and requests under an expired registration fail as for an unknown client, `invalid_client` at the token endpoint. Registration expiry does not revoke tokens already issued; outstanding grants are local policy, and short access token lifetimes bound the tail.

The client needs no new signal to learn its registration's validity: it holds the statement, and the registration's validity is the statement's `exp`. Whether to operate registrations under this model is deployment policy; a server that does not adopt it consumes the statement as plain {{RFC7591}} input and this section does not apply.

## Revalidation {#revalidation}

The client renews a statement-governed registration by delivering a replacement statement, in either of two ways:

* in the `software_statement` parameter of a token request under the registration, authenticated as the registered client under the registration's own method; or
* in a new {{RFC7591}} registration request, or an update where the deployment offers {{RFC7592}} registration management.

The replacement MUST validate under {{ISSUANCE}} with this server in its audience and MUST have the governing statement's `iss` and `sub`. On success the authorization server MUST replace the recorded statement identity and `exp` atomically; the registration's validity extends to the replacement's `exp`. Whether attested changes in the replacement update the registration record is local registration policy. A refresh-token request that omits a replacement the validity model requires, or delivers one failing these rules, is rejected with `invalid_grant`; any other token request under an expired registration is rejected with `invalid_client`.

## Version Changes {#version-changes}

A review covers the software the issuer evaluated, and `software_version` is the DCR profile's change signal: its value changes on any update to the software. When the statement attests it, the binding rules of {{ISSUANCE}} reject a registration or replacement whose `software_version` differs, so a new version requires a new review and a statement covering it, and the client updates its registration alongside. A version string is a vendor-asserted label, not a byte binding; the guarantee is review-of-what-the-vendor-calls-that-version.

The CIMD profile's change signal is `cimd_digest`, which is byte-exact: a mismatch against the currently published document is a policy input under {{ISSUANCE}}, and re-issuance against the new document is the remedy. In both profiles the statement's bounded lifetime caps how stale a review can get: drift the change signal misses still expires with the statement.

# Runtime Presentation {#cimd-presentation}

Under the CIMD profile, the client presents its statement inside the request itself, and the authorization server validates the statement under policy for a trusted issuer and applies the attested metadata to that request, creating no persistent client registration. This establishes an otherwise unregistered client at request time, as {{CIMD}} resolution does, with the issuer's review carried inline instead of fetched.

## Presentation Request {#runtime-presentation}

A client presents a software statement by including the following parameter in a token request or a pushed authorization request:

`software_statement`:
: REQUIRED for presentation. The software statement ({{cimd-profile}}).

The request's `client_id` is the client's Client ID Metadata Document URL. It MUST exactly equal the statement's `sub`; the authorization server MUST reject a presentation where they differ. The effective `client_id` is the statement's `sub`, and the authorization server assigns none.

The request MUST also carry the proof inputs required by the accepted sender-constraint mode: client authentication parameters, a DPoP proof, the fields defined by {{ABCA}}, or a Client Instance Assertion as permitted by {{CLIENT-INSTANCE}}. A successful presentation establishes the client only for the request and the grant it opens.

A presentation is compatible with an access-granting request: it is client establishment carried alongside the request, not a request for issuance. A request that initiates issuance under {{ISSUANCE}} MUST NOT carry the `software_statement` parameter; the authorization server rejects the combination with `invalid_request`. A request that carries the parameter and is neither a presentation, a revalidation delivery ({{revalidation}}), nor a registered-client delivery ({{registered-delivery}}) is rejected with `invalid_request`.

An authorization server advertises support through `software_statement_presentation_supported` ({{authorization-server-metadata}}).

### Authorization Endpoint {#authorization-requests}

Presentation in an authorization request MUST use a pushed authorization request {{RFC9126}}. The statement and its proof are sent to the pushed authorization request endpoint, where the processing rules of {{processing}} apply. The subsequent authorization request MUST use a `client_id` exactly equal to the establishment's `sub`. A statement never appears in a front-channel URL, just as {{ISSUANCE}} keeps the issued statement out of authorization responses.

### Token Endpoint

Presentation at the token endpoint requires no additional profile: the client includes the parameter in the token request alongside the grant.

## Presentation Processing {#processing}

On receiving a presentation, the authorization server proceeds as follows, rejecting as {{errors}} defines at the first failure:

1. Validate the statement under {{ISSUANCE}}: format, issuer trust, audience, lifetime, and namespace, with digest comparison as the policy input defined there. Complete all validation that does not require a client-controlled network retrieval before performing such a retrieval.
2. Verify the profile of {{cimd-profile}} and the `client_id` rule of {{runtime-presentation}}.
3. Verify the sender-constraint chain ({{sender-constraint}}).
4. Derive the effective metadata and evaluate the request against it ({{effective-metadata}}).
5. On success, create the establishment ({{grant-lifecycle}}).

## Sender Constraint {#sender-constraint}

A runtime presentation MUST be sender-constrained, and the constraint MUST chain to the statement. The presenter proves possession of the Proven Key through the applicable client authentication method, a DPoP proof {{RFC9449}}, or a proof-of-possession mechanism defined by {{ABCA}} or {{CLIENT-INSTANCE}}. The authorization server accepts exactly one of the following modes:

Directly Attested Key:
: If the statement authoritatively attests `jwks` or `jwks_uri`, the Proven Key MUST appear in that key set. When effective metadata specifies a client authentication method, the presenter MUST use that method. A DPoP proof by itself is sufficient only when its key is directly attested and the effective metadata does not require another authentication method. No endorsement mode can bypass authoritatively attested key material.

Locally Trusted Client Attestation:
: This mode is available only when the statement does not authoritatively attest key material. The authorization server MUST validate the client attestation and its proof according to {{ABCA}}. The attestation subject MUST exactly equal the statement's `sub`, its confirmation claim MUST identify the Proven Key, and the client attester MUST be trusted for that exact subject or its configured namespace.

Attested Instance Issuer:
: This mode is available only when the statement does not authoritatively attest key material and authoritatively attests `instance_issuers`. At the token endpoint, the request MAY carry a Client Instance Assertion as specified by {{CLIENT-INSTANCE}}. The authorization server MUST validate it against an attested issuer descriptor, and the assertion's sender key is the Proven Key. This specification does not define this mode at the pushed authorization request endpoint because {{CLIENT-INSTANCE}} does not define its wire parameters there.

Each mode is gated by the authority behind it: an attested member counts only where the server treats the statement issuer as authoritative for that member ({{ISSUANCE}}), and attester trust counts only within its configured `sub` scope. The authorization server MUST reject a presentation without such a proof, or whose Proven Key no accepted mode covers.

A statement that attests neither key material nor an instance-key delegation, presented at a server whose attester trust does not cover its `sub`, cannot satisfy this section and is consumable only through registration.

Verifying a key at the attested `jwks_uri` is a retrieval at presentation time. A fetch failure leaves the chain unverified, and the presentation is rejected as `invalid_client`; the server MUST NOT fall back to a weaker mode. A server MAY reuse a recently retrieved key set within ordinary HTTP caching bounds ({{external-retrieval}}).

## Effective Metadata {#effective-metadata}

Having validated the statement and its proof, the authorization server derives the client's effective metadata for the request:

* An attested member for which the server treats the issuer as authoritative ({{ISSUANCE}}) has the value the statement gives it, with the precedence {{RFC7591}} defines.
* An attested member outside that set does not take attested precedence; per the policy rule of {{ISSUANCE}}, the server ignores it or treats it as client-asserted.
* A member the statement does not attest takes its value from the client's current Client ID Metadata Document, the document at the statement's `sub`, or from a default defined by {{RFC7591}} where that default is compatible with {{CIMD}} and this runtime model. Such a member is client-asserted metadata, not part of the review. The authorization server MUST resolve that document when the request depends on an unattested member. In particular, a shared-secret authentication method cannot be inferred because no client secret is assigned by presentation and {{CIMD}} does not support shared-secret client authentication.

The effective metadata is therefore attested member by member, not wholesale. The authorization server MAY require specific members, notably the redirection URIs of a redirect flow, to be attested, rejecting a presentation whose statement omits them.

The request is evaluated against the effective metadata: a `redirect_uri` MUST match an effective redirection URI, and any requested grant type, response type, or scope MUST fall within the effective metadata. A grant or response type supported by the authorization server but not authorized by the effective metadata fails with `unauthorized_client`; a scope outside the effective metadata fails with `invalid_scope`.

A trusting authorization server SHOULD use the statement's `sub`, `iss`, and `jti` to inventory the establishments derived from one statement and MAY bound their concurrent number.

## Grant Lifecycle {#grant-lifecycle}

A successful presentation creates an establishment comprising:

* the validated `sub`;
* the statement identity, its `iss` and `jti`, and its expiry;
* the effective metadata ({{effective-metadata}});
* the issuer trust decision; and
* the sender-constraint mechanism and Proven Key.

The establishment persists for the grant it opens. The authorization server MUST bind the resulting `request_uri` and any authorization code to the establishment. The token request that redeems the code MUST have a `client_id` exactly equal to the establishment's `sub`, MUST demonstrate possession of the same Proven Key under the same sender-constraint mechanism, and MUST NOT carry the `software_statement` parameter. A redemption carrying a statement is rejected with `invalid_request`; a wrong client identifier or failed key binding is rejected with `invalid_grant`.

A statement MUST be unexpired when presented. Expiry after presentation does not invalidate an establishment already bound, just as expiry does not undo an {{RFC7591}} registration.

### Refresh {#refresh}

On refresh-token use the authorization server MUST verify possession of the establishment's Proven Key under the same sender-constraint mechanism. It MAY, by local policy, additionally require a current unexpired statement.

When policy requires one, the client presents the replacement in the `software_statement` parameter of the refresh request. The replacement:

* MUST validate under {{ISSUANCE}} with this server in its audience;
* MUST have the establishment's `iss` and `sub`; and
* MUST authorize the establishment's Proven Key ({{sender-constraint}}).

The refreshed access MUST fall within the replacement's effective metadata; the grant never widens beyond the original authorization. Because the replacement must authorize the existing Proven Key, this operation does not rotate the establishment's key. A client that needs a new key performs a new presentation and opens a new establishment. On success, the establishment's statement identity, expiry, effective metadata, and trust decision are replaced atomically. A refresh that fails these requirements, or omits a statement that policy requires, is rejected with `invalid_grant` and leaves the establishment unchanged.

## Registered CIMD Clients {#registered-delivery}

A client can be registered at an authorization server under its Client ID Metadata Document URL as its `client_id` and also present at runtime. Local policy decides whether to accept presentations alongside the registration. When both exist, the registration record governs requests that do not carry a statement, and the presented statement's effective metadata governs the presented request only.

Such a registration can also be statement-governed: the validity and revalidation model of {{registration-validity}} and {{revalidation}} applies, with the delivered statement's `sub` equal to the registered `client_id` and validated under the CIMD profile. The request authenticates as the registered client, under the registration's own method; the delivered statement renews validity and does not otherwise alter the registration. A refresh-token request that omits a statement this policy requires, or delivers one failing these rules, is rejected with `invalid_grant`; any other token request failing the currency requirement is rejected with `invalid_client`.

# Deployment Model: Centrally Curated Software {#deployment-model}

This section is non-normative.

An enterprise that reviews and approves client software operates a statement issuer. Approval of an application is the issuance of a short-lived statement: the DCR profile for software its providers register through {{RFC7591}}, the CIMD profile for software with hosted metadata, each with `aud` naming the authorization servers of the providers where the approval should hold. Renewal is automatic while the approval stands, so registrations stay valid and presentations keep succeeding.

The controls fall out of the lifetime machinery. Onboarding a provider is one trust configuration, the issuer and its identifier scope, after which every approved application arrives carrying its approval instead of being re-keyed into a console. Offboarding an application is ceasing renewal: its registrations expire at their recorded `exp` at every provider simultaneously, and its presentations stop at the same boundary, with no per-server deprovisioning. Narrowing an approval is re-issuing with narrower attested metadata, which takes effect at the next revalidation or refresh. The enterprise's allowlist stops being N console configurations that drift and becomes one issuance policy that every trusting authorization server enforces on schedule.

# Error Responses {#errors}

A statement consumed at registration is rejected with the {{RFC7591}} error codes as {{ISSUANCE}} defines. A rejected presentation or delivery uses the error responses of {{RFC6749}} for the endpoint at which it was presented. At the token endpoint:

`invalid_client`:
: the statement or its proof fails to establish the client, including a failed profile ({{profiles}}), chain ({{sender-constraint}}), or `jwks_uri` retrieval; also any request under an expired statement-governed registration ({{registration-validity}}).

`invalid_scope`:
: a requested scope falls outside the effective metadata.

`unauthorized_client`:
: the client is not permitted by its effective metadata to use the requested grant type or response type, although the authorization server supports it.

`invalid_request`:
: any other rejection, including a redemption or issuance request carrying the `software_statement` parameter ({{runtime-presentation}}, {{grant-lifecycle}}).

At the pushed authorization request endpoint, the corresponding {{RFC9126}} error responses apply. On refresh-token use, {{refresh}}, {{revalidation}}, and {{registered-delivery}} take precedence: failures there are `invalid_grant`, the grant's continuation failing rather than client establishment.

When an otherwise valid proof protocol defines a recoverable, proof-specific error, including an error that supplies a DPoP nonce or client-attestation challenge, or requests a fresh attestation, that error from {{RFC9449}}, {{ABCA}}, or {{CLIENT-INSTANCE}} takes precedence over the generic errors above.

This surface is coarser than the registration codes and does not tell a client whether to seek a corrected statement or a different issuer. The authorization server MAY return non-sensitive diagnostics in `error_description` to a client it has authenticated, but MUST NOT reveal issuer-trust, subject-namespace, or attester-policy details to an unauthenticated requester.

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

The authorization server validates the replacement, matches its `iss` and `sub` to the registration's governing statement, and atomically extends the registration's validity to the replacement's `exp`. Had the request arrived after expiry without a replacement, it would have failed with `invalid_grant`.

## Presenting at the Token Endpoint

The following presents an already-issued CIMD-profile statement at the token endpoint, sender-constrained by a client attestation {{ABCA}}. The `OAuth-Client-Attestation` header carries an attestation of the instance key from a client attester the server trusts for the statement's `sub`, with that `sub` as its subject; the `OAuth-Client-Attestation-PoP` header proves that key; the `software_statement` parameter carries the issuer's review; and `client_id` is the Client ID Metadata Document URL named by the statement's `sub`. No client record exists at this server.

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
: OPTIONAL. Boolean value indicating whether the authorization server accepts a software statement presented at runtime in an authorization or token request ({{runtime-presentation}}), rather than only through {{RFC7591}} registration. If omitted, the default value is `false`. This member describes the consuming role. A value of `true` advertises the path as this specification defines it, at every endpoint the server's supported grant types make applicable; there is no per-endpoint signal. It does not imply acceptance of any particular statement issuer, subject namespace, attested member, or sender-constraint mode, and it does not signal the registration-validity model of {{registration-validity}}, which needs no signal because the client holds the statement whose `exp` bounds it. A client also examines the authorization server metadata for the proof mechanism it intends to use, including client-authentication and DPoP metadata, `client_attestation_pop_methods_supported` from {{ABCA}}, and `client_instance_assertion_supported` from {{CLIENT-INSTANCE}}.

# Extension Points {#extensions}

This specification defines runtime presentation through the `software_statement` request parameter. Additional endpoint-specific discovery and proof modes are left to extensions. Two other profiles of the same path are also left to extensions: carrying the statement within a client attestation {{ABCA}}, and a Client ID Metadata Document that references where the client publishes its current statements, so a resolving server fetches the review out of band. The latter differs from embedding a statement in the document, which the digest rule of {{ISSUANCE}} forbids. A discovery signal finer than `software_statement_presentation_supported`, advertising which chains a server accepts, is likewise left to an extension.

# Security Considerations {#security-considerations}

## Statement Theft

A runtime presentation is safe for an unregistered presenter because the proof chains to the statement. Possession of an arbitrary key constrains only the request; the chain to the statement makes possession of a stolen statement insufficient unless the attacker also controls a key covered by a mode the authorization server accepts. The registration path cannot offer this property; {{ISSUANCE}} bounds a stolen statement's use there through audience, lifetime, and registration limits instead, and a stolen DCR-profile statement is additionally inert at servers where the software is already registered, since the binding rules match it to an existing registration rather than creating standing.

Restricting the accepted chain by what the statement attests prevents downgrade: when key material is attested, no other chain is accepted, so a reviewed key cannot be bypassed by a presenter selecting a weaker proof form. The authority gates of {{sender-constraint}} serve the same end: an issuer trusted for other metadata, or an attester trusted for other subjects, does not authorize runtime keys.

## Validation Scope

A consumed statement is validated under the full ruleset of {{ISSUANCE}}: configured issuer trust with keys obtained from authorization server metadata and never from the statement, audience and lifetime checks, and identifier-scope authorization for the `sub`; digest comparison remains the policy input {{ISSUANCE}} defines. Nothing in this specification relaxes those rules; it adds the sender-constraint chain, the grant bindings of {{grant-lifecycle}}, and the registration-validity model on top of them.

## Key Location versus Keys

The `jwks_uri` chain verifies keys at an attested location, not attested keys: `cimd_digest` binds the metadata document's bytes and never the key set served at the URI, so compromise of the client's key host adds keys that satisfy the chain with no digest signal. Deployments for which that exposure is unacceptable prefer statements that attest `jwks` inline, at the cost of digest-visible rotation, or pin observed keys and alert on change. A server reusing a cached key set under {{sender-constraint}} additionally accepts that a just-removed key can briefly continue to verify.

## External Retrieval and Resource Exhaustion {#external-retrieval}

Runtime presentation can cause the authorization server to retrieve the Client ID Metadata Document, an attested `jwks_uri`, or keys and metadata associated with an attested instance issuer. Every such retrieval inherits the server-side request forgery protections of {{CIMD}}. A trusted signature does not make a URL safe: the authorization server MUST apply its URL, redirect, address-range, transport, and content-type policy independently to every referenced location.

Before initiating a client-controlled retrieval, the authorization server MUST complete the statement checks that do not depend on that retrieval, including signature, issuer, audience, lifetime, subject, and claim-contract validation. It SHOULD bound JWT size and parsing work, concurrent retrievals, response size, and response time, and SHOULD cache successful and failed retrieval results for an appropriate period. A retrieval failure leaves the relevant metadata or proof chain unverified; the authorization server MUST reject the request and MUST NOT fall back to a weaker sender-constraint mode.

## Enforcement Bounds

Expiry is enforced at every presentation and at every statement-governed registration, so a lapsed statement stops new grants at once and lapses the registration at its recorded `exp`, a continuous-enforcement property a plain persistent registration does not provide. Grants already open continue under {{grant-lifecycle}}; the refresh-time statement requirements of {{refresh}}, {{revalidation}}, and {{registered-delivery}} are the policy levers that wind down existing access when an issuer ceases renewal, and a narrowed re-review takes effect through the replacement's effective metadata or the updated registration. Detection of post-issuance metadata change depends on the change signal each profile carries, `software_version` as a vendor-asserted label, `cimd_digest` as exact bytes with retrieval-dependent enforcement, and the statement's bounded lifetime caps what either signal misses.

Renewal cadence is a deployment trade: short lifetimes tighten the issuer's control loop and increase issuance and delivery traffic, and a fleet of registrations issued together expires together, so issuers SHOULD stagger expiries or renew ahead of the boundary to avoid synchronized lapses.

## Client-Asserted Metadata

Members the statement does not attest are client-asserted, and omission can mean the issuer declined to attest a value. The attested-members-required policy of {{effective-metadata}} is the control for deployments that do not want client-asserted values, notably redirection URIs, entering effective metadata. Resolving a Client ID Metadata Document for unattested members inherits the resolution considerations of {{CIMD}}, including server-side request forgery and availability; a server MAY cache resolution results within the document's caching directives, and the statement's digest binds the review to specific bytes regardless of cache state.

## Statement Handling

A software statement remains a sensitive artifact in transit and at rest: possession alone does not enable presentation, but statements reveal attested metadata and audience relationships, and servers SHOULD avoid logging them.

Error responses can also disclose trust configuration. The restrictions in {{errors}} prevent unauthenticated probing of issuer, namespace, and attester policy.

# Privacy Considerations

A presentation or delivery reveals to the authorization server the client's issuer relationship and the full attested metadata, including the audience list, which names the other authorization servers the client intends to establish relationships with. Issuers can bound the disclosure by keeping audiences narrow, as {{ISSUANCE}} recommends. The pushed authorization request requirement of {{authorization-requests}} keeps statements out of browser history, referrers, and front-channel logs. A central issuer additionally learns, through renewal requests, which of its statements are in active use; issuance and renewal logs deserve the same care as the statements themselves.

# IANA Considerations {#iana}

## OAuth Parameters Registry

This specification requests registration of the `software_statement` parameter in the IANA "OAuth Parameters" registry established by {{RFC6749}}, for runtime presentation and statement delivery of the artifact defined by {{RFC7591}}:

The Dynamic Client Registration Metadata registry already contains a metadata member with the same name. That entry is in a different registry and is unaffected by this registration.

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
