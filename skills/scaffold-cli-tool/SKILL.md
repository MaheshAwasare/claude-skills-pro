---
name: scaffold-cli-tool
description: Scaffold a production-grade CLI — Cobra (Go) or Commander (TS/Node) with config, structured logging, subcommands, shell completion, signed releases via goreleaser/np, and homebrew/scoop tap. Use when starting a new CLI you intend to ship to other people; not for one-off scripts.
---

# Scaffold a CLI Tool

CLIs that get used (vs. abandoned) follow the same shape: subcommand structure, sane config precedence, JSON + human output, completion, signed releases, easy install. This scaffold is that shape.

## When to use

- Building a CLI you'll ship publicly or share across a team.
- Tool is non-trivial: ≥3 subcommands, takes config from multiple sources.
- You'll publish via Homebrew, Scoop, npm, or GitHub Releases.

## When NOT to use

- One-off bash script — overkill.
- Embedded CLI inside a larger app (e.g. `next dev`) — different scaffolding.
- TUI-heavy interactive tool — see Bubble Tea / Ink scaffolds (separate skills).

## Pick a language

| Choose | When |
|---|---|
| **Go (Cobra)** | Single static binary, no runtime to install, cross-compile easy, fastest startup. Default for distribution to non-developers. |
| **TS/Node (Commander)** | You're already in JS/TS shop, npm install is fine for users, you want to share types with backend code. |

Both produce the same UX. The rest of this skill shows both.

## Decisions made for you

| Decision | Go | TS/Node |
|---|---|---|
| CLI lib | `spf13/cobra` | `commander` |
| Config | `spf13/viper` (Cobra-friendly) | `cosmiconfig` + `env-paths` |
| Output | `tablewriter` + JSON | `cli-table3` + JSON |
| Logging | `slog` | `pino` |
| Errors | sentinel + `errors.Is` | typed Error subclasses |
| Tests | stdlib + `testify` | vitest |
| Releases | `goreleaser` | `np` + GitHub Actions |
| Distribution | Homebrew tap, Scoop bucket, GH Releases | npm, optional Homebrew |

## Config precedence (the table users want)

Every CLI must document this. Highest priority wins:

| # | Source | Example |
|---|---|---|
| 1 | Flag | `--api-url=https://x` |
| 2 | Env var | `MYCLI_API_URL=https://x` |
| 3 | Project config | `./mycli.yaml` |
| 4 | User config | `~/.config/mycli/config.yaml` (XDG) |
| 5 | Built-in default | hardcoded |

XDG paths matter: don't dump dotfiles in `$HOME`. Use `os.UserConfigDir()` (Go) / `env-paths` (Node). On Linux: `~/.config/mycli/`. Mac: `~/Library/Application Support/mycli/`. Windows: `%APPDATA%\mycli\`.

## File structure (Go / Cobra)

```
mycli/
  cmd/
    mycli/main.go               # entrypoint
  internal/
    cli/
      root.go                   # rootCmd, persistent flags
      version.go                # mycli version
      config.go                 # mycli config get/set
      foo.go                    # mycli foo subcommand group
      foo_list.go               # mycli foo list
      foo_create.go             # mycli foo create
    config/
      config.go                 # struct + loader
    output/
      table.go                  # human format
      json.go                   # JSON format
  .goreleaser.yaml
  Makefile
  README.md
  go.mod
  .github/workflows/release.yml
```

## File structure (TS / Commander)

```
mycli/
  src/
    bin/mycli.ts                # shebang + dispatch
    cli/
      root.ts
      version.ts
      config.ts
      foo/
        index.ts                # foo subgroup
        list.ts
        create.ts
    config/loader.ts
    output/{table.ts,json.ts}
  package.json                  # "bin": { "mycli": "dist/bin/mycli.js" }
  tsconfig.json
  vitest.config.ts
  .github/workflows/release.yml
```

## Cobra root command

```go
// internal/cli/root.go
var rootCmd = &cobra.Command{
    Use:           "mycli",
    Short:         "Do the thing",
    SilenceUsage:  true,                // don't print usage on every error
    SilenceErrors: true,                // we print our own
    PersistentPreRunE: func(cmd *cobra.Command, _ []string) error {
        return loadConfig(cmd)
    },
}

func init() {
    rootCmd.PersistentFlags().String("config", "", "config file (default: $XDG_CONFIG_HOME/mycli/config.yaml)")
    rootCmd.PersistentFlags().StringP("output", "o", "table", "output format: table|json")
    rootCmd.PersistentFlags().Bool("verbose", false, "verbose logs to stderr")
}

func Execute() {
    if err := rootCmd.Execute(); err != nil {
        fmt.Fprintf(os.Stderr, "Error: %s\n", err)
        os.Exit(1)
    }
}
```

`SilenceUsage: true` is non-obvious but critical — without it, every error prints the full --help, which makes piped CLIs unreadable.

## Output: human vs JSON (every command, both)

