# Field pitfalls and incident ledger

Real failures observed while operating this cluster, each with symptom, root cause,
fix, and the check that proves the fix. Read this before touching production: every
entry here cost real downtime, and several were initially misdiagnosed in a way that
is easy to repeat.

Use placeholders (`<node-domain>`, `<panel-ssl-dir>`, `<node-guid>`) — never paste real
hostnames, UUIDs, subscription IDs, or keys into this file.

## 1. Certificates

### 1.1 A hand-copied certificate silently expires while 1Panel keeps renewing its own

Symptom: HY2 fails with a client-side x509 error while VMess and Reality on the same
host keep working. `openssl x509 -dates` on the file the inbound points at shows an
expiry a few days in the past; the panel's certificate for the same domain is valid for
months.

Root cause: the inbound was configured against a one-off copy (for example
`<data-dir>/cert/<domain>.fullchain.pem`) made once at deploy time, while the managed
certificate lives under the panel's per-site directory and is renewed automatically.
Nothing links the two.

Fix: mount the panel's certificate directory read-only into the container and point the
inbound at the mount, so renewal is the only thing that matters:

```yaml
volumes:
  - <panel-ssl-dir>:/root/certpanel:ro
```

```text
certificateFile: /root/certpanel/fullchain.pem
keyFile:         /root/certpanel/privkey.pem
```

Then keep the two archives on disk consistent with the inbound. Rename stale copies to
`*.expired-<date>` instead of deleting them, so a rollback stays possible.

### 1.2 Xray does not hot-reload certificates

Symptom: the file on disk is renewed and valid, but the running process still serves the
old certificate.

Root cause: the certificate is read at config load; renewal alone changes nothing.

Fix: a small guard script plus cron that restarts the container only when the
certificate is newer than the container start time. Restarting unconditionally on a
schedule drops live connections for no reason:

```bash
if [ "$(stat -c %Y /root/certpanel/fullchain.pem)" -gt \
     "$(date -d "$(docker inspect -f '{{.State.StartedAt}}' <container>)" +%s)" ]; then
  docker restart <container>
fi
```

Verify: run the script once and expect `no-op` immediately after a restart, then confirm
the in-container `openssl x509 -dates` matches the host file.

## 2. Subscription generation (the most expensive class of bug)

### 2.1 A new inbound needs three separate records, and each omission has a different symptom

This is the single most costly pitfall in this repo. Inserting an inbound directly into
`x-ui.db` without completing all three produces a node that either never appears in the
subscription or appears and rejects every client — with **no error in panel or logs**.

| Missing piece | Symptom | Visible anywhere? |
|---|---|---|
| `inbounds.settings.clients` | Node never appears in the subscription | No |
| `client_inbounds` rows | Node appears in the subscription, every handshake fails instantly | Only on the client |
| `hosts` row (Master-local inbounds only) | Node never appears in the subscription | No |

Because the first and third share one symptom, do not conclude "the generator does not
support this combination" — check the data first.

Checklist for a hand-created inbound:

1. `settings.clients` mirrors the clients you expect in the link (clear `flow` for
   non-Vision transports such as XHTTP; `xtls-rprx-vision` there breaks the handshake).
2. `client_inbounds` binds every intended client; this is what fills `"clients"` in the
   generated `config.json`. Confirm with
   `docker exec <container> sh -c 'python3 -c "import json;print(len(json.load(open(\"/app/bin/config.json\"))[\"inbounds\"][0][\"settings\"][\"clients\"]))"'`,
   or simply grep the generated config.
3. For Master-local inbounds, insert the `hosts` row **and** keep its `path`, `alpn`,
   `security`, `sni`, `address`, `port` consistent with the inbound. Node-owned inbounds
   can be published without one.

### 2.2 `inbounds.tag` is globally unique

Creating the same tag on a second host (for example `in-8444-udp` on two nodes) fails
with `UNIQUE constraint failed: inbounds.tag`. Suffix tags per host
(`in-8444-udp-<node>`) when mirroring an inbound across nodes.

### 2.3 `enable = 0` does not remove an inbound from the subscription

Symptom: an inbound is disabled, stops listening, yet still appears in the generated
subscription and clients keep selecting a dead node.

Fix: delete the inbound (and its `client_inbounds` rows) when it must disappear from the
subscription. Disabling only stops the listener.

### 2.4 The Master re-adopts node state, so deletion order matters

Symptom: an inbound deleted on the Master comes back as a new row (new id) after a
restart; or an inbound deleted on a node keeps being served by the subscription.

Root cause: the Master reconciles its records from the owning node. Deleting only one
side is not durable.

Order that works:

1. Delete on the **node** first (stop container, delete rows, start).
2. Then delete the leftover rows on the **Master**.
3. `docker restart` the Master afterwards. Its inbound list is partly cached in memory;
   without the restart the subscription can still contain a node that no longer exists
   anywhere.

Verify by parsing the generated subscription and counting proxies, not by looking at the
panel.

### 2.5 hysteria2 obfs and port-hopping are not distributable through the subscription

Confirmed on 3.7.0 and 3.11.0: neither `salamander` obfs nor hop `ports` reaches the
generated Clash/Mihomo proxy, no matter where the value is stored. Attempts that all
failed:

- `hysteriaSettings.obfs` + `obfs_password`, plus the `obfsPassword` and `obfs-password`
  spellings
- `hosts.final_mask` with the same JSON used by `streamSettings.finalmask`

Consequence: server-side support works (a hand-built client connects), but every device
would need a manual client override, and a subscription refresh silently reverts it. Do
not ship these as if they were subscription features; if they are required, plan a
client-side override or a panel upgrade and re-test the generator first.

