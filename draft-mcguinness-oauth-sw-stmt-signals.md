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

This specification defines the subject identification, event types, payload claims, the durable records a receiver keeps, and its processing rules. It defines no new endpoint, transport, subject identifier format, or trust establishment mechanism.

Two properties bound what the mechanism can do, and are normative in {{processing}}: an event can only reduce standing, and a receiver that misses events enforces expiry exactly as it does today. A withdrawal is durable rather than momentary, which {{withdrawal-records}} specifies, so a client holding an unexpired statement cannot restore what an event ended. Deployments therefore keep bounded statement lifetimes, and neither {{ISSUANCE}} nor {{PRESENTATION}} depends on this specification.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

Transmitter, Receiver, Stream, and the delivery and configuration mechanisms are defined by {{SSF}}. Security Event Token, or SET, is defined by {{RFC8417}}. Subject identifier formats are defined by {{RFC9493}}. Software statement, issuer roles including the establishment and tenant approval roles, and statement validation are defined by {{ISSUANCE}}. Registration validity, runtime presentation, tenant approval evaluation, and the per-grant establishment state are defined by {{PRESENTATION}}.

This specification additionally defines the following terms:

Statement Issuer:
: The party that signed a software statement, acting as a Transmitter of the events defined here.

Consuming Authorization Server:
: A trusting authorization server that has configured the statement issuer, acting as a Receiver of the events defined here.

Standing:
: What a client may do at a consuming authorization server by virtue of a software statement: its registration and that registration's validity ({{PRESENTATION}}), its establishment at request time, and any tenant approval in force. Standing is created and extended only by statements, never by the events defined here.

Withdrawal Record:
: The state a receiver retains when it applies a withdrawal event, defined in {{withdrawal-records}}.

# Relationship to the Statement Family {#relationship}

The events defined here carry no authority of their own. They report that a decision the issuer already had the authority to make has ended earlier than its expiry announced.

The scope of an event's effect follows the event type, which names the layer it bears on. An `establishment-withdrawn` event bears on the client's existence at that server, for every tenant. An `approval-withdrawn` event bears only on the tenant its transmitting issuer speaks for, which the receiver resolves from its own configuration; events carry no tenant identifier. A receiver MUST reject an event whose type names a layer for which it does not accept the transmitting issuer, and where an issuer holds both roles ({{ISSUANCE}}) the event type alone determines scope.

A consuming authorization server MUST NOT accept an event from an issuer it has not configured, and MUST verify the SET using keys obtained as it obtains that issuer's statement verification keys, from the issuer's authorization server metadata {{RFC8414}}. It MUST NOT derive event trust from any key-location value carried in the event.

A SET carrying an event defined here MUST use the explicit `typ` header value `secevent+jwt` ({{RFC8417}}), so that a receiver validating JWTs from a configured issuer distinguishes an event from a software statement, whose own typing {{ISSUANCE}} fixes. Its `aud` MUST contain the receiving authorization server's issuer identifier as defined by {{RFC8414}}, which is the value that appears in a statement's `aud`; a stream audience negotiated under {{SSF}} does not replace it.

# Subject Identification {#subjects}

The subject of every event defined here is client software, identified as the `sub` of the statements the event concerns. Events use the `sub_id` claim of {{SSF}} with the formats of {{RFC9493}}:

* Software identified by a Client ID Metadata Document URL {{CIMD}} uses the `uri` format, whose `uri` member carries that URL exactly as it appears in the statement's `sub`.
* Software identified by an {{RFC7591}} `software_id` uses the `iss_sub` format, whose `iss` member carries the statement issuer's identifier and whose `sub` member carries the `software_id`.

A consuming authorization server MUST match the subject by exact comparison against the statements it holds and the subjects of its Withdrawal Records: the `uri` member against a statement's `sub`, and for an `iss_sub` identifier both members, its `iss` against the statement's `iss` and its `sub` against the statement's `sub`. A statement whose `sub` is a client identifier already registered at the authorization server ({{ISSUANCE}}) has no subject identifier format defined here and is outside this specification.

A receiver MUST record a withdrawal for a subject it does not currently recognize ({{withdrawal-records}}) rather than discarding the event. A subject unknown at the time of an event is the ordinary case in the runtime profile, where a receiver holds no state for software until it is first presented.

# Withdrawal Records {#withdrawal-records}

A withdrawal event names a decision that ended. Its effect is durable, and a receiver that only ceases some current activity has not applied it: in the runtime profile there is no current activity to cease, and in the registration profile the client holds a statement that would otherwise restore what the event withdrew.

