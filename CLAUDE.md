# CLAUDE.md

Notes for making changes. User-facing docs are in README.md.

## Shape

- One executable file, `adguard-manager` (no `.py`), Python 3.9+, stdlib only. Keep it that way: no deps, no package split.
- No tests, no CI. Check with `python3 -m py_compile adguard-manager` and a manual run against AdGuard Home (tested on v0.107.79).
- Screenshots: `tools/screenshot out.png [KEY ...]` runs the app in a pty against a fake AdGuard Home with sample data and saves a PNG (also prints the screen as text). Keys: `F2`, `Down`, `Enter`, `Esc`, `wait:2`, or text to type. `--new-every 2` adds a query every 2s to show refreshes. Before shot: `git fetch origin main && git show origin/main:adguard-manager > /tmp/old && tools/screenshot --app /tmp/old before.png ...`. Needs `python3 -m pip install pyte pillow` (the tool only; the app stays stdlib). UI-changing PRs attach before/after shots.
- Demo GIF: `tools/demo-gif [out.gif]` drives the same fake server through a fixed script and saves `docs/demo.gif` for the README.
- Smoke test: `tools/smoke` replays a scripted session in a pty (filter, F6 remove/undo, rules add/validate/Ctrl+Z-undo/submit, help, F7 cycle, quit, config migration, `--debug` log) and asserts on the terminal content. Needs pyte only. `.github/workflows/check.yml` runs `py_compile` on 3.9–3.13 plus the smoke test on every push/PR.
- PRs are squash-merged with a short plain-English title, e.g. `Ctrl+Q quits like F10 (#6)`.

## Map (top to bottom)

| Part | What |
|------|------|
| constants | `LOG_LIMIT`, `FIRST_PAGES`, `PAGES`, `KEEP`, `AUTO`, `PAD`, `CLIENT_MAX`, `DETAIL_ROWS`: all tuning lives here |
| `Api` | persistent-connection wrapper over `/control/*` (one reused TCP/TLS connection, lock-guarded for the fetch thread). `request()` returns `(set-cookies, parsed JSON)` and maps 401/403 to `AuthError`, everything else (HTTP, socket, timeout, 429) to `RuntimeError`; a dead idle connection is retried once on a fresh one |
| `load_conf`/`save_conf` | JSON at `$XDG_CONFIG_HOME/adguard-manager.json`, mode 600 (an old `adguard-manager` file is renamed on load until 2026-10-26; then drop `OLD_CONF`). Keys: `profiles` ({name: {url, cookie}}), `profile`, `ignore`, `client` |
| curses helpers | `addstr` (clips, never raises), `footer`/`footer_layout`, `scroll`, `in_thread`, `mouse`, `Field` |
| popups | `login_form` (warns before a cleartext HTTP login to a non-local host), `choose` (filterable list), `ask` (one-line input), `confirm`, `show_help` (key list) |
| `RulesView` | F2 screen: custom rules list; edit/add/delete validate lightly and log `ops` as `(kind, old, new, index)`; F6/Ctrl+Z undoes the last op; Back (`leave()`) asks, then `submit()` |
| log helpers | `verdict`, `client_name`, `qname`, `levels`, `covers`, `clip`, `row_buttons`, `key`, `fetch_new` |
| `LogView` | main screen: query log, details pane, F-key actions, background fetching |
| `main` | colors, mouse, IXON off, login loop, write conf on exit if changed |

## Conventions

- Keys come from `get_wch()`: `str` for characters, `int` for `curses.KEY_*`. Check both forms where needed (`"\x7f"`, `"\b"`, `KEY_BACKSPACE`; `"\n"`, `"\r"`, `KEY_ENTER`). Ctrl+Q is `"\x11"`.
- Commands live on F-keys only, because any printable key types into the filter. Each view has an `actions` dict `{KEY_Fn: method}` in its run loop; F9/F10 are handled before it. F1 opens the key help in both views.
- Adding a key means updating three places: the `actions` dict, `keys()`/`KEYS` (footer), and the README key table plus the module docstring (and the `HELP_*` list if it belongs there).
- Ctrl+Z is repurposed for undo in the rules view: `main` disables the tty suspend key (`VSUSP`), so it no longer suspends the app while it runs. Document this wherever keys are listed.
- F7 cycles the log view through hide-ignored / show-ignored / blocked-only (`self.view`); the footer label names what F7 switches to next.
- Footer labels must be `"F<n>"` when the list is passed to `mouse()`; it turns a footer click into `KEY_F0 + int(key[1:])`. `login_form` uses other labels and does not pass them to `mouse()`.
- Errors: actions raise `RuntimeError` and the loop shows it in `self.msg`. `AuthError` is not caught in views; it goes up to `main`, which drops the cookie and shows the login form again.
- Status text goes in `self.msg` (right side of the footer). Messages are prepended like `"Ignoring x; " + self.msg` after `apply()` rewrites it.
- After a popup closes, `choose`/`ask` call `scr.touchwin()`, so the caller just redraws.
- Mouse: a row click sets `self.pad = 0` for one draw so the clicked row doesn't move. Buttons (`row_buttons`) show only on the selected row and map to F4/F6. On the selected row a client click calls `set_client`, a domain click sets `self.part = (entry, suffix_at(...))`, which F4/F6 use instead of their menu; a second click on that part offers F6/F4 in a `choose` popup; any other click on the row clears it, and `draw()` drops it once the selection moves. `self.on_client` is the same for a picked client. Left/Right/Enter go through `row_key()`, which returns an F key to run (clicks reuse it by turning into `"\n"`).

