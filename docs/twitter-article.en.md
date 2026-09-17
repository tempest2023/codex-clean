# X Article draft (English)

> Suggested title: **"The Dismissible Banner That Wasn't"**
> Alternate: **"No setting turns it off, so I wrote 200 lines"**

---

The moment your quota runs out, the Codex desktop app pins a card to the interface:

```text
You’re out of Codex and Work usage
Your rate limit resets on Sep 19, 1:35 AM.
Use one of your rate limit resets or add credits to continue now.
```

That is not a bug. Telling me I am out of usage is fair. The problem is that it stays there, and I
see it every time I open the app.

So I set one rule for myself before touching anything: **do not change server logic, do not bypass
the rate limit, hide only this local UI element.**

The result is `codex-clean`, a small macOS utility of a bit over 200 lines. Getting there involved
three dead ends and one rewrite. The process is more interesting than the script, so here it is.

## 1. There is no official setting

First stop: the config file. Codex has `config.toml`, and it does contain something that looks
right:

```toml
[notice]
hide_rate_limit_model_nudge = true
```

Nothing. That only hides the nudge suggesting you switch models when you are rate limited.

```toml
[tui]
notifications = false
```

Also nothing. That only affects notifications.

Conclusion: this banner cannot be disabled by configuration. It is UI, so it lives in the DOM.

## 2. Get into the Electron process

ChatGPT / Codex desktop is an Electron app, so it can be launched with a debugging port:

```bash
open -n /Applications/ChatGPT.app --args --remote-debugging-port=9222
```

Attach to `localhost:9222` through `chrome://inspect` and you get a pile of renderers:

```text
sandbox
ChatGPT  app://-/index.html
ChatGPT  app://-/detached-window.html
```

Only `app://-/index.html` renders the main UI. Attach to the wrong one and you get "the text is
clearly on screen, but `innerText` does not contain it."

## 3. Dead end one: the apostrophe

In the right renderer:

```js
document.body.innerText.includes("You're out of Codex and Work usage")
```

`false`.

But the text is right there on screen.

The app renders it with a typographic apostrophe:

```text
You’re     <- what the UI renders
You're     <- what I typed
```

That dead end produced a permanent decision: never match the quote character again. Match only:

```js
"out of Codex and Work usage"
```

## 4. Dead end two: I hid the wrong layer

Version one found the text node, walked up to a container that also held `Reset usage` and
`Add Credits`, and set `display: none`.

Half success. The text was gone, the buttons were gone, but the rounded border, the usage icon,
and the empty space were all still there.

I had hidden the card's **inner content** instead of the card. The outer container was still
rendered, just emptied out, which looked worse than before.

Walking further up found the real wrapper:

```html
<aside class="relative isolate flex w-full ... rounded-3xl shadow-xs">
</aside>
```

The whole thing collapsed into one line:

```js
document.querySelector("aside")?.style.setProperty("display", "none", "important");
```

Text, buttons, icon, border, and the reserved space disappeared together. Visually it was as if
the card had never existed.

## 5. Dead end three: React puts it back

The hide worked, until the next render. Anything you hand-edit in a React tree gets overwritten.

The fix is a `MutationObserver` rather than guessing at render timing.

There is one deliberate tradeoff here: **do not match on CSS classes.** Matching the Tailwind
class list would be easier, but that list changes with every frontend build, and when it changes
the script fails silently. So only two conditions remain:

```text
element is <aside>
+
text contains a usage marker
```

Matching text instead of structure is the thing that keeps this alive longer.

## 6. The real bug: switching chats brought it back

Version one worked. Then I switched chats:

```text
current chat:   banner hidden ✓
another chat:   banner back   ✗
```

Version one assumed the renderer that existed at launch would keep existing. Codex creates or
replaces renderers when you switch chats, and a brand new `app://-/index.html` was never injected.

The bug was not weak hiding logic. The bug was that **injection happened exactly once.**

So version two turned injection into a resident service: a background Node watcher polls CDP
targets every 750 ms and injects into every new renderer. Inside the renderer, four layers keep it
gone:

```text
1. new renderer watcher   polls CDP targets every 750 ms
2. MutationObserver       re-hides whenever React rebuilds the banner
3. SPA route changes      pushState / replaceState / popstate / hashchange
4. 1 s safety sweep       full rescan even if the layers above miss
```

## 7. Where the line is

Be clear about what this is: **a local UI customization.**

What it does not do:

```text
add usage
bypass a rate limit
modify account quota
modify server state
reset usage
```

All it does is `display: none`. Server-side limits behave exactly as before. You wait when you
have to wait. The card just stops being in front of your face.

Two honest caveats:

1. The CDP port binds to `127.0.0.1`, so it is not exposed to your network, but any local process
   on the same Mac can still reach it. Use it as a development tool, not a leave-it-on config.
2. This targets unreleased UI. A redesign can break it. Because it matches text rather than class
   names, the fix is usually one line.

## Two things worth keeping

**Match semantics, not styling.** Class names and DOM structure are implementation details. Text
is closer to intent, and it survives redesigns far more often.

**"Works" and "holds up" are different problems.** Version one worked perfectly inside one
session, right up until I did something completely ordinary: switched chats. What settled the
design was not the hiding logic, it was recognizing that injection happened only once.

Code is here, MIT, use it and change it:

https://github.com/tempest2023/codex-clean

---

## Publishing notes (do not paste into the article)

- Suggested images: (1) the original banner with `Reset usage` / `Add Credits`, (2) the `<aside>`
  highlighted in DevTools, (3) `tail -f watcher.log` showing `injected: app://-/index.html`.
  The DevTools shot explains the whole thing best.
- Single-tweet version:
  > Codex Desktop's "out of Codex and Work usage" banner has no official off switch. I hooked into
  > Electron over CDP, found the `<aside>`, set `display:none`, then added a resident watcher to
  > survive React re-renders and chat switches. 200 lines, MIT: https://github.com/tempest2023/codex-clean
