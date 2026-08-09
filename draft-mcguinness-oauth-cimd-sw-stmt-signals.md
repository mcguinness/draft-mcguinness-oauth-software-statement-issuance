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
 - Revocation

stand_alone: yes
pi: [toc, sortrefs, symrefs]

author:
 -
    ins: K. McGuinness
    name: Karl McGuinness
    organization: Independent
    email: public@karlmcguinness.com

normative:
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
  SSF:
    target: https://openid.net/specs/openid-sharedsignals-framework-1_0.html
    title: "OpenID Shared Signals Framework Specification 1.0"

informative:
  CAEP:
    target: https://openid.net/specs/openid-caep-1_0.html
    title: "OpenID Continuous Access Evaluation Profile 1.0"
  STATUS-LIST:
    target: https://datatracker.ietf.org/doc/draft-ietf-oauth-status-list
    title: "Token Status List"

--- abstract

A software statement carries a reviewer's decision about client software, bounded by an expiry the issuer chose. Ending that decision early has no standard mechanism, so how quickly a withdrawal takes effect is determined by how short the issuer made the statement's lifetime. This specification profiles the Shared Signals Framework for software statement lifecycle events, so that a statement issuer can tell the authorization servers that rely on its statements that a review, or a single statement carrying it, is withdrawn before it expires. Events can only reduce a client's standing, never create or extend it, and a receiver that misses an event falls back to the expiry and lifecycle controls it would have applied anyway, so the mechanism shortens the interval between a decision and its effect without becoming load-bearing for security.

--- middle

# Introduction

{{STATEMENT}} defines a software statement, in which an issuer vouches for a reviewed Client ID Metadata Document, and how a trusting authorization server consumes one: at registration, where the statement's expiry can bound the registration's validity, and at runtime, where it establishes a client for a request. In both cases the decision ends when the statement expires unrenewed.

That leaves responsiveness coupled to lifetime. An issuer that wants a withdrawal to take effect within minutes must issue statements that live for minutes, and pay the renewal traffic for every client at every audience. An issuer that wants a manageable renewal cadence accepts that a withdrawal takes as long as the remaining lifetime.

The coupling is unnecessary, because the parties are already in a configured relationship. A trusting authorization server records each issuer's identifier, key source, scope, and role in order to accept its statements at all ({{STATEMENT}}). This specification uses that relationship to carry lifecycle events over the Shared Signals Framework {{SSF}}: the statement issuer transmits, the trusting authorization server receives, and events are Security Event Tokens {{RFC8417}} delivered by the framework's push {{RFC8935}} or poll {{RFC8936}} bindings.

This specification defines the subject identification, the event, its payload claims, the durable records a receiver keeps, and its processing rules. It defines no new endpoint, transport, subject identifier format, or trust establishment mechanism.

Two properties bound what the mechanism can do, and are normative in {{processing}}: an event can only reduce standing, and a receiver that misses events enforces expiry exactly as it does today. A withdrawal is durable rather than momentary, which {{withdrawal-records}} specifies, so a client holding an unexpired statement cannot restore what an event ended. Deployments therefore keep bounded statement lifetimes, and {{STATEMENT}} does not depend on this specification.

This mechanism buys latency, not correctness. A statement is carried by the client it admits, and a client has no reason to stop presenting one, so withdrawing a review before its expiry is what these events are for. A decision a consuming server evaluates afresh on every request, such as one carried by an identity assertion, needs no event at all.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

Transmitter, Receiver, Stream, and the delivery and configuration mechanisms are defined by {{SSF}}. Security Event Token, or SET, is defined by {{RFC8417}}. Subject identifier formats are defined by {{RFC9493}}. The software statement, its claims, its validation, issuer trust configuration, registration validity, runtime presentation, and the per-grant establishment state are defined by {{STATEMENT}}.

This specification additionally defines the following terms:

Statement Issuer:
: The party that signed a software statement, acting as a Transmitter of the events defined here.

Consuming Authorization Server:
: An authorization server that has configured the statement issuer under {{STATEMENT}}, acting as a Receiver of the events defined here.

Standing:
: What a client may do at a consuming authorization server by virtue of a software statement: its registration and that registration's validity, and its establishment at request time ({{STATEMENT}}). Standing is created and extended only by statements, never by the events defined here.

