```table-of-contents
```
# [RAIL](https://web.dev/articles/rail?hl=zh-cn#response_process_events_in_under_50ms)

![[Pasted image 20240527012158.png]]用户对性能延迟的感知

|               |                                                                                                              |
| ------------- | ------------------------------------------------------------------------------------------------------------ |
| 0 至 16 毫秒     | 用户非常擅长跟踪运动，如果动画不流畅，他们就会不喜欢。只要每秒渲染 60 帧，这类动画就会感觉很流畅。也就是每帧** 16 毫秒**（包括浏览器将新帧绘制到屏幕上所需的时间），让应用生成一帧大约 **10 毫秒**。 |
| 0 至 100 毫秒    | 在此时间范围内响应用户操作，让用户感觉能够立竿见影。时间再长，操作与反应之间的连接就会中断。                                                               |
| 100 至 1000 毫秒 | 在此窗口中，事情感觉像是任务自然和持续推进的一部分。对于网络上的大多数用户，加载页面或更改视图代表着一个任务。                                                      |
| 1000 毫秒或以上    | 一旦超过 1,000 毫秒（1 秒），用户就会失去专注于他们正在执行的任务的注意力。                                                                   |
| 10000 毫秒或更长   | 一旦超过 10,000 毫秒（10 秒），用户就会感到沮丧，并可能放弃任务。他们以后不一定会回来。                                                            |
## 目标与准则
### 响应：在 50 毫秒内处理事件

**目标**：在 100 毫秒内完成由用户输入发起的转换，让用户感觉互动是瞬时完成的

**准则**：
- 为确保在 100 毫秒内获得可见响应，请在 50 毫秒内处理用户输入事件。这适用于大多数输入，例如点击按钮、切换表单控件或启动动画。这不适用于轻触拖动或滚动。
- 虽然听起来可能有违常理，但立即响应用户输入并不总是合适的。您可以使用以下 100 毫秒的窗口执行其他开销较大的工作，但要小心不要阻止用户。请尽可能在后台运行。
- 对于需要 50 毫秒以上才能完成的操作，请始终提供反馈。

 **50 毫秒还是 100 毫秒？**
我们的目标是在 100 毫秒以内响应输入，那么为什么我们的预算只有 50 毫秒呢？这是因为，除了输入处理之外，通常还会执行其他工作，并且这些工作会占用可用于获得可接受输入响应的部分时间。如果某个应用在空闲时间内以推荐的 50 毫秒分块执行工作，则意味，即使输入恰好需要在一系列工作中排队，也有多达50毫秒的排队时间。考虑到这一点，假设只有剩余的 50 毫秒可用于实际的输入处理是比较安全的。下图直观显示了这种影响，该图显示了空闲任务期间接收的输入如何排队，从而缩短了可用的处理时间：![[Pasted image 20240527012633.png]]

### 动画：在 10 毫秒内生成一帧

**目标**：
- 在 10 毫秒或更短的时间内生成动画中的每一帧。从技术上讲，每一帧的最大预算为 16 毫秒（1000 毫秒 / 60 帧/秒约 16 毫秒），但浏览器需要大约 **6 毫秒**才能渲染每一帧，因此准则是每帧 10 毫秒。
- 力求实现视觉流畅。当帧速率发生变化时，用户会注意到。

