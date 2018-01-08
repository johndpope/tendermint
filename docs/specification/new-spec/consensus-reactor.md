# Consensus Reactor

Consensus Reactor defines a reactor for the consensus service. It contains ConsensusState service that 
manages the state of the Tendermint consensus internal state machine. 
When Consensus Reactor is started, it starts Broadcast Routine and it starts ConsensusState service. 
Furthermore, for each peer that is added on the Consensus Reactor, it creates (and manage) known peer state 
(that is used extensively in gossip routines) and starts the following three routines for the peer p: 
Gossip Data Routine, Gossip Votes Routine and QueryMaj23Routine. Finally, Consensus Reactor is responsible 
for decoding message received from a peer and for adequate processing of the message depending on its type and content.
The processing normally consists of updating the known peer state and for some messages 
(`ProposalMessage`, `BlockPartMessage` and `VoteMessage`) also forwarding message to ConsensusState module 
for further processing. In the following text we specify the core functionality of those separate unit of executions 
that are part of the Consensus Reactor. 

## Consensus State

Consensus State handles execution of the Tendermint BFT consensus algorithm. It processes votes and proposals, 
and upon reaching agreement, commits blocks to the chain and executes them against the application.
The internal state machine receives input from peers, the internal validator and from a timer.

Inside Consensus State we have the following units of execution: TimeoutTicker and Receive Routine.
TimeoutTicker is a timer that schedules timeouts conditional on the height/round/step that are processed 
by the Receive Routine. 


### Receive Routine