## LogView data flow (the tricky part)

- `cache[search] -> entries`, newest first. `search` is the client IP while filtering by client (the server search also matches domains, so `apply()` filters on the exact IP too), else `""`. `all` = `cache[search]`.
- `apply(keep, jump)` rebuilds `rows` from `all` in one pass (it runs on every key typed, over up to `KEEP` entries): client filter, ignore list (`ignored()` against a set, skipped when `view == 1`), blocked-only (`view == 2`), typed filter. `keep` is found by identity. Call it after anything that changes `all`, `ignore`, `view`, `typed` or `client`.
- `set_client` seeds `cache[ip]` from `cache[""]` (and `more[ip]` from its oldest time), so a client filter shows at once and only fetches the gap and older pages.
- Loading: `got_first` (20 rows) -> `got_more` pages of 100 until `LOG_LIMIT`, with `more[search]` holding the `older_than` cursor. `refresh()`/`fetch_new` go backwards from newest and stop at the first known `key()`; if the gap is more than `KEEP` the list is replaced.
- Background: `wait_ms()` sets the `get_wch` timeout, and on timeout `idle()` either starts one job via `in_thread` or collects it. `self.job = (kind, search, base list, Future)`. The result is thrown away if the view switched or the list object changed (`is not base`), so a manual F5 always wins.
- F-key actions (F5, F6, rules) make network calls on the main thread and block. Only idle fetches run in a thread. A view that blocks for long (`RulesView.run`) calls `scr.timeout(-1)`, and `LogView.run` resets it on exit for the login form.
- F6 stores undo state on the entry dict itself: `f6` (ops list) and `was_reason`/`was_rules`/`was_service_name`. Prepending new entries keeps the same dicts; a full list replacement loses it. One F6 action is one `set_user_rules` write: the need for the new rule is decided from a pre-write `check_host` plus the query-time rules before anything is sent.
- Rule edits always re-read rules from the server first (`user_rules()`), change that list, then `set_user_rules()`, so web UI edits made meanwhile are kept. Keep this when adding rule operations. `RulesView` batches: it compares the fresh list to `saved` (as loaded) and, if they differ, replays `ops` by rule text.
- `check_host` matches by name only; answer-IP rules don't show there, so `toggle_block` still adds the allow rule when other rules matched. Timestamps are shown in the local timezone (`local_time`/`local_datetime`); unparseable ones fall back to the raw slice.
- A failed first-load page keeps its `older_than` cursor and is retried after a backoff (`more_backoff`: 5s, 10s, … capped at 60s) instead of abandoning the load.

## Drawing

- `LogView.draw`: header row, list (`body` rows), `─ Details ─` pane (`DETAIL_ROWS`, grows for wrapped text, dropped when the screen is too short), footer on the last line. The client column width fits the visible names (also while filtering by client), capped at `CLIENT_MAX`; it grows at once, but shrinks one cell per second and only while scrolling (`cw`, `cw_at`).
- `addstr` drops the last cell of the bottom row (curses errors writing the bottom-right corner). Always draw through it, never `win.addstr` directly.
- Colors: pair 1 red (BLOCK/errors), 2 green (ALLOW, `@@` rules), 3 cyan (REWR), 4 grey (ignored rows, shown with F7); default background.

## Disk/network goals (keep them)

- The config file is written only at login and, on exit, when it differs from disk. No writes on refresh.
- Small first fetch, small refresh pages; never refetch what's cached.
- One persistent HTTP connection per `Api` (keep-alive), shared with the fetch thread under a lock.
- `--debug` logs method/path/status/timing to `~/.config/adguard-manager-debug.log`, never request bodies (the login one carries the password).
