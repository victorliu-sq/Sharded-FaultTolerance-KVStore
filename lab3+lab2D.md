# General Idea

## ClientOp

ClientOp is the change client makes to kvserver. All operations have to go long raft to be replicated before we apply them to state of kvserver.

**why we cannot apply client operations directly but send them through log and apply?**

(1) majority of group should apply client operations together

![](C:\Users\jiaxi\6.824\notes\lab2Dlab3_Figures\applyMsg.png)

(2) client operations should be applied in same order in majority of group

![](C:\Users\jiaxi\6.824\notes\lab2Dlab3_Figures\applyMsgSort.png)

**why all reads have to go through leader and logs?**

(1) if not go through leader --> stale data

(2) if not go through logs --> we cannot observe latest read 

​	eg: writeX1, writeX2, readX --> readX returns 1 when writeX2 is not applied



## Snapshot

Snapshot is an Operation of special type, all servers can change state of raft as long as they satisfy the condition

new logStartIndex = index of last log  that has been applied to state of kvserver

![](C:\Users\jiaxi\6.824\notes\lab2Dlab3_Figures\snapshot.png)

## InstallSnapshot

if nextIndex of 1 follower is stable, we have to make logStartIndex same by InstallSnapshot

![](C:\Users\jiaxi\6.824\notes\lab2Dlab3_Figures\installSnapshot.png)



# Template

## RPC

| No   | location | step                                                         | concurrency                  |
| ---- | -------- | ------------------------------------------------------------ | ---------------------------- |
| 1    | sender   | prepare args from state / get args directly                  | exclusive if state is needed |
| 2    | sender   | send RPC args                                                | in parallelism               |
| 3    | receiver | receive RPC args, advance state and reply, and send back reply | exclusive if state is needed |
| 4    | receiver | receive RPC reply, advance state                             | exclusive if state is needed |



## Op(change state)

| No   | location                   | step                                                         | concurrency                  |
| ---- | -------------------------- | ------------------------------------------------------------ | ---------------------------- |
| 1    | kvserver                   | prepare change(op) from state OR get args directly           | exclusive if state is needed |
| 2    | kvserver                   | stub(send) the change into raft to make it replicated in group | in parallelism               |
| 3    | applier(leader + follower) | receive applyMsg from applyCh and advance state (if leader, send op back to kvserver) | exclusive if state is needed |
| 4    | kvserver                   | receive op (reply to client if needed)                       | exclusive if state is needed |



# Overview

## Clerk RPC

Only leader receives clientOp **(change)** from client, stubs clientOp into logs of raft, and replies to client.

Both leader and follower will receive applyMsg formed by clientOp and apply clientOp to state of kvserver.

![](C:\Users\jiaxi\6.824\notes\lab2Dlab3_Figures\applyMsg.png)

### process

| No   | step    | operation                                                    | synchronization |
| ---- | ------- | ------------------------------------------------------------ | --------------- |
| 1    | prepare | args <- Clerk                                                | exclusive       |
| 2    | send    | ok := ck.servers[ck.leaderId].Call("KVServer.Get", &args, &reply) | parallel        |
| 3    | receive | if reply.Err == retry, update leaderId and try resend        | exclusive       |

| unordered problem        | method        |
| ------------------------ | ------------- |
| leader + 1 is not sorted | not a problem |



### code

Get

```go
func (ck *Clerk) Get(key string) string {
	// 1. prepare clientOp RPC
	ck.mu.Lock()
	args := ck.newGetArgs(key)
	reply := ck.newGetReply()
	ck.mu.Unlock()
	// 2. while not success, send clientOp RPC
	for {
		ok := ck.servers[ck.leaderId].Call("KVServer.Get", &args, &reply)
		// 3. receive clientOp RPC and advance state of client
		if reply.Err == Retry {
			// (1) fail to send (2) not leader (3) time out --> resend
			ck.mu.Lock()
			ck.leaderId = (ck.leaderId + 1) % len(ck.servers)
			ck.mu.Unlock()
			time.Sleep(OpTime)
		} else if ok && reply.Err == OK {
			// (2) send successfully, return value
			return reply.Value
		}
	}
}
```

Put, Append

