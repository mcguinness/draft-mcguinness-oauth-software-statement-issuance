---
title: "Shared Signals Events for CIMD Software Statements"
abbrev: oauth-cimd-sw-stmt-signals
docname: draft-mcguinness-oauth-cimd-sw-stmt-signals-latest
category: std

ipr: trust200902
area: "Security"
workgroup: "Web Authorization Protocol"

keyword:
 - OAuth
 - Software Statement
 - Shared Signals
 - Security Event Token
 - Status List

stand_alone: yes
pi: [toc, sortrefs, symrefs]

author:
 -
    ins: K. McGuinness
    name: Karl McGuinness
    organization: Independent
    email: public@karlmcguinness.com

normative:
  RFC6755:
  RFC9967:
  RFC8935:
  RFC8936:
  RFC8414:
  RFC8417:
  RFC9493:
  CIMD:
    target: https://datatracker.ietf.org/doc/draft-ietf-oauth-client-id-metadata-document
    title: "OAuth Client ID Metadata Document"
  STATEMENT:
    target: https://datatracker.ietf.org/doc/draft-mcguinness-oauth-cimd-sw-stmt
    title: "CIMD Software Statement"
  STATUSLIST:
    target: https://datatracker.ietf.org/doc/draft-ietf-oauth-status-list
    title: "Token Status List"
  SSF:
    target: https://openid.net/specs/openid-sharedsignals-framework-1_0.html
    title: "OpenID Shared Signals Framework Specification 1.0"

informative:
  REGISTRATION:
    target: https://datatracker.ietf.org/doc/draft-mcguinness-oauth-cimd-sw-stmt-registration
    title: "CIMD Software Statement Registration"
  RFC5646:
  CAEP:
    target: https://openid.net/specs/openid-caep-1_0.html
    title: "OpenID Continuous Access Evaluation Profile 1.0"
  ISSUANCE:
    target: https://datatracker.ietf.org/doc/draft-mcguinness-oauth-cimd-sw-stmt-issuance
    title: "CIMD Software Statement Issuance"

--- abstract

A software statement records a reviewer's decision about client software. An issuer withdraws that decision before the statement expires by publishing a status through Token Status List, which a trusting authorization server resolves on the list's own schedule, so that schedule determines how quickly a withdrawal takes effect. This specification profiles the Shared Signals Framework so that an issuer can notify the servers relying on its statements that a status has changed, prompting them to resolve it at once. The status list remains the authority; an event only says when to look. A receiver that misses every event reaches the same result on its ordinary schedule, so the mechanism reduces latency without becoming necessary for correctness.

--- middle

# Introduction

{{STATEMENT}} defines a software statement, in which an issuer vouches for a reviewed Client ID Metadata Document, and the `status` claim by which a statement locates itself in the issuer's Status List Token {{STATUSLIST}}. An issuer withdraws a decision before its expiry by setting that status, and a trusting authorization server learns of the change when it next resolves the list.

Responsiveness therefore depends on the fetch interval. For a withdrawal to take effect within minutes, every consumer has to poll at that interval, and most of those requests report no change.

The parties already have a configured relationship: a trusting authorization server records each issuer's identifier, key source, scope, and lifetime policy in order to accept its statements at all ({{STATEMENT}}). This specification uses that relationship to carry a notification over the Shared Signals Framework {{SSF}}. The statement issuer transmits, the trusting authorization server receives, and events are Security Event Tokens {{RFC8417}} delivered by the push {{RFC8935}} or poll {{RFC8936}} delivery methods.

An event carries no decision: it reports that the issuer changed a status, and the receiver resolves that status as it would have later anyway. The status list remains the authority on whether a statement stands. {{processing}} makes two properties normative: an event can only prompt a resolution and never itself increases what a client may do, and a receiver that misses events enforces status and expiry exactly as it would without them.

This specification defines the subject identification, the event, its payload claims, and the receiver's processing rules. It defines no new endpoint, transport, subject identifier format, durable receiver record, or trust establishment mechanism. {{STATEMENT}} does not depend on it.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

