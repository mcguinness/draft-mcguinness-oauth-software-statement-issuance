---
title: "CIMD Software Statement Registration"
abbrev: oauth-cimd-sw-stmt-registration
docname: draft-mcguinness-oauth-cimd-sw-stmt-registration-latest
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
  RFC7591:
  RFC8414:
  RFC9126:
  RFC9700:
  STATEMENT:
    target: https://datatracker.ietf.org/doc/draft-mcguinness-oauth-cimd-sw-stmt
    title: "CIMD Software Statement"
  CIMD:
    target: https://datatracker.ietf.org/doc/draft-ietf-oauth-client-id-metadata-document
    title: "OAuth Client ID Metadata Document"

informative:
  RFC7592:
  ISSUANCE:
    target: https://datatracker.ietf.org/doc/draft-mcguinness-oauth-cimd-sw-stmt-issuance
    title: "CIMD Software Statement Issuance"
  UK-OPEN-BANKING:
    target: https://openbankinguk.github.io/dcr-docs-pub/v3.3/dynamic-client-registration.html
    title: "Open Banking UK Dynamic Client Registration"
  AU-CDR:
    target: https://consumerdatastandardsaustralia.github.io/standards/
    title: "Consumer Data Standards"

--- abstract

RFC 7591 does not define how long a registration based on a software statement remains valid or how that statement is renewed. This specification defines how an authorization server consumes a software statement for a Client ID Metadata Document in a registration request, taking every metadata value from the reviewed document; how the statement's expiry bounds the resulting registration; and how a client renews the registration with a replacement statement. It builds on the companion specification that defines the statement and its runtime consumption.

--- middle

# Introduction

{{RFC7591}} defines no expiry or renewal procedure for a dynamic client registration. A software statement (Section 2.3 of {{RFC7591}}) can carry a reviewer's approval into a registration request, but the registration can outlive the statement and the review it represents, leaving an organization that reviews client software no interoperable way to keep that review current at the authorization servers that relied on it.

Two regulated ecosystems already use statements at registration. The UK Open Banking Directory and the Australian Consumer Data Right Register each run a central issuer whose statements many unrelated authorization servers consume through the {{RFC7591}} `software_statement` member ({{UK-OPEN-BANKING}}, {{AU-CDR}}). Both carry client metadata in the statement rather than binding it to a document the consumer retrieves.

{{STATEMENT}} defines the statement, its validation, issuer trust configuration, and runtime consumption. This specification defines what registration adds: taking the registration's metadata from the reviewed document ({{dcr-presentation}}), bounding its validity by the statement ({{registration-validity}}), and renewing it with a replacement statement ({{revalidation}}). It depends on {{STATEMENT}}; {{STATEMENT}} does not depend on it.

## Protocol Overview

The following non-normative sequence summarizes a statement-governed registration:

1. The client registers through {{RFC7591}}, carrying a statement in the `software_statement` member.
2. The authorization server derives the registration's metadata from the reviewed document and records the statement's identity and `exp`; the registration is valid until that expiry, or earlier where the server caps statement lifetime ({{registration-validity}}).
3. The client delivers a replacement statement in an authenticated token or pushed authorization request or, where supported, an {{RFC7592}} update request, extending the registration's validity to the replacement's `exp` under the same cap.
4. If no replacement arrives, the registration expires and requests under it fail.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

OAuth terminology is defined by {{RFC6749}}. Client metadata and software statement terminology is defined by {{RFC7591}}. Client ID Metadata Document terminology is defined by {{CIMD}}. Runtime Presentation and Refusal Record are defined by {{STATEMENT}}.

This specification defines the following term.

Statement-Governed Registration:
: An {{RFC7591}} client registration, at a server advertising `software_statement_registration_validity_supported`, whose validity is bound to a software statement's `exp` and renewed by replacement statements ({{registration-validity}}).

# Consumption at Registration {#dcr-presentation}

The statement is consumed in the `software_statement` member of an {{RFC7591}} registration request. The authorization server validates it under the validation rules of {{STATEMENT}} and registers the client under its ordinary registration policy. This section defines how the registration's metadata is derived; its subsections define the registration's validity and renewal.

In {{RFC7591}}, values in a software statement take precedence over those in the request; a statement under {{STATEMENT}} carries no client metadata, so the reviewed document takes that role. An authorization server consuming a statement under this specification:

