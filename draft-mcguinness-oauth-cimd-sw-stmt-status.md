---
title: "CIMD Software Statement Status"
abbrev: oauth-cimd-sw-stmt-status
docname: draft-mcguinness-oauth-cimd-sw-stmt-status-latest
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
 - Withdrawal

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
  CAEP:
    target: https://openid.net/specs/openid-caep-1_0.html
    title: "OpenID Continuous Access Evaluation Profile 1.0"

--- abstract

This specification defines how the status of a CIMD software statement is published and checked. Token Status List supplies the authoritative status: an issuer publishes it, and a trusting authorization server resolves it on the list's own schedule and refuses a withdrawn statement. Optional Shared Signals events tell the servers relying on an issuer's statements that a status has changed, prompting them to check at once. An event only says when to look, so a server that misses every event reaches the same result on its ordinary schedule.

--- middle

# Introduction

{{STATEMENT}} lets an issuer withdraw a decision before the statement's expiry, and has a trusting authorization server that learns of a withdrawal hold a refusal record, but it requires and defines no withdrawal mechanism. This specification defines two. {{token-status-list}} profiles Token Status List {{STATUSLIST}}: an issuer withdraws a decision by setting the statement's status, and a trusting authorization server learns of the change when it next resolves the list.

A withdrawal therefore takes effect only as quickly as consumers poll. As an optional addition, this specification defines a Shared Signals Framework {{SSF}} event by which an issuer tells the trusting authorization servers that have configured it ({{STATEMENT}}) that a status has changed, prompting them to resolve it at once. An event carries no status, and the status list remains the authority on whether a statement stands ({{processing}}).

An issuer and a trusting authorization server can use the status list without the events, and the requirements of {{relationship}} through {{configuration}} apply only to implementations that support the events. This specification defines no new endpoint, transport, subject identifier format, durable receiver record, or trust establishment mechanism. {{STATEMENT}} does not depend on this specification.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

Transmitter, Receiver, Stream, and the delivery and configuration mechanisms are defined by {{SSF}}. Security Event Token, or SET, is defined by {{RFC8417}}. Subject identifier formats are defined by {{RFC9493}}. Status List Token, and the validation that resolves a status, are defined by {{STATUSLIST}}. The software statement, its claims, its validation, Issuing Authorization Server, Trusting Authorization Server, issuer trust configuration, runtime presentation, withdrawal, and Refusal Record are defined by {{STATEMENT}}.

For the events defined here, the issuing authorization server acts as a Transmitter, and a trusting authorization server that has configured it acts as a Receiver.

# Token Status List {#token-status-list}

An issuer may use Token Status List {{STATUSLIST}} as the withdrawal mechanism of {{STATEMENT}} for its statements. A trusting authorization server may resolve those statuses for any issuer it has configured.

## Status Publication {#status-publication}

An issuer that uses Token Status List publishes the status of its statements as {{STATUSLIST}} defines, and each statement locates itself in the issuer's Status List Token with its `status` claim ({{STATEMENT}}). An issuer that publishes status:

* MUST publish it, and carry the `status` claim, for every statement it issues under a given `iss` from the point it begins publishing, so that, once statements issued before that point have expired, a trusting authorization server can read an absent claim as meaning the issuer publishes no status;
* MUST NOT reuse an index across statements, so that withdrawing a statement affects no other: withdrawing a superseded statement does not withdraw its replacement, and withdrawing a replacement does not restore its predecessor;
* MUST sign the Status List Token with a key published at the `jwks_uri` of its authorization server metadata {{RFC8414}}, which {{STATEMENT}} keeps apart from its statement signing keys;
* MUST include `exp` in every Status List Token it publishes, so that an older token cannot stand in for a newer one indefinitely; and
* MUST give each Status List Token it publishes at a given list URI an `iat` later than that of any token it published there before, so that a trusting authorization server can tell the newer of two apart.

## Status Resolution {#status-resolution}

Where the statement carries `status` and the trusting authorization server resolves statuses for that issuer, it MUST reject a statement whose status is `INVALID`, and MUST apply its configured policy for that issuer to a statement whose status is `SUSPENDED`. A status of `INVALID`, or of `SUSPENDED` where that policy refuses the statement, is a withdrawal: the server holds a refusal record ({{STATEMENT}}). Token Status List defines no expiry for a withdrawal, so the refusal ends only when a later resolution returns a status that no longer refuses the statement.