On applying `statement-revoked`, `approval-withdrawn`, or `establishment-withdrawn`, a consuming authorization server MUST create or update a Withdrawal Record holding the transmitting issuer, the subject, the event type, an effective time, and, for `statement-revoked`, the revoked `jti`.

The effective time is the later of the event's `event_timestamp` and the time the receiver accepted the event. A receiver cannot verify that an issuer stopped issuing when it decided to withdraw, and an issuer whose renewal is automated may mint a scheduled statement between the decision and the event's arrival; taking the later of the two times keeps such a statement from restoring what the withdrawal ended. An issuer MUST NOT issue a statement for a subject after transmitting a withdrawal for that subject unless it has made a new decision to do so, and SHOULD suspend automated renewal for a subject before transmitting.

While a Withdrawal Record is retained, the receiver MUST refuse a statement matching it:

* a `statement-revoked` record refuses the statement with that `jti`, whatever its `iat`;
* an `approval-withdrawn` or `establishment-withdrawn` record refuses every statement from that issuer for that subject whose `iat` is at or before the record's effective time.

A statement from that issuer for that subject with an `iat` after the record's effective time is a later decision by the same authority. It is accepted under the ordinary rules of {{ISSUANCE}} and {{PRESENTATION}}, and restores what the withdrawal ended. This is the only recovery path, and it requires the issuer to act.

A refusal under this section takes effect wherever the statement would otherwise be consumed: a registration governed by a refused statement is expired as {{PRESENTATION}} defines for a lapsed registration, a runtime presentation carrying one fails as it does for a statement that is not current, and a tenant approval carrying one is treated as absent. A grant already open under a refused statement continues only as far as its next currency check: a receiver MUST treat a refused statement as not current when {{PRESENTATION}} requires currency at refresh, so that a withdrawal reaches running grants on the same terms as an expiry rather than waiting for the statement's own.

A receiver MAY discard a `statement-revoked` record once the revoked statement's `exp` has passed, and MAY discard any Withdrawal Record once every statement it could refuse has expired, which a receiver that does not retain statement lifetimes bounds by the maximum lifetime it honors for that issuer ({{ISSUANCE}}). Discarding a record earlier reopens the withdrawal.

# Event Types {#events}

Each event is a member of the SET `events` claim, whose value is the event payload object. All payloads share these claims:

`event_timestamp`:
: REQUIRED. A NumericDate value giving the time the issuer made the decision the event reports. Receivers use it to order events ({{processing}}).

`software_statement_jti`:
: REQUIRED in `statement-revoked`, where it names the artifact withdrawn. It MUST NOT appear in `approval-withdrawn` or `establishment-withdrawn`, which withdraw a decision rather than an artifact and therefore reach every statement the issuer has issued for the subject up to the record's effective time; a receiver MUST ignore it if present in those events. Narrowing a decision withdrawal to one artifact would leave a client's other unexpired statements untouched, which is the outcome {{withdrawal-records}} exists to prevent.

`reason_admin`:
: OPTIONAL. A human-readable explanation intended for an administrator, as {{CAEP}} defines the member.

## Statement Revoked {#statement-revoked}

The issuer withdraws a specific statement before its expiry, for example because it was mis-issued or its holder's key was compromised. `software_statement_jti` is REQUIRED for this event type.

A receiver MUST record the revocation ({{withdrawal-records}}) and stop honoring the named statement. Where that statement governs a registration's validity ({{PRESENTATION}}) at a server implementing that model, the registration is expired as though its recorded `exp` had passed. A replacement statement restores it under the revalidation rules of that specification, subject to the refusal rules of {{withdrawal-records}}, which prevent the revoked statement itself from serving as the replacement.

## Approval Withdrawn {#approval-withdrawn}

A tenant approval issuer withdraws its tenant's approval of the subject software.

A receiver MUST record the withdrawal ({{withdrawal-records}}) and stop treating the subject as approved for that issuer's tenant, with the effect the tenant approval rules of {{PRESENTATION}} give an absent approval. Where the receiver holds a recorded tenant approval from that issuer for that subject and tenant, it is discarded unless its `iat` is after the record's effective time, in which case it was recorded from a later decision and is retained. The client's establishment, and every other tenant, are unaffected.

## Establishment Withdrawn {#establishment-withdrawn}

An establishment issuer withdraws the subject software's listing, for example on removal from a marketplace.