* MUST resolve the Client ID Metadata Document at the statement's `sub`;
* MUST validate the document as {{CIMD}} requires, including that its `client_id` member matches the client identifier URL it is held for, which is the statement's `sub`;
* MUST verify that the digest of the retrieved representation equals `cimd_digest`, and MUST derive the registered metadata by parsing the same octets it digested, not a second retrieval or a cached copy it has not digested;
* MUST take every client metadata value from that document, and MUST NOT take any client metadata value from the registration request, whether or not the document carries that member. A request MAY carry metadata, as {{RFC7591}} clients do; it does not contribute to the registration. The registered `client_id` is assigned as {{RFC7591}} provides, and the document's own `client_id` member is the URL recorded as `sub`;
* never consumes a statement from a `software_statement` member of the reviewed document, which {{STATEMENT}} forbids at every consumption point; and
* MUST reject the registration with `invalid_client_metadata` where any of the document's redirection URIs could be claimed by another application on the same device (see the public-client presentation rules of {{STATEMENT}}), whatever authentication method the document declares.

Such a redirection URI delivers authorization codes to whichever local application claims it. Registration has no review-only form, as runtime presentation does ({{STATEMENT}}), because the registration is the admission.

Taking every value from the document prevents metadata substitution. Constraining only the members the document carries would leave every omitted member attacker-supplied: a document naming `jwks_uri` and no `jwks` would admit a request-supplied `jwks`, giving a holder of someone else's statement a registration with reviewed branding and its own key.

An authorization server that pre-registers a client identifier URL, which {{CIMD}} permits and names as the expected enterprise pattern, participates by retaining the exact octets it digested when it onboarded that URL. Such a server MAY satisfy the resolution requirement above by comparing `cimd_digest` against the digest of the retained octets, and MUST retrieve the document afresh where that comparison fails. It SHOULD revalidate retained octets on the schedule the document's caching directives allow, for example with a conditional request, since otherwise a statement over an older document keeps matching bytes the publisher no longer serves. A server holding no such octets retrieves the document.

The authorization server handles three failures as follows:

* Where the digest does not match, the document has changed since review, and the authorization server MUST reject the registration with `invalid_software_statement`; the remedy is re-issuance over the current document.
* Where the document carries metadata the authorization server's policy refuses, it rejects the registration with `invalid_client_metadata`, since the refused values are the client's own.
* Where the retrieval does not complete, the authorization server MUST reject with `temporarily_unavailable` and SHOULD use HTTP status code 503, so that a client retries rather than discarding a sound statement.

The retrieval is client-controlled and reachable before any client is registered, so the external-retrieval protections and resource bounds of {{STATEMENT}} apply to it as to a runtime presentation.

## Registration Validity {#registration-validity}

An authorization server that advertises `software_statement_registration_validity_supported` as `true` MUST apply the following validity model to every registration it creates from a validated software statement, whichever issuer signed it, so that a client can rely on the signal ({{authorization-server-metadata}}) before it registers:

* It MUST record the governing statement's `iss`, `jti`, `sub`, and `iat`, and its `status` claim where it carries one, with the registration, and the registration's effective expiry: the earlier of the statement's `exp` and its `iat` plus the maximum statement lifetime the server records for that issuer in its issuer trust configuration ({{STATEMENT}}). The effective expiry is what a renewal extends and what `registration_expires_at` reports.
* The registration is valid until that effective expiry.
* Once the effective expiry passes without a replacement ({{revalidation}}), it MUST reject requests under the registration, other than the revalidation requests {{revalidation}} permits: with `invalid_client` at the token and pushed authorization request endpoints, and with `statement_required` at the authorization endpoint ({{errors}}).
* It SHOULD retain the expired record so that it can process a later authenticated revalidation ({{oracle-considerations}}); since a valid replacement restores the registration ({{revalidation}}), no grace period is needed.

The effective expiry is an upper bound. Where the server resolves status for the governing statement's issuer and that status resolves as `INVALID`, or as `SUSPENDED` where its policy for that issuer refuses it ({{STATEMENT}}), the registration ceases to be valid as it does at its effective expiry, and {{revalidation}} is the recovery path; the client learns of the withdrawal only from the rejection. A server that does not resolve status is bounded by the effective expiry alone.

The disposition of outstanding grants is local policy ({{enforcement-bounds}}).

