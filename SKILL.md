---
name: master-node-skills
description: Deploy, extend, repair, and validate a Docker Compose 3x-ui Master plus remote-node cluster with 1Panel-visible Nginx reverse proxies and a unified Clash subscription. Use when configuring a new Master/Node VPS, adding nodes, fixing VMess/Hysteria2/Reality connectivity, creating managed-host overrides, or applying the bundled Clash template.
---

# 3x-ui Master / Node cluster

Use this skill for an operator-managed 3x-ui cluster: one Master serves the subscription and holds the canonical client; remote Nodes provide inbounds and are registered to the Master. Treat every domain, IP, API key, token, UUID, subscription ID, credential, private panel path, and certificate key as secret. Do not put real values into scripts, templates, logs, screenshots, or the final summary unless the user explicitly asks for that specific value.

Read [the deployment runbook](references/deployment-runbook.md) before making changes. Use [the Clash template](assets/sample.yaml) when configuring the Master Clash global template.

## Collect the deployment contract first

Before any mutation, ask the user to supply or confirm all of the following. Ask concise grouped questions; do not infer missing domains or reuse credentials without confirmation.

1. Master SSH endpoint and authentication; each Node SSH endpoint and authentication.
2. Master and Node public DNS names: one subscription hostname, one Master panel hostname, and one panel/proxy hostname per Node. Ask the user to confirm that A/AAAA records point directly to the intended VPS and whether a CDN proxy is enabled.
3. For every 1Panel host: panel URL, API key, and web-login user/password. Ask for existing certificate details or permission to issue/manage certificates. If a different Nginx panel is in use, collect its URL, login, API access, and the equivalent website/proxy management method.
4. Whether 3x-ui is a fresh installation or existing service, the desired panel user/password (generate unique credentials if not specified), public panel base path, node display prefix/name, and whether public panel access is intentionally unrestricted or IP-allowlisted.
5. The desired external HY2 UDP port (default `8443`) and Reality TCP port (default `10443`), plus permission to open/check firewall and provider security-group rules.

Do not request a Master API token until the Master is reachable. Create node API tokens only when registering that specific node; they are generally write-only and must be delivered once securely.

## Required architecture

- Create or retain explicit website records in the management panel for every panel and subscription hostname. The reverse-proxy configuration must be visible and editable in the Web/Website UI, not only hidden in an arbitrary Nginx file.
- Keep the 3x-ui web panel and subscription service loopback-only or private-network-only. Nginx terminates HTTPS and proxies the panel path and subscription route.
- The required exception is VLESS Reality: it needs a dedicated TCP listener. Publish the selected Reality port directly, or forward it through a TCP stream proxy; document that exception to the user.
- Default per-host protocol set: VMess + TLS + WebSocket, Hysteria2, and VLESS + Reality + Vision. Use a consistent country/location prefix, but do not call a Node `master` in its display name.
- Keep one Master client named `main`: the Node VMess and VLESS client UUIDs must equal `main.uuid`; the Node Hysteria client `auth` must equal `main.auth`. Preserve its subscription ID.
- For additional clients, set the client-level `flow` to `xtls-rprx-vision` (or an equivalent per-inbound override) so every VLESS Reality link carries Vision; do not add that flow to VMess or HY2 links. Treat a 3x-ui client update as a full-state update: preserve the existing UUID/auth, subscription ID, quota, enabled state, and intended inbound bindings, then verify node sync and the generated subscription.
- Default Reality-Vision inbound configuration: every VLESS Reality inbound must set `streamSettings.realitySettings` with `minClientVer: "1.0.0"`, `maxClientVer: ""`, `maxTimediff: 0`, and a `settings` block containing the node public key plus `fingerprint: "chrome"`, `serverName: ""`, `spiderX: "/"`. Do not leave `minClientVer` at restrictive values such as `"1.8.0"`, and do not omit `settings.fingerprint`. When these fields are missing, the generated Clash/Mihomo Reality proxy loses `client-fingerprint: chrome` or is rejected during the Vision handshake, so Reality can time out even though a native Xray client connects successfully. Always parse the generated Clash YAML and confirm every Reality proxy carries `client-fingerprint: chrome`.

## Execution workflow

