---
status: accepted
date: 2026-09-04
---

# The application runs in a container; its state does not

The backend runs as a container. Postgres and Redis are installed on the host. The
frontend is static files served by the host's Caddy, in no container at all.

Three different answers to "should this be containerised", and the reasons differ.

## The application: yes

The first deployment ran the jar directly under systemd, and it worked. What it lacked
was worth going back for:

- **Rollback is a rebuild.** With a bare jar, going back a version means checking out a
  tag and rebuilding. With images, the previous tag is still on the host.
- **The runtime is a host dependency.** The unit pinned
  `/usr/lib/jvm/java-21-openjdk-amd64/bin/java` by absolute path, because this host also
  has Java 17 and the default is not ours to change. An image carries its own runtime and
  that whole sentence stops being true.

Images are tagged twice: `:current`, which moves, and `:<jar-checksum>`, which does not.
A rollback needs something immutable to name.

## The stateful services: no

This was an instruction rather than a deduction, and it is the right one. What the
decision actually cost is where the interesting part is — see the networking section.

Isolation is per-application rather than per-container:

- **Postgres** — a role with no `SUPERUSER`, `CREATEDB` or `CREATEROLE`; a database with
  `CONNECT` revoked from `PUBLIC`; and a named schema, because `public` is writable by
  every role in a stock cluster.
- **Redis** — a separate `redis-server` process on 6380 with its own memory ceiling and
  eviction policy, not a `SELECT` index on a shared one. An index is a namespace, not a
  boundary: indexes share a memory ceiling, an eviction policy and a lifetime, so a
  neighbour can evict this application's keys and one flush takes out every tenant.

## The frontend: no, and not for the same reason

The built SPA is 372 KB of static files. Containerising it means running a web server
*inside a container* to serve them, when Caddy is already on the host serving both
hostnames. That is an image, a container and a process bought for nothing.

The general rule this is an instance of: **containerise processes, not files.**

**That answer is incomplete, and the gap is accepted rather than unnoticed.** The reason
given above for containerising the backend was that rollback would otherwise be a
rebuild — and the same test, applied to the frontend, fails. Its deploy deletes the
directory and copies a new one in: not versioned, so a rollback *is* a rebuild, and not
atomic, so there is a brief window where Caddy serves a half-copied directory.

A container is not the only fix and is the more expensive one, because Caddy has to stay
either way — it terminates the tunnel hop and splits `/api/*` from the SPA, so a frontend
container becomes an upstream for it rather than a replacement, adding a process and a
network hop to serve 372 KB. The cheaper fix is versioned release directories with an
atomic symlink swap: `releases/frontend-<hash>/` immutable, `frontend` a symlink, deploy
by re-pointing it, roll back by pointing it back.

Deferred deliberately for a demo where the frontend changes rarely and a bad deploy is
repaired by re-running the build. It is a real gap for anything with users, and
[06-roadmap.md](../06-roadmap.md) carries it as R-14.

## Networking, which is where the real trade-off lives

The application must reach Postgres on `127.0.0.1:5432` and Redis on `127.0.0.1:6380`.
A container cannot reach the host's loopback. Three options, and the middle one is the
trap:

**`host.docker.internal`.** On Linux this needs
`--add-host=host.docker.internal:host-gateway`, and it resolves to `172.17.0.1` — the
**default bridge gateway**. Verified on the host: the name resolves, and every port is
refused, because the databases are bound to loopback. Making it work means binding them
to that gateway, where **every container on the machine can reach them**. This host runs
a GitHub Actions runner; anything a CI job starts lands on that bridge. The option that
looks like it avoids the trade-off *is* the trade-off.

**A dedicated docker network.** A user-defined bridge gets its own subnet and gateway,
so binding the databases there limits them to containers on that one network. Genuinely
better than the default bridge, and it keeps the application's network namespace. It
costs `listen_addresses`, a `pg_hba.conf` rule, a pinned subnet, and a database
accepting TCP on a non-loopback interface. Reasonable; not chosen.

**`--network host`.** Chosen. The databases do not move at all — they stay loopback-only
and their configuration is untouched, which is the strongest form of the isolation that
was asked for. What is given up is the container's network namespace.

That loss is smaller than it looks, because the application binds `127.0.0.1` itself and
the namespace was not what protected it. But it makes one setting load-bearing:

## The consequence that was learned the hard way

`SERVER_ADDRESS=127.0.0.1` lived in the jar unit's `Environment=` line. The container
unit replaced that file, and the setting was not carried across. With host networking the
application then bound `*:8090` — reachable from the LAN, bypassing Caddy, into a process
that trusts `CF-Connecting-IP` **without verifying it** because
[ADR-0006](./0006-two-hostname-topology.md)'s topology says nothing else can reach it.
Every per-IP limit in FR-6 was decoration for anyone on that network for about an hour.
A firewall rule was the only thing in the way, and a firewall rule is not the guarantee
the design claims.

It now sits in the unit as `--env`, immediately below the `--network host` flag that
makes it necessary, with a comment saying what happens without it. Not in the
environment file: it is a property of *this deployment shape*, not a secret, and it
belongs beside its reason rather than in a list where the next rewrite can miss it again.

**The general lesson is about migrations, not about Docker.** Moving a service between
runtimes moves its configuration between formats, and a setting whose absence is silent
— no error, no failed start, just a wider bind — is the one that survives the move
unnoticed. The check for it is `ss -tln`, before and after, every time.

## Consequences

- **`docker` becomes a dependency of the deploy.** It was already installed here.
- **Two environment-file formats.** systemd's `EnvironmentFile` strips surrounding
  quotes from a value; docker's `--env-file` keeps them. A password quoted in an editor
  works under one and fails SMTP authentication under the other, as a mail problem
  nowhere near the change that caused it. `containerize.sh` refuses to cut over when it
  finds quoted values.
- **The image is not reproducible from source alone.** The jar is copied in rather than
  built inside, because Java bytecode is architecture-independent and cross-building
  under emulation buys nothing. It reproduces from a git tag plus `./gradlew bootJar`.
- **Logs are unchanged.** The journald log driver keeps `journalctl -u url-shortener`
  working exactly as it did with the jar.
- **The jar unit is kept, disabled, beside the container one**, and the cutover script
  restores it automatically if the container does not come up healthy. A cutover that
  leaves a service down while somebody reads an error message is worse than one that
  did not happen.
