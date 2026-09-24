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
color = "red"                     # optional: a named colour or "#rrggbb"
auth = { kind = "token_file", path = "~/.devtron-token" }   # "~" is expanded
```

`auth` declares how this instance authenticates (constitution II).

- In this feature the only `kind` is `token_file`, whose `path` defaults to `~/.devtron-token`.
  When `auth` is absent, that default is used.
- Any other `kind` is refused on load with "auth kind '{kind}' is not supported by this version".
- Future kinds (an OS keyring, a username/password login) are backlog items. They will add their
  own fields under `auth`, and nothing else in the file changes.

- Unknown keys are kept when the tool rewrites the file.
- The file never contains credentials. On load, the tool refuses any key named `token`, `password`
  or `secret`, at any depth, and says why.

## Credentials for an instance

Credentials come only from the instance's `auth` declaration. No environment variable and no
command-line flag can supply or override them (FR-002). For `kind = "token_file"`, the checks run
in this order:

- The file exists and is readable, else "cannot read token file {path}".
- Permissions, with a warning when `mode & 0o077 != 0` (FR-005).
- The JWT can be decoded (for `exp` and `email`), else it is treated as an opaque token with an
  unknown expiry.
- `GET /user/check/roles` succeeds (A1).

## First launch (FR-001)

With no config file, or an empty `instances` table, the **register dialog** opens. It asks for:

- name
- URL
- credentials: the token file path (pre-filled with `~/.devtron-token`)
- colour (optional)

The dialog checks the connection before saving. The first registered instance becomes
`last_instance`.