A receiver MUST record the withdrawal ({{withdrawal-records}}) and stop treating the subject as established. At a server implementing the registration-validity model of {{PRESENTATION}}, registrations governed by statements from that issuer for that subject are expired; at a server that does not, the registrations remain subject to that server's own client lifecycle, and the record still refuses further statements. A consuming authorization server implementing this specification MUST retain, for each registration it creates from a statement, that statement's `iss`, `sub`, and `iat`, whether or not it implements the registration-validity model, since a withdrawal names registrations by those values. Runtime presentation of a refused statement fails as {{PRESENTATION}} defines for a statement that is not current. The effect reaches every tenant at that authorization server.

## Metadata Changed {#metadata-changed}

The issuer reports that the software's published metadata no longer matches what it reviewed. This event carries information a receiver could otherwise learn only by retrieval, and is advisory: the issuer is not withdrawing its decision.

`observed_cimd_digest`:
: OPTIONAL. The digest the issuer now observes at the subject's Client ID Metadata Document, in the encoding {{ISSUANCE}} defines. It is not the statement's attested `cimd_digest`, and MUST NOT be recorded as one.

`observed_software_version`:
: OPTIONAL. The {{RFC7591}} `software_version` the issuer now observes, for a subject whose statements attest that member rather than a digest ({{ISSUANCE}}).

A receiver that retrieves and compares metadata ({{ISSUANCE}}) applies its post-issuance change policy as though it had made that observation itself. A receiver that does not implement retrieval and comparison MAY ignore this event.

The event can only add a reason to distrust current metadata. A receiver MUST NOT treat it as attesting metadata, and MUST NOT use it to clear a restriction already applied, including where an observed value matches what a statement attests; only a retrieval the receiver performs itself, or a later statement, can do that.

## Reverse-Direction Reporting

A consuming authorization server reporting back to an issuer where its statements are consumed would give the issuer an inventory it cannot otherwise obtain. It is not defined here: the direction reverses the roles, so the authorization server would publish transmitter configuration, the issuer would discover it and create a stream, and the processing rules of {{processing}} would need a counterpart for the issuer as receiver. That is a separate profile, with privacy considerations of its own, since presentation-time reporting would carry near-real-time authorization activity per tenant.

# Receiver Processing {#processing}

A consuming authorization server that receives an event defined here MUST:

1. verify the SET as {{RFC8417}} requires, including its `typ`, and verify that its issuer is configured and its keys were obtained as {{relationship}} requires;
2. reject an event whose `aud` does not contain its issuer identifier ({{relationship}}), and an event whose type it does not recognize or whose type names a layer it does not accept that issuer for;
3. resolve the subject ({{subjects}}) and the scope of the effect from the event type ({{relationship}}); and
4. apply the effect the event type defines, recording it as {{withdrawal-records}} requires.

A receiver MUST treat an event it has already applied as successfully delivered and acknowledge it as {{RFC8935}} or {{RFC8936}} requires, rather than reporting a delivery error; duplicate delivery is ordinary retry behavior and rejecting it can stall or disable a stream carrying later withdrawals. Duplicate detection is per transmitting issuer, since SET `jti` values are unique only within an issuer, and a receiver MAY bound the identifiers it retains for that purpose by the maximum statement lifetime it honors for the issuer.

Two constraints bound every event:

* An event MUST NOT create standing, extend a statement's lifetime, restore a statement previously revoked, or otherwise increase what a client may do. A receiver MUST ignore any payload member that would have such an effect. Restoring standing requires a statement, presented or delivered as {{PRESENTATION}} defines.
* A receiver MUST continue to enforce statement expiry independently of this mechanism. Stream loss, transmitter unavailability, or delivery failure leaves standing to end at expiry, which remains the floor.

Events may arrive out of order or be duplicated. A receiver MUST apply every withdrawal event it accepts, whatever its `event_timestamp` relative to events already applied, since withdrawals accumulate and an older one may name a statement a newer one does not. The ordering rule constrains reversal only: a receiver MUST NOT discard or narrow a Withdrawal Record, or discard standing-bearing state whose `iat` is after the effective time of the event in hand, on the basis of an event whose `event_timestamp` precedes the event that created it. A delayed or replayed withdrawal therefore cannot destroy an approval the tenant recorded after making it. Records are kept per subject and issuer, and per `jti` for `statement-revoked`, so that a later event never silently supersedes an earlier one of a different kind.

