<<<<<<< HEAD
[The article](https://web.dev/articles/howbrowserswork) was way old, written in 2011, when CSS3 and Blink engine were not released. Hopefully, the principles have not changed much. 
```table-of-contents
```
# 高层架构(High-level infrastructure)
1. 用户界面：包括地址栏、前进/后退按钮、书签菜单等。显示在请求的网页之外。
2. 浏览器引擎：marshals actions between the UI and the rendering engine。
3. 渲染引擎：负责显示请求的内容。如果请求的是HTML，会解析HTML和CSS，并把解析的内容显示在屏幕上。
4. 网络：负责http请求等网络调用。
5. 用户界面后端：用于绘制基本 widget，如组合框和窗口。此后端公开了与平台无关的通用接口。在底层使用操作系统界面方法。
6. Javascript解释器：解析和执行JS代码。
=======
# 高层架构(High-level infrastructure)
1. 用户界面：包括地址栏、前进/后退按钮、书签菜单等。显示在请求的网页之外。
2. 浏览器引擎：marshals actions between the UI and the rendering engine.
3. 渲染引擎：负责显示请求的内容。如果请求的是HTML，会解析HTML和CSS，并把解析的内容显示在屏幕上。
4. 网络：负责http请求等网络调用。
5. 用户界面后端：用于绘制基本 widget，如组合框和窗口。此后端公开了与平台无关的通用接口。在底层使用操作系统界面方法。
6. Javascript解释器：解析和执行JS代码
>>>>>>> 4905b57dfce6b5cca637ffcf7915eb49dec0f288
7. 数据存储。这是一个持久层，在内地把偶你各种数据，cookies, localStorage, IndexDB等。
![[Pasted image 20240530165806.png]]
# 渲染引擎主流程
![[Pasted image 20240530165908.png]]
![[Pasted image 20240530170707.png]]
<center>WebKit主流程</center>
![[Pasted image 20240530170735.jpg]]
<center>Mozilla 的 Gecko 渲染引擎主流程</center>
## 解析 - 常规
解析意味着将文档转换为可供代码使用的结构，其结果通常是表示文档结构的节点数，这称为parse tree 或 syntax tree。
解析表达式 `2 + 3 - 1` 可能会返回以下树：
![[Pasted image 20240530174959.png]]
## 语法
解析基于文档所遵循的语法规则（编写文档时所使用的语言或格式）。所有可以解析的格式都必须包含，由**词法**（vocabulary）和**语法规则**组成的**确定性语法**。这被叫做**与上下文无关的语法**（context free grammar)。人类语言不属于这种语言，所以不能被常规的解析方法解析。
## 解析器和词法分析器的组合(Parser - Lexer combination)
（注意token同时被翻译为“标记”或“词元”的混乱，markup尽量翻译为“标签”，hopefully the meaning could make sense）
<<<<<<< HEAD

=======
>>>>>>> 4905b57dfce6b5cca637ffcf7915eb49dec0f288
解析可以分为两个子过程：词法分析和语法分析(lexical analysis and syntax analysis)。

词法分析是将输入内容分解成多个词元(tokens)的过程。词元是语言词汇，即有效组成要素的集合（collection of valid building blocks)。在人类语言中，它由一门语言的字典中出现的所有单词组成。
Lexical analysis is the process of breaking the input into tokens. Tokens are the language vocabulary: the collection of valid building blocks. In human language it will consist of all the words that appear in the dictionary for that language.
语法分析是指语法规则的应用。

解析通常会将工作划分到两个组件之间：词法分析器（又称为标记生成器），它负责将输入内容分解为有效标记(valid tokens)，而解析器则负责根据语言语法规则分析文档结构进而构建解析树。
<<<<<<< HEAD

词法分析器知道如何删除不相关的字符，例如空格和换行符。（在`2+3-1`的例子中，是分解并获取数字2，3，1和操作符+，-的过程。）

解析器通常会向词法分析器要一个新标记(token)，并尝试将该token与某条语法规则匹配。如果规则匹配，则会将token对应的节点加到解析树中，然后解析器会请求另一个token。这个过程是迭代的。

=======
词法分析器知道如何删除不相关的字符，例如空格和换行符。（在`2+3-1`的例子中，是分解并获取数字2，3，1和操作符+，-的过程。）
解析器通常会向词法分析器要一个新标记(token)，并尝试将该token与某条语法规则匹配。如果规则匹配，则会将token对应的节点加到解析树中，然后解析器会请求另一个token。这个过程是迭代的。
>>>>>>> 4905b57dfce6b5cca637ffcf7915eb49dec0f288
如果没有规则匹配，解析器会在内部存储token，并不断请求令牌，直到发现一条规则匹配所有内部存储的token。如果没有这样的规则，解析器会引发异常，这意味着文档无效，包含语法错误。
![[Pasted image 20240530182143.png]]

## 翻译
在许多情况下，解析树不是最终产品。解析通常在翻译过程中使用：把输入文档转换为另一种格式。
编译就是一个例子，它先把源代码解析为解析树然后再翻译为机器码文件。
![[Pasted image 20240530202724.png]]
再下图中，我们通过一个数学表达式构建了一个解析树。
![[Pasted image 20240530174959.png]]
 <center>2+3-1</center>
语法：
1. 语言语法构建块（language syntax building blocks）是表达式、项（terms）和运算符。
2. 我们的语言可以包含任意数量的表达式
3. 表达式的定义是：一个项后面跟一个运算符，再跟另一项
4. 操作符是加号或者减号
5. 一个项是一个整数或者表达式
与规则匹配的第一个子字符串是`2`：根据rule 5它是一个项；
第二个匹配是`2+3`：rule 3；
下一次匹配在输入的最后：`2+3-1`是一个表达式，因为已知`2+3`是一个项。

