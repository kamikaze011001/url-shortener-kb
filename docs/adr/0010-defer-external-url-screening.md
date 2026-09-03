---
status: accepted
date: 2026-09-03
---

# Local SSRF and open-redirect guards now; external reputation screening deferred

Destination Screening is an interface with exactly one implementation today:

```java
public interface DestinationScreener {
    ScreeningResult screen(URI destination);
}
```

`LocalRulesScreener` enforces three rules with no external dependency: the scheme must
be `http` or `https`; the destination must not resolve to a **Private Destination**
(`10/8`, `172.16/12`, `192.168/16`, `127/8`, `169.254/16`, IPv6 unique-local and
loopback); and it must not point at this service's own hostnames, which would create a
redirect loop.

**External reputation screening — Google Safe Browsing or equivalent — is deliberately
not built**, and the interface above is where it plugs in.

## Why the local rules are built and the external check is not

They are different kinds of problem. The private-address rule closes a hole *this
service would otherwise open*: without it, anyone can shorten
`http://192.168.1.1/admin` or `http://169.254.169.254/latest/meta-data/` and use the
redirect as a probe into the network the server sits in — which, for a service running
on a home machine behind a tunnel, is a live concern rather than a theoretical one. It
is roughly thirty lines, has no runtime dependency, and can never be unavailable.

An external reputation check is a different shape: it is a network call on the creation
path, and it forces a question the demo cannot answer well. **When Safe Browsing times
out, do we fail open or fail closed?** Fail open and a malicious Destination is
accepted. Fail closed and an outage at Google stops all Link creation. The honest
answer is neither — it is fail open, queue for re-screening, and retroactively disable
the Link if it comes back bad — and that requires a background job system this project
does not have.

Deferring it is therefore the correct engineering call and *the discussion is worth
more here than the implementation would be in the demo*.

## Consequences

- A Visitor can be redirected to a phishing page. Stated plainly: this service does not
  screen for malicious content today. Mitigating factors are that registration is
  effectively private and rate limits cap the blast radius — neither is a defence.
- Private-address checking must happen **after DNS resolution, not on the string**, or
  a hostname resolving to a private address walks straight through. This is the way the
  rule is usually implemented wrongly.
- There remains a DNS-rebinding gap: a hostname can resolve to a public address at
  creation and a private one at redirect time. Closing it means re-resolving on every
  redirect, which is unacceptable on a 20 ms path. Documented, not fixed.
- The interface is the whole point of the deferral. [R-2](../06-roadmap.md) is a second
  implementation and a configuration flag, not a change to the creation flow.
