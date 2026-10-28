# Security Hardening — A Case Study

How I took my homelab from "most of the stack reachable over the internet" to "one TLS
port exposed, everything else private." Written before → after, with how I verified each
change.

> Anonymised throughout. Values like `home.example`, `192.168.1.0/24` and `2001:db8::/64`
> are placeholders.

---

## The problem

I run a single-host Docker homelab. It was designed behind IPv4 NAT, where `-p 8080:8080`
effectively keeps a service on the LAN — nothing forwards it, so nothing outside can reach
it. On that assumption, publishing ports freely was fine.

Then I moved house. The new ISP box handed my server a **public, globally-routable IPv6
address** — and IPv6 has no NAT. Two things I'd relied on quietly stopped being true:

1. Docker's `-p 8080:8080` binds **both** `0.0.0.0` *and* `[::]`.
2. Docker doesn't enable NAT for IPv6 by default, so each published port was being served
   over IPv6 by a userland `docker-proxy` process on `[::]` — directly on the host's new
   public IPv6 address.

So a setup that was safe behind IPv4 NAT became, overnight, a stack of services on the open
internet: databases, an MQTT broker, a container-management UI, a personal-finance app and
the full media stack. I caught this while bringing the homelab back up after the move, and
redesigned the host so the exposure couldn't come back.

I confirmed the exposure was real before changing anything: from my phone on mobile data
(off my network, IPv6), several of those services loaded.

---

## What changed

### 1. Host firewall — from none to default-deny

**Before:** no host firewall at all (no `nftables`/`iptables` rules, no `ufw`/`firewalld`).
The only packet rules present were Docker's own.

**After:** a dedicated `nftables` table, kept separate from Docker's tables so the two never
conflict:

- `policy drop` on `input`
- allow loopback, established/related, ICMP/ICMPv6 (so the IPv6 stack stays healthy)
- allow SSH and the LAN subnet
- **drop all new inbound IPv6** — I leave the stack enabled in the kernel so a package
  update or reboot can't silently re-expose it; it's simply firewalled off at the edge
- rate-limited logging of drops

Persisted with a `systemd` unit that reloads the ruleset at boot, ordered after Docker so
they never race. See [`../firewall/fw_home.nft`](../firewall/fw_home.nft).

**Verified:** `nft list ruleset` showed the active table; from the off-network phone,
previously-open services now time out; SSH and LAN access confirmed still working.

### 2. Docker port exposure — from ~40 published ports to one

**Before:** on the order of ~40 `docker-proxy` listeners on `0.0.0.0`/`[::]`.

**After:** I re-architected every service to publish **no host port**. Services join a
shared internal Docker network and are reached by container name through the reverse-proxy.
Data services (PostgreSQL, MQTT, InfluxDB, Redis) are internal-only to their stack.

**Verified:** `ss -tlnp` and `docker ps` showed `docker-proxy` listeners drop from ~40 to
**0**; the only container publishing a host port is the reverse-proxy on `443`.

### 3. Reverse proxy — a single ingress

Caddy is now the single public listener (`443`, HTTP/3). It terminates TLS with wildcard
certificates issued via **DNS-01** (no inbound ACME needed), reaches every backend by
**container name** over the private network, and **gates admin UIs to LAN + WireGuard**
source ranges, returning nothing to anyone else. Native services that aren't containers —
Plex, for instance — are only reachable from outside *through* Caddy on `443`, never on
their own port. See [`../edge/Caddyfile`](../edge/Caddyfile).

**Verified:** each migrated hostname returned its real app response through Caddy, internal
app-to-app links (indexer → \*arr, telemetry → metrics DB) confirmed by container-to-container
tests, and a full proxy **recreate** (not just a restart) confirmed the config is durable.

### 4. SSH — already strong, confirmed

Keys-only (`PasswordAuthentication no`), root login disabled, a non-standard port, and
`fail2ban` active. I left it as-is after verifying it; the firewall keeps it reachable so
remote access can't be locked out.

### 5. Infrastructure-as-code + removing the GUI

**Before:** stacks were managed through a web GUI, with compose definitions stored in its
root-owned data volume.

**After:** I lifted every stack into a plain, version-controllable `docker-compose.yml` in a
user-owned directory, with secrets in `chmod 600` env files kept out of the compose. I then
removed the management GUI (keeping its data volume). The host is now managed with
`docker compose` + SSH + WireGuard — one less admin service exposed.

**Verified:** all migrated stacks healthy; data preserved by reusing the original named
volumes; GUI container removed and its proxied route dropped.

---

## Before → after

| | Before | After |
|---|---|---|
| Host firewall | none | nftables default-deny, IPv6 blocked, systemd-persistent |
| Internet-facing ports | ~40 (over public IPv6) | 1 (`443`, TLS) |
| DB / MQTT exposure | reachable from internet | internal-only |
| Service access | direct `host:port` | reverse-proxy by container name |
| Admin UIs | open on LAN + internet | LAN + WireGuard only, behind TLS |
| Stack management | web GUI (root-owned volume) | compose files + SSH + WireGuard |

## Still pending (roadmap)

- Single sign-on / auth gateway (e.g. Authelia/Authentik) in front of admin UIs.
- Verify backups are immutable / offline (ransomware resistance).
- Pin container image tags instead of `:latest`.
- Optional: filter inbound IPv6 at the router too, for defence in depth.

## Takeaways

- `docker -p` is not "LAN only." The moment the host has a public IPv6 and no firewall, it's
  an internet publish. A network change (new ISP box, new prefix) can turn a safe setup into
  an exposed one — so I now bind deliberately, or don't publish at all and proxy by name.
- A host firewall belongs in front of Docker regardless — as the backstop that catches the
  next mistake, not the primary control.
- Reusing named volumes made even stateful migrations (databases) safe and reversible.