```go
func (ck *Clerk) PutAppend(key string, value string, op string) {
	// 1. prepare clientOp RPC
	ck.mu.Lock()
	// write Op needs to update seqId
	ck.seqId++
	args := ck.newPutAppendArgs(key, value, op, ck.seqId)
	reply := ck.newPutAppendReply()
	ck.mu.Unlock()
	// 2. while not success, send clientOp RPC
	for {
		ok := ck.servers[ck.leaderId].Call("KVServer.PutAppend", &args, &reply)
		// 3. receive clientOp RPC and advance state of client
		if reply.Err == Retry {
			// (1) fail to send (2) not leader (3) time out --> resend
			ck.mu.Lock()
			ck.leaderId = (ck.leaderId + 1) % len(ck.servers)
			ck.mu.Unlock()
			time.Sleep(OpTime)
		} else if ok && reply.Err == OK {
			// (2) send successfully, return
			return
		}
	}
}
```





## KVServer clientOp

### process

| Num  | state             | operations                                                   | concurrency |
| ---- | ----------------- | ------------------------------------------------------------ | ----------- |
| 1    | leader            | prepare and stub clientOp                                    | parallel    |
| 2    | leader            | wait for clientOp to reply client                            | parallel    |
| 3    | leader + follower | receive applyMsg of clientOp, advance state and send back Op if leader | exclusive   |
| 4    | leader            | receive replicated clientOp from applier to reply client     | parallel    |

### code

#### step124

```go
func (kv *KVServer) Get(args *GetArgs, reply *GetReply) {
	// prepare
	op := kv.NewGetOp(*args)
	// send
	index, _, isLeader := kv.rf.Start(op) // index, term, isLeader
	if !isLeader {
		return
	}
	kv.mu.Lock()
	opChan := kv.GetOpChan(index)
	kv.mu.Unlock()
	// wait for Op from Opchan of [index]
	// applier will apply Op to state one by one and value of key has been stored in newOp
	newOp := kv.WaitForOp(opChan)
	if IsOpEqual(op, newOp) {
		reply.Value = newOp.Value
		reply.Err = OK
	}
}
```



PutAppend

```go
func (kv *KVServer) PutAppend(args *PutAppendArgs, reply *PutAppendReply) {
	// 1. Prepare
	op := kv.NewPutAppendOp(*args)
	// 2. Send and get (index, term, isLeader)
	index, _, isLeader := kv.rf.Start(op)
	if !isLeader {
		// 2.(1) if not leader, return immediately
		return
	}
	// 2.(2) if leader, waits for op to be replicated and reply to client
	kv.mu.Lock()
	opChan := kv.GetOpChan(index)
	kv.mu.Unlock()
	newOp := kv.WaitForOp(opChan)
	// 4. after op has been replicated, reply to client
	if IsOpEqual(op, newOp) {
		reply.Err = OK
	}
}
```



#### step3

```go
// 3. Advance state from applyMsg of client Op
func (kv *KVServer) ProcessCommandMsg(applyMsg raft.ApplyMsg) {
	kv.mu.Lock()
	defer kv.mu.Unlock()
	// 1. Extract command(Op) from applyMsg
	op := applyMsg.Command.(Op)
	index := applyMsg.CommandIndex
	// 2. Advance state of kvserver with Op
	kv.AdvanceState(&op, index)
	// 3. if server is leader, we need to send back op to reply
	if kv.rf.IsLeaderLock() {
		// it is possible that log is applied before OpChan is created in Get / Put
		// so we may create OpChan here
		opChan := kv.GetOpChan(index)
		opChan <- op
	}
}
```

advance state of kvserver

1. if read, save value in op to reply client and return
2. if write, update state, maxSeqId and lastStateIndex



## Snapshot Op

Both leader and follower will run a Daemon to receive signals to snapshot**(change)** from raft

Once servers get signals, they should make changes to logs of raft immediately.

There can be some inconsistency between leader and follower but this is not a problem thanks to InstallSnapshot

![](C:\Users\jiaxi\6.824\notes\lab2Dlab3_Figures\snapshotOverview.png)

### process

all servers (leader + follower) will take snapshot, so there is no need to replicate snapshot msg

| No   | step              | operation                                                    | synchronization |
| ---- | ----------------- | ------------------------------------------------------------ | --------------- |
| 0    | Daemon            | try to snapshot periodically                                 | exclusive       |
| 1    | prepare(kvserver) | check encode state of kvserver into snapshot                 | exclusive       |
| 2    | stub(kvserver)    | go kv.rf.Snapshot(kv.lastStateIndex, data)                   | parallelism     |
| 3    | receive(raft)     | check concurrent snapshots, cutStart and save data in rf.persister | exclusive       |

### code

#### code-012