Withdrawal Record:
: The state a receiver retains when it applies a withdrawal event, defined in {{withdrawal-records}}.

# Relationship to the Statement Family {#relationship}

The events defined here carry no authority of their own. They report that a decision the issuer already had the authority to make has ended earlier than its expiry announced.

An event bears on the statements the transmitting issuer has made about the named subject, and on nothing else. A withdrawal by one issuer never reaches statements another issuer made about the same software, whatever those two issuers each vouched for.

A consuming authorization server MUST NOT accept an event from an issuer it has not configured, and MUST verify the SET using keys obtained as it obtains that issuer's statement verification keys, from the issuer's authorization server metadata {{RFC8414}}. It MUST NOT derive event trust from any key-location value carried in the event.

A SET carrying an event defined here MUST use the explicit `typ` header value `secevent+jwt` ({{RFC8417}}), so that a receiver validating JWTs from a configured issuer distinguishes an event from a software statement, whose own typing {{STATEMENT}} fixes. Its `aud` MUST contain the receiving authorization server's issuer identifier as defined by {{RFC8414}}, which is the value that appears in a statement's `aud`; a stream audience negotiated under {{SSF}} does not replace it.

# Subject Identification {#subjects}

The subject of every event defined here is client software, identified as the `sub` of the statements the event concerns. Events use the `sub_id` claim of {{SSF}} with the formats of {{RFC9493}}:

Software is identified by its Client ID Metadata Document URL {{CIMD}} using the `uri` format, whose `uri` member carries that URL exactly as it appears in the statement's `sub`.

A consuming authorization server MUST match the subject by exact comparison of the `uri` member against a statement's `sub`, and against the subjects of its Withdrawal Records. The transmitting issuer is part of every record, so a withdrawal by one issuer never reaches statements another issuer made about the same software. A receiver MUST record a withdrawal for a subject it does not currently recognize ({{withdrawal-records}}) rather than discarding the event. A subject unknown at the time of an event is the ordinary case in the runtime profile, where a receiver holds no state for software until it is first presented.

# Withdrawal Records {#withdrawal-records}

A withdrawal event names a decision that ended. Its effect is durable, and a receiver that only ceases some current activity has not applied it: in the runtime profile there is no current activity to cease, and in the registration profile the client holds a statement that would otherwise restore what the event withdrew.

On applying a withdrawal, a consuming authorization server MUST create or update a Withdrawal Record holding the transmitting issuer, the subject, an effective time, and the withdrawn `jti` where the event named one.

The effective time is the event's `event_timestamp`, so every receiver applies the same cutoff whatever its delivery latency; deriving it from arrival time would make a legitimate reissued statement acceptable at a prompt receiver and refused at a delayed one.

Two rules keep that cutoff honest. An issuer MUST set `event_timestamp` at or after the `iat` of the most recent statement it has issued for the subject, so the cutoff covers everything already minted, and MUST NOT afterwards issue a statement for that subject with an `iat` at or before it; it SHOULD suspend automated renewal for the subject before transmitting. A receiver MUST reject an event whose `event_timestamp` lies in the future beyond its clock-skew allowance, since a future-dated cutoff would refuse every statement the issuer could later mint and leave the subject unrecoverable at some receivers while others discarded the record and reopened it.

While a Withdrawal Record is retained, the receiver MUST refuse a statement matching it:

* a record naming a `jti` refuses that statement, whatever its `iat`;
* a record naming none refuses every statement from that issuer for that subject whose `iat` is at or before the record's effective time.

A statement from that issuer for that subject with an `iat` after the record's effective time is a later decision by the same authority. It is accepted under the ordinary rules of {{STATEMENT}}, and restores what the withdrawal ended. This is the only recovery path, and it requires the issuer to act.

A refusal under this section takes effect wherever the statement would otherwise be consumed: a registration governed by a refused statement is expired as {{STATEMENT}} defines for a lapsed registration, and a runtime presentation carrying one fails as it does for a statement that is not current. A grant already open under a refused statement continues only as far as its next currency check: a receiver MUST treat a refused statement as not current when {{STATEMENT}} requires currency at refresh, so that a withdrawal reaches running grants on the same terms as an expiry rather than waiting for the statement's own.

