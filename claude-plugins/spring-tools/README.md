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

Installing the plugin provides the shared skills and launcher, but does not register an MCP server in Codex. Enable the server separately in each Spring Boot project's `.codex/config.toml` as described below. Projects without this configuration do not start a Spring Language Server, provided no user-level or other inherited configuration registers it.

The first start downloads the checksum-verified JAR; Maven/Gradle project import can also require network access. To avoid spending the MCP startup timeout downloading the JAR, run `node claude-plugins/spring-tools/install.js` before starting the server (or build it with `update-local-jars.sh`).

The shared skills are `validate`, `quickfix`, and `refresh-workspace`. Project generation is intentionally not included. Ask Codex to validate the project, explain or fix a diagnostic, or refresh the workspace after external changes.

The portable `plugin.json` explicitly disables bundled hooks for Codex, including automatic discovery of the Claude Code hooks in `hooks/hooks.json`. The optional `hooks/codex.json` template calls `refreshWorkspace` after `apply_patch` (including its `Edit`/`Write` aliases), covering multi-file patches, additions, deletions, and renames without assuming a single `file_path` argument. Claude Code retains its existing MCP registration and hooks.

For automatic refresh, follow the [project-local hook example](#project-local-refresh-hook-optional) below. Without it, use the `refresh-workspace` skill after changes; `validate` always refreshes before querying diagnostics, including after shell or external edits.

The launcher, server JAR, explanations, and skills are shared between both hosts. The `.codex-plugin/plugin.json` file supplies compatibility metadata; the portable `mcp.json` intentionally contains no server registrations.

### Project-local MCP configuration

Add this to `.codex/config.toml` in each Spring Boot project's root, adapting the absolute launcher path to your installation. Codex loads project configuration only for trusted projects.

```toml
[mcp_servers.spring-tools-mcp]
command = "node"
args = ["D:/path/to/spring-tools/claude-plugins/spring-tools/launcher.js"]
startup_timeout_sec = 180
tool_timeout_sec = 120
```

No `cwd` attribute is needed in the tested Codex Desktop setup: the server starts in the project's context, and the launcher uses `process.cwd()` as the project root. This was verified with two separately configured projects. For a multi-module build, use the common Maven/Gradle build root. Other MCP clients must start the launcher with the intended project as their working directory.

Remove or disable any previous Spring MCP entry in your user-level `~/.codex/config.toml` or other inherited configuration, and remove any global Spring refresh hooks. The former `SPRING_TOOLS_WORKSPACE_DIR` environment variable is no longer read and can be removed. Existing installed plugin copies must be updated to pick up the removal of the bundled server and hooks.

This configuration can also be used without installing the plugin, but exposes the tools only, without the skills or hooks. For direct usage, call `refreshWorkspace` after disk changes, `getProjectList` to obtain project names, and `getProjectDiagnostics` with the chosen `projectName`.

### Project-local refresh hook (optional)

First configure `spring-tools-mcp` in the project's `.codex/config.toml` as shown above. Then create `.codex/hooks.json` in the same project with this content (also available as `hooks/codex.json` in the plugin):

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "^(apply_patch|Edit|Write)$",
        "hooks": [
          {
            "type": "mcp_tool",
            "server": "spring-tools-mcp",
            "tool": "refreshWorkspace",
            "input": {}
          }
        ]
      }
    ]
  }
}
```

The resulting project layout is:

```text
your-spring-boot-project/
├── .codex/
│   ├── config.toml   # Project-local MCP server
│   └── hooks.json    # Project-local refresh hook
├── pom.xml          # Or the Gradle build file
└── src/
```

If `.codex/hooks.json` already exists, append the example's matcher object to its `hooks.PostToolUse` array, creating that array if necessary. Preserve other events and hooks. Do not also register the same hook inline in `config.toml`, because both definitions would run.

To activate and check the hook:

1. Open a terminal in this project's root and start `codex`. Trust the project if prompted so its local configuration can load.
2. Enter `/hooks` in the Codex CLI. Review and trust the hook from this project's `.codex/hooks.json`; enable it if it was disabled. New or changed hook definitions are skipped until reviewed. Hooks are enabled by default; if your configuration explicitly sets `[features] hooks = false`, change that setting to `true`.
3. Start a new Codex Desktop task for this project to load the updated configuration. `/hooks` is the documented CLI review flow; this Desktop hook setup has not yet been tested end to end.
4. Ask Codex to make a small source edit using `apply_patch`. After the edit, the hook calls `refreshWorkspace` on the already connected `spring-tools-mcp` server. Check the hook execution output and request diagnostics to verify the index was refreshed.

The `server` value must match the name in `[mcp_servers.spring-tools-mcp]`. No launcher path or `cwd` belongs in `hooks.json`: the hook uses the existing MCP connection and does not start a server. Shell commands and external editor changes do not match this hook; use `refresh-workspace` or `validate` after those changes. Remove this project's Spring refresh hook if you disable its MCP server.

See the official [Codex hooks documentation](https://learn.chatgpt.com/docs/hooks) for configuration locations and the review/trust flow.

### Verification and local updates

Verify that a project without local or inherited Spring MCP configuration starts no Spring server. In a configured project, verify that the server initializes and exposes `getProjectList`, `getProjectDiagnostics`, `fileChanged`, `fileDeleted`, and `refreshWorkspace`. Check that only the intended build and its modules appear in `getProjectList`. Repeat with two separately configured projects to verify that their server contexts remain separate. Request diagnostics, then make a relevant source change and verify the diagnostics update after refresh. General Java compiler errors still require Maven or Gradle checks.

Codex installs a cached copy of local plugins. After editing this source, remove and add the plugin again, then start a new task/session:

```bash
codex plugin remove spring-tools@spring-tools-local
codex plugin add spring-tools@spring-tools-local
```

See the official [plugin packaging](https://developers.openai.com/plugins/build/plugins), [MCP](https://developers.openai.com/codex/mcp), and [hooks](https://developers.openai.com/codex/hooks) documentation.

## Project directory

The launcher uses its startup working directory (`process.cwd()`) as the project root for both Claude Code and Codex. Start it in the Spring Boot project or the common root of a multi-module Maven/Gradle build, rather than a parent workspace containing unrelated projects.

The directory is read at server startup; changing it requires a server restart. Verify the selected path in the `Initializing workspace project directory:` entry in `boot-ls.log` beside `launcher.js`.

The standalone server also reads its `.claude/spring-tools.properties` and `.claude/spring-tools.json` settings from this project root.

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
├── plugin.json              # Portable manifest; bundled Codex hooks disabled
├── mcp.json                 # Empty portable MCP server registry (project-local opt-in)
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

Claude Code reads the MCP configuration in `.claude-plugin/plugin.json`; Codex starts the server only when configured through `.codex/config.toml` in the project (or an inherited MCP configuration). Both start `launcher.js` with the project as its working directory. The launcher checks if the heavy Java JAR is downloaded. If not, it executes `install.js` to download it from Spring's CDN. Then it boots the standalone Spring Tools Language Server, instructing it to expose its MCP tools over `stdio` (the language server's own LSP socket transport is disabled, since nothing in this plugin connects to it).