```go
// all servers will take SnapshotOp periodically
// 0. Daemon to take snapshot
func (kv *KVServer) snapshotDaemon() {
	for !kv.Killed() {
		kv.TrySnapshot()
		time.Sleep(30 * time.Millisecond)
	}
}

func (kv *KVServer) TrySnapshot() {
	kv.mu.Lock()
	defer kv.mu.Unlock()
	if kv.lastStateIndex == 0 || kv.maxraftstate == -1 {
		return
	}
	// 1. Prepare: check condition of snapshot and encode state of kvserver into snapshot
	if kv.persister.RaftStateSize() >= kv.maxraftstate {
		DPrintf("[kv %v] snapshot the state, raft state size is %v", kv.me, kv.persister.RaftStateSize())
		data := kv.EncodeSnapshot()
		// 2. Send: stub snapshot(change) to raft
		go kv.rf.Snapshot(kv.lastStateIndex, data)
	}
}
```

#### code-3

```go
// 3. Receive snapshot of kvserver
func (rf *Raft) Snapshot(lastStateIndex int, snapshot []byte) {
	rf.mu.Lock()
	defer rf.mu.Unlock()
	// there can be concurrent snapshot RPC msgs
	if lastStateIndex <= rf.logStartIndex {
		return
	}
	// delete log entries from start to index - 1, rf.start = index
	rf.CutStart(lastStateIndex)
	rf.persistStateAndSnapshot(snapshot)
}
```



## InstallSnapshot RPC

leader's raft initializes the snapshot and sends it to followers. Only followers send snapshot applyMsg and install snapshot

![](C:\Users\jiaxi\6.824\notes\lab2Dlab3_Figures\installSnapshotOverview.png)

| No   | location      | opeartion                                                    | concurrency    |
| ---- | ------------- | ------------------------------------------------------------ | -------------- |
| 1    | raft-leader   | initialize and prepare InstallSnapshot RPC                   | one by one     |
| 2    | raft-leader   | send InstallSnapshot args to follower                        | in parallelism |
| 3    | raft-follower | receive InstallSnapshot args, cut logs and return InstallSnapshot reply and send snapshot applyMsg to applyCh | one by one     |
| 5    | raft-leader   | receive InstallSnapshot reply, update nextIndex and matchIndex of peer | one by one     |

initiate & broadcast

```go
// Broadcast AppendEntry
func (rf *Raft) BroadcastAppendEntry() {
	for peer := range rf.peers {
		if peer == rf.me {
			continue
		}
		// if nextIndex is outdated, we need to install snapshot
		if rf.GetNextIndex(peer) <= rf.GetFirstIndex() {
			snap_args := rf.NewInstallSnapshotArgs()
			snap_reply := rf.NewInstallSnapshotReply()
			go rf.InstallSnapshotSender(peer, &snap_args, &snap_reply)
		} else {
			AE_args := rf.newAEArgs(peer)
			AE_reply := rf.newAEReply()
			go rf.AppendEntrySender(peer, &AE_args, &AE_reply)
		}
	}
}
```

sender

```go
// *****************************************************************************************
// InstallSnapshotSender
func (rf *Raft) InstallSnapshotSender(peer int, args *InstallSnapshotArgs, reply *InstallSnapshotReply) {
	if !rf.sendInstallSnapshot(peer, args, reply) {
		return
	}
	rf.mu.Lock()
	defer rf.mu.Unlock()
	// cascading reply
	if args.Term != rf.GetTerm() {
		return
	}
	// all server rule
	if reply.Term > args.Term {
		rf.ConvertToFollower(reply.Term)
		return
	}
	// update nextIndex and matchIndex (considering unordered replies)
	if args.StartLogIndex+1 > rf.GetNextIndex(peer) {
		rf.SetNextIndex(peer, args.StartLogIndex+1)
		rf.SetMatchIndex(peer, args.StartLogIndex)
	}

	// send AE msg to followers after installing snapshots
	AE_args := rf.newAEArgs(peer)
	AE_reply := rf.newAEReply()
	go rf.AppendEntrySender(peer, &AE_args, &AE_reply)
}
```

receiver

