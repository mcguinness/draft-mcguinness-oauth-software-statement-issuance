# CIMD Software Statement Issuance

This is the working area for the individual Internet-Draft, "CIMD Software Statement Issuance".

RFC 7591 standardizes how a client presents a software statement and how a registration endpoint consumes it, but not how the client obtains one. This specification defines the missing issuance protocol: a redirect flow returning a `software_statement_code` redeemed at the token endpoint, a token exchange profile for pre-authorized issuance and renewal, and deferred processing under Deferred Token Response so approval can complete out of band over hours or days. Clients are identified by a Client ID Metadata Document, and every statement is bound by a byte-exact digest to the document reviewed.

* [Editor's Copy (HTML)](https://mcguinness.github.io/draft-mcguinness-oauth-cimd-sw-stmt-issuance/draft-mcguinness-oauth-cimd-sw-stmt-issuance.html)
* [Editor's Copy (TXT)](https://mcguinness.github.io/draft-mcguinness-oauth-cimd-sw-stmt-issuance/draft-mcguinness-oauth-cimd-sw-stmt-issuance.txt)
* [Datatracker Page](https://datatracker.ietf.org/doc/draft-mcguinness-oauth-cimd-sw-stmt-issuance/) (after first submission)

## CIMD Software Statement

This repository also hosts the companion Internet-Draft, "CIMD Software Statement", which defines the artifact, its validation, the issuer trust a consumer configures, and the two points at which a statement is consumed: in a registration request, where its expiry can bound the registration, and at runtime, where it establishes a client for one request without creating a registration.

* [Editor's Copy (HTML)](https://mcguinness.github.io/draft-mcguinness-oauth-cimd-sw-stmt-issuance/draft-mcguinness-oauth-cimd-sw-stmt.html)
* [Editor's Copy (TXT)](https://mcguinness.github.io/draft-mcguinness-oauth-cimd-sw-stmt-issuance/draft-mcguinness-oauth-cimd-sw-stmt.txt)
* [Datatracker Page](https://datatracker.ietf.org/doc/draft-mcguinness-oauth-cimd-sw-stmt/) (after first submission)

## Shared Signals Events for CIMD Software Statements

The repository also hosts "Shared Signals Events for CIMD Software Statements", an optional profile that reduces how long a withdrawal takes to reach a consumer. An issuer ends a decision before its expiry by publishing status through Token Status List, which a trusting authorization server resolves on its own schedule; this profile lets the issuer say that a status changed so the server resolves it at once instead. The event names no status, so the list remains the only authority on whether a statement stands, and a receiver that misses every event reaches the same answer on its ordinary schedule. Neither of the other drafts depends on this one.

* [Editor's Copy (HTML)](https://mcguinness.github.io/draft-mcguinness-oauth-cimd-sw-stmt-issuance/draft-mcguinness-oauth-cimd-sw-stmt-signals.html)
* [Editor's Copy (TXT)](https://mcguinness.github.io/draft-mcguinness-oauth-cimd-sw-stmt-issuance/draft-mcguinness-oauth-cimd-sw-stmt-signals.txt)
* [Datatracker Page](https://datatracker.ietf.org/doc/draft-mcguinness-oauth-cimd-sw-stmt-signals/) (after first submission)

## Deployment Model

[Deployment Model: Portable Review Across Many Authorization Servers](deployment-model.md) sketches the end-to-end scenario the drafts are built for: a provider marketplace deciding which software may exist as a client, an enterprise deciding which of it may operate in its tenant, and the four layers that keep those decisions separate. Non-normative.

## A Mobile App, End to End

[Sketch: A Mobile App from Install to Third Party](mobile-app-sketch.md) walks one scenario through all three drafts: an app store install refused by an enterprise identity provider as unreviewed, a statement obtained and awaited while an administrator approves, sign-in, and then a third-party SaaS reached through an identity assertion. It stops at what the family still does not give a mobile install: the device key binds a grant to one phone, but nobody vouches for that key, so a server learns the same installation came back and never which one it is. Non-normative.

## A Customer Approving a Vendor Integration

[Sketch: A Customer Approving a Vendor Integration at Their Own Root](saas-integration-sketch.md) takes the case these drafts serve least well, a vendor-to-vendor integration with no user present while it runs. The customer approves the vendor once at the enterprise root their organization already operates, and every platform configured with that root honours it. Whether a platform's own marketplace listed the integration is that platform's business and not the customer's. The prize is one revocation, performed once by the customer, that every platform sees. Non-normative.

## Tenant Admin Consent

[Sketch: Tenant Admin Consent Without a Stored Grant](admin-consent-sketch.md) takes Microsoft Entra ID's tenant-wide admin consent as the reference and asks what each of its parts looks like built from open specifications. Three have generic forms already, one of them better than the original, since Entra's approval loop cannot be automated at all. The fourth, a consent that binds a population rather than a principal, has no form anywhere, and the sketch argues that is the right outcome. Non-normative.

## Composing with OpenID Federation

[Composing with OpenID Federation](openid-federation-sketch.md) sketches how these drafts could take issuer trust from a federation rather than enrolling each reviewer, how a review could travel as a Trust Mark carrying `cimd_digest`, and the one place the two models cannot be reconciled: Federation derives an entity's metadata by policy, so it is not the octets a digest covers. Non-normative, and not a draft.

## Related Drafts

* [OAuth Client ID Metadata Document](https://datatracker.ietf.org/doc/draft-ietf-oauth-client-id-metadata-document/) (normative dependency)
* [Token Status List](https://datatracker.ietf.org/doc/draft-ietf-oauth-status-list/) (normative dependency)
* [Deferred Token Response](https://datatracker.ietf.org/doc/draft-gerber-oauth-deferred-token-response/) (normative dependency)
* [OpenID Federation 1.0](https://openid.net/specs/openid-federation-1_0.html) (composition sketched above)
* [OAuth 2.0 Client Instance Assertion](https://datatracker.ietf.org/doc/draft-mcguinness-oauth-client-instance-assertion/) (instance layer; composes with this draft)
* [OAuth Identity Assertion Trust Framework](https://datatracker.ietf.org/doc/draft-mcguinness-oauth-id-assertion-framework/) (issuer trust generalization)

## Contributing

See the [guidelines for contributions](CONTRIBUTING.md).

Contributions can be made by creating pull requests. The GitHub interface supports creating pull requests using the Edit (✏) button.

## Command Line Usage

Formatted text and HTML versions of the draft can be built using `make`.

```sh
$ make
```

Command line usage requires that you have the necessary software installed. See [the instructions](https://github.com/martinthomson/i-d-template/blob/main/doc/SETUP.md).
