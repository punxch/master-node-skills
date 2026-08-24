# master-node-skills

用于部署、扩容、修复和验收 **Docker Compose + 3x-ui** 的 Master / Node 集群：Master 统一管理订阅与客户端，多个 Node 提供代理入站，并由 1Panel 中可见的网站反向代理对外提供面板、订阅和 TLS/WS 接入。

默认每台主机创建三种协议：

| 协议 | 默认公网入口 | 说明 |
| --- | --- | --- |
| VMess + TLS + WebSocket | `443/TCP` | Nginx 终止 TLS，并反向代理 WebSocket。 |
| Hysteria2 | `8443/UDP` | Nginx `stream` 转发至 Xray。 |
| VLESS + Reality + Vision | `10443/TCP` | Reality 使用独立 TCP 入口，是不走 HTTP 反代的必要例外。 |

## 安装

将仓库克隆到 Codex 的个人技能目录：

```powershell
git clone https://github.com/punxch/master-node-skills.git `
  "$env:USERPROFILE\.codex\skills\master-node-skills"
```

如果已安装，可更新：

```powershell
git -C "$env:USERPROFILE\.codex\skills\master-node-skills" pull --ff-only
```

重新打开任务或新建对话后，直接说明需求，例如：

```text
使用 master-node-skills 部署一个 Master，并添加两台 Node。
```

也可以直接描述扩容、修复订阅、检查 HY2 超时或更新 Clash 模板等任务；Codex 会在适用时选用此技能。

## 开始前需要准备

技能会先向你索取或确认：

- Master 与所有 Node 的 SSH 地址和认证方式；
- 订阅域名、Master 面板域名、各 Node 面板/代理域名，以及 DNS 已正确指向的确认；
- 每台 1Panel 的地址、API Key、Web 登录用户名和密码；若使用其他 Nginx 面板，也需提供等效管理方式；
- 可用证书或签发/续期证书的授权；
- 3x-ui 面板账号密码、面板路径、节点展示名称前缀及公网访问策略；
- HY2 和 Reality 端口（默认分别为 `8443/UDP`、`10443/TCP`）以及开放防火墙/安全组的授权。

请勿把凭据、API Token、订阅 ID、UUID、私密面板路径或证书私钥写入仓库、截图或公开 Issue。

## 部署结果

技能会在 Master 上维护统一的 `main` Client：

- Node 的 VMess 与 VLESS 使用 `main.uuid`；
- Node 的 Hysteria2 使用 `main.auth`；
- Node 注册回 Master 后，通过 Managed Host 将内部入站改写为实际公网域名、端口、TLS/SNI 和 WS 参数；
- Master 生成统一订阅与 Clash YAML。

在 1Panel 中，每个面板/订阅域名都必须有**可见的网站条目**与明确的反向代理配置；仅创建静态站点或私自放置未被加载的 Nginx 文件不算完成。

## 内置 Clash 模板

默认模板在 [assets/sample.yaml](assets/sample.yaml)。将其写入 3x-ui 的 Clash 全局规则时，删除整个 `proxies:` 段落；3x-ui 会自动填充动态代理节点，保留代理组、规则集和规则即可。

## 验收标准

部署不会因为“某一个节点能用”而视为完成。技能要求从独立网络逐条检查：

- VMess 输出必须是 `443`、`tls: true`，且 SNI、WS Host、路径均属于对应 Node；
- HY2 输出必须使用公网 UDP 端口，证书/SNI 正确，且可收到上游 UDP 响应；
- Reality 输出必须有正确端口、Public Key、Short ID 和 Vision flow；
- 额外客户端的 Reality 输出必须包含 `flow: xtls-rprx-vision`。更新客户端时要提交完整状态，保留 UUID/auth、`subId`、配额、启用状态和目标入站绑定；成功响应不等于远程 Node 已完成同步。
- 订阅 YAML 可解析，节点数量、名称、代理组、规则正确；
- 所有节点均能实际代理外部 HTTP 请求；
- 面板和订阅 URL（含路径）、证书与用户选择的访问控制策略均真实生效。

## 常见问题

| 现象 | 优先检查 |
| --- | --- |
| VMess 超时 | Master Managed Host 的 TLS、SNI、WS Host、路径与 Node 实际 Nginx 配置是否一致。 |
| 全部 HY2 超时 | `stream` 是否生效、UDP 端口是否冲突、防火墙是否放行；不要配置 `proxy_responses 0`，它会丢弃上游响应。 |
| HY2 改端口后 VMess 超时 | HY2 与 VMess 可能共用容器 443 的回环映射。迁移时同步迁移 Docker TCP/UDP 回环端口、Nginx stream、所有 VMess WebSocket `proxy_pass`，以及 AcePanel/1Panel 中可见的网站代理目标；搜索并清除旧回环端口引用。 |
| HY2 新端口未出现在订阅 | 检查并新增/更新对应 Managed Host 的公网 UDP 端口、节点范围、SNI 和 ALPN，再重新解析所有客户端订阅。 |
| 额外客户端 Reality 缺少 flow | 将该客户端的 flow 设为 `xtls-rprx-vision`，检查完整客户端状态和所有目标入站绑定，推送/同步 Node 后重新解析 YAML。 |
| 客户端更新后 Node 只剩 `main` | 检查是否因部分更新把客户端变为禁用或丢失 `inboundIds`；使用官方客户端绑定/入站更新 API 恢复，再验证 Node 配置和订阅。 |
| 1Panel 有网站条目但公网面板 404 | 用 `nginx -T` 验证该网站的代理配置是否已加载；执行 `nginx -t`、成功重载后再测试完整路径。 |
| 容器重建后失效 | Nginx 不应指向临时 Docker IP；Host-network OpenResty 场景应使用已验证的静态私有 Docker 地址。 |
| Node 已同步但订阅参数错误 | 检查是否存在重复或错误绑定的 Managed Host，修正后重新生成并解析订阅。 |

详细实施与修复步骤见 [references/deployment-runbook.md](references/deployment-runbook.md)。

## 安全与交付

只在获得服务器和面板所有者授权后执行变更。部署结束时应交付完整的 3x-ui 面板 URL（含路径）、用户名、密码、协议端口及逐节点验收结果；随后立即轮换临时 SSH 密码和 1Panel Web 登录密码，并将新生成的 Token/凭据保存到密码管理器。
