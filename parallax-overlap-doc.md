# 视差叠压过渡效果 — AI 实现文档

## 效果描述

白底页面之间的过渡动效。当用户向下滚动时，下一个页面像一张纸从屏幕底部推上来，
逐渐压住当前页面。当前页面同时轻微缩小并向上退，产生"纸张叠在桌上"的物理纵深感。
全程无任何中间颜色覆盖，只靠位移与缩放的速度差实现层次感。

---

## HTML 结构

```html
<!-- 每个页面由 .scene + .panel 组成 -->
<!-- .scene 撑开滚动空间，.panel 是可见的白色页面 -->

<div class="scene">          <!-- height: 200vh，制造停留+过渡空间 -->
  <div class="panel">        <!-- position: sticky; top: 0; height: 100vh -->
    <!-- 页面内容 -->
  </div>
</div>

<div class="scene">
  <div class="panel">
    <!-- 下一页内容 -->
  </div>
</div>

<!-- 最后一个 scene 只需 height: 100vh -->
```

---

## 核心 CSS

```css
/* 场景容器：用高度制造滚动停留时间 */
.scene {
  position: relative;
  height: 200vh;   /* 100vh 停留 + 100vh 过渡。改成 250vh 可让过渡更慢 */
}
.scene:last-child {
  height: 100vh;   /* 最后一页不需要延长 */
}

/* 白色面板 */
.panel {
  position: sticky;
  top: 0;
  height: 100vh;
  width: 100%;
  background: #f9f9f9;
  overflow: hidden;
  will-change: transform;
  transform-origin: center bottom; /* 缩放锚点在底部，退场更自然 */
}
```

---

## 核心 JS 算法

每一帧只做两件事：

**① 退场面板（panel[i]）**：随滚动进度缩小 + 上移
```js
// progress: 0 → 1，代表"这个 scene 的过渡区滚完的百分比"
const sceneTop   = scene.getBoundingClientRect().top + window.scrollY;
const overscroll = scene.offsetHeight - window.innerHeight; // = 100vh
const progress   = clamp((scrollY - sceneTop) / overscroll, 0, 1);

panel.style.transform = `scale(${lerp(1, 0.92, progress)}) translateY(${lerp(0, -3, progress)}%)`;
```

**② 入场面板（panel[i+1]）**：从屏幕下方推入
```js
// 同一个 progress 同步驱动入场
nextPanel.style.transform = `translateY(${lerp(100, 0, progress)}vh)`;

// 可选：入场时顶边有轻微阴影，落定后消失
const shadow = lerp(0, 0.06, progress) * lerp(1, 0, Math.max(0, (progress - 0.8) / 0.2));
nextPanel.style.boxShadow = `0 -12px 40px rgba(0,0,0,${shadow})`;
```

完整工具函数：
```js
const clamp    = (v, lo, hi) => Math.min(Math.max(v, lo), hi);
const lerp     = (a, b, t)   => a + (b - a) * clamp(t, 0, 1);
```

滚动监听：
```js
window.addEventListener('scroll', update, { passive: true });
```

初始化（关键）：
```js
// panel[0] 在屏幕上，其余全部在屏幕下方待命
panels.forEach((p, i) => {
  p.style.transform = i === 0 ? 'scale(1) translateY(0)' : 'translateY(100vh)';
  p.style.transition = 'none'; // 必须关闭，否则会有弹跳
});
```

---

## 可调参数

| 参数 | 推荐值 | 说明 |
|---|---|---|
| `scene height` | `200vh` | 越大过渡越慢 |
| `scale 终点` | `0.92` | 越小退场感越强，`0.96` 几乎不可见，`0.88` 很夸张 |
| `translateY 终点` | `-3%` | 退场面板向上移动的幅度 |
| `shadow alpha` | `0.06` | 入场阴影强度，`0` 可完全去掉 |
| `transform-origin` | `center bottom` | 改成 `center center` 会从中心缩小 |

---

## 扩展方向

**加内容视差**：内容与面板的移动速度不一致，增加层次
```js
// 内容比面板慢 30%，产生视差
panelContent.style.transform = `translateY(${lerp(0, -8, progress)}%)`;
```

**退场时内容淡出**：
```js
panel.style.opacity = lerp(1, 0.6, progress);
```

**入场时内容从下方微移入**：
```js
nextPanelContent.style.transform = `translateY(${lerp(4, 0, progress)}%)`;
nextPanelContent.style.opacity   = lerp(0, 1, Math.max(0, (progress - 0.3) / 0.7));
```

---

## 注意事项

1. `transition: none` 必须加在 panel 上，JS 帧驱动不能有 CSS 过渡干扰
2. `will-change: transform` 开启 GPU 合成层，防止重排
3. `scroll-behavior: auto`（或不设置），不要用 `smooth`，会导致 JS 读取位置滞后
4. 最后一个 `.scene` 高度只设 `100vh`，否则页面底部会有多余空白
5. `transform-origin: center bottom` 让缩放从底部开始，视觉上更像"往桌上放"而不是"在空中缩小"