Transmitter, Receiver, Stream, and the delivery and configuration mechanisms are defined by {{SSF}}. Security Event Token, or SET, is defined by {{RFC8417}}. Subject identifier formats are defined by {{RFC9493}}. Status List Token, and the validation that resolves a status, are defined by {{STATUSLIST}}. The software statement, its claims including `status`, its validation, issuer trust configuration, and runtime presentation are defined by {{STATEMENT}}, and registration validity by {{REGISTRATION}}.

This specification additionally defines the following terms:

Statement Issuer:
: The Issuing Authorization Server ({{STATEMENT}}) that signed a software statement and publishes its status ({{ISSUANCE}}), acting as a Transmitter of the events defined here.

Consuming Authorization Server:
: A Trusting Authorization Server ({{STATEMENT}}) that has configured the statement issuer, acting as a Receiver of the events defined here.

# Relationship to the Statement Family {#relationship}

An event bears only on the statements the transmitting issuer has made about the named subject, never on statements another issuer made about the same software.

A consuming authorization server MUST NOT accept an event from an issuer it has not configured. It MUST verify the SET using keys from the `jwks_uri` of the issuer's Transmitter configuration {{SSF}}, discovered from the issuer identifier it has configured. The SET's `iss` is that issuer identifier, which is also the `iss` of the statements the event concerns.

The Transmitter configuration's `jwks_uri` MUST differ both from the issuer's `software_statement_jwks_uri` ({{STATEMENT}}) and from the `jwks_uri` in its authorization server metadata {{RFC8414}}, whose keys verify its Status List Tokens. A receiver:

* MUST NOT create or keep a stream whose Transmitter configuration names either of those locations;
* MUST NOT use SET keys to verify statements; and
* MUST NOT derive event trust from any key-location value carried in the event.

A SET carrying an event defined here MUST use the explicit `typ` JOSE header parameter value `secevent+jwt` ({{RFC8417}}), so that a receiver validating JWTs from a configured issuer can distinguish an event from a software statement, which {{STATEMENT}} types differently, and from a Status List Token. Its `aud` claim MUST contain the receiving authorization server's issuer identifier as defined by {{RFC8414}}, the value a statement's `aud` carries; a stream audience negotiated under {{SSF}} does not replace it.

# Subject Identification {#subjects}

The subject of every event defined here is client software, identified by the `sub` of the statements the event concerns. Events carry it in the `sub_id` claim of {{SSF}} using the `uri` format of {{RFC9493}}, whose `uri` member carries the Client ID Metadata Document URL {{CIMD}} exactly as it appears in the statement's `sub`.

A consuming authorization server MUST match the subject by exact comparison of the `uri` member against a statement's `sub`. A receiver MAY receive an event for a subject it holds no state for, as it ordinarily does for software it has never seen presented, and such an event requires nothing of it.

# Event Types {#events}

Each event is a member of the SET `events` claim, whose value is the event payload object. All payloads share these claims:

`event_timestamp`:
: REQUIRED. A NumericDate value giving the time the issuer changed the status the event reports. {{CAEP}} defines this member as optional and as the time the event occurred; this specification requires it and narrows it to the status change. It is informational, for logging and audit: a receiver does not use it to order, bound, or scope anything, since the resolved status governs ({{processing}}).

`software_statement_jti`:
: OPTIONAL. The `jti` of a single statement whose status changed. Where absent, the event does not name the statements whose status changed ({{processing}}).

`reason_admin`:
: OPTIONAL. A JSON object whose members are language tags {{RFC5646}} and whose values are human-readable explanations intended for an administrator, as {{CAEP}} defines the member.

## Status Changed {#status-changed}

The event type identifier is `urn:ietf:params:oauth:event-type:software-statement-status-changed` ({{iana-event-type}}), used as a member name of the SET `events` claim.

