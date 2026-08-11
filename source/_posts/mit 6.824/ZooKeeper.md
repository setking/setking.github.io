---
title: ZooKeeper
author: setKing
date: 2026-06-24 11:33:00 +0800
categories: [MIT 6.824]
tags: [论文, 分布式, 原理]
pin: true
math: true
mermaid: true
---

<center>ZooKeeper</center>

### 核心理念

ZooKeeper是一个协调分布式应用的基础架构，提供一个简单的、高性能的内核，供客户端构建更复杂的协调原语。ZooKeeper在多副本、中心化的服务中，组合了消息群发（_group messaging_）, 共享寄存器（_shared registers_）和分布式锁（_Distributed Lock_）服务。ZooKeeper 提供的接口有无等待的共享寄存器和一个事件驱动机制，与分布式文件系统的 cache 失效机制类似，提供了一个简单但功能强大的协调服务。ZooKeeper 还为每个客户端请求提供了 **FIFO 执行**的语义，和**ZooKeeper 状态更新请求线性化（\*Linearizable\*）** 的语义。ZooKeeper 可以广泛应用于各种客户端应用程序。

### 关键细节

#### 服务

ZooKeeper 客户端通过API库向ZooKeeper 递交请求，客户端在管理 client 和 ZooKeeper 服务间的网络连接的时候会暴露 ZooKeeper 服务的 client API 接口

1. ZooKeeper提供了一种称为“数据节点(znode)”的抽象，这些数据节点通过分层的命名空间来组织。客户端通过 ZooKeeper API 操纵在这个层级中的数据对象。znode的内部通过使用标准的 UNIX 文件系统路径符号来对命名空间分层。所有的 `znode` 都存储数据，并且除了临时节点（`ephemeral znodes`）外的所有 `znode` 都能拥有子节点。
   客户端可以创建两种 `znode`:
   - 常规节点（`Regular`）: client 可以通过创建或删除来显式操纵 `regular znodes`；
   - 临时节点（`Ephemeral`）: client 创建临时节点，这类节点要么显式删除它们，要么让系统在创建它们的会话终止时（故意或由于故障）自动删除它们。

   > 客户端创建新的 `znode` 的时候，可以设置 _sequential_ 标志。带有 _sequential_ 标志创建的节点会在节点名称后附加一个单调递增的计数器的值。

   ZooKeeper通过实现watch的通知机制使得client可以及时接收到值的变化。client设置watch=true时，服务器会在返回的信息发生变化时通知客户端，这不影响其他操作。Watches 是与会话关联的一次性触发器； 一旦被触发或者该会话关闭，它们将被注销。 Watches 表明节点发生了变更，但并不提供变更的内容。

   **数据模型**

   ZooKeeper 的数据模型本质上是一个简化了API的文件系统，只支持完整数据的读写，或者可以说是一个带有层级式 key 的 key/value 表。分层命名空间便于为不同应用的命名空间分配子树，也便于为这些子树设置访问权限。znodes 不是为通用数据存储设计的。

   **Sessions**

   session是ZooKeeper 维护client存活的重要指标，client连接到ZooKeeper 会初始化session并设置超时时间。如果session过期且在超时时间内没收到client任何消息，则认为client故障。client主动断开session或client故障时，session结束。`session` 中，客户端可以观察到一系列反应其操作执行的状态变化。 `session` 使客户端能够在 ZooKeeper 集群中透明地从一台服务器转移到另一台服务器，从而在 ZooKeeper 服务器之间持续存在。

   **Client API**
   - `create(path, data, flags)`: 根据路径名称 `path`，它存储的`data[]`，创建一个 `znode`, 并返回这个新的 `znode` 的名称。`flags` 允许客户端选择选定的 `znode` 类型：`regular`, `ephemeral` 及设置 `sequential` flag。
   - `delete(path, version)`: 如果 `znode` 符合给定的 `version` 版本，则删除`path` 下的 `znode`。
   - `exists(path, watch)`: 如果 `path` 下的 `znode` 存在，返回 true, 否则返回 false.`watch` 标志可以使 client 在 `znode` 上设置 watch。
   - `getData(path, watch)`: 返回 `znode` 的数据和 znode 相关的元数据（例如版本信息）。`watch` 和 `exists()` 里面的作用一样，不同之处在于，如果`znode`不存在，则 ZooKeeper 不会设置`watch`。
   - `setData(path, data, version)`: 如果 `version` 是 `znode` 现有的版本，把 `data[]` 写进 `znode`.
   - `getChildren(path, watch)`: 返回 `path` 对应的 `znode` 的子节点集合。
   - `sync(path)`: 等待操作开始时所有没有同步的更新传播到 client 连接到的服务器。 该 `path` 当前被忽略。

   API 中的所有的方法都有一个同步版本和一个异步版本。执行单个 ZooKeeper 操作且没有要并发执行的任务时，使用同步API。并行执行多个未完成的 ZooKeeper 操作和其他任务时使用异步API。

