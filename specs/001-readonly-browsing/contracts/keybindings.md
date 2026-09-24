# Contract: Views and key bindings

This is the UI contract for FR-018 and FR-019 and for constitution IV. `snapshots.rs` and
`update.rs` check it.

## Global keys (never rebound by a view)

| Key | Action |
|---|---|
| `:` | Command bar. `:apps`, `:instances` (alias `:ctx`), `:help`, `:q`. Tab completes. |
| `/` | Filter the current list, or search in a log view. `Esc` clears it. |
| `Enter` | Open the selected item |
| `Esc` | Back (one level up the view stack). Does nothing on the root view. In the filter bar, command bar or a dialog: cancel. |
| `?` | Help: all keys, global and for the current view |
| `r` | Refresh the current view (FR-022) |
| `q` | Quit, from any view (constitution IV). While typing in the filter bar, command bar or a dialog field, `q` is text. |
| `Ctrl-C` | Quit from anywhere |
| `j`/`k`, `↓`/`↑`, `PgDn`/`PgUp`, `g`/`G` | Move, page, top, bottom |

## Navigation

```text
:instances ──Enter──▶ (switch) ──▶ Apps
Apps ──Enter──▶ App detail (environments)
App detail ──h──▶ History(env) ──Enter──▶ Deployment (stages) ──Enter──▶ Log viewer (stage)
App detail ──b──▶ Builds [pipeline selector if >1] ──Enter──▶ Log viewer (build)
App detail ──p──▶ Pods(env) ──Enter──▶ [container picker dialog if >1] ──▶ Log viewer (pod)
```

`Esc` goes back through the stack. Going back to Apps keeps its filter and selection (US2, scenario 4).

## View-specific keys (shown in the footer)

| View | Keys |
|---|---|
| Instances | `a` register (register dialog), `d` remove (confirm dialog, default **No**), `Enter` switch |
| Apps | none beyond the global keys |
| App detail | `h` history, `b` builds, `p` pods (all three act on the selected environment, except `b`, which is per app) |
| Builds | `Tab` next pipeline (when there are several) |
| Pods | `Enter` logs of the first or only container |
| Log viewer | `f` toggle follow, `n`/`N` next/previous match, `c` switch container (pod logs), `P` previous logs of this container (FR-027), `w` toggle line wrap |

## Always-visible chrome

- **Header**: `⎈ {instance name}` in the instance colour · `{email}` · `token until {date}` (red when
  expired or under 7 days) · breadcrumb of the view stack · `READ-ONLY` badge
- **Footer**: the keys of the current view, then `? help`
- **Body states**: Loading (spinner and what is loading), Empty (explicit sentence, for example
  "no running pods"), Error (cause plus `r to retry`), Too small (below 80×24: "terminal too small,
  80×24 needed")
