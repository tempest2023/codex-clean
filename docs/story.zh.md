# codex-clean 是怎么做出来的

一份完整的调试记录。三个死胡同、两次改架构，以及最后为什么它只剩下一个 DOM 操作。

---

## 一句话版本

Codex / ChatGPT Desktop 在你用完 Codex 和 Work 额度之后，会在界面里常驻一个提示框。官方配置关不掉它。最后我用 Electron 的调试协议连进渲染进程，找到那个 `<aside>`，把它 `display: none`，再想办法让这件事在 React 重渲染和切换 chat 之后依然成立。

它不碰额度，不碰服务端，只隐藏本地的一个 UI 元素。

---

## 起点：一个关不掉的框

额度用完之后，界面上会出现这张卡片：

```text
You’re out of Codex and Work usage
Your rate limit resets on Sep 19, 1:35 AM.
Use one of your rate limit resets or add credits to continue now.

[ Reset usage ]   [ Add Credits ]
```

问题不是"提醒我额度用完了"，这很合理。问题是它会一直挂在那里，占着侧边栏，每次打开都要看见一遍。

所以我一开始给自己定了四条约束：

1. 不改服务端额度逻辑；
2. 不绕过 rate limit；
3. 只隐藏这个 UI 提示框；
4. 要自动生效，而不是每次手动打开 DevTools 粘贴一段 JS。

第 4 条是真正决定后面所有设计的约束。

---

## 第一次尝试：官方配置（死胡同）

第一反应是找官方开关。Codex 有 `config.toml`，里面确实有看起来相关的项：

```toml
[notice]
hide_rate_limit_model_nudge = true
```

没用。它只隐藏"额度用完后建议你换模型"的那条提醒，不隐藏这张 usage 卡片。

又试了：

```toml
[tui]
notifications = false
```

也没用，那只影响通知，不影响这个 Desktop UI banner。

结论：这个提示框没有任何官方配置可以关掉。于是转向渲染层——既然它是 UI，那它就在 DOM 里。

---

## 转向：用 CDP 连进 Electron

ChatGPT / Codex Desktop 是 Electron 应用，所以可以带着调试端口启动：

```bash
APP="/Applications/ChatGPT.app"
[ -d "$APP" ] || APP="/Applications/Codex.app"

open -n "$APP" --args --remote-debugging-port=9222
```

然后在 Chrome 里打开 `chrome://inspect/#devices`，把 `localhost:9222` 加进去，就能看到一堆 renderer：

```text
sandbox
ChatGPT  app://-/index.html?initialRoute=...
ChatGPT  app://-/index.html
ChatGPT  app://-/detached-window.html
```

这里有个不起眼但重要的点：真正承载主界面的是 `app://-/index.html`，不是 `sandbox`，也不是 `detached-window`。连错目标，就会得到"界面上明明有这段文字，但 `document.body.innerText` 里找不到"的结果。

---

## 坑 1：找不到那段文字（弯引号）

第一次在正确的 renderer 里执行：

```js
document.body.innerText.includes("You're out of Codex and Work usage")
```

返回 `false`。

但那行字明明就在屏幕上。

原因很朴素：界面用的是 Unicode 弯引号。

```text
You’re     ← 界面上的
You're     ← 我在代码里打的
```

换成弯引号之后立刻变成 `true`。

这个坑换来一条永久性决定：**后面所有匹配都不再包含引号部分**，统一只匹配这个子串：

```js
"out of Codex and Work usage"
```

不碰引号，就不会再被引号咬到。

---

## 坑 2：隐藏错了层级

有了正确的匹配条件，第一版隐藏逻辑是"找到含这段文字的元素，再往上找同时包含 `Reset usage` 和 `Add Credits` 的节点"，然后 `display: none`。

结果是半成功：

- 文字没了；
- 按钮没了；
- 但**圆角边框还在、usage 图标还在、整块空白占位也还在**。

说明我隐藏的是卡片**内部的内容节点**，而不是卡片本身的外层容器。外层容器仍然在那里，只是内容被抽空了，于是呈现出更难看的空壳。

继续顺着 DOM 往上找，才找到真正的 wrapper：

```html
<aside
  class="relative isolate flex w-full overflow-hidden
         border bg-surface text-sm text-pretty
         rounded-3xl shadow-xs ..."
>
</aside>
```

结构是这样的：

```text
<aside>                        ← 真正要隐藏的东西
  ├── usage icon
  └── usage 内容
      ├── You’re out of Codex...
      ├── Reset usage
      └── Add Credits
</aside>
```

于是隐藏方式变成一句话：

```js
usageInner.closest("aside")
  ?.style.setProperty("display", "none", "important");
```

这一次，文字、按钮、图标、圆角外框、占位空间，一起消失。视觉上和这张卡片从未存在过一样。

---

## 坑 3：React 重渲染之后它又回来了

隐藏成功，但只维持到下一次渲染。

这是 React 应用的典型问题：手动改的 DOM 会被下一次渲染覆盖回去。解决办法不是去猜渲染时机，而是加一个观察者：

```js
new MutationObserver(hideUsageBanner).observe(
  document.documentElement,
  { childList: true, subtree: true, characterData: true }
);
```

