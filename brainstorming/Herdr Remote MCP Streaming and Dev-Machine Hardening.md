# Herdr Remote MCP Streaming and Dev-Machine Hardening

**Prepared:** August 21, 2026

## 1. Herdr output streaming over Tailscale Funnel

Herdr’s documented architecture is a terminal workspace manager. Its server owns panes and process state; clients attach to that server. Herdr renders terminal output, sends input to processes, and preserves panes across client detach. Its CLI communicates with the server over the same local socket API used by integrations and agents.[1] [2]

The important distinction is that Herdr documents **terminal-state streaming and event subscriptions**, not a native remote-MCP token stream. The socket API includes event subscriptions, output waits, and agent-state waits. CLI and API reads return terminal text snapshots; recent reads default to the last 80 rendered terminal rows unless a larger line count is requested.[2] [3]

Therefore, with a design such as `Claude → Tailscale Funnel → MCP gateway → Herdr`, Tailscale transports the HTTPS bytes, while the gateway translates MCP tool calls into bounded Herdr CLI/socket operations. A Funnel does not add token limits, model throttling, or MCP semantics. It also does not make a continuous Herdr terminal stream efficient automatically.

### Recommended gateway behavior

| Control | Recommended behavior |
|---|---|
| Output size | Cap returned output by lines, bytes, and characters. Start with a conservative limit and add pagination or an explicit “read more” tool. |
| Streaming | Prefer event-driven waits or bounded snapshots. Do not repeatedly return the full terminal buffer after every poll. |
| Time | Enforce per-tool and per-request deadlines. A wait should terminate cleanly if the agent remains working or disconnected. |
| Concurrency | Limit simultaneous Herdr reads, prompts, and LLM requests per connector identity. |
| Repetition | Deduplicate identical reads and suppress aggressive polling from the model. |
| Token cost | Count exact model tokens only at the selected LLM/API boundary. A gateway can estimate from characters, but Tailscale and Herdr do not know the model tokenizer. |
| Permissions | Begin with read-only tools. Treat prompt submission, key sending, process control, and destructive operations as separate capabilities. |

Herdr’s experimental pane-graphics transport has a documented 32 MiB headless wire limit, but that is not a general model-token budget and should not be treated as one.[3] The exact Claude context and output limits depend on the Claude surface, model, and MCP client behavior. The safe operational assumption is that **every terminal line returned to Claude becomes potential context cost**, so the adapter must control the amount returned.

## 2. IAM hardening if the tunnel endpoint is compromised

The tunnel should terminate at a dedicated gateway identity, not at your daily user account. Create a service account with no interactive login shell, no sudo rights, no administrative groups, no SSH keys, and no access to browser profiles, password stores, cloud credentials, unrelated project secrets, or the Docker socket. On typical Linux installations, access to the Docker socket is effectively root-equivalent.

Give the gateway a single explicit wrapper for the allowed Herdr operations. The wrapper should validate operation names, target identifiers, argument length, path formats, and timeouts before invoking Herdr. It should never accept arbitrary shell text, arbitrary executable paths, arbitrary filesystem paths, or a generic “send any key” capability. Use separate adapters or authorization scopes for read-only inspection, prompt submission, and terminal input.

Keep credentials independent. The Claude connector credential, OAuth signing keys, refresh tokens, Tailscale auth material, Cloudflare tunnel token, Cloudflare service token, Herdr adapter secret, and LLM-provider keys should not be interchangeable. Store them in a protected secret store or root/service-user-readable files with restrictive permissions. Never put credentials in URLs, logs, prompts, source control, or model-visible output.

On Linux, apply systemd sandboxing to the gateway where compatible with the runtime:

```ini
[Service]
User=herdr-mcp
Group=herdr-mcp
NoNewPrivileges=yes
PrivateTmp=yes
PrivateDevices=yes
ProtectSystem=strict
ProtectHome=yes
CapabilityBoundingSet=
RestrictSUIDSGID=yes
RestrictNamespaces=yes
RestrictAddressFamilies=AF_UNIX AF_INET AF_INET6
ReadWritePaths=/var/lib/herdr-mcp
```