If {{CIMD}} defines an expiry that a document asserts for its own client identifier, that value MAY only shorten the effective expiry and MUST NOT extend it, so that an issuer cannot lengthen the life of a client identifier its subject has declared ephemeral.

An authorization server advertising this model MUST publish `pushed_authorization_request_endpoint`, since {{revalidation}} otherwise leaves a client whose only grant type is the authorization code, and which holds no refresh token, with no way to renew.

A server applying this model MUST return a `registration_expires_at` member, a NumericDate giving the latest time the registration remains valid, in the {{RFC7591}} registration response and in the response to any request that renews the registration ({{revalidation}}), and SHOULD return it in any {{RFC7592}} read or update response it supports. A client cannot compute this boundary from the statement alone, because the server can cap the lifetime it honors for the issuer ({{STATEMENT}}).

A server that omits the signal or advertises `false` does not bound registrations by statement expiry but still consumes the statement as {{dcr-presentation}} requires, since a statement carries no metadata and consuming one as ordinary {{RFC7591}} input would leave the registration self-asserted.

## Revalidation {#revalidation}

A client renews a statement-governed registration by delivering a replacement statement in one of the following:

* the `software_statement` parameter of a token request under the registration, authenticated as the registered client under the registration's own method;
* the `software_statement` parameter of an authenticated pushed authorization request {{RFC9126}} under the registration; or
* the `software_statement` member of an authenticated {{RFC7592}} update request, where the deployment offers registration management.

The update request carries the complete client metadata that {{RFC7592}} requires, but that metadata is not authoritative: the renewed record is derived from the reviewed document.

The replacement MUST:

* validate under the validation rules of {{STATEMENT}}, including its audience where it carries one;
* carry the governing statement's `iss` and `sub`;
* be unexpired; and
* have an `iat` later than the recorded statement's `iat`.

For a replacement, the authorization server MUST:

* obtain the Client ID Metadata Document at the replacement's `sub`, by retrieval or from retained octets as {{dcr-presentation}} permits;
* verify that the digest of those octets equals the replacement's `cimd_digest`; and
* re-derive the registration's metadata from those same octets, under the rules of {{dcr-presentation}}.

Renewing on the statement alone would leave the registration carrying metadata the new review never covered, so a removed key, redirection URI, or scope would survive its own withdrawal.

On success, the authorization server MUST replace the recorded statement identity, `iat`, `exp`, and derived metadata in a single atomic update, and concurrent deliveries resolve to the most recently issued statement.

When a request under an expired registration contains a replacement, the authorization server MUST authenticate the client against the retained registration and evaluate the replacement before applying the expiry rejection. A valid replacement therefore restores the registration; an omitted or invalid replacement does not.

The rejection of a request under an expired registration that carries no replacement ({{registration-validity}}) MUST NOT by itself trigger the refresh-token family revocation of {{RFC9700}}; the client recovers by delivering a valid replacement.

A request whose replacement fails the rules above is rejected, whether or not the registration has expired, with `statement_required` at the token and pushed authorization request endpoints and with the codes of {{errors}} at the registration management endpoint, so that the client learns of the failure at once and can tell a bad replacement from a missing one. A rejected delivery leaves the recorded statement unchanged.

A registration request without an {{RFC7592}} registration access token creates a new registration and never renews an existing one.

# Statements from an Established Client {#registered-delivery}

A client already established at an authorization server, whether registered through {{RFC7591}} or under its Client ID Metadata Document URL as its `client_id`, can still carry a statement. Runtime presentation ({{STATEMENT}}) establishes clients the server does not have and does not reopen metadata for one it has. A statement from an issuer the server does not accept for that client is rejected as {{errors}} defines.

For a statement-governed registration, a statement from the governing issuer is a delivery under {{revalidation}}: the request authenticates as the registered client under the registration's own method, the statement's `sub` equals the `sub` recorded for the registration, and the statement renews validity and re-derives the registration's metadata. Where the registration is still valid and the server requires a current statement, a refresh-token request that omits one or delivers one failing these rules is rejected with `statement_required`. Where the registration has already expired, {{revalidation}} governs, including its error codes.

Where the registration is not statement-governed, a statement neither renews nor alters it; the server applies a reviewed change, if at all, through its own registration policy or {{RFC7592}}.

# Repeated Registration {#repeated-registration}

