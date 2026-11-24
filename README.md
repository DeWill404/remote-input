# remote-input

See and control a Windows PC's screen, mouse and keyboard from another device on the same LAN, through two tiny HTTP endpoints. Built so an AI coding agent (for example Claude Code) on one machine can use apps on a Windows machine the way a human would: look, scroll, click, drag, type. The original use case is debugging a web app in Chrome, but any app on the desktop works.

The input is real OS input (`SendInput` through `pyautogui`), so the browser sees ordinary trusted events. No debugger, extension or browser automation is attached.

```
controller (any OS)                          Windows host
  curl GET  /screenshot  ───────────────▶   server.py  ──▶ PNG of the screen
  curl POST /act {"action":"click",...} ─▶   server.py  ──▶ mouse / keyboard
```

## Files

| file | what it is |
|---|---|
| `server.py` | the server, one file, standard library plus `pyautogui` and `pillow` |
| `start.bat` | starts the server for the controller IP in `REMOTE_INPUT_ALLOW` |
| `skill/remote-input/SKILL.md` | a Claude Code skill that teaches an agent on the controller how to use it |

## Controller

Any device on the LAN that can send an HTTP request with a header and view a PNG: macOS, Linux, Windows, even a phone with an HTTP client. The only restriction is that the server accepts exactly one client IP, the one you start it with.

## Security model

**Treat the token as full control of your Windows account.** The API has only two endpoints, but keyboard input reaches every app: a controller can press `win+r`, type `cmd` and run any command you could run yourself. Anything the logged-in user can do, a token holder can do. The safeguards below limit who gets that access and for how long; they do not limit what it can do once connected.

Because of that, access is locked down on purpose:

- **Manual start only.** No service, no autostart. The server exits by itself after `--minutes` (default 30).
- **Kill switch.** Moving the real mouse into any screen corner stops it immediately (pyautogui failsafe).
- **One client IP.** Enforced twice: by a Windows Firewall rule (Private profile only) and by a check inside the script.
- **Bearer token.** Random, generated fresh on every start, printed only in the console.
- **Two endpoints only.** No API for commands, files or clipboard. (As said above, keyboard input can still reach a terminal, so this narrows the attack surface of the server itself, not what a connected client can do.)

Known limits: traffic is plain HTTP, so anyone who can sniff your LAN can read the token while a session runs. Use it only on a network you trust. IP allow-listing does not stop someone on the same network who spoofs or takes over the allowed IP. Only the primary monitor is captured, and `type` sends ASCII only.

## Setup (Windows host)

1. **Python 3.9+** with the `py` launcher, then:
   ```powershell
   py -3 -m pip install --user pyautogui pillow
   ```
2. **Network profile must be Private.** Check with `Get-NetConnectionProfile`. Only change it (`Set-NetConnectionProfile -InterfaceAlias "<alias>" -NetworkCategory Private`, admin) for a network you trust.
3. **Set the controller's IP** once (open a new window afterwards):
   ```powershell
   setx REMOTE_INPUT_ALLOW 192.168.1.50
   ```
4. **Firewall rule** in an admin PowerShell. Use the real interpreter path, because the `python` on PATH can be a Windows Store stub:
   ```powershell
   $py = py -3 -c "import sys; print(sys.executable)"
   New-NetFirewallRule -DisplayName "Remote Input (controller only)" -Direction Inbound -Action Allow -Protocol TCP -LocalPort 8765 -RemoteAddress 192.168.1.50 -Profile Private -Program $py
   ```
   If Windows shows an "allow Python?" prompt anyway, click Cancel, so it does not add a broad rule. An existing Block rule for `python.exe` on the Private profile overrides this rule; check with `Get-NetFirewallRule | Where-Object DisplayName -like "*python*"`.
5. Optional: put a shortcut to `start.bat` on the Desktop or Start menu. Point the shortcut at `cmd.exe /c "<path>\start.bat"` if you want Windows to offer Pin to Start.

If the controller's IP changes, update both `REMOTE_INPUT_ALLOW` and the firewall rule. A DHCP reservation in the router avoids this.

## Use

On the Windows host run `start.bat` (or `py -3 server.py --allow <controller-ip> [--minutes 60] [--port 8765]`). It prints:

```
URL    http://<windows-ip>:8765
TOKEN  <random string>
```

Send both lines to the controller; the token changes on every start.

From the controller (bash, zsh, Git Bash):

```bash
U=http://<windows-ip>:8765; T=<token>
curl -s -H "Authorization: Bearer $T" "$U/screenshot" -o remote.png
curl -s -X POST -H "Authorization: Bearer $T" -H 'Content-Type: application/json' -d '{"action":"click","x":640,"y":360}' "$U/act"
```

From PowerShell 7.3+, use `curl.exe` with the same single-quoted JSON (do not escape the inner quotes).

Screenshots are downscaled so the long edge is at most 1568 px, and all action coordinates are in that screenshot space; the server converts them to real screen pixels.

| action | fields |
|---|---|
| `move` | `x`, `y`, optional `duration` (seconds, default 0.3) |
| `click` | `x`, `y`, optional `button` (`left`/`right`/`middle`), `clicks` (2 = double-click), `duration` |
| `drag` | `x`, `y` (start), `x2`, `y2` (end), optional `button`, `duration` |
| `scroll` | `x`, `y`, `notches` (negative = down, positive = up) |
| `type` | `text` (ASCII only), optional `interval` |
| `key` | `keys`, a list pressed as one hotkey, for example `["ctrl","l"]`, `["enter"]`, `["f12"]` |

Responses: `200 {"ok": true}`, `400 {"error": ...}` for a bad action, `401` wrong token, `403` wrong client IP.

## Claude Code skill

Copy `skill/remote-input` to `~/.claude/skills/remote-input` on the controller (or to a project's `.claude/skills/`), then:

```
/remote-input open http://localhost:3000 in Chrome, click Login and tell me what the console shows
```

The skill asks for the URL and token, then works in a screenshot, one action, screenshot loop. It only touches the apps and windows the task names, and it will not open a terminal, the Run dialog, Settings, an installer or a UAC prompt unless the task explicitly asks for that. These are instructions the agent follows, not limits the server enforces.

## Tips for the Windows host

- Use a separate Windows account for these sessions, with no saved passwords or personal logins.
- During a session, set sleep and screen lock to Never and keep the screen unlocked. A locked screen gives black screenshots, and UAC prompts cannot be seen or clicked remotely.
- Do not use the machine yourself while the controller is acting; input from both sides lands in whichever window is in front.