```go
func (rf *Raft) InstallSnapshot(args *InstallSnapshotArgs, reply *InstallSnapshotReply) {
	rf.mu.Lock()
	defer rf.mu.Unlock()
	// stale leader
	reply.Term = rf.GetTerm()
	if args.Term < rf.GetTerm() {
		return
	}
	// all server rule
	if args.Term > rf.GetTerm() {
		rf.ConvertToFollower(args.Term)
	}

	// There can be unordered snapshots that later snapshot come first, we do not want to cut back
	// if args.StartLogIndex <= rf.GetFirstIndex() {
	// 		return
	// }
	// And we do not want to apply msg again
	if args.StartLogIndex <= rf.lastApplied {
		return
	}

	if rf.IsLogExist(args.StartLogIndex, args.StartLogTerm) {
		// if existing log entry has same index and term as snapshot's last included entry,
		// retain log entry following it and reply
		rf.CutStart(args.StartLogIndex)
	} else {
		// Discard the entire log and add startLog
		rf.DiscardEntireLog(args.StartLogIndex, args.StartLogTerm)
	}
	// save state of raft and snapshot
	rf.persistStateAndSnapshot(args.Data)
	// Apply snapshot
	// 1. Advance commitIndex and lastAppliedIndex
	rf.commitIndex = max(rf.commitIndex, args.StartLogIndex)
	rf.lastApplied = max(rf.lastApplied, args.StartLogIndex)
	// 2.Send snapshot applyMsg to applyCh
	rf.mu.Unlock()
	rf.applyCh <- ApplyMsg{
		CommandValid:  false,
		SnapshotValid: true,
		Snapshot:      args.Data,
		// these two vars cannot be ignored
		SnapshotTerm:  args.StartLogTerm,
		SnapshotIndex: args.StartLogIndex,
	}
	rf.mu.Lock()
}
```



## InstallSnapshot Op

Only followers send snapshot applyMsg and install snapshot

| No   | location                    | opeartion                                                    | concurrency |
| ---- | --------------------------- | ------------------------------------------------------------ | ----------- |
| 1    | kv-server applier(follower) | Daemon to receive snapshot applyMsg from applyCh, process applyMsg(decode snapshot into state of kv-server) | one by one  |

```go
// Advance state from applyMsg of snapshot
func (kv *KVServer) ProcessInstallSnapshotMsg(applyMsg raft.ApplyMsg) {
	kv.mu.Lock()
	defer kv.mu.Unlock()
	kv.DecodeSnapshot(applyMsg.Snapshot)
}

// Decode Snapshot into 2 maps
func (kv *KVServer) DecodeSnapshot(data []byte) {
	if data == nil || len(data) < 1 {
		return
	}
	r := bytes.NewBuffer(data)
	d := labgob.NewDecoder(r)
	var state map[string]string
	var cid2maxSeqId map[int64]int
	var lastStateIndex int
	if d.Decode(&state) != nil || d.Decode(&cid2maxSeqId) != nil || d.Decode(&lastStateIndex) != nil {
		return
	} else {
		kv.state = state
		kv.cid2maxSeqId = cid2maxSeqId
		kv.lastStateIndex = lastStateIndex
	}
}
```



# Debug

##  Unordered Msgs

### Difference between RPC(function) & channel

|                | send           | receive   | sort of sending and receiving(led by parallel sendings) |
| -------------- | -------------- | --------- | ------------------------------------------------------- |
| function / RPC | in parallelism | exclusive | different                                               |
| channel        | one by one     | exclusive | same                                                    |

if we can **prepare**, **receive** RPC w/o  synchronization if all variables we need are read-only

#### why do we need to prepare / receive one by one

because we need to read from / write into state from same updates

#### what's meaning of sort of sending and receiving

if sort of sending is different from receiving, we need to consider how order influences result of processing





## KVServer clientOp

### why do we need to wait for another op after stubbing it into raft

| operation                                   | meaning                                         |
| ------------------------------------------- | ----------------------------------------------- |
| stub op into raft and get its index in logs | op is stored in logs of leader successfully     |
| get op from opChan                          | op is stored in majority of servers succesfully |



### how to wait for applied Op from OpChan

| situation                         | operation       |
| --------------------------------- | --------------- |
| Opchan waits op for too long time | return empty Op |
| Opchan receives op in time        | return op       |

```go
func (kv *KVServer) WaitForOp(opChan chan Op) Op {
	select {
	case op := <-opChan:
		return op
	case <-time.After(TimeOutDuration):
		return Op{}
	}
}
```



### how to receives applied Op from OpChan

| situation                                   | opertaion                                                |
| ------------------------------------------- | -------------------------------------------------------- |
| kv-server gets an empty op                  | if op != new Op, tell client to retry                    |
| kv-server gets a op == op stubbed into raft | if op == newOp, load value in op for Get, reply.Err = OK |

Notice that reply.OK should be updated at last to achieve atomic

```go
// Get
if IsOpEqual(op, newOp) {
	reply.Value = newOp.Value
	reply.Err = OK
}
```



### how to advance state of kv-server

1. check if opType == read

2. check duplicate with maxSeqId

3. advance state of kvserver