2. ZooKeeper 有两个基本的顺序保证：
   - **线性写入**（**Linearizable Writes**）: 所有更新 ZooKeeper 状态的请求都会形成一个全局一致的提交顺序，并满足线性一致性。
   - **FIFO的客户端顺序**（**FIFO client order**）: 给定客户端发送的所有请求都按照客户端发送顺序有序执行。

   > ZooKeeper中的linearizability是 _A-linearizability_ (asynchronize linearizability，异步线性)。ZooKeeper在一个客户端有多个未完成的操作的情况下保证 FIFO 顺序。

   一个典型的例子：ZooKeeper 的Leader 选举流程。当一个新的 Leader 接管整个系统时，它必须更新大量的配置参数，并在更新结束的时候通知其它进程。

   leader 靠以下流程更新配置：
   - 删除 `ready znode`
   - 更新各种配置的 `znode`
   - 重新创建 `ready` 来完成其他进程需求

   这些变更可以被流水线处理，并发起一个异步请求来快速更新配置状态。为了将这些请求的延迟降到最低，ZooKeeper 选择异步发出请求。由于顺序保证，如果进程看到 ready `znode`，则它还必须看到新的 Leader 所做的所有配置更改。 如果新的 Leader 在创建完 ready `znode` 之前宕机，则其他进程知道该配置尚未完成，因此不使用它。也就是说新的Leader 在配置更新完成之前其他进程不会更新配置。

   这里需要解决一个问题如何让一个进程在leader变更之前知道这个znode是ready，ZooKeeper 的解决方式是提前通知。让client在收到配置变化之前，收到 ready `znode` 变化的通知。还有一个问题就是如何解决两个client同步的问题，ZooKeeper 提供了 `sync` 请求：当 `sync` 请求之后，下一个请求是读，则构成了一个慢速的读取。`sync` 会保证客户端连接的服务器至少同步到 sync 请求开始时 ZooKeeper 已提交的状态，因此后续读可以看到这些更新。

   ZooKeeper 遵循在大部分服务器是活跃的并且可以通信的情况下ZooKeeper 服务是可用的。

3. 原语示例
   ZooKeeper 服务对client API在客户端上实现的强大原语一无所知。一些常见的原语也是无等待的。
   **配置**
   ZooKeeper 可以被用来实现分布式应用中的动态配置。配置被存储在 `znode` `Zc` 中， 进程以 `Zc` 的完整路径名启动。启动进程通过读取 `Zc` 并将 watch 标志设置为 true 来获取其配置。如果`Zc`中的配置出现任何更新，进程会收到通知并读取新配置，然后再次将 watch 标志设置为 true。

   在客户端无法提前知道连接 master 需要的 Address 和 Port 等相关信息的情况下，可以通过 client 创建 `rendezvous znode` （即 `Zr`）来解决这个问题。客户端把 `Zr` 的整个路径作为启动参数传给 master 和 worker 进程。当 master 启动时，它会把自己的地址和端口信息填充进 `Zr`.当 workers 启动的时候，它们会以 watch flag 读取 `Zr`。如果 `Zr` 还没有被填充， worker 就会等待 `Zr` 被更新。如果 `Zr` 是一个 `ephemeral` 节点，master 和 worker 进程可以 watch `Zr` 是否被删除，并在 client 终止时自行清理。

   `ephemral` 节点可以用来实现group membership（ephemeral node 的生命周期与 Session 绑定）。用一个 `znode` `Zg`来代表这个 group 。当一个group的成员启动后，它在 `Zg` 下创建一个临时节点。如果每个进程都有一个唯一的名称或者标识符，那么这个名称就会被用作创建的子 `znode` 的名称; 否则，他会用 `SEQUENTIAL` flag 来创建一个有唯一名称的 `znode`。进程可以将进程信息放入子`znode`的数据中，比如该进程使用的地址和端口。

   ZooKeeper不是锁服务，但可以用来实现锁。最简单的锁实现使用“lock files”。该锁由 `znode` 表示。为了获取锁，客户端尝试使用`EPHEMERAL`标志创建指定的`znode`。如果创建成功，则客户端将持有该锁。否则，客户端可以设置 watch 标志读取`znode`，以便在当前领导者宕机或显式删除`znode`以释放锁时，得到通知。其他等待锁定的客户端一旦观察到`znode`被删除，就会再次尝试获取锁(即创建 `znode` )。这个锁存在一些问题。具有惊群效应，仅实现互斥锁（没有实现读写锁等模式）。下面列出解决方案
   1. 惊群效应：client 尝试获取锁，根据请求的顺序获得一个序列号。序号小的持有该锁，否则持有锁的 `znode`，将在此客户端的 `znode` 之前获得锁的 `znode` 。通过仅查看客户端`znode`之前的`znode`，通过仅在释放锁或放弃锁请求时才唤醒一个进程，避免了惊群效应。
   2. 读/写锁：写锁仅在命名上有所不同。只有较早的写锁`znode`会阻止客户端获得读锁。多个客户端等待读锁时，当删除具有较低序号的`write-` znode时，我们可能会收到 “惊群效应”。这是符合预期的。

   **双重屏障**

   双重屏障使 client 能够同步计算的开始和结束。在 ZooKeeper 中用`znode` 表示一个屏障，每个进程都会在进入时通过将`znode`创建为屏障`znode`的子节点来向屏障`znode`注册，并在准备离开时注销，即删除该子节点。当屏障`znode`的子`znode`数量超过障碍阈值时，进程可以进入屏障。 当所有进程都删除了其子进程时，进程可以离开屏障。 watch用来监视进程进入和退出屏障。

