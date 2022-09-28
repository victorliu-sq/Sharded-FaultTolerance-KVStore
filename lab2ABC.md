# RPC, function and channel

## Similarity between function and RPC

| func / RPC | format of args & reply                      |
| ---------- | ------------------------------------------- |
| funciton:  | reply = func(args)                          |
| RPC:       | sender.call("receiver.func", &args, &reply) |



## General idea of concurrency

|                | prepare    | send           | receive    | sort of sending and receiving |
| -------------- | ---------- | -------------- | ---------- | ----------------------------- |
| function / RPC | one by one | in parallelism | one by one | different                     |
| channel        | one by one | one by one     | one by one | same                          |

if we can **prepare**, **receive** RPC w/o  synchronization if all variables we need are not shared by other threads

### why do we need to prepare / receive one thread by one thread

because we want all variables in 1 thread to read from / write into state from same state(same # of threads)

### what's meaning of sort of sending and receiving

if sort of sending is different from receiving, msgs that we receive are unsorted and we should consider methods to deal with it



# Synchronization

## how to lock in Ticker

| goroutine | operations                                                   |
| --------- | ------------------------------------------------------------ |
| RV        | try to election periodically, we only lock in ElectionTick() |
| AE        | try to send heartbeat msg periodically, we only lock in heartBeartTick() |



## how to lock in goroutine?

go {

​	lock() + defer unlock()

}



## how to lock in rpc?

| RPC      | operations                                                   |
| -------- | ------------------------------------------------------------ |
| sender   | (1) call rpc(receiver, args, reply) (2) lock + defer unlock (3) operations like RV / AE |
| receiver | (1) lock + defer unlock (2) operations like RV / AE          |



# Term problem

## Sender(leader/candidate):

| situation(after sending rpc) | meaning                                                      | operations         |
| ---------------------------- | ------------------------------------------------------------ | ------------------ |
| sender.Term > args.Term      | the reply is cascading(Note that converting back to follower may be led by AE during RV election) | return immediately |
| reply.Term > sender.Term     | current server should convert back to follower and stop all candidiate / leader stuff like RV / AE | return immediately |
| reply.Term <= sender.Term    | receiver's term is same or set to same as sender's           | continue to work   |



## Receiver(follower):

| situation(after receving rpc) | meaning                                               | operations                 |
| ----------------------------- | ----------------------------------------------------- | -------------------------- |
|                               |                                                       | reply.Term = receiver.Term |
| args.Term >= receiver.Term    | receiver is stale and should convert back to follower | continue to work           |
| args.Term < receiver.Term     | sender's term is stale                                | return immediately         |



## Rule for all servers:

After merge existed group with server with bigger term (no matter AE msg or RV msg), all servers will have same bigger term.

All servers in a group for RV(election) or AE(log replication) have the same term. 

![ruleForAllServers](C:\Users\jiaxi\6.824\notes\Lab2ABC_Figures\ruleForAllServers.png)



## why do we need to avoid cascading reply?

there can be split brain

![](C:\Users\jiaxi\6.824\notes\Lab2ABC_Figures\cascadingReply.png)



# State Transfer

## how to convert to follower

if there is a server with bigger term, all of its reachable servers will become followers and advance their terms to this bigger one

| non-valatile state               |
| -------------------------------- |
| rf.state = follower              |
| rf.voteFor = -1                  |
| rf.term becomes new term(bigger) |



## how to convert to leader

| non-valatile state | volatile state                                 |
| ------------------ | ---------------------------------------------- |
| state = leader     | for each peer: nextIndex[peer] = LastIndex + 1 |
|                    | for each peer: matchIndex[peer] = FirstIndex   |



## how to convert to candidiate

| non-valatile state   |
| -------------------- |
| rf.state = candidate |
| rf.voteFor = rf.me   |
| rf.term++            |



# RV-Request Vote:

## RV ticker

### how to try to start election periodically?

while thread is not dead:

​	(1) try to start election

​	(2) sleep for 30 ms



### who can start election? leader? candidate? follower?

| state                | operations after timing out                               |
| -------------------- | --------------------------------------------------------- |
| leader               | reset electionTimer and do not start election             |
| candidate / follower | reset electionTimer + convert to candidate + broadcast RV |

other questions

| questions                 | solution                                                     |
| ------------------------- | ------------------------------------------------------------ |
| how to check timeout      | if current time is later than electionTime, the server is timeout |
| how to reset electionTime | new electionTime = current time +  random time duration between 1000 ms to 1300 ms |
| how to broadcast RV       | for each peer except candidate itself: start a goroutine for RV RPC |



## RV RPC

### RV args

| RV_args                    | function                                                     |
| :------------------------- | :----------------------------------------------------------- |
| Term                       | rule for all servers                                         |
| LastLogTerm + LastLogIndex | only grant for server who has all comitted logs / whose logs are more up-to-date |
| CandidateId                | if granted, vote for candidateId                             |

### RV reply

| RV_reply    | funciton             |
| :---------- | :------------------- |
| Term        | rule for all servers |
| VoteGranted | vote or not          |

### what can be guaranteed by RV rule

| condition           | result                                          |
| ------------------- | ----------------------------------------------- |
| random electionTime | there will be only 1 candidate after timing out |
| voteFor             | there will be only 1 leader after election      |

![](C:\Users\jiaxi\6.824\notes\Lab2ABC_Figures\RV.png)



## RV sender:

### how to deal with Rule for all servers? 

| situation                   | meaning                                              |
| --------------------------- | ---------------------------------------------------- |
| if args.Term > sender.Term  | check RV_reply is cascading or not                   |
| if reply.Term > sender.Term | check whether candidate can convert back to follower |

### how to convert to leader?

for each RV_reply from RV_reveiver:

| situaion               | operations                                     |
| ---------------------- | ---------------------------------------------- |
| if VoteGranted == true | votes += 1                                     |
| if votes >= majority   | convert to leader and broadcast heartBeat Msgs |



#### how to avoid split-brain / how to deal with cascading reply?

| operations                         | meaning                             |
| ---------------------------------- | ----------------------------------- |
| sender checks args.Term != rf.Term | avoid cascading reply               |
| receiver checks if voteFor == -1   | avoid votes for multiple candidates |



## RV receiver:

### how to deal with Rule for all servers?

| situation                    | meaning                                              | opaerations                                     |
| ---------------------------- | ---------------------------------------------------- | ----------------------------------------------- |
|                              |                                                      | reply.Term = receiver.Term                      |
| if args.Term > receiver.Term | follower's term is stale                             | convert to follower and continue to try to vote |
| if args.Term <= rf.Term      | check whether candidate can convert back to follower | sender's term is stale and return immediately   |



### how to vote for candidate?

| situations                                                   | meaning                                                      | operations                                                   |
| ------------------------------------------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| votes != -1                                                  | receiver has granted votes for other candidates              | reply.voteGranted = false                                    |
| checkUpToDate(args.LastLogEntry, receiver.LastLogEntry) == false | sender may not have all committed log entries                | reply.voteGranted = false                                    |
| votes == -1 && checkUpToDate(args.LastLogEntry, receiver.LastLogEntry) == true | receiver has not voted yet && sender's last log entry of candidate is at least up-to-date as receiver | (1) rf.voteFor = args.candidateId (2) reply.VoteGranted = true. (3) valid RV_RPC: reset election timer |

#### how to check up-to-date?

(1) term of last log entry of candidate > that receiver

(2) term of last log entry of candidate == that of  receiver && index of last log entry of candidate >= that of receiver

##### what can be granted by only voting for server with most up-to-date log?

the new leader will only come from servers with all committed logs

![](C:\Users\jiaxi\6.824\notes\Lab2ABC_Figures\upToDate.png)



# AE-Append Entry:

## AE ticker

### when to send heartbeat msg / AE msg separately?

| when                                        | where         |
| :------------------------------------------ | :------------ |
| after a leader stubbing a new log into logs | start()       |
| after a candidate becoming new leader       | RV_Receiver() |



### how to keep linearizability?

| state    | logs                                            |
| :------- | :---------------------------------------------- |
| leader   | read / write logs are both stubbed from start() |
| follower | logs are replicated from leader                 |



### how to send heartbeat msg / AE msg periodically?

-while thread is not dead:

​	-if state is leader: broadcast AE to each server except leader itself

​	

### what will happen due to lots of separate heartbeat msgs / AE msgs? 

|          | problem                                                      | solution                                                     |
| :------- | :----------------------------------------------------------- | :----------------------------------------------------------- |
| sender   | Xindex in replies are in different position                  | (1) do nothing is next == lastIndex + 1 (2) if success, only make nextIndex bigger (3) if unsuccessful, only make nextIndex smaller |
| receiver | entries in args are of variable length; long entries may come first | only append entries absent in logs or conflicting with existing log |



## AE RPC

### AE args

| AE_args             | function                                                     |
| :------------------ | :----------------------------------------------------------- |
| Term                | rule for all server                                          |
| prevIndex, prevTerm | check contain and match                                      |
| Entries[]           | if contain and match, append as many Entries as possible     |
| CommitIndex         | most up-to-date commitIndex of leader used to update commitIndex of followers |

### AE reply

| AE_reply | function                                                     |
| :------- | :----------------------------------------------------------- |
| Term     | rule for all servers                                         |
| Success  | check whether contain and match prevIndex, prevTerm          |
| XValid   | differentiate not contain prevIndex or contain prevIndex but mismatch prevTerm |
| Xindex   | not contain -> first index of empty entry, mismatch ->  first index conflicting term |

### what can be guaranteed by AE rule

![](C:\Users\jiaxi\6.824\notes\Lab2ABC_Figures\AE.png)

### what is relationship between matchIndex, commitIndex and lastApplied

![](C:\Users\jiaxi\6.824\notes\Lab2ABC_Figures\match_commit_apply.png)

###  



## AE sender

### how to deal with Rule for all servers

| situation                       | job                            |
| :------------------------------ | :----------------------------- |
| if reply is cascading           | return immediately             |
| if sender.Term <= receiver.Term | convert to follower and return |
| if sender.Term > receiver.Term  | process reply                  |



### how to prcocess unordered replies to update nextIndex and matchIndex

| situation                                      | meaning                                           | reply.XIndex                                 | new nextIndex                     | new matchIndex                | job                                        |
| :--------------------------------------------- | :------------------------------------------------ | :------------------------------------------- | :-------------------------------- | :---------------------------- | :----------------------------------------- |
| nextIndex == lastIndex + 1                     | all logs haven been replicated into this follower |                                              |                                   |                               | return immediately                         |
| reply.success == true                          | prevIndex contains log and term matches           |                                              | args.prevIndex + len(Entries) + 1 | args.prevIndex + len(Entries) | try to set nextIndex and matchIndex higher |
| reply.success == false && reply.XValid = true  | prevIndex contains log but term mismatches        | first index with mismatched term in follower | reply.XIndex                      |                               | try to set nextIndex lower                 |
| reply.success == false && reply.XValid = false | prevIndex does not contains log                   | first index with empty space(last index + 1) | reply.XIndex                      |                               | try to set nextIndex lower                 |

![](C:\Users\jiaxi\6.824\notes\Lab2ABC_Figures\success_XValid.png)



### how to advance commitIndex in leader

| lower bound of idx             | upper bound of idx | job                                                          |
| :----------------------------- | :----------------- | ------------------------------------------------------------ |
| rf.commitIndex + 1(not enough) | lastIndex          | (1) if matchIndex[peer] >= idx, count += 1 (2) if  count >= majority, update commitIndex and set isAdvance = true (3) if isAdvance = true, applyCond.Signal() |

----------------------FIGURE------------------------------

#### what if commitIndex == 0 on restart

initialize idx as max(logStartIndex, rf.commitIndex + 1)



## AE receiver

### how to deal with Rule for all servers

| situation                  | job                                      |
| :------------------------- | :--------------------------------------- |
| args.Term > receiver.Term  | convert to follower and continue to work |
| args.Term <= receiver.Term | return immediately                       |



### how to deal with multiple unordered AE args for receiver

iterate each entry in args.entries one by one

| situations                                                   | meaning                                        | job                                                          |
| ------------------------------------------------------------ | :--------------------------------------------- | :----------------------------------------------------------- |
| prevLogIndex + 1 + idx of entry > LastIndex                  | empty in corresponding index of receiver       | append all current and following logs to empty position      |
| prevLogIndex + 1 + idx of entry <= LastIndex && entry.Term == logs[correspondingIndex].Term | conflicting in corresponding index of receiver | append all current and following logs to conflicting position |
| prevLogIndex + 1 + idx of entry <= LastIndex && entry.Term != logs[correspondingIndex].Term | matching in corresponding index of receiver    | current log have been replicated, check next log             |

![](C:\Users\jiaxi\6.824\notes\Lab2ABC_Figures\unorderedAE.png)



### how to advance commitIndex in follower

if args.commitIndex > receiver.commitIndex: 

​	advance commitIndex and applyCond.Signal()



## applier

### how to apply committed logs whenever commitIndex is advanced

start a goroutine applier to try to apply most up-to-date log 

| variables                     | function                           |
| :---------------------------- | ---------------------------------- |
| mu                            |                                    |
| applyCond = sync.NewCond(&mu) | wait until commitIndex is advanced |
| applyCh chan ApplyMsg         | send applyMsg to applyCh           |



### how to deal with mu with condition variable and channel

| operations if lastApplied + 1 <= commitIndex | state of mu                                          | operations if lastApplied + 1 > commitIndex | state of mu                                    |
| -------------------------------------------- | ---------------------------------------------------- | ------------------------------------------- | ---------------------------------------------- |
| lastApplied += 1                             | locked                                               | applyCond.Wait()                            | unlocked during waiting, locked after Signal() |
| create applyMsg with log at lastApplied      | locked                                               |                                             |                                                |
| unlock mu                                    | unlocked                                             |                                             |                                                |
| applyCh <- new applyMsg                      | (1) locked during sending (2) unlocked after sending | go to next iteration                        |                                                |
| lock mu                                      | locked                                               |                                             |                                                |
| go to next iteration                         |                                                      |                                             |                                                |

#### what if commitIndex / lastApplied == 0 on restart

for each iteration, set commitIndex to at least logStartIndex