One unexpired statement can be consumed more than once ({{STATEMENT}}), and a server whose policy permits it can create more than one registration from it. The safe default is one registration per (`iss`, `sub`) at one authorization server, counted across replacements, since a replacement carries a new `jti` and a bound keyed on it would reset at every renewal. On repeated consumption, local policy can reject the request, treat it as idempotent, or create another registration; {{RFC7591}} defines no duplicate-registration protocol. An idempotent response MUST NOT return the existing registration's credentials, such as its `registration_access_token`, since a repeated request can come from any holder of a copy of the statement.

Where the reviewed document carries `jwks` or `jwks_uri`, every registration derived from the statement uses that key material rather than an instance-supplied replacement. Software whose instances hold their own keys therefore cannot register them: {{dcr-presentation}} takes no key from the request, since at registration a statement is a bearer artifact and a request-supplied key would let any holder of a copy register reviewed branding under its own key.

# Error Responses {#errors}

Rejections at a registration endpoint use the error response of Section 3.2.2 of {{RFC7591}}, with the error codes the following table assigns. Rejections at other endpoints use the error responses of {{STATEMENT}}.

| Condition | Registration endpoint |
| --- | --- |
| Malformed, or failing signature or claim validation | `invalid_software_statement` |
| Valid but not acceptable here: issuer not configured, `aud` excludes this server, `sub` outside the issuer's scope, `aud_tenant` not this request's tenant or absent where required, `consumable_at` excludes this point | `unapproved_software_statement` |
| Expired, or refused by a refusal record, including a status resolved as `INVALID`, or as `SUSPENDED` where policy refuses it, or superseded under the `iat` floor of {{STATEMENT}} | `invalid_software_statement` |
| Required statement absent | `unapproved_software_statement` |
| Digest does not match the retrieved document | `invalid_software_statement` |
| Document carries metadata this server's policy refuses | `invalid_client_metadata` |
| Retrieval did not complete | `temporarily_unavailable` |
| Review-only client where reviewed software is required ({{STATEMENT}}) | `invalid_client_metadata` ({{dcr-presentation}}) |

A registration management request ({{RFC7592}}) carrying a failing replacement uses the same codes. An authorization server SHOULD use HTTP status code 503 with `temporarily_unavailable` and 400 with the others.

Registration expiry is reported as `invalid_client` at the token and pushed authorization request endpoints, because the registration rather than the grant has lapsed, and as `statement_required` at the authorization endpoint, which tells the client to deliver a replacement through the pushed authorization request endpoint ({{STATEMENT}}). Neither rejection indicates refresh-token replay ({{RFC9700}}).

# Example {#example}

## Revalidating a Statement-Governed Registration

The following non-normative example shows a registered client, `client_id` `s6BhdRkqt3`, renewing its registration by delivering a replacement statement on an ordinary refresh, authenticated under its registered method:

~~~ http
POST /token HTTP/1.1
Host: as.example
Content-Type: application/x-www-form-urlencoded
Authorization: Basic czZCaGRSa3F0Mzo3RmpmcDBaQnIxS3REUmJuZlZkbUl3

grant_type=refresh_token
&refresh_token=tGzv3JOkF0XG5Qx2TlKWIA
&software_statement=eyJ0eXAiOiJzb2Z0d2FyZS1zdGF0ZW1l...
~~~

The authorization server validates the replacement, matches its `iss` and `sub` to the governing statement, re-derives the registration's metadata, and atomically extends the registration's validity, returning the new boundary in the `registration_expires_at` member of the token response. After expiry, the same request without a replacement would fail with `invalid_client` ({{registration-validity}}).

# Authorization Server Metadata {#authorization-server-metadata}

This specification defines the following authorization server metadata {{RFC8414}} value:

`software_statement_registration_validity_supported`:
: OPTIONAL. Boolean value indicating whether every registration the authorization server creates from a validated software statement is governed by the validity and revalidation rules of {{registration-validity}} and {{revalidation}}. If omitted, the default value is `false`. A value of `true` does not imply support for runtime presentation.

# Security Considerations {#security-considerations}

## Statements at Registration {#statement-bearer}

A statement consumed at registration is a reusable bearer artifact until it expires. An issuer limits that exposure with a narrow audience and a short lifetime, and a statement permits registration only where its `consumable_at` claim names registration ({{STATEMENT}}). The repeated-consumption bounds of {{STATEMENT}} and {{repeated-registration}} limit what a stolen statement can create.

## Renewal Authenticates the Credential

