# Raft

## Replicated State Machine

![Overview](./images/replicated_state_machine.svg)

- [1] Client 发送访问 State Machine 的指令，如查询、新增、修改、删除操作；
- [2] 一致性模块记录并保证集群内其他 Server 日志内容一致；
- [3] 每个 Server 采用一致的操作修改 State Machine；
- [4] 结果返回给 Client。

## 简介

### 设计目的

Paxos 一致性算法不易理解，不易实现。Raft 目的是设计一个容易实现且不违反直觉的一致性算法。Raft 算法主要特征有：

- 更强的 Leader: 日志分发只能从 Leader 到其他 Servers
- Leader Election: 使用随机时长的定时器选举 Leader
- 成员变更(Membership changes): 集群成员变更时（新增节点或移除节点）时，采用 new joint consensus 方法，保证原集群正常服务

下图为 Raft 节点状态转换图：

![Raft Server Status](./images/server_status.svg)

Term 示例图：

![Term](./images/term.svg)

### Raft 基本步骤

- 选择一个 Leader
- Leader 接收从 Client 发出的 Log Entry，并分发给其他 Server，并通知其他 Server 合适可将 Log Entries 应用的状态
- 当 Leader 崩溃或不可达时，重新选举

## 算法结构

### 基础规则

#### State

全部服务器需要持久化的状态（需要在响应 RPC 前，记录在可靠的存储上）：

- **currentTerm**: Server 看到的最近的 Term（首次启动时为 0，单调递增）
- **voteFor**: 当前 Term 的 candidateId，如果没有，为 null
- **log[]**: Log Entries。包括访问、修改状态机的指令；及服务器收到 Entry 时的 Term。

全部服务器的非持久状态：

- **commitIndex**: 服务器已知的已提交的 Log Entries 的最大编号（初始为 0，单调递增）
- **lastApplied**: 服务器应用到 State Machine 的 Log Entries 的最大编号（初始为 0，单调递增）

Leader 非持久状态（每次选举后，重新初始化）：

- **nextIndex[]**: 记录每一个 Server 发送至 Leader 的下一个 Log Entry 的编号（初始化为 Leader 的最后一个 Log Entry 编号加 1）
- **matchIndex**: 记录每一个 Server 已备份的 Log Entry 最大编号（初始为 0，单调递增）

#### Append Entries RPC

本方法应由 Leader 调用以分发 Log Entries，也可用做心跳

参数：

- **term**: Leader 的 Term
- **leaderId**: 非 Leader 服务器可重定向 Client 请求
- **prevLogIndex**: 新 Log Entry 的前一 Log Entry 编号
- **prevLogTerm**: prevLogIndex 的 Term
- **entries[]**: 需要存储的 Log Entries
- **leaderCommit**: Leader 的 commintIndex

应答：

- **term**: 当前 Term，Leader 需要更新
- **success**: 如果 Server 包含 **prevLogIndex** 及 **prevLogTerm** 所确认的 Log Entry，返回 **true**

实现：

- term < currentTerm，返回 false
- 不包含 **prevLogIndex** 及 **prevLogTerm** 所确认的 Log Entry，返回 false
- 如果有冲突的 Log Entry，则删除冲突的 Log Entry 及其后所有的 Log Entries
- 追加全部不存在的 Log Entries
- 如果 leaderCommit > commitIndex，设置 commitIndex 为 min(leaderCommit, index of the last new entry)

流程图如下：

![Append Entry RPC Procedure](./images/append_entry_rpc.svg)

#### Request Vote RPC

由 Candidates 发起，搜集选票

参数：

- **term**: Candidate 当前 Term
- **candidateId**: Candidate ID
- **lastLogIndex**: Candidate 的最后一个 Log Entry 编号
- **lastLogTerm**: Candidate 的最后一个 Log Entry 的 Term

应答：

- **term**: currentTerm，供 Candidate 更新自身
- **voteGranted**: 如果投票给 Candidate，返回 true

实现：

- term < currentTerm，返回 false
- voteFor 为 null 或 **candidateId**，且 Candidate 的 Log Entries 记录不老于 Server，返回 true

流程图如下：

![Request Vote RPC Procedure](./images/request_vote_rpc.svg)

#### 全部 Server 需要遵守的规则

![Rules for All Servers](./images/rules_for_all_servers.svg)

