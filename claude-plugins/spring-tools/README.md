# Spring Tools Language Server — Claude Code and Codex Plugin

A plugin for [Claude Code](https://code.claude.com) and [Codex](https://developers.openai.com/codex), exposing Spring Boot diagnostics, bean/request-mapping lookups, and other project insights via MCP tools.

Unlike the VS Code extension, this plugin uses the **standalone** variant of the language server which operates **without** JDT Language Server. Project classpath is computed directly via Maven and Gradle tooling; type indexing uses Jandex.

## Requirements

- Java 21+ on `PATH`
- Node.js on `PATH`
- Maven or Gradle projects in your workspace

## Claude Code usage

### 1. Add the Marketplace

First, add either the Release or Snapshot marketplace to Claude Code:

**To use the stable release:**
```bash
claude plugin marketplace add https://cdn.spring.io/spring-tools/release/claude-plugins/marketplace.json
```

**To use the bleeding-edge snapshot:**
```bash
claude plugin marketplace add https://cdn.spring.io/spring-tools/snapshot/claude-plugins/marketplace.json
```

### 2. Install the Plugin

Once the marketplace is added, install the plugin:

**If you added the stable release marketplace:**
```bash
claude plugin install spring-tools@spring-tools-marketplace
```

**If you added the snapshot marketplace:**
```bash
claude plugin install spring-tools@spring-tools-snapshots
```

### 3. Update the Plugin

When new versions of the plugin are published to the marketplace, update it by running:

```bash
claude plugin marketplace update
claude plugin update spring-tools
```

### 4. Testing the Plugin

To verify that the Spring Boot Language Server is correctly booting up and serving MCP tools to Claude Code, you must run Claude Code **interactively** (don't use the `-p` single-shot flag, as it will kill the CLI before the language server finishes initializing).

Open a Spring Boot project and start Claude Code:
```bash
claude
```

Then, ask Claude a test query to verify the MCP integration. For example:
> "Check the project diagnostics for CoffeeController.java."

Claude will wait for the language server to initialize, call the `getProjectDiagnostics` MCP tool, and summarize the exact Spring Boot warnings and quick fixes it reports.

### 5. Local Testing

We maintain a local marketplace configuration (`claude-plugins/.claude-plugin/marketplace.json`) to make testing the plugin directly from the source tree easy.

1. Run the update script to build the standalone language server JAR and copy it into this plugin's directory (run this from the `claude-plugins` directory):
   ```bash
   ./update-local-jars.sh
   ```
2. Add the local `claude-plugins` directory as a marketplace (run this from the `sts4` root directory):
   ```bash
   claude plugin marketplace add ./claude-plugins
   ```
3. Install the plugin from your new local marketplace:
   ```bash
   claude plugin install spring-tools@spring-tools-local
   ```

## Codex usage

From this repository's root, register the existing local marketplace and install the plugin:

```bash
codex plugin marketplace add ./claude-plugins
codex plugin add spring-tools@spring-tools-local
```

The server uses `SPRING_TOOLS_WORKSPACE_DIR` as the project root when set; otherwise it uses its startup working directory. Desktop hosts may start the server in the plugin installation directory, so configure the workspace explicitly as described below. The first start downloads the checksum-verified JAR; Maven/Gradle project import can also require network access. To avoid spending the MCP startup timeout downloading the JAR, run `node claude-plugins/spring-tools/install.js` before installing the local plugin (or build it with `update-local-jars.sh`).

The shared skills are `validate`, `quickfix`, and `refresh-workspace`. Project generation is intentionally not included. Ask Codex to validate the project, explain or fix a diagnostic, or refresh the workspace after external changes.

The portable `plugin.json` declares `hooks/codex.json` for Codex. This hook calls `refreshWorkspace` after `apply_patch` (including its `Edit`/`Write` aliases), covering multi-file patches, additions, deletions, and renames without assuming a single `file_path` argument. Claude Code retains its existing hooks.

**Codex 0.155.0 limitation:** Installation and all 22 MCP tools were verified, but this version did not discover hooks declared in the portable manifest. To enable automatic refresh on this version, merge the contents of `hooks/codex.json` into the target project's `.codex/hooks.json` (do not overwrite existing hooks), then enable and review/trust them using `/hooks` in the CLI. This manual hook configuration has not been end-to-end tested. Without it, use the `refresh-workspace` skill after changes; `validate` always refreshes before querying diagnostics, including after shell or external edits.

The launcher, server JAR, explanations, and skills are shared between both hosts. The `.codex-plugin/plugin.json` file supplies compatibility metadata; the portable `mcp.json` supplies the tested MCP connection.

### Direct MCP configuration

To use only the server, or with a Codex version that does not load bundled MCP servers, add this to your Codex `config.toml`, adapting both absolute paths:

```toml
[mcp_servers.spring-tools-mcp]
command = "node"
args = ["D:/path/to/spring-tools/claude-plugins/spring-tools/launcher.js"]
cwd = "D:/path/to/your-spring-boot-project"
startup_timeout_sec = 180
tool_timeout_sec = 120
```

Use either the bundled MCP server or this direct configuration to avoid starting two servers for the same project. Direct MCP configuration exposes the tools only; it does not install the skills or hooks. For direct usage, call `refreshWorkspace` after disk changes, `getProjectList` to obtain project names, and `getProjectDiagnostics` with the chosen `projectName`.

### Verification and local updates

Verify that the server initializes and exposes `getProjectList`, `getProjectDiagnostics`, `fileChanged`, `fileDeleted`, and `refreshWorkspace`. Check that your project appears in `getProjectList`, request diagnostics, then make a relevant source change and verify the diagnostics update after refresh. General Java compiler errors still require Maven or Gradle checks.

Codex installs a cached copy of local plugins. After editing this source, remove and add the plugin again, then start a new task/session:

```bash
codex plugin remove spring-tools@spring-tools-local
codex plugin add spring-tools@spring-tools-local
```

See the official [plugin packaging](https://developers.openai.com/plugins/build/plugins), [MCP](https://developers.openai.com/codex/mcp), and [hooks](https://developers.openai.com/codex/hooks) documentation.

## Configuring the workspace directory

Set `SPRING_TOOLS_WORKSPACE_DIR` to the absolute path of your project or a workspace containing multiple Maven/Gradle projects. This applies to both Claude Code and Codex and takes precedence over the launcher's working directory. If the variable is unset or empty, the launcher falls back to `process.cwd()`.

For a persistent Windows user setting, run in PowerShell, replacing the example path:

```powershell
[Environment]::SetEnvironmentVariable('SPRING_TOOLS_WORKSPACE_DIR', 'D:\Projects\workspace', 'User')
```

Fully exit and restart the host application so its MCP server inherits the new environment. For a CLI launched from the current PowerShell session, also set `$env:SPRING_TOOLS_WORKSPACE_DIR = 'D:\Projects\workspace'` before launching it. The directory is read at server startup; changing it requires a server restart. Verify the selected path in the `Initializing workspace project directory:` entry in `boot-ls.log` beside `launcher.js`.

The standalone server also reads its `.claude/spring-tools.properties` and `.claude/spring-tools.json` settings from this selected root. When using a shared workspace directory, place shared server settings there.

## Configuring language server preferences

You can customize validation severities and other language server settings on a per-project basis by placing a settings file inside the `.claude/` directory of your project. This shared server configuration path also applies when using Codex. Two formats are supported and may coexist — the properties file provides the base values and the JSON file overrides them.

### Properties format (`.claude/spring-tools.properties`)

Flat `key=value` format where each key is the full dot-separated settings path. This is the simplest format to get started with:

```properties
# Category enablement toggles (AUTO / ON / OFF)
boot-java.validation.java.boot2=OFF
boot-java.validation.java.boot3=ON
boot-java.validation.spel.on=ON
boot-java.validation.java.version-validation=OFF

# Per-problem severity overrides (IGNORE / HINT / INFO / WARNING / ERROR)
spring-boot.ls.problem.boot2.JAVA_PUBLIC_BEAN_METHOD=IGNORE
spring-boot.ls.problem.boot2.JAVA_AUTOWIRED_CONSTRUCTOR=IGNORE
```

### JSON format (`.claude/spring-tools.json`)

Nested JSON matching the VSCode `boot-java` / `spring-boot` configuration structure. Useful when you want to express several settings for the same category together:

```json
{
  "boot-java": {
    "validation": {
      "java": {
        "boot2": "OFF",
        "boot3": "ON"
      },
      "spel": { "on": "ON" },
      "version-validation": { "on": "OFF" }
    }
  },
  "spring-boot": {
    "ls": {
      "problem": {
        "boot2": {
          "JAVA_PUBLIC_BEAN_METHOD": "IGNORE",
          "JAVA_AUTOWIRED_CONSTRUCTOR": "IGNORE"
        }
      }
    }
  }
}
```

### Available settings

**Category enablement toggles** (`boot-java.validation.*`) accept `AUTO`, `ON`, or `OFF`.

**Per-problem severity overrides** (`spring-boot.ls.problem.<category>.<code>`) accept `IGNORE`, `HINT`, `INFO`, `WARNING`, or `ERROR`.

The full list of available categories and problem codes is embedded in the language server JAR as `problem-types.json`. They are the same keys used in the VSCode extension's settings.

Settings are applied once at startup. Restart the language server by starting a new Claude Code or Codex session for changes to take effect.

## What the language server provides

Via MCP tools:

- **Diagnostics** — Spring-specific warnings and quick fixes (missing annotations, incorrect bean wiring, etc.), including version validation results
- **Project insight** — bean, component, and request-mapping lookups; resolved project classpath

Via Claude Code hooks (`hooks/hooks.json`), the plugin also tracks file and project changes on disk to keep its internal index up to date, and exposes a command to refresh the index manually. These hooks are gated to Java/Kotlin/Groovy source files and build/config files (`.java`, `.kt`, `.kts`, `.groovy`, `.xml`, `.properties`, `.yml`, `.yaml`, `.gradle`) — edits to unrelated files don't trigger them. The workspace-refresh hook fires only on `git` commands. For Codex hook behavior and compatibility, see the Codex usage section above.

## Plugin structure

```
spring-tools/
├── .claude-plugin/
│   └── plugin.json          # Claude Code manifest (MCP server config)
├── .codex-plugin/
│   └── plugin.json          # Codex compatibility metadata
├── plugin.json              # Portable manifest and Codex hook declaration
├── mcp.json                 # Portable MCP server configuration
├── launcher.js              # Node.js script that downloads the JAR (if missing) and starts Java
├── install.js               # Node.js script that downloads the JAR
├── hooks/                   # Hooks that notify the language server of file/project changes
├── language-server/         # Populated by install.js on first run (gitignored)
│   └── spring-boot-language-server-standalone-exec.jar
├── skills/                  # Shared Claude Code and Codex skills
│   ├── validate/
│   ├── refresh-workspace/
│   └── quickfix/
├── explanations/            # Markdown files with problem explanations and fixes
└── README.md
```

## How it works

Claude Code reads the MCP configuration in `.claude-plugin/plugin.json`; Codex reads the portable `plugin.json` and `mcp.json`. Both start `launcher.js`, which checks if the heavy Java JAR is downloaded. If not, it executes `install.js` to download it from Spring's CDN. Then it boots the standalone Spring Tools Language Server, instructing it to expose its MCP tools over `stdio` (the language server's own LSP socket transport is disabled, since nothing in this plugin connects to it).
