# nica-vet — Claude Code plugin

Operate nica.vet through its MCP operation registry (`POST /mcp` on the brand's MCP host, `https://ai.nica.vet/mcp`). Every tool is one registered operation, authorized, audited, and scoped to one Org and one membership: the person who connected.

A person connects by signing in: the assistant opens nica.vet's consent screen (`/connect-ai` on operator-web); the person signs in, picks the Org and the areas, signs in again and approves. Nothing is copied by hand. The connection acts as that person, inside what their role reaches, never on `platform` or `identity`, with every write confirmed per call.

Members who prefer an API key connect it from the nica.vet app (Configuración › Claves de API); that path is documented there, not in this plugin.

Prerequisites:

- The person is an `owner`, `admin` or `member` of the Org and has a second factor (password + OTP, a passkey, or password + security key).
- The Org holds the `mcp` entitlement.
- A membership holds at most 10 active assistant connections, counted across every assistant sign-in.

## Install

### From the Claude directory

1. Open claude.ai → Directory and pick the nica.vet plugin.
2. Install it. Claude Code loads it as `nica-vet@synced` in sessions signed in with that account.
3. For claude.ai, Claude Desktop and the mobile apps, add the paired nica.vet connector from the same directory listing: it connects to `https://ai.nica.vet/mcp` and opens the same consent screen.

### From the public repository

The repository `FlameFactoryIO/nica-vet-claude-plugin` holds this folder as its root. Its `.claude-plugin/marketplace.json` lists the plugin, so the repository is also a marketplace. From docs, untested until the repository is public:

```bash
claude plugin marketplace add FlameFactoryIO/nica-vet-claude-plugin
claude plugin install nica-vet@nica-vet
```

One command, Claude Code v2.1.292 or later (from docs, untested):

```bash
claude plugin install nica-vet --marketplace FlameFactoryIO/nica-vet-claude-plugin
```

### Local development from the monorepo

From the repository root, add the monorepo marketplace by relative path and install:

```bash
claude plugin marketplace add ./
claude plugin install nica-vet@flamefactoryio
```

To use a local API, set the `nica.vet MCP URL` option to `http://localhost:3001/mcp` (see Configuration).

### Configuration

The plugin declares one option, `mcpUrl`, in `userConfig`, and its `.mcp.json` declares one server, `nica-vet`, whose `url` is `${user_config.mcpUrl}`, with no `Authorization` header:

| Option   | Required | Default                   | Meaning                                                                                                                           |
| -------- | -------- | ------------------------- | --------------------------------------------------------------------------------------------------------------------------------- |
| `mcpUrl` | no       | `https://ai.nica.vet/mcp` | MCP endpoint of the nica.vet API you connect to; keep the default for production, set `http://localhost:3001/mcp` for a local API |

Claude Code prompts for `mcpUrl` when the plugin is enabled and uses the default when the field is left empty; change it later in `/config`. The URL has to be an `MCP_BRAND_HOSTS` origin followed by `/mcp`: the sign-in is issued for that resource, another spelling of the host is refused (`invalid_target`), and a host that is not mapped answers 404.

## Data and privacy

The plugin gives Claude Code access to the nica.vet data of one Org, as the person who signed in: customers, patients, appointments, catalog and inventory records they could already open in nica.vet. What a tool returns goes into the conversation with Anthropic's Claude, under Anthropic's terms. The plugin stores nothing on the machine besides Claude Code's own sign-in tokens. Every write is confirmed per call and audited under the person's identity. How nica.vet handles assistant connections: https://nica.vet/privacidad#asistentes. What the connection does and how to disconnect it: https://nica.vet/ai.

## Support

soporte@nica.vet, https://nica.vet/soporte.

## Connect by signing in

1. Start Claude Code with the plugin installed. The `nica-vet` server answers 401 and Claude Code lists it as needing authentication.
2. Run `/mcp`, pick `nica-vet`, pick **Authenticate** (or run `claude mcp login nica-vet` in a shell). Claude Code reads the API's metadata and opens the browser on the consent screen, `/connect-ai`.
3. Sign in to nica.vet, pick the Org, check the areas the assistant may use, and press «Verificar identidad y aprobar». The second sign-in is the fresh-auth proof; no second person takes part.
4. The browser returns to `http://localhost:<port>/callback`, a port Claude Code opened for this sign-in, and Claude Code shows `nica-vet` as connected. When the browser cannot reach that port (a remote shell), paste the address-bar URL into the prompt Claude Code shows.

The consent screen reads «Conectar Claude Code con <empresa>» and «Lo pide claude.ai.»: the name and the host are the ones of the client document Anthropic publishes at `https://claude.ai/oauth/claude-code-client-metadata`, never text the client sent. The connection has no lifetime cap: it lasts until the person disconnects it (`platform.mcp.oauth.connection.revoke`), an owner or admin ends it, or a change of the person's role or their offboarding ends it. Claude Code renews its 1-hour access token by itself, and each renewal rotates a refresh token that stays good for 30 days of inactivity.

To disconnect: Administración › IA (`/ai`) → «Conexiones con asistentes» → «Desconectar» in nica.vet, or `/mcp` → `nica-vet` → **Clear authentication** in Claude Code. To change the areas, disconnect and connect again.

## Claude Desktop and claude.ai

No plugin is needed. Add the nica.vet connector from the Claude directory (claude.ai → Directory → nica.vet → Connect), or Settings › Connectors › «Add custom connector», URL `https://ai.nica.vet/mcp`, then «Connect»: the same consent screen opens and reads «Conectar Claude con <empresa>» and «Lo pide claude.ai.». The connector lives in the person's claude.ai account, so it also appears in Claude Desktop, the mobile apps, and Claude Code signed in with that account.

## Write confirmation (plan 046)

1. A gated tool (high-stakes, or non-reversible; a sign-in sets `confirmWrites: true`) answers the first call with `{ "status": "CONFIRMATION_REQUIRED", confirmationId, confirmationFingerprint, operation, summary, expiresAt }` and does nothing.
2. Claude shows the operation, summary, and arguments and waits for your explicit yes.
3. The second call resends identical arguments plus `confirmationId` and `confirmationFingerprint`; it executes once.
4. Confirmations expire after 10 minutes.
5. On an ambiguous outcome, `platform__mcp__confirmation_status { confirmationId }` returns the state and stored receipt.
