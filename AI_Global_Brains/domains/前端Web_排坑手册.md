# 前端Web排坑手册 (Domains)

> 绑定浏览器 API、CSS 渲染、HTML5 特性及第三方库的前端踩坑与填坑指南。

## 1. MathJax 排版冲突与加载错位
* **表现**：MathJax 渲染出的根号没有左侧勾，部分连体字母拥挤重叠；绝对值竖线与分数线等符号在垂直方向产生向下的莫名偏移。
* **根因 1 (全局 Reset 污染)**：`* { margin: 0; padding: 0; }` 会强行剥夺 MathJax `<svg>` 及内部图元的自带边距，导致矢量计算失真。
  * **填坑**：必须缩小 CSS Reset 范围至具体的文本 HTML 标签。为了绝对隔离，可以在 CSS 强制给 MathJax 容器重置：`mjx-container[jax="SVG"] { line-height: 0 !important; }`。
* **根因 2 (Web字体时序差)**：若在 Google Fonts 等自定义外部字体（如 Inter）加载生效前，就触发了 MathJax 的 `.typesetPromise()`，MathJax 会根据当前系统的默认 Serif 字体计算相对尺寸，等 Inter 生效后原本对齐的 `ex` 单位基线就会发生错位。
  * **填坑**：必须将页面初始化时的首次 MathJax 渲染包裹在异步锁中：`document.fonts.ready.then(() => { ... })`。
* **根因 3 (局部 DOM 解析混乱)**：局部更新部分元素引发重渲染时容易串台。
  * **填坑**：在 MathJax 配置项中，推荐将 `svg: { fontCache: 'local' }` 以隔绝 DOM 多次拼接复用带来的解析残影。

## 2. HTTPS 与 HTTP 的 Mixed Content (跨站导入拦截)
* **表现**：用户在一个外网的 HTTPS 页面（如 `https://ikuuu.win`）点击了本地注入的书签脚本 (Bookmarklet)，试图通过 `fetch` 或 XHR 往局域网管理端（`http://192.168.1.17:8092`）POST 提交 Cookie。结果网络请求直接失败，毫无反应。
* **根因**：浏览器的安全同源/混合内容策略（Mixed Content），严格禁止在 HTTPS 页面内向 HTTP 地址发起后台通信。
* **填坑**：抛弃后台 `fetch`，改用纯粹的顶层导航跳转。在 JS 脚本中动态创建一个 `<form method="POST" action="http://192.168.1.17:8092/import-cookie-nav" style="display:none">`，塞入 Input 隐藏域，最后执行 `form.submit()`。顶级表单跳转可轻易穿透混合内容拦截。

## 3. Clipboard API 静默失败降级
* **表现**：前端页面提供“点击复制”按钮，调用了 `navigator.clipboard.writeText(text)`。部分用户点击后毫无反应，剪贴板里依然是旧内容。
* **根因**：该 API 严重依赖安全上下文（仅对 HTTPS 或 localhost 开放），在部分环境或权限被用户禁用时会抛出异常。
* **填坑**：必须提供兜底降级方案：
  ```javascript
  navigator.clipboard.writeText(str).then(...).catch(() => {
      // 降级：依赖系统原生 prompt 提供手工全选界面
      prompt("自动复制失败，请手动全选并复制以下内容:", str);
  });
  ```

## 4. 移动端 HTML5 `<video controls>` 遮挡
* **表现**：自定义悬浮于底部的视频操作栏（如“上一集、下一集”），在手机竖屏时，刚刚好严密地覆盖在视频右下角的原生全屏图标之上。
* **填坑**：使用 `@media (max-width: 640px)` 等查询条件。当进入窄屏时，一方面调高控制层的 `bottom` 值（例如 `bottom: 120px`）硬让出空间，另一方面把原本横向 Flex 布局的按钮阵列强制改为 `flex-direction: column` 竖向堆叠，收缩横向控制面积。

## 5. 手机浏览器 Object-Fit 缩放回退
* **表现**：视频打开瞬间显示完好，但一发生 metadata 读取完毕或者拉伸缩放，画面就被截断放大。
* **填坑**：不能只靠初始 CSS。不仅要写明 `width: auto; height: auto; object-fit: contain; max-width: 100%;`，还要单独写一个 `applyVideoDisplayMode()` 的 JS 函数。在 `<video>` 的 `loadedmetadata` 以及 `window.resize` 事件中，用 JS 强行再次覆盖 `video.style.objectFit = 'contain'` 以纠正系统回退。