Resolution follows {{STATUSLIST}}, including its requirement to reject where the referenced index lies outside the list, a rejection that also uses the refusal-record row of the error responses of {{STATEMENT}}, and its caching rules, which also apply when the server checks a recorded statement at refresh ({{STATEMENT}}); a statement whose status the server cannot resolve is bounded by `exp`, as {{STATEMENT}} provides. A server MUST NOT replace a Status List Token it holds with one whose `iat` is earlier, and MUST treat one whose `iat` is equal but whose contents differ as a resolution failure, since a host serving an older token could otherwise restore a status the issuer has withdrawn.

A server MUST obtain a Status List Token's verification keys from the `jwks_uri` of the issuer's authorization server metadata {{RFC8414}}, reached from the configured `iss`, never from the list itself.

# Event Issuer Trust and Keys {#relationship}

A trusting authorization server MUST NOT accept an event from an issuer it has not configured. It MUST verify the SET using keys from the `jwks_uri` of the issuer's Transmitter configuration {{SSF}}, discovered from the issuer identifier it has configured, and never from a key-location value carried in the event. The SET's `iss` is that issuer identifier, which is also the `iss` of the statements the event concerns.

The Transmitter configuration's `jwks_uri` MUST differ both from the issuer's `software_statement_jwks_uri` ({{STATEMENT}}) and from the `jwks_uri` in its authorization server metadata {{RFC8414}}, whose keys verify its Status List Tokens. A receiver MUST NOT create or keep a stream whose Transmitter configuration names either of those locations.

A SET carrying an event defined here MUST use the explicit `typ` JOSE header parameter value `secevent+jwt` ({{RFC8417}}), which distinguishes it from a software statement and from a Status List Token. Its `aud` claim MUST contain the receiving authorization server's issuer identifier {{RFC8414}}; a stream audience negotiated under {{SSF}} does not replace it.

# Subject Identification {#subjects}

The subject of every event defined here is client software. Events carry it in the `sub_id` claim of {{SSF}} using the `uri` format of {{RFC9493}}, whose `uri` member carries the Client ID Metadata Document URL {{CIMD}} exactly as it appears in the `sub` of the statements the event concerns.

A trusting authorization server MUST match the subject by exact comparison of the `uri` member against a statement's `sub`.

# Event Types {#events}

The payload ({{RFC8417}}) of an event defined here has the following claims:

`event_timestamp`:
: REQUIRED. A NumericDate value giving the time the issuer changed the status the event reports. This narrows the {{CAEP}} member, which is optional and gives the time the event occurred. It is informational: a receiver does not use it to order, bound, or scope anything, since the resolved status governs ({{processing}}).

`software_statement_jti`:
: OPTIONAL. The `jti` of a single statement whose status changed ({{processing}}).

`reason_admin`:
: OPTIONAL. An administrative message, as {{CAEP}} defines the member.

## Status Changed {#status-changed}

The event type identifier is `urn:ietf:params:oauth:event-type:software-statement-status-changed` ({{iana-event-type}}).

The event reports that the issuer has changed the published status of one or more statements for the subject. A transmitter MUST NOT transmit the event until a Status List Token reflecting the change is retrievable at the status list URI, including through any caching layer it operates, since a receiver resolving earlier would fetch the state the event exists to correct.

The event does not say what the new status is, and a receiver MUST NOT infer one from it; a receiver that resolves a status of `VALID` after an event has applied the event correctly.

# Receiver Processing {#processing}

A trusting authorization server that receives an event defined here MUST:

1. verify the SET as {{RFC8417}} requires, including its `typ`, and verify that its issuer is configured and its keys were obtained as {{relationship}} requires;
2. reject an event whose `aud` does not contain its issuer identifier ({{relationship}}), and an event whose type it does not recognize;
3. resolve the subject ({{subjects}}); and
4. resolve the status of the affected statements from the issuer's Status List Token as {{STATUSLIST}} defines, without waiting for the schedule it would otherwise have used, and apply the resolved status as {{status-resolution}} defines.

The affected statements are those the transmitting issuer made about the subject, never statements another issuer made about the same software: the one the event's `software_statement_jti` names or, where the event names none, those the receiver holds or has cached a status for. An event about a subject for which the receiver holds no state requires nothing of it.

A receiver MUST fetch the Status List Token for the affected statements afresh rather than answer from a cached copy, since a cached copy is what the event exists to correct. An event is never grounds for refusal: until the fetch succeeds, the copy the receiver holds remains in effect within its validity, and where resolution does not complete, the receiver applies the rules of {{STATEMENT}} as it would had no event arrived.

A receiver MUST treat an event it has already applied as successfully delivered and acknowledge it as {{RFC8935}} or {{RFC8936}} requires, rather than reporting a delivery error, since rejecting a retry can stall or disable a stream carrying later events. Duplicate detection is per transmitting issuer, since SET `jti` values are unique only within an issuer.

