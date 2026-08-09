# CIMD Software Statement Issuance

This is the working area for the individual Internet-Draft, "CIMD Software Statement Issuance".

RFC 7591 standardizes how a client presents a software statement and how a registration endpoint consumes it, but not how the client obtains one. This specification defines the missing issuance protocol: a redirect flow returning a `software_statement_code` redeemed at the token endpoint, a token exchange profile for pre-authorized issuance and renewal, and deferred processing under Deferred Token Response so approval can complete out of band over hours or days. Clients are identified by a Client ID Metadata Document, and every statement is bound by canonical digest to the exact metadata reviewed.

* [Editor's Copy (HTML)](https://mcguinness.github.io/draft-mcguinness-oauth-cimd-sw-stmt-issuance/draft-mcguinness-oauth-cimd-sw-stmt-issuance.html)
* [Editor's Copy (TXT)](https://mcguinness.github.io/draft-mcguinness-oauth-cimd-sw-stmt-issuance/draft-mcguinness-oauth-cimd-sw-stmt-issuance.txt)
* [Datatracker Page](https://datatracker.ietf.org/doc/draft-mcguinness-oauth-cimd-sw-stmt-issuance/) (after first submission)

## OAuth 2.0 Software Statement Runtime Presentation

This repository also hosts the companion Internet-Draft, "OAuth 2.0 Software Statement Runtime Presentation", which defines the second consumption path for the issued statement: a client presents it at runtime in an authorization or token request, sender-constrained by a proof that chains to the statement, and the authorization server applies the attested metadata to that request without creating a persistent registration.

* [Editor's Copy (HTML)](https://mcguinness.github.io/draft-mcguinness-oauth-cimd-sw-stmt-issuance/draft-mcguinness-oauth-cimd-sw-stmt.html)
* [Editor's Copy (TXT)](https://mcguinness.github.io/draft-mcguinness-oauth-cimd-sw-stmt-issuance/draft-mcguinness-oauth-cimd-sw-stmt.txt)
* [Datatracker Page](https://datatracker.ietf.org/doc/draft-mcguinness-oauth-cimd-sw-stmt/) (after first submission)

## Shared Signals Events for CIMD Software Statements

The repository also hosts "Shared Signals Events for CIMD Software Statements", an optional profile that lets a statement issuer withdraw a decision before the statement expires. The issuer transmits lifecycle events over the Shared Signals Framework to the authorization servers that already trust it: statement revoked, approval withdrawn, establishment withdrawn, metadata changed, plus an optional reverse-direction event by which a consuming server reports where a statement is in force. Events can only reduce standing, and a receiver that misses one falls back to expiry, so neither of the other drafts depends on this one.

* [Editor's Copy (HTML)](https://mcguinness.github.io/draft-mcguinness-oauth-cimd-sw-stmt-issuance/draft-mcguinness-oauth-cimd-sw-stmt-signals.html)
* [Editor's Copy (TXT)](https://mcguinness.github.io/draft-mcguinness-oauth-cimd-sw-stmt-issuance/draft-mcguinness-oauth-cimd-sw-stmt-signals.txt)
* [Datatracker Page](https://datatracker.ietf.org/doc/draft-mcguinness-oauth-cimd-sw-stmt-signals/) (after first submission)

## Deployment Model

[Marketplace Listing and Enterprise Approval](deployment-model.md) sketches the end-to-end scenario the two drafts are built for: a provider marketplace deciding which software may exist as a client, an enterprise deciding which of it may operate in its tenant, and the four layers that keep those decisions separate. Non-normative.

## Related Drafts

* [OAuth Client ID Metadata Document](https://datatracker.ietf.org/doc/draft-ietf-oauth-client-id-metadata-document/) (normative dependency)
* [Deferred Token Response](https://datatracker.ietf.org/doc/draft-gerber-oauth-deferred-token-response/) (normative dependency)
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
