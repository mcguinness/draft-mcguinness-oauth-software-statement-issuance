# CIMD Software Statements

This is the working area for a family of individual Internet-Drafts on software statements for OAuth clients identified by a [Client ID Metadata Document](https://datatracker.ietf.org/doc/draft-ietf-oauth-client-id-metadata-document/) (CIMD).

A Client ID Metadata Document carries a client's own claims and can change at any time. These drafts let a party that reviews client software, such as a platform marketplace or an enterprise security team, record its review of the exact bytes of that document in a signed software statement. Any authorization server that trusts the reviewer can then enforce that review when the client arrives, without registering it.

New to the drafts? Start with the [FAQ](https://github.com/mcguinness/draft-mcguinness-oauth-software-statement-issuance/wiki/FAQ).

## The Drafts

| Draft | What it defines | Editor's copy |
| --- | --- | --- |
| **CIMD Software Statement** | The statement and its digest binding, validation, issuer trust, and runtime admission by a statement presented in a request or pulled from where the client's document points | [HTML](https://mcguinness.github.io/draft-mcguinness-oauth-software-statement-issuance/draft-mcguinness-oauth-cimd-sw-stmt.html), [TXT](https://mcguinness.github.io/draft-mcguinness-oauth-software-statement-issuance/draft-mcguinness-oauth-cimd-sw-stmt.txt) |
| CIMD Software Statement Registration | Consuming a statement in an RFC 7591 registration: metadata taken from the reviewed document, validity bounded by the statement, and renewal by a replacement statement | [HTML](https://mcguinness.github.io/draft-mcguinness-oauth-software-statement-issuance/draft-mcguinness-oauth-cimd-sw-stmt-registration.html), [TXT](https://mcguinness.github.io/draft-mcguinness-oauth-software-statement-issuance/draft-mcguinness-oauth-cimd-sw-stmt-registration.txt) |
| CIMD Software Statement Issuance | How a client obtains a statement: token exchange, a redirect flow returning a `software_statement_code`, and asynchronous review under Deferred Token Response | [HTML](https://mcguinness.github.io/draft-mcguinness-oauth-software-statement-issuance/draft-mcguinness-oauth-cimd-sw-stmt-issuance.html), [TXT](https://mcguinness.github.io/draft-mcguinness-oauth-software-statement-issuance/draft-mcguinness-oauth-cimd-sw-stmt-issuance.txt) |
| Shared Signals Events for CIMD Software Statements | An optional event telling an authorization server that a statement's status changed, so it checks the status list at once rather than on its next scheduled check | [HTML](https://mcguinness.github.io/draft-mcguinness-oauth-software-statement-issuance/draft-mcguinness-oauth-cimd-sw-stmt-signals.html), [TXT](https://mcguinness.github.io/draft-mcguinness-oauth-software-statement-issuance/draft-mcguinness-oauth-cimd-sw-stmt-signals.txt) |

The statement draft is the core. The other three build on it, and it depends on none of them.

The statement and registration drafts are planned for submission; their Datatracker pages will be available after first submission:
[statement](https://datatracker.ietf.org/doc/draft-mcguinness-oauth-cimd-sw-stmt/),
[registration](https://datatracker.ietf.org/doc/draft-mcguinness-oauth-cimd-sw-stmt-registration/).
The issuance and signals drafts are working drafts kept here for discussion and are not planned for submission yet.

## MCP Profile

[Reviewed Client Software](mcp/reviewed-client-software.mdx) is a draft MCP authorization extension, written in the format of MCP's extensions repository for proposal there. MCP authorization servers pull statements from the `software_statements_uri` a client's metadata document names, treat desktop clients with loopback redirects as review-only, and compose the review with Enterprise-Managed Authorization.

## Background and Sketches

These documents explore deployments and edge cases. They are non-normative and not drafts.

| Document | What it covers |
| --- | --- |
| [Deployment Model](sketches/deployment-model.md) | A provider marketplace deciding which software may exist as a client, an enterprise deciding which of it may operate in its tenant, and the layers that keep those decisions separate |
| [A Mobile App, End to End](sketches/mobile-app-sketch.md) | An app store install through review, administrator approval, sign-in, and a third-party SaaS reached through an identity assertion, ending at what a per-install key still lacks |
| [A Customer Approving a Vendor Integration](sketches/saas-integration-sketch.md) | A vendor-to-vendor integration with no user present, comparing the customer's own statement with an approval made once at the customer's root and revoked once for every platform |
| [Tenant Admin Consent](sketches/admin-consent-sketch.md) | Microsoft Entra ID's tenant-wide admin consent, rebuilt from open specifications |
| [Composing with OpenID Federation](sketches/openid-federation-sketch.md) | Issuer trust from a federation, a review carried as a Trust Mark, and why Federation's resolved metadata cannot carry a digest |
| [Pulling the Review, and an MCP Profile](sketches/mcp-profile-sketch.md) | The design notes behind pulled statements and the MCP profile, both now in the drafts |

## Related Specifications

Dependencies:

* [OAuth Client ID Metadata Document](https://datatracker.ietf.org/doc/draft-ietf-oauth-client-id-metadata-document/), for every draft
* [Token Status List](https://datatracker.ietf.org/doc/draft-ietf-oauth-status-list/), for withdrawing a statement before it expires
* [Identity Assertion Authorization Grant](https://datatracker.ietf.org/doc/draft-ietf-oauth-identity-assertion-authz-grant/), for the `aud_tenant` claim
* [Deferred Token Response](https://datatracker.ietf.org/doc/draft-ietf-oauth-deferred-token-response/), for asynchronous issuance only
* [OpenID Shared Signals Framework](https://openid.net/specs/openid-sharedsignals-framework-1_0.html), for the signals draft only

Related work:

* [OAuth 2.0 Attestation-Based Client Authentication](https://datatracker.ietf.org/doc/draft-ietf-oauth-attestation-based-client-auth/), which attests a running client instance and composes with a statement
* [OAuth 2.0 Client Instance Assertion](https://datatracker.ietf.org/doc/draft-mcguinness-oauth-client-instance-assertion/), which identifies instances within one client
* [OpenID Federation 1.0](https://openid.net/specs/openid-federation-1_0.html), sketched above
* [OAuth Identity Assertion Trust Framework](https://datatracker.ietf.org/doc/draft-mcguinness-oauth-id-assertion-framework/), which generalizes issuer trust

## Contributing

See the [guidelines for contributions](CONTRIBUTING.md).

Contributions can be made by creating pull requests. The GitHub interface supports creating pull requests using the Edit (✏) button.

## Command Line Usage

Formatted text and HTML versions of the drafts can be built using `make`.

```sh
$ make
```

Command line usage requires that you have the necessary software installed. See [the instructions](https://github.com/martinthomson/i-d-template/blob/main/doc/SETUP.md).
