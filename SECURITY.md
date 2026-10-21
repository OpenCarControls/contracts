# Security policy

This project is experimental (see the warning in the [README](README.md)). The contracts define what can be sent to a vehicle controller, so security reports are very welcome.

## Reporting a vulnerability

Please **do not open a public issue** for security problems. Report them privately through GitHub: open the **Security** tab of this repository and choose **Report a vulnerability**.

Include what you found, how to reproduce it, and what an attacker could do with it. You will get an acknowledgement as soon as the maintainer can respond; this is a hobby project, so please allow some time. Please give us a chance to fix the problem before disclosing it publicly.

## What counts

Because this project controls vehicle hardware, we are especially interested in:

*   Schema or transport changes that make it possible to bypass authentication, pairing or the read-only safeguards in the firmware.
*   Messages that let a malformed payload crash or confuse a consumer.
*   Secrets or credentials committed to the repository.

Only the latest `main` is supported.