Receive Routine of the ConsensusState handles messages which may cause internal consensus state transitions.
It is the only routine that updates RoundState that contains internal consensus state. 
Updates (state transitions) happen on timeouts, complete proposals, and 2/3 majorities. 
It receives messages from peers, internal validators and from TimeoutTicker 
and invokes the corresponding handlers, potentially updating the RoundState. 
The details of the protocol (together with formal proofs of correctness) implemented by the Receive Routine are 
discussed in separate document (see [spec](https://github.com/tendermint/spec)). For understanding of this document
it is sufficient to understand that the Receive Routine manages and updates RoundState data structure that is 
then extensively used by the gossip routines to determine what information should be sent to peer processes.

## Round State

RoundState defines the internal consensus state. It contains height, round, round step, a current validator set,
a proposal and proposal block for the current round, locked round and block (if some block is being locked), set of 
received votes and last commit and last validators set.  

```
type RoundState struct {
	Height             int64 
	Round              int
	Step               RoundStepType
	Validators         ValidatorSet
	Proposal           Proposal
	ProposalBlock      Block
	ProposalBlockParts PartSet
	LockedRound        int
	LockedBlock        Block
	LockedBlockParts   PartSet
	Votes              HeightVoteSet
	LastCommit         VoteSet 
	LastValidators     ValidatorSet
}    
``` 

Internally, consensus will run as a state machine with the following states:
RoundStepNewHeight, RoundStepNewRound, RoundStepPropose, RoundStepPropose, RoundStepPrevote,       
RoundStepPrevoteWait, RoundStepPrecommit, RoundStepPrecommitWait and RoundStepCommit.        

## Peer Round State

Peer round state contains the known state of a peer. It is being updated by the Receive routine of
Consensus Reactor and by the gossip routines upon sending a message to the peer.

```
type PeerRoundState struct {
	Height                   int64               // Height peer is at
	Round                    int                 // Round peer is at, -1 if unknown.
	Step                     RoundStepType       // Step peer is at
	Proposal                 bool                // True if peer has proposal for this round
	ProposalBlockPartsHeader PartSetHeader 
	ProposalBlockParts       BitArray       
	ProposalPOLRound         int                 // Proposal's POL round. -1 if none.
	ProposalPOL              BitArray            // nil until ProposalPOLMessage received.
	Prevotes                 cmn.BitArray        // All votes peer has for this round
	Precommits               cmn.BitArray        // All precommits peer has for this round
	LastCommitRound          int                 // Round of commit for last height. -1 if none.
	LastCommit               BitArray            // All commit precommits of commit for last height.
	CatchupCommitRound       int                 // Round that we have commit for. Not necessarily unique. -1 if none.
	CatchupCommit            BitArray            // All commit precommits peer has for this height & CatchupCommitRound
}
```   

## Receive method of Consensus reactor

When a message is received from a peer p, it is treated differently based on the channel message belongs to.

TODO: Add high level description of the logic of Receive method. 

## Gossip Data Routine

It is used to send the following messages to the peer: `BlockPartMessage`, `ProposalMessage` and 
`ProposalPOLMessage` on the DataChannel. The gossip data routine is based on the local RoundState (denoted rs) 
and the known PeerRoundState (denotes prs). The routine repeats forever the logic shown below:

```
1a) if rs.ProposalBlockPartsHeader == prs.ProposalBlockPartsHeader and the peer does not have all the proposal parts then
        Part = pick a random proposal block part the peer does not have 
        Send BlockPartMessage(Height, Round, Part) to the peer on the DataChannel 
        if send returns true, record that the peer knows the corresponding block Part
	    Continue  
 
1b) if (0 < prs.Height) and (prs.Height < rs.Height) then
        help peer catch up using gossipDataForCatchup function
        Continue

1c) if (rs.Height != prs.Height) or (rs.Round != prs.Round) then 
        Sleep PeerGossipSleepDuration
        Continue   

//  at this point rs.Height == prs.Height and rs.Round == prs.Round
1d) if (rs.Proposal != nil and !prs.Proposal) then 
        Send ProposalMessage(rs.Proposal) to the peer
        if send returns true, record that the peer knows Proposal
	    if 0 <= rs.Proposal.POLRound then
	    polRound = rs.Proposal.POLRound 
        prevotesBitArray = rs.Votes.Prevotes(polRound).BitArray() 
        Send ProposalPOLMessage(rs.Height, polRound, prevotesBitArray)
        Continue  

2)  Sleep PeerGossipSleepDuration 
```

## Gossip Votes Routine

It is used to send the following message: `VoteMessage` on the VoteChannel.
The gossip votes routine is based on  the local RoundState (denoted rs) 
and the known PeerRoundState (denotes prs). The routine repeats forever the logic shown below:

```
1a) if rs.Height == prs.Height then
        if prs.Step == RoundStepNewHeight then    
            vote = random vote from rs.LastCommit the peer does not have  
            Send VoteMessage(vote) to the peer 
            if send returns true, continue
  
        if prs.Step <= RoundStepPrevote and prs.Round != -1 and prs.Round <= rs.Round then                 
            Prevotes = rs.Votes.Prevotes(prs.Round)
            vote = random vote from Prevotes the peer does not have  
            Send VoteMessage(vote) to the peer 
            if send returns true, continue

        if prs.Step <= RoundStepPrecommit and prs.Round != -1 and prs.Round <= rs.Round then   
     	    Precommits = rs.Votes.Precommits(prs.Round) 
            vote = random vote from Precommits the peer does not have  
            Send VoteMessage(vote) to the peer 
            if send returns true, continue
	
        if prs.ProposalPOLRound != -1 then 
            PolPrevotes = rs.Votes.Prevotes(prs.ProposalPOLRound)
            vote = random vote from PolPrevotes the peer does not have  
            Send VoteMessage(vote) to the peer 
            if send returns true, continue         

1b)  if prs.Height != 0 and rs.Height == prs.Height+1 then
        vote = random vote from rs.LastCommit peer does not have  
        Send VoteMessage(vote) to the peer 
        if send returns true, continue
  
1c)  if prs.Height != 0 and rs.Height >= prs.Height+2 then
        Commit = get commit from BlockStore for prs.Height  
        vote = random vote from Commit the peer does not have  
        Send VoteMessage(vote) to the peer 
        if send returns true, continue

2)   Sleep PeerGossipSleepDuration 
```

## QueryMaj23Routine

It is used to send the following message: `VoteSetMaj23Message`. `VoteSetMaj23Message` is sent to indicate that a given 
BlockID has seen +2/3 votes. This routine is based on the local RoundState (denoted rs) and the known PeerRoundState 
(denotes prs). The routine repeats forever the logic shown below. 

```
1a) if rs.Height == prs.Height then
        Prevotes = rs.Votes.Prevotes(prs.Round)
        if there is a ⅔ majority for some blockId in Prevotes then
        m = VoteSetMaj23Message(prs.Height, prs.Round, Prevote, blockId)
        Send m to peer
        Sleep PeerQueryMaj23SleepDuration

1b) if rs.Height == prs.Height then
        Precommits = rs.Votes.Precommits(prs.Round)
        if there is a ⅔ majority for some blockId in Precommits then
        m = VoteSetMaj23Message(prs.Height,prs.Round,Precommit,blockId)
        Send m to peer
        Sleep PeerQueryMaj23SleepDuration

1c) if rs.Height == prs.Height and prs.ProposalPOLRound >= 0 then
        Prevotes = rs.Votes.Prevotes(prs.ProposalPOLRound)
        if there is a ⅔ majority for some blockId in Prevotes then
        m = VoteSetMaj23Message(prs.Height,prs.ProposalPOLRound,Prevotes,blockId)
        Send m to peer
        Sleep PeerQueryMaj23SleepDuration

1d) if prs.CatchupCommitRound != -1 and 0 < prs.Height and 
        prs.Height <= blockStore.Height() then 
        Commit = LoadCommit(prs.Height)
        m = VoteSetMaj23Message(prs.Height,Commit.Round,Precommit,Commit.blockId)
        Send m to peer
        Sleep PeerQueryMaj23SleepDuration

2)  Sleep PeerQueryMaj23SleepDuration
```

## Broadcast routine

The Broadcast routine subscribes to internal event buss to receive new round steps, votes messages and proposal
heartbeat messages, and broadcasts messages to peers upon receiving those events.
It brodcasts `NewRoundStepMessage` or `CommitStepMessage` upon new round state event. Note that
broadcasting these messages does not depend on the PeerRoundState. It is sent on the StateChannel. 
Upon receiving VoteMessage it broadcasts `HasVoteMessage` message to its peers on the StateChannel. 
`ProposalHeartbeatMessage` is sent the same way on the StateChannel.



 


 
 

    


    



  


