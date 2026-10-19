# adguard-manager

A terminal (curses) front end for the [AdGuard Home](https://github.com/AdguardTeam/AdGuardHome) query log and custom filtering rules. Single Python file, standard library only.

```
Time     Client               Type  Status Domain
12:00:03 laptop               A     BLOCK  ads.example.com
12:00:02 10.0.0.6             A            tracker.foo.net
12:00:00 laptop               A            example.org
───────────────────────── Details ─────────────────────────
     Time 2026-10-18 12:00:03.1-07:00
   Client laptop (10.0.0.5) via DOH
    Query ads.example.com  A
   Result FilteredBlackList
     Rule ||ads.example.com^  [AdGuard DNS filter]
 Response NOERROR: A 0.0.0.0 ttl 10s
 Upstream none  0.4ms
F2Rules F3Client F4Ignore F5Refresh F6Unblock F7Ignored F8Auto 5s F9Logout F10Quit
```

## Install

```sh
install -m755 adguard-manager ~/.local/bin/
adguard-manager
```

Needs Python 3.9 or newer. The first start asks for the server URL (default `http://adguard.lan`), username and password.

## Keys

All commands are on function keys, so typing always filters.

| Key | Main list | Custom rules (F2) |
|-----|-----------|-------------------|
| F2  | open the custom rules manager | back |
| F3  | show one client (pick from a list) / all clients | |
| F4  | ignore a domain or subdomain (hide it in the list) | edit rule |
| F5  | refresh | reload |
| F6  | block / unblock the selected query; again = undo | |
| F7  | list of ignored domains (Enter removes) | add rule (at the end) |
| F8  | auto-refresh interval: 5s, 10s, 30s, off | delete rule (asks first) |
| F9  | log out | |
| F10 | quit | back |

Typing filters the list (domain, client name or IP), Backspace edits, Esc clears. Arrows, PgUp/PgDn, Home/End move. In popups Enter chooses, Esc cancels; typing filters there too.

Mouse: click a row to select it, click its [BLK]/[UBLK] or [IGN] button to press F6 or F4, the wheel moves the selection, click a footer key to press it, click an item in a popup to choose it. Hold Shift to select text with the mouse.

## How it works

- **Login:** `POST /control/login`. Only the `agh_session` cookie is stored, never the password, in `~/.config/adguard-manager` (JSON, mode 600). When the session expires you are asked again.
- **Few disk writes:** the config file is written at login, and on exit only if the ignore list or the client filter changed (or you logged out). Refreshing writes nothing.
- **Little network traffic:** the first load fetches 20 entries (fills the screen), then pages of 100 up to 500 between keypresses. Refresh (F5, or automatic every 5s) asks for 20 entries and stops at the first one it already has; longer gaps take bigger pages. New entries are added on top.
- **Block/unblock (F6):** edits the custom filtering rules. Unblock removes a matching `||name^` rule, and adds `@@||name^` if the name is still blocked; block works the other way. You choose the level (`ads.x.example.com`, `x.example.com` or `example.com`, each with its subdomains). The row then shows AdGuard's current verdict (`/control/filtering/check_host`) and the details keep the original one. F6 again on the same row undoes exactly what it did.
  Note: AdGuard's check is by name only, so rules that match answer IPs are not applied in it.
- **Ignore (F4):** hides queries in this app only; AdGuard keeps logging them.
- **Custom rules (F2):** the rules in stored order (new rules are appended, so oldest first). Each change re-reads the list from the server first, so edits made meanwhile in the web interface are kept.

Tested with AdGuard Home v0.107.79.

## License

GPL-3.0, see [LICENSE](LICENSE).
