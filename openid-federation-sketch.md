# Composing with OpenID Federation: A Sketch

This is a non-normative sketch, not a draft. It describes how the specifications in this repository could compose with OpenID Federation 1.0 (Final, 17 February 2026), and marks the one place where they cannot. If it were adopted it would become a short profile, roughly one section of normative text per profile below, adding no endpoint and no artifact beyond a Trust Mark type. Section numbers refer to OpenID Federation 1.0.

The summary is that the two compose on trust and stop at metadata. Federation supplies exactly what [the statement draft](draft-mcguinness-oauth-cimd-sw-stmt.md) declares out of scope, and the statement draft's metadata rule is incompatible with Resolved Metadata by construction.

The reason the two meet at all is worth stating before the profiles. Federation's metadata travels inside signed JWTs, so it needs no digest to be authentic. A digest matters where metadata is an unsigned, mutable JSON document fetched over HTTPS, which is what a Client ID Metadata Document is. A federation whose members identify clients that way has no means today of saying which version of such a document a reviewer looked at, and that is the gap these profiles fill. It is the same gap outside a federation, which is why the statement draft exists at all.

## Profile A: Federation as the trust source for statement issuers

The statement draft says it "defines no in-band issuer discovery or trust decision" and has a consuming server record seven things per issuer. Federation is that missing mechanism. A consuming server configures Trust Anchors and their public keys, which is all Federation requires of a relying party (§10), and resolves each issuer rather than enrolling it.

| What a consumer records today | Under this profile |
| --- | --- |
| The exact `iss` identifier | The statement's `iss` MUST be the Entity Identifier of an entity resolvable to a configured Trust Anchor |
| The source of the issuer's signing keys | The resolved `oauth_authorization_server` metadata for that entity |
| Signing algorithms accepted | Local, optionally narrowed by federation policy |
| Client identifier namespaces the issuer may attest | A metadata parameter bounded by superiors, below |
| Audience identifiers the issuer may name | Local |
| Maximum statement lifetime honored | Local |
| Policy on repeated and multiple registration | Local |

Three of the seven come from the chain: the identifier, the key source, and the namespaces the issuer may attest. The four that stay local are the consumer's own risk posture rather than facts about the issuer, so no discovery mechanism should supply them.

**Keys.** A software statement is signed with a protocol key, not a Federation Entity Key. Federation keeps these families apart and says Federation Entity Keys "SHOULD NOT be used in other protocols" (§3.1.1), so a consumer takes statement verification keys from the issuer's resolved `oauth_authorization_server` metadata, by `jwks`, `jwks_uri`, or `signed_jwks_uri`. The chain vouches for that metadata; it does not supply the signing key directly.

**Scope.** Federation's `naming_constraints` (§6.2.2) bounds which entities may sit beneath an intermediate, which is not the same question as which client identifiers a reviewer may vouch for. It answers the family's question only in the topology where the software publishers are themselves subordinates of the reviewer. The general mechanism is a metadata parameter on the issuer's `oauth_authorization_server` metadata naming the client identifier namespaces it attests, bounded by superiors with `subset_of`. That operator is restriction-only, so it raises none of the problems in the metadata boundary below. A superior can narrow what a reviewer claims for itself and can never widen it.

**The prohibition this has to clear.** The statement draft requires a consumer to derive trust from local configuration and forbids deriving it from an `iss`, `jku`, or `x5u` carried in a presented statement. A chain rooted at a locally configured Trust Anchor is local configuration with an indirection, not trust taken from the artifact. A statement MAY carry a chain as a hint, the way Federation itself permits a `trust_chain` header on a registration request, provided the consumer still verifies it to a Trust Anchor it configured. What stays forbidden is accepting a key location because the statement named it.

## Profile B: the review as a Trust Mark

Federation already has an artifact for a third party's signed assertion about an entity. A Trust Mark carries `iss`, `sub`, `trust_mark_type`, and `iat` as required claims, permits `exp`, and says plainly that "Additional Claims MAY be defined and used" (§7.1). A review fits with one added claim.

| Claim | Value in this profile |
| --- | --- |
| `iss` | The reviewer's Entity Identifier |
| `sub` | The client's Entity Identifier, which is also its Client ID Metadata Document URL |
| `trust_mark_type` | A type identifier for a CIMD review, to be assigned |
| `iat`, `exp` | As Federation defines, with `exp` REQUIRED here |
| `cimd_digest` | The digest of the exact octets the reviewer evaluated |

**`exp` is the one place this profile must override Federation.** Federation makes `exp` optional and says that absent, "the Trust Mark does not expire." A review that never expires is the condition this whole family exists to end, so the profile requires it. Everything the statement draft says about bounded lifetime carries over unchanged.

**Conveyance costs nothing new.** Trust Marks travel in the `trust_marks` claim of the client's Entity Configuration (§3.1.2), from the Trust Mark endpoint (§8.6), or in a resolve response, which returns only marks the resolver has verified (§8.3). A consumer already resolving the client obtains the review in the same pass.

