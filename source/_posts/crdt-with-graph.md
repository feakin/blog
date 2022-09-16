---
title: CRDT 在 Feakin 中的应用
date: 2022-09-16 22:50:20
tags:
---



与常规的在线可视化协作相比较，对于 Feakin 这一类的图即代码的绘图工具来说，其在线协作可以直接简化为三个元素：

* 在线：通讯协议与数据格式
* 协作：中心化还是去中心化？
* 编辑：多端 CRDT与编辑器集成

从技术的层面来说，这些问题并不复杂，只是熟悉概念需要一个过程。但是呢，「中心化还是去中心化」这个问题非常有意思，毕竟从 Web 3.0 的韭菜热度来看，未来人们更想到**去中心化**的世界。

PS：在线绘图 Demo：<https://online.feakin.com/> ，可以通过复制 Room ID 给其他人来实现协作。服务器部署在 Heroku 上，代码见：<https://github.com/feakin/feakin/> 。

## 在线：通讯协议

在线协作，意味着实时性，依赖于构建持续的长连接。关于如何构建这种连接的过程与方式，在不同的场景下，它被总结为不同的通讯协议，诸如于在 IoT 领域流行的 MQTT、CoAP、LWM2M 等。

### 通讯协议：WebSocket vs HTTP

回到，如何保持在线协议这个问题，在浏览器端，基本上不就是无脑 WebSocket 嘛，学习门槛最低。

