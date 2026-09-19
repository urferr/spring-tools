---
name: validate
description: Check a Spring Boot project for Spring-specific diagnostics using the Spring Tools MCP server. Use when asked to find Spring problems or validate Spring Boot changes.
---
Use the `spring-tools-mcp` MCP server for Spring-specific diagnostics. If it is unavailable, report that validation could not be performed; do not treat missing tools as a clean result.

To perform this validation:

1. Call `refreshWorkspace` before validation so the index includes changes made through patches, shell commands, or external editors. Use `getProjectList` to obtain the exact project names, then call `getProjectDiagnostics` with `projectName` for each project in scope. Wait for these calls to complete; report indexing or timeout failures instead of claiming validation succeeded.
2. Carefully review the returned diagnostics.
3. Identify the specific error code, file path, and text range for each Spring-related diagnostic (e.g. error code "WEB_SECURITY_CONFIGURER_ADAPTER").
4. For each Spring-specific error code, read the [quickfix skill](../quickfix/SKILL.md) with the error code, file path, and text range as context to retrieve the bundled explanation and fix instructions.
5. Report the findings. Apply fixes only when requested by the user, then validate again.
6. These tools do not report general Java compiler errors. Use the project's Maven or Gradle checks when compilation or build validation is part of the request.
