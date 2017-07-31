---
layout: page
title: "[翻译]Next Consensus Architecture Proposal"
description: ""
---
{% include JB/setup %}

原文：<https://github.com/hyperledger/fabric/blob/master/proposals/r1/Next-Consensus-Architecture-Proposal.md>

作者: Elli Androulaki, Christian Cachin, Angelo De Caro, Konstantinos Christidis, Chet Murthy, Binh Nguyen, Alessandro Sorniotti, and Marko Vukolić

翻译：[梧桐树](https://wutongtree.com)

---

* 目录
{:toc}

---

本文阐述的是一个区块链基础架构，它的区块链节点角色划分为*Peer节点（peers）*（维护状态/总账的节点）和*投票节点（consenters）*（投票赞成区块链状态中交易顺序的节点）。在通常的区块链架构中（包括2016年7月的Hyperledger fabric），这些角色都是统一的（参看Hyperledger fabic的*validating peer*）。这个架构还引入了背书节点（endorsers）的概念，它其实是一种特殊类型的节点，用来模拟执行和*背书*交易（大概和Hyperldger fabric 0.5-developer-preview的执行/验证交易是类似的）。

这种架构和peers/consenters/endorsers统一的设计相比有如下的优势：

* **链码信任的灵活性（chaincode trust flexibility）**。从架构上，区分开链码（区块链应用程序）的*信任假设*与共识服务的信任假设。就是说，参与到共识服务中的节点可能是背书节点（consenters），也能容忍一些节点失效或者破坏。还有，每个链码的背书节点可能会是不一样的。
* **可扩展性（Scalability）**。因为负责特定链码的背书节点和投票节点是正交的，这比所有功能都集中在同一个节点能更好的扩展。尤其是，当最终不同的链码都指定完全不同的背书节点时，链码会在不同的背书节点上分散开，链码执行（背书：endorsement）就可以并行了。而且，链码执行是非常耗时的，不再是共识服务的关键路径了。
* **机密性（Confidentiality）**。这个架构使对内容和执行状态更新有*机密性*要求的链码部署更容易了。
* **共识模块性（Consensus modularity）**。这个架构是*模块化的*，运行插件化的共识实现。

## 目录
1. 系统架构
1. 基本的交易背书工作流
1. 背书策略
1. 区块链数据结构
1. 状态转换和检查点
1. 机密性

---

## 1. 系统架构（System architecture） {#section1}
区块链是由很多相互通信的节点组成的分布式系统。区块链运行程序（称为链码），保存状态和总账数据，执行交易。链码是处于中心位置，交易是在链码上执行的操作，并且只有链码才能更改状态。交易必须要有背书，只有有背书的交易才能被提交，才能使状态生效。有一个或者多个具有管理功能和参数的特殊链码，统称*系统链码（system chaincodes）*。

### 1.1. 交易（Transactions）

交易有两种类型：

* *部署交易（Deploy transactions）* 用程序作为它的一个参数创建新的链码。当部署交易成功执行以后，我们就说链码被安装到“链上”了。
* *调用交易（Invoke transactions）* 在前面部署的链码环境上执行一个操作。一个调用交易指的是链码及其提供的功能。如果成功的话，链码会执行指定的功能，可能会修改相关的状态，然后返回输出。

后面还会介绍，部署交易是调用交易的特殊情况，调用交易创建新的链码就是在系统链码上的调用交易。

**备注**：*本文假定一个交易要么创建新的链码，要么调用一个已经部署链码提供的交易。本文不会涉及如下内容：a) 跨链交易的支持; b) 查询交易（只读的）的优化*。

### 1.2. 状态（State）

**区块链状态（Blockchain state）**。区块链的状态（世界状态：world state）有一个简单的结构，模型化就是一个版本化的键值存储（KVS），其中键是名称，值是任意的二进制大对象。这些条目由运行在区块链上的链码（应用）通过`put`和`get`的KVS进行操作。这些状态是被永久存储的，状态的更新也会有记录。注意版本化的KVS只是状态模型，实现方式可以使用真正的KVS系统，也可以采用关系型数据库系统或者其他的解决方案。

正式点表示，区块链状态`s`是`K -> (V X N)`映射的一个元素，其中：

* `K`是键的集合
* `V`是值的集合
* `N`是无数个有序的版本号集合。单射函数`next: N -> N`输入`N`的一个元素，返回下一个版本号。

`V`和`N`都有一个特殊的元素`\bot`，代表`N`最小的元素。初始化的时候，所有的键都被映射到`(\bot,\bot)`。`s(k)=(v,ver)`这个表达式中，`v`用`s(k).value`来表示，`ver`用`s(k).version`来表示。

KVS操作是这样定义的：

* `put(k,v)`操作。对`k\in K`和`v\in V`键值对，区块链状态`s`的新状态`s'`计算方法是：`s'(k)=(v,next(s(k).version))`。并且对所有的`k'!=k`，表达式`s'(k')=s(k')`都成立。
* `get(k) `操作。返回`s(k)`。

**状态分区（State partitioning）**。KVS中的键可以通过名称就能识别出它们属于哪个链码，所以只有特定链码的交易才能修改属于这个链码的键。原则上，任意的链码都能读取属于其他链码的键（机密链码的状态是不能明文读取的，参考第6部分）。*修改2个或者多个链码状态的跨链交易，以后会支持。*

**总账（Ledger）**。区块链状态的变化过程（历史）是保存在*总账*中的。总账是交易区块的哈希链（hashchain），总账中的交易是全序的。

