# Shard Controller

## general idea

It is very clear that shard controller is exactly the same as kvserver

![](C:\Users\jiaxi\6.824\notes\lab4A\read.png)

![](C:\Users\jiaxi\6.824\notes\lab4A\write.png)



The only different is that shard controller has different state and we need extra steps to advance state to achieve consistency



## state of shard controller

group: a group of servers(leader, followers) responsible for some shards

shard: each shard belongs to 1 group

![](C:\Users\jiaxi\6.824\notes\lab4A\state.png)

# Shard Controller Operations

## types of operations

| op                | type  |
| ----------------- | ----- |
| Move, Join, Leave | write |
| Query             | read  |



## Move

move 1 shard from group A to group B

1. newConfig <= lastConfig
2. shards[shard_id] = groupB (previously shards[shard_id] = groupA)
3. append newConfig to sc.configs

### Do we need extra steps to keep consistency?

No. Because each applyMsg consists 1 move operation, which is same as Put/Append.



## Join

add a list of groups.

Each Join operation will add a list of groups, we need to consider how to keep consistency

![](C:\Users\jiaxi\6.824\notes\lab4A\joinState.png)

1. Get sorted group_ids of groups to add

2. iterate each new group_id from small to big

   1. add new group

   2. assign avg# of shards to new group, 

      each shard comes from the group with max shards and min gid



## Leave

delete a list of groups

Each Join operation will delete a list of groups, we need to consider how to keep consistency

![](C:\Users\jiaxi\6.824\notes\lab4A\leaveState.png)

1. Get sorted group_ids of groups to delete

2. iterate each new group_id from small to big

   1. delete the group

   2. for each shard of  group to delete

      assign shard to the group with min shards and min gid



## Query

read the config of configNum from configs, if Num does not exist, return the last config