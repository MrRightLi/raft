# Raft Developer Documentation

This documentation provides a high level introduction to the `hashicorp/raft`
implementation. The intended audience is anyone interested in understanding
or contributing to the code.

## Contents

1. [Terminology](#terminology)
2. [Operations](#operations)
   1. [Apply](./apply.md)
3. [Threads](#threads)


## Terminology

术语

This documentation uses the following terms as defined.

* **Cluster（集群）** - the set of peers in the raft configuration
* **Peer** （节点）- a node that participates（参与） in the consensus（共识） protocol using `hashicorp/raft`. A
  peer may be in one of the following states: **follower**, **candidate**, or **leader**.
* **Log** - the full set of log entries.
* **Log Entry** - an entry（条目） in the log. Each entry has an index that is used to order it
  relative to other log entries.
  * **Committed** -  A log entry is considered committed if it is safe for that entry to be
    applied to state machines. A log entry is committed once the leader that created the
    entry has replicated it on a majority of the peers. A peer has successfully
    replicated the entry once it is persisted.
  
    如果日志条目被安全的提交给了状态机，就可以被认为日志被提交了。一旦leader节点创建的条目在主要的节点上被复制了。日志条目一旦被持久化就会被节点成功复制。
  
  * **Applied** - log entry applied to the state machine (FSM)
    提交：日志条目被提交到有限状态机
  * **Term** - raft divides time into terms of arbitrary length. Terms are numbered with
    consecutive integers. Each term begins with an election, in which one or more candidates
    attempt to become leader. If a candidate wins the election, then it serves as leader for
    the rest of the term. If the election ends with a split vote, the term will end with no
    leader.
  
    任期：raft将时间切分成任意多个段。任期就是连续的时间段。每个任期伴随着选举，一个或多个候选人尝试成为优选人。
    如果候选人赢得了选举，那么当前这个节点就会在当前任期成为leader。如果选举选举分票了，当前任期没有节点。
  
* **FSM（有限状态机）** - finite state machine, stores the cluster state 

  有限状态机：存储集群状态

* **Client** - the application that uses the `hashicorp/raft` library
    
    客户端：使用 `hashicorp/raft` 库的应用

## Operations

    操作

### Leader Write
    leader 节点写

Most write operations must be performed on the leader.
大多数写操作都必须在leader节点进行。

* RequestConfigChange - update the raft peer list configuration
  RequestConfigChange - 更新 raft 节点列表配置
* Apply - apply a log entry to the log on a majority of peers, and the FSM. See [raft apply](apply.md) for more details.
  Apply（提交） - 在主要节点，有限状态机上提交 entry 到日志中。
* Barrier - a special Apply that does not modify the FSM, used to wait for previous logs to be applied
  Barrier（界限） - 特殊的提交不会修改有限状态机，用于等待以前的被提交到日志
* LeadershipTransfer - stop accepting client requests, and tell a different peer to start a leadership election
  LeadershipTransfer（leader转换） - 停止客户端请求，告诉其他节点进行选主
* Restore (Snapshot) - overwrite the cluster state with the contents of the snapshot (excluding cluster configuration)
  Restore（重存储）- 通过快照重置集群状态（忽略集群配置） 
* VerifyLeader - send a heartbeat to all voters to confirm the peer is still the leader
  VerifyLeader（主验证）发送心跳给所有的投票节点确认自己是否还是leader

### Follower Write

从节点写

* BootstrapCluster - store the cluster configuration in the local log store
  BootstrapCluster - 存储集群配置到本地存储

### Read

读

Read operations can be performed on a peer in any state.

节点任何状态都可以读

* AppliedIndex - get the index of the last log entry applied to the FSM
  AppliedIndex - 提交最后一个日志条目到状态机
* GetConfiguration - return the latest cluster configuration
  GetConfiguration - 返回集群最新的配置
* LastContact - get the last time this peer made contact with the leader
  LastContact - 获取节点和主节点最新契约
* LastIndex - get the index of the latest stored log entry
  LastIndex - 获取已存储的最新日志条目
* Leader - get the address of the peer that is currently the leader
  Leader - 获取当前节点 leader节点的地址
* Snapshot - snapshot the current state of the FSM into a file
  Snapshot - 将当前的有限状态机的快照存到本地文件
* State - return the state of the peer
  State - 返回当前节点的状态
* Stats - return some stats about the peer and the cluster
  Stats - 返回节点和集群的性能统计信息

## Threads

Raft uses the following threads to handle operations. The name of the thread is in bold,
and a short description of the operation handled by the thread follows. The main thread is
responsible for handling many operations.

Raft 使用以下线程去处理操作。主线程用于处理很多操作。

* **run** (main thread) - different behaviour based on peer state （基于不同的状态有不同的行为）
   * follower
      * processRPC (from rpcCh)
         * AppendEntries 追加日志
         * RequestVote 请求投票
         * InstallSnapshot 安装快照
         * TimeoutNow 超时
      * liveBootstrap (from bootstrapCh) 
      * periodic heartbeatTimer (HeartbeatTimeout)
   * candidate - starts an election for itself when called
      * processRPC (from rpcCh) - same as follower 处理RPC请求
      * acceptVote (from askPeerForVote) 接受选票
   * leader - first starts replication to all peers, and applies a Noop log to ensure the new leader has committed up to the commit index
      * processRPC (from rpcCh) - same as follower, however we don’t actually expect to receive any RPCs other than a RequestVote
      * leadershipTransfer (from leadershipTransferCh) - 
      * commit (from commitCh) -
      * verifyLeader (from verifyCh) -
      * user restore snapshot (from userRestoreCh) -
      * changeConfig (from configurationChangeCh) -
      * dispatchLogs (from applyCh) - handle client Raft.Apply requests by persisting logs to disk, and notifying replication goroutines to replicate the new logs
      * checkLease (periodically LeaseTimeout) -
* **runFSM** - has exclusive access to the FSM, all reads and writes must send a message to this thread. Commands:
   * apply logs to the FSM, from the fsmMutateCh, from processLogs, from leaderLoop (leader) or appendEntries RPC (follower/candidate)
   * restore a snapshot to the FSM, from the fsmMutateCh, from restoreUserSnapshot (leader) or installSnapshot RPC (follower/candidate)
   * capture snapshot, from fsmSnapshotCh, from takeSnapshot (runSnapshot thread)
* **runSnapshot** - handles the slower part of taking a snapshot. From a pointer captured by the FSM.Snapshot operation, this thread persists the snapshot by calling FSMSnapshot.Persist. Also calls compactLogs to delete old logs.
   * periodically (SnapshotInterval) takeSnapshot for log compaction
   * user snapshot, from userSnapshotCh, takeSnapshot to return to the user
* **askPeerForVote (candidate only)** - short lived goroutine that synchronously sends a RequestVote RPC to all voting peers, and waits for the response. One goroutine per voting peer.
* **replicate (leader only)** - long running goroutine that synchronously sends log entry AppendEntry RPCs to all peers. Also starts the heartbeat thread, and possibly the pipelineDecode thread. Runs sendLatestSnapshot when AppendEntry fails.
   * **heartbeat (leader only)** - long running goroutine that synchronously sends heartbeat AppendEntry RPCs to all peers.
   * **pipelineDecode (leader only)**
