---
title: "Shared Signals Events for OAuth Software Statements"
abbrev: oauth-sw-stmt-signals
docname: draft-mcguinness-oauth-sw-stmt-signals-latest
category: std

ipr: trust200902
area: "Security"
workgroup: "Web Authorization Protocol"

keyword:
 - OAuth
 - Software Statement
 - Shared Signals
 - Security Event Token
 - Revocation

stand_alone: yes
pi: [toc, sortrefs, symrefs]

author:
 -
    ins: K. McGuinness
    name: Karl McGuinness
    email: public@karlmcguinness.com

normative:
  RFC8414:
  RFC8417:
  RFC9493:
  CIMD:
    target: https://datatracker.ietf.org/doc/draft-ietf-oauth-client-id-metadata-document
    title: "OAuth Client ID Metadata Document"
  ISSUANCE:
    target: https://datatracker.ietf.org/doc/draft-mcguinness-oauth-software-statement-issuance
    title: "OAuth 2.0 Software Statement Issuance"
  PRESENTATION:
    target: https://datatracker.ietf.org/doc/draft-mcguinness-oauth-sw-stmt-presentation
    title: "OAuth 2.0 Software Statement Consumption and Runtime Presentation"
  SSF:
    target: https://openid.net/specs/openid-sharedsignals-framework-1_0.html
    title: "OpenID Shared Signals Framework Specification 1.0"

informative:
  RFC7591:
  RFC8935:
  RFC8936:
  CAEP:
    target: https://openid.net/specs/openid-caep-1_0.html
    title: "OpenID Continuous Access Evaluation Profile 1.0"
  STATUS-LIST:
    target: https://datatracker.ietf.org/doc/draft-ietf-oauth-status-list
    title: "Token Status List"

--- abstract

A software statement carries a reviewer's decision about client software, bounded by an expiry the issuer chose. Ending that decision early has no standard mechanism, so how quickly a withdrawal takes effect is determined by how short the issuer made the statement's lifetime. This specification profiles the Shared Signals Framework for software statement lifecycle events, so that a statement issuer can tell the authorization servers that rely on its statements that a statement is revoked, that an approval is withdrawn, that an establishment is withdrawn, or that reviewed metadata has changed. Events can only reduce a client's standing, never create or extend it, and a receiver that misses an event falls back to expiry, so the mechanism shortens the interval between a decision and its effect without becoming load-bearing for security.

--- middle

# Introduction

{{ISSUANCE}} defines a software statement in which an issuer attests reviewed client metadata, and {{PRESENTATION}} defines how a trusting authorization server consumes one: at registration, where the statement's expiry can bound the registration's validity; at runtime, where it establishes a client for a request; and as a tenant approval evaluated in one tenant's context. In every case the decision ends when the statement expires unrenewed.

That leaves responsiveness coupled to lifetime. An issuer that wants a withdrawal to take effect within minutes must issue statements that live for minutes, and pay the renewal traffic for every client at every audience. An issuer that wants a manageable renewal cadence accepts that a withdrawal takes as long as the remaining lifetime.

The coupling is unnecessary, because the parties are already in a configured relationship. A trusting authorization server records each issuer's identifier, key source, scope, and role in order to accept its statements at all ({{ISSUANCE}}). This specification uses that relationship to carry lifecycle events over the Shared Signals Framework {{SSF}}: the statement issuer transmits, the trusting authorization server receives, and events are Security Event Tokens {{RFC8417}} delivered by the framework's push {{RFC8935}} or poll {{RFC8936}} bindings.

This specification defines the subject identification, event types, payload claims, and receiver processing rules for those events, and an optional reverse-direction event by which a consuming authorization server reports where a statement is in force. It defines no new endpoint, transport, subject identifier format, or trust establishment mechanism.

Two properties bound what the mechanism can do, and are normative in {{processing}}: an event can only reduce standing, and a receiver that misses events enforces expiry exactly as it does today. Deployments therefore keep bounded statement lifetimes, and neither {{ISSUANCE}} nor {{PRESENTATION}} depends on this specification.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

Transmitter, Receiver, Stream, and the delivery and configuration mechanisms are defined by {{SSF}}. Security Event Token, or SET, is defined by {{RFC8417}}. Subject identifier formats are defined by {{RFC9493}}. Software statement, issuer roles, and statement validation are defined by {{ISSUANCE}}. Establishment, tenant approval, registration validity, and runtime presentation are defined by {{PRESENTATION}}.

This specification additionally defines the following terms:

Statement Issuer:
: The party that signed a software statement, acting as a Transmitter of the events defined here.

Consuming Authorization Server:
: A trusting authorization server that has configured the statement issuer, acting as a Receiver of the events defined here.

# Relationship to the Statement Family {#relationship}

