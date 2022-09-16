---
title: Feakin 是如何设计与构建的？
date: 2022-09-16 22:50:00
tags:
---

高中，读过几本 3D 图形编程相关的书。怎么说呢，自那以后，图形学相关的东西，都不在我的兴趣范围里了。直到最近，我重新燃起了一点兴趣：

* 架构治理工具 [ArchGuard](https://github.com/archguard/archguard) 依赖于「[图即代码](https://www.phodal.com/blog/diagram-as-code/)」，用于生成架构图，以更好的进行架构治理。
* 年初，开源的知识管理工具 [Quake](https://github.com/phodal/quake/) 中，需要支持「**概念构建系统**」这样一个理念。
* 需要管理多种不同的图形格式。

欢迎尝试在线 Demo：<https://online.feakin.com/> ， GitHub：<https://github.com/feakin/feakin/>，当然了 Bug 超级多。

## 引子：开源绘图工具实现的浅析

在设计 Feakin 的时候，参考了一些几个常用的图形工具。分析了它们的大致实现，以及部分的源码：

**Graphviz**

AT&A 实验室的作品，作为最古老的图形即代码的工具，它还提供了一个**图形描述语言**：Dot，可以直接将代码转换为图形。它的生态体系足够的完善，所以你在哪都能看到它的影子。

**Mermaid**

同样也是一个图形即代码的工具，使用的是纯 JavaScript 实现，从语法解析到图形渲染。Mermaid 使用 Jison 作为解析器，然后将其转换为不同的图模型，如流、时序等，再使用 graphlib、dargre 进行布局，最后使用 dagre-d3、d3 进行渲染。因此，在 Mermaid 里有三个核心要素：语法解析、图形布局、图形渲染。而，Mermaid 不存在一个图形模型，也变成了一个神奇的存在。

**Cytoscape**

第一次看到这个图形引擎的时候，是看到 ArchGuard 前人留下的一个功能：**布局算法切换**。所以，在源码实现上，Cytoscape 提供了这种算法上的扩展性，具体可以看[官方网站](https://js.cytoscape.org/)。布局上的抽象，提供了更好的可扩展性 —— 我们就可以参考它的实现了。在它的图形模型里，Node（节点） 和 Edge（边） 从形式上都算是 Element，然后在渲染时根据图形类型展开。于是在渲染时，直接采用 HTML5 里的 Canvas 进行绘制即可。

**Excalidraw**

对我来说，其最有意思的是引入了**射影几何**，来进行节点变化时的，自动 Edge 跟踪；即当 A 从 B 的左边移动到右边时，对应的线自动连接到 B 右边的边上。当然了，其中的各种神奇算法，我也没看懂。对于其他人，可能就是使用 roughjs 来生成手绘风格的图。当然了， 就目前的代码实现来说，roughjs 在 renderElement 里过度的耦合，图形模型也耦合在其中。

**MaxGraph**

MaxGraph 是 Draw.io 底层的 mxGraph 的 TypeScript 实现，最开始研究时，是为了导入 Draw.io 生成的图。从模型上来说，MaxGraph 应该是几个工具里做得最好的，包含一系列的可参考的 Shape、Edge 等等。其次，也提供了 AbstractCanvas2D 这样的实现，虽然它没有实现真正的 HTML5 Canvas2D，但是抽象接口已经非常像了，诸如 `.moveTo`、`.lineTo` 等。可能它就提供 SVG 和 XML，前者用于网页渲染，后者用于导出。

所以，从上述的几个工具里，我们就能得到一个绘图工具底层的基本要素：

* 图形模型。即对图形建模，理清 Diagram/Graph、Node、Edge、Shape、Element 之间的关系，并包含基本的图形表示关系。
* 图形绘制。即定义如何对图形进行绘制/渲染，如采用 SVG、Canvas 等不同的形式。

为了丰富这些功能：

* 布局算法。提供自动化的布局方式，如 Cytoscape 这一类自动计算的方式。
* 语法解析。诸如于为了支持图即代码（即 DSL）的形式来提供快捷的绘制方式。
* 自动连线。即如 Excalidraw、Draw.io 中提供的功能，两者实现的方式完全不一样。
* 图形风格。诸如于 Excalidraw 提供的手绘图形的功能。
* 图形库。这也是 Drawio 最受欢迎的地方，也是 Excalidraw 一个很有意思的功能。
* 等等

结合这些功能，我们就可以造出一些有意思的东西，比如 Feakin 中的二阶段渲染。

## Step 1：实现第一个概念证明

为了 Feakin 能进行下去，我们所要做的就是快速实现一个 PoC（概念证明）。在这个 PoC 里，主要实现如下的功能：

*  DSL （领域特定语言）解析。
* 图形模型生成。
* 图形绘制。

如下图所示：

 ![](/processor/blog/images?file_name=2022-08-27T07:57:34.024Z.png)

这样一来，我们就有一个「It works」了。

### 从图形引擎的误区中出来

在实现第一个 PoC 的时候，遇到的第一个困难是技术选型，到底是：SVG 还是 Canvas？SVG 可以方便于我们进行 TDD（测试驱动开发），只要所有的测试是通过的，理论上结果就是过的。但是，如我们所看到的那样，SVG 容易遇到性能瓶颈。Canvas 提供自由的绘制 API，测试时依赖于快照测试（snapshot），不易于编写测试。所以，结论就是：我们都要了吧。只需要像 MaxGraph 提供一个抽象图形接口，我们就能实现对于两种模式的支持。

随后，发现这样是不合理的，只在 PoC 阶段，并且没有经验的情况下，做一个 AbstractCanvas 还是存在很高的成本。于是乎，需要寻找一个合理的绘制引擎，诸如于 Raphaël、Fabric、Konva 等。最后，选择了 Konva，因为它支持了 React 框架。正所谓，**工作用 Angular 心不累，业余用 React 放我自我**。

### 原型：语法解析-图形模型-图形绘制

在构建了基本的图形领域的相关知识之后，要构建出一个绘图工具并不困难。

* 参考（复制） Mermaid 的语法解析。将通过 parser 解析类似于 Graphviz、Mermaid 设计的语法，将其转换为图形模型。
* 引入  Dagre.js 作为图形布局引擎。通过 Dagre.js 来计算布局，返回我们所需要的图形模型。
* 使用 React Konva 进行渲染。将图形模型匹配到 Konva 中的图形，如 RectangleShap 对应于 `<Rect>` 组件、Edge 对应于 `<Line>`、 `<Arrow>`等。

过程中，遇到的一个比较坑的点是：Lerna + Nx.js 管理 monorepo。React + Craco 的组合、风格各异的代码库，带来了持续失败的 CI，还好 GitHub Action 不会统计失败率。持续集成不来点失败，怎么能发挥它的用处呢。

## Step 2：对模型进行反复重构（持续）

在 Poc 里，我们需要遇到不同的模型转换：

* 解析器获得的模型。包含节点的信息，以及节点的关系（诸如于 A 到 B、A 依赖于 B 等）。
* 布局引擎生成的模型。通常来说，只是补充一下模型里的层次关系（children/parent）、坐标信息（x、y）、几何信息（width、height）等。
* 图形绘制引擎的模型。我们需要将上述的信息，再次转换到 Konva 的模型中。而其中会存在一些差距，比如 Konva 使用 Polygon（多边型）来表示Triangle（三角型）、Diamond（菱形）等。

所以，如何设计一个有用的模型，成为了个有意思的问题。

### GIM：图中间模型

在那一篇《[图的抽象：概念与模型的构建](https://www.phodal.com/blog/abstract-graph/)》中，我们介绍了从认知语义学的角度，如何仅凭基本的概念，设计出可用的模型？不过，这样的模型是未经验证的。那么，什么样的模型是经常验证的呢？自然是开源社区中，已经充分使用的代码模型。虽然说，各个模型受限于自己的场景，与其他软件的模型存在一定的差距。但是呢，在基本的核心概念**图的表示**上，它们是大差不差的。于是乎，我们有了一个 GIM（Graph Intermedia Model），图中间模型。

这个图模型的来源是源自其他图形工具成熟的模型，如下图所示：

 ![](/processor/blog/images?file_name=2022-08-27T03:37:39.376Z.png)

所以，在持续的建模、提炼之后，我们可以轻松地进行我们的图模型转换。在有了 TDD 的加持之后，这个过程就更加地简单了。

在模型这一点上，Feakin 的设计初衷与 ArchGuard 底层的 [Chapi](https://github.com/modernizing/chapi) （<https://github.com/modernizing/chapi>） 语言模型的想法是一致的。而这种所谓的通用模型会遇到的问题是，需要抛弃一些细节，诸如于只实现 80% 的核心功能。

### 图的模型

对于一个图（Graph）来说，它的模型也就变得相当的简单：

```typescript
export interface Graph {
  nodes: Node[];
  edges: Edge[];
  props?: GraphProperty;
  subgraphs?: Graph[];
}
```

围绕于这四个核心元素再往下展开：

* 节点（Node）。主要包含坐标信息，形态信息等，可以用于构建出不同的 shape。
* 边（Edge）。主要包含点（Point），可以用于构建普通的直线、贝塞尔曲线（Bézier）曲线等，还有
* 属性（Props）。这里只属于命名为 props 是为了对齐 🐶 ，对应于图形属性，诸如于 fillColor、color、strokeColor 等。
* 子图（Graph\[\]）。一个抽象的概念，在不同的图示中有不同的形式，如 Group、子集等。

而如上所述 Shape 和 Edge 就是两个大家庭，包含了一系列的子类，诸如于 Shape 包含了 PolygonShape（又包含了 TriangleShape、DiamondShape、RectangleShape 、HexagonShape）。

### 状态与属性（TBD）

对于属性来主，尚未进一步展开，但是初步分为：FillState、FontState、StrokeState、ImageState 等。

如果你也感兴趣的话，欢迎一起来设计。

## Step 3：核心特性基础：二阶段绘图

在反复的设计了各种 Importer/Exporter 之后，并持续不断的进行模型重构之后，就构成了的核心特性的基础：二阶段绘图。简单来说，就是把绘图分为了两阶段：

* 通过 DSL 生成图或者导入生成图。
* 使用图形工具对生成的图进行编辑。

以在不同的工具之间转换，并实现图的互转。

### 二阶段绘图示例

在这里就可以尝试使用：<https://online.feakin.com/> ，虽然还只是一个早期的版本，仍旧还有一系列的 bug，但是还可以尝试的。

 ![](/processor/blog/images?file_name=2022-08-27T08:04:50.237Z.png)

如上图所示，我们可以


1. 通过 File → Import 导入 Draw.io 或者 Excalidraw，又或者是通过 Graphviz 的 Dot 语法编写。
2. 通过 Export 导出到 Draw.io 或者是 Excalidraw

图中，左边的编辑器是使用 Monaco Editor，配合了简单的 Dot 语法支持；右边则是一个早期版本的 Feakin Render，只能简单地渲染一下，看看效果。当前，仅支持简单的拖拉拽，还容易出错。

### 决策：过程一致或者结果一致？

在这个过程中，还有一系列有意思的东西，比如 Shape 在不同的图形工具是不一样的。先让我们看个代码示例：

```javascript
digraph G {
  a [shape="triangle"];
  b [shape="diamond"];
  a -> b;
}
```

也是截图中的代码简化，节点 a 的 shape 是一个 Triangle（三角形），然而：

* 在 Excalidraw 中不存在三角形，需要自己用 Line 绘制一个。
* 在 Draw.io 中默认的 Triangle 和正确的三角形不一样，正确的类似应该是`mxgraph.basic.acute_triangle` 。

于是乎，为了结果上的一致，我们需要在对应的 ExcalidrawExporter、DrawioExporter 进行对应的 Shape 的处理和 Mapping。

## Step 4：从 MVP 到真实世界

在这个 MVP（最小可行性产品）里，我们所构建的只是一个可以工作的原型，依旧有一系列的工作要完成。诸如于：

**更丰富的图形**

当前只支持基础的图形，在未来，支持其它工具的图形库 —— 有了 GIM，我们就不需要自己设计了。

**图形的属性**

从颜色到边框，一个功能也没有。难点主要在于，如何进行对应的属性抽象。在 MaxGraph 是一个胖模型，这种模型不利于维护，会带来额外的知识负载，它还是按字母顺序排序的，头疼。

**生态兼容性**

诸如于，虽然我们能成功导出 Excalidraw 的图形，也可以实现模型之间的绑定。但是，在一些属性上还是有区别的。

当然，这也是其中非常有意思的地方 —— 原来你只需要写一份未经验证的代码，现在你要看 N 份。

## 下一步：远程多人协作

既然，我们将代码作为第一等公民，那么实现代码的远程协作，也就是这个工具非常有意思的地方。一提到代码的多人协作，我就想起了我熟知的 Intellij IDEA。作为一个熟悉 Intellij IDEA Community 源码的人，我就联想到了 Fleet 架构里的 [Rope Architecture Model](https://blog.jetbrains.com/zh-hans/fleet/2022/02/fleet-below-deck-part-ii-breaking-down-the-editor/) 与 [State Management](https://blog.jetbrains.com/zh-hans/fleet/2022/06/fleet-below-deck-part-iii-state-management/) 两篇相关文章。大体是关于如何使用 Rope 模型来管理 AST（抽象语法树），以及如何管理多人协作的状态问题。

除此，之前读过的 Xi Editor，也有关于 Rope 模型也有很好的介绍：<https://xi-editor.io/docs.html>。它提供了一个很好的 Rust 实现，这样一来，我们就可以使用 Rust 来开发 Feakin 的协作部分。

如果你也有兴趣，欢迎一起来用爱发电。如此一来，也不枉我花三个小时写的这篇文章。