**There is no circularity, which is what makes this work.** The statement draft refuses to treat a statement embedded in a reviewed document as a review, because a document cannot carry a statement issued over itself. That constraint does not apply here. An Entity Configuration is published at the Entity Identifier plus `/.well-known/openid-federation` (§9), so it is a different resource from the Client ID Metadata Document at the identifier itself. A Trust Mark inside the Entity Configuration can carry a digest over the document without covering its own bytes. This gives the "CIMD-native conveyance" extension point the statement draft defers a working home, in one deployment shape.

**Issuer authorization is native.** A Trust Anchor's `trust_mark_issuers` claim names which issuers may issue which Trust Mark types (§3.1.2), so a federation states centrally which reviewers it accepts for CIMD review. That is the same decision Profile A configures per consumer, expressed once for the federation.

**Withdrawal uses the Trust Mark Status endpoint.** A POST carrying the mark returns a signed response whose `status` is `active`, `expired`, `revoked`, or `invalid` (§8.4). Within a federation this replaces Token Status List for the same purpose. The properties the signals draft depends on hold either way: the answer comes from the issuer, the consumer resolves it, and expiry remains the floor.

**One default to change.** Client authentication is not used at federation endpoints unless a federation requires it (§8.8), and Federation treats trust-mark holdership as public infrastructure data: §19.2 protects the verifier from being tracked by the issuer and recommends bulk-fetching the list of trust-marked entities to that end. A review that is a commercial fact about a publisher survives that posture. A review that records which organizations approved a vendor does not, and a deployment carrying those has to require `private_key_jwt` at the trust mark endpoint and not publish a listing endpoint. Federation supplies the authentication primitives and no authorization model to go with them.

## The metadata boundary

A consuming server under either profile takes the client's registration metadata from the Client ID Metadata Document, digest-verified, and not from Federation's Resolved Metadata. This is a boundary rather than a precedence rule, because no precedence rule would help.

Resolved Metadata is a derived object that two parties other than the publisher can change:

* A Subordinate Statement's `metadata` claim has "precedence and override identically named parameters under the same Entity Type in the subject's Entity Configuration" (§3.1.1).
* The `value`, `add`, and `default` policy operators create or replace values, and `add` initializes a parameter that was absent entirely (§6.1.3.1).
* Under Explicit Registration, "The OP MAY modify the received RP metadata" (§12.2.2).

The statement draft requires a consumer to derive registered metadata by parsing the same octets it digested and to take every metadata value from that document. Resolved Metadata is by construction not those octets. The draft already generalizes the rule that settles this: any condition that makes retrieval depend on who is asking defeats the comparison, whatever its cause. Metadata policy differs by chain, so Resolved Metadata differs by who resolved it, deliberately.

A deployment that wants Resolved Metadata to govern its clients should use Federation's own registration and not this family. The failure mode to avoid is running both for the same client and assuming they agree.

## What Federation already does better

Stated plainly, because it narrows what these drafts are worth inside a federation.

* **Expiring standing is native.** "The validity of an Automatic or Explicit Registration at an OP MUST NOT exceed the lifetime of the Trust Chain the OP used to create the registration" (§12.3), and a chain's expiry is the minimum `exp` in it (§10.4). The registration-validity model in the statement draft answers a question a federated deployment has already answered. It applies to servers keeping a persistent RFC 7591 registration outside a federation.
* **Issuer trust at scale, key discovery, and rotation.** Only Trust Anchor keys are configured locally; everything else chains. The statement draft's issuer-trust section defines no mechanism at all.
* **Central statements of who is accepted**, through `trust_mark_issuers` and metadata policy, rather than per-consumer configuration.

## When not to use this

If a deployment is already operating a federation and is content for Resolved Metadata to define its clients, Federation's own registration serves it and neither profile is needed. These profiles are for a deployment that has chosen CIMD documents as the metadata source and wants a reviewer's decision bound to a specific version of one.

## Open questions

* **The Trust Mark type identifier needs a namespace**, and the venue question is the same one [the signals draft](draft-mcguinness-oauth-cimd-sw-stmt-signals.md) carries: the artifact and its digest are IETF work, Federation is OpenID Foundation work.
* **Whether one URL can serve both roles is unverified.** Federation Entity Identifiers permit a path and forbid query and fragment, and the Entity Configuration is a suffix beneath the identifier, so co-location looks available. This has not been checked against what CIMD requires of a client identifier URL, and that check should come before anything here is written up.
* **Should the statement and the Trust Mark be interchangeable**, or is the Trust Mark the only Federation-native form? Carrying the same review in two artifacts reopens the question of which governs when they disagree.
* **Federation 1.1 exists** and has not been diffed against 1.0 for anything above.
* **Profile A and Profile B are independent.** A deployment can take issuer trust from a federation while still conveying reviews as software statements, and that may be the more common shape than full Trust Mark conveyance.