The event reports that the issuer has changed the published status of one or more statements for the subject, for example on delisting software, on discovering that a statement was mis-issued, or on compromise of the client's key. A transmitter MUST NOT transmit the event until a Status List Token reflecting the change is retrievable at the status list URI, including through any caching layer it operates, since a receiver resolving earlier would fetch the state the event exists to correct.

The event does not say what the new status is, and a receiver MUST NOT infer one from it. The new status is what {{STATUSLIST}} resolution returns; a receiver that resolves a status of `VALID` after an event has applied the event correctly.

Carrying the new status in the event would create a second source for it, which the receiver would have to reconcile with the list, and a forged or replayed event could then assert a status the issuer never published.

# Receiver Processing {#processing}

A consuming authorization server that receives an event defined here MUST:

1. verify the SET as {{RFC8417}} requires, including its `typ`, and verify that its issuer is configured and its keys were obtained as {{relationship}} requires;
2. reject an event whose `aud` does not contain its issuer identifier ({{relationship}}), and an event whose type it does not recognize;
3. resolve the subject ({{subjects}}); and
4. resolve the status of the affected statements from the issuer's Status List Token as {{STATUSLIST}} defines, without waiting for the schedule it would otherwise have used, and apply the resolved status under the rules of {{STATEMENT}}.

The affected statements are the one the event's `software_statement_jti` names or, where the event names none, the statements for that subject and issuer that the receiver holds or has cached a status for.

A receiver MUST fetch the Status List Token for the affected statements afresh rather than answer from a cached copy, since a cached copy is what the event exists to correct. Until the fetch succeeds, the copy it holds remains in effect within its validity.

An event is never grounds for refusal. Pending resolution, a receiver applies the status it last resolved, and where resolution does not complete, it applies the rules of {{STATEMENT}} as it would had no event arrived.

A receiver MUST treat an event it has already applied as successfully delivered and acknowledge it as {{RFC8935}} or {{RFC8936}} requires, rather than reporting a delivery error; duplicate delivery is ordinary retry behavior and rejecting it can stall or disable a stream carrying later events. Duplicate detection is per transmitting issuer, since SET `jti` values are unique only within an issuer.

Events can also arrive out of order. Because a receiver applies each accepted event by resolving status, a later resolution returns the later state, and a replayed or delayed event costs a resolution and changes nothing else.

Two constraints bound every event:

* An event MUST NOT by itself create standing, extend a statement's lifetime, or otherwise increase what a client may do. A receiver MUST ignore any payload member that would have such an effect. What a client may do follows from a valid statement and its resolved status, as {{STATEMENT}} defines.
* A receiver MUST continue to resolve status on its own schedule, and to enforce statement expiry, independently of this mechanism. Stream loss, transmitter unavailability, or delivery failure leaves both controls in force.

Applying an event does not revoke access tokens already issued. A receiver applies its own grant and token lifetime policy, as it does when a statement expires or its status changes.

# Stream Configuration {#configuration}

A statement issuer supporting this specification publishes Transmitter configuration metadata as {{SSF}} defines, discoverable from the issuer identifier the consuming authorization server has already configured. Stream creation, subject management, verification, and delivery follow {{SSF}}; this specification adds no configuration mechanism.

A consuming authorization server SHOULD create one stream per configured issuer, covering every subject that issuer attests rather than an enumerated set. A receiver cannot enumerate subjects: under runtime presentation ({{STATEMENT}}) it holds no state for software until its first presentation, by which time an event about that software would already have been missed.

A transmitter supporting this specification MUST therefore advertise `default_subjects` as `ALL` in its transmitter configuration {{SSF}}, so that a stream carries every subject appropriate to it without the receiver adding any. The subjects appropriate to a stream are those of the issuer's statements whose `aud` is absent or names the receiving authorization server; a receiver discards events for subjects outside the identifier scope for which it accepts that issuer ({{STATEMENT}}).

A receiver SHOULD request the event this specification defines, and SHOULD use the stream verification facility of {{SSF}} on a schedule, since a stream delivering nothing because it was misconfigured is otherwise indistinguishable from an issuer with nothing to report.

# Security Considerations

## What an Event Cannot Do

