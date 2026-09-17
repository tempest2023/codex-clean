# codex-clean

A tiny macOS utility that hides the **"You're out of Codex and Work usage"** banner in the
ChatGPT / Codex desktop app.

That is all it does. It does not add usage, reset your quota, bypass a rate limit, or touch
any server-side state. It hides one locally rendered UI card.

```text
codex-clean            # launch the app with localhost CDP and start the watcher
codex-clean --restart  # quit the app first if it is already running
```

## Why this exists

When your Codex and Work usage runs out, the desktop app shows a persistent card:

![The usage banner: "You're out of Codex and Work usage", with Reset usage and Add Credits buttons](docs/images/out-of-usage-banner.png)

```text
You're out of Codex and Work usage
Your rate limit resets on <date>.
Use one of your rate limit resets or add credits to continue now.

[ Reset usage ]  [ Add Credits ]
```

There is no official setting to hide it. `[notice] hide_rate_limit_model_nudge = true` only
covers the post-rate-limit model nudge, and `[tui] notifications = false` only affects
notifications. Neither one removes this desktop banner.

`codex-clean` hides it at the renderer level and keeps it hidden across React re-renders,
chat switches, and new windows.

## Requirements

- macOS
- Node.js on `PATH` (used via built-in modules only)
- ChatGPT.app or Codex.app in `/Applications`

## Install

```bash
git clone https://github.com/tempest2023/codex-clean.git
cd codex-clean
mkdir -p ~/.local/bin
cp codex-clean ~/.local/bin/codex-clean
chmod +x ~/.local/bin/codex-clean
```

Make sure `~/.local/bin` is on your `PATH`:

```bash
echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.zshrc
source ~/.zshrc
```

Verify:

```bash
which codex-clean
```

## Usage

```bash
# First run: launches the app with --remote-debugging-port on 127.0.0.1, then starts the watcher.
codex-clean

# App already running without CDP? Let codex-clean restart it.
codex-clean --restart
```

The command returns once the background watcher is running. The banner disappears as soon as
the app's renderer is available.

## Configuration

| Variable | Default | Purpose |
| --- | --- | --- |
| `CODEX_CLEAN_APP` | auto-detected | Absolute path to the app bundle, e.g. `/Applications/Codex.app` |
| `CODEX_CLEAN_PORT` | `9222` | Local CDP port |

```bash
CODEX_CLEAN_APP=/Applications/Codex.app CODEX_CLEAN_PORT=9333 codex-clean
```

## Files it writes

```text
~/.cache/codex-clean/watcher.mjs   # generated watcher
~/.cache/codex-clean/watcher.pid   # PID of the running watcher
~/.cache/codex-clean/watcher.log   # log
```

Follow the log while the app creates new renderers:

```bash
tail -f ~/.cache/codex-clean/watcher.log
```

```text
[2026-09-16T05:31:02.114Z] codex-clean watcher started on http://127.0.0.1:9222
[2026-09-16T05:31:03.887Z] injected: app://-/index.html (A1B2C3...)
```

Running `codex-clean` again stops the previous watcher before starting a new one, so background
processes do not pile up.

## How it works

```text
codex-clean
      |
      +-- launch ChatGPT / Codex with --remote-debugging-address=127.0.0.1
      |
      +-- background Node watcher (every 750 ms)
      |       |
      |       +-- poll http://127.0.0.1:9222/json
      |       +-- find page targets whose URL starts with app://-/index.html
      |       +-- attach over CDP and install the blocker
      |
      +-- inside the renderer
              +-- MutationObserver on document.documentElement
              +-- patched history.pushState / replaceState, popstate, hashchange
              +-- 1 s safety sweep
```

Attaching over CDP shows a pile of targets, and only one kind renders the main UI. This is the
reason the watcher filters on the `app://-/index.html` prefix instead of just taking the first
page it finds:

![chrome://inspect listing the Codex targets, including the app://-/index.html renderers](docs/images/cdp-targets.png)

The blocker is intentionally boring. It walks `<aside>` elements, checks whether the text
contains a usage marker, and sets `display: none !important`:

```js
for (const aside of document.querySelectorAll("aside")) {
  if ((aside.textContent || "").includes("out of Codex and Work usage")) {
    aside.style.setProperty("display", "none", "important");
  }
}
```

It deliberately matches on **text**, not on Tailwind class names, so a styling change in a
future build does not silently break it. If the wrapper element ever stops being an `<aside>`,
a geometry-based fallback looks for the smallest container that is wide enough to be the whole
card.

Matched markers:

```text
out of Codex and Work usage
out of Codex usage
out of Codex messages
```

The app renders the first line with a typographic apostrophe (`You’re`), which is also why
nothing here matches on `You're` with an ASCII quote.

## Safety and limits

What it does:

```text
hides a DOM node in the local renderer
```

What it does not do:

```text
add usage / reset usage / raise a rate limit
modify account or server state
patch the app bundle on disk
change any file inside ChatGPT.app or Codex.app
```

Two honest caveats:

- The CDP port binds to `127.0.0.1`, but any local process on the same Mac can still reach it
  while the app runs with debugging enabled. That is why this is a local development tool, not
  something to leave enabled out of habit.
- This is unreleased UI territory. A future redesign can move or restructure the banner. When
  that happens the matcher stops finding it, and the fix is usually one line.

## Uninstall

```bash
kill "$(cat ~/.cache/codex-clean/watcher.pid)" 2>/dev/null
rm -rf ~/.cache/codex-clean ~/.local/bin/codex-clean
```

Then quit and reopen the app without the debugging flags.

## Unofficial

Unofficial UI customization. Not affiliated with, endorsed by, or supported by OpenAI.

## License

MIT

## How it was built

The full build log, including the dead ends and the reason chat switching broke v1, is in
[docs/story.zh.md](docs/story.zh.md).
