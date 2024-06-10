---
title: "Implement Raft"
summary: "6.824 Lab 2: Raft"
date: 2022-01-31T00:25:34+08:00
---

### Scenarios

The character of nodes in raft can be *leader*, *candidate* and *follower*, and RPC calls send by candidate and leader are RequestVote and AppendEntries(and InstallSnapshot) respectively. Due to network partition and delay, any node at any character can receives any rpc request call, so I think the best way to avoid bugs is to list all of the scenarios. Here we go:

| character \ RPC call received | RequestVote (send by candidate)                                                                                                                                                                                                                                                                                                                                                                                   | AppendEntries (send by leader)                                                                                                                                                                                                                                                                                                                                                                            | InstallSnapshot (send by leader)                                                                                                                                                                                                                                                                                                                       |
| ----------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| leader                        | if request term ? current term<br />> : a follower timeout and transitions to candidate, maybe due to network partition or network delay of AppendEntries<br />  action:<br />    1. transition to follower<br />    2. check if the candidate's log is up to date<br />    3. if up to date, grant; otherwise reject.<br /><= : expired RequestVote<br />  action: <br />    reject and send back *current term* | if request term ? current term<br />> : a new leader has been elected maybe due to network partition or network delay<br />  action: <br />    1. transition to follower<br />    2. return false and send back *current term*<br />= : impossible scenario due to $5.2 of paper<br />< : expired rpc call<br />  action: <br />    1. return false and send back *current term*                          | if request term ? current term<br />> : a new leader has been elected maybe due to network partition or network delay<br /> action: <br /> 1. transition to follower<br /> 2. install snapshot<br />3. return *current term*<br />= : impossible scenario due to $5.2 of paper<br />< : expired rpc call<br /> action: <br /> 1. return *current term* |
| candidate                     | (same as leader)                                                                                                                                                                                                                                                                                                                                                                                                  | if request term ? current term<br />>= : maybe restart and connect back to the majority partition<br />  action: <br />    1. transition to follower<br />    2. return false and send back *current term*<br />< : expired rpc call<br />  action: <br />    1. return false and send back *current term*                                                                                                | if request term ? current term<br />>= : maybe restart and connect back to the majority partition<br /> action: <br /> 1. transition to follower<br /> 2. install snapshot<br />3. return *current term*<br />< : expired rpc call<br /> action: <br /> 1. return *current term*<br />                                                                 |
| follower                      | if request term ? current term<br />> : a follower timeout and transitions to candidate, maybe network partition or network delay of AppendEntries<br />  action:<br />    1. check if the candidate's log is up to date<br />    2. if up to date, grant and reset election timer; otherwise reject.<br /><= : expired RequestVote<br />  action: <br />    reject and send back *current term*                  | if request term ? current term<br />> : maybe restart and connect back to the majority partition<br />  action: <br />1. reset election timer<br />2. sync with leader<br />3. return true if sync ok otherwise false and send back *current term*<br />= : (normal)<br />  action: <br />    (same as >)<br />< : expired rpc call<br />  action: <br />    1. return false and send back *current term* | if request term ? current term<br />> : maybe restart and connect back to the majority partition<br /> action: <br />1. reset election timer<br />2. install snapshot<br />3. return *current term*<br />= : (normal)<br /> action: <br /> (same as >)<br />< : expired rpc call<br /> action: <br /> 1. return *current term*<br />                   |

### Confusions

1. what if there are 2 candidates request for vote simultaneously?
   
   According to $5.1 of paper, the leader election only do ONCE per term, if 2 candidates request for vote simultaneously, all requests will fail and wait for next election.