```go
// internal/cli/foo_list.go
var fooListCmd = &cobra.Command{
    Use:   "list",
    Short: "List foos",
    RunE: func(cmd *cobra.Command, _ []string) error {
        foos, err := api.ListFoos(cmd.Context())
        if err != nil {
            return err
        }
        format, _ := cmd.Flags().GetString("output")
        return output.Render(cmd.OutOrStdout(), foos, format)
    },
}
```

`--output=json` is non-negotiable. Users will pipe your tool to `jq`. If you only have a table output, your CLI is dead in scripts.

## Stdout vs stderr discipline

| Stream | What goes there |
|---|---|
| `stdout` | Data the user is asking for. `mycli foo list` → list of foos. |
| `stderr` | Logs, progress, errors, warnings, prompts. |

Why: piping `mycli foo list | wc -l` should count foos, not log lines. Get this wrong and your CLI fails silently inside larger pipelines.

## Shell completion

```go
// internal/cli/completion.go
var completionCmd = &cobra.Command{
    Use:   "completion [bash|zsh|fish|powershell]",
    Short: "Generate shell completion",
    Args:  cobra.ExactArgs(1),
    RunE: func(cmd *cobra.Command, args []string) error {
        switch args[0] {
        case "bash":       return rootCmd.GenBashCompletion(os.Stdout)
        case "zsh":        return rootCmd.GenZshCompletion(os.Stdout)
        case "fish":       return rootCmd.GenFishCompletion(os.Stdout, true)
        case "powershell": return rootCmd.GenPowerShellCompletion(os.Stdout)
        }
        return nil
    },
}
```

README must show how to install completion. Most users won't do it, but the 10% who do become evangelists.

## Releases (Go via goreleaser)

`.goreleaser.yaml`:
```yaml
project_name: mycli
builds:
  - main: ./cmd/mycli
    env: [CGO_ENABLED=0]
    goos: [linux, darwin, windows]
    goarch: [amd64, arm64]
    ldflags:
      - -s -w -X main.version={{.Version}} -X main.commit={{.ShortCommit}}
archives:
  - format_overrides:
      - goos: windows
        format: zip
brews:
  - tap:
      owner: yourorg
      name: homebrew-tap
    folder: Formula
    homepage: https://github.com/yourorg/mycli
    description: Do the thing
checksum:
  name_template: "checksums.txt"
signs:
  - cmd: cosign
    args: ["sign-blob", "--yes", "--output-signature=${signature}", "${artifact}"]
    artifacts: checksum
```

GitHub Actions workflow on tag push: `goreleaser release --clean`. Cosign sigs make supply-chain auditors happy.

## Releases (TS via np + GitHub Releases)

`package.json`:
```json
{
  "bin": { "mycli": "dist/bin/mycli.js" },
  "files": ["dist"],
  "scripts": {
    "build": "tsup src/bin/mycli.ts --format esm --target node20 --clean",
    "release": "np --no-yarn"
  }
}
```

`tsup` produces a single bundled JS file with proper shebang. Node 20+ supports `--experimental-sea-config` if you want true single-binary distribution.

## Anti-patterns

- **No `--output=json`** — kills automation use. Add it from the start.
- **Logs on stdout** — breaks piping. stderr for everything except output data.
- **Dotfiles in `$HOME`** (`~/.mycli`) — pollutes home dir. Use XDG / OS conventions.
- **Hardcoded API URLs without env override** — every other tool a user has supports `MYCLI_API_URL=...`. Yours should too.
- **No `--version` or version that lies** — embed git SHA via ldflags. `mycli version` should print version + commit + build date.
- **Interactive prompts without a `--yes`/`--no-confirm` escape hatch** — blocks scripts and CI. Always allow non-interactive.
- **Errors via panic** — print to stderr, set exit code, leave gracefully. `os.Exit(1)` ≠ panic.
- **Exit code only 0 or 1** — Unix convention: 2 for misuse, 64–78 for sysexits, custom codes documented. At minimum: 0=ok, 1=error, 2=usage.
- **No completion** — users who want completion are the ones who'll evangelize.

## Verify it worked

- [ ] `mycli --help` shows clean usage; subcommands listed.
- [ ] `mycli foo list -o json | jq '.[0].name'` produces a value.
- [ ] `mycli foo list 2>/dev/null` returns clean output (logs aren't on stdout).
- [ ] `mycli version` prints version, commit SHA, build date.
- [ ] Setting `MYCLI_API_URL=https://x` overrides the default; `--api-url=https://y` overrides the env.
- [ ] `mycli completion bash > /tmp/c && source /tmp/c && mycli <TAB><TAB>` shows subcommands.
- [ ] Config in `~/.config/mycli/config.yaml` is read; flags override it.
- [ ] `goreleaser release --snapshot --clean` builds binaries for linux/darwin/windows × amd64/arm64.
- [ ] First Homebrew tap install works: `brew tap yourorg/tap && brew install mycli`.
- [ ] Exit code is 0 on success, 1 on runtime error, 2 on bad flags.
