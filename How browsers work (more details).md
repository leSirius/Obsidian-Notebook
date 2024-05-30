# 高层架构(High-level infrastructure)
1. 用户界面：包括地址栏、前进/后退按钮、书签菜单等。显示在请求的网页之外。
2. 浏览器引擎：marshals actions between the UI and the rendering engine.
3. 渲染引擎：负责显示请求的内容。如果请求的是HTML，会解析HTML和CSS，并把解析的内容显示在屏幕上。
4. 网络：负责http请求等网络调用。
5. 用户界面后端：用于绘制基本 widget，如组合框和窗口。此后端公开了与平台无关的通用接口。在底层使用操作系统界面方法。
6. Javascript解释器：解析和执行JS代码
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
解析可以分为两个子过程：词法分析和语法分析(lexical analysis and syntax analysis)。

词法分析是将输入内容分解成多个词元(tokens)的过程。词元是语言词汇，即有效组成要素的集合（collection of valid building blocks)。在人类语言中，它由一门语言的字典中出现的所有单词组成。
Lexical analysis is the process of breaking the input into tokens. Tokens are the language vocabulary: the collection of valid building blocks. In human language it will consist of all the words that appear in the dictionary for that language.
语法分析是指语法规则的应用。

解析通常会将工作划分到两个组件之间：词法分析器（又称为标记生成器），它负责将输入内容分解为有效标记(valid tokens)，而解析器则负责根据语言语法规则分析文档结构进而构建解析树。
词法分析器知道如何删除不相关的字符，例如空格和换行符。（在`2+3-1`的例子中，是分解并获取数字2，3，1和操作符+，-的过程。）
解析器通常会向词法分析器要一个新标记(token)，并尝试将该token与某条语法规则匹配。如果规则匹配，则会将token对应的节点加到解析树中，然后解析器会请求另一个token。这个过程是迭代的。
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
# HTML解析器
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
### 解析算法
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
文档状态随后设置为”完成“（complete），并且一个”（已）读取“（load）事件会触发。
















