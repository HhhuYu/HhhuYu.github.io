---
title: Talent plan Tinykv | project3 | Multi-raft KV
categories:
  - project
  - tinykv
tags:
  - tinykv
  - raft
mathjax: false
date: 2022-01-10 00:59:59
---


# project3

## project3a

### 目标

1. 实现leader的转移
2. 实现conf的切换
3. 通过project3a

### 实现细节

1. 在`stepLeader`中实现`MsgTransferLeader`的类型处理调用`handleTransferLeader`函数，在该函数中：首先对重复请求进行提前返回；检查目标节点的可用性；重置选举时间，即希望该节点能在选举时间范围内完成选举，否则认为此次选举失败；如果节点的日志不够新，则将日志同步至最新，如果节点日志足够新，则发送`MsgTimeoutNow`至目标节点，目标节点接收到这个请求，则马上开始选举。`leader`在处理`MsgTransferLeader`时，直到选举超时之前，不能处理新的`Propose`请求。

2. 由`rawnode`的`ProposeConfChange`函数发起`EntryConfChange`类型的`Propose`请求，需要经过raft内达成一致才能启用。其中的`PendingConfIndex`变量用来标识在log中但还未applied的conf，它需要在发起`Propose`时进行记录，同时当前日志的applied index需要小于这个日志的index，否则修改类型为`EntryType_EntryNormal`。同时这里还需要一些简单的函数`addNode`和`deleteNode`。其中，当节点不存在时，是不能变成leader，需要做选举判断。

## project3b

### 目标

1. Leader改变与conf改变
2. 修改Peer
3. region的split

### 实现细节

1. `TransferLeader`只需要在`Propose`的直接处理就好了，不需要进入 Raft Group 达到一致性，也就是说改命令只要直接通过`RawNode`的`TransferLeader`发给Leader就好了，然后返回callback。

2. 对于`ChangePeer`命令的话首先需要在`Propose`中产生`ConfChange`，
  - 相对于普通的`Propose`来说，需要判断当前删除的peer是否是leader, 如果为当前leader则需要通过`transferleader`给region内的任意成员，并直接返回；如果不是，只需要更改最后的`Propose`为`ProposeConfChange`，其他和普通propose相同。
  - `AddNode`：讲peer加入到region的Peers中，同时加入到peerCache中；
  - `RemoveNode`：如果当前的peer就是所删除的peer，可以调用maybedestroy，继续destory，然后返回。如果不是，则需要在region中的peers和PeerCache删除相关数据。
  - 分别处理之后，需要为regionEpoch中的confver变更其版本，将region持久化，同时还需要修改storeMeta中的regions信息。
  - 需要在peer applied snapshot的时候更新其storeMeta中的regions和regionRages
  - 之后再调用`ApplyConfChange`apply 此次conf change
  - 因为存在peer的添加，但是peer的create是通过`maybeCreatePeer`，所以需要通过HeartBeat来触发`peer`的create
  - 返回callback

3. 对于`Split`命令，它的Propose和非admin的相同，主要是判断一下SplitKey是否存在当前的region中。它也需要所有的raft达成一致后，在进行处理。它需要将当前的region split掉，维持两个region，一个老region，startKey不变，endKey变成splitKey。一个新region，startKey为splitKey，endKey为原老region的endKey。同时，大量涉及storeMeta的修改：
  - 如果原region内的peer数量和newPeerIds数量不匹配，则直接返回StaleCommand
  - storeMeta删除原有的region和regionRanges相关信息
  - 构建一个新region：RegionEpoch初始化为(InitEpochConfVer,InitEpochVer), 本身split仅有newPeerId，所以需要和store进行绑定，需要绑定对应store上的db store id
  - 创建新peer时需要注意，本region的peer可能已经由其他store send的消息激活创建，所以需要判断peer是否存在。若不存在，通过`CreatePeer`创建新peer,同时注册到router中，初始化Peer
  - 更新老region的EndKey，RegionEpoch的Version
  - 将Storemeta中的regions与regionRanges进行更新
  - 持久化region信息
  - 返回callback

## project3c

### 目标

1. 通过heartbeat的信息，更新scheduler中的状态
2. 实现region balance scheduler

### 细节

1. 对于能够在cluster找到的region，先查看收到的region信息中的confver和version是否有落后，如果落后直接返回Stale。不是所有的情况都需要更新region信息，当一下5种情况都满足时可以跳过： confver和version没有变化，leader未变化，pendingpeer为空，ApproximateSize没有变化，peer没有变化。对于不能够找到的region，查询其包含其key范围的region，遍历所有的region，如果region信息中的confver和version有落后，则直接返回stale。其他情况正常更新region和store

2. 先查找需要且合适的store来移动region，需要store 为`up`且`DonwTime`不能超过`MaxStoreDownTime`；然后根据他们的reigon size 从大到小进行排序；按顺序从store中查找合适的region，按顺序从`GetPendingRegionsWithLock`
`GetFollowersWithLock` 和 `GetLeadersWithLock` 若找到一个直接跳查找；检查region当前的状态是否符合移动；然后从所有未在该region中的store中查找最小region size的store；检查两个store region size是否满足移动；生成新peer,创建op