An event names no status, so a forged, replayed, or reordered event cannot change whether any statement stands. Its only effect is a resolution the receiver would have performed anyway, against a Status List Token signed by the issuer.

The remaining exposure is resource cost: an attacker holding the issuer's SET signing key, or a misbehaving transmitter, can drive resolutions. A receiver SHOULD bound the rate at which it resolves in response to events, coalescing events for the same issuer, and MUST NOT let event-driven resolution displace its scheduled resolution.

## Status Remains the Authority

Because a receiver resolves status independently, an attacker who suppresses events, by disrupting delivery or the transmitter, delays a withdrawal at most until the receiver's next scheduled resolution and, for new presentations, never beyond the affected statements' expiry. Grants already open continue as the refresh policy of {{STATEMENT}} allows. Deployments therefore choose a resolution schedule and statement lifetimes they would accept with no event stream, and treat delivery as an accelerator.

A receiver SHOULD alert on stream loss rather than assume quiescence, since the absence of events looks the same on a healthy stream and a suppressed one. Periodic stream verification ({{configuration}}) makes the difference observable.

## Key Separation and Compromise

A receiver verifies statements only against the issuer's statement key set ({{STATEMENT}}), Status List Tokens only against the issuer's `jwks_uri`, and SETs only against the Transmitter configuration's `jwks_uri`, which {{relationship}} requires to differ from both. This separation holds only while the transmitter publishes no SET key in either of the other key sets.

With the keys separated, compromise of a SET key lets an attacker drive resolutions and nothing more. Compromise of a statement signing key is the serious event, because statements grant standing and events cannot. A receiver responding to such a compromise removes trust in the issuer or its scope as {{STATEMENT}} describes, which also ends event acceptance.

## Relationship to Scheduled Resolution

{{STATUSLIST}} resolution costs a fetch and depends on the status endpoint's availability. This specification changes neither and adds no second source of status: it only shortens the interval between an issuer's change and a receiver's next fetch, for deployments where that interval matters more than the cost of maintaining a stream. A deployment that finds its scheduled interval acceptable does not need it.

# Privacy Considerations

Subject identifiers in these events name software an issuer has reviewed and, in aggregate, describe an organization's approved software estate. A transmitter SHOULD scope each stream to the subjects of statements whose `aud` is absent or names the receiving authorization server ({{configuration}}), and a receiver SHOULD apply to event logs the handling it applies to statements ({{STATEMENT}}).

Because this specification defines no durable receiver-side record, it adds no negative state about a client identifier that outlives the issuer's own published status. A receiver that logs events does create such a record, and SHOULD bound its retention accordingly.

# IANA Considerations

## OAuth URI Registry {#iana-event-type}

{{RFC8417}} identifies an event type by a URI and establishes no registry of event types, and no `urn:ietf:params:secevent` sub-namespace exists. This specification therefore takes its identifier from the `oauth` sub-namespace {{RFC6755}}, following the class and identifier structure that sub-namespace suggests. {{RFC9967}} types security events the same way under `urn:ietf:params:scim:event`, with a registry of its own; a single event type does not warrant one, so this specification requests registration of the following value in the "OAuth URI" registry:

URN:
: `urn:ietf:params:oauth:event-type:software-statement-status-changed`

Common Name:
: CIMD Software Statement Status Changed Event Type

Change Controller:
: IESG

Specification Document(s):
: This specification, {{status-changed}}

## SET Payload Claims

This specification defines the payload claim `software_statement_jti` for the event payload of {{events}}. Under Section 2 of {{RFC8417}}, payload claims need not be registered as JWT claims and are defined by the specification profiling the event; no IANA registry of them exists, so no registration is requested, and the claim is scoped to the event type that carries it. The `reason_admin` member is used as {{CAEP}} defines it, and `event_timestamp` as {{events}} narrows it.

--- back

# Acknowledgments
{:numbered="false"}

This profile draws on the Shared Signals Framework and the Continuous Access Evaluation Profile for its transport and payload conventions.
