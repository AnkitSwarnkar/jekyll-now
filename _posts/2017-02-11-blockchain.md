## Various Blockchain Consensus algorithm Part I

A blockchain is simply a distributed ledger system. Block chain is now a talk of each and every feild. This disruptive technology has created lots of buzz in domains like financial, IOT, academic, assest and more. Many big tech companies like IBM, Disney, Intel, Samsung have started leveraging this amazing technology. In this post I will try to summarize the most important part i.e. heart of block chain, the consensus protocol. 

So what is a consesus protocol?
The term ‘consensus’ refers to a decision-making process which is done in a collabrative way. Generally, for centralized architecture there a centrallized machine (node) which takes all the decision however for block chain its different. Being a distributive architecture, there need to be a set of rule by which various different computers/nodes connected with each other comes to an agreement. More technically, consensus algorithm is required by the blockchain to ensure that every node has a copy of a recognized version of the total ledger.[1] Consensus algorithm allow secure updating of state according to some rules or conditions.  

Since the inception of PBFT consensus algorithm, there are plethora of consensus algorithms like Proof of Work, Proof of Stake, Proof of Concepts etc. in world of blockchain. In this post I will introduce to some of the amazing new consensus protocols. 

#### 1. Tangaroa

#### Introduction
Tangaroa consesus protocol or BFT Raft is a Byzantine server tolerance enhancement to the Raft algorithms i.e. it tried to mitigate the risk of byzantine nodes in previous RAFT[https://raft.github.io/]. The Tangaroa consensus algorithm just like RAFT algorithm divides the consensus algorithm into two problem of log replication and leader selection. A BFT Raft cluster that tolerates f Byzantine failures must contain at least n ≥ 3f + 1 nodes, where n − f nodes form a quorum. At any given time the node can be in only three states of leader,follower, or candidate. 

#### Protocol
When there is a leader, all of the other servers are then in the follower state. Tangaroa like Raft uses remote procedure calls (RPC) as form of communication and divides the time in terms of terms.

In each of the term servers in the network select a leader and is called a election. The node which initiates election act as a client. There are four types of RPCs used in Tangaroa:

* AppendEntries RPC: Used by leader to replicate log entries, provide a heartbeat, and communicate a successful election. 
* RequestVote RPC: Used candidates during the election.
* SendRequest RPC: Used clients to request change of leadership.
* UpdateLeader RPC: Used by clients to request leadership change.

##### Leader Selection
The elections are triggered by a term transition. When a server in the cluster needs to start a new term, it increments its term number, puts itself into the candidate state, and sends a RequestVote RPC to each of the other servers in the cluster. This procedure is similar to RAFT, however the modifications come primarily in the recipient of a RequestVote RPC. When a node receives a RequestVote RPC with a valid signature, it grants a vote only if all five conditions are true:

* the node has not handled a heartbeat from its current leader within its own timeout 
* the new term is between its current term + 1 and current term + H. H is high-term water mark. We choose H such that it is statistically unlikely to encounter H consecutive split votes when a leader has actually failed
* the request sender is an eligible candidate
* the node has not voted for another leader for the proposed term
* the candidate shares a log prefix with the node that contains all committed entries

BFT Raft also uses a randomized timeouts to trigger leader elections. A candidate wins an election if it receives votes from a quorum of the nodes. The leader of each term periodically sends heartbeat messages i.e. empty AppendEntries RPCs to maintain its authority. If any follower receives no communication from a leader over a randomly chosen period of time, the election timeout, then it becomes a candidate and initiates a new election. With Tangoroa, if a client observes no progress with a leader for a period of time, it calls the progress timeout i.e. it broadcasts UpdateLeader RPCs to all nodes, telling them to ignore future heartbeats from what the client believes to be the current leader in the current term.

Note While waiting for votes, a candidate may receive an AppendEntries RPC from another node claiming to be leader. If that leader’s term is at least as large as the candidate’s current term and the leader provides enough votes to support its authority, then the candidate returns to follower state. The candidate continues in the candidate state until one of the three things happens: it wins the election, another node establishes itself as a leader, or a period of time goes by with no winner.

There is also a provision for lazy voting in Tangaroa. Suppose if a RequestVote is valid and for a new term, and the candidate has a sufficiently up to date log, but the recipient is still receiving heartbeats from the current leader. One thing the node can do is to record its vote locally, and then send a vote response if the node itself undergoes an election timeout or hears from a client that the current leader is unresponsive. 

##### Log Replication
A consistent log is maintain across the cluster of servers. Each node stores a cryptographic hash for each log entry known as incremental hash. To compute an incremental hash at index i, the node computes the hash of the incremental hash at index i−1 appended to the log entry at index i. This recursive nature of incremental hashing enables it to verify the integrety of all log entries up through index i which gives BFT Raft a variant of the log matching property robust to Byzantine nodes who may re-order or drop entries from the log. When two nodes agree on an incremental hash at index i, they have identical log entries at index i and all entries prior to i.

The log is maintained using the AppendEntries RPC call. When a client request execute action which chances the state of the cluster, the leader adds an entry to log. Once the proposed log is build, it send it to the other members of the cluster using an RPC. If it gets enough “Yes” votes from other cluster members, then the log entry becomes committed. The log replication process work as follow:
 
1* A client issues a SendRequest RPC to the leader or the node to which it think is a leader. The SendRequest RPC has first a signature, to prevent Byzantine node from forging client requests and a unique identifier, to prevent duplicacy. Similar to PBFT where clients need
to wait for f + 1 matching replies to each request before exposing that result to application logic. The client records all responses to the pending SendRequest RPC. Each time a client receives a response to the pending request, it resets the progress timeout. If a client has not received a response over a period of time called the request timeout, it resends the SendRequest RPC

2* When a leader receives a SendRequest RPC, it sends a signed AppendEntries RPCs in parallel to each replica. A leader will include a quorum of signed votes to support its authority in the current term on the first AppendEntries RPC to each node in each term. Subsequent AppendEntries RPCs can succeed without it once the leader sees a successful reply from that node, indicating that the replica accepts that the current leader won the election for the current term.

3* When a node receives an AppendEntries RPC, it checks if it was from what it believes is the current leader for the current term. If this is not the case, it can reply with an “unconvinced” response, indicating that the leader should send its votes again. The node will then check that it has a matching log prefix using incremental hash and also checks the authenticity of each of the new entries for itself. If the node has a matching previous entry and the new entries are valid, the node will append new entries to its log and compute the incremental hash at each new index which it will broadcast in its AppendEntriesResponse to each other node.

4* When a node receives an AppendEntriesResponse, it will save it if it is for an index higher than the node’s current commit index. Once a node receives a quorum of matching AppendEntriesResponses for a particular index, it is safe to commit everything up to that log entry. When a leader receives an AppendEntriesResponse, in addition to storing the commit information to eventually detect a quorum, the leader will check if the AppendEntries RPC was successful

Some of the safety precautions are similar to RAFT like Leader Safety (only one leader in a network), Leader Completeness(if a log entry is committed in a given term, then that entry will be present in the logs of the leaders for all higher-numbered terms), Leader Append-Only ( non-Byzantine leader never overwrites or deletes entries in its log, it only appends new entries); Log Matching: if two nodes have the same incremental hashes at the same index, then their logs are identical in all entries up through the given index State Machine Safety ( if a non-Byzantine node has applied a log entry at a given index to its state machine, no other non-Byzantine node will ever apply a different log entry for the same index )

This is pretty much for Tangaroa. If you want to learn more, please refer to great [paper](http://www.scs.stanford.edu/14au-cs244b/labs/projects/copeland_zhong.pdf).

I will discuss following consensus in my coming post. Stay tune and keep learning.

#### Template based PBFT

#### Proof of Elapsed Time

#### Delegated Proof of Stake

#### Condra

### References
* Erik Zhang, A Byzantine Fault Tolerance Algorithm for Blockchain[1]
* SIMPLER CONSENSUS WITH RAFT, http://www.goodmath.org/blog/2015/02/14/simpler-consensus-with-raft/[2]
* Christopher Copeland and Hongxia Zhong, Tangaroa: a Byzantine Fault Tolerant Raft[3]