Renewal proves possession of the registration's credential and the currency of a statement sharing the governing `iss` and `sub`, not that the renewing party is the reviewed software. An attacker holding a stolen client credential can renew indefinitely with any current statement for that software, since such statements circulate to every deployment of it.

Deployments SHOULD pair statement-governed registrations with credential rotation, sender-constrained client authentication, and the repeated-consumption limits of {{STATEMENT}} and {{repeated-registration}}, and SHOULD treat a credential compromise as requiring re-registration rather than renewal. Runtime presentation does not have this limitation, because every presentation binds the presenter to the reviewed document, by a key the document carries or, for a public client, by the redirection URIs it lists.

## Registration Fraud and Impersonation {#registration-fraud}

Open registration permits `client_name`, `logo_uri`, and `client_uri` values that imitate trusted software on consent screens. Requiring a statement replaces self-asserted branding with issuer-reviewed values. Servers that render such values on consent screens SHOULD prefer those from a reviewed document and SHOULD apply heightened scrutiny to registrations that claim user-visible branding without a statement.

Requiring a statement for registration also means each new client identity needs another issuer decision, so a discarded client cannot return at no cost, and the per-`sub` bounds of {{repeated-registration}} limit how many registrations one piece of software obtains. Neither control makes metadata true: a client that misleads review can obtain a genuine statement for fraudulent metadata, so the depth of the issuer's verification remains decisive ({{ISSUANCE}}).

## Enforcement Bounds {#enforcement-bounds}

A lapsed statement makes requests under a statement-governed registration fail at the recorded `exp`. Grants under the registration remain subject to the server's grant policy after it expires, and a registration at a server that does not advertise the validity model is not bounded by the statement.

## Observable State {#oracle-considerations}

A retained expired registration is distinguishable from an unknown client, because recovery requires the server to authenticate the registration and evaluate a replacement before rejecting. Only a requester that authenticates as the registration's client can observe this, and what it learns is the state it must act on. Servers publishing `software_statement_registration_validity_supported` also disclose that statement-derived registrations expire there, which a client needs to know before registering. Neither disclosure reveals issuer trust, subject scope, or attester policy, which {{STATEMENT}} keeps from unauthenticated requesters.

# Privacy Considerations

A registration request reveals to the authorization server the client's issuer relationship and, where the statement carries an `aud` claim, the other authorization servers the client intends to establish relationships with, as a runtime presentation does ({{STATEMENT}}).

# IANA Considerations {#iana}

## OAuth Dynamic Client Registration Metadata Registry

This specification requests registration of the following client metadata member, returned by an authorization server in a registration response:

Client Metadata Name:
: `registration_expires_at`

Client Metadata Description:
: Latest time at which a statement-governed registration remains valid, as a NumericDate. A withdrawal can end it earlier. Returned in a client registration response, in a registration management response, and in the token or pushed authorization request response to a request that renews the registration.

Change Controller:
: IESG

Specification Document(s):
: This specification, {{registration-validity}}

## OAuth Parameters Registry

This specification requests registration of the following parameter in the IANA "OAuth Parameters" registry established by {{RFC6749}}, for the responses that renew a statement-governed registration:

Parameter Name:
: `registration_expires_at`

Parameter Usage Location:
: token response, pushed authorization request response

Change Controller:
: IESG

Specification Document(s):
: This specification, {{registration-validity}}

## OAuth Authorization Server Metadata Registry

This specification requests registration of the following value in the IANA "OAuth Authorization Server Metadata" registry established by {{RFC8414}}.

Metadata Name:
: `software_statement_registration_validity_supported`

Metadata Description:
: Boolean value indicating whether registrations created from validated software statements are governed by the statement validity and revalidation rules of this specification.

Change Controller:
: IESG

Specification Document(s):
: This specification, {{authorization-server-metadata}}

## OAuth Extensions Error Registry

This specification requests that IANA add this specification as a reference for the `temporarily_unavailable` error and extend its usage location to include the client registration error response:

Error Name:
: `temporarily_unavailable`

Existing Registration:
: {{RFC6749}}

Error Usage Location:
: client registration error response, in addition to the locations already registered

Change Controller:
: IESG

Specification Document(s):
: {{RFC6749}}, this specification ({{errors}})

--- back

# Acknowledgments
{:numbered="false"}

This specification was separated from {{STATEMENT}}, where registration consumption was first defined, so that runtime admission could be implemented without it.
