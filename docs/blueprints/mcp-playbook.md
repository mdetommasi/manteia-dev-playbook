# MCP (Model Context Protocol) — Playbook

- **Server**: espone tool dichiarativi (manifest, schema input/output, auth).
- **Client**: agent/app che negozia capabilities e invoca tool.
- **Contract**: versionare manifest, backward compatibility, test di handshake.
- **Sicurezza**: scoping dei permessi (principle of least privilege), rate-limit per tool.
- **Testing**: simulare errori di rete/timeout; golden I/O per ogni tool.