**准则**：
- 在诸如动画之类的高压力点中，关键是力所能及的方面都做不到，而绝对最小为不能。请尽可能利用 100 毫秒响应来**预先计算**开销非常大的工作，以便最大限度提高达到 60 fps 的几率。
- 如需了解各种动画优化策略，请参阅[渲染性能](https://web.dev/articles/rendering-performance?hl=zh-cn)

**注意** ：识别所有类型的动画。动画不仅仅是精美的界面效果。这些互动中的每一种都被视为动画：

- 视觉动画，例如进入和退出、[补间动画](https://www.webopedia.com/TERM/T/tweening.html)和加载指示器。
- 滚动。这包括快速滑动，即用户开始滚动，然后松开，页面会继续滚动。
- 正在拖动。动画通常出现在用户互动之后，例如平移地图或双指张合进行缩放。

### 空闲：最大限度地延长空闲时间

**目标**：最大限度地延长空闲时间，以提高网页在 50 毫秒内响应用户输入的几率。

**准则**：
- 利用空闲时间完成推迟的工作。例如，对于初始网页加载，请尽可能少加载数据，然后使用[空闲时间](https://developer.mozilla.org/docs/Web/API/Window/requestIdleCallback)加载其余数据。
- 在空闲时间不超过 50 毫秒时执行工作。如果时间更长，则可能会干扰应用在 50 毫秒内响应用户输入的能力。
- 如果用户在空闲时间工作期间与页面交互，则用户互动应始终具有最高优先级，并中断空闲时间工作。

### 加载：提交内容并在 5 秒内实现互动

如果网页加载速度缓慢，用户的注意力就会分散，用户就会认为任务无法正常工作。网站加载速度快，[平均会话数更长、跳出率更低、广告可见度更高](https://www.thinkwithgoogle.com/intl/en-154/insights-inspiration/research-data/need-mobile-speed-how-mobile-latency-impacts-publisher-revenue/)（funny, the link is unavailable)。

**目标**：
- 根据用户的设备和网络功能进行优化，以实现快速加载性能。目前，在 3G 网速较慢的中端移动设备上加载网页并可在 [5 秒或更短的时间内](https://web.dev/articles/performance-budgets-101?hl=zh-cn#establish_a_baseline)加载网页是一个不错的目标。[](https://web.dev/articles/tti?hl=zh-cn)
- 对于后续加载，最好在 2 秒内加载页面。

**准则**：

- 对用户共用的移动设备和网络连接测试负载性能。您可以使用 [Chrome 用户体验报告](https://web.dev/articles/chrome-ux-report?hl=zh-cn)来了解用户的[连接分布情况](https://developer.chrome.com/blog/chrome-ux-report-looker-studio-dashboard?hl=zh-cn#using-the-dashboard)。如果无法为您的网站提供相关数据，可以参阅 [The Mobile Economy 2019](https://www.gsma.com/mobileeconomy/) 报告，全球基准数据最好是中端 Android 手机（如 Moto G4）和速度较慢的 3G 网络（定义为 400 毫秒 RTT 和 400 kbps 的传输速度）。可以在 [WebPageTest](https://www.webpagetest.org/easy) 上提供这种组合。
- 请注意，虽然典型移动用户的设备可能会声称其使用的是 2G、3G 或 4G 连接，但实际上，由于数据包丢失和网络变化，有效连接速度往往会明显变慢。[](https://web.dev/articles/adaptive-serving-based-on-network-quality?hl=zh-cn#how_it_works)
- [避免使用会阻塞渲染的资源](https://developer.chrome.com/docs/lighthouse/performance/render-blocking-resources?hl=zh-cn)。
- 您无需在 5 秒内加载所有内容，才能产生加载完毕的感觉。考虑[延迟加载图片](https://web.dev/articles/browser-level-image-lazy-loading?hl=zh-cn)、[代码拆分 JavaScript 软件包](https://web.dev/articles/reduce-javascript-payloads-with-code-splitting?hl=zh-cn)，以及 [web.dev 上建议的其他优化](https://web.dev/explore/fast?hl=zh-cn)。

**注意** ：
确定影响网页加载性能的因素：
- 网速和延迟
- 硬件（例如运行速度较慢的 CPU）
- 缓存逐出
- L2/L3 缓存的差异
- 解析 JavaScript

## Tools for measuring
### Chrome DevTools

[Chrome DevTools](https://developer.chrome.com/docs/devtools) provides in-depth analysis on everything that happens while your page loads or runs. See [Get Started With Analyzing Runtime Performance](https://developer.chrome.com/docs/devtools/evaluate-performance) to get familiar with the **Performance** panel UI.

The following DevTools features are especially relevant:
- [Throttle your CPU](https://developer.chrome.com/docs/devtools/evaluate-performance/reference#cpu-throttle) to simulate a less-powerful device.
- [Throttle the network](https://developer.chrome.com/docs/devtools/evaluate-performance/reference#network-throttle) to simulate slower connections.
- [View main thread activity](https://developer.chrome.com/docs/devtools/evaluate-performance/reference#main) to view every event that occurred on the main thread while you were recording.
- [View main thread activities in a table](https://developer.chrome.com/docs/devtools/evaluate-performance/reference#activities) to sort activities based on which ones took up the most time.
- [Analyze frames per second (FPS)](https://developer.chrome.com/docs/devtools/evaluate-performance/reference#fps) to measure whether your animations truly run smoothly.
- [Monitor CPU usage, JS heap size, DOM nodes, layouts per second, and more](https://developer.chrome.com/blog/new-in-devtools-64#perf-monitor) in real-time with the **Performance Monitor**.
- [Visualize network requests](https://developer.chrome.com/docs/devtools/evaluate-performance/reference#network) that occurred while you were recording with the **Network** section.
- [Capture screenshots while recording](https://developer.chrome.com/docs/devtools/evaluate-performance/reference#screenshots) to play back exactly how the page looked while the page loaded, or an animation fired, and so on.
- [View interactions](https://developer.chrome.com/docs/devtools/evaluate-performance/reference#interactions) to quickly identify what happened on a page after a user interacted with it.
- [Find scroll performance issues in real-time](https://developer.chrome.com/docs/devtools/evaluate-performance/reference#scrolling-performance-issues) by highlighting the page whenever a potentially problematic listener fires.
- [View paint events in real-time](https://developer.chrome.com/docs/devtools/evaluate-performance/reference#paint-flashing) to identify costly paint events that may be harming the performance of your animations.

### Lighthouse

[Lighthouse](https://developer.chrome.com/docs/lighthouse/overview) is available in Chrome DevTools, at [PageSpeed Insights](https://pagespeed.web.dev/), as a Chrome Extension, as a Node.js module, and within WebPageTest. You give it a URL, it simulates a mid-range device with a slow 3G connection, runs a series of audits on the page, and then gives you a report on load performance, as well as suggestions on how to improve.
The following audits are especially relevant:

**Response**
- [Max Potential First Input Delay](https://developer.chrome.com/docs/lighthouse/performance/lighthouse-max-potential-fid). Estimates how long your app will take to respond to user input, based on main thread idle time.
- [Does not use passive listeners to improve scrolling performance](https://developer.chrome.com/docs/lighthouse/best-practices/uses-passive-event-listeners).
- [Total Blocking Time](https://developer.chrome.com/docs/lighthouse/performance/lighthouse-total-blocking-time). Measures the total amount of time that a page is blocked from responding to user input, such as mouse clicks, screen taps, or keyboard presses.
- [Time To Interactive](https://developer.chrome.com/docs/lighthouse/performance/interactive). Measures when a user can consistently interact with all page elements.

**Load**
- [Does not register a service worker that controls page and start_url](https://developer.chrome.com/docs/lighthouse/pwa/service-worker). A service worker can cache common resources on a user's device, reducing time spent fetching resources over the network.
- [Page load is not fast enough on mobile networks](https://developer.chrome.com/docs/lighthouse/pwa/load-fast-enough-for-pwa).
- [Eliminate render-blocking resources](https://developer.chrome.com/docs/lighthouse/performance/render-blocking-resources).
- [Defer offscreen images](https://developer.chrome.com/docs/lighthouse/performance/offscreen-images). Defer the loading of offscreen images until they're needed.
- [Properly size images](https://developer.chrome.com/docs/lighthouse/performance/uses-responsive-images). Don't serve images that are significantly larger than the size that's rendered in the mobile viewport.
- [Avoid chaining critical requests](https://developer.chrome.com/docs/lighthouse/performance/critical-request-chains).
- [Does not use HTTP/2 for all of its resources](https://developer.chrome.com/docs/lighthouse/best-practices/uses-http2).
- [Efficiently encode images](https://developer.chrome.com/docs/lighthouse/performance/uses-optimized-images).
- [Enable text compression](https://developer.chrome.com/docs/lighthouse/performance/uses-text-compression).
- [Avoid enormous network payloads](https://developer.chrome.com/docs/lighthouse/performance/total-byte-weight).
- [Avoid an excessive DOM size](https://developer.chrome.com/docs/lighthouse/performance/dom-size). Reduce network bytes by only shipping DOM nodes that are needed for rendering the page.