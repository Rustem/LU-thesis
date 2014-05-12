https://ramcloud.stanford.edu/~ongaro/userstudy/quizzes.html

1. a) - yes
   b) - no bad
   c) - no
   d) - yes
2. No it could not. Let's assume that another proposal, say, 6.5 proposes new value Y. Since the majority of the nodes must answer on prepare request, then they should overlap with one that was accepted previously X. That one that accepted X will return in prepare response as Prepare(acceptedProposal, acceptedValue). Then Proposer of 6.5 receives from that server previously acceptedValue X and updates it to choose

3. a) The lower bound is zero prepare rpcs. For instance, after a leader is established and before sending prepare rpc, it could be failed somehow.
b) The upper bound is one round of Prepare RPC is required. Rather than doing prepare commands once per entry, a leader can do it once per Entire log. When he receives "noMoreAccepted" for proposal number from majority, that's considered that acceptors will no longer accepts proposals from any one beyond this proposal number. 5 points
4. Suppose acceptor really skips checking proposal number in the entries with the received one, then it could update all entries, even from other old servers that are not chosen, and that are chosen. The upshot: inconsistent log, inconsistent behaviour of system, system failure.
5. a) No it's not. The proposal number could be either greater or lower. If it's lower, then proposal will be rejected and tried again with greater number then current. If it's greater than ok.
b) No it's not.
