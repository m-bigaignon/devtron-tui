# Contract: Command line, configuration file and token resolution

## Command line

```text
devtron-tui [--instance <name>] [--config <path>] [--readonly]
```

| Flag | Behaviour |
|---|---|
| `--instance <name>` | Start on this registered instance. If the name is unknown, exit code 2 with "unknown instance '<name>'; registered: a, b" (spec edge case). |
| `--config <path>` | Use a different config file. Default: `$XDG_CONFIG_HOME/devtron-tui/config.toml`, else `~/.config/devtron-tui/config.toml`. |
| `--readonly` | Hides and disables every mutating action. In this feature it is the only mode, and it is accepted so scripts and aliases don't need to change later (constitution I). |

There is no flag that takes a token (constitution II, FR-002).

Exit codes:

- 0: normal quit
- 1: fatal error, with the terminal restored and the message on stderr
- 2: usage error

## Configuration file

```toml
last_instance = "work"            # written on each successful switch

[instances.work]
url = "https://devtron.example.com" # a trailing "/" or "/orchestrator" is accepted and stripped
token_file = "~/.devtron-token"   # "~" is expanded
color = "red"                     # optional: a named colour or "#rrggbb"
```

- Unknown keys are kept when the tool rewrites the file.
- The file never contains a token. On load, the tool refuses any key named `token` and says why.

## Token resolution for the active instance

1. `DEVTRON_TOKEN`, only for the instance chosen at launch (`--instance`, or `last_instance`).
   After a switch it no longer applies, and the new instance uses its own file.
2. The instance's `token_file`.
3. `~/.devtron-token`, when `token_file` is not set.

Checks, in order:

- The file exists and is readable, else "cannot read token file {path}".
- Permissions, with a warning when `mode & 0o077 != 0` (FR-005).
- The JWT can be decoded (for `exp` and `email`), else it is treated as an opaque token with an
  unknown expiry.
- `GET /user/check/roles` succeeds (A1).

## First launch (FR-001)

With no config file, or an empty `instances` table, the **Register** form opens. It asks for:

- name
- URL
- token file (pre-filled with `~/.devtron-token`)
- colour (optional)

The form checks the connection before saving. The first registered instance becomes
`last_instance`.