An effect applied by an event does not revoke access tokens already issued. A receiver applies its own grant and token lifetime policy, as it does when a statement expires.

# Stream Configuration {#configuration}

A statement issuer supporting this specification publishes Transmitter configuration metadata as {{SSF}} defines, discoverable from the issuer identifier the consuming authorization server has already configured. Stream creation, subject management, verification, and delivery follow {{SSF}}; this specification adds no configuration mechanism.

A consuming authorization server SHOULD create one stream per configured issuer, and that stream MUST cover every subject the issuer attests rather than an enumerated subject set. A receiver cannot enumerate subjects: in the runtime profile it holds no state for software until first presentation, which is exactly when an unenumerated withdrawal would already have been missed. A transmitter that supports this specification MUST accept a stream request that names no subjects and MUST NOT require subject enumeration as a condition of delivery.

A receiver SHOULD request every event type this specification defines that the transmitting issuer's roles can produce, and SHOULD use the stream verification facility of {{SSF}} on a schedule, since a stream delivering nothing because it was misconfigured is otherwise indistinguishable from an issuer with nothing to report.

# Security Considerations

## What an Event Cannot Do

The constraints of {{processing}} are the security argument for this mechanism. A forged, replayed, or reordered event cannot grant a client anything, because no event increases standing, and {{withdrawal-records}} makes the reduction durable rather than momentary. Its worst outcome is the premature loss of a legitimate client's standing, which the issuer corrects by issuing a statement with a later `iat`. Forging an event requires the issuer's key, against which rate limiting is not a control; a receiver bounds the resource cost of event processing instead, and MUST NOT discard events it has accepted in order to shed load, since discarding a withdrawal is the suppression this mechanism exists to avoid.

## Expiry Remains the Floor

Because a receiver enforces expiry independently, an attacker who suppresses events, by disrupting delivery or the transmitter, delays a withdrawal at most until the affected statements expire. Deployments therefore choose lifetimes they would accept without this mechanism, and treat event delivery as an accelerator. A receiver SHOULD alert on stream loss rather than assume quiescence, since a healthy stream and a suppressed one are indistinguishable from the absence of events. Periodic stream verification ({{configuration}}) is what makes the difference observable.

## Key Separation and Compromise

A transmitter MAY sign SETs with a key distinct from its statement signing key, published in the same metadata. Compromise of a statement signing key is the more serious event, because statements grant standing and events cannot; a receiver responding to such a compromise removes trust in the issuer or its scope as {{ISSUANCE}} describes, which also ends event acceptance.

## Comparison with Acceptance-Time Status

A status mechanism such as {{STATUS-LIST}} lets a receiver pull a statement's state at the moment it is consumed, at the cost of a retrieval on the consumption path and an availability dependence on the status endpoint. The events defined here push state changes ahead of consumption, at the cost of stream state and the suppression exposure above. The two are complementary, and a deployment can use both.

# Privacy Considerations

Subject identifiers in these events name software an issuer has reviewed and, in aggregate, describe an organization's approved software estate. A transmitter SHOULD scope each stream to the subjects the receiving authorization server can act on, and a receiver SHOULD apply to event logs the handling it applies to statements ({{ISSUANCE}}).

Withdrawal Records ({{withdrawal-records}}) persist an issuer's negative decisions at every receiver, in some cases beyond the lifetime of the statements they concern. A receiver SHOULD retain them no longer than the refusal rules require and SHOULD apply to them the handling it applies to statements.

# IANA Considerations

## Event Type Assignment

The event types defined in {{events}} require URI identifiers under a namespace this specification does not yet control. This version uses the placeholder base `https://schemas.openid.net/secevent/sw-stmt/event-type/` with the terminal segments `statement-revoked`, `approval-withdrawn`, `establishment-withdrawn`, and `metadata-changed`. Assignment of the final base URI, and registration where the publishing body maintains a registry of security event types, is to be resolved before publication.

## SET Payload Claims

The payload claims `software_statement_jti`, `observed_cimd_digest`, and `observed_software_version` are defined by this specification for use in the event payloads of {{events}}. Registration in a registry of SET payload claims, where the publishing body maintains one, is to be resolved before publication. The `event_timestamp` and `reason_admin` members are used as {{CAEP}} defines them.

--- back

# Acknowledgments
{:numbered="false"}

This profile exists because bounding a review's lifetime and withdrawing it promptly are different requirements that a single expiry cannot serve. It draws on the Shared Signals Framework and the Continuous Access Evaluation Profile for its transport and payload conventions.