Two constraints bound every event:

* An event MUST NOT by itself create standing, extend a statement's lifetime, or otherwise increase what a client may do. A receiver MUST ignore any payload member that would have such an effect.
* A receiver MUST continue to resolve status on its own schedule, and to enforce statement expiry, independently of this mechanism, so that stream loss, transmitter unavailability, or delivery failure leaves both controls in force.

# Stream Configuration {#configuration}

An issuing authorization server that supports the events defined here publishes Transmitter configuration metadata as {{SSF}} defines, discoverable from its issuer identifier. Stream creation, subject management, verification, and delivery follow {{SSF}}.

A trusting authorization server that supports these events SHOULD create one stream per configured issuer that offers them, covering every subject that issuer attests rather than an enumerated set, since under runtime presentation ({{STATEMENT}}) it holds no state for software until its first presentation, by which time an event about that software would already have been missed.

A transmitter supporting these events MUST therefore advertise `default_subjects` as `ALL` in its Transmitter configuration {{SSF}}. The subjects appropriate to a stream are those of the issuer's statements whose `aud` is absent or names the receiving authorization server, and a transmitter SHOULD scope each stream to them. A receiver discards events for subjects outside the identifier scope for which it accepts that issuer ({{STATEMENT}}).

A receiver SHOULD request the event this specification defines, and SHOULD use the stream verification facility of {{SSF}} on a schedule, since a misconfigured stream that delivers nothing is otherwise indistinguishable from an issuer with nothing to report.

# Security Considerations

## Status Resolution Schedule

Because {{STATUSLIST}} aggregates many statements into one list, a retrieval tells the issuer only that some trusting authorization server is checking. A server SHOULD retrieve the list on the list's own schedule rather than once per request, so that its request timing does not disclose the client population it serves and each request the statement governs does not depend on issuer availability.

## What an Event Cannot Do

An event names no status, so a forged, replayed, delayed, or reordered event cannot change whether any statement stands. Its only effect is a resolution the receiver would have performed anyway, against a Status List Token signed by the issuer.

The remaining exposure is resource cost: an attacker holding the issuer's SET signing key, or a misbehaving transmitter, can drive resolutions. A receiver SHOULD bound the rate at which it resolves in response to events, coalescing events for the same issuer.

## Status Remains the Authority

Because a receiver resolves status independently ({{processing}}), an attacker who suppresses events delays a withdrawal at most until the receiver's next scheduled resolution and, for new presentations, never beyond the affected statements' expiry; grants already open continue as the refresh policy of {{STATEMENT}} allows. Deployments therefore choose a resolution schedule and statement lifetimes they would accept with no event stream.

A receiver SHOULD alert on stream loss rather than assume quiescence, since the absence of events looks the same on a healthy stream and a suppressed one. Periodic stream verification ({{configuration}}) makes the difference observable.

## Key Separation and Compromise

The distinct key set locations of {{relationship}} separate SET keys from statement and Status List Token keys only while the transmitter publishes no SET key in either of the other key sets. With the keys separated, compromise of a SET key lets an attacker drive resolutions and nothing more, while compromise of a statement signing key has wider effect, because statements grant standing and events cannot. Removing trust in the issuer as {{STATEMENT}} describes also ends event acceptance.

# Privacy Considerations

Subject identifiers in these events name software an issuer has reviewed and, in aggregate, describe an organization's approved software estate, which is why a stream is scoped as {{configuration}} provides. A receiver SHOULD apply to event logs the handling it applies to statements ({{STATEMENT}}).

Because this specification defines no durable receiver-side record, it adds no negative state about a client identifier that outlives the issuer's own published status. A receiver that logs events does create such a record, and SHOULD bound its retention accordingly.

# IANA Considerations

## OAuth URI Registry {#iana-event-type}

{{RFC8417}} establishes no registry of event types, so this specification takes its identifier from the `oauth` sub-namespace {{RFC6755}} and requests registration of the following value in the "OAuth URI" registry:

URN:
: `urn:ietf:params:oauth:event-type:software-statement-status-changed`

Common Name:
: CIMD Software Statement Status Changed Event Type

Change Controller:
: IESG

Specification Document(s):
: This specification, {{status-changed}}

## SET Payload Claims

Under Section 2 of {{RFC8417}}, the `software_statement_jti` payload claim of {{events}} is defined by this specification and need not be registered as a JWT claim; no IANA registry of them exists, so no registration is requested.

--- back

# Acknowledgments
{:numbered="false"}

This profile draws on the Shared Signals Framework and the Continuous Access Evaluation Profile for its transport and payload conventions.