## 词汇和语法的正式定义
词汇通常由**正则表达式**表示。
刚才的语言可以定义为
```
INTEGER: 0|[1-9][0-9]*
PLUS: +
MINUS: -
```
语法通常以[BNF](http://en.wikipedia.org/wiki/Backus%E2%80%93Naur_Form)的格式定义，如下
```
expression :=  term  operation  term
operation :=  PLUS | MINUS
term := INTEGER | expression
```
**[与上下文无关的语法](http://en.wikipedia.org/wiki/Context-free_grammar)的直接定义，就是可以完全用BNF表示的语法。它可以被常规解析器解析。**
## 解析器类型
分为自上而下的解析器和自下而上的解析器。
自上而下的解析器检测语法的高层次结构，并尝试找到一条规则匹配。自下而上的解析器从输入开始，并且逐渐把它转化为语法规则。，从低层次规则开始直到高层次匹配。
（高低层次，指的是语法定义的结构的层次，前例中，term是低层次，expression是高层次。）

在上面的示例中`2+3-1`:
自上而下的解析器会从更高层次开始：它会将`2+3`识别为表达式，然后识别`2+3-1`为表达式。识别过程是渐进的，进而匹配其他规则，从起点是最高层的规则
自下而上的解析器将扫描输入内容，知道规则匹配。然后将匹配的输入替换为规则。持续到输入内容的结尾。部分匹配的表达式会放在解析器的栈中。

| Stack                | Input     |
| -------------------- | --------- |
|                      | 2 + 3 - 1 |
| term                 | + 3 - 1   |
| term operation       | 3 - 1     |
| expression           | - 1       |
| expression operation | 1         |
| expression           |           |
（留意前面的表述，匹配的输入被替换为了规则）
这种自下而上的解析器也叫做移位归约解析器（shift-reduce parser），因为输入会向右移动（指向输入首位的指针向右移动），并逐渐被归纳到语法规则。
## Generating parsers automatically
There are tools that can generate a parser. You feed them the grammar of your language - its vocabulary and syntax rules - and they generate a working parser. Creating a parser requires a deep understanding of parsing and it's not easy to create an optimized parser by hand, so parser generators can be very useful.

WebKit uses two well known parser generators: [Flex](http://en.wikipedia.org/wiki/Flex_lexical_analyser) for creating a lexer and [Bison](http://www.gnu.org/software/bison/) for creating a parser (you might run into them with the names Lex and Yacc). Flex input is a file containing regular expression definitions of the tokens. Bison's input is the language syntax rules in BNF format.
<<<<<<< HEAD
# HTML解析
=======
# HTML解析器
>>>>>>> 4905b57dfce6b5cca637ffcf7915eb49dec0f288
HTML解析器的任务是将HTML标签解析为解析树。
## HTML语法
HTML的词汇和语法由W3C组织创建的规范定义。不过，所有的传统解析器都不适用于HTML（前面掰扯了半天的所谓传统解析器可以用来解析CSS和JS）。
HTMl无法用解析器所需的上下文无关的的语法(context-free grammar, CFG)轻松定义，也无法用BNF格式定义（上文的直接定义）。
不过HTML可以被一种角DTD(Document Type Definition)的正式格式定义，但不是CFG。

这乍看起来很奇怪：HTML很接近XML。存在很多XML解析器可用，也有一种XML变体的HTML-XHTML，所以这有什么很大区别呢？
不同之处在于HTML更加“宽容”：它允许你省略某些特定标记（会被隐式添加），有时也可以省略开始或者结束标记。总体而言，它是一种“软”语法，与XML严格的语法相反。

这个看起来微不足道的细节造成很大不同。一方面这是HTML如此流行的原因：允许一定的错误，简化网络作者的工作。另一方面，增加了写一个正式语法的难度。
总之，常规解释器无法轻易解析HTML，因为它是上下文有关的。所以，HTML语法定义采用DTD格式

## DOM
解析树是由DOM元素节点和属性节点组成的树。DOM是HTML文本的对象呈现方式，也是HTML元素与外部世界，例如JS，的接口。
树的根是`Document`对象，DOm与标签之间几乎是一对一关系。
```
<html>  
	<body>    
		<p>      
			Hello World    
		</p>    
		<div> 
			<img src="example.png"/>
		</div>  
	</body>
</html>
```
![[Pasted image 20240530235142.png]]
<<<<<<< HEAD
## 解析算法
=======
### 解析算法
>>>>>>> 4905b57dfce6b5cca637ffcf7915eb49dec0f288
HTML无法使用常规解析器，因为：
1. 语言的宽容本质。
2. 浏览器传统上具有容错能力的事实，来支持无效的HTML
3. 解析过程会反复进行。对于其他语言，源代码在解析过程中不会更改，但在 HTML 中，动态代码（例如包含 `document.write()` 调用的JS）可能会添加额外的标签，因此解析过程实际上会修改输入。

浏览器的HTML解析算法[由HTML5标准详细定义](http://www.whatwg.org/specs/web-apps/current-work/multipage/parsing.html)，算法包含两部分，**词元化**(Tokenization)和**树构建**。
词元化是词法分析，将输入内容解析成词元。HTML 词元包括开始标记、结束标记、属性名称和属性值。
标记生成器识别标记，将其提供给树构造函数，然后消耗下一个字符以识别下一个标记，依此类推，直到输入的末尾。
![[parsing-model-overview.svg]]
### 标记化算法
该算法被表示为状态机，它的输出结果是HTML标记（token)。每个状态使用输入流的一个或多个字符，并根据这些字符更新下一个状态。当前的标记化状态和树构建状态会影响该决定。这意味着，用相同的字符，也可能会得到不同的正确新状态，具体取决于当前状态。
该算法太复杂（这描述也已经够抽象了），以下是一个简单示例：
```
<html>  
	<body>    
		Hello world  
	</body>
</html>
```
%%此处token译为标记，tag为标签，emitted为提交%%
1. 初始状态为“**数据状态**”(Data State)。
2. 当遇到`<`字符，状态变为“**标签打开状态**”(tag open state)。
3. 遇到一个`a-z`字符进入“**标签名称状态**”（tag name state）。并保持这个状态到遇到`>`字符。每个字符都会添加进新的标记（token）名称中，在这个例子中，先创建的是一个`html` 标记。
4. 到达`>`字符时，当前标记提交，状态回到“**数据状态**”。`body`标记的处理相同
5. 遇到`Hello world` 的`H`字符，会创建并提交一个字符标记，直到到达`</body>` 的`<`为止，这时回到“**标签打开状态**”。
6. 遇到`/`字符会创建一个`end tag` 标记并进入“**标签名字状态**”。
7. 到达`>`后，`</html>`跟`<html>`一样，会生成`/html`标记，新的标签标记会提交，返回**数据状态**。

![[Pasted image 20240531004418.png]]
<<<<<<< HEAD
=======

>>>>>>> 4905b57dfce6b5cca637ffcf7915eb49dec0f288
### 树构建算法
解析器被创建时，文档对象也会被创建。
在树构建阶段，以Document为根的DOM树会被修改，并添加元素。每个被标记生成器（tokenizer）提交的节点都将由树构造函数处理。对每个标记，规范都会定义相应的DOM元素，然后DOM元素会被创建。这个元素会被添加到DOM树，以及一个存放开放元素的栈上。栈被用来更正嵌套错误和未关闭的标签。该状态也可以被描述为状态机，这些状态被称为“插入模式”(insertion modes)。
```
<html>  
	<body>    
		Hello world  
	</body>
</html>
```
树构建阶段的输入是来自来自标记化阶段的一些列标记。
（模式中的before，in，after的描述时相对于当前DOM树结构而言的，例如，before html意思是在当前DOM树插入html节点前）
1. 第一种模式时“初始模式”。
2. 收到`html`标记后，系统进入”before html“模式，并在该模式重新处理标记。
3. 这会创建`HTMLHTMLElement`元素，并附加到Document根对象。状态更改为”before head“。
5. 然后收到`body`标记，虽然示例代码中没有`head`标记，但会隐式创建`HTMLHeadElement`，并加入树中。
6. 现在，我们进入了”in head“模式，然后进入”after head“模式。
7. 系统重新处理`body`标记，创建并插入`HTMLBodyElement`，同时模式转变为”in body“。
9. 现在接受”hello world“字符串的字符标记，第一个字符用于创建和插入”Text“节点，其他字符将附加到该节点。（似乎的确是，字符标记是逐个创建和提交的）
10. 接受body结束标记将转换为”after body“模式
11. 接受html结束标记会进入”after after body“模式，接受文件结束标记会结束解析。
![[tree-construction-exampl-4e9757a851f96.gif]]
解析完成后的操作：
在这个阶段，浏览器会把文档标记为可交互的（interactive），并且开始解析被标记为”deferred“的脚本。
文档状态随后设置为”完成“（complete），并且一个”（已）读取“（load）事件会触发。这里的事件应该是[`DOMContentLoaded`](https://developer.mozilla.org/zh-CN/docs/Web/API/Document/DOMContentLoaded_event)事件，这个事件本身不会等待样式表加载，但如果有`deferred`脚本，脚本会等待样式表加载。该事件也不等待`async`脚本和图片等资源。`load` 事件在整个页面及所有依赖资源如样式表和图片都已完成加载时触发。

### 浏览器的容错能力
浏览网页时，基本不会看到“无效语法”的错误，因为浏览器会修复无效内容并继续运行。

```
<html>  
	<mytag>  
	</mytag>  
	<div>  
		<p>  
	</div>    
		Really lousy HTML  
	</p>
</html>
```
We have to take care of at least the following error conditions:
1. The element being added is explicitly forbidden inside some outer tag. In this case we should close all tags up to the one which forbids the element, and add it afterwards.
2. We are not allowed to add the element directly. It could be that the person writing the document forgot some tag in between (or that the tag in between is optional). This could be the case with the following tags: HTML HEAD BODY TBODY TR TD LI (did I forget any?).
3. We want to add a block element inside an inline element. Close all inline elements up to the next higher block element.
4. If this doesn't help, close elements until we are allowed to add the element - or ignore the tag.
[Examples](https://web.dev/articles/howbrowserswork?hl=zh-cn#browsers_error_tolerance)

# CSS 解析
CSS是一种CFG
```
comment   \/\*[^*]*\*+([^/*][^*]*\*+)*\/
num       [0-9]+|[0-9]*"."[0-9]+
nonascii  [\200-\377]
nmstart   [_a-z]|{nonascii}|{escape}
nmchar    [_a-z0-9-]|{nonascii}|{escape}
name      {nmchar}+
ident     {nmstart}{nmchar}*
```
“ident”是标识符 (identifier) 的缩写，例如类名称。 “name”是元素 ID（以“#”表示）

```
ruleset  
	: selector [ ',' S* selector ]*    '{' S* declaration [ ';' S* declaration ]* '}' S*  
	;
selector  
	: simple_selector [ combinator selector | S+ [ combinator? selector ]? ]?  
	;
simple_selector  
	: element_name [ HASH | class | attrib | pseudo ]*  | [ HASH | class | attrib | pseudo ]+  
	;
class  
	: '.' IDENT  
	;
element_name  
	: IDENT | '*'  
	;
attrib  
	: '[' S* IDENT S* [ [ '=' | INCLUDES | DASHMATCH ] S*    [ IDENT | STRING ] S* ] ']'  
	;
pseudo  
	: ':' [ IDENT | FUNCTION S* [IDENT S*] ')' ]  
	;
```
A ruleset: 
```
div.error, a.error {  color:red;  font-weight:bold;}
```

`div.error` 和 `a.error` 是选择器。大括号内的部分包含由此规则集应用的规则。该结构的正式定义如下：

```
ruleset  
	: selector [ ',' S* selector ]*    '{' S* declaration [ ';' S* declaration ]* '}' S*  
	;
```
这意味着规则集是一个选择器，或者多个由英文逗号和空格（S 代表空格）分隔的多个选择器。 规则集包含大括号，其中包含一个声明，或者多个由英文分号分隔的声明（可选）。“声明”和“选择器”将在下面的 BNF 定义中定义。

## WebKit CSS 解析器
WebKit使用Flex（词法）和Bison（语法）解析生成器，根据CSS语法文件自动生成解析器。Bison会创建自底向上的语法解析器。Firefox使用一个手写的自顶向下的语法解析器。这两种方法都会把CSS文件解析为一个StyleSheet对象。每个对象都包含CSS规则。CSS规则对象包含选择器和声明对象，以及其他符合CSS语法的对象。
![[Pasted image 20240601011159.png]]

# 脚本和样式表的处理顺序
## 脚本
Web的模型是同步的。解析器遇到 `<script>`标签时会立即解析并执行脚本。文档解析会停止，等待脚本执行完。如果脚本是外部的，资源必须先从网络下载完。（在没有添加`defer`或`async`属性的情况下）
## 推测解析
WebKit和Firefox均会执行此优化。执行脚本时，另一个线程会解析文档的其余部分，找出并加载需要从网络加载的其他资源。以这种方式，资源可通过**并行**的连接加载，从而提高加载速度。

（推测解析器-speculative parser-只加载外部资源，不修改DOM结构，详见Perfomance的Preload Scanner部分）
## 样式表
样式表采用不同的模型。由于CSS不会更改DOM树，似乎没有必要等待样式表并停止文档解析。但这样会出现
FOUC—Flash of unstyled content—问题，以及因为脚本可能请求样式信息，如果样式尚未加载解析，脚本会获得错误答案。
（阻塞解析器的 `<script>` 必须等待所有阻塞渲染的 CSS 资源到达并得到解析，然后浏览器才能执行这些资源）
如果存在正在加载和解析的样式表，Firefox会阻止所有脚本。WebKit只有当脚本尝试访问可能受未加载样式表影响的样式属性时，才会阻塞脚本。
# 渲染树构建
在DOM树构建期间，浏览器还会构建渲染树。这棵树由可视元素组成，并按它们的显示顺序排列，是文档的可视化表示。其目的是确保按正确的顺序绘制内容。
Firefox称树中的元素为“frames”，Webkit称之为"renderer"或渲染对象。

渲染对象知道如何布局，如何绘制自身及其子元素。
Webkit的RenderObject类，渲染对象的基类，由如下定义：
```
class RenderObject{  
	virtual void layout();  
	virtual void paint(PaintInfo);  
	virtual void rect repaintRect();  
	Node* node;  //the DOM node  
	RenderStyle* style;  // the computed style  
	RenderLayer* containgLayer; //the containing z-index layer
}
```
每个渲染对象表示一个矩形区域，通常对应于节点的CSS盒子，包含宽高和位置等**几何**信息

盒子的类型受到`display`属性的影响。以下是决定根据显示属性，决定渲染对象类型的WebKit代码
```
RenderObject* RenderObject::createObject(Node* node, RenderStyle* style){
	Document* doc = node->document();    
	RenderArena* arena = doc->renderArena();    
	...    
	RenderObject* o = 0;    
	switch (style->display()) { 
		case NONE:            
			break;        
		case INLINE:            
			o = new (arena) RenderInline(node);            
			break;        
		case BLOCK:            
			o = new (arena) RenderBlock(node);            
			break;        
		case INLINE_BLOCK:            
			o = new (arena) RenderBlock(node);            
			break;        
		case LIST_ITEM:            
			o = new (arena) RenderListItem(node);            
			break;       
		...    
	}    
	return o;
}
```
元素类型也会被考虑，例如，表达控件和表格就有特殊的对象。

在WebKit中，如果一个元素要创建一个特殊渲染对象，它会重写`createRenderer()`方法。渲染对象会指向包含非几何信息的样式对象。

## 渲染树与DOM树的关系
渲染对象对应于DOM元素，但并非严格一一对应。非可视DOM元素不会插入渲染树。一个例子是head元素。当然一个display属性是none的元素也不会出现在渲染树中（不过hidden会）。

有的DOM元素对应多个可视对象。这些元素通常是结构复杂的元素，无法用单个矩阵来描述。例如，“select”元素有三个渲染对象：一个显示区域，一个下拉列表，一个按钮。此外当文本因为宽度不足被分为多行时，新行将作为额外的渲染程序添加。

另一个多个渲染对象的例子是损坏的HTML。根据CSS规范，一个inline元素要么只包含block元素，要么只包含inline对象，否则，匿名的block渲染对象会被创建来包裹inline对象。

某些渲染对象对应DOM节点，但不再树的相同位置上。float和absolute元素脱离文本流（out of flow），放在树的一个不同位置，并映射到真实的渲染对象。一个展位渲染对象是他们应该放的位置
(... ... and mapped to the real frame. A placeholder frame is where they should have been.)
![[Pasted image 20240602001941.png]]
###  构建树的流程
In Firefox, the presentation is registered as a listener for DOM updates. The presentation delegates frame creation to the `FrameConstructor` and the constructor resolves style (see [style computation](https://web.dev/articles/howbrowserswork#style_computation)) and creates a frame.

在WebKit中，解析和创建渲染对象的过程叫做attachment。每个DOM节点都有一个attach方法。Attachment时同步的，把一个节点插入DOM树会调用新节点的attach方法。

处理html和body标签会创建渲染树树根。跟渲染对象，对应于css标准所说的containing block：最顶层的包含所有block的block。它的尺寸就是视口（浏览器显示区域）的尺寸。火狐叫它ViewPortFrame，WebKit叫它RenderView。这是document对象所指向的渲染对象，该树其余部分以DOM节点插入的形式构造而成。
## 样式计算
构建渲染树需要计算每个渲染对象的视觉属性，这是通过计算每个元素的样式属性来完成的。
样式包括不同来源的样式表、内嵌样式元素和HTML中的视觉属性，后者将转换为匹配的CSS样式属性。
样式表的来源则有浏览器的默认样式表、网页作者提供的样式表以及由浏览器用户提供的用户样式表（浏览器允许您定义喜欢的样式，例如，在 Firefox 中，可通过将样式表放入“Firefox Profile”文件夹来完成此操作）

样式计算带来了一些难题：
1. 样式数据是一个非常大的结构，存储了无数的样式属性，这可能会导致内存问题。
2. 如果不进行优化，为每个元素查找匹配的规则可能会导致性能问题。要找出匹配元素，遍历整个规则列表是一项艰巨的任务。选择器可能具有复杂的结构，可能会导致匹配过程从看似有希望的路径开始，但事实证明这一路径是无效的，必须尝试另一条路径。
	例如，这个复合选择器：
```
div div div div{...}
```
这些规则适用于3个嵌套`div`的子代`<div>`；在查找过程中，可能发现很多层数不够的潜在路径。
3. 应用这些规则涉及到相当复杂的级联规则（用于定义规则的层次）。
## 共享样式数据
WebKit节点会引用样式对象(RenderStyle)。在某些情况下，这些对象可以由节点共享，这些节点得是同级节点，并且：
1. 这些元素必须处于相同的鼠标状态（例如，不能一个是 :hover 状态，而另一些不是）
2. 每个元素都不能有 ID
3. 标签名称一致
4. 类属性一致
5. 映射的属性集必须完全相同
6. link states必须一致
7. 焦点状态必须一致
8. 两个元素都不应受到属性选择器的影响，这里的“受影响的”是指，在选择器中的任何位置都使用属性选择器的任何选择器匹配
9. 元素中不得包含内嵌样式属性
10. 不能使用任何同级选择器。当遇到任何同级选择器时，WebCore 只会抛出一个全局开关，并停用整个文档的样式共享（如果存在）。这包括 + 选择器以及 :first-child 和 :last-child 等选择器。
## FireFox规则树
FireFox还有两个额外的树来简化计算：规则树和样式上下文树。WebKit 也有样式对象，但它们并不存储在样式上下文树这样的树中，只有 DOM 节点指向其相关样式。
![[Pasted image 20240602011154.png]]
样式上下文包含结束值。这些值的计算方法如下：按正确顺序应用所有匹配规则，并将其从逻辑值转换为具体值。例如，如果逻辑值是屏幕空间的百分比，则会计算此值并将其转换为绝对单位。 规则树的点子真的很巧妙。它支持在节点之间共享这些值，以避免重复计算。这也可以节省空间。

所有匹配的规则都存储在树中。路径中底层节点的优先级较高。规则树包含找到的所有规则匹配的路径。规则的存储是延迟进行的。树状结构不会在开始时就每个节点进行计算，但每当需要计算节点样式时，系统都会将计算的路径添加到树中。

这个想法相当于将树状路径视为词典中的单词。假设我们已经计算了此规则树：
![[Pasted image 20240602013717.png]]
假设我们需要匹配内容树中另一个元素的规则，并发现匹配的规则（按正确顺序）为 B-E-I。由于我们已经计算了路径 A-B-E-I-L，因此在树中已有此路径。我们现在可执行的操作会减少。
我们来看看这棵树是如何使我们的工作成果的。
## 结构体划分
样式上下文被划分为很多结构体，这些结构体包含一个特定类别的样式信息，像是border或者color。一个结构的所有属性要么是继承的，要么是非继承的。继承属性除非被元素定义，否则从父元素继承。非继承属性（或称为“重置”属性）如果没有定义，则使用默认值。

该树通过缓存所有结构体（包含计算出的结束值）来帮助我们。具体思路是，如果底层节点没有提供结构体的定义，则可以使用上层节点中的缓存结构体。
## 使用规则树计算样式上下文

在计算特定元素的样式上下文时，我们首先计算规则树中的路径或使用现有路径。然后，我们开始在路径中应用规则，以填充新样式上下文中的结构体。我们从路径的底层节点（优先级最高的节点（通常是最具体的选择器））开始，并向上遍历树，直到结构体填满。 如果该规则节点中没有结构体规范，我们可以进行大幅度优化 - 我们沿树状结构向上爬，直到找到一个完全指定该结构的节点并指向它，这是最好的优化方法 - 整个结构体会共享。 这可以节省结束值的计算量和内存。

如果我们找到部分定义，就会向上遍历结构树，直到结构体被填充为止。
  
如果我们未找到结构体的任何定义，那么如果该结构体是“继承”类型，我们会在**上下文树**中指向父结构的结构。在本例中，我们还成功共享了结构体。如果是重置结构体，将使用默认值。

如果最具体的节点确实添加了值，那么我们需要进行一些额外的计算，才能将其转换为实际值。然后，我们会将结果缓存在树节点中，以供子节点使用。

如果某个元素有指向同一树节点的同级或同级元素，它们之间就可以共享**整个样式上下文**。

让我们看一个示例： 假设我们有如下的 HTML 代码：

```
<html> 
	<body>    
		<div class="err" id="div1">      
			<p> 
				this is a 
				<span class="big">		
					big error 
				</span>        
				this is also a        
				<span class="big"> 
					very  big  error
				</span> 
				error      
			</p>    
		</div>    
		<div class="err" id="div2">
			another error
		</div>  
	</body>
</html>
```
以及以下规则：
```
div {margin: 5px; color:black}
.err {color:red}
.big {margin-top:3px}
div span {margin-bottom:4px}
#div1 {color:blue}
#div2 {color:green}
```
为了简化操作，假设我们只需填写两个结构体：color 结构和 margin 结构。color 结构体仅包含一个成员：color；margin 结构体包含四条边。

生成的规则树将如下所示（节点以节点名称（即它们所指向的规则的编号）标记）：
![[Pasted image 20240611194558.png]]
上下文树将如下所示（节点名称：它们指向的规则节点）：
![[Pasted image 20240611194623.png]]

假设我们解析 HTML 并得到第二个 `<div>` 标记，我们需要为此节点创建样式上下文，并填充其样式结构。

匹配规则后，我们发现 `<div>` 的匹配规则为 1、2 和 6。 这意味着树中已有一个路径可供我们的元素使用，我们只需为规则 6（规则树中的节点 F）再添加一个节点。

我们将创建一个样式上下文，并将其放入上下文树中。新的样式上下文将指向规则树中的节点 F。

现在需要填充样式结构体。首先，填充 margin 结构。 由于最后一个规则节点 (F) 没有添加到 margin 结构，我们可以向上访问树，直到找到在先前的节点插入操作中计算过的缓存结构，然后使用该结构。 我们将在节点 B 上找到它，节点是指定外边距规则的最高节点。

我们已经有了 color 结构的定义，因此无法使用缓存的结构。 由于 color 有一个属性，我们不需要上到树中填充其他属性。我们将计算结束值（将字符串转换为 RGB 等），并将经过计算的结构体缓存到此节点上。

第二个 `<span>` 元素处理起来更加轻松。我们匹配规则并得出结论，它指向规则 G，就像之前的 span 一样。 由于我们有指向同一节点的同级，因此我们可以分享整个样式上下文，并且只需指向上一个 span 的上下文。

对于包含从父节点继承的规则的结构，缓存在上下文树上进行（颜色属性实际上是继承的，但 Firefox 将它视为重置，并将其缓存在规则树上）。

例如，如果我们在一个段落中添加了字体规则：
```
p {font-family: Verdana; font size: 10px; font-weight: bold}
```
然后，该段落元素（上下文树中 div 的子项）可能会共享与其父项相同的字体结构。如果未为该段落指定任何字体规则，就会出现这种错误。

在 WebKit 中，由于没有规则树，因此匹配的声明会被遍历四次。首先应用不重要的高优先级属性（由于其他属性依赖于这些属性，因此应该先应用的属性，例如 display），然后是高优先级重要规则，然后是普通优先级非重要规则，最后是普通优先级重要规则。 这意味着多次出现的属性会根据正确的级联顺序进行解析。最后一方获胜。

总而言之：共享样式对象（整个对象或对象中的部分结构体）可以解决问题 1 和 3。Firefox 规则树还有助于以正确的顺序应用属性。
## 处理规则以实现轻松匹配
样式规则有多个来源：
1. CSS 规则（在外部样式表或样式元素中）。 `css p {color: blue}`
2. 内嵌样式属性，例如 `html <p style="color: blue" />`
3. HTML 视觉属性（映射到相关样式规则） `html <p bgcolor="blue" />` 后两项很容易与元素匹配，因为元素拥有样式属性，而 HTML 属性可以使用元素作为键进行映射。

如之前的问题 2 中所述，CSS 规则匹配可能会比较棘手。 为了解决这一难题，系统会操纵规则以简化访问。

样式表解析完毕后，系统会根据选择器将规则添加到某个哈希映射中。 其中包括按 ID、类名称、标记名称划分的映射，以及不属于上述类别的任何内容的通用映射。如果选择器是 ID，规则将添加到 ID 映射；如果选择器是类，则将添加到类映射中，以此类推。

这种处理可以大大简化规则匹配。无需查看每个声明：我们可以从映射中提取元素的相关规则。这种优化可以排除 95% 以上的规则，因此在匹配过程中甚至不需要考虑这些规则(4.1)。

我们以下面的样式规则为例：

```
p.error {color: red}
#messageDiv {height: 50px}
div {margin: 5px}
```

第一条规则将插入类映射中。将第二个插入 ID 映射，将第三个放入标签映射。

对于以下 HTML 片段：

```
<p class="error">an error occurred</p>
<div id=" messageDiv">this is a message</div>
```

我们首先会尝试为 p 元素寻找规则。类映射将包含一个“error”键，在下面可以找到“p.error”的规则。div 元素在 ID 映射（键为 ID）和标记映射中具有相关规则。 因此，剩下的工作就是找出由键提取的哪些规则真正匹配。

例如，如果 div 的规则是：

```
table div {margin: 5px}
```

它仍会从标记映射中提取，因为键是最右边的选择器，但是它与没有表祖先的 div 元素不匹配。

WebKit 和 Firefox 均会执行这一操作。

## 样式表级联顺序
样式对象具有与每个视觉属性相对应的属性（所有 CSS 属性，但更通用）。 如果该属性未由任何匹配的规则定义，则某些属性可由父元素样式对象继承。其他属性具有默认值。

当有多个定义时，问题就开始了 - 这里使用级联顺序来解决问题。

一个样式属性的声明可能会出现在多个样式表中，也可能在一个样式表中出现多次。这意味着应用这些规则的顺序非常重要。这称为“级联”顺序。 根据 CSS2 规范，级联顺序为（从低到高）：

1. 浏览器声明
2. 用户常规声明
3. 作者常规声明
4. 作者重要声明
5. 用户重要声明

浏览器声明是最不重要的，仅当声明被标记为重要时，用户才会替换作者的声明。 具有相同顺序的声明将按特异性排序，然后按指定顺序排序。[](https://web.dev/articles/howbrowserswork?hl=zh-cn#specificity) HTML 可视化属性会转换为匹配的 CSS 声明。它们被视为低优先级的作者规则。
## 特异性
选择器的特异性由 [CSS2 规范](http://www.w3.org/TR/CSS2/cascade.html#specificity)定义，如下所示：

1. 如果来源声明是“style”属性而不是带有选择器的规则，则记为 1，否则记为 0 (= a)
2. 统计选择器中 ID 属性的数量 (= b)
3. 统计选择器中其他属性和伪类的数量 (= c)
4. 统计选择器中元素名称和伪元素的数量 (= d)

将四个数 a-b-c-d（在有大数进制的数值系统中）串联起来，即可确定特异性。

您使用的基数取决于某个类别中的最高计数。

例如，如果 a=14，您可以使用十六进制。在极少数情况下，当 a=17 时，您需要使用 17 位数字。 如果采用如下的选择器，可能会出现后一种情况： html body div div p...（选择器中有 17 个标记...不太可能）。

一些示例：

```
 *             {}  /* a=0 b=0 c=0 d=0 -> specificity = 0,0,0,0 */ 
 li            {}  /* a=0 b=0 c=0 d=1 -> specificity = 0,0,0,1 */ 
 li:first-line {}  /* a=0 b=0 c=0 d=2 -> specificity = 0,0,0,2 */ 
 ul li         {}  /* a=0 b=0 c=0 d=2 -> specificity = 0,0,0,2 */ 
 ul ol+li      {}  /* a=0 b=0 c=0 d=3 -> specificity = 0,0,0,3 */ 
 h1 + *[rel=up]{}  /* a=0 b=0 c=1 d=1 -> specificity = 0,0,1,1 */ 
 ul ol li.red  {}  /* a=0 b=0 c=1 d=3 -> specificity = 0,0,1,3 */ 
 li.red.level  {}  /* a=0 b=0 c=2 d=1 -> specificity = 0,0,2,1 */ 
 #x34y         {}  /* a=0 b=1 c=0 d=0 -> specificity = 0,1,0,0 */ 
 style=""          /* a=1 b=0 c=0 d=0 -> specificity = 1,0,0,0 */
```

## 规则排序

匹配规则后，它们会根据级联规则进行排序。WebKit 对小型列表使用冒泡排序，对大型列表使用合并排序。WebKit 通过替换规则的 `>` 运算符来实现排序：

```
static bool operator >(CSSRuleData& r1, CSSRuleData& r2){    
	int spec1 = r1.selector()->specificity();    
	int spec2 = r2.selector()->specificity();    
	return (spec1 == spec2) : r1.position() > r2.position() : spec1 > spec2;
}
```

## 逐步处理

WebKit 使用一个标记来标记是否所有顶级样式表（包括 @imports）均已加载完毕。如果在添加样式时样式未完全加载，则会使用占位符并在文档中对其进行标记，并在样式表加载完毕后重新计算占位符。
# 布局
渲染对象在被创建并加入树中时，不具有位置和大小，计算这些值的过程叫布局或者回流（reflow）。

HTML 使用基于流的布局模型，这意味着大多数情况下，只需一次遍历即可计算出几何图形。“流中”后出现的元素通常不会影响早出现在“流中”的元素的几何形状，因此布局可以按从左到右、从上到下的顺序遍历文档。也有一些例外情况：例如，HTML tables可能需要多次遍历。
  
坐标系是相对于根框架而言的。使用上坐标和左坐标。

布局是一个递归的过程，它从根渲染对象开始，继续以递归方式遍历部分或者全部的渲染对象结构，为每个需要几何信息的元素进行计算。

根渲染程序的位置为 (0,0)，其尺寸为视口（即浏览器窗口的可见部分）。

所有渲染对象都有“布局”或“回流”方法，每个渲染对象都会调用需要布局的子项的“布局”方法。

## Dirty位系统
为避免对所有细微更改都进行完整的布局，浏览器使用“脏位”系统。如果渲染对象发生更改或添加，则会将其自身及其子项标记为“脏”：需要布局。

有两个标志：“dirty”和“children are dirty”，表示尽管渲染对象本身没有问题，但它至少有一个子项需要布局。

## 全局布局和增量布局
可以在整个渲染树上触发布局，这就是“全局”布局。 这种情况可能是由以下原因导致的：
1. 影响所有渲染程序的全局样式更改，例如字体大小更改。
2. 屏幕大小调整。

布局可以是增量式，仅布局 dirty 渲染对象（这可能会造成一些损坏，导致需要额外的布局）。

当渲染对象处于脏状态时，会（异步）触发增量布局。例如，当额外内容来自网络并添加到 DOM 树之后，新的渲染对象才附加到渲染树。

![[Pasted image 20240611205502.png]]
<center>增量布局 - 仅布局脏渲染器及其子项</center>
## 异步布局和同步布局
增量布局是异步执行的。Firefox 将增量布局的“reflow commands”加入队列，而调度器会触发这些命令的批量执行。 WebKit 还有一个用于执行增量布局的计时器：对布局树进行遍历，并对“脏”渲染程序进行布局。

请求样式信息（例如“offsetHeight”）的脚本可以同步触发增量布局。（布局抖动）

全局布局通常会被同步触发。

有时，由于某些属性（例如滚动位置）发生了变化，布局会在初始布局之后作为回调触发。
## 优化
  
当布局是由“resize”或渲染对象位置（而非大小）的变化触发时，系统会从缓存中获取渲染大小，而不会重新计算...

在某些情况下，只有子树会被修改，布局不是从根节点开始的。如果更改是局部的，并且不影响其周围环境，就会发生这种情况，例如插入文本字段的文本（否则每次按键都会触发从根开始的布局）。
## 布局流程
布局通常具有以下模式：
1. 父级渲染对象会自行确定宽度。
2. 父级会处理子级，并执行以下操作：
    1. 放置子渲染对象（设置其 x 和 y）。
    2. 调用子对象布局方法，如果有必要的话 — 譬如，它们是脏的，或者我们处于全局布局，或出于某种其他原因 — 这会计算子项的高度。
3. 父级使用子级的累计高度以及外边距和内边距的高度来设置自己的高度，此值将由父级渲染器的父级使用。
4. 将其脏位设置为 false。

（父对象布局方法调用->父级确定宽度->放置子对象位置（这里应当是逐个放置的，调用完一个子对象的方法，它会计算自己的宽度，父元素再调用下一个子元素的方法），调用子对象布局方法（流程类似）-> 获取子对象高度->使用子对象累计高度计算自身高度）

Firefox 使用“state”对象 (nsHTMLReflowState) 作为布局方法的参数（称为“reflow”）。状态包括父项宽度等。

Firefox 布局方法返回“metrics”对象(nsHTMLReflowMetrics)。它将包含计算出的渲染程序高度。
## 宽度计算
渲染对象的宽度根据容器块的宽度、渲染对象的样式“width”属性、外边距和边框计算得出。

例如，以下 div 的宽度：
```
<div style="width: 30%"/>
```
WebKit 的计算公式如下（RenderBox 类中的 calcWidth 方法）：
- 容器宽度是容器 availableWidth 和 0 中的较大值。在本例中， availableWidth 是 contentWidth，计算公式如下：
```
clientWidth() - paddingLeft() - paddingRight()
// 内部宽度减去padding
```
clientWidth 和 clientHeight 表示一个对象的内部的宽高（不包括边框和滚动条）。
- 元素的宽度是“width”样式属性。系统会通过计算容器宽度的百分比来计算出绝对值。
- 现在添加了水平边框和内边距。

到目前为止，这是“首选(preferred)宽度”的计算结果。 现在将计算最小和最大宽度。

如果首选宽度大于最大宽度，则使用最大宽度。 如果宽度小于最小宽度（最小的不可破坏单位），则使用最小宽度。

在需要布局但宽度不变的情况下，系统会缓存这些值。
## 换行
当处于布局中间的渲染对象确定需要换行时，它会停止，并告知布局的父项需要被换行。 父组件会创建额外的渲染对象，并对其调用布局方法。
# 绘制
在绘制阶段，系统会遍历渲染树，并调用渲染程序的“paint()”方法，以在屏幕上显示内容。绘制会使用界面基础架构组件。
## 全局和增量
与布局一样，绘制也可以是全局（绘制整个树）或增量的。在增量绘制中，部分渲染对象发生了更改，但不会影响整个树。更改后的渲染对象会使其在屏幕上的矩形失效。这会导致操作系统将其视为“脏区域”并生成“绘制”事件。 操作系统很巧妙地将多个区域合并成了一个区域。在 Chrome 中，情况要更复杂一些，因为渲染的进程与主进程不同。Chrome 会在某种程度上模拟操作系统的行为。 展示层会监听这些事件，并将消息委托给渲染根节点。系统会遍历呈现树，直到找到相关的渲染对象。它会自行重绘（通常也包括其子代）。

## 绘制顺序
[CSS2（rather old article。。。）定义了绘制过程的顺序](http://www.w3.org/TR/CSS21/zindex.html)。 这实际上是元素在[堆叠上下文](https://web.dev/articles/howbrowserswork?hl=zh-cn#the_painting_order)中的堆叠顺序。这些堆栈会从后往前绘制，因此这个顺序会影响绘制。块渲染程序的堆叠顺序如下：
1. 背景颜色
2. 背景图片
3. 边框
4. 子节点
5. Outline
## Firefox 显示列表
Firefox 遍历渲染树，为绘制的矩形构建一个显示列表。 它包含与矩形相关的渲染对象，按正确的绘制顺序（渲染对象的背景，然后是边框等）排列。

这样一来，在重新绘制时，只需遍历一次树，而不是多次遍历 - 绘制所有背景，然后绘制所有图片，再绘制所有边框等等。

Firefox对处理过程进行了优化，它不添加隐藏的元素，比如完全位于其他不透明元素下方的元素。

 WebKit矩形存储

在重新绘制之前，WebKit会将旧的矩形保存为位图。然后只绘制新旧矩形之间的增量。
## 动态变化
在发生变化时，浏览器会尽可能地做出尽可能少的操作。因此，元素的颜色改变后，只会对该元素进行重绘。 更改元素位置将导致元素及其子元素（可能还有同级元素）的布局和重绘。 添加 DOM 节点将导致对该节点进行布局和重新绘制。重大更改（例如增大“html”元素的字体大小）会导致缓存失效、对整个树进行重新布局和重新绘制。
## 渲染引擎的线程
渲染引擎采用单线程。几乎所有操作（网络操作除外）都是在单个线程中进行的。在 Firefox 和 Safari 中，该线程是浏览器的主线程。在 Chrome 中，它是标签页进程主线程。
网络操作可由多个并行线程执行。并行连接的数量是有限的（通常为 2 至 6 个）。（似乎可以测试一下）
## 事件循环
浏览器的主线程是事件循环。它是一个无限循环，可以使进程保持活跃状态。它会等待事件（如布局和绘制事件）并处理这些事件。以下是用于主事件循环的 Firefox 代码：

```
while (!mExiting)    NS_ProcessNextEvent(thread);
```
