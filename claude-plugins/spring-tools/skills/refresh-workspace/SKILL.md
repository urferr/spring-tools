---
name: refresh-workspace
description: Refresh the Spring Tools MCP workspace after a Git checkout, shell changes, or external edits so its index reflects files on disk.
---

Call `refreshWorkspace` on the `spring-tools-mcp` MCP server and wait for completion. Confirm that the refresh was requested, or report the tool error. Do not claim the project is free of diagnostics; use `getProjectDiagnostics` for that check.