> 双重屏障实现的是多个参与者的阶段一致性（Phase Consistency），而不是数据一致性。

#### 实现

ZooKeeper通过在组成服务的每台服务器上复制 ZooKeeper 数据来提供高可用性。 复制数据库是一个包含整个数据树的内存数据库。每个数据库中的 `znode` 最多存储 1MB 的数据，这个值在特定情况下是可配置的。为了可恢复性，更新记录会强制在写入内存数据库之前写入磁盘。

每个 ZooKeeper 的服务器都可以为客户端提供服务。客户端连接一台服务器来提交它的请求。写请求会被转发给单独的服务器，该服务器被称为 _leader_ ，其他的 _ZooKeeper_ 服务器被称为 _follower_ ，它们从 leader 接受包含状态变更的提案, 并就状态的更改达成一致。

ZooKeeper集群中消息层是atomic的，用于保证副本不会出现分歧。与客户端的请求不同，事务是幂等的。

所有更新 ZooKeeper 的请求都会被转发给 leader, leader 执行请求，并用原子广播协议 `Zab` 广播 ZooKeeper 的状态变更。从客户端接收请求的服务器在收到对应的状态变更后响应客户端。zab使用服从多数（_2f+1_ 台服务器）的方式决定提案。 Zab 保证 leader 广播的变化按照发送的顺序，并且上一个 leader 的变更会在这个 leader 的变更之前发送。

1. 复制数据库
   每个副本对 ZooKeeper 状态在内存中都有一个拷贝。 当 ZooKeeper 服务器从崩溃中恢复时，它需要恢复该内部状态。ZooKeeper 使用周期定时快照完成这个任务，只需要发送快照之后的状态变更。ZooKeeper 快照称为模糊快照(fuzzy snapshot)。

2. 客户端-服务器交互
   当一个服务器处理写请求的时候，它也会发送通知并清除和这个更新有关的 watch。服务器顺序处理 write，并且以非并行的方式处理读写。这严格保证了通信的顺序。

   > 服务器在本地处理通知。 仅客户端连接到的服务器跟踪并触发该客户端的通知。

   读请求由每个服务器在本地处理，每个读取请求都经过处理并用 zxid 标记，该 zxid 对应于服务器看到的最后一个事务。 该 zxid 定义了读取请求相对于写入请求的偏序。

   > 这种处理方式获得了出色的读取性能，不过使用快速读取的一个缺点是不能保证读取操作的优先顺序。为了保证一个给定的读操作返回最新的值，客户端可以在 `sync` 之后调用读。

   ZooKeeper 按照 FIFO 的顺序处理客户端的请求。返回值包括对应的 `zxid` 。即使是没有请求中间的心跳，服务端也会带上服务端对应的 zxid 给客户端。服务端通过客户端最后一个zxid确保它的 ZooKeeper 数据视图至少和客户端的数据视图一样新。

   为了检测客户端 session 的 failure, ZooKeeper 使用了 timeout。如果在 timeout 时间内，没有其他服务器收到一个 client session 对应的消息，即判定为 failure。如果客户端足够频繁地发送请求，则无需发送任何其他消息。 否则，客户端会在活动不足时发送心跳消息。

### 总结

ZooKeeper是一个分布式协调内核。它通过 znode、watch、session、ephemeral node 等少量基础原语，维护一个线性一致的共享状态空间，并允许应用在此基础上构建配置管理、服务发现、Group Membership、Leader Election、Barrier、分布式锁等各种协调机制。ZooKeeper 将一致性维护与业务协调解耦：底层通过 Zab 保证共享状态的一致性，上层通过 watch 感知状态变化，实现分布式应用之间的协作。
