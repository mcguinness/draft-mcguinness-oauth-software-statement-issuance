# OAuth 2.0 Software Statement Issuance

This is the working area for the individual Internet-Draft, "OAuth 2.0 Software Statement Issuance".

RFC 7591 standardizes how a client presents a software statement and how a registration endpoint consumes it, but not how the client obtains one. This specification defines the missing issuance protocol: a redirect flow returning a `software_statement_code` redeemed at the token endpoint, a token exchange profile for pre-authorized issuance and renewal, and deferred processing under Deferred Token Response so approval can complete out of band over hours or days. Clients are identified by a Client ID Metadata Document, and every statement is bound by canonical digest to the exact metadata reviewed.

* [Editor's Copy (HTML)](https://mcguinness.github.io/draft-mcguinness-oauth-software-statement-issuance/draft-mcguinness-oauth-software-statement-issuance.html)
* [Editor's Copy (TXT)](https://mcguinness.github.io/draft-mcguinness-oauth-software-statement-issuance/draft-mcguinness-oauth-software-statement-issuance.txt)
* [Datatracker Page](https://datatracker.ietf.org/doc/draft-mcguinness-oauth-software-statement-issuance/) (after first submission)

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
