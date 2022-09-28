# General Idea

## shard controller + groups of kvserver

shard controller: state of shards

groups of kvserver: kvstate + state of shards

![](C:\Users\jiaxi\6.824\notes\lab4B\state_shardkv.png)



## clientOp

key may not be included in the shard of current kvserver Or not yet





## pullConfigOp

sharded kvserver will ask shard controller for new config to update(change) their state

![](C:\Users\jiaxi\6.824\notes\lab4B\changeOfConfig.png)

change state:

(1) which state is kept

(2)  which state is absent and needed from other groups

(3)  which state is useless for local group but needed by other groups

![](C:\Users\jiaxi\6.824\notes\lab4B\state_change.png)



## pullShardOp

after updating state, sharded kvserver will **ask other groups for all absent state**. 

![](C:\Users\jiaxi\6.824\notes\lab4B\pullShard.png)



## garbageCollectionOp

after receiving absent state from other group, sharded kvserver should **eliminate that state and ask other group to eliminate their useless state**

![](C:\Users\jiaxi\6.824\notes\lab4B\garbageCollection.png)





# Overview

## clientOp

processApplyMsgClientOp

1. **check if the shard of key exists in this group**

2. **check if the shard of key has come to this group(not in comeInShards)**

3. check if op == read

4. check if duplicate writes

5. advance state of shardkv

   ```go
   func (kv *ShardKV) ProcessOpReply(op *Op) {
   	// check if shard of key is in this right group
   	if !kv.IsInGroupShard(op.Key) {
   		op.OpType = ErrWrongGroup
   		return
   	}
   	// check if shard and state have come to this group
   	if _, isInComeInShards := kv.comeInShards[key2shard(op.Key)]; isInComeInShards {
   		op.OpType = ErrNoKeyYet
   		return
   	}
   
   	// check if read
   	if op.OpType == GET {
   		op.Value = kv.state[op.Key]
   		return
   	}
   	// check if duplicate
   	maxSeqId, found := kv.maxSeqIds[op.CId]
   	if found && op.SeqId <= maxSeqId {
   		return
   	}
   	// write
   	switch op.OpType {
   	case PUT:
   		kv.state[op.Key] = op.Value
   	case APPEND:
   		kv.state[op.Key] += op.Value
   	}
   	kv.maxSeqIds[op.CId] = op.SeqId
    	//update lastStateIndex   
       kv.lastStateIndex = index
   }
   ```



## pullConfigOp

| No   | step    | location                   | operation                                                    | concurrency    |
| ---- | ------- | -------------------------- | ------------------------------------------------------------ | -------------- |
| 0    | prepare | shardkv(leader)            | query shard controller for new config if comeInShards == 0   | in parallelism |
| 1    | send    | shardkv(leader)            | stub new config into raft (no client => do not wait)         | in parallelism |
| 2    | receive | shardkv(leader + follower) | receive applyMsg of change from new config and change the state | exclusive      |

how to change the state of shardkv?

1. iterate old shards, if not in newShards, move it to comeOutShards
2. iterate new shards, if not in old Shards, set it as comeInShards
3. move out state whose shards are in comeOutShards

