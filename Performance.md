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


# HTML性能的一般注意事项
要想构建可快速加载的网站，第一步就是要及时从服务器接收网页 HTML 的响应。当您在浏览器的地址栏中输入网址时，浏览器会向服务器发送 [`GET` 请求](https://developer.mozilla.org/docs/Web/HTTP/Methods/GET)进行检索。网页的第一个请求针对的是 HTML 资源，因此，确保 HTML 以最短延迟快速到达是关键性能目标。

## 尽量减少重定向
请求资源时，服务器可能做出一个重定向响应，可能是永久重定向(301 Moved Permanently 相应)或临时重定向(302 Found 响应)。
它会降低网页加载速度，因为需要浏览器向新位置发出额外的HTTP请求。
重定向有两种类型：
1. 完全发生在源站内的_同源重定向_。这些类型的重定向完全由您控制，因为管理它们的逻辑完全位于您的 Web 服务器上。
2. 由其他源启动的_跨域重定向_。这些类型的重定向通常无法控制。

广告、网址缩短服务和其他第三方服务通常会使用跨源重定向。虽然跨源重定向超出了您的控制范围，但您可能仍需要检查是否避免了多次重定向。例如，将广告链接到 HTTP 网页，而该网页又重定向到其 HTTPS 等效网页，或者跨源重定向到达您的来源，但随后触发同源重定向。

## 缓存HTML响应
缓存HTML响应并不容易，因为响应可能包含CSS，JS，图片等其他重要资源。这些资源可能有[一个独特的指纹](https://bundlers.tooling.report/hashing/)在它们的响应文件名中，随着文件内容变化。这意味着你缓存的HTML文件可能在一次部署后过时，因为它指向了过时的子资源。

不过，一个短的缓存生命时间—而不是没有缓存—是有益处的，例如允许一个资源被CDN缓存—减少源服务器处理的请求数，或者在浏览器允许资源重验证而不是再下载一次。这种方法最适用于不会变化的静态内容，并且设置适当的缓存时间。对静态HTML内容，一般5分钟被认为是安全的。

如果一个网页的HTML内容需要一定程度的定制—像是认证用户，你很可能出于一些考虑不倾向于缓存，例如安全和时效性。
如果一个HTML响应被用户浏览器缓存， 你就不能够使缓存失效。所以这种情况下最好避免缓存。
一种谨慎的缓存方法是[`ETage`](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Headers/ETag)或者[`Last-modified`](https://developer.mozilla.org/docs/Web/HTTP/Headers/Last-Modified)响应头。一个`ETag` （或者标目标签）是一个代表请求内容的独立识别器，通常是通过[散列化](https://en.wikipedia.org/wiki/Hash_function)资源内容获得。
```
ETag: "33a64df551425fcc55e4d42a148795d9f25f89d4"
```
每当资源改变时，`ETag`需要重新生成。后续的请求中，浏览器通过 [`If-None-Match`](https://developer.mozilla.org/docs/Web/HTTP/Headers/If-None-Match)请求头发送`ETag`。如果服务器的`ETag`与浏览器发送的匹配，服务器回复`304 Not Modified` 浏览器使用缓存里的资源。虽然这种方式仍有网络延迟，但一个304回应比HTML回应小得多。
跟网络应用开发的许多其他方面一样，权衡与拖鞋是无可避免的。
## 测量服务器响应时间
在不应用缓存时，服务器响应时间很大程度取决于托管服务提供商和后端应用。与静态页面相比，动态生成的响应可能TTFB更高。
如果用户在[字段](https://web.dev/articles/lab-and-field-data-differences?hl=zh-cn#field_data)遇到 TTFB 缓慢的问题，您可以使用 [`Server-Timing` 响应标头](https://developer.mozilla.org/docs/Web/HTTP/Headers/Server-Timing)公开有关时间在服务器上的什么位置的信息：
```
Server-Timing: auth;dur=55.5, db;dur=220
```
`Server-Timing` 标头的值可以包含多个指标，以及每个指标的时长。然后，可以[在运行时使用 Navigation Timing API](https://web.dev/articles/navigation-and-resource-timing?hl=zh-cn) 从用户那里收集这些数据，并进行分析，以了解用户是否遇到延迟。在前面的代码段中，响应标头包含两个显示时间：
- 对用户进行身份验证的时间 (`auth`)，用时 55.5 毫秒。
- 数据库访问时间 (`db`)，用时 220 毫秒。
%% [更多](https://web.dev/articles/optimize-ttfb?hl=zh-cn#understanding_high_ttfb_with_server_timing)关于`Server-Timing` 响应标头%%

## 压缩 
1. **尽可能使用 Brotli**。最常用的压缩算法是 gzip 和 Brotli。Brotli 比 gzip 提高了约 15% 到 20%。任何压缩都比不压缩好
2. **文件大小至关重要**。非常小的资源（小于 1 KiB）压缩得不太好，有时甚至根本压缩不到。任何类型的数据压缩的效果都取决于能够使用压缩算法找到更多可压缩数据位的大量数据。文件越大，压缩效果就越好，但是，您不会仅仅因为压缩效率更高就提供非常大的资源。大型资源（如 JavaScript 和 CSS）在浏览器_解压缩_后，需要大量时间来解析和评估，并且，即使它们仅发生微小变化，变化频率也可能会更高，因为任何更改都会导致不同的[文件哈希值](https://bundlers.tooling.report/hashing/)。
3. **了解动态压缩和静态压缩**。动态压缩和静态压缩是确定何时应压缩资源的不同方法。动态压缩会在请求资源时压缩资源，有时甚至在每次请求资源时压缩资源。另一方面，静态压缩会提前压缩文件，因此在收到请求时无需执行压缩。静态压缩消除了压缩本身涉及的延迟时间，在使用动态压缩的情况下，这可能会增加服务器响应时间。JavaScript、CSS 和 SVG 图片等静态资源应动态压缩，而 HTML 资源（尤其是为经过身份验证的用户动态生成的资源）应动态压缩。
自行进行压缩非常具有挑战性，通常最好让内容分发网络 (CDN)（将在下一部分讨论）为您处理此操作。
## 内容分发网络 (CDN)
[内容分发网络 (CDN)](https://web.dev/articles/content-delivery-networks?hl=zh-cn) 是分布式服务器网络，服务器从源服务器缓存资源，反过来再从物理上更靠近用户的边缘服务器传送资源。在距离用户较近时，可以缩短[往返时间 (RTT)](https://en.wikipedia.org/wiki/Round-trip_delay)，而 HTTP/2 或 HTTP/3、缓存和压缩等优化技术则可以让 CDN 更快地提供内容，而不是从源服务器提取内容。在某些情况下，使用 CDN 可以显著改善网站的 TTFB。

# [了解关键渲染路径](https://web.dev/learn/performance/understanding-the-critical-path?hl=zh-cn)
## 渐进式渲染

网络是天生散布的。与在使用前安装的原生应用不同，浏览器不能依赖于具有呈现网页所需的所有资源的网站。因此，浏览器非常擅长渐进地呈现网页。原生应用通常包括安装阶段和运行阶段。但是，对于网页和 Web 应用，这两个阶段之间的界限没有明显，浏览器在设计时就考虑到了这一点。
浏览器获取用于呈现网页的资源后，通常会开始呈现网页。因此，需要选择何时呈现：何时显示过早？

如果浏览器在只需要一些 HTML 时（但在它尚未添加任何 CSS 或必要的 JavaScript 之前）尽快呈现，网页就会立即看起来损坏，并且会在进行最终呈现时发生显著变化。这种体验比最初显示空白屏幕一段时间，直到浏览器具有初始渲染所需的更多资源，从而提供更好的用户体验，这种体验会更糟糕。

另一方面，如果浏览器等待所有资源可用，而不是进行任何顺序渲染，那么用户就会等待很长时间；如果网页在很早之前就可用，这往往是不必要的。

浏览器需要了解其应等待的最低资源数量，以免呈现明显中断的体验。另一方面，在向用户显示某些内容之前，浏览器也不应等待超过必要时间。浏览器在执行初始渲染之前执行的一系列步骤称为“关键渲染路径”。

## （关键）渲染路径
呈现路径涉及以下步骤：
- 通过 HTML 构建文档对象模型 (DOM)。
- 通过 CSS 构建 CSS 对象模型 (CSSOM)。
- 应用任何会更改 DOM 或 CSSOM 的 JavaScript。
- 通过 DOM 和 CSSOM 构建渲染树。
- 在页面上执行样式和布局操作，看看哪些元素适合显示。
- 在内存中绘制元素的像素。
- 如果有任何像素重叠，则合成像素。
- 以物理方式将所有生成的像素绘制到屏幕上。
![[fig-1-v2.svg]]这一呈现过程会发生多次。初始渲染会调用此流程，但随着更多会影响网页渲染的资源可用，浏览器将会重新运行此流程（或许只是其中的一部分），以更新用户看到的内容。关键渲染路径侧重于初始渲染的流程，并依赖于执行初始渲染所需的关键资源。

## 关键渲染路径上有哪些资源？
浏览器需要等待一些关键资源下载完毕，然后才能完成初始渲染。这些资源包括：
- HTML 的一部分。
- `<head>` 元素中阻塞渲染的 CSS。
- `<head>` 元素中的阻塞渲染的 JavaScript。
关键在于浏览器以流式方式处理 HTML。浏览器一旦获取网页 HTML 的任何部分，就会开始对其进行处理。然后，浏览器就可以（并且通常确实）决定先呈现网页，然后再接收网页的其余部分 HTML。

重要的是，在首次渲染时，浏览器通常不会等待：
- 所有 HTML。
- 字体。
- Images.
- `<head>` 元素外的非阻塞渲染的 JavaScript。（例如，位于 HTML 末尾的 `<script>` 元素）
- `<head>` 元素外或[`media` 属性](https://developer.mozilla.org/docs/Web/HTML/Element/link#conditionally_loading_resources_with_media_queries)值不适用于当前视口的 CSS，不会阻止内容渲染。
浏览器通常将字体和图片视为要在后续页面重新渲染时填充的内容，因此不需要延迟初始渲染。（详见渲染中的CLS部分）
## 阻塞渲染的资源
有些资源被认为非常关键，以至于浏览器会暂停网页渲染，直到它处理完毕。CSS 默认属于此类别。

当浏览器看到 CSS（无论是 `<style>` 元素中的内嵌 CSS，还是由 `<link rel=stylesheet href="...">` 元素指定的外部引用的资源）时，浏览器在完成对该 CSS 的下载和处理之前，将避免渲染更多内容。
%%
尽管 CSS 默认会阻塞渲染，但也可以通过更改 `<link>` 元素的 `media` 属性来指定与当前条件不匹配的值，将其转换为不阻塞渲染的资源：`<link rel=stylesheet href="..." media=print>`。 [以前会使用此方法](https://www.filamentgroup.com/lab/load-css-simpler/)来允许非关键 CSS 以不阻塞渲染的方式加载。
%%
资源阻塞渲染并不一定意味着它会阻止浏览器执行任何其他操作。当浏览器发现需要下载某项 CSS 资源时，它会请求该 CSS 资源并暂停渲染，但仍会继续_处理_其余 HTML 并寻找其他工作。

Render-blocking resources, like CSS, used to block all rendering of the page when they were discovered. This means that whether some CSS is render-blocking or not depends on whether the browser has discovered it. Some browsers ([Firefox initially](https://jakearchibald.com/2016/link-in-body/), and now [also Chrome](https://chromestatus.com/feature/5696805480169472)) only block rendering of content below the render-blocking resource. This means that for the critical render-blocking path, we're typically interested in render-blocking resources in the `<head>`, as they effectively block rendering of the entire page.

最近的一项创新是 [Chrome 105 中](https://chromestatus.com/feature/5452774595624960)新增的 [`blocking=render` 属性](https://html.spec.whatwg.org/multipage/urls-and-fetching.html#blocking-attributes)。这样一来，开发者可以将 `<link>`、`<script>` 或 `<style>` 元素明确标记为阻塞渲染，直到该元素处理完毕，但仍允许解析器继续处理文档。
## Parser-blocking resources
Parser-blocking resources are those that prevent the browser from looking for other work to do by continuing to parse the HTML. 
JavaScript by default is parser-blocking (unless specifically marked as [asynchronous](https://developer.mozilla.org/docs/Web/HTML/Element/script#async) or [deferred](https://developer.mozilla.org/docs/Web/HTML/Element/script#defer)), as JavaScript can change the DOM or the CSSOM upon its execution. Therefore, it's not possible for the browser to continue processing other resources until it knows the full impact of the requested JavaScript on a page's HTML. Synchronous JavaScript therefore blocks the parser.

Parser-blocking resources are effectively render-blocking as well. Since the parser can't continue past a parsing-blocking resource until it has been fully processed, it can't access and render the content after it. The browser can render any HTML received so far while it waits, but where the critical rendering path is concerned, any parser-blocking resources in the `<head>` effectively mean that all page content is blocked from being rendered.

Blocking the parser can have a huge performance cost—much more than just blocking rendering. For this reason, browsers will try to reduce this cost by using a secondary HTML parser known as the [preload scanner](https://web.dev/articles/preload-scanner) to download upcoming resources while the primary HTML parser is blocked. While not as good as actually parsing the HTML, it does at least allow the networking functions in the browser to work ahead of the blocked parser, meaning it will be less likely to be blocked again in the future.

# 优化资源加载
## 阻塞渲染
CSS 是一种阻塞渲染的资源，因为它会阻止浏览器渲染任何内容，直至您构建了 [CSS 对象模型 (CSSOM)](https://developer.mozilla.org/docs/Web/API/CSS_Object_Model)。[](https://web.dev/articles/critical-rendering-path/render-blocking-css?hl=zh-cn)浏览器会阻止呈现，以防止出现[非样式内容闪烁 (FOUC)](https://en.wikipedia.org/wiki/Flash_of_unstyled_content)，这从用户体验的角度来看是不希望发生的。
![[fouc.webm]]
%%
阻塞解析器的 `<script>` 还必须等待所有阻塞渲染的 CSS 资源到达并得到解析，然后浏览器才能执行这些资源。这也是设计要求，因为脚本可能会访问阻止呈现的样式表中声明的样式（例如，通过使用 `element.getComputedStyle()`）。
%%
## The preload scanner
为了充分利用预加载扫描器，服务器发送的 HTML 标记中应包含关键资源。预加载扫描器无法发现以下资源加载模式：
- 由 CSS 使用 `background-image` 属性加载的图片。这些图片引用位于 CSS 中，预加载扫描器无法发现这些引用。
- 动态加载的脚本，采用 `<script>` 元素标记（使用 JavaScript 注入 DOM）或使用[动态 `import()`](https://developer.mozilla.org/docs/Web/JavaScript/Reference/Operators/import) 加载的模块。
- 使用 JavaScript 在客户端上呈现的 HTML。此类标记包含在 JavaScript 资源的字符串中，预加载扫描器无法发现此类标记。
- CSS `@import` 声明。
如果无法避免此类模式，您或许可以使用 `preload` 提示来避免资源发现的延迟。
[more](https://web.dev/articles/preload-scanner#how_to_tell_when_the_preload_scanner_is_working)

## CSS

### 缩减大小
缩减空格换行等字符
```
/* Unminified CSS: *//* Heading 1 */h1 {  font-size: 2em;  color: #000000;}/* Heading 2 */h2 {  font-size: 1.5em;  color: #000000;}
```

```
/* Minified CSS: */h1,h2{color:#000}h1{font-size:2em}h2{font-size:1.5em}
```
%%
某些高级 CSS 缩减器可能会采用其他优化措施，例如将冗余规则合并到多个选择器中。 不过，此类高级 CSS 优化可能存在风险，可能无法针对所有 CSS 方法或设计系统顺畅运行或扩展。
%%

### 移除未使用的CSS
[覆盖率工具](https://developer.chrome.com/docs/devtools/css/reference/?hl=zh-cn#coverage)
移除未使用的 CSS 会产生双重效果：除了缩短下载时间之外，您还可以优化渲染树的构建，因为浏览器需要处理的 CSS 规则更少。
%%
Depending on the architecture of your website, it may not be possible to completely eliminate unused CSS—nor should you expect to. Focus on big wins: if you see a large part of a CSS file that is unused by the current page, it may either be used by a different page (which you can move to a different file altogether), or be deleted entirely if the CSS is no longer being used in your project.
%%
### 避免使用@import
```
/* Don't do this: */@import url('style.css');
```
只有先下载包含该声明的CSS文件，才可以下载被引入的文件。这会导致请求链，进而延缓网页绘制。此外preload scanner也无法发现这种声明加载的文件。
大多数情况下，可以使用`<link rel="stylesheet">`替换
%%
If you need to use `@import`—such as for [cascade layers](https://developer.mozilla.org/docs/Learn/CSS/Building_blocks/Cascade_layers) or third-party style sheets—you can mitigate the delay by using the `preload` directive for the imported style sheet. Additionally, CSS preprocessors—such as SASS or LESS—commonly use the `@import` syntax as part of a developer experience improvement that allows for separate and more modularized source files. However, when a CSS preprocessor encounters `@import` declarations, the referenced files are bundled and written into a single style sheet, avoiding the consecutive request penalty that `@import` causes in plain CSS.
%%

### 内嵌critical CSS
%%
Critical CSS refers to the styles required to render content that is visible within the initial viewport. The initial viewport concept is sometimes referred to as "above the fold". The remaining content on a page is left unstyled while the remaining CSS is loaded asynchronously.
%%
在文档 `<head>` 中内嵌关键样式可以消除对 CSS 资源的网络请求，并且如果操作正确，可以在用户的浏览器缓存尚未准备好时缩短初始加载时间。其余 CSS 可以[异步](https://www.filamentgroup.com/lab/load-css-simpler/)加载，也可以附加到 `<body>` 元素的末尾。
```
<head>  
	<title>Page Title</title>  
	<!-- ... -->  
	<style>h1,h2{color:#000}h1{font-size:2em}h2{font-size:1.5em}</style></head>
<body>  
<!-- Other page markup... -->  
<link rel="stylesheet" href="non-critical.css">
</body>
```
%%
提取和维护关键样式可能并非易事。应包含哪些样式？应定位到哪个/哪些视口？这个过程可以自动完成吗？如果用户在非关键 CSS 加载完成之前向下滚动，会发生什么情况？如果用户遇到 FOUC，会有何影响？这些都是值得考虑的好问题，因为您网站的架构可能会使关键 CSS 的使用变得极其困难。不过，在某些特定情况下，性能优势可能是值得的，因此请调查关键 CSS 是否是您网站的可行方案！
%%
但其缺点是，内嵌大量 CSS 会导致初始 HTML 响应的字节增多。由于 HTML 资源通常无法缓存很长时间（甚至根本无法缓存），因此对于可能在外部样式表中使用同一 CSS 的后续网页，系统不会缓存内联的 CSS。请测试和衡量网页的性能，以确保权衡取舍是值得的。

## JavaScript
### 阻塞渲染的JS
加载不带 `defer` 或 `async` 属性的 `<script>` 元素时，浏览器会阻止解析和渲染，直到脚本下载、解析并执行完毕。同样，内联脚本也会阻止解析器，直到解析和执行脚本。

### ### `async` 与 `defer`
这两个声明允许加载外部脚本，而不阻塞HTML解析。scripts (including inline scripts) with `type="module"` are deferred automatically.
![[fig-2.svg]]使用saync标签加载的脚本在下载后立即解析和执行，使用defer加载的脚本会在文档解析完成后执行—与浏览器DOMContentLoaded事件同时发生。此外，async不一定按顺序执行，defer则按顺序执行。
`type="module"` 脚本执行与defer相同，Javascript注入的脚本则像async
### 客户端渲染
Generally, you should avoid using JavaScript to render any critical content or a page's [LCP element](https://web.dev/articles/lcp#what-elements-are-considered) (Largest Content Paint). This is known as client-side rendering, and is a technique used extensively in Single Page Applications (SPAs).
JS呈现的标签会绕过预加载扫描程序，这可能延迟关键资源的下载，因为只有脚本下载执行完毕后，浏览器才会下载这些资源。
[避免链接关键请求](https://developer.chrome.com/docs/lighthouse/performance/critical-request-chains)。

### 缩减大小
更快的下载、解析和编译
此外，缩减 JavaScript 的大小时，不仅会去除空格、制表符和注释等内容，而且源 JavaScript 中的符号也会被缩短。此过程有时称为“伪造”（uglification）。
```
// Unuglified JavaScript source code:export function injectScript () {  
	const scriptElement = document.createElement('script');  
	scriptElement.src = '/js/scripts.js';  
	scriptElement.type = 'module';  
	
	document.body.appendChild(scriptElement);
}
```
```
// Uglified JavaScript production code:export function injectScript(){const t=document.createElement("script");t.src="/js/scripts.js",t.type="module",document.body.appendChild(t)}
```
您可以看到源代码中人类可读的变量 `scriptElement` 已缩短为 `t`。当应用于大量脚本时，您可以节省相当大的开销，而不会影响网站正式版 JavaScript 提供的功能。


# [不要与预加载扫描程序冲突](https://web.dev/articles/preload-scanner?hl=zh-cn#how_to_tell_when_the_preload_scanner_is_working)
CSS和没有`async`和`defer`的js脚本都会阻塞**解析和渲染**，js脚本还会等待css的到达和解析，详见前文注释

## 怎样判断预加载扫描何时运行
The preload scanner exists _because_ of blocked rendering and parsing. If these two performance issues never existed, the preload scanner wouldn't be very useful. The key to figuring out whether a web page benefits from the preload scanner depends on these blocking phenomena.

在下面例子中，由于 CSS 文件会同时阻止呈现和解析，因此会通过代理服务人为地为样式表设置 2 秒的延迟。
![[Pasted image 20240530005440.png]]
虽然样式表在开始加载前人为地通过代理延迟两秒，绘制和解析被阻止，但预加载扫描器会发现标记载荷中靠后的图片。
## 注入`async`脚本
假设您的 `<head>` 中有包含一些内嵌 JavaScript 的 HTML，如下所示：
```
<script>  const scriptEl = document.createElement('script');  scriptEl.src = '/yall.min.js';  document.head.appendChild(scriptEl);</script>
```
注入的脚本默认为 [`async`](https://developer.mozilla.org/docs/Web/HTML/Element/script#attr-async)，因此注入此脚本时，其行为就像是已应用 `async` 属性一样。这意味着它会尽快运行，且不会阻塞渲染。不过，如果您假设此内嵌 `<script>` 位于用于加载外部 CSS 文件的 `<link>` 元素之后，则效果会不太理想：
![[Pasted image 20240530005804.png]]
Let's break down what happened here:
1. At 0 seconds, the main document is requested.
2. At 1.4 seconds, the first byte of the navigation request arrives.
3. At 2.0 seconds, the CSS and image are requested.
4. Because the parser is blocked loading the stylesheet and the inline JavaScript that injects the `async` script comes _after_ that stylesheet at 2.6 seconds, the functionality that script provides isn't available as soon as it could be.
预下载扫描无法发现注入的脚本。

```
<script src="/yall.min.js" async></script>
```
![[Pasted image 20240530010202.png]]
现在可以了。

 可能有提议使用[`rel=preload`](https://developer.mozilla.org/docs/Web/HTML/Link_types/preload)解决问题，不过这可能带来副作用。
 ![[Pasted image 20240530010648.png]]
 设置预加载修复了之前的问题，但来了一个新的。再前两个demo中的脚本是以**低优先级**加载的，而样式表是**最高优先级**。而最后一个demo中，样式表仍是最高，但脚本被提升为了高优先级。
 当某个资源的优先级提高时，浏览器会为其分配更多带宽。这意味着，即使样式表的优先级最高，但脚本的高优先级也可能会导致带宽争用。这可能是连接速度缓慢或在资源非常大的情况下的因素。
 （这三个demo的模拟环境是移动设备通过3G连接网页）
 %%
 您可以在现代浏览器的“网络”标签页中找到资源优先级。尤其是在 Chrome 开发者工具中，[您可以右键点击列标题](https://developer.chrome.com/docs/devtools/network/reference/?hl=zh-cn#columns)，确保优先级列可见。
 %%
## 使用JS延迟加载(lazy loading)
延迟加载是一种节省数据流量的好方法，通常应用于图片。不过，有时延迟加载会错误地应用于“首屏”的图片，可以这么说（so to speak）。
```
<img data-src="/sand-wasp.jpg" alt="Sand Wasp" width="384" height="255">
```
The use of a `data-` prefix is a common pattern in JavaScript-powered lazy loaders. When the image is scrolled into the viewport, the lazy loader strips the `data-` prefix, meaning that in the preceding example, `data-src` becomes `src`. This update prompts the browser to fetch the resource.

This pattern isn't problematic until it's applied to images that are in the viewport during startup. Because the preload scanner doesn't read the `data-src` attribute in the same way that it would an `src` (or `srcset`) attribute, the image reference isn't discovered earlier. **Worse, the image is delayed from loading until _after_ the lazy loader JavaScript downloads, compiles, and executes.**
![[Pasted image 20240530012048.png]]
The image resource is unnecessarily lazy-loaded, even though it is visible in the viewport during startup. This defeats the preload scanner and causes an unnecessary delay.

The solution is to change the image markup:
```
<img src="/sand-wasp.jpg" alt="Sand Wasp" width="384" height="255">
```
![[Pasted image 20240530012154.png]]
The result in this simplified example is a 100-millisecond improvement in LCP on a slow connection.
%%
Other sources, like [`<iframe>` element](https://developer.mozilla.org/docs/Web/HTML/Element/iframe) , an also be affected, and since `<iframe>` elements can load many sub resources, the impact of performance could be substantially worse.
%%
## CSS 背景图片
预加载扫描器只扫描标签，它不会扫描其它资源类型，例如CSS，它里边可能有`background-image`图片。
与 HTML 一样，浏览器会将 CSS 处理到自己的对象模型（称为 [CSSOM](https://developer.mozilla.org/docs/Web/API/CSS_Object_Model)）中。如果在构建 CSSOM 时发现了外部资源，系统会在发现时请求这些资源，而不是由预加载扫描器请求。
如果您网页的LCP候选元素有CSS`background-image`属性的元素，以下是资源加载时的状况
![[Pasted image 20240530012619.png]]
可以预加载该图片，帮浏览器更快地发现图片：
```
<!-- Make sure this is in the <head> below any     stylesheets, so as not to block them from loading -->
<link rel="preload" as="image" href="lcp-image.jpg">
```
![[Pasted image 20240530012855.png]]
%%
如果您的 LCP 候选内容来自 `background-image` CSS 属性，但相应图片因视口大小而异，则您需要在 `<link>` 元素上指定 [`imagesrcset` 属性](https://developer.mozilla.org/docs/Web/HTML/Element/link#attr-imagesrcset)。
%%

## 内嵌资源过多
内联是一种将资源放入 HTML 中的做法。您可以在 `<style>` 元素中内嵌样式表、在 `<script>` 元素中内嵌脚本，以及使用 [base64 编码](https://developer.mozilla.org/docs/Glossary/Base64)在几乎所有其他资源中内嵌样式表。
内嵌资源比下载资源更快，因为系统不会针对相应资源发出单独的请求。它就在文档中，可以即时加载。不过，这样做也存在一些显著的缺点：

- 如果您没有缓存 HTML，并且在 HTML 响应为动态时无法缓存，则内联资源永远不会被缓存。这会影响性能，因为内联资源不可重复使用。
- 即使您可以缓存 HTML，内联资源也不会在文档之间共享。与可在整个源中缓存和重复使用的外部文件相比，这会降低缓存效率。
- 如果内嵌内容过多，则会导致预加载扫描器延迟在文档中发现资源，因为下载额外的内嵌内容会花费更长的时间。
以[此页面](https://preload-scanner-fights.glitch.me/inline-nothing.html)为例。在某些情况下，LCP 候选元素是位于页面顶部的图片，而 CSS 则位于由 `<link>` 元素加载的单独文件中。该页面还使用了四种网页字体，它们作为与 CSS 资源分开请求的文件。
![[Pasted image 20240530013312.png]]网页的 LCP 候选内容是从 `<img>` 元素加载的图片，但会被预加载扫描器发现，因为网页加载所需的 CSS 和字体是在单独的资源中加载的，这不会延迟预加载扫描器执行工作。

如果 CSS 和所有字体以 base64 资源的形式内嵌，会发生什么情况？
![[Pasted image 20240530013529.png]]网页的 LCP 候选内容是从 `<img>` 元素加载的图片，但是在 内嵌 CSS 及其四种字体资源会延迟预加载扫描程序发现该图片，直到这些资源完全下载为止。LCP图片的绘制时间从3.5秒来到了7秒。

预加载能否改进这里的内容？当然可以。您可以预加载 LCP 映像并缩短 LCP 时间，但使用内嵌资源让可能无法缓存的 HTML 变得膨胀，会产生其他负面影响。[First Contentful Paint (FCP)](https://web.dev/articles/fcp?hl=zh-cn) 也会受到此模式的影响。在未内嵌任何内容的网页版本中，FCP 约为 2.7 秒。在内联所有内容的版本中，FCP 约为 5.8 秒。
在将内容内嵌到 HTML 中时要格外小心，尤其是以 base64 编码的资源。一般不建议这样做，除非资源非常少。尽可能少内嵌，因为内联过多会起火。

## 使用客户端JS渲染标签
开发者不仅依赖JS提供交互，还依赖它提供内容。这可以提升开发体验，但对开发者的好处并不总是能转化为对用户的好处。
这种模式也可能让Preload Scanner失败。
![[Pasted image 20240530014334.png]]由于内容包含在 JavaScript 中并依靠框架进行呈现，因此客户端呈现的标记中的图片资源会对预加载扫描器隐藏。展示了等效的服务器渲染体验。下图为等效的服务器渲染体验：
![[Pasted image 20240530014453.png]]
当浏览器中的 JavaScript 完全包含标记有效负载并完全由其呈现时，该标记中的任何资源实际上对预加载扫描器而言都是不可见的。这延迟了重要资源的发现，无疑会影响 LCP

### 客户端渲染劣势
这有点偏离本文的重点，但在客户端上渲染标记的影响远不止于打败预加载扫描程序。

首先，introducing JavaScript to power an experience that doesn't require it introduces **unnecessary processing time** that can affect **[Interaction to Next Paint (INP)**](https://web.dev/articles/inp).

在客户端渲染大量标签更可能导致[长任务](https://web.dev/articles/long-tasks-devtools?hl=zh-cn)，相比与相同效果的服务端渲染。原因是，除了 JavaScript 涉及的额外处理之外,浏览器从服务器流式传输标记，并分块渲染，这种渲染倾向于限制长任务。客户端渲染相反，会把它作为单个的整体任务。除了 INP 之外，这还可能会影响页面响应能力指标，例如[总阻塞时间 (TBT)](https://web.dev/articles/tbt?hl=zh-cn) 或 [First Input Delay (FID)](https://web.dev/articles/fid?hl=zh-cn)。

The remedy for this scenario depends on the answer to this question: **Is there a reason why your page's markup can't be provided by the server as opposed to being rendered on the client?** If the answer to this is "no", server-side rendering (SSR) or statically generated markup should be considered where possible, as it will help the preload scanner to discover and opportunistically fetch important resources ahead of time.