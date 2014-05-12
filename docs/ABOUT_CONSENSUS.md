## Paxos (single decree)

In basic Paxos the system must agree on only one value to be chosen.

Goal: Replicated Log

Main features of Paxos are:
* consensus module that ensures proper log replication
* system makes progress as long as any majority of servers are up
* failure model: fail-stop, delayed-lost messages
* Replicated state machine implied by replicated log


Requirements:
* Safety:
..* only a single value may be chosen
..* a server never learns that a value has been chosen unless it is.
* Liveness:
..* some proposed value is eventually chosen
..* if a value is chosen, servers eventually learn about it

Paxos Components:

Proposers - active components that:
* put forth particular value to be chosen
* handle client requests

Acceptors:
* passive: respond to messages from proposers
* responses repr. votes that form consensus
* store chosen value, state of decision process
* want to know which value to be chosen

Problems and Solutions

* What if we have only one acceptor ?
  Answer: quorum based on multiple acceptors (3, 5, ...). Based on quorum the value v is chosen if accepted by majority of acceptors. If one acceptor crashes after the value has been chosen, quorum has this value

* What if the votes are split (e.g. value x and y has quorum) ?
  Answer: 2-phase protocol (once the value has been chosen, future proposals must propose/choose that same value)

* What if conflicting choices occured ?
  Answer: Order proposals, reject old ones. Each server should stores maxRound: the largest Round Number it has seen so far. It should persist on disk.

Two-phase approach:
* Phase 1: broadcast Prepare RPC
  * Find out about any chosen values (split votes resolution)
  * Block older proposals that have not yet been completed (conflict resolution)
* Phase 2: broadcast Accept RPCs
  * Ask acceptors to accept a specific value

Full Protocol

1) Proposer receives value x from client. It chooses new proposal number n
2) Proposer broadcast Prepare(n) to all of the acceptors in the cluster
3) Acceptor(s) receives the proposal number n. Acceptor does two things:
  3.1 It keeps track on minProposal value that contains the highest value it has seen so far. So if n > minPropal, then update: minProposal = n
  3.2 If it accepted so far any proposals, then it returns in response them as Return(acceptedProposal, acceptedValue)
4) Proposer receives the majority of responses:
   4.1 If any acceptedValue returned, then replace value x with acceptedValue for highest acceptedProposal
5) Proposer broadcasts Accept(n, value) to all acceptors
6) Acceptor(s) respond to Accept(n, value):
  6.1 If n >= minProposal then
         acceptedProposal=minProposal=n
         acceptedValue=value
  6.2 else: Return(minProposal) ()
7) Proposer receives majority of accept responses:
  7.1 if any rejections (result > n) ? goto (1)
  7.2 value is considered choosen and stored in log

Notes: Proposer can use information in result to build new proposal number
       Acceptors must record minProposal, acceptedProposal and acceptedValue on disk.


Problems in single-decree Paxos
* Livelock between competing proposers

s1 Prepare (3.1)
s2 Prepare (3.1)
s3 Prepare (3.1)
  s3 Prepare (3.5)
  s4 Prepare (3.5)
  s5 Prepare (3.5)
     s1. Reject Accept (3.1)
     s2. Reject Accept (3.1)
     s3. Reject Accept (3.1)
        s1. Prepare (4.1)
        s2. Prepare (4.1)
        s3. Prepare (4.1)
           s3. Reject Accept (3.5)
           s4. Reject Accept (3.5)
           s5. Reject Accept (3.5)
              ...
                ...
Solution: randomized delay before restarting or leader election

* only proposers knows which value has been chosen
* if other servers want to know, must execute Paxos with their own proposals

## Multi-Paxos

Transition

* Which log entry to use for a given client request ?
Acceptor finds in its log the first place that its known not to be chosen and it tries to arrange new value to be chosen for that entry. Execute Paxos, if it's not been chosen do it again with next place that its known not to be chosen. (nextUnchosenSlot or nextUnchosenIndex).

With this approach server can handler multiple requests concurrently by selecting different log entries for each. But those commands must be executed in order on state machines.
* Leader
Strong leader in a system resolves the livelock conflict mentioned before. At any given time, only one server acts as Proposer. Another point, with only single proposer we can eliminate a prepare RPCs by preparing only once for the entire log, and then execute all proposals in accept log.

Leader elections can be done as follows:
  * let the server with highest ID acts as a leader
  * each server sends a heartbeat msg to every other server every T ms (msg contain id of the server)
  * if a server hasn't received heartbeat from server with higher ID in last 2T ms, it acts as a leader.
* Ensuring full replication of log
  1) keep retrying Accept RPCs until all acceptors respond (in background)
  2) keep track of entries that are known to be chosen
     acceptedProposal[i] = Infinity (the largest value that could be exist) (never get overwritten); maintain firstUnchosenIndex value
  4) proposer piggybacks firstUnchosenIndex in Accept RPCs. Acceptor marks all entries i chosen if: i < request.firstUnchosenIndex and acceptedProposal[i] == request.proposal (acceptor knows about most chosen entries)
  Accept(proposal=3.4, index=8, value=v, firstUnchosenIndex=7)
  5) entries from old leaders that are not considered chosen. Acceptor returns its fristUnchosenIndex in Accept replies. If proposer's firstUnchosenIndex > firstUnchosenIndex from response, then proposer sends Success RPC in background. Success(index, v): notifies acceptor of chosen entry.
  Acceptor updates information as follows:
    acceptedValue[index] = v
    acceptedProposal[index] = INFINITY
    return firstUnchosenIndex => this allows to proposer send another Success RPC
* Client protocol
Send commands to leader
..* if leader unknown, contact any server
..* if contacted server is not a leader, it will redirect to leader
Leader does not respond until command has been chosen for log entry and executed by leader's state machine.
If request times out (e.g. leader crash)
* client reissues command to some other server
* eventually redirected to new leader
* eventually the request will succeed
But what if leader crash after executing command but before responding ?
* client embeds unique id in each command, then server includes id in log entry. State machine records most recent command executed for each client. Before executing command, state machine checks to see if command already executed. If so: ignore new command, return response from old command.
  Result: exactly-once semantics as long as client doesn't crash.
* Configuration changes
Reasons: replace failed machines, change degree of replication

Safety requirement:
  During configuration changes, it must not be possible for different majorities to choose different values for the same log entry:
   ----
   ssssssssss
   ----------
  Solution by Leslie Lamport: use log to manage configuration changes:
    * configuration is stored as log entry
    * replicated like any other log entry
    * configuration for choosing entry i determined by entry i - alpha. Suppose a = 3:
    1 2 3 4 5 6 7 8 9 10 11
    c1  c2
    -----
     c0
         ----
         c1
              ---------------
                    c2
Notes: alpha limits concurrency: can't choose entry i+alpha until entry i chosen.



