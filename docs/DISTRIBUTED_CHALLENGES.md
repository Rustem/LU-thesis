## Spanner

Google demonstrates that running 2PC and synchronous replication is feasible in the wide area. Although individual transaction latencies are high (due to multiple network roundtrips), the overall system throughput can still be good.

Key ideas:
- Semi-relation data model and general purpose transactions
- "External consistency" - transactions are ordered by their commit time, and commit time correspond to "TrueTime"
- Paxos-based replication, with number of replicas and distance of replicas controllable
- Data sharding accross thousands of servers

### External consistency

Within each Paxos group all writes in a transaction are globally ordered by timestamp. In addition, Spanner preserves the following invariant: if the start of a transaction T2 occurs after the commit of a transaction T1, then commit timestamp of T2 must be greater than commit timestamp of T1.

To provide time safety accross all nodes, Google uses time syncronization protocols where nodes exchange times with each other and time syncronization (GPS, atomic clocks).

Google exposes TrueTime API that estimates the accuracy of a node's clock, guaranteeing that a timestamp T belongs to an interval (T.earliest, T.latest).


Client Message:
M = {
    group_ids = set(),
    coordinator_id = #host_id,
    message_tag = uuid(),
    op = WRITE
    value = v
}

Proposer:
PREPARE(n, index, t(prepare, i))
ACCEPT(index, n, M)

State of each node:
LatestAssignedTimestamp - the latest timestamp that was assigned
firstUnacceptedTransaction - the last index that is not yet to be chosen


NC(i) Non-coordinator participant leader
C(j) Coordinator participant leader (such as j does not belong to any i )
GROUPS = groups(i..j..)
PAXOS(i) is MULTI-PAXOS of non coordinator group
PAXOS(j) is MULTI-PAXOS of coordinator group
ack(i) is acknowledgement message on commit message

### Write transaction (T -> s)
1. The CLIENT chooses the GROUPS(i..j..k..) and coordinator C(j) needed to perform the transaction T
2. CLIENT sends protocol message M to all participant leaders NC(i) + C(j)
3. NC(i) receives message M (at some time T(i)enter)
4. NC(i) acquires write locks (2PC expanding phase)
5. NC(i) chooses prepare timestamp: t(prepare, i) such that it greater than any previously assigned timestamps
6. NC(i) proposes message M with t(prepare, i) by applying PREPARE() to PAXOS(i)
7. NC(i) responses to C(j) with t(prepare, i)
- C(j) acquires lock for T. (2PC expanding phase)
8. If C(j) receives from the majority NC(i) timestamp t(prepare, i):
       C(j) chooses commit timestamp S such that:
       ..* it greater than any commit timestamps it previously assigned
       ..* it greater than any t(prepare,i)
       ..* it greater than TT.now.latest()
   else:
       TIMEOUT, go to (3)
9.1 C(j) send ACCEPT(M, S) to PAXOS(j)
9.2 C(j) waits until TT.after(S) = True
10. C(j) broadcasts S to NC(i) and CLIENT
11. NC(i) receives S
12. NC(i) sends ACCEPT (M,S) to PAXOS(i)
13. NC(i) release its lock (2PC shrinking phase)
14. C(j) receives ack(i)
15. If C(j) receives ack(i) from the majority NC(i):
    15.1 C(j) releases lock (2PC shrinking phase)
    15.2 C(j) go to step (10)
16. END

Questions:

What happens if network partitioned ?
Suppose the transaction T is concerned three groups: G1, G2, G3.
Coordinator is NC(1)
Others are C(2), C(3)

What happens if G3 has not received commit timestamp S ? For exampe network suddenly partitioned. But then, later it will recover.