1. Verify DNS from each relevant VPS and from an independent resolver/network. Inspect Docker, listening sockets, firewalls, 1Panel/OpenResty configuration, existing certificates, and whether OpenResty uses Docker `host` networking before choosing ports or proxy targets.
2. Create the Docker Compose service and a durable certificate mount. Use the network/port decision table in the runbook; never bind a container and host-network OpenResty to the same host UDP port. Record generated panel URL, username, and password for a secure handoff.
3. Create visible 1Panel website entries and certificates. Add explicit HTTPS WebSocket/panel/subscription proxy rules and, where required, an explicit Nginx `stream` rule for HY2. Validate the active Nginx configuration, reload it successfully, then test the public URL—not merely the saved configuration.
4. Configure three inbounds, bind the unified `main` client correctly, and register the Node with the Master. Create/update Managed Host rows for every proxied inbound; remove only exact, confirmed duplicate or incorrect rows.
5. Apply the bundled Clash template through the Master settings API/UI. The global Clash rules must not contain a `proxies:` section because 3x-ui supplies dynamic proxies.
6. Regenerate the subscription and perform the acceptance checks in the runbook from an independent network. Diagnose actual packet flow before changing settings. Do not dismiss systematic failures as caching.

## Non-negotiable validation

Validate every generated proxy, not just one successful node:

- VMess: public port `443`, `tls: true`, SNI and WebSocket Host equal that node's domain, and the intended WebSocket path.
- HY2: public UDP port equals the published stream port (normally `8443`), certificate/SNI is the node domain, and end-to-end UDP responses arrive.
- Reality: public dedicated TCP port, correct public key and short ID, Vision flow, a permitted decoy SNI/fingerprint, and generated YAML that carries `client-fingerprint: chrome` for every Reality proxy.
- Parse the generated Clash YAML; check the expected proxy count, display names, groups/rules, and no duplicate/wrong Managed Host binding.
- Independently connect through all proxies and fetch an external HTTP endpoint through each one. Confirm both master and node panels/subscription paths are public only to the extent the user selected.

If a VMess subscription says `tls: false` while Nginx owns public 443, correct the Managed Host TLS/SNI/Host/path override. If HY2 times out through Nginx, inspect the UDP stream configuration and remove `proxy_responses 0`; that directive suppresses upstream responses. See the runbook's repair table before modifying production.

Port-migration invariant: the public HY2 port and the Docker loopback port are
different concerns. When moving HY2 to another public UDP port, first enumerate
all TCP/UDP host mappings and listeners. If the old loopback port was also used
by the container's TCP 443 mapping for VMess, move both container mappings to a
dedicated loopback port (for example, an unexposed 18xxx port), update the Nginx
UDP stream and every VMess WebSocket `proxy_pass`, then update the visible
website proxy entry, firewall/security-group rule, the exact Managed Host row,
and every generated subscription. Never let Nginx and Docker bind the same host
UDP port.

After any HY2 port migration, search the active Nginx configuration and all
panel-managed site fragments for the old loopback target. A stale VMess
WebSocket target can make only VMess time out while HY2 appears fixed. Validate
with `nginx -T`, `nginx -t`, a successful reload, and an independent Mihomo
VMess probe; do not rely on a saved panel setting alone.

Additional-client invariant: a partial `POST /panel/api/clients/update/{email}`
may be accepted while zeroing omitted fields or rebuilding the current
client/inbound relation. Before changing Reality flow, fetch the complete
client record and its intended `inboundIds`; submit a complete normalized
payload, keep `enable: true` and the existing quota/subscription identity, then
verify that the Master still lists every intended inbound. A disabled client is
filtered from node reconciliation, so a remote Node can fall back to `main`
only even though the Master update returned success. If that happens, restore
the intended bindings through the official client attach/inbound update API,
push/reconcile the Node, and inspect the generated Clash YAML. Every VLESS
Reality proxy for that client must contain `flow: xtls-rprx-vision`.

## Completion handoff

Report the architecture, protocol names, verified public panel URLs (including their base paths), panel usernames, and passwords only as authorized by the user. State the result of each acceptance test and any intentional public-access policy. Remind the user to rotate the temporary SSH password and every 1Panel web-login password after deployment, and to store newly created API tokens/panel credentials in a password manager.
