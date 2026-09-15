# Fuwari 侧边栏网易云音乐播放器

给 [Fuwari](https://github.com/saicaca/fuwari) 主题的侧边栏加一个音乐播放器，**歌单数据来自网易云，不需要任何后端服务**。

```
┌──────────────────────────┐
│ ▍音乐                     │
│  ⬤   曲名                 │
│      歌手                  │
│      0:00 / 4:45  🔊 ▬▬   │
│  ──────────────────────   │
│  ⇄   ⏮   ▶   ⏭   ☰       │
│  ┌ 歌单（展开/收起）┐      │
│  └────────────────┘       │
└──────────────────────────┘
```

配色全部使用 Fuwari 已有的主题变量，跟随主题色切换、自动适配明暗模式。

## 原理

核心是绕开"网易云接口不能跨域、所以要搭后端"这个前提。

```
构建时（服务器上 pnpm build）
  fetch 歌单接口  →  曲目列表内联进 HTML
        ↓
运行时（访客浏览器）
  <audio src="https://music.163.com/song/media/outer/url?id=<ID>.mp3">
  直接向网易云 CDN 取流
```

两个关键事实：

1. **`<audio>` 不受 CORS 限制。** HTML 媒体元素允许加载跨域资源（只有用 Web Audio API 处理音频数据时才需要 CORS）。网易云的外链接口 `song/media/outer/url?id=<ID>.mp3` 会 302 跳转到真实 CDN 地址，实测不校验 `Referer`，可以直接作为 `<audio src>`。
2. **歌单接口在构建时抓。** 网易云歌单接口没有 CORS 响应头，浏览器不能直接调；但在 Node 里（构建期）没有跨域限制。

因此：**不需要部署后端、不需要服务器中转音频**。访客的浏览器直接与网易云 CDN 通信。

## 依赖

- Fuwari（Astro 5 + Svelte 5 + Tailwind）
- `@iconify/svelte`（Fuwari 已自带）
- `@iconify-json/material-symbols`（Fuwari 已自带）

## 安装

把两个文件放到 `src/components/widget/`：

```
src/components/widget/
├── MusicPlayer.astro          # 构建期抓歌单，外层卡片
└── MusicPlayerClient.svelte   # 客户端播放器
```

然后在 `src/components/widget/SideBar.astro` 里挂上（位置随意，这里放在「分类」下面）：

```astro
import MusicPlayer from "./MusicPlayer.astro";

<Activity ... />
<Categories ... />
<MusicPlayer></MusicPlayer>
<Tag ... />
```

## 配置

改 `MusicPlayer.astro` 顶部的歌单 ID：

```js
const PLAYLIST_ID = "18254065221";
```

歌单 ID 就是网易云歌单链接里的 `id` 参数：

```
https://music.163.com/#/playlist?id=18254065221
                                  ^^^^^^^^^^^
```

歌单必须是**公开**的。

## 实现要点

### 构建期抓歌单

```js
const res = await fetch(
  `https://music.163.com/api/v6/playlist/detail?id=${PLAYLIST_ID}&n=1000`,
  {
    headers: {
      "User-Agent": "Mozilla/5.0 ...",
      Referer: "https://music.163.com/",
    },
  },
);
const songs = (await res.json()).playlist.tracks.map((t) => ({
  id: t.id,
  name: t.name,
  artist: (t.ar ?? []).map((a) => a.name).join(" / "),
  // 接口返回 http 地址，HTTPS 页面会拦混合内容，必须升级
  cover: String(t.al?.picUrl ?? "").replace(/^http:/, "https:"),
  duration: t.dt ?? 0,
}));
```

拿到的数组作为 props 传给 Svelte 组件。抓取失败时降级显示一行提示，不会让整个构建挂掉。

### 歌单展开动画

用 `grid-template-rows: 0fr → 1fr` 做高度过渡，不需要预先知道内容高度：

```css
.mp-drawer {
  display: grid;
  grid-template-rows: 0fr;
  opacity: 0;
  transition:
    grid-template-rows 0.3s cubic-bezier(0.4, 0, 0.2, 1),
    opacity 0.3s cubic-bezier(0.4, 0, 0.2, 1);
}
.mp-drawer.open {
  grid-template-rows: 1fr;
  opacity: 1;
}
.mp-drawer-inner {
  overflow: hidden;
  min-height: 0; /* 必须，否则 grid 子项不会收缩 */
}
```

比 `max-height: 0 → 999px` 更好：后者要么动画速度不均，要么得猜一个足够大的值。

### 封面旋转

播放时旋转，暂停时**停在当前角度**：

```css
.mp-cover img {
  animation: mp-spin 12s linear infinite;
  animation-play-state: paused; /* 动画常驻，仅暂停 */
  transform-origin: center;
}
.mp-cover.spinning img {
  animation-play-state: running;
}
```

注意动画必须始终挂着，只切换 `animation-play-state`。直接增删类名会导致 `animation` 被移除，旋转角度重置回 0°。

### 播放状态由事件驱动

`playing` 只由 `<audio>` 的 `onplay` / `onpause` 事件驱动，不做乐观赋值——否则 `play()` 被拒（自动播放策略、网络失败）时界面会与实际状态不一致。

需要"切歌后自动播放"的意图时，用非响应式的 `autoplay` 标记，而不是读响应式的 `playing`：

```js
let autoplay = false;

$effect(() => {
  const _ = src; // 只依赖 src
  currentTime = 0;
  realDuration = 0;
  if (autoplay) audioEl.play().catch(() => {});
});
```

在 Svelte 5 中，`$effect` 会收集函数体内**所有**被读取的响应式变量作为依赖。如果这里读 `playing`，那么每次暂停/播放都会触发重置。

### 拖动与键盘

进度条和音量条用 Pointer 事件 + `setPointerCapture` 实现拖动（鼠标与触屏通用）：

```js
function startDrag(e, el, onMove) {
  e.preventDefault();
  const rect = el.getBoundingClientRect();
  const fire = (ev) =>
    onMove(Math.min(Math.max((ev.clientX - rect.left) / rect.width, 0), 1));
  fire(e);
  el.setPointerCapture(e.pointerId); // 指针移出元素后仍能继续拖
  // ...绑定 pointermove / pointerup / pointercancel
}
```

`setPointerCapture` 是关键：否则指针一旦移出那个细条，拖动就断了。

键盘支持：`←`/`→` 步进 5 秒或 5%，`Shift` 加速到 30 秒，`Home`/`End` 跳首尾。CSS 里加 `touch-action: none` 避免触屏拖动时滚动页面。

### 错误处理

网易云部分歌曲因版权、地区或下架取不到播放地址。`<audio>` 的 `onerror` 里跳到下一首，并累计失败次数——全部失败时停止，避免无限循环。

## 已知限制

- **加歌需要重新构建。** 歌单是构建时抓取的，往网易云歌单添加歌曲后需要重新构建部署一次。
- **首次加载图标有延迟。** `@iconify/svelte` 默认在运行时向 `api.iconify.design` 请求图标数据（实测约 3 秒），首次访问会先看到空白的控制按钮。可以在构建期把用到的图标内联进页面来消除。
- **不校验歌曲可用性。** 构建时不会预先探测每首歌能否播放，遇到失效歌曲由前端在运行时跳过。
- 仅在 Fuwari 主题上测试过。

## 许可

MIT