2. what will happen when a partitioned orphan candidate connects back to the majority partition?
   
   A candidate increases its *current term* when starting to elect. So if the candidate is in a network partition and can't get votes from the majority of votes to be leader, it's *current term* will continually increase until it connects back to the majority partition. At the time the candidate connects back, its *current term* may be far more larger than any other nodes, while its log is still out of date, so the candidate cannot be elected as a leader.
   
   At the same time, we know that if any node receives a rpc request or reply with *term* larger than *current term* , it should turn itself to follower and set *current term* to be equal to the received larger *term*.
   
   - the candidate's RequestVote with a larger *term* will be received by other nodes, causing other nodes update their *current term* and turn themself to follower. (now there's no leader in the cluster)
   - because the candidate's log is out of date, none of it's RequestVote should be granted, the candidate remains its character as candidate and do the election again until election timeout. (but its election will fails again due to expired log)
   - the follower whose log is up to date transitions to candidate when its election timer timeout. (Only this one can be elected as leader successfully, any other follower fails to be elected if its log is out of date)
   - the new candidate now has a larger term than the one connected back, so the candidate connected back receives a RequestVote with *term* larger than *current term*, it turn itself to follower. (now the world come back to peace)

3. what if client write to leader and read from a follower before it catch up commit index to the leader?
   
   Well it depends on your needs. 
   
   If you expect the system to be [*linearizable*](https://en.wikipedia.org/wiki/Linearizability), to be short, disallows staled data returnd to client, in which case you can only read from the leader.
   
   If your app is able to loosen the requirement of data instantaneity, you can balance the client requests to followers to improve the read performance.

4. if a leader blocked and connects back after a new leader being elected, there will be 2 leaders in the cluster.
   
   The leader with lower term will transition to follower when it receives an AppendEntries from the new leader with higher term. So technically, **there can be multiple leaders in the cluster, but there can only be one leader of the same term**.

### Note

- [Students' Guide to Raft :: Jon Gjengset](https://thesquareplanet.com/blog/students-guide-to-raft/) provided on [6.824 Lab 2: Raft](https://pdos.csail.mit.edu/6.824/labs/lab-raft.html) has an advice inconsitent with the [paper](https://pdos.csail.mit.edu/6.824/papers/raft-extended.pdf) . The guide tells that you should initialize matchIndex to -1, while the paper and the test cases show that it should be initialized to 0 . [6.824 Lab 2: Raft](https://pdos.csail.mit.edu/6.824/labs/lab-raft.html) also noted that the student's guide is old and keep your eyes open before following any advice.

### Summary

- DO NOT make assumptions on the order or the time an asynchronous function being invoked or a lock being acquired. Even if you are testing your code in LAN.
  
  When I wrote the first version of my bug -- send AppendEntries request, it was like this:

```go
reqAppendEntries := func(rf *Raft, handler func(from int, reply *AppendEntriesReply)) {
    for i := range rf.peers {
        if i == rf.me {
            continue
        }
        go func(serverIdx int, rf *Raft) {
            rf.mu.Lock()
            args := &AppendEntriesArgs{
                Term:         rf.currentTerm,
                LeaderId:     rf.me,
                PrevLogIndex: rf.nextIndex[serverIdx] - 1,
                PrevLogTerm:  rf.log[rf.nextIndex[serverIdx]-1].Term,
                Entries:      rf.log[rf.nextIndex[serverIdx]:],
                LeaderCommit: rf.commitIndex,
            }
            log.Printf(LogPrefix(rf)+"append entries send: %v to: %v, nextIndex: %v\n", args, serverIdx, rf.nextIndex)
            rf.mu.Unlock()
            reply := &AppendEntriesReply{}
            if rf.sendAppendEntries(serverIdx, args, reply) {
                handler(serverIdx, reply)
            }
        }(i, rf)
    }
}
for !rf.killed() {
    // Your code here to check if a leader election should
    // be started and to randomize sleeping time using
    // time.Sleep().
    <-rf.electionTimer.C
    if rf.killed() {
        break
    }

    rf.mu.Lock()
    switch rf.character {
    case LEADER:
        ...
        reqAppendEntries(rf, func(from int, reply *AppendEntriesReply) {
            rf.mu.Lock()
            defer rf.mu.Unlock()
            ...
            // turn to FOLLOWER if my term is lower than others and return
            // commit entries if the majority of nodes replied my request
        }
    ...
    }
    rf.mu.Unlock()
    ...
}
```

  I was aiming to call reqAppendEntries periodically to send asynchronous AppendEntries request to  other nodes, and handle reply in a callback function. Then I got a multi-leaders error randomly:

  `2022/01/29 18:40:21 raft.go:530: impossible scenario: multiple leaders in the same term: me: 0, other: 3`

  Where the bug is? After struggled a while in the log, I figured out that the problem is **I assumed that all go functions inside reqAppendEntries() can be invoked before any response returns**, while **actually some of them was scheduled to be invoked after another, also after previous responses having been handled by callback functions**.

  Let's make an example:

1. put go func1 and go func2 in schedule but not invoked

2. go func1 invoked but go func2 not
   
   go func1 send args as a LEADER with term T1

3. handler inside go func1 invoked and find out the currentTerm is lower than reply.Term, then transfer to FOLLOWER and set term to T2(reply.Term), where T2 > T1

4. go func2 from the previous LEADER case eventually invoked now, send args with term T2 and believes it is a LEADER

5. other peers receives AppendEntries request both from the real LEADER and the fake LEADER with same term
   
   So the key is to ensure the args passed to sendAppendEntries() holds the states before any callback handler being invoked. I fixed it by making args outside the go func like this:
   
   ```go
   reqAppendEntries := func(rf *Raft, handler func(from int, reply *AppendEntriesReply)) {
    for serverIdx := range rf.peers {
        if serverIdx == rf.me {
            continue
        }
        args := &AppendEntriesArgs{
            Term:         rf.currentTerm,
            LeaderId:     rf.me,
            PrevLogIndex: rf.nextIndex[serverIdx] - 1,
            PrevLogTerm:  rf.log[rf.nextIndex[serverIdx]-1].Term,
            Entries:      rf.log[rf.nextIndex[serverIdx]:],
            LeaderCommit: rf.commitIndex,
        }
        log.Printf(LogPrefix(rf)+"append entries send: %v to: %v, nextIndex: %v\n", args, serverIdx, rf.nextIndex)
        go func(serverIdx int, args *AppendEntriesArgs) {
            reply := &AppendEntriesReply{}
            if rf.sendAppendEntries(serverIdx, args, reply) {
                handler(serverIdx, reply)
            }
        }(serverIdx, args)
    }
   }
   ```
- Don't forget to use ctrl+\ to diagnose deadlock issue. When I was implementing the InstallSnapshot, I encountered a deadlock problem after I add a lock in Snapshot(). The deadlock cycle is:
  
  1. raft.go, ApplyEntries handler: Acquired rf.mu.Lock() in ApplyEntries handler and wait to push ApplyMsg to applyCh when updating commit index. Only after cfg.applierSnap() consumes an entry could carry on here.
  
  2. config.go, cfg.applierSnap(): consume applyCh only after cfg.rafts[i].Snapshot(m.CommandIndex, w.Bytes()) returned.
  
  3. raft.go, Snapshot(): Require and block at rf.mu.Lock() because the lock is being holded by ApplyEntries handler.
  
  Ops, the earth stops rotating now. I can only edit the raft.go due to the game rule and can't remove the lock in Snapshot() or a race condition will occur. So I solved this by adding a breath room to the applyCh:
  
  ```go
  func Make(peers []*labrpc.ClientEnd, me int,
      persister *Persister, applyCh chan ApplyMsg) *Raft {
      ...
      applyBuffer := make(chan ApplyMsg, 10)
      rf.applyCh = applyBuffer
      go func() {
          for {
              msg := <-applyBuffer
              applyCh <- msg
          }
      }()
      ...
  }
  ```
  
  It's ugly but I can't come up with a better idea for now. If you have, please tell me.

### References

- [extended Raft paper](https://pdos.csail.mit.edu/6.824/papers/raft-extended.pdf)

- [6.824 Lab 2: Raft](https://pdos.csail.mit.edu/6.824/labs/lab-raft.html)

- [Students' Guide to Raft :: Jon Gjengset](https://thesquareplanet.com/blog/students-guide-to-raft/)
