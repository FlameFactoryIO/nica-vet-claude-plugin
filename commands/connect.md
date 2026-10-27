---
description: Connect this Claude Code to nica.vet by signing in
---

Connect the `nica-vet` MCP server for the person at this machine. Arguments given: `$ARGUMENTS`.

1. Connect by signing in; nothing is copied by hand. It acts as that person in one Org, inside what their role reaches, and confirms every write. Members who prefer an API key connect it from the nica.vet app (Configuración › Claves de API); that path is documented there, not in this plugin.
2. You cannot sign in for them; tell them the steps and wait:
   - The plugin connects to production, `https://ai.nica.vet/mcp`, by default (the `mcpUrl` option, changeable in `/config`; `http://localhost:3001/mcp` for a local API). No user-scope server named `nica-vet` with an `Authorization` header exists (`claude mcp get nica-vet`; remove it with `claude mcp remove nica-vet`).
   - They run `/mcp`, pick `nica-vet`, pick **Authenticate**. The browser opens nica.vet's consent screen.
   - They sign in, pick the Org, check the areas, and press «Verificar identidad y aprobar».
   - Back in Claude Code, `/mcp` shows `nica-vet` connected. If the screen showed an error instead, relay its sentence and stop.