区块链状态和总账会在[第4部分](#section4)详细描述。

### 1.3. 节点（Nodes）

节点是区块链的通信实体。节点是一个逻辑的概念，不同类型的节点是可以运行在同一个物理服务器上的。重要的是节点是怎么被分组成“信任域（trust domains）”，怎么和控制它们的逻辑实体关联的。

有3种类型的节点：

1. **客户端（Client）**或者**提交客户端（submitting-client）**：提交实际交易请求的客户端。

2. **Peer节点（Peer）**：一个提交交易，维护状态和总账副本的节点。Peer节点有两种特殊的角色：
 
    a. **请求节点（submitting peer或者submitter）**

    b. **背书节点（endorsing peer或者endorser）**

3. **共识服务节点（Consensus-service-node）**或者**投票节点（consenter）**：一个运行了有送达保证（delivery guarantee，比如原子广播）通信服务的节点，送达保证典型的实现方法是运行共识服务。

注意，投票节点和客户端是不维护总账和区块链状态的，只有Peer节点才会。

下面详细解释节点的类型。

#### 1.3.1. 客户端（Client）

客户端表示的是代表终端用户的实体，它必须连接到Peer节点才能和区块链通信。客户端可以根据自己的选择连接到任意一个Peer节点上，创建交易再调用交易。

#### 1.3.2. Peer节点（Peer）

Peer节点通过共识服务通信维护区块链状态和总账。它们从共识服务接收有序的更新状态，然后更新本地维护的状态。

Peer节点还可以有下面描述的两种角色：

* **请求节点（Submitting peer）**。*请求节点*是一种特殊的角色，它给客户端提供接口，这样客户端就可以连接到请求节点调用交易和获取结果。这个Peer节点代表一个或者多个客户端和其他节点通信来执行交易。

* **背书节点（Endorsing peer）**。*背书节点*的特殊功能是对特定的链码，在其提交交易前对它进行*背书*。每个链码都可以指定一个*背书策略*，可能会包含一个背书节点的集合。策略定义了一个有效交易背书（典型情况是背书节点的签名集合）的充要条件，这会在[第2部分](#section2)和[第3部分](#section3)阐述。一个特殊情况是，安装新链码的部署交易中，（部署）背书策略是系统链码的背书策略指定的。

我们强调一个Peer节点同时有请求节点和背书节点角色的时候，就叫它*提交节点（committing peer）*。

#### 1.3.3. 共识服务节点（Consensus service node (Consenters)） {#section1_3_3}

*投票节点*构成了*共识服务*，也即，一个提供交付保证的通信组织。共识服务可以有多种实现方式：从中心化的服务（比如：部署和测试）到其目标是不同网络和节点失效模型的分布式协议。

Peer节点是共识服务的客户端，共识服务给它提供了一个有交易信息广播服务的共享*通信通道*。Peer节点连接到通道上，可以发送或者接收消息。通道支持所有消息的*原子*交付，就是，消息通信是全序交付的和（跟实现相关）可靠的。换句话说，通道输出给所有连接的节点相同的消息，而且输出的逻辑顺序是相同的。原子通信保证又叫*全序广播（total-order broadcast）*，*原子广播（atomic broadcast）*，或者分布式系统环境下的*共识（consensus）*。通信过的消息就是作为包含在区块链状态中的候选交易。

**共识通道分区（Partitioning (consensus channels)）**。共识服务可能支持多*通道*，类似发布/订阅（pub/sub）消息系统的*主题（toptics）*。客户端连接到一个指定的通道，就可以发送或者获取到达的消息。通道可能会有分区，客户端连接到一个通道是不知道其他通道的存在的，但是客户端可以连接到多个通道。为简单起见，本文后面的部分，除非明确的提到了的其他情况，我们都假设共识服务是由单个通道/主题组成的。

**共识服务API（Consensus service API）**。Peer节点是通过共识服务的API连接到共识服务提供的通道的。共识服务API有两种基本的操作（更通用的说法：*异步事件*）：

* `broadcast(blob)`：请求节点调用它在通道上广播任意的消息`blob`。这在BFT中，当给服务发送一个请求时，又称为`request(blob)`。

* `deliver(seqno, prevhash, blob)`：共识服务调用它给Peer节点发送包含非负序列号`seqno`和最近一次发送消息哈希`prevhash`的消息`blob`。换句话说，它是共识服务的输出事件。`deliver()`在发布/订阅系统中称为`notify()`，BFT系统中称为`commit()`。

    注意共识服务客户端（即Peer节点）只通过`broadcast()`和`deliver()`事件和服务进行交互。

**共识内容（Consensus properties）**。共识服务（或者原子广播通道）有如下的保证，回答了这些问题：`广播消息发生了什么`，`交付消息间有什么关系`?

1. **安全性：一致性保证（Safety (consistency guarantees)）**：只要Peer节点连接到通道有足够长的时间（它们可以断开或者宕掉，重启或者重新连接就可以），它们就能看到`相同`顺序的交付消息`(seqno, prevhash, blob)`。这意味着，给所有Peer节点的输出（`deliver()`事件）都是`相同顺序`的，相同的序号都是`相同的内容`（`blob`和`prevhash`）。需要注意的是，这只是一个`逻辑顺序`，并且，一个Peer节点的`deliver(seqno, prevhash, blob)`并不需要和另外一个Peer节点上输出了相同消息的`deliver(seqno, prevhash, blob)`有时间上的关联。换句话说，给定一个特定的`seqno`，`不会`有两个正常的Peer节点发送`不同`的`prevhash`和`blob`。而且是，除非有共识客户端（Peer节点）真正的调用了`broadcast(blob) `，是不会有`blob`消息发送的，最好是，每个广播的`blob`只发送`一次`。

    还有，`deliver()`事件包含了上一个`deliver()`事件的加密哈希`prevhash`。当共识服务执行一个原子广播保证时，`prevhash`是序号为`seqno-1`的`deliver()`事件的加密哈希。这就在不同的`deliver()`事件之间建立了一个哈希链，能够用来帮助验证共识输出的完整性，后面的[第4部分](#section4)和[第5部分](#section5)会讨论这个。第一个`deliver()`事件是一个特殊情况，`prevhash`会有一个默认值。

2. **活跃度：交付保证（Liveness (delivery guarantee)）**：共识服务的活跃度保证是共识服务的实现指定的。精确的保证要依赖网络和节点失效模型。

    原则上，如果提交没有失败，共识服务应该保证每个连接到共识服务的Peer节点最终都能提交每个发出的交易。

总结一下，共识服务保证了下面的内容：

* *协议（Agreement）*。正常Peer节点的任意两个事件，`deliver(seqno, prevhash0, blob0)`和`deliver(seqno, prevhash1, blob1)`，如果有相同的`seqno`，则有`prevhash0==prevhash1`，`blob0==blob1`成立；

* *哈希链完整性（Hashchain integrity）*。正常Peer节点的任意两个事件，`deliver(seqno-1, prevhash0, blob0)`和`deliver(seqno, prevhash, blob)`，有`prevhash = HASH(seqno-1||prevhash0||blob0)`；

* *不跳跃（No skipping）*。如果一个共识服务给一个正常节点`p`输出`deliver(seqno, prevhash, blob)`，如果`seqno>0`，则`p`一定已经交付了`deliver(seqno-1, prevhash0, blob0)`事件；

* *不创建（No creation）*。一个正常节点的任意`deliver(seqno, prevhash, blob)`事件，前面一定有一个Peer节点发送了`broadcast(blob)`事件；

* *不重复（No duplication，可选的）*。对任意的两个事件`broadcast(blob)`和`broadcast(blob')`，当正常Peer节点交付了两个事件，`deliver(seqno0, prevhash0, blob)`和`deliver(seqno1, prevhash1, blob')`，如果`blob`==`blob'`，则有`seqno0==seqno1`和`prevhash0==prevhash1`成立；

* *活跃度（Liveness）*。如果一个正常Peer节点产生了`broadcast(blob)`事件，则每个正常Peer节点“最终”都会发出一个`deliver(*, *, blob)`事件，其中`*`代表任意值。

---

## 2. 交易背书的基本流程 (Basic workflow of transaction endorsement) {#section2}

下面我们概要性的介绍一个交易的高层请求流程。

**备注**：*注意后面的协议并不假定所有的交易都是确定性的，允许不确定性的交易*。

### 2.1. 客户端创建一个交易并发送给自己选择的一个请求节点 {#section2_1}

要调用一个交易，客户端发送如下的消息给请求节点`spID`。

`<SUBMIT,tx,retryFlag>`，其中：

- `tx=<clientID,chaincodeID,txPayload,clientSig>`，其中：
    - `clientID`是提交客户端的ID，
    - `chaincodeID`指的是交易所属的链码，
    - `txPayload`是发出的交易本身的有效载荷，
    - `clientSig`是客户端`tx`消息中其他项的签名。
- `retryFlag`是一个布尔值，告诉请求节点万一交易失败了要不要重传，

调用交易和部署交易的`txPayload`是不一样的（即，调用交易会引用一个部署相关的系统链码）。如果是**调用交易**，`txPayload`只有一个项：

- `invocation = <operation, metadata>`，其中：
    - `operation`代表链码的操作（函数）和参数，
    - `metadata`代表调用相关的属性。

如果是**部署交易**，会有两个项：

- `chainCode = <source, metadata>`，其中：
    - `source`代表链码的源码路径，
    - `metadata`代表链码和应用相关的属性。
- `policies`包含了所有Peer节点都能访问的链码策略，比如背书策略。

**待办**：确定是否要在客户端显示的包含本地/逻辑时间（时间戳）。

### 2.2. 请求节点准备一个交易并发送给背书者获取背书 {#section2_2}

请求节点收到客户端发来的`<SUBMIT,tx,retryFlag>`消息后，首先要验证客户端的签名`clientSig`，然后就准备交易。请求节点会临时的*执行*交易（`txPayload`），过程是通过执行交易关联的链码（`chaincodeID`）和拷贝请求节点本地保存的状态。

执行的结果是，请求节点会计算*状态更新*（`stateUpdate`）和*版本依赖*（`verDep`），这在DB语言中又叫*MVCC+postimage*。

还记得状态是键值对（k/v）组成的吧。所有的k/v条目都是版本化的，就是说，每个条目都包含排过序的版本信息，每个键更新的时候都会递增版本信息。Peer节点解析链码访问交易记录的所有键值对，可以读可以写，它还没有更新它的状态。更具体的说：

* `verDep`是一个元组`verDep=(readset,writeset)`。给定请求节点执行交易前的一个状态`s`：
    * 对交易读取的每个键`k`，把`(k,s(k).version)`加入到`readset`中。
    * 对交易修改的每个键`k`，把`(k,s(k).version)`加入到`writeset`中。
* 另外，对交易修改的每个键`k`的新值`v'`，把`(k,v')`加入到`stateUpdate`中。`v'`也可以是新值相对旧值`s(k).value`的增量。

实现时可以把`verDep.writeset`和`stateUpdate`放到同一个数据结构中。

然后，`tran-proposal := (spID,chaincodeID,txContentBlob,stateUpdate,verDep)`，其中：`txContentBlob`是链码/交易相关的信息，目的是能标识`tx`（比如，`txContentBlob=tx.txPayload`）。更详细的信息[第6部分](#section6)会介绍。

所有节点都会用`tran-proposal`的加密哈希来作唯一交易标识符`tid`（即：`tid=HASH(tran-proposal)`）。

然后请求节点就把交易（`tran-proposal`）发送给链码关联的背书者。背书节点是根据解析策略，Peer节点的可用性和与请求节点的连通性来选择的。比如，可以把交易发送给指定`chaincodeID`*所有*的背书节点。有可能，有的背书节点是离线的，还有一些会拒绝对交易进行背书。请求节点尽量用可用的背书节点满足策略要求。

请求节点`spID`给背书节点`epID`发送的交易消息是：

`<PROPOSE,tx,tran-proposal>`。

**潜在优化**：实现上可以优化下`tx.chaincodeID`和`tran-proposal.chaincodeID`里重复的`chaincodeID`，`tx.txPayload`和`tran-proposal.txContentBlob`里重复的`txPayload`。

最后，请求节点在内存中保存`tran-proposal`和`tid`，等待背书节点的回复。

**其他设计**：*这里请求节点和背书节点是直接通信的。这可以是共识服务的一个功能，在这种情况下，需要确定faric需不需要对它们的通信采用原子广播交付保证，还是用简单的p2p通信。这时共识服务也要负责根据策略收集背书再发送给请求节点*。

**待办**：需要确定请求节点和背书节点之间的通信：用p2p还是通过共识服务。

### 2.3. 背书节点接收交易并给交易背书

链码`tran-proposal.chaincodeID`对应的背书节点收到通过`PROPOSE`消息发送来的交易后，执行如下步骤：

* 背书节点验证签名`tx.clientSig`，检查`tx.chaincodeID==tran-proposal.chaincodeID`是否相等。

* 背书节点模拟交易（用`tx.txPayload`），验证状态更新和依赖信息都是正确的。如果所有的都是有效的，它就对`(TRANSACTION-VALID, tid)`进行数字签名，生成`epSig`。然后背书节点发送`<TRANSACTION-VALID, tid,epSig>`消息给请求节点（`tran-proposal.spID`）。

* 如果背书者模拟交易是失败了，有几种情况：

    a. 如果背书者获取到的状态更新和`tran-proposal.stateUpdates`里的不一样，它就对`(TRANSACTION-INVALID, tid, INCORRECT_STATE)`进行签名并发送给请求节点。

    b. 如果背书者发现了比`tran-proposal.verDeps`更新的数据版本，它就对`(TRANSACTION-INVALID, tid, STALE_VERSION)`进行签名并发送给请求节点。

    c. 如果背书者因为其他的一些原因（内部的背书策略、交易错误等）不想对交易进行背书，它就对`(TRANSACTION-INVALID, tid, REJECTED)`进行签名并发送给请求节点。

注意背书者在这一步还没有改变状态，状态的更新也不会写日志。

**其他设计**：*对无效交易，背书节点可以不通知请求节点，不用显示的发送`TRANSACTION-INVALID`通知*。

**其他设计**：*背书节点把`TRANSACTION-VALID/TRANSACTION-INVALID`消息及其签名发送给共识服务*。

**待办**：确定采用上面的哪种设计。

### 2.4. 请求节点收集交易的背书并通过共识服务广播出去 {#section2_4}

请求节点会一直等待，直到接收的消息和对`(TRANSACTION-VALID, tid)`进行的签名，足够判断这个交易提议（transaction proposal）是背书过（可能包含它自己的签名）的。这个过程依赖背书策略（再看看[第3部分](#section3)）。如果满足背书策略，交易就是`背书`过了，注意这会儿它还没有提交（committed）。从背书节点收集到的能确定交易是背书过的签名就叫*背书（endorsement）*，请求节点把它们存储到`endorsement`里。

如果请求节点没有收集到交易提议的背书，它就丢弃掉这个交易并通知提交客户端。如果提交客户端设置（看[步骤1](#section2_1)和`SUBMIT`消息）了`retryFlag`，请求节点可能（根据请求节点的策略）会对交易进行重试（[步骤2](#section2_2)）。

有了有效背书的交易，我们开始使用fabric的共识服务。请求节点使用`broadcast(blob)`调用共识服务，其中`blob=(tran-proposal, endorsement)`。

### 2.5. 共识服务发布交易给Peer节点 {#section2_5}

当出现一个`deliver(seqno, prevhash, blob)`事件，一个Peer节点更新所有序号小于`seqno`消息的状态，过程是这样的：

* Peer节点根据链码（`blob.tran-proposal.chaincodeID`）的策略检查`blob.endorsement`是否有效（这个步骤可以不用等到更新序号小于`seqno`的状态这个时候）。

* Peer节点同时验证依赖`blob.tran-proposal.verDep`是有效的。

根据状态更新选择的一致性内容（consistency property）或者“隔离保证（isolation guarantee）”不同，依赖验证有多种实现方式。比如，**可串行性（serializability）**可以要求每个`readset`和`writeset`里键对应的版本号必须和状态里面键的版本号相同，并抛弃掉不能满足这个要求的交易。另外一个例子，**快照隔离（snapshot isolation ）**要求`writeset`里所有的键，状态里的键和依赖数据里的版本号都是一样的。数据库著作里包含了更多的隔离保证。

**待办**：确定坚持可串行性还是允许链码指定隔离级别。

* 如果所有的检查都通过了，这个交易就被认为是*有效的（valid）*或者*提交的（committed）*。这就是说，一个Peer节点添加一个交易到总账上，然后会在区块链状态上执行`blob.tran-proposal.stateUpdates`。只有提交的交易才会修改状态。

* 如果有任何检查失败了，交易就是无效的，Peer节点会丢弃这个交易。重要的是要注意无效的交易是没有提交的，不会修改状态，也不会被记录。

另外，请求节点会通知客户端丢弃的交易。如果提交客户端设置（看[步骤1](#section2_1)和`SUBMIT`消息）了`retryFlag`，请求节点可能（根据请求节点的策略）会对交易进行重试（[步骤2](#section2_2)）。

![交易流程图解](../images/flow-2.png)

图1. 交易流程图解（通用情况路径）

---

## 3. 背书策略 {#section3}

### 3.1. 背书策略规范

**背书策略**是会对交易进行背书的*条件*。背书策略是链码安装的时候`deploy`交易指定的。只有根据策略背书策略过的交易才是有效的。链码的调用交易会先获取满足链码策略的背书，否则是提交不了的。这是通过请求节点和背书节点之间的交互完成的，在[第2部分](#section2)已经介绍过了。

形式上，背书策略是对交易、背书和可能有的下一步状态判断是TRUE或者FALSE的断言。部署交易的背书策略是从系统层面的策略获取的（比如，从系统链码获取）。

形式上，背书策略是关于特定变量的断言。实际上，它可以是：

1. 链码相关的键或者标识符（链码的元数据里面能找到），比如，背书节点集合；
2. 更多的链码元数据；
3. 交易本身的元素；
4. 可能还有其他的。

背书策略断言的评估必须是确定性的。背书策略不能是复杂的，也不能是“小的链码（mini chaincode）”。背书策略规范语言必须是有限的，并且要能够增加确定性。

断言列表是由简单到丰富，复杂性是由易到难的。就是说，支持只有键和节点标识符的策略是相对比较简单的。

**待办**：确定背书策略的参数。

断言可能会包含结果是TRUE或者FALSE的逻辑表达式。一般情况下，这个条件里会包含链码背书节点对交易调用签发的数字签名。

假设链码指定了一个背书节点集合`E = {Alice, Bob, Charlie, Dave, Eve, Frank, George}`，一些示范性的策略：

- E集合所有元素的有效签名。
- E集合任意一个元素的有效签名。
- 满足`(Alice OR Bob) AND (any two of: Charlie, Dave, Eve, Frank, George)`这个条件的背书节点的有效签名。
- 7个背书节点中任意5个的有效签名。（更通用的情况是，有`n > 3f`个背书节点的链码，需要`n`个背书节点中有`2f+1`个有有效签名，或者任意一个*超过*`(n+f)/2`个背书节点的组有有效签名。）。
- 假设背书节点都有一个“投注”或者“权重”，比如`{Alice=49, Bob=15, Charlie=15, Dave=10, Eve=7, Frank=3, George=1}`，总投注是100：策略要求多数投注集合的有效签名（即，一个总投注严格大于50的组），比如任何和George不一样`X`的`{Alice, X}`，或者`{除开Alice的所有人}`等等。
- 上面的例子里投注可以是静态的（链码的元数据里写死的）或者动态的（比如，依赖链码的状态并且可以在执行过程中修改）。

这些策略能起到多少作用还依赖于应用，对有背书节点失效或者破坏时期望的恢复力，还有其他不同的属性。

### 3.2. 实现

通常情况下，背书策略会根据背书节点要求的签名来制定。链码的元数据必须要包含相应的签名验证密钥。

通常，背书是由一组签名组成的。每个Peer节点或者能获取到链码元数据（包含签名验证密钥）的投票节点都可以本地验证背书，因为它们不需要和其他节点进行交互。节点也不需要访问状态才能验证背书。

链码其他元数据的背书也可以用同样的方法验证。

**待办**：形式化背书策略，设计具体的实现。

---

## 4. 区块链数据结构 {#section4}

区块链包含3种数据结构：a) *原始总账（raw ledger）*，b) *区块链状态（blockchain state）*，c) *已验证总账（validated ledger）*。区块链状态和已验证总账是为了效率维护的，它们都可以从原始总账导出来。

* *原始总账（Raw ledger (RL)）*。原始总账包含了Peer节点的共识服务输出的所有数据。它是`deliver(seqno, prevhash, blob)`事件的序列，计算`prevhash`后组成了一个哈希链，前面已经介绍过。原始总账包含了`有效的`和`无效的`交易，能够提供系统操作过程中出现过的所有成功和不成功的状态更新、改变状态的尝试等可验证的历史记录。

    原始总账允许重放Peer节点所有交易的历史并重建区块链状态（见下文）。它还给请求节点提供了的`无效的`（未提交的）交易的信息，基于这个信息请求节点的操作已经在[第2.5部分](#section2_5)描述过了。

* *区块链状态（(Blockchain) state）*。状态是Peer节点维护的（KVS的形式），它是可以从原始总账中通过**过滤掉无效交易**导出来（[第2.5部分](#section2_5)介绍过，看第5步的图1），然后更新有效交易到状态上（对`stateUpdate`里的每个`(k,v)`，都执行一下`put(k,v)`，或者执行相对上一个状态的增量）。

    就是说，有了共识保证，所有的正常Peer节点都能收到相同顺序的`deliver(seqno, prevhash, blob)`事件。因为对背书策略和状态更新（[第2.5部分](#section2_5)）版本依赖的计算方法都是确定的，所有的正常Peer节点都能确定blob里面的交易是否是有效的。因而，所有Peer节点都是以相同的方式提交、采用同样的交易序列并更新它们的状态。

* *已验证总账（Validated ledger (VL)）*。为了维护只包含有效的和提交的交易（比如，比特币里面有），除了状态和原始总账，Peer节点还维护了`已验证总账`。这是从原始总账中过滤掉无效交易后形成的哈希链。

### 4.1. 批量处理和块信息

共识服务应该*批量*输出blobs，而不是输出单个的交易（blobs）。这种情况下，共识服务必须要利用并告知每个批块里确定性的交易顺序。每个批块（batch）里交易的数量是共识实现动态选择的。

共识的批量处理不会影响原始总账的构建，原始总账仍然是交易的哈希链。和输出单个交易不同的是，原始总账变成了批块的哈希链而不是单个交易的哈希链。

批量处理时，已验证总账（可选的）*区块*的构建过程是这样的：因为原始总账里可能包含无效的交易（即，无效背书的交易或者无效版本依赖的交易），Peer节点会先过滤掉这些交易再交付给区块。每个Peer节点都是自己独自完成这个过程的。一个区块就是过滤掉无效交易后的共识批块。这些块的大小是可以动态调整的，也可能是空的。图2是区块构建的图解。

![从原始总账批量到已验证总账图解](../images/blocks-2.png)

图 2. 从原始总账批量到已验证总账图解

### 4.2. 区块形成链 {#section4_2}

跟[第1.3.3部分](#section1_3_3)描述的一样，共识服务输出原始总账的批块后组成了一个哈希链。

每个Peer节点都会把已验证区块链成一个哈希链。一个批处理有效的和提交的交易形成一个区块，所有的区块链在一起形成一个哈希链。

具体来说，每个已验证总账包含：

* 前面一个区块的哈希

* 区块号

* 从上一个区块形成之后Peer节点提交的所有有效交易的有序列表（即，相应批块的有效交易列表）

* 产生当前区块的相应批块的哈希

Peer节点会把所有的信息连接在一起并计算哈希，得出已验证总账里区块的哈希。

## 5. 状态传输和检查点 {#section5}

通常情况下，正常运行时，Peer节点会从共识服务收到一系列的`deliver()`事件（包含批块交易），然后把这些批块追加到原始总账中，并相应地更新区块状态、已验证总账。

然而，由于网络划分或者Peer节点临时停电等原因，一个Peer节点可能错过原始总账的多个批块。这时，Peer节点就必须从其他Peer节点那里*传输状态*才能和网络中的其他Peer节点保存同步。本节就来看一个实现方法。

### 5.1. 原始总账状态传输（批量传输） {#section5_1}

为了解释清楚基本的*状态传输*怎么实现的，假定Peer节点`p`原始总账的本地拷贝里最后一个批块序号是25（也就是，最后收到的`deliver()`事件的`seqno`等于25）。过了一段时间后，Peer节点`p`收到共识服务的`deliver(seqno=54,hash53,blob)`事件。

这个时候，Peer节点`p`发现它的原始总账副本里缺少26-53号批块。`p`采用p2p通信从其他节点获取缺失的批块。它呼叫其他Peer节点传给它缺失的区块。在传输缺失批块过程中，`p`继续监听来自共识服务的新批块。

注意`p不需要信任任何通过状态传输从那里获取缺失批块的Peer节点`。因为Peer节点`p`有批块53的哈希（就是，`hash53`），这是直接从共识服务那里获取到的，是`p`信任的，当收到了所有的批块，`p`就可以验证缺失区块的完整性。验证过程会检查他们是否是一个完整的哈希链。

当`p`获取到了所有的缺失批块并验证了缺失的批块号26-53，它就可以按照[第2.5部分](#section2_5)的步骤处理26-54的每一个批块，然后构造区块链状态和已验证总账。

注意即使`p`缺失一些序号比较大的批块，它仍然可以在收到序号小的区块的时候就开始重建区块链状态和已验证总账。但是，在保存状态和提交区块到已验证总账之前，Peer节点`p`还需要完成缺失区块的状态传输（我们给的例子里，要包含53号批块），还要处理[第2.5部分](#section2_5)描述的单独传输的批块。

### 5.2. 检查点

原始总账里包含了无效的交易，它们不需要永久保存。但是，Peer节点不能在建立了相应的已验证区块后就简单的丢弃原始总账批块，进而精简原始总账。就是说，这个时候，如果有新的Peer节点加入网络，其他Peer节点不能给它传输被丢弃的批块（在原始总账里），也不能让新加入的Peer节点相信传输给它的（已验证）区块的有效性。

为了便于精简原始总账，本文介绍一种`检查点`机制。它是在Peer节点网络之间建立已验证总账区块的有效性，允许建立了检查点的已验证总账区块替换丢弃的原始总账批块。这就减少了存储空间，因为没有必要存储单个的交易了。它还减少了新加入Peer节点重建状态的工作（因为它们不需要从原始总账中重构状态的时候构建单个交易的有效性了，只需要简单的重放已验证总账里的状态更新）。

注意检查点有利于精简原始总账，同时，它也只是一种性能优化，对于正确的设计来说检查点并不是必须的。

#### 5.2.1. 检查点协议

Peer节点每`CHK`个区块就周期性的执行检查点操作，`CHK`是可以配置的参数。开始的时候，Peer节点会给其他Peer节点广播`<CHECKPOINT,blocknohash,blockno,peerSig>`消息，其中，`blockno`是当前的区块号，`blocknohash`是它的哈希值，`peerSig`是Peer节点对`(CHECKPOINT,blocknohash,blockno)`的签名，这里的区块都是已验证总账里的。

一个Peer节点收集`CHECKPOINT`消息，直到它有足够正确的和`blockno`、`blocknohash`匹配的签名信息，就开始构建`有效的检查点`（看第5.2.2部分）。

给区块号为`blockno`、哈希为`blocknohash`的区块建立检查点的时候，一个Peer节点：

* 如果`blockno>latestValidCheckpoint.blockno`，则它设置`latestValidCheckpoint=(blocknohash,blockno)`;

* 保存构成一个有效检查点的Peer节点签名到集合`latestValidCheckpointProof`中；

* （可选）精简批块号小于等于`blockno`的原始总账。

#### 5.2.2. 有效检查点

显然，检查点协议抛了这个问题：*什么时候精简原始总账？多少`CHECKPOINT`消息是“足够多”？*。这是*检查点有效性策略*里面定义的，（至少）有两种可能的方法，也可能一起用：

* *本地检查点有效性策略（Local (peer-specific) checkpoint validity 策略(LCVP)，每个Peer节点相关的）*。本地策略就是给Peer节点`p`指定一个Peer节点集合，这个集合里的Peer节点都是`p`信任的，它们的`CHECKPOINT`信息就足够构建一个有效的检查点。比如，Peer节点*Alice*的LCVP定义的是，需要从`Bob`或者*同时收到Charlie和Dave*收到`CHECKPOINT`消息。

* *全局检查点有效性策略（Global checkpoint validity policy (GCVP)）*。检查点有效性策略可以全局指定。GCVP和LCVP非常相似，不同的地方在于GCVP是在系统层面（区块链）定义的，而LCVP是在Peer节点层面定义的。举个例子，GCVP可能会这么设置：

    * 每个Peer节点可以信任有*7*个不同Peer节点确认过的检查点；

    * 有这么一个部署环境，每个投票节点都是一个Peer节点，有`f`个投票节点可能是（拜占庭）故障的，每个Peer节点可以信任有`f+1`个不同Peer节点确认过的检查点。

#### 5.2.3. 已验证总账状态传输（区块传输）

除了能帮助精简原始总账外，检查点可以在已验证总账区块传输的时候传输状态。这可以部分替代原始总账的批块传输。

从概念上来说，区块传输机制和批块传输是类似的。前面有一个例子，Peer节点`p`丢失了序号为26-53的批块，从已经给50号区块建立了有效检查点的Peer节点`q`那里获取了状态。状态传输分为2步：

* 首先，Peer节点`p`尝试从Peer节点`q`那里获取检查点已经更新到第50区块的已验证总账。为此，`q`给`p`发送它本地的`(latestValidCheckpoint,latestValidCheckpointProof) `，我们这个例子是`latestValidCheckpoint=(hash50,block50)`。如果`latestValidCheckpointProof`满足`p`的检查点有效性策略，就可以传输26-50区块了。否则，`p`不会相信`q`的本地检查点是有效的。`p`可能选择传输原始总账（[第5.1部分](#section5_1)）。

* 如果26-50区块的传输都成功了，`p`还需要获取已验证总账51-53区块或者原始总账51-53批块的状态传输。为此，`p`可以简单的按照原始总账批量传输协议（[第5.1部分](#section5_1)），从`q`或者其他Peer节点获取这些信息。注意已验证总账区块包含各自原始总账批块的哈希（[第4.2部分](#section4_2)）。因而，即使Peer节点`p`在其原始总账中没有包含第50批块，原始总账的批量传输依然可以完成，因为第50区块是包含第50批块的。

## 6. 机密性（Confidentiality） {#section6}

这个部分介绍下这种架构怎么搞定那些要求敏感数据对某些Peer节点保密的部署。

**Fabic级别的机密性策略（Fabric-level confidentiality policies）**。简言之，这种架构在*fabric*层提供了一定的机密性：

* 机密链码的背书节点能够获取的明文信息：
    * 链码部署交易有效载荷；
    * 链码调用交易有效载荷；
    * 链码状态和状态更新；
* 其他Peer节点是获取不到这些明文信息的。

这里我们有一个假设，机密链码的背书集合里的背书节点是链码创建者信任的，它们可以获取链码的资源和维护机密性。

链码使用fabric机密性特性的等级是在部署阶段由部署者在**机密性策略**中指定的。具体来说，fabric提供如下机密性策略的支持：

| 策略ID | 部署有效载荷 | 调用有效载荷| 状态更新 |
|---|---|---|---|---|
| 策略`000` | 不受限制的 | 不受限制的 | 不受限制的 |
| 策略`010` | 不受限制的 | 机密的 | 不受限制的 |
| 策略`011` | 不受限制的 | 机密的 | 机密的 |
| 策略`110` | 机密的 | 机密的 | 不受限制的 |
| 策略`111` | 机密的 | 机密的 | 机密的 |
| 策略* `0xx` | 不受限制的 | 任意的 | 任意的 |
| 策略* `1xx` | 机密的 | 任意的 | 任意的 |

**表6.1.** Fabric机密性策略

*机密的（Confidential）*指的是交易的背书节点被限制访问相应的交易内容（就是部署/调用有效载荷或者状态更新等）。*不受限制的（In-the-clear）*指的是所有Peer节点都能读取或者访问相应的交易内容。*任意的（Any）*表明可以是*机密的（Confidential）*或者*不受限制的（In-the-clear）*。

下面的我们都假定部署交易的有效载荷（包括代码和应用数据/元数据）算是一个整体。但是，以后的设计规划里，我们会从机密性策略的角度把它们看成两个独立的部分。

本文档后面的部分，一个`机密的`的*链码*是其机密性策略不同于`000`的链码。还有，我们假设每个Peer节点都是有注册（加密）公钥的，[Hyperledger fabric协议规范](https://github.com/hyperledger/fabric/blob/master/docs/protocol-spec.md)里面有介绍。特别说明，所有的Peer节点都知道每个背书节点`e`的公钥`ePubKey`，背书节点自己知道对应的解密私钥。

**声明**：

*隐藏链码的交易活动*。要注意的是，现在的设计并没有隐藏链码标识符执行了哪些交易，也没有隐藏交易更新了链码状态的哪个部分。就是说，交易和状态都是加密的，但是提交节点是能获得活动和状态变化的。链码创建者允许提交节点通知第三方链码的活动，信任提交节点不会泄露这些信息。但是，我们在修订版的设计中准备修改它。

*状态机密性的粒度*。现在的设计把链码和它们的状态看成一个机密性域，没有划分成不同的键值对。可以在应用层给不同的状态部分设置不同的机密性策略，未来的目标是客户端的一个集合可以看到状态的某些部分，却看不到其他部分。就是说，从架构上是不反对有这种效果的，在单个或者多个链码有了足够的加密工具，应用层也是可以实现的。这会出现*链码层的机密性*。其他的要求，比如对提交节点隐藏背书节点标识符和背书策略会在以后的设计迭代中处理。

### 6.1 机密链码部署

#### 6.1.1 创建一个部署交易

要部署有机密性支持的链码`cc`，客户端（`cc`的部署者）部署`cc`需要设置：

* 链码本身，包括源代码，包含在`chainCode`里相关的元数据；
* 链码关联的和交易背书一起的策略，就是：
    - 背书策略`ccEndorsementPolicy`；
    - 背书者集合`ccEndorserSet`；
    - 链码机密性策略`ccConfidentialityPolicy`，指定的机密性级别在表6.1中有描述；
* 链码相关的加密材料，即，非对称加密密钥对`ccPubKey/ccPrivKey`，用来给链码的交易、状态、状态更新提供机密性。

这些信息是包含在（部署）交易`tx`里的，然后传给提交节点的`SUBMIT`消息。客户端按照下面的方法构造部署交易`cc`的有效载荷`txPayload`。

部署者先给`txPayload.policies`设置<`ccEndorsementPolicy, ccEndorserSet, ccConfidentialityPolicy`>。

要填写`txPayload.payload`，它先要检查`ccConfidentialityPolicy`是不是设置了`Deploy Payload = Confidential`。如果设置了，部署者用`ccPubKey`加密`chainCode`，所以只有给了权限的Peer节点才能看到来链码及其元数据。即：`txPayload.chainCode := Enc(ccPubKey, chainCode)`。

另外，链码相关的解密密钥`ccPrivKey`会分发给链码`cc`所有的背书节点（`ccEndorserSet`里的背书者）。过程是，对每个背书者`e`都会用其公钥对`ccPrivKey`进行加密，得到`ccEndorserSet, wrappedKey_e := Enc(ePubKey, ccPrivKey)`，其中，`ePubKey`是`e`的注册公钥（enrollment public key ）。然后，部署者再创建一个额外的项：`txPayload.endorserMessages := ccEndorserMessages`，其中，`ccEndorserMessages`包含了`ccEndorserSet`里所有`e`的`wrappedKey_e`。

我们强调一下部署交易可能会包含更多的项，这里忽略了是为了表述简单起见。另外，给所有背书节点发送加密的链码密钥时，实际加密`chainCode`时可以用混合的加密模式来提升更好的性能。详细的信息本文以后的版本会介绍。

链码`cc`会分配一个标识符，以下称为`chaincodeID`，可以是部署交易`tx`的哈希，[第2.2部分](#section2_2)介绍过。我们这里假定每个链码的`chaincodeID`都是唯一的，对所有Peer节点都是可见的。

**部署交易背书（Endorsement of Deploy Transaction）**。前面说过，每个处理链码部署的部署交易可以看成是系统链码的调用交易。假设链码用`dsc`表示，背书策略和背书节点集合分别为`dscEndorsementPolicy`和`dscEndorserSet`。这意味着，任何链码的部署交易，都需要根据`dscEndorsementPolicy`进行背书。

**待办**：需要考虑机密链码的情况下，部署交易本身的背书策略是不是能满足正在部署的链码背书策略，也即，`cc`的部署是不是满足`ccEndorsementPolicy`。或者，看是不是可以把部署交易拆分成两个部分，比如一个是`deploy`（只描述部署信息）和一个是创建链码的`install`。

#### 6.1.2 Peer节点处理一个部署交易

当一个Peer节点`e`交付了一个链码`cc`的部署交易，它至少有权限访问`chaincodeID`和`txPayload.policies`，里面包括了`ccConfidentialityPolicy`、`ccEndorsementPolicy`和`ccEndorserSet`。

如果`e`还是`cc`的背书节点，它还可以获取如下的内容：

* 链码的解密密钥，`ccPrivKey := Dec(ePrivKey, wrappedKey_e)`；
* 部署有效载荷的明文，`chainCode := Dec(ccPrivKey, txPayload.chainCode)`，如果`Deploy Payload = Confidential`。

鉴于有部署有效载荷`chainCode`的明文，在实际的安装之前，Peer节点`e`可以按照下面描述的方法进行一致性检查。它在后面的部署过程中，就可以用`chainCode`的明文了。

**一致性（Consistency）**。在部署链码的时候，协议需要保证每个背书节点`e`实际安装和运行相同的代码，即使链码的创建者（提交部署交易的Peer节点）想要背书节点运行不同的代码。要达到这种效果，`e`应该在部署交易的执行过程中有一个验证步骤，大体要能确保这些：部署交易要么`成功`然后Peer节点输出会被执行的链码`cc`，要么`失败`就输出对应的错误。部署要确保这个条件成立：如果两个独立的背书节点（正常非故障Peer节点）部署都成功了，它们部署的链码是相同的。这和拜占庭一致性广播的一致性属性（*Consistency property of a Byzantine Consistent Broadcast*）[[CGR11; Sec. 3.10]](http://distributedprogramming.net)条件是一样的。

因为每个背书节点执行的是从共识服务那里获取的部署交易，所有的背书节点收到的是相同的`chainCode`，里面可能包含了加密信息。但当背书节点用自己的密钥解密的时候，不能自然地保证每个背书节点从`cc`解密出的结果是一样的。

这可以有多种方式解决：一个解决方案是采用一个专门的在密文中包含随机数的多接受者加密模式。还可以，用零知识证明（zero-knowledge proof ）的可验证加密机制，所有接受者的明文都是相同的（好像这比第一种的效率低）。不管用哪种，部署者都要在部署交易`tx`中把确保一致性条件的附加数据包含进来。

**待办**：还有些细节没有说明，比如：

* 给出`dsc`实现的更多信息（可能要用专门的一个章节）。具体包含：
	- `dsc`代码本身及其实现`dscEndorsmentPolicy`（参考[第2.4部分](#section2_4)）
    - `dsc`里`tran-proposal`的细节。还是单独的一个章节？
* 考虑部署后链码标识符（`chaincodeID`）的其他实现；
* 描述怎么处理破坏了上面说的一致性属性的情况，比如，因为创建者给背书节点提供了错误的加密密钥。

### 6.2 机密链码调用

机密链码的调用交易必须要符合部署阶段（`ccConfidentialityPolicy`）指定的`静态`机密性策略。以后的版本会考虑系统运行时确定的`动态`机密性策略的可能。

机密链码和其他链码的调用方式类似。不同的地方是，一个和机密链码相关交易的提交节点必须是这个链码的背书节点。这就是说，它能访问保护链码及其状态的密钥。要维护无状态的客户端，每个Peer节点都要知道给定链码有哪些背书节点（参考[第6.1部分](#section6_1)，`ccEndorserSet`），还能给客户端推荐合适的提交节点。所以，后面的内容我们都假定提交节点同时也是背书节点。

#### 6.2.1 创建和提交一个机密交易

客户端是清楚创建交易的链码及其背书节点的。机密交易调用的`SUBMIT`消息的组成元素和非机密性的是一样的，也是`<SUBMIT, tx, retryFlag>`，其中，`tx=<clientID,chaincodeID,txPayload,clientSig>`，里面的`clientID`是fabric层某种形式的客户端标识，比如一个交易证书，`clientSig`是客户端对`tx`其他项的签名。

注意为了安全的目的，`tx`可以有更多的项，这里都故意忽略了，只是为了表述简单一点。

和非机密交易不同的是：如果和链码关联的机密性策略`ccConfidentialityPolicy`指定了`Invoke Payload = Confidential`，则客户端需要额外的加密调用参数和元数据，即`txPayload.invocation`是`invocation`用`ccPubKey`加密而来的，就是`txPayload.invocation := Enc(ccPubKey, invocation)`。

同样，混合加密机制也可以用来获得更好的性能。同样，部署阶段设置的密钥还可以用来生成其他的密钥，比如，加密关键状态的密钥，这样可以减少需要管理/分发密钥的总数量。

**待办**：可选择地提供一个还可以隐藏链码标识符的自定义加密方法。

#### 6.2.2 背书一个机密交易

收到和验证`<SUBMIT, tx, retryFlag>`消息时，为了临时执行交易相关的链码和准备发送给共识服务的交易，提交节点首先要解密机密交易有效载荷。更确切地说，如`ccConfidentialityPolicy`指定了`Invoke Payload`是机密的，提交节点先要获取对应的链码相关的解密密钥`ccPrivKey`再解密`txPayload.invocation`。已有假设提交节点是链码的背书节点。提交节点，假设为`e`，可以从链码`chaincodeID`的部署交易获取`wrapped_e`，再通过`wrapped_e`获得`ccPrivKey`。然后，就可以计算：`invocation := Dec(ccPrivKey, txPayload.invocation)`。

有了`invocation`的操作和元数据，提交节点就可以临时地用它本地状态执行交易来生成一个交易提议了。如果链码的机密性策略指定了`State`是机密的，提交节点就用`ccPrivKey`在边读取状态的时候边解密状态值了。

还有，当机密性策略指定了`State = Confidential`，要进行状态更新，提交节点需要用`ccPubKey`来加密`stateUpdates`里的新状态值。状态是键值对的形式，只有变化的值会被加密。版本依赖是不加密的。

提交节点的交易提议现在是这样构成的：`tran-proposal := (spID, chaincodeID, txContentBlob, stateUpdates, verDep)`，其中，`txContentBlob`是客户端提交的调用交易`tx`的某种形式。

提交节点创建一个`PROPOSE`消息发送给背书集合里其他的背书节点（[第2.2部分](#section2_2)描述过）：`<PROPOSE, tx, tran-proposal>`。

注意背书节点必须要验证`tx`里的`chaincodeID`和`tran-proposal`是一致的。

总之，这种机制确保了即使状态是加密的，链码的背书节点能够无障碍的访问状态，其他Peer节点就不能。一旦共识服务发送`stateUpdates`给Peer节点后，每个Peer节点都更新本地的状态。注意链码的背书节点还会透明的更新和操作密文，它们只有在对下一个交易背书的时候才需要访问明文。

**TOTO**：重新看看[第4部分](#section4)，更新的描述哪些部分需要放到总账上。






