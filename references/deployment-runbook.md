# Deployment and repair runbook

This runbook is intentionally domain-neutral. Substitute only values confirmed by the operator. Preserve existing user-owned configuration unless the requested change requires it.

## Preflight and network choice

From every relevant VPS, verify each domain locally and with independent resolution. Before editing, inspect the actual deployed services and listeners:

```bash
getent ahosts <host-domain>
dig +short A <host-domain>
dig +short AAAA <host-domain>
ss -lntup
docker ps --format 'table {{.Names}}\t{{.Ports}}\t{{.Networks}}'
docker inspect <openresty-container> --format '{{.HostConfig.NetworkMode}}'
nginx -T
```

Check cloud security groups and host firewalls for TCP 80/443, UDP `<hy2-port>`, and TCP `<reality-port>`. Do not publicly publish 3x-ui panel or subscription upstream ports.

| OpenResty placement | HTTP/WS and panel upstream | HY2 forwarding |
| --- | --- | --- |
| Can reach host loopback | Publish relevant 3x-ui upstream ports on `127.0.0.1` only. | Nginx stream listens publicly and forwards to a loopback-only UDP Xray port. |
| Docker `host` network | Use a dedicated Docker bridge with a validated static 3x-ui address; proxy there. | Nginx stream listens publicly and proxies to the same private address. Do not publish a conflicting Docker UDP port. |

When a static address is required, validate no routing/Docker subnet overlap first. Example only:

```yaml
networks:
  xui_private:
    driver: bridge
    ipam:
      config:
        - subnet: 172.30.0.0/24
services:
  3x-ui:
    networks:
      xui_private:
        ipv4_address: 172.30.0.2
```

Reality is an explicit exception to loopback-only exposure: it needs a dedicated externally reachable TCP listener. Default it to `10443/TCP` and disclose that exception.

## Docker, certificates, and visible website configuration

Use an operator-approved 3x-ui image, persistent data, and durable certificate mounts. Typical bindings, only after confirming image ports and existing listeners:

```yaml
ports:
  - "127.0.0.1:2053:2053/tcp"  # 3x-ui panel
  - "127.0.0.1:10882:10882/tcp" # subscription service, if used
  - "10443:10443/tcp"           # Reality exception
```

Ensure certificate renewal updates the mounted file and reloads/restarts the component that consumes it.

In 1Panel, create a visible **Website** row for each Master panel, subscription, and Node panel hostname; attach a valid certificate there; add an explicit proxy entry/configuration in that same website. Do not write directly to the 1Panel database. A static site row is not enough if the active virtual host does not include its proxy fragment.

```nginx
location /<panel-base-path>/ {
    proxy_pass http://<private-upstream>:2053;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto https;
}

location /<ws-path> {
    proxy_pass http://<vmess-private-upstream>:<vmess-port>;
    proxy_http_version 1.1;
    proxy_set_header Host $host;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
}
```

For HY2, ensure the active Nginx main configuration includes a `stream` block, then create an explicit managed stream config:

```nginx
server {
    listen <hy2-public-port> udp reuseport;
    proxy_pass <xray-private-ip>:<hy2-xray-port>;
    proxy_timeout 300s;
}
```

Never use `proxy_responses 0` for HY2. It discards upstream UDP responses and causes client timeouts. Confirm `nginx -T`, run `nginx -t`, reload successfully, and test the precise public panel base-path/WS/HY2 listener. A saved setting or a reload command alone is not proof that Nginx is actually serving it.

### HY2 public-port migration

Treat a public HY2 port change as a cross-layer change, not a single Nginx edit. Before changing it, record `ss -lntup`, Docker port bindings, the active stream file, firewall/security-group rules, and the Master Managed Host row. If the container publishes its internal 443/TCP and 443/UDP through one loopback host port, that port may also be the VMess WebSocket upstream. Move both Docker mappings to a dedicated loopback port (for example `127.0.0.1:18443:443/tcp` and `/udp`), set the stream `listen` to the new public UDP port and `proxy_pass` to that loopback port, and update every HTTP VMess `proxy_pass` that used the old loopback port. Keep the loopback port private.

Then, in order:

1. Add the new UDP firewall/security-group rule and verify no host-port collision.
2. Recreate the container with the new loopback mapping and verify the TCP and UDP sockets.
3. Update and reload the Nginx stream and the TLS/WS virtual host; search all active and panel-generated fragments for the old target.
4. Add or update the exact Master Managed Host for the HY2 inbound, including public port, domain/SNI, ALPN, node scope, and no duplicate row.
5. Regenerate and parse every affected subscription. Check both `main` and any additional clients; the HY2 port must be the new public port and VMess must still be `443/TCP` with the correct WS path.
6. From an independent VPS, run a real HY2 handshake and a real Mihomo VMess probe. A UDP packet reaching the firewall or an HTTP 400 from a non-WebSocket curl is only a reachability hint, not protocol acceptance.

## Default inbounds and Master integration

Create three entries on the Master and every Node, named with the user-approved prefix:

| Protocol | Public listener | Required settings |
| --- | --- | --- |
| VMess + TLS + WS | 443/TCP at Nginx | Nginx terminates TLS; client SNI and WS Host are the node domain; use a unique WS path. |
| Hysteria2 | 8443/UDP by default via stream | Valid certificate for node domain, SNI equals that domain, and Xray can read renewed certificate files. |
| VLESS + Reality + Vision | 10443/TCP by default | Raw Reality, Vision flow, valid Reality keypair/short ID, and an approved decoy target/SNI/fingerprint. |