4. update maxSeqId

   ```go
   func (kv *KVServer) AdvanceStateClientOp(op *Op, index int) {
   	// if read, save value in op to reply client and return
   	if op.OpType == GET {
   		op.Value = kv.state[op.Key]
   		return
   	}
   	// check duplicate writes, if duplicate, do nothing
   	maxSeqId, found := kv.cid2maxSeqId[op.CId]
   	if found && op.SeqId <= maxSeqId {
   		return
   	}
   	// if write, update state, maxSeqId and lastStateIndex
   	switch op.OpType {
   	case PUT:
   		kv.state[op.Key] = op.Value
   	case APPEND:
   		kv.state[op.Key] += op.Value
   	}
   	// update max seqId at last
   	kv.cid2maxSeqId[op.CId] = op.SeqId
   	// since applyCh sends msg from left to right, index must be larger than last index
   	kv.lastStateIndex = index
   }
   ```

   #### why we need to update maxSeqId at last

   atomic

   #### why we need to update lastStateIndex

   if state exceeds limit, we need lastStateIndex to cut logs in raft (with snapshot)

   #### why we need to create an opChan here rather than after we stub it into raft

   it is possible that log has been replicated and sent to applyCh just after stubbing.

   [replicated in raft, sent to applyCh, receive from applyCh, create opChan] faster than [stub into raft, create opChan]



## Snapshot Op

| unordered problem                           | method                                                       |
| ------------------------------------------- | ------------------------------------------------------------ |
| raft cuts ealier start or save earlier data | if args.lastStateIndex <= rf.startLogIndex, return immediately |

### how to deal with unordered msg and abandon earlier msgs

if args.lastStateIndex < rf.logStartIndex, it means that this is an earlier snapshot and we should ignore it

![](C:\Users\jiaxi\6.824\notes\lab2Dlab3_Figures\unordered_snapshot.png)

### how to cut start

kvserver advances state then update lastStateIndex with command

![](C:\Users\jiaxi\6.824\notes\lab2Dlab3_Figures\cutStart.png) 

### how to persist

save nv-state of raft along with data of kvserser into persister



## InstallSnapshot RPC

| unordered problem                                 | method                                                       |
| ------------------------------------------------- | ------------------------------------------------------------ |
| raft cuts applied logs and we need to apply again | if args.lastStateIndex <= rf.startLogIndex, return immediately |
| leader gets smaller nextIndex                     | nextIndex & matchIndex can only be updated bigger            |

### when to installsnapshot

during broadcasting AE

| situation                              | meaning                                                   | operation       |
| -------------------------------------- | --------------------------------------------------------- | --------------- |
| next[peer] >= leader.logStartIndex + 1 | log to replicate in follower is still contained in leader | AE              |
| next[peer] < leader.logStartIndex + 1  | log to replicate in follower is NOT contained in leader   | installsnapshot |



### how to process installsnapshot args

#### how to deal with rule for all server

| situation          | operation           |
| ------------------ | ------------------- |
| sender is stale    | return immediately  |
| receiver  is stale | convert to follower |

#### how to deal with unordered args

| situation                                | meaning                                   | operation          |
| ---------------------------------------- | ----------------------------------------- | ------------------ |
| if args.lastStateIndex <= rf.lastApplied | if we cut the log, we need to apply again | return immediately |
| if args.lastStateIndex > rf.lastApplied  | new state includes logs not applied yet   | cut log of raft    |

![](C:\Users\jiaxi\6.824\notes\lab2Dlab3_Figures\unordered_installsnapshot.png)

#### how to update state of raft

| state                       | operation                                  |
| --------------------------- | ------------------------------------------ |
| logStartIndex, logStartTerm | try to cutStart / abandon the whole log    |
| commitIndex, lastApplied    | update if cutStart / abandon the whole log |
| rf.persister                | update with args.snapshot given by leader  |
| rf.applyCh                  | send a snapshot Msg                        |



### (raft-leader)how to receive reply from follower

#### how to deal with rule for all server

| situation       | operatoin                              |
| --------------- | -------------------------------------- |
| cascading reply | return immediately                     |
| stale leader    | convert to follower return immediately |

#### how to deal with unordered replies

| state                                                        | operation                       |
| ------------------------------------------------------------ | ------------------------------- |
| args.StartLogIndex + 1 <= nextIndex[peer] or args.StartLogIndex <= matchIndex[peer] | ignore                          |
| args.StartLogIndex + 1 > nextIndex[peer] or args.StartLogIndex > matchIndex[peer] | update nextIndex and matchIndex |

#### how to update state of raft(leader)

| state            | operation                          |
| ---------------- | ---------------------------------- |
| nextIndex[peer]  | update with args.StartLogIndex + 1 |
| matchIndex[peer] | update with args.StartLogIndex     |