#### Followers 需要遵守的规则

- 响应 Leader 或 Candidates 的 RPC 请求
- 如果 election timer 超时时，没有收到 Leader 的 **AppendEntriesRPC** 请求也没有投票给 Candidates，转换角色为 Candidate

#### Candidates 需要遵守的规则

- 角色转变为 Candidate 后，发起选举
	- 增加 currentTerm
	- 投票给自己
	- 重置 election timer
	- 向其他所有 Server 发起 **RequestVoteRPC**
- 如果收到多数投票，变为 Leader
- 如果收到新 Leader 的 **AppendEntriesRPC** 请求，变为 Follower
- election timer 超时，发起新的选举

#### Leader 需要遵守的规则

- 选举后，发送空 **AppendEntriesRPC**，并在空闲时发送，防止心跳超时
- 收到 Client 请求后，将 Log Entry 追加到本地日志，并在应用至 State Machine 后返回应答
- 如果某个 follower 的 lastLogEntry >= nextIndex，发起 **AppendEntriesRPC**
	- 调用成功：更新 follower 的 nextIndex 及 matchIndex 
	- 调用失败：nextIndex 减一，重试
- 如果存在 N，满足  N > commitIndex，且多数 matchIndex[i] >= N 且 log[N].term == currentTerm，那么设置 **commitIndex = N**

### Leader Election

Raft 使用心跳机制来触发 Leader Election。

![Overview](./images/leader_election_overview.svg)

当一个 Server 启动时，初始为 Follower。Follower 只要定期收到 Leader 或 Candidates 的心跳，就维持 Follower 状态。

Raft 使用随机的 Election Timeout 来保证尽快选择出 Leader，如下图所示：

![Random Election Timeout](./images/random_election_timeout.svg)

### Log Replication

Log 结构如下：

![Log Entries Overview](./images/logs_overview.svg)

Leader 接收 Client 请求后，将 Log Entries 记录至本地 Log 中，然后并发的调用 **AppendEntriesRPC** 来复制 Log Entries；如果发生 Follower 崩溃、网络延迟等因素，Leader 会一直调用 **AppendEntriesRPC**（即使在给了 Client 应答后） 来确保全部 Followers 拥有全部记录。

Leader 决定何时应用 Log Entry 至 State Machine，被应用的 Log Entry 称为 Committed Entry。

Raft 确保全部 Committed Entries 是持久化的，且最终会应用到 State Machine。

Leader 持续跟踪 Committed Entries 的最大编号，并会在 **AppendEntriesRPC** 中包含这个值，这样可以确保其他 Servers 最终都会知道该值。

当日志不一致时，如下图所示：

![Inconsistent Log Entries](./images/inconsistent_log_entries.svg)

Raft 强制使用 Leader 的 Log 覆盖其他 Followers 的 Log。Leader 永远不会覆盖或删除自己的日志记录。

### Safty

#### Election Restriction

在投票阶段，Candidates 调用 **RequestVoteRPC** 时，会附带自己拥有的 **lastLogIndex** 及 **lastLogTerm**，当 Followers 发现自己拥有的日志条目多于 Candidates 时，拒绝投票。

这保证了，新选择的 Leader 拥有全部 Committed Entries。

#### Committing entries from previous terms

![Committing from Previous Terms](./images/committing_from_prev_terms.svg)

为了避免上图的错误情况， Leader 在 Commit 时需要满足：**只允许 Leader 提交包含当前 Term 的日志**

### Cluster Membership Changes

更改配置，可能生成的多 Leader 情况：

![Add Server Problem](./images/add_servers_problem.svg)

可以看出，问题之所以出现，是因为新、旧配置文件有时间段重合的作用时间。

Raft 通过引入 **joint consensus** 配置来解决问题。我们将旧配置文件记录为 C-old，新配置文件记录为 C-new，那么 **joint consensus** 记录为 C-old,new。Leader 接收到配置变更命令后，将 C-old,new 作为一个普通 Log Entry 发布，当 C-old,new 被 Commit 后，系统切换至 C-old,new 配置；然后系统创建 C-new，并在 C-new 被 Commit 后，切换至新配置。如下图：

![Joint Consensus Configuration](./images/joint_consensus_configuration.svg)

当一个 Server 接收到新配置项后（C-old,new C-new），马上使用新配置工作，而不管这个配置项是否已 Committed。