Add only the filesystem paths and address families the adapter genuinely needs. If the adapter needs the Herdr Unix socket, grant access to that socket specifically rather than broad access to `/run`, the user’s home directory, or the entire Herdr state directory. Test each sandbox control because some runtimes need a narrowly scoped exception.

A container or VM is preferable when feasible. Put the exposed gateway and its adapter in a separate security boundary from password managers, SSH agents, browser sessions, personal documents, development credentials, and the Docker daemon. The goal is that a gateway compromise exposes the intended MCP tools, not the workstation.

## 3. Firewall and network hardening

Use default-deny inbound policy on the host and home router. Do not port-forward Herdr, the local LLM proxies, or the MCP gateway from the router. Tailscale Funnel and `cloudflared` initiate outbound connections, so the local setup should not require an Internet-facing listener for the tunnel itself.

Allow SSH only from the management interface or private VPN, and preferably only to a dedicated administrative account with key-based authentication. Do not expose the Herdr socket, LLM proxy admin endpoints, metrics endpoints, or container-management APIs to the LAN unless there is a specific reason.

Bind the origin services narrowly:

```text
Herdr adapter: 127.0.0.1:8787
MCP gateway:   127.0.0.1:8788
LLM proxy A:   127.0.0.1:4000
LLM proxy B:   127.0.0.1:4001
```

The public tunnel should point only to the gateway port. The gateway should connect only to the named loopback services and the specific Herdr socket it needs. Do not configure a generic localhost proxy.

Restrict egress from the gateway or its container where practical. Deny access to router-management ranges, unrelated LAN devices, cloud metadata addresses such as `169.254.169.254`, password-management services, and arbitrary private subnets. Allow only the tunnel control-plane destinations, required loopback/private service addresses, and explicitly required identity or token endpoints.

Example Linux policy concepts, to adapt and test for your distribution, are:

```text
INPUT:  drop by default
INPUT:  allow established,related
INPUT:  allow SSH only from management VPN/CIDR
OUTPUT from gateway: allow loopback and required tunnel/identity destinations
OUTPUT from gateway: deny LAN admin ranges and cloud metadata
FORWARD: deny unless an explicit container/VM route is required
```

The exact commands differ between UFW, nftables, firewalld, Docker, and macOS’s application firewall. The principle is more important than blindly copying a rule: **no inbound port for the tunnel, no broad origin listener, and no unrestricted egress from the exposed process**.

## 4. Failure and recovery tests

Before enabling write-capable tools, test from an external network that is not joined to your tailnet. Confirm that the public endpoint presents a valid certificate, that unauthenticated MCP requests fail closed, that only the intended tool names are available, and that oversized output is truncated or paginated.

Stop Herdr, the gateway, and the tunnel client independently. The public hostname may remain resolvable, but it should return a controlled failure and must not reveal local paths, provider keys, tunnel tokens, or administrative details. Reboot the machine and confirm that only the intended services restart automatically.

Finally, rehearse credential response: revoke the Claude OAuth grant or static connector token, rotate the tunnel credential, rotate any LLM-provider key that the gateway could reach, inspect logs for unexpected tool calls, and disable the Funnel or Cloudflare route with one command. This is the practical difference between a contained endpoint incident and a workstation compromise.

## References

[1]: https://herdr.dev/docs/concepts/ "Herdr Docs — Concepts"

[2]: https://herdr.dev/docs/cli-reference/ "Herdr Docs — CLI reference"

[3]: https://herdr.dev/docs/api/ "Herdr Docs — Socket API"

[4]: https://tailscale.com/docs/features/tailscale-funnel "Tailscale Docs — Tailscale Funnel"

[5]: https://claude.com/docs/connectors/building/authentication "Claude Docs — Authentication for connectors"