The Master owns one canonical client named `main`. Node VMess and VLESS client UUIDs must equal `main.uuid`; each Hysteria client `auth` must equal `main.auth` (not the UUID). Save Node API tokens once securely, register only after public Node panel paths work, probe the node, and confirm expected inbound sync.

After sync, create/update Master Managed Host records for the public topology:

- VMess: `443`, `tls: true`, SNI, Host header, and WebSocket path match its node.
- HY2: public UDP port, not internal inbound port; TLS/SNI match its node; expected ALPN.
- Reality: external Reality port; public key, short ID, server name, and Vision flow intact.

Enumerate records before cleanup. Delete only the exact confirmed duplicate or wrong node/inbound binding.

### Additional clients and Reality Vision flow

For a client other than `main`, the VLESS Reality entries must carry
`flow: xtls-rprx-vision`. Set the client-level flow (or an explicit
per-inbound flow override) and leave VMess/HY2 client entries without a flow.
Do not assume that a successful client update means the rest of the client
state survived.

The 3x-ui client update endpoint can accept a partial body but may rebuild the
client record with omitted fields at zero/false values. On a Master/Node
deployment this can also remove the current remote-inbound relation; if the
client is disabled, Node reconciliation filters it and the Node may revert to
`main` only. Use this sequence:

1. Fetch the complete client record and record its existing UUID/auth,
   `subId`, quota, `enable` state, and intended `inboundIds`.
2. Submit a complete normalized update body, changing only `flow` to
   `xtls-rprx-vision` for the Reality-capable client. Preserve `enable: true`,
   quota, credentials, and subscription identity; do not send a flow on
   VMess/HY2-only records.
3. Re-read the client list and verify the intended inbound IDs are still
   attached. If remote IDs disappeared, use the official client attach or
   inbound update API to restore them, then push/reconcile the affected Nodes.
   Verify the Node inbound settings contain the client and that it remains
   enabled after the next sync cycle.
4. Fetch each affected Clash subscription, parse the YAML, and assert that
   every VLESS Reality proxy for the client has `flow:
   xtls-rprx-vision`; also assert that the expected operator-selected protocol
   set is still present.

Never repair this by editing the SQLite database directly. If an API update
returns success but the generated subscription loses nodes, treat the update
as incomplete until Master bindings, Node settings, and the subscription all
agree.

## Clash template and acceptance

The included `assets/sample.yaml` is the baseline. Before placing it in 3x-ui global Clash rules, remove its entire `proxies:` block because 3x-ui creates dynamic proxies. In versions requiring full settings validation, first fetch all settings, replace only the Clash fields, submit the complete object, then restart/reload 3x-ui if necessary.

Parse the newly generated YAML before testing traffic. From an independent network and with a current Mihomo/Clash-compatible client, test every entry through an external HTTP endpoint. For every VMess check TLS/SNI/WS path/Host; for HY2 check public UDP port/certificate/SNI and replies; for Reality check port/public key/short ID/Vision flow. Confirm expected names, groups, rules, proxy count, certificates, and public-access policy.

| Symptom | Check and correction |
| --- | --- |
| VMess timeout while another works | Compare generated TLS/SNI/Host/path/port to its exact Managed Host and Nginx site; fix `tls: false` when public 443 terminates TLS. |
| All HY2 entries timeout | Inspect active stream config, UDP socket ownership, Docker/OpenResty binding conflicts, firewall, and packet flow; remove `proxy_responses 0`. |
| HY2 works after a port change but VMess times out | Search active Nginx and panel-managed site fragments for stale `127.0.0.1:<old-port>` VMess targets. Update both the TLS WebSocket locations and the visible website proxy entry to the new private TCP loopback port, then test with Mihomo. |
| New HY2 port is not reflected in subscription | Enumerate Managed Hosts before editing. Add or update the exact HY2 row with the public UDP port, node scope, domain/SNI, and ALPN; then regenerate and parse each client subscription. |
| Additional client's Reality link lacks `flow` | Fetch the complete client state, set `flow` to `xtls-rprx-vision`, preserve `enable`/quota/UUID/auth/subId, restore any lost inbound bindings through the official API, reconcile Nodes, and re-parse the client's Clash YAML. |
| Client update succeeds but remote Nodes show only `main` | Check whether omitted update fields disabled the client or removed its `inboundIds`; restore the full client state and bindings, then verify the next Node reconciliation. |
| Public panel path returns 404 | Inspect `nginx -T`; ensure the visible site contains/inherits the proxy config; test config, reload/restart, then test the exact path. |
| Node sync works but subscription is wrong | Inspect Managed Host rows and generated YAML; correct wrong host/node/inbound binding and regenerate. |
| It breaks after container recreate | Replace ephemeral container-IP targets with a validated static private network address or atomically update/retest target. |

Complete only after every generated proxy connects and carries traffic. Hand over the fully qualified panel URLs including base paths and authorized credentials. Remind the operator to immediately rotate temporary SSH and 1Panel web-login passwords and put new panel/API secrets in a password manager.