不过呢，在探索的过程中，有一个社区发起的同步 HTTP 协议 [Braid Protocol](https://braid.org/)，引用了我的兴趣。它自定义了一个 209  [Subscriptions](https://github.com/braid-org/braid-spec/issues/106) 状态码，使用 HTTP 长连接来接受更新，使用 HTTP Patch 来更新 patch。从语义上来说，Patch 操作确实很适合于这个场景。在有了这一类的状态同步协议，那么使用 HTTP 也可以实现 P2P 网络。不过，其主要用途还是用于使 Web 资源能够在多个客户端、服务器和代理之间**自动同步**，并支持多个编写者在任意网络延迟和分区下进行任意同时编辑，同时使用 OT、CRDT 或其他算法保证一致性。

简单来说，协议的初衷就是**为协作而设计的**。作为一个早期阶段的协议，自然是没有多大胆量采用。不过，试了试官方的 Demo，在 Firefox 和 Chrome 浏览器上都可以工作，Chrome 浏览器还显示了自定义的 Header。

### 数据格式：JSON vs Binary

随后，在协作的过程中，会产生大量的数据，我们也需要定义好数据。从当前的研究来看，主流采用的都是二进制的形式，从性能上来更优。

不过，在当前的 Feakin 版本里，为了调试方便（主要是没有 E2E 测试），采用的就是 JSON 形式的数据格式，未来已经切到二进制文档。最后在 Feakin 里，我们使用了 Actix + WebSocket 来实现这个功能。从实现上还是有点 trick，官方的 demo 实现的时候有点奇怪。基本的数据结构如下：

```rust
#[derive(Debug, Clone, Serialize, Deserialize, PartialEq, Eq)]
#[serde(tag = "type", content = "value")]
pub enum ActionType {
  CreateRoom(CreateRoom),
  JoinRoom(JoinRoom),
  LeaveRoom(LeaveRoom),

  // patches
  UpdateByVersion(UpdateByVersion),
  OpsByPatches(OpsByPatches),
}
```

感谢 Rust 神奇的 Enum 类型，简化了我的复制和粘贴。不过，从命名的角度来说，依旧有很大的改善空间。

## 协作算法：中心化还是去中心化？

接着，让我们再回到核心问题上。

多人协作本质就是一个分布式系统中的一致性问题。在我们协作的过程中，自然会出现各种的冲突：当 A 修改了文档，这个变更会在本地保存，并异步复制给其他正在编辑此文档的用户。而当 A 在修改标题时，B 也在修改时，此时就会造成冲突。应对于冲突的产生，有不同的方案，诸如于一种简单的方案是在此时将 title 锁定住，不让其他人修改；还可以是 Last-write-wins 策略，即谁最后写入就用谁的。

大量的分布式系统相关的问题，都可以在数据库相关领域的书籍中看到，诸如于《数据密集型应用系统设计》、《数据库系统内幕》。对于我来说，有一些概念，我也是从中现学现用的。不过，如果你要是有兴趣，也可以从开发 Feakin 的过程中学到更多。

###  OT 算法 vs CRDT

在协作上，当前基本上主流的就是两个流派：

* 中心化。客户端需要一直保持与服务器的连接，一旦离线了，那么就无法协作。代表是 OT ，核心部分是：管理转换过程的通用控制算法、 执行操作的转换函数，为此这里的操作需要细化到**原子操作**。
* 去中心化。客户端允许短暂的离线，并在恢复后同步，还允许没有服务端的存在。代表是 CRDT ，一种简化分布式数据存储系统和多用户应用程序的数据结构。主要可用于跨设备同步（如 Apple Notes）、分布式数据库、协作软件、大规模数据存储和处理系统等。

这也是从顶层设计上， OT 与 CRDT 的巨大差异之处。另外一些差异在于 OT 更多的是针对于文本数据，而 CRDT 则可以针对于文本、任意 JSON 数据。这也就是为什么大量的分布式数据库，诸如于 Redis、Riak 会使用它的原因。

**OT**

从名称上来说，OT（Operational Transform，操作转换） 主要基于操作的，客户端生成操作，再将由服务端处理，服务端处理完，推给其他客户端。根据不同的场景，可以支持不同的操作，如 CKEditor 中的：Insert、Delete、Split、Merge 等。

服务端是整个 OT 的核心所在，而客户端在接收到更新的请求后，也需要具备和服务端相同的合并代码。既然如此，那么我们为什么不做成去中心化的呢？

**CRDT**

从名称上来说，(Conflict-Free Replicated data types**，**无冲突复制数据类型) 主要是基于状态的，CRDT 的思路是尽可能避免冲突，如此一来，我们就不需要解决冲突。在发生变更时，就生成 patch，发送到其他端，如服务器、客户端等。当我们使用 CRDT 进行文本协作时，每一个字符视为一个实体。

当我们在客户端编辑的时候，可以生成一个 patch，这个 patch 可以由其他端进行 merge，诸如于客户端：`self.inner.decode_and_add(&patches)`，又或者是客户端的：`doc.mergeBytes(bytes)`。


---

从结论说起， OT 设计起来很简单，实现想来复杂，但是在极端场景下很复杂。而 CRDT 则是在设计阶段很复杂，除非能设计一个理想的**数据模型**，否则在需求不断变化的情况下，模型就成了一个问题。诸如于，针对富文本的属性（attribute）这一点上，与纯文本存在巨大的差异。这也就是富文本与纯代码实现的差异，诸如于如 CKEditor 的作者在「[Lessons learned from creating a rich-text editor with real-time collaboration](https://ckeditor.com/blog/Lessons-learned-from-creating-a-rich-text-editor-with-real-time-collaboration/))」一文中所说，OT 具有更好的可扩展性 —— 因为不需要更多的预先设计，可以很方便地添加一些操作，诸如于重命名、插件等。

在此这里，我还没有仔细研究各类 CRDT 实现的差异，这些差异点的分析，留给未来写数据库的时候来实现。如果你对 CRDT 有兴趣，可以看这个视频：[CRDTs: The Hard Parts](https://www.youtube.com/watch?v=x7drE24geUw)。

### 歪个楼：回顾一下 Git 的基本概念

从设计理念上来说，Git  也是一款针对于分布式设计的 “数据库管理” 工具：结合 SHA-1 哈希值来进行对象库（object database）的管理，并通过 `refs`、`HEAD`、`index` 等几个要素来构建其底层世界。

OT/CRDT  等在实现上与 Git 极为相似，只是 OT/CRDT 更像是一种实时的 Git-Rebase，获得 patch，自动就 rebase。除此在使用上，我们也并不会像 CRDT 一样使用 Git —— 为了保存这种最终强一致性：变更一个字符，便同步一次；删除一个字符，又同步一次。如果我们在真实的项目中，写入一个字符更 commit 一次，push 一次，虽然基本上不会产生冲突，但是我们的流水线大概率  99% 的时间都是挂的。

而我们的 ”编辑器/IDE” 则会实现写入一个字符更 “提交/触发” 变更，以便提供更智能的编辑服务。

### 协作技术选型：Rust 与 Diamond-type

从成熟度来看，OT 显然是一种更成熟的方案，但是 CRDT 是一种更有前景的方案 —— 又学到了一种没有用的屠龙术。

**Rust**

为什么都是 Rust 语言呢？~~因为我基本上只会在工作的时候使用 Java~~，在去中心化的场景下，一种能跨端、系统、设备的语言必然是一种更好的选择。任何能够用 Rust 实现的应用系统，最终都必将用 Rust 实现。

**Rust CRDT**

在 CRDT 的技术选型上，有一系列的成熟选择：

* 基于 Rust 语言的 [Automerge RS](https://github.com/automerge/automerge-rs) 提供了全方面的解决方案：服务器、浏览器端（WASM）、浏览器（JS） 等。
* 基于 Rust 语言与 Y.js 成熟经验的 [Y CRDT](https://github.com/y-crdt/y-crdt) 是一个更靠谱的方案。
* 基于 Rust 语言但是性能更好的 **[Diamond-type](https://github.com/josephg/diamond-types) ，**其作者原来是 Google Wave 开发者，外加 ShareJS、ShareDB 的创始人。

最后，我们选择的是 [Diamond-type](https://github.com/josephg/diamond-types)，虽然它 API 不完全，可能会带来更多的问题。但是，正经、成熟的方案谁在工作之后用啊。这种特别容易带来 error 的代码库，总会让你去想着，我要再造一个更好的轮子。

Tips：与采用代码相应的库相比，还有一种作法是通过数据库来解决，诸如于 [ShareDB](https://github.com/share/sharedb/)，不过它当前只支持 OT。在底层内建于对 CRDT 与协作的支持，会降低我们的开发成本。

## 编辑：多端 CRDT 与编辑器集成

在选用了 Rust 作为 CRDT 的语言之后，我们就自然可以很好利用 Rust 语言的跨平台特性，将它编译为 WASM。如此一来，在浏览器端与服务端中，我们就可以使用同样的 CRDT API。

所以，剩下的工作就是日常的搬砖。

### 服务端：Actix + Diamond Types + CRDT

对于服务端来说，它本身其实也是个客户端，只需要接受客户端生成的 patch 即可，在合并了 patch 之后，将它广播出去即可：

```rust
let before_version = live.lock().unwrap().version();
after_version = self.ops_by_patches(agent_name, patches).await;
// or let after_version = self.insert(content, pos).await;
// or after_version = self.delete(range).await;
let patch = coding.patch_since(&before_version);
```

Feakin，当前版本在这里，除了支持 patch，还可以同时支持 ins、del 这样的操。核心的代码就这么几行，剩下的代码都是 CRUD，没啥好玩的。

### 客户端：编辑生成 patches

从结果代码来说，这部分相当的简单：：

```javascript
let localVersion = doc.getLocalVersion();
event.changes.sort((change1, change2) => change2.rangeOffset - change1.rangeOffset).forEach(change => {
  doc.ins(change.rangeOffset, change.text);
  doc.del(change.rangeOffset, change.rangeLength);
})
let patch = doc.getPatchSince(localVersion);
```

由于在前端中 Feakin 采用的是 monaco 的实现，需要在发生变更时，执行 `ins` 和 `del` 等，以生成 patch。

### 客户端：编辑器应用 patches

对于客户端来说，接受 patch 并应用也不复杂，然而我被坑了一晚上（被坑在了如何动态更新 Monaco 的模型上）：

```javascript
let merge_version = doc.mergeBytes(bytes)
doc.mergeVersions(doc.getLocalVersion(), merge_version);

let xfSinces: DTOperation[] = doc.xfSince(patchInfo.before);
xfSinces.forEach((op) => {
   ...
});        
```

唯一比较麻烦的点在于，当我们接受了 patch 之后，还需要把变更同步给编辑器。诸如于，我们接受了在 12 位置插入 a 字符，编辑器需要去插入这个 a。同时，在这个短暂的瞬间，我们还需要把编辑器锁住。

TIP：顺带一提，[yjs](https://github.com/yjs) 提供了不同编辑器的支持，可以在开发时，参考一下如何使用编辑器的 API —— 只要是 Monaco Editor 的 API 文档，一言难尽。

## 小结

最后，我们再回顾一下我们所需要的三个元素：

* 在线。如何选择合适的通讯协议和数据格式？
* 协作。如何基于 CRDT 构建去中心化的协作？
* 编辑。如何实现多端同步与编辑？

在这里，虽然我们简单完成了 Feakin 的在线协作，但是我们依旧有一系列的东西可以玩：

* 编码与解码优化。JSON 的序列化与反序列化会带来性能问题。
* 完善协作形态。诸如于 Cursor 的显示等。
* 异常场景处理。尚未处理各种异常状态

除此呢，下一步，我们应该如何有结地结合在线协作与图即代码？诸如于：

* 基于代码化的在线 DDD 协作设计
* 基于代码化的架构图绘制

如果你对这些有兴趣，也欢迎来联系我们，加入 Feakin 的开发。

参考资源：

* 《[I was wrong. CRDTs are the future](https://josephg.com/blog/crdts-are-the-future/)》
* 《[5000x faster CRDTs: An Adventure in Optimization](https://josephg.com/blog/crdts-go-brrr/)》
* <https://crdt.tech/>
* <https://github.com/yjs/y-monaco>
* [Awesome Opensource Graph Library](https://github.com/feakin/awesome-frontend-graph-library)
