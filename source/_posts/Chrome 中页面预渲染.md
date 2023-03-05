---
title: Chrome 中的页面预渲染
---

Chrome 团队一直致力于将用户可能浏览的未来页面的完整预渲染恢复过来。这次预渲染的现代重启将从 Chrome108 开始。

## 预渲染简史

在过去，Chrome 支持 `<link rel="preender" href="/next-page/">` 资源提示，但是在 Chrome 之外并没有得到广泛的支持，而且它也不是一个非常有表现力的 API。

这种通过 `rel=prerender` 提示的遗留预呈现方式不被推荐使用，而是支持 [NoState](https://developer.chrome.com/blog/nostate-prefetch/) 预取，它取得了未来页面所需的资源，但是没有完全预呈现页面，也没有执行 JavaScript。NoState 预取通过改进资源加载确实有助于提高页面性能，但是不会像完整预取那样提供即时页面加载。

Chrome 团队现在准备重新在 Chrome 中引入完整的预渲染。为了避免现有用法的复杂性，并考虑到将来预渲染的扩展，这个新的预渲染机制将不会使用 `< link rel = "preender" ... >` 语法，这种语法仍然适用于 NoState 预取，以便在将来的某个时候退出。

## 页面是如何 prerendered ?

页面可以通过以下三种方式 prerendered，这三种方式都是为了让导航更快:

1. 当你在 Chrome 地址栏(也称为“ omnibox”)输入一个 URL 时，如果 Chrome 对你访问该页面有很高的信心，它可能会自动为你预先准备页面。
2. 当你在 Chrome omnibox 中输入一个搜索词时，Chrome 会根据搜索引擎的指示自动预先显示搜索结果页面。
3. 站点可以使用 [Speculation Rules API](https://github.com/jeremyroman/alternate-loading-modes/blob/main/triggers.md#speculation-rules)，以编程方式告诉 Chrome 预呈现哪些页面。 这取代了 ` <link rel="prerender"...>` 过去的做法，并允许网站根据页面上的推测规则主动预呈现页面。 这些可以静态存在于页面上，或者由页面所有者认为合适的 JavaScript 动态注入。

在每一种情况下，prerendered 的行为就好像页面已在不可见的背景选项卡中打开，然后通过用该预呈现页面替换前景选项卡来“激活”。 如果页面在完全预呈现之前被激活，则其当前状态为“前景”并继续加载，这意味着您仍然可以获得良好的体验。

由于预渲染页面是在隐藏状态下打开的，所以很多会导致侵入行为的API（比如提示）在这个状态下都不会被激活，而是会延迟到页面被激活。 在少数情况下这还不可能，预渲染被取消。 Chrome 团队正致力于将 prerendered 取消原因作为 API 公开，并增强 DevTools 功能以更轻松地识别此类边缘情况。

> Prerendered 页面应被视为提示和渐进增强。 与预加载不同，预渲染是一种推测性提示，浏览器可能会根据用户设置、当前内存使用情况或其他试探法选择不对提示采取行动。

## 预渲染的影响

当在 web.dev 上启用 Prerendered 时，预呈现允许近乎即时的页面加载，如下面的视频所示，其中 Prerendered 了第一篇博客文章：

<video poster="https://wd.imgix.net/image/W3z1f5ZkBJSgL1V1IfloTIctbIF3/zFNgVLiXhoPWpuveGPgY.png?auto=format" playsinline muted loop controls> 
<source type="video/mp4" src="https://storage.googleapis.com/web-dev-uploads/video/W3z1f5ZkBJSgL1V1IfloTIctbIF3/rO1MGC5jaBAIyo79KrVT.mp4"> 
</video>

web.dev 站点已经是一个快速站点，但即便如此，您也可以看到预呈现如何改善用户体验。 因此，这也会对网站的核心网络活力产生直接影响，LCP 接近于零，CLS 减少（因为任何加载 CLS 发生在初始视图之前），并改进 FID（因为加载应该在用户交互之前完成）。

即使页面在完全加载之前激活，抢先加载页面也应该改善加载体验。 当链接被激活而预渲染仍在进行时，预渲染页面将移动到主框架并继续加载。

但是，预渲染确实会使用额外的内存和网络带宽。 注意不要以用户资源为代价过度渲染。 仅当页面被导航到的可能性很高时才 prerendered。

## 通过 Chrome 的工具做预测

对于前两个用例，您可以在 `chrome://predictors` 页面中查看 Chrome 对 URL 的预测：

![](https://zzh-cat.oss-cn-beijing.aliyuncs.com/assets/2023-03-05-02.png)

绿线表示有足够的信心触发预渲染。 在此示例中，键入“s”会给出合理的置信度（琥珀色），但一旦您键入“sh”，Chrome 就会有足够的置信度，您几乎总是会导航到 https://sheets.google.com。

此屏幕截图是在相对较新的 Chrome 安装中截取的，并过滤掉了零置信度预测，但如果您查看自己的预测器，您可能会看到相当多的条目，并且可能需要更多字符才能达到足够高的置信度水平。

这些预测因素也是驱动地址栏建议选项的原因，您可能已经注意到：

![](https://zzh-cat.oss-cn-beijing.aliyuncs.com/assets/2023-03-05-01.png)

Chrome 会根据您的输入和选择不断更新其预测器。

- 对于大于 50% 的置信度（以琥珀色显示），Chrome 会主动预连接到域，但不会 prerender 页面。
- 对于大于 80% 的置信度（以绿色显示），Chrome 将 prerender URL。

## 使用 Speculation Rules API prerender 页面

对于第三方 prerender 选项，Web 开发人员可以将 JSON 指令插入到他们的页面上，以通知浏览器预渲染哪些 URL：

```html
<script type="speculationrules">
{
  "prerender": [
    {
      "source": "list",
      "urls": ["next.html", "next2.html"]
    }
  ]
}
</script>
```

Speculation Rules API 计划通过添加预取、[分数](https://github.com/WICG/nav-speculation/blob/main/triggers.md#scores)（例如，导航的可能性）和语法来扩展这个简单的示例，以实现[文档规则](https://github.com/WICG/nav-speculation/blob/main/triggers.md#document-rules)而不是列表规则（例如，匹配页面上的 href 模式），它们可以结合起来，例如仅在按下鼠标时预呈现链接。

目前，Chrome 只支持上述语法，这是一个简单的预渲染 url 列表。

对于 Chrome 108 中的初始启动，prerender 仅限于在同一选项卡中打开的同源页面。 Chrome 109 计划扩展此功能以允许预呈现同站点跨源页面（例如，https://a.example.com 可以预呈现 https://b.example.com 上的页面，其他站点已选择加入 同站点、跨源预渲染）。 未来的版本也可能在新标签中启用预渲染。

Speculation Rules：
- 静态插入页面的 HTML 中。 例如，新闻媒体站点或博客可能会预呈现最新文章，如果这通常是大部分用户的下一个导航。
- 通过 JavaScript 动态插入到页面中。 这可能基于应用程序逻辑、针对用户进行个性化设置，或者基于特定的用户操作，例如将鼠标悬停在某个链接上或单击该链接——正如许多库过去通过 `preconnect`、`prefetch` 甚至 `preload` 所做的那样。 建议那些喜欢动态插入的人关注推测规则支持，因为文档规则可能会允许浏览器处理您的许多用例，因为这将在未来引入。

Speculation Rules 可以添加在主框架的 `<head>` 或 `<body>` 中。 子帧中的 Speculation Rules 不会被执行，而 prerender 页面中的推测规则仅在该页面被激活后才会被执行。

## 多重 Speculation Rules

也可以将多个 Speculation Rules 添加到同一页面，并将它们附加到现有规则。 因此，以下不同的方式都会导致 `one.html` 和 `two.html` 预渲染：

List of URLs:

```html
<script type="speculationrules">
{
  "prerender": [
    {
      "source": "list",
      "urls": ["one.html", "two.html"]
    }
  ]
}
</script>
```

Multiple `speculationrules`:

```html
<script type="speculationrules">
{
  "prerender": [
    {
      "source": "list",
      "urls": ["one.html"]
    }
  ]
}
</script>
<script type="speculationrules">
{
  "prerender": [
    {
      "source": "list",
      "urls": ["two.html"]
    }
  ]
}
</script>
```

Multiple lists within one set of `speculationrules`:

```html
<script type="speculationrules">
{
  "prerender": [
    {
      "source": "list",
      "urls": ["one.html"]
    },
    {
      "source": "list",
      "urls": ["two.html"]
    }
  ]
}
</script>
```

## 检测 Speculation Rules API 支持

您可以使用标准 HTML 检查来检测 Speculation Rules API 支持：

```javascript
if (HTMLScriptElement.supports && HTMLScriptElement.supports('speculationrules')) {
  console.log('Your browser supports the Speculation Rules API.');
}
```

## 通过 JavaScript 动态添加 Speculation Rules

下面是一个使用 JavaScript 添加预渲染 Speculation Rules 的简单示例：

```javascript
if (HTMLScriptElement.supports &&
    HTMLScriptElement.supports('speculationrules')) {
  const specScript = document.createElement('script');
  specScript.type = 'speculationrules';
  specRules = {
    prerender: [
      {
        source: 'list',
        urls: ['/next.html'],
      },
    ],
  };
  specScript.textContent = JSON.stringify(specRules);
  console.log('added speculation rules to: next.html');
  document.body.append(specScript);
}
```

您可以在此演示页面上查看使用 JavaScript 插入的 Speculation Rules API 预渲染 [demo](https://prerender-demos.glitch.me/)。

## 取消 Speculation Rules

删除 Speculation Rules 将导致预渲染被取消，但在这种情况发生时，资源可能已经用于启动预渲染，因此如果有可能需要取消预渲染，建议不要预渲染。

## Speculation Rules 和内容安全策略

由于 Speculation Rules 使用 `<script>` 元素，即使它们只包含 JSON，如果站点使用它，它们也需要包含在 `script-src` [Content-Security-Policy](https://web.dev/csp/) 中——使用哈希或随机数。 将来，`script-src` 将支持新的 `inline-speculation-rules`，允许支持页面上的所有 `<script type="speculationrules">` 元素，而无需哈希或随机数。

## 在 JavaScript 中检测 prerender

`document.prerendering` API 将在页面预呈现时返回 true。 页面可以使用它来阻止或延迟预呈现期间的某些活动，直到页面实际被激活。

一旦预呈现文档被激活，`PerformanceNavigationTiming` 的 `activationStart` 也将被设置为一个非零时间，表示预呈现开始和文档实际被激活之间的时间。

您可以使用一个函数来检查预渲染和预渲染页面，如下所示：

```javascript
function pagePrerendered() {
  return (
    document.prerendering ||
    self.performance?.getEntriesByType?.('navigation')[0]?.activationStart > 0
  );
}
```

查看页面是否已预呈现的最简单方法是在预呈现发生后打开 DevTools，然后在控制台中键入 `performance.getEntriesByType('navigation')[0].activationStart`。 如果返回非零值，则您知道该页面已预呈现：

当页面被查看页面的用户激活时，将在文档上调度 `prerenderingchange` 事件，然后可以使用该事件启用以前默认在页面加载时启动但您希望延迟到页面加载的活动用户实际查看。

使用这些 API，JavaScript 可以适当地检测 prerendered 页面并对其采取行动。

以上就是 chrome 新预加载功能。

来源: [Prerender pages in Chrome for instant page navigations](https://developer.chrome.com/blog/prerender-pages/)