A receiver MAY discard a record naming a `jti` once that statement's `exp` has passed, and MAY discard any Withdrawal Record once every statement it could refuse has expired, which a receiver that does not retain statement lifetimes bounds by the maximum lifetime it honors for that issuer ({{STATEMENT}}). Discarding a record earlier reopens the withdrawal.

# Event Types {#events}

Each event is a member of the SET `events` claim, whose value is the event payload object. All payloads share these claims:

`event_timestamp`:
: REQUIRED. A NumericDate value giving the time the issuer made the decision the event reports. Receivers use it to order events ({{processing}}).

`software_statement_jti`:
: OPTIONAL. Present when the event withdraws one artifact rather than the decision behind it. Omitting it is the safe default: a withdrawal scoped to a single `jti` leaves a client's other unexpired statements untouched, which is the outcome {{withdrawal-records}} exists to prevent.

`reason_admin`:
: OPTIONAL. A human-readable explanation intended for an administrator, following the convention of {{CAEP}}.

## Withdrawn {#withdrawn}

The issuer ends a review before the statements carrying it expire, for example on delisting software, on discovering that a statement was mis-issued, or on a compromise of the client's key.

Without `software_statement_jti` the event withdraws the decision: the receiver refuses every statement from that issuer for that subject issued at or before the record's effective time. With it, the event withdraws one artifact and the receiver refuses that `jti` alone, which suits a mis-issued statement where the underlying review still stands.

A receiver MUST record the withdrawal ({{withdrawal-records}}) and stop treating the affected statements as current. At a server implementing the registration-validity model of {{STATEMENT}}, a registration governed by a refused statement is expired as that specification defines for a lapsed registration; at a server that does not, the registration remains subject to that server's own client lifecycle, and the record still refuses further statements. A runtime presentation carrying a refused statement fails as {{STATEMENT}} defines for a statement that is not current, and a grant already open under one continues only as far as its next currency check.

# Receiver Processing {#processing}

A consuming authorization server that receives an event defined here MUST:

1. verify the SET as {{RFC8417}} requires, including its `typ`, and verify that its issuer is configured and its keys were obtained as {{relationship}} requires;
2. reject an event whose `aud` does not contain its issuer identifier ({{relationship}}), and an event whose type it does not recognize;
3. resolve the subject ({{subjects}}) and the scope of the effect ({{relationship}}); and
4. apply the effect the event type defines, recording it as {{withdrawal-records}} requires.

A receiver MUST treat an event it has already applied as successfully delivered and acknowledge it as {{RFC8935}} or {{RFC8936}} requires, rather than reporting a delivery error; duplicate delivery is ordinary retry behavior and rejecting it can stall or disable a stream carrying later withdrawals. Duplicate detection is per transmitting issuer, since SET `jti` values are unique only within an issuer, and a receiver MAY bound the identifiers it retains for that purpose by the maximum statement lifetime it honors for the issuer.

Two constraints bound every event:

* An event MUST NOT create standing, extend a statement's lifetime, restore a statement previously withdrawn, or otherwise increase what a client may do. A receiver MUST ignore any payload member that would have such an effect. Restoring standing requires a statement, presented or delivered as {{STATEMENT}} defines.
* A receiver MUST continue to enforce statement expiry independently of this mechanism. Stream loss, transmitter unavailability, or delivery failure leaves standing to end at expiry where the registration-validity model of {{STATEMENT}} is in force, and otherwise at whatever the receiver's own client lifecycle provides; in both cases a missed event refuses later statements but does not reach standing already granted.

Events may arrive out of order or be duplicated. A receiver MUST apply every withdrawal event it accepts, whatever its `event_timestamp` relative to events already applied, since withdrawals accumulate and an older one may name a statement a newer one does not. The ordering rule constrains reversal only: a receiver MUST NOT discard or narrow a Withdrawal Record, or discard standing-bearing state whose `iat` is after the effective time of the event in hand, on the basis of an event whose `event_timestamp` precedes the event that created it. A delayed or replayed withdrawal therefore cannot destroy an approval the tenant recorded after making it. Records are kept per subject and issuer, and per `jti` where an event named one, so that a later event never silently supersedes an earlier one of narrower scope.

An effect applied by an event does not revoke access tokens already issued. A receiver applies its own grant and token lifetime policy, as it does when a statement expires.