### 2.6 Two subscription endpoints, two client families

`/<clash|sub>/<subId>` style routes differ per install. Verify both before handing a URL
to a user: the Clash route returns YAML for mihomo-family clients, the generic route
returns base64 URI list (`vless://`, `hysteria2://`) for v2rayNG-class clients. Fetching
one and assuming the other works is a common source of "the subscription is broken".

## 3. Protocol and version compatibility

### 3.1 Reality on Xray ≥ 26.9 requires the X25519MLKEM768 key share

Symptom: `REALITY authentication failed` on every client while the same inbound worked
before the server upgrade. Every TLS fingerprint (chrome/firefox/safari/random/none)
fails identically.

Root cause: the server now requires the post-quantum hybrid key share in the ClientHello.
Clients that do not send it are rejected at authentication, not with a version error.

Fix: on mihomo, set it explicitly:

```yaml
reality-opts:
  public-key: <pbk>
  short-id: <sid>
  support-x25519mlkem768: true
```

3.11's generator emits this automatically for Reality proxies; older generators do not,
and a subscription that predates the upgrade will keep failing until regenerated.
sing-box-based clients do not send the share and will fail against these servers — route
those devices to a node still running an older Xray.

### 3.2 Reality field name drifted: `target` vs `dest`

Inbounds created by 3.6.0 store `realitySettings.target`; newer releases use `dest`.
Scripts that read `rs["dest"]` crash with `KeyError` on older rows. Always read
`rs.get("dest") or rs.get("target")` when cloning an existing Reality inbound.

### 3.3 XHTTP-H3 and XHTTP-Reality are not the same axis

- `network: xhttp` + `security: tls` + `alpn: ["h3"]` → QUIC listener (**UDP only**,
  requires a real certificate). Without `h3` in ALPN it silently becomes a TCP
  HTTP/2 listener on the same port, which is a different transport.
- `network: xhttp` + `security: reality` → TCP HTTP/2 (no certificate, borrows a decoy
  handshake).
- REALITY over QUIC does not exist; the two cannot be combined.
- Publishing the port is part of the contract: an H3 inbound needs a `udp` port mapping,
  a Reality inbound a `tcp` one. A correct inbound with no published port fails exactly
  like a broken protocol.

### 3.4 Do not infer a version incompatibility without isolating it

Three separate failures in one session were blamed on "too old Xray" / "too old panel";
all three were missing client bindings (§2.1). Before concluding a version problem,
prove the server side independently — for XHTTP, a real HTTP/3 probe against the inbound
path should answer `400` (handler reached, session parameters missing) while a wrong path
answers `404`. If both differ, the server is fine and the fault is in the client
configuration or the generated link.

## 4. Docker and host operations

### 4.1 A digest-pinned image never upgrades

`image: ghcr.io/<owner>/<repo>@sha256:...` makes `docker compose up -d` a no-op forever.
Change the reference to a tag (or the new digest) before expecting an upgrade, then
verify with `docker exec <container> /app/x-ui -v` and
`docker exec <container> bin/xray-linux-amd64 version`.

### 4.2 Cascading backups can fill the disk and take the panel down

Copying `x-ui.db*` to `.bak.<timestamp>` inside the same directory is fine once. Looping
it copies the previous backups too: one cluster accumulated 8241 files / 11 GB on a 24 GB
disk, after which containers failed to start and the subscription returned 5xx.

Rules:

- Write backups to a dedicated directory: `/root/backup-3xui-<timestamp>/`.
- Never glob `x-ui.db*` into a path that already contains backups.
- If it happens: `find <dir> -maxdepth 1 -name "*.bak.*" -delete`, then check
  `df -h` and `PRAGMA integrity_check` before restarting anything.

### 4.3 Low-memory hosts drop SSH during container recreation

On a 1.5 GB host running a dozen containers, `docker compose up -d` can push load high
enough that the SSH session dies mid-command, leaving the container stopped and the
database edit unapplied. Split the operation into small steps (start container → edit DB
→ start container), and use `ssh -o ServerAliveInterval=10`.

### 4.4 When the local proxy core dies, every `ssh` looks like a server outage

`ssh: Connection closed by remote host` while the host is perfectly healthy: the client's
tunnel core was not running, so TCP to the VPS went into a TUN interface with no data
path. Confirm the local side before debugging the server:

```bash
pgrep -fl mihomo            # core process present?
lsof -nP -iTCP -sTCP:LISTEN | grep -E '7890|9090'
curl -s -m 8 -o /dev/null -w '%{http_code}\n' https://www.gstatic.com/generate_204
```

Workaround that keeps work moving while the GUI is down: bind SSH to the physical
interface, `ssh -o BindAddress=$(ipconfig getifaddr en1) <host>` (macOS; use
`ip -4 addr` on Linux).

## 5. Diagnosis discipline

- Separate "the protocol is broken" from "the IP is blocked". If TCP inbounds all time
  out while a UDP inbound on the same host answers in milliseconds, the box is fine and
  the IP's TCP path is blackholed; adding or editing TCP inbounds there is wasted work.
- A handshake error (x509, authentication) proves packets are flowing. Silence proves
  nothing — check the port mapping and firewall before the protocol.
- After any inbound change: regenerate the subscription, parse the YAML, then probe
  **every** proxy end to end. A panel that renders correctly is not evidence that the
  generated link is connectable.
- Keep the acceptance script idempotent and table-shaped (node → delay → data result). It
  is the only fast way to tell a single-node regression from a fleet-wide one.