```go
func (kv *ShardKV) ProcessPullConfigApplyMsg(applyMsg raft.ApplyMsg) {
	// update comeInShards, comeOutShards, config, state[comeInShards]
	// check if new config is duplicated
	newConfig := applyMsg.Command.(shardctrler.Config)
	oldConfig := kv.config
	if newConfig.Num <= oldConfig.Num {
		return
	}

	if oldConfig.Num == 0 {
		kv.shards = kv.GetNewShards(newConfig)
		kv.config = newConfig
		return
	}

	newShards := kv.GetNewShards(newConfig)
	shards := CopyShards(kv.shards)
	comeOutShards := make(map[int]bool)
	comeInShards := make(map[int]bool)
	// comeOutShards: remove shards not in newConfig and
	// iterate shards in old config
	for shard, _ := range kv.shards {
		if _, found := newShards[shard]; !found {
			// if shard cannot be found in new config, set it as comeOutShards
			delete(shards, shard)
			comeOutShards[shard] = true
		}
		// if shard can be found in new config, do nothing
	}

	// comeInShards: add shards not in current config but in new config
	// iterate shards in new config
	for shard, _ := range newShards {
		if _, found := kv.shards[shard]; !found {
			// if new shard cannot be found in old config, set it as comeInShards
			comeInShards[shard] = true
		}
	}

	// remove kv state in outShards
	kvState := CopyState(kv.state)
	comeOutShard2state := make(map[int]map[string]string)
	// iterate each k, v in kv.state
	for k, v := range kvState {
		shard := key2shard(k)
		if _, ok := comeOutShards[shard]; ok {
			// if shard of k is in comeOutState, move it from kvstate to comeOutState
			comeOutShard := shard
			if _, ok := comeOutShard2state[comeOutShard]; !ok {
				comeOutShard2state[comeOutShard] = make(map[string]string)
			}
			delete(kvState, k)
			comeOutShard2state[comeOutShard][k] = v
		}
	}
	// atomic
	kv.shards = shards
	kv.state = kvState
	kv.comeInShards = comeInShards
	kv.comeOutShards2state[oldConfig.Num] = comeOutShard2state
	kv.config = newConfig
}
```





## pullShardOp

### pullShardRPC

| No   | step    | location                 | operation                                                    | concurrency    |
| ---- | ------- | ------------------------ | ------------------------------------------------------------ | -------------- |
| 0    | prepare | shardkv(leader)-sender   | get comeInShards and configNum of comeInShards if comeInShards > 0, set them as RPC args | exclusive      |
| 1    | send    | shardkv(leader)-sender   | for each comeInShard, broadcast RPC args to all servers of group in which comeInShard are | in parallelism |
| 2    | receive | shardkv(leader)-receiver | receive RPC args, load shard and state from comeOutState and reply to sender | exclusive      |

### pullShardOp

| No   | step    | location                          | operation                                                    | concurrency    |
| ---- | ------- | --------------------------------- | ------------------------------------------------------------ | -------------- |
| 0    | prepare | shardkv(leader)-sender            | receive pullShard reply (comeInShard and its state) from other group | parallelism    |
| 1    | send    | shardkv(leader)-sender            | stub reply (comeInShard and its state) into raft             | in parallelism |
| 2    | receive | shardkv(leader + follower)-sender | receive applyMsg of pullShardReply, add the comeInShard state, set comeInShard as garbage shard | exclusive      |





## garbageCollectionOp

### garbageCollectionRPC

| No   | step    | location                 | operation                                                    | concurrency    |
| ---- | ------- | ------------------------ | ------------------------------------------------------------ | -------------- |
| 0    | prepare | shardkv(leader)-sender   | get garbage shards and configNum of garbageShards if garbageShards > 0, set them as RPC args | exclusive      |
| 1    | send    | shardkv(leader)-sender   | for each garbageShard(comeInShard), broadcast RPC args to all servers of group in which the garbageShard is | in parallelism |
| 2    | receive | shardkv(leader)-receiver | receive RPC args, stub the change into raft and reply to sender | exclusive      |

### garbageCollectionOp-Receiver

| No   | step    | location                            | operation                                                    | concurrency    |
| ---- | ------- | ----------------------------------- | ------------------------------------------------------------ | -------------- |
| 0    | prepare | shardkv(leader)-receiver            | receive garbageCollection args (garbageShard) from other group, update reply | in parallelism |
| 1    | send    | shardkv(leader)-receiver            | stub reply (garbageShard) into raft                          | in parallelism |
| 2    | receive | shardkv(leader + follower)-receiver | receive applyMsg of pullShardReply, delete the garbageShard(comeInShard) from comeOutState | exclusive      |

### garbageCollectionOp-Sender

| No   | step    | location                          | operation                                                    | concurrency    |
| ---- | ------- | --------------------------------- | ------------------------------------------------------------ | -------------- |
| 0    | prepare | shardkv(leader)-sender            | receive garbageCollection reply (garbageShard) from other group | in parallelism |
| 1    | send    | shardkv(leader)-sender            | stub reply (garbageShard) into raft                          | in parallelism |
| 2    | receive | shardkv(leader + follower)-sender | receive applyMsg of pullShardReply, delete the garbageShard(comeInShard) from garbageShards | exclusive      |