The events defined here carry no authority of their own. They report that a decision the issuer already had the authority to make has ended earlier than its expiry announced.

The scope of an event's effect follows the role in which the consuming authorization server accepts the transmitting issuer ({{ISSUANCE}}), exactly as the scope of a statement's effect does. An event from an issuer accepted in the establishment role bears on the client's existence at that server, for every tenant. An event from an issuer accepted in the tenant approval role bears only on the tenant that issuer speaks for. An event carries no tenant identifier; the receiver resolves scope from its own configuration.

A consuming authorization server MUST NOT accept an event from an issuer it has not configured, and MUST verify the SET using keys obtained as it obtains that issuer's statement verification keys, from the issuer's authorization server metadata {{RFC8414}}. It MUST NOT derive event trust from any key-location value carried in the event.

# Subject Identification {#subjects}

The subject of every event defined here is client software, identified as the `sub` of the statements the event concerns. Events use the `sub_id` claim of {{SSF}} with the formats of {{RFC9493}}:

* Software identified by a Client ID Metadata Document URL {{CIMD}} uses the `uri` format, whose `uri` member carries that URL exactly as it appears in the statement's `sub`.
* Software identified by an {{RFC7591}} `software_id` uses the `iss_sub` format, whose `iss` member carries the statement issuer's identifier and whose `sub` member carries the `software_id`.

A consuming authorization server MUST match the subject by exact comparison against the `sub` of the statements it holds, and MUST NOT treat a subject it does not recognize as an error.

# Event Types {#events}

Each event is a member of the SET `events` claim, whose value is the event payload object. All payloads share these claims:

`event_timestamp`:
: REQUIRED. A NumericDate value giving the time the issuer made the decision the event reports. Receivers use it to order events ({{processing}}).

`software_statement_jti`:
: OPTIONAL. The `jti` of the statement the event concerns. When present, the event concerns that statement alone. When absent, the event concerns every statement the transmitting issuer has issued for the subject.

`reason_admin`:
: OPTIONAL. A human-readable explanation intended for an administrator, as {{CAEP}} defines the member.

## Statement Revoked {#statement-revoked}

The issuer withdraws a specific statement before its expiry, for example because it was mis-issued or its holder's key was compromised. `software_statement_jti` is REQUIRED for this event type.

A receiver MUST stop honoring the named statement. Where that statement governs a registration's validity ({{PRESENTATION}}), the registration is expired as though its recorded `exp` had passed, and a replacement statement restores it under the revalidation rules of that specification.

## Approval Withdrawn {#approval-withdrawn}

A tenant approval issuer withdraws its tenant's approval of the subject software.

A receiver MUST stop treating the subject as approved for that issuer's tenant, with the effect the tenant approval rules of {{PRESENTATION}} give an absent approval. The client's establishment, and every other tenant, are unaffected.

## Establishment Withdrawn {#establishment-withdrawn}

An establishment issuer withdraws the subject software's listing, for example on removal from a marketplace.

A receiver MUST stop treating the subject as established. Registrations governed by statements from that issuer for that subject are expired, and runtime presentation of those statements fails as {{PRESENTATION}} defines for a statement that is not current. The effect reaches every tenant at that authorization server.

## Metadata Changed {#metadata-changed}

The issuer reports that the software's published metadata no longer matches what it reviewed. This event carries information a receiver could otherwise learn only by retrieval, and is advisory: the issuer is not withdrawing its decision.

`cimd_digest`:
: OPTIONAL. The digest the issuer now observes at the subject's Client ID Metadata Document, in the encoding {{ISSUANCE}} defines.

A receiver applies its post-issuance change policy ({{ISSUANCE}}) as though it had retrieved the document and observed the mismatch. It MUST NOT treat the event as attesting the new metadata.

## Consumption Reported {#consumption-reported}

This event travels in the reverse direction: the Transmitter is the consuming authorization server and the Receiver is the statement issuer. Support is OPTIONAL for both parties, and a consuming authorization server MUST NOT transmit it unless configured to do so for that issuer.

The event reports that the issuer's statement for the subject was consumed, so that an issuer can inventory where its decisions are in force. Its payload carries `event_timestamp` and `software_statement_jti`, and:

`consumption`:
: REQUIRED. One of `registration`, `presentation`, `delivery`, or `refused`, naming the consumption point of {{PRESENTATION}} at which the statement was used, or that it was refused.

The event conveys no authority in either direction, and a statement issuer MUST NOT treat its absence as evidence that a statement is unused.

# Receiver Processing {#processing}

A consuming authorization server that receives an event defined here MUST:

1. verify the SET as {{RFC8417}} requires, and verify that its issuer is configured and its keys were obtained as {{relationship}} requires;
2. reject an event whose `aud` does not include an identifier of this authorization server, and an event whose `jti` it has already processed;
3. resolve the subject ({{subjects}}) and the scope of the effect from the transmitting issuer's configured role ({{relationship}}); and
4. apply the effect the event type defines, at the scope resolved in step 3.

Two constraints bound every event:

* An event MUST NOT create standing, extend a statement's lifetime, restore a statement previously revoked, or otherwise increase what a client may do. A receiver MUST ignore any payload member that would have such an effect. Restoring standing requires a statement, presented or delivered as {{PRESENTATION}} defines.
* A receiver MUST continue to enforce statement expiry independently of this mechanism. Stream loss, transmitter unavailability, or delivery failure leaves standing to end at expiry, which remains the floor.

Events may arrive out of order or be duplicated. A receiver MUST NOT reverse an effect on the basis of an event whose `event_timestamp` precedes the event that produced it, and SHOULD retain the timestamp of the last applied event per subject and issuer for that comparison.

An effect applied by an event does not revoke access tokens already issued. A receiver applies its own grant and token lifetime policy, as it does when a statement expires.

# Stream Configuration {#configuration}

A statement issuer supporting this specification publishes Transmitter configuration metadata as {{SSF}} defines, discoverable from the issuer identifier the consuming authorization server has already configured. Stream creation, subject management, verification, and delivery follow {{SSF}}; this specification adds no configuration mechanism.

A consuming authorization server SHOULD create one stream per configured issuer and SHOULD request every event type this specification defines that the issuer's role can produce. A transmitter MUST NOT require a receiver to accept the reverse-direction event of {{consumption-reported}} as a condition of receiving the others.

# Security Considerations

## What an Event Cannot Do

The constraints of {{processing}} are the security argument for this mechanism. A forged, replayed, or reordered event cannot grant a client anything, because no event increases standing. Its worst outcome is the premature loss of a legitimate client's standing, which is an availability failure the issuer corrects by issuing a fresh statement, and which the receiver can bound by rate-limiting a transmitter's events.

## Expiry Remains the Floor

Because a receiver enforces expiry independently, an attacker who suppresses events, by disrupting delivery or the transmitter, delays a withdrawal at most until the affected statements expire. Deployments therefore choose lifetimes they would accept without this mechanism, and treat event delivery as an accelerator. A receiver SHOULD alert on stream loss rather than assume quiescence, since a healthy stream and a suppressed one are indistinguishable from the absence of events.

## Key Separation and Compromise

A transmitter MAY sign SETs with a key distinct from its statement signing key, published in the same metadata. Compromise of a statement signing key is the more serious event, because statements grant standing and events cannot; a receiver responding to such a compromise removes trust in the issuer or its scope as {{ISSUANCE}} describes, which also ends event acceptance.

## Comparison with Acceptance-Time Status

A status mechanism such as {{STATUS-LIST}} lets a receiver pull a statement's state at the moment it is consumed, at the cost of a retrieval on the consumption path and an availability dependence on the status endpoint. The events defined here push state changes ahead of consumption, at the cost of stream state and the suppression exposure above. The two are complementary, and a deployment can use both.

# Privacy Considerations

Subject identifiers in these events name software an issuer has reviewed and, in aggregate, describe an organization's approved software estate. A transmitter SHOULD scope each stream to the subjects the receiving authorization server can act on, and a receiver SHOULD apply to event logs the handling it applies to statements ({{ISSUANCE}}).

The reverse-direction event of {{consumption-reported}} tells an issuer where its statements are consumed, which for an enterprise issuer is its own estate but for a marketplace issuer reveals customer deployment activity at the provider. A consuming authorization server MUST treat transmission of that event as disclosure subject to its own tenant policy, and MAY omit it for any issuer or tenant.

# IANA Considerations

## Event Type Assignment

The event types defined in {{events}} require URI identifiers under a namespace this specification does not yet control. This version uses the placeholder base `https://schemas.openid.net/secevent/sw-stmt/event-type/` with the terminal segments `statement-revoked`, `approval-withdrawn`, `establishment-withdrawn`, `metadata-changed`, and `consumption-reported`. Assignment of the final base URI, and registration where the publishing body maintains a registry of security event types, is to be resolved before publication.

## SET Payload Claims

The payload claims `software_statement_jti`, `consumption`, and `cimd_digest` are defined by this specification for use in the event payloads of {{events}}. Registration in a registry of SET payload claims, where the publishing body maintains one, is to be resolved before publication. The `event_timestamp` and `reason_admin` members are used as {{CAEP}} defines them.

--- back

# Acknowledgments
{:numbered="false"}

This profile exists because bounding a review's lifetime and withdrawing it promptly are different requirements that a single expiry cannot serve. It draws on the Shared Signals Framework and the Continuous Access Evaluation Profile for its transport and payload conventions.