这里有一个刻意的设计选择：**不用 CSS 类名做匹配**。

因为最省事的写法其实是匹配那一串 Tailwind 类：

```text
rounded-3xl bg-surface flex ...
```

但这类类名会随前端每次构建而变化。一旦变化，脚本就静默失效，而且失效得很难察觉。所以只保留两个匹配条件：

```text
元素是 <aside>
+
文本包含 usage marker
```

这两个条件的半衰期比类名长得多。

---

## v1：能用了

到这一步，第一版 `codex-clean` 成形：

```text
启动 ChatGPT / Codex
        ↓
开启 localhost:9222 CDP
        ↓
找到 app://-/index.html renderer
        ↓
Runtime.evaluate 注入 JS
        ↓
安装 MutationObserver
        ↓
隐藏 usage banner
```

用法：

```bash
codex-clean            # 正常启动
codex-clean --restart  # 如果 App 已经在跑
```

支持 `ChatGPT.app` 和 `Codex.app` 两种路径，也支持用 `CODEX_CLEAN_APP` 环境变量指定。

到这里本可以收工了。然后我切换了一下 chat。

---

## v2 的由来：切换 chat 之后它复活了

现象很具体：

```text
当前 chat：        banner 消失 ✓
切换到另一个 chat：banner 又出现 ✗
```

原因是 v1 的架构里有一个隐含假设：**脚本启动时存在的那个 renderer，会一直存在。**

但 Codex 切换 chat 时可能会新建 renderer、替换 renderer，或者重新创建页面上下文。新出现的 `app://-/index.html` 从来没有被注入过，自然就带着完整的 banner。

所以问题不在"隐藏逻辑不够强"，而在"注入这件事只做了一次"。

---

## v2：常驻 watcher + 四层防护

改法是把"注入"从一次性动作变成常驻服务。`codex-clean` 现在会在后台拉起一个 Node watcher：

```text
codex-clean
    ↓
启动 ChatGPT / Codex + CDP
    ↓
后台启动 Node watcher
    ↓
每 750ms 拉取一次 CDP targets
    ↓
发现新的 app://-/index.html
    ↓
立刻注入 blocker
```

同时在 renderer 内部，保护也变成四层：

**第一层 · 监听新 renderer**
每 750ms 轮询 `http://127.0.0.1:9222/json`，只要出现新的 `app://-/index.html` 就注入。这一层专门解决切换 chat 的问题。

**第二层 · MutationObserver**
只要 React 把 banner 加回来、或者重建这段 DOM，立刻重新扫描并隐藏。

**第三层 · SPA 路由变化**
Codex 是单页应用。`history.pushState`、`history.replaceState`、`popstate`、`hashchange` 都被接管，路由一变就重新执行隐藏逻辑。

**第四层 · 1 秒兜底扫描**
即使前面三层因为某种原因没捕获到，每隔 1 秒也会全量重扫一次。

只要页面上出现：

```html
<aside> ... out of Codex and Work usage ... </aside>
```

它就会被隐藏。

这里还有一个细节：watcher 注入时会同时调用 `Page.addScriptToEvaluateOnNewDocument`。这一步是为了让**这个 renderer 之后再创建的文档**也自动带上 blocker，而不只是当前这一个。

---

## 边界与安全

这个方案的定位很清楚：**本地 UI customization**。

它不做的事：

```text
增加 Codex usage
绕过 rate limit
修改账号额度
修改服务端状态
重置 usage
```

它做的全部事情就是：

```js
aside.style.display = "none";
```

服务端的额度限制照常生效，该等就等，该 reset 就 reset。只是这张卡片不再出现在我眼前。

关于调试端口：脚本绑定的是 `127.0.0.1:9222`，不是 `0.0.0.0:9222`，所以不会暴露到局域网。但要诚实地说，同一台 Mac 上的其他本地进程理论上仍然可以访问这个端口。这也是为什么它适合作为本地开发工具使用，而不是一个"开着就别关"的常驻配置。

还有一点需要提前说明：这是针对未发布 UI 的做法，Codex 前端改版就可能失效。但因为匹配的是文本而不是类名，通常失效的时候，修法是改一行。

---

## 时间线

```text
1. 官方配置关不掉 banner              → 死胡同
2. 用 CDP 连进 Electron renderer      → 找到正确的目标
3. 匹配 "You're" 失败                 → 发现是弯引号
4. 隐藏内容节点                       → 留下空壳
5. 找到外层 <aside>                   → 干净隐藏
6. 加 MutationObserver                → 抗 React 重渲染
7. 切换 chat 后 banner 复活           → v1 架构缺陷
8. 常驻 watcher + 四层防护            → v2 最终方案
```

---

## 两条值得记下来的教训

**匹配语义，不要匹配样式。** 类名和 DOM 结构都是实现细节，文本才更接近意图。选择匹配文本，让这个东西在改版之后仍然有较大概率继续工作。

**"能用"和"可靠"是两个不同的问题。** v1 在单次会话里完全能用，直到我做了那个再普通不过的动作——切换 chat。真正把方案定下来的不是隐藏逻辑，而是"注入只做了一次"这个架构判断。
