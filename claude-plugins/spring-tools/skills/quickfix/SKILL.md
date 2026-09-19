---
name: quickfix
description: Retrieves the explanation and fix instructions for a specific Spring Boot diagnostic error code and applies the fix to a specific file. Use this when you encounter a Spring Boot warning or error and need to know how to fix it.
---
Use the diagnostic error code, file path, and optional text range from the user's request or the Spring Tools diagnostic result. If the diagnostic is ambiguous, obtain it with `getProjectDiagnostics` before choosing a fix.

MUST DO: If the error code includes a prefix like `errorCode=` or `code=`, strip it out before you continue to use it anywhere.

To find the official explanation and potential fixes for this issue, you must read the explanation file located at:
`../../explanations/<error_code>.md`, resolved relative to this `SKILL.md`, not the workspace directory.

If the file does not exist, say that no bundled explanation is available and use your general Spring Boot knowledge. Explain the fix without editing files when the user only requested an explanation or validation.

Based on the provided "Explanations" and "Fixes" in that file:
1. Analyze the context of the user's project and the specific file to determine which of the suggested fixes is the most appropriate.
2. If there are multiple potential fixes and it is unclear which one to apply based on the project context, stop and ask the user which solution they prefer.
3. When fixing is requested and a solution is chosen, apply it to the specified file.
4. If a text range is provided, keep the fix focused on that location. Spring Tools diagnostic lines and columns are zero-based.
5. Notify the MCP server with `fileChanged` using the absolute file path, then call `getProjectDiagnostics` for the affected project to verify the result.