# Stream Configuration {#configuration}

A statement issuer supporting this specification publishes Transmitter configuration metadata as {{SSF}} defines, discoverable from the issuer identifier the consuming authorization server has already configured. Stream creation, subject management, verification, and delivery follow {{SSF}}; this specification adds no configuration mechanism.

A consuming authorization server SHOULD create one stream per configured issuer, and that stream MUST cover every subject the issuer attests rather than an enumerated subject set. A receiver cannot enumerate subjects: in the runtime profile it holds no state for software until first presentation, which is exactly when an unenumerated withdrawal would already have been missed. A transmitter that supports this specification MUST accept a stream request that names no subjects and MUST NOT require subject enumeration as a condition of delivery.

A receiver SHOULD request the withdrawal event this specification defines, and SHOULD use the stream verification facility of {{SSF}} on a schedule, since a stream delivering nothing because it was misconfigured is otherwise indistinguishable from an issuer with nothing to report.

# Security Considerations

## What an Event Cannot Do

The constraints of {{processing}} are the security argument for this mechanism. A forged, replayed, or reordered event cannot grant a client anything, because no event increases standing, and {{withdrawal-records}} makes the reduction durable rather than momentary. Its worst outcome is the premature loss of a legitimate client's standing, which the issuer corrects by issuing a statement with a later `iat`. Forging an event requires the issuer's key, against which rate limiting is not a control; a receiver bounds the resource cost of event processing instead, and MUST NOT discard events it has accepted in order to shed load, since discarding a withdrawal is the suppression this mechanism exists to avoid.

## Expiry Remains the Floor

Because a receiver enforces expiry independently, an attacker who suppresses events, by disrupting delivery or the transmitter, delays a withdrawal at most until the affected statements expire. Deployments therefore choose lifetimes they would accept without this mechanism, and treat event delivery as an accelerator. A receiver SHOULD alert on stream loss rather than assume quiescence, since a healthy stream and a suppressed one are indistinguishable from the absence of events. Periodic stream verification ({{configuration}}) is what makes the difference observable.

## Key Separation and Compromise

A transmitter MAY sign SETs with a key distinct from its statement signing key, published in the same metadata. Compromise of a statement signing key is the more serious event, because statements grant standing and events cannot; a receiver responding to such a compromise removes trust in the issuer or its scope as {{STATEMENT}} describes, which also ends event acceptance.

## Comparison with Acceptance-Time Status

A status mechanism such as {{STATUS-LIST}} lets a receiver pull a statement's state at the moment it is consumed, at the cost of a retrieval on the consumption path and an availability dependence on the status endpoint. The events defined here push state changes ahead of consumption, at the cost of stream state and the suppression exposure above. The two are complementary, and a deployment can use both.

# Privacy Considerations

Subject identifiers in these events name software an issuer has reviewed and, in aggregate, describe an organization's approved software estate. A transmitter SHOULD scope each stream to the subjects the receiving authorization server can act on, and a receiver SHOULD apply to event logs the handling it applies to statements ({{STATEMENT}}).

Withdrawal Records ({{withdrawal-records}}) persist an issuer's negative decisions at every receiver, in some cases beyond the lifetime of the statements they concern. A receiver SHOULD retain them no longer than the refusal rules require and SHOULD apply to them the handling it applies to statements.

# IANA Considerations

## Event Type Assignment

The event defined in {{events}} requires a URI identifier under a namespace this specification does not yet control. This version uses the placeholder base `https://schemas.openid.net/secevent/sw-stmt/event-type/` with the terminal segment `withdrawn`. Assignment of the final base URI, and registration where the publishing body maintains a registry of security event types, is to be resolved before publication.

## SET Payload Claims

The payload claim `software_statement_jti` is defined by this specification for use in the event payload of {{events}}. Registration in a registry of SET payload claims, where the publishing body maintains one, is to be resolved before publication. The `event_timestamp` and `reason_admin` members follow the conventions of {{CAEP}}.

--- back

# Acknowledgments
{:numbered="false"}

This profile exists because bounding a review's lifetime and withdrawing it promptly are different requirements that a single expiry cannot serve. It draws on the Shared Signals Framework and the Continuous Access Evaluation Profile for its transport and payload conventions.
