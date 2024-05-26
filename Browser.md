```table-of-contents
title: 
style: nestedList # TOC style (nestedList|inlineFirstLevel)
minLevel: 0 # Include headings from the specified level
maxLevel: 0 # Include headings up to the specified level
includeLinks: true # Make headings clickable
debugInConsole: false # Print debug info in Obsidian console
```
# 多进程架构
For the renderer process, multiple processes are created and assigned to each tab. Until very recently, Chrome gave each tab a process when it could; now it tries to give each site its own process, including iframes (see [Site Isolation](https://developer.chrome.com/blog/inside-browser-part1#site-isolation)).
![[Pasted image 20240524230345.png]]
具体说来，Chrome 的主要进程及其职责如下：
- Browser Process：
	1. 负责包括地址栏，书签栏，前进后退按钮等部分的工作；
	2. 负责处理浏览器的一些不可见的高优先级的底层操作，比如网络请求和文件访问；
- Renderer Process：
	负责一个 tab 内关于网页呈现的所有事情
- Plugin Process：
	负责控制一个网页用到的所有插件，如 flash
- GPU Process
		负责处理 GPU 相关的任务

## Benefits
1. Earlier, I mentioned Chrome uses multiple renderer process. In the most simple case, you can imagine each tab has its own renderer process. Let's say you have 3 tabs open and each tab is run by an independent renderer process. If one tab becomes unresponsive, then you can close the unresponsive tab and move on while keeping other tabs alive. If all tabs are running on one process, when one tab becomes unresponsive, all the tabs are unresponsive.
![[Pasted image 20240524230711.png]]
2. Another benefit of separating the browser's work into multiple processes is security and sandboxing. Since operating systems provide a way to restrict processes' privileges, the browser can sandbox certain processes from certain features.
Because processes have their own private memory space, they often contain copies of common infrastructure (like V8 which is a Chrome's JavaScript engine). This means more memory usage as they can't be shared the way they would be if they were threads inside the same process. In order to save memory, Chrome puts a limit on how many processes it can spin up. The limit varies depending on how much memory and CPU power your device has, but when Chrome hits the limit, it starts to run multiple tabs from the same site in one process.

# 导航过程
浏览器 Tab 外的工作主要由 Browser Process 掌控，Browser Process 又对这些工作进一步划分，使用不同线程进行处理：
- UI thread ： 控制浏览器上的按钮及输入框；
- network thread: 处理网络请求，从网上获取数据；
- storage thread: 控制文件等的访问
## 过程
当我们在浏览器地址栏中输入文字，并点击回车获得页面内容的过程在浏览器看来可以分为以下几步：
1. **处理输入**
UI thread 需要判断用户输入的是 URL 还是 query；
	![[Pasted image 20240524234609.png]]

2. **开始导航** 
当用户点击 Enter 键时，界面线程会发起网络调用来获取网站内容。标签页的一角会显示“正在加载”旋转图标，并且网络线程会通过适当的协议（例如 DNS 查找）并为请求建立 TLS 连接。此时，网络线程可能会收到类似于 HTTP 301 的服务器重定向标头。在这种情况下，网络线程会与服务器请求重定向的界面线程通信。然后，系统将发起另一个网址请求。
![[Pasted image 20240524221330.png]]

3. **读取响应**
当响应正文（载荷）开始传入后，网络线程会根据需要查看流的前几个字节。响应的 Content-Type 标头应该会显示具体的数据类型，但由于可能缺失或错误，因此请在此处完成 [MIME 类型嗅探](https://developer.mozilla.org/docs/Web/HTTP/Basics_of_HTTP/MIME_types)。如果响应是 HTML 文件，下一步就是将数据传递给渲染程序进程；但如果它是 ZIP 文件或其他某个文件，则意味着它是一个下载请求，所以它们需要将数据传递给下载管理器。
系统还会在此处进行[SafeBrowsing](https://safebrowsing.google.com/?hl=zh-cn)检查。 如果该网域和响应数据似乎与某个已知的恶意网站匹配，则网络线程会发出提醒并显示一个警告页面。此外，系统会进行Cross-Origin Read Blockingg检查，以确保敏感的跨网站数据不会进入渲染程序进程。
![[Pasted image 20240524235028.png]]
![[Pasted image 20240524235035.png]]

4. **查找渲染进程**
在完成所有检查且网络线程确信浏览器应导航到所请求的网站后，网络线程会告知界面线程数据已准备就绪。然后，界面线程会找到渲染程序进程来继续渲染网页。
![[Pasted image 20240524235105.png]]

5. **提交导航**
现在数据和渲染程序进程已准备就绪，浏览器进程会向渲染器进程发送 IPC 以提交导航。它还会传递数据流，以便渲染程序进程可以继续接收 HTML 数据。浏览器进程听到在渲染程序进程中发生提交的确认信息后，导航即告完成，文档加载阶段随即开始。
此时，地址栏会更新，安全指示器和网站设置界面会反映新页面的网站信息。系统会更新该标签页的会话历史记录，以便使用“往返”按钮 逐步跳转到用户刚刚访问过的网站为便于您在关闭标签页或窗口时恢复标签页/会话，系统会将会话历史记录存储在磁盘上。
![[Pasted image 20240524235410.png]]
6. 额外步骤：初始加载完成
提交导航后，渲染器进程会继续加载资源并渲染页面。渲染程序进程“完成”渲染后，会将 IPC 发送回浏览器进程（在网页中的所有帧上触发所有 `onload` 事件并完成执行后）。此时，界面线程会停止标签页上的加载旋转图标。

7. **导航到其他网站**
如果用户再次在地址栏中输入不同的网址，会发生什么情况？浏览器进程会执行相同的步骤来导航到不同的网站。 但在此之前，它需要先确认当前呈现的网站是否关注 [`beforeunload`](https://developer.mozilla.org/docs/Web/Events/beforeunload) 事件。`beforeunload` 可在您尝试离开此网站或关闭标签页时创建“要离开此网站吗？”提醒。 标签页中的所有内容（包括 JavaScript 代码）均由渲染器进程处理，因此当新的导航请求时，浏览器进程必须检查当前的渲染器进程。
![[Pasted image 20240524235722.png]]
如果导航是从渲染程序进程启动的（例如用户点击链接或客户端 JavaScript 已运行 `window.location = "https://newsite.com"`），渲染程序进程会先检查 `beforeunload` 处理程序。接着，执行与浏览器进程发起的导航相同的流程。唯一的区别在于，导航请求是从渲染器进程发送到浏览器进程。
当新导航前往与当前呈现的网站不同的网站时，系统会调用单独的渲染进程来处理新的导航，同时保留当前渲染进程来处理 `unload` 等事件。如需了解详情，请参阅[页面生命周期状态概览](https://developers.google.com/web/updates/2018/07/page-lifecycle-api?hl=zh-cn#overview_of_page_lifecycle_states_and_events)以及如何使用 [Page Lifecycle API](https://developers.google.com/web/updates/2018/07/page-lifecycle-api?hl=zh-cn) 接入事件。

# 渲染进程
## 处理网页内容
**渲染进程**负责标签页内发生的一切。在渲染程序进程中，**主线程**会处理您发送给用户的大多数代码。有时，如果您使用 Web Worker 或 Service Worker，则部分 JavaScript 将由**工作器线程**处理。**合成器**和**光栅线程**也会在渲染程序进程内运行，以便高效、流畅地渲染页面。
渲染程序进程的核心任务是将 HTML、CSS 和 JavaScript 转换为用户可与之互动的网页。
![[Pasted image 20240525000753.png]]
## 解析
### 构建DOM
当渲染器进程收到导航的提交消息并开始接收 HTML 数据时，主线程开始解析文本字符串 (HTML)，并将其转换为一个对象**模**态 (DOM)。DOM 是浏览器的内部网页表示形式，也是 Web 开发者可通过 JavaScript 与之交互的数据结构和 API。
### 加载子资源
网站通常使用图片、CSS 和 JavaScript 等外部资源。这些文件需要从网络或缓存加载，主线程在解析以构建 DOM 时，可以逐个请求这些元素，但为了加快速度，系统会并发运行“预加载扫描程序”。如果 HTML 文档中包含 `<img>` 或 `<link>` 等内容，预加载扫描器会查看由 HTML 解析器生成的令牌，并将请求发送到浏览器进程中的网络线程。

![[Pasted image 20240525001040.png]]
### JS可能会阻止
当 HTML 解析器找到 `<script>` 标记时，它会暂停解析 HTML 文档，并必须加载、解析并执行 JavaScript 代码。为什么呢？因为 JavaScript 可以使用 `document.write()` 之类的东西改变文档的形状，而这会改变整个 DOM 结构（HTML 规范中的[解析模型概述](https://html.spec.whatwg.org/multipage/parsing.html#overview-of-the-parsing-model)有一个很不错的图表）。因此，HTML 解析器必须先等待 JavaScript 运行完毕，然后才能继续解析 HTML 文档。如果您想了解 JavaScript 执行过程中会发生什么，[V8 团队会对此发表演讲和撰写博文](https://mathiasbynens.be/notes/shapes-ics)。
### 提示浏览器如何加载资源
Web 开发者可以通过多种方式向浏览器发送提示，以便妥善加载资源。如果您的 JavaScript 未使用 `document.write()`，您可以将 [`async`](https://developer.mozilla.org/docs/Web/HTML/Element/script#attr-async) 或 [`defer`](https://developer.mozilla.org/docs/Web/HTML/Element/script#attr-defer) 属性添加到 `<script>` 标记中。然后，浏览器会异步加载并运行 JavaScript 代码，也不会阻止解析。您也可以使用 [JavaScript 模块](https://developers.google.com/web/fundamentals/primers/modules?hl=zh-cn)（如果适用）。`<link rel="preload">` 用于通知浏览器当前导航确实需要该资源，而您希望尽快下载该资源。如需了解详情，请参阅[资源优先级 - 让浏览器为您提供帮助](https://developers.google.com/web/fundamentals/performance/resource-prioritization?hl=zh-cn)。

## 样式计算
拥有 DOM 并不足以了解网页是什么样子，因为我们可以通过 CSS 设置页面元素的样式。主线程解析 CSS 并确定为每个 DOM 节点计算出的样式。了解根据 CSS 选择器将何种样式应用于每个元素。您可以在开发者工具的 `computed` 部分查看此信息。

## 布局
渲染程序进程已经知道文档的结构以及每个节点的样式，但这不足以渲染网页。假设您正尝试通过电话向朋友描述一幅画。“一个大的红色圆圈和一个小的蓝色方块”不足以让您的好友知道这幅画到底是什么样子。
![[Pasted image 20240525002937.png]]
布局是查找元素几何形状的过程。主线程会遍历 DOM 和计算出的样式，并创建包含 x y 坐标和边界框大小等信息的布局树。布局树的结构可能与 DOM 树类似，但它仅包含与页面上可见内容相关的信息。如果应用 `display: none`，则该元素不属于布局树的一部分（但是，具有 `visibility: hidden` 的元素位于布局树中）。同样，如果应用了内容类似于 `p::before{content:"Hi!"}` 的伪类，则它会包含在布局树中（即使它不在 DOM 中）。
![[Pasted image 20240525003015.png]]

## 绘制
![[Pasted image 20240525003446.png]]
拥有 DOM、样式和布局仍然不足以渲染网页。假设您正试图再现一幅画您知道元素的大小、形状和位置，但仍需要判断绘制元素的顺序。
例如，可能会为某些元素设置 `z-index`，在这种情况下，按 HTML 中编写的元素顺序绘制会导致呈现错误。
![[Pasted image 20240525003415.png]]
在此绘制步骤中，主线程会遍历布局树以创建绘制记录。绘制记录是绘制过程的备注，例如“先背景，然后是文本，最后是矩形”。如果您使用 JavaScript 在 `<canvas>` 元素上绘制，那么您可能很熟悉此过程。
![[Pasted image 20240525003648.png]]
### 更新渲染流水线的成本高昂

![[d7zOpwpNIXIoVnoZCtI9.mp4]]
<center>DOM+样式、布局和绘制树（按生成顺序排列）</center>
在渲染流水线中，最重要的一点是，在每一步中，前一个运算的结果都会用于创建新数据。例如，如果布局树发生变化，则需要为文档的受影响部分重新生成绘制顺序。

## 合成
![[AiIny83Lk4rTzsM8bxSn.mp4]]
<center>简单光栅过程的动画</center>
### 您将如何绘制页面
浏览器已经知道文档的结构、每个元素的样式、页面的几何图形和绘制顺序，接下来该如何绘制页面呢？将这些信息转换为屏幕上的**像素**就称为光栅化。

一个简单的处理方法就是对**视口内**的部分进行光栅处理。如果用户滚动页面，则移动光栅帧，并通过更多光栅化来填充缺失部分。这就是 Chrome 首次发布光栅化时的处理方式。不过，现代浏览器运行着一个更复杂的过程，即合成。
### 什么是合成
![[Aggd8YLFPckZrBjEj74H.mp4]]
合成是一种技术，可将页面的各个部分分离成图层，单独将其光栅化，然后在单独的线程（称为合成器线程）中合成为页面。如果发生滚动，由于图层已经光栅化，您只需合成一个新帧即可。通过移动层和合成新帧，可以采用相同的方式实现动画。

### 分成多个图层

![[Pasted image 20240525005815.png]]
  
为了找出哪些元素需要位于哪些层，**主线程**会遍历布局树来创建层树（此部分在开发者工具的“性能”面板中称为“更新层树”）。如果网页中本应成为单独图层的某些部分（例如滑入式侧边菜单）没有显示该部分，那么您可以在 CSS 中使用 `will-change` 属性来提示浏览器。
您可能想要为每个元素都赋予层，但与过多的层进行合成可能会导致操作速度比每帧光栅化地对网页的一小部分进行光栅化，因此衡量应用的渲染性能至关重要。如需详细了解该主题，请参阅[坚持仅合成器的属性和管理层数](https://developers.google.com/web/fundamentals/performance/rendering/stick-to-compositor-only-properties-and-manage-layer-count?hl=zh-cn)。

### 主线程以外的光栅图像和合成图像
图层树创建完毕并确定绘制顺序后，**主线程**会将该信息提交到**合成器线程**。然后，合成器线程会光栅化每个图层。图层可能像页面的整个长度一样大，因此合成器线程会将其划分为图块，并将每个图块发送到**光栅线程**。光栅线程会光栅化每个图块并将其存储在 GPU 内存中。
![[Pasted image 20240525010206.png]]
合成器线程可以优先处理不同的光栅线程，以便首先对视口（或附近）中的内容进行光栅化。一个图层还具有多个适用于不同分辨率的平铺，以处理放大操作等任务。
将图块进行光栅化后，合成器线程会收集称为“**绘制四边形**”的图块信息，以创建**合成器帧**。

*绘制四边形：包含功能块在内存中的位置，以及考虑到页面合成的情况下要绘制功能块在页面中的位置等信息。*
*合成器帧：一组绘制四边形，表示一个页面帧。*

然后通过 IPC 将合成器帧提交到浏览器进程。此时，对于浏览器界面更改，可以从界面线程添加另一个合成器帧，对于扩展程序，可以从其他渲染程序进程添加。这些合成器帧会发送到 GPU 以在屏幕上显示。如果滚动事件传入，合成器线程会创建另一个要发送到 GPU 的合成器帧。
![[Pasted image 20240525010622.png]]
合成的优势在于，它在完成时不涉及主线程。合成器线程不需要等待样式计算或 JavaScript 执行。因此，[仅合成动画](https://www.html5rocks.com/en/tutorials/speed/high-performance-animations/)被认为是实现流畅性能的最佳方式。如果需要再次计算布局或绘制，则必须涉及主线程。

## 延伸内容
移动元素的动画推荐使用 transform 属性的translate，rotate，scale等，以及opacity。尽量避免使用触发布局或者重绘的属性，确定属性对渲染流水线的影响。
### 渲染管道
要在网页上显示内容，浏览器必须执行以下依序步骤：
1. **样式**：计算应用于元素的样式。
2. **布局**：为每个元素生成几何图形和位置。
3. **绘制**：将每个元素的像素填充到层中。
4. **Composite**：将图层绘制到屏幕上。
当您为已加载网页上的内容添加动画效果时，必须重复执行这些步骤。此过程从为了播放动画而必须更改的步骤开始。如前所述，这些步骤是按顺序执行的。例如，如果您为会更改布局的内容添加动画效果，那么绘制和合成步骤也必须再次运行。因此，为会更改布局的内容添加动画效果比为仅更改合成的内容设置动画的开销更高。
### 为布局属性添加动画效果
布局更改涉及计算受此更改影响的所有元素的几何图形（位置和大小）。如果您更改了某个元素，则可能需要重新计算其他元素的几何图形。例如，如果您更改 `<html>` 元素的宽度，则它的任何子元素都可能会受到影响。由于元素溢出和相互影响的方式，在树中再向下的更改有时可能会导致布局计算回到顶端。
可见元素树越大，执行布局计算所需的时间就越长。
### 为绘制属性添加动画效果
[绘制](https://developer.chrome.com/blog/inside-browser-part3?hl=zh-cn#paint)是确定应按何种顺序将元素绘制到屏幕上的过程。它通常是流水线中运行时间最长的任务。
在现代浏览器中，大多数绘制都是在[软件光栅化工具](https://software.intel.com/content/www/us/en/develop/articles/software-vs-gpu-rasterization-in-chromium.html)中完成的。根据将应用中的元素分组为层的方式，除了已更改的元素之外，可能还需要对其他元素进行绘制。
### 为复合属性添加动画效果
[合成](https://developer.chrome.com/blog/inside-browser-part3?hl=zh-cn#what-is-compositing)是将页面拆分为多个图层、将有关网页外观的信息转换为像素（光栅化），以及将图层组合在一起以创建页面（合成）的过程。
因此，`opacity` 属性包含在制作动画开销很低的内容列表中。只要此属性位于自己的层中，GPU 就可以在合成步骤中处理对其的更改。基于 Chromium 的浏览器和 WebKit 会为在 `opacity` 上具有 CSS 过渡或动画的任何元素创建一个新层。
### 什么是图层？
通过将具有动画效果或过渡到新图层的内容放置到新图层上，浏览器只需要重新绘制这些内容，而无需重新绘制其他内容。您可能比较熟悉 Photoshop 的“图层”概念，图层包含一系列可以一起移动的元素。浏览器渲染层与此类似。
### CSS 与 JavaScript 的性能对比
您可能会好奇：从性能的角度来看，为动画使用 CSS 还是 JavaScript 更好吗？
基于 CSS 的动画和[网页动画](https://web.dev/articles/web-animations?hl=zh-cn)（在支持该 API 的浏览器中）通常在称为“合成器线程”的线程上处理。这与浏览器的主线程不同，主线程在主线程中执行样式、布局、绘制和 JavaScript 操作。 这意味着，如果浏览器正在主线程上运行一些开销很大的任务，则这些动画可以继续播放而不会中断。
如本文所述，在许多情况下，对变形和不透明度进行的其他更改也可由合成器线程处理。
如果任何动画触发了绘制和/或布局，则需要主线程执行工作。 这一点对 CSS 和 JavaScript 动画而言都是如此，布局或绘制的开销可能会使与 CSS 或 JavaScript 执行相关的任何工作变得轻而易举，使问题变得毫无意义。

reading：
https://developer.mozilla.org/zh-CN/docs/Web/Performance/How_browsers_work
https://www.smashingmagazine.com/2016/12/gpu-animation-doing-it-right/

# 事件
当发生用户手势（如轻触屏幕）时，**浏览器进程**是最先收到该手势的进程。不过，浏览器进程只会知道该手势的发生**位置**，因为标签页中的内容是由**渲染进程**处理的。因此，浏览器进程会将**事件类型**（例如 `touchstart`）及其**坐标**发送到渲染进程。渲染程序进程通过**查找事件目标**并运行附加的**事件监听器**来适当地处理事件。
![[Pasted image 20240525135710.png]]
我们了解了合成器如何通过合成光栅化图层来流畅地处理滚动。如果页面未附加任何输入事件监听器，合成器线程可以完全独立于主线程创建新的复合帧。但是，如果页面中附加了一些事件监听器，该怎么办？合成器线程如何确定是否需要处理该事件？
## 了解不可快速滚动的区域
由于运行 JavaScript 是主线程的作业，因此在合成页面时，**合成器线程**会将附加了事件处理程序的页面区域标记为“**非快速滚动区域**”。有了这些信息，合成器线程可以确保在输入事件发生时将该输入事件发送到**主线程**。如果输入事件来自此区域之外，则合成器线程会继续合成新帧，而不会**等待**主线程。
### 编写事件处理脚本时请注意
Web 开发中常见的事件处理模式是事件委托。由于事件会以气泡形式显示，因此您可以在最顶层的元素附加一个事件处理脚本，并根据事件目标委派任务。您可能已经看到或编写过如下代码。
```
document.body.addEventListener('touchstart', event => {    if (event.target === area) {        event.preventDefault();    }});
```
由于您只需为所有元素编写一个事件处理脚本，因此此事件委托模式的工效学设计非常具有吸引力。不过，如果您从浏览器的角度查看此代码，就会发现整个页面现在会被标记为非快速滚动区域。这意味着，即使应用并不关注来自页面某些部分的输入，合成器线程也必须与**主线程通信**，并在每次有输入事件传入时等待。因此，合成器的流畅滚动功能会受到影响。
![[Pasted image 20240525140320.png]]为避免发生这种情况，您可以在事件监听器中传递 `passive: true` 选项。这会提示浏览器您仍想监听主线程中的事件，但合成器也可以继续合成新帧。[More Info](https://developer.mozilla.org/en-US/docs/Web/API/EventTarget/addEventListener#using_passive_listeners).
```
document.body.addEventListener('touchstart', event => {    if (event.target === area) {        event.preventDefault()    } }, {passive: true});
```
%% MDN
If an event has a default action — for example, a [`wheel`](https://developer.mozilla.org/en-US/docs/Web/API/Element/wheel_event "wheel") event that scrolls the container by default — the browser is in general unable to start the default action until the event listener has finished, because it doesn't know in advance whether the event listener might cancel the default action by calling [`Event.preventDefault()`](https://developer.mozilla.org/en-US/docs/Web/API/Event/preventDefault). If the event listener takes too long to execute, this can cause a noticeable delay, also known as [jank](https://developer.mozilla.org/en-US/docs/Glossary/Jank), before the default action can be executed.

By setting the `passive` option to `true`, an event listener declares that it will not cancel the default action, so the browser can start the default action immediately, without waiting for the listener to finish. If the listener does then call [`Event.preventDefault()`](https://developer.mozilla.org/en-US/docs/Web/API/Event/preventDefault), this will have no effect.
%%
## 检查活动是否可以取消
![[Pasted image 20240525141115.png]]
假设您在网页上有一个框，您希望将滚动方向限制为仅水平滚动。
在指针事件中使用 `passive: true` 选项意味着页面可以流畅滚动，但垂直滚动可能在您希望 `preventDefault` 时开始，以限制滚动方向。您可以使用 `event.cancelable` 方法对此进行检查。
```
document.body.addEventListener('pointermove', event => {    
	if (event.cancelable) {        
		event.preventDefault(); // block the native scroll 
		/* do what you want the application to do here  */    
	}
}, {passive: true});
```
或者，您也可以使用 CSS 规则（如 `touch-action`）来完全删除事件处理脚本。
```
#area {  touch-action: pan-x;}
```
## 查找事件目标
![[Pasted image 20240525153901.png]]
当合成器线程向主线程发送输入事件时，首先要运行的是查找事件目标的命中测试。点击测试会使用在渲染过程中生成的绘制记录数据，找出事件发生点坐标下方的内容。
## 尽量减少分派给主线程的事件
在上一篇博文中，我们讨论了典型的显示屏每秒刷新 60 次屏幕，以及如何跟上流畅动画的节奏。对于输入，典型触摸屏设备每秒传送触摸事件 60-120 次，典型鼠标每秒传送事件 100 次。输入事件的保真度高于屏幕可以刷新的保真度。
如果像 `touchmove` 这样的连续事件每秒发送到主线程 120 次，则可能会触发过多的命中测试和 JavaScript 执行（与屏幕刷新速度相比）。
![[Pasted image 20240525154202.png]]
为了最大限度地减少对主线程的过多调用，Chrome 会合并连续事件（例如 `wheel`、`mousewheel`、`mousemove`、`pointermove`、`touchmove`），并将调度延迟到下一个 `requestAnimationFrame` 之前。
![[Pasted image 20240525154401.png]]
系统会立即分派 `keydown`、`keyup`、`mouseup`、`mousedown`、`touchstart` 和 `touchend` 等离散事件。
## 使用 `getCoalescedEvents` 获取帧内事件
对于大多数 Web 应用，合并事件应足以提供良好的用户体验。不过，如果您要构建应用以及根据 `touchmove` 坐标设置路径等，为了绘制平滑线条，可能会丢失两者之间的坐标。在这种情况下，您可以在指针事件中使用 `getCoalescedEvents` 方法来获取有关这些合并事件的信息。
![[Pasted image 20240525154418.png]]
```
window.addEventListener('pointermove', event => {    const events = event.getCoalescedEvents();    for (let event of events) {        const x = event.pageX;        const y = event.pageY;        // draw a line using x and y coordinates.    }});
```





