# Cosmos and Tendermint Research

The following is a list of open research problems that the Tendermint team is interested in exploring.  If you are interested in helping research or implement any of these ideas, please get in touch with us, and we'd be happy to setup a bounty to reward you for you contributions!


### Advanced signature schemes
* Using BLS or Shnorr signatures for Tendermint signing
    * Can we aggregate signatures during the gossip in the core layer?
* Using BLS singatures on transactions to aggregate signatures in a block
* Tradeoffs of different post-quantum signature schemes
* Recovery mechanism for breaking of non quantum-resistant signatures like BLS (what do we do when quantum signatures are broken??)


### Tendermint Consensus
* Pipelined Tendermint
    * Tendermint currently uses two round trips of "voting" (pre-votes and pre-commits) in order to achieve byzantine fault tolerance. Can we **optimistically** "pipeline" two tendermint blocks by using the pre-vote for the next block as the pre-commit for the previous block.
* Cryptographic Sortition

    * Instead of requiring the entire validator set to validate every single block, can we randomly select a subset of the validators (for example randomly select 100 out of total set of 10000) to validate each block.
    * Would need a secure randomness beacon that is deterministic but unpredictable.
* BFT Time (in progress) 
* Alternatives to Stake for determining validator weight for public systems
    * Proof of Useful Work / Storage?
* Novel proposal mechanisms for Tendermint consensus (alternatives to round robin proposer)
    * Nakamoto consensus chain as proposal mechanism
        * PoW Nakamoto chain (Casper FFG)
        * PoS Nakamoto Chain (Snow White, Ouroboros Praos, etc)
    * Tendermint-NG (Apply ideas from Bitcoin-NG to Tendermint)
    * P2P Communication DAG (hashgraph-like?) as proposal mechanism into Tendermint for in-block ordering 
* Formal verification of Tendermint consensus
* Performance testing at scale of Tendermint consensus engines
* Model Tendermint in reference to other more abstract consensus algorithms
    * Modeling Tendermint as parameterization of Casper CBC
    * Tendermint as instantiation of Stellar Federated Byzantine Agreement
    
    
### ABCI
* Building other ABCI consensus engines
    * Currently Tendermint Core is the only consensus engine that matches the ABCI interface spec. Can we create implementations of other consensus protocols such as Casper CBC and HoneyBadger BFT to also fit the ABCI interface, so blockchain developers can choose which consensus engine to use for their application.
* Creating ABCI interface for p2p layer
    * How the ABCI creates an abstraction between the blockchain application and the consensus layer, can we create a similar abstraction between the consensus and peer to peer layers.  This will allow blockchains to choose their preferred gossip protocol or networking stack.

    
### Cosmos SDK
* Dependency-based Partial Ordering of Transactions in block rather that total global ordering
    * Optimize software to take advance of GPUs/multicore CPUs/parallel threading?
* Self-executing transactions (transcation waiting in mempool for time or state boolean)
* Address serialization
    * Bech32?
* Privacy features?
* Implementing existing applications/VMs as Cosmos Modules or ABCI Apps
    * Chainmint (Chain.com's Bitcoin-like VM)
    * ZCash


### Inter Blockchain Communication (IBC)
* Multi-hop IBC transfers (no needing user input between hops)
* Cross-Chain asynchronous Smart Contract Calls over IBC
* Send Non Fungible Asset/Token Transfer over IBC
* Modeling IBC in capability approach modeling of interchain communication and transfers
* Integrating IBC into other platforms
    * Integrating IBC natively into post-Casper Ethereum and other systemns with finality
    * Tooling to easily add IBC support into non Cosmos SDK based ABCI apps
    * Peg Zones
        * EVM (Ethereum, Ethereum Classic) Peg Zones (In Progress)
        * Bitcoin and Bitcoin-like Peg Zones
    * Use of IBC for Ethereum Plasma
* Private Chains
    * Wrapper around traditional systems to treat them as a "1-validator blockchain"
    * Tooling for using private blockchains as data oracles for public systems
    

### Multichain Security Models
* Fraud Proofs of invalid state transitions on other chains
* What is proper mechanism for dealing with forks in other sovereign chains?
* Hosted Consensus Model
    * All Cosmos Hub validators operate hosted chains, but this can be done in parallel rather than in sequence.
    * Model scalability of this model
    * Sharding
        * Allowing different subsets of Cosmos Hub validator set to operate different hosted chains but have fraud proof proof submission to entire validators set 
* Plasma
    * Implementation of Plasma security into IBC to offer chains as an optional model
    * Chain is operated by its own validators, but users and validators can avoid censorship by escaping on parent chain
* Model different multi-chain/shard models using Stellar Federated Byzantine Models


### Proof of Stake and Economics (Staking and Slashing)
* Can we allow for delegators to instantly rebond to a different validator without going through unbonding period?
* Tradeoffs in slashing conditions to punish byzantine behavior but not disincentivize participation 
* Cartelization and Censorship Resistance
    * What are the tradeoffs in methods for dealing with unattributable faults such as validator availability?
    * How can we increase coordination cost for cartels?
    * How does governance affect potential for cartelization?
* How do we model the security and cost differences between the separate staking/fee token model vs the single token model?
* Effects of Proof of Work vs Proof of Stake *for Coin Distribution* on Decentralization
* Incentivization of Full Nodes
    * Integration with distributed storage (like IPFS)
        * Payments based on Proof of Storage / Bandwidth?
    * Light Clients pay Full Nodes for serving data
* Resource pricing and Fees
    * Storage Rent
    * CheckTx returns variable gas costs based on Message Type
        * "Gas counting" for Cosmos SDK?
    * Multi-Fee Tokens
        * Tooling for fetching live market data, DEX data, etc



### Governance and Hard Forks
* Effects of governance mechanisms
* Optimal Off-chain communication mediums
* Best practices for software update hard forks
* Impacts of novel experimental governance mechanisms
    * Futarchy?  Liquid Democracy?


### Next Generation Blockchain VMs
* UTXO-based smart contracting systems
    * [Smart Signatures] (https://www.slideshare.net/ChristopherA/smart-signaturesexperiments-in-authentication-stanford-bpase-2018-final)
    * Excecution in zero-knowledge?
* WebAssembly-based VMs
* Formal Verification
* Measuring effects of performance differences between VMs on scalability  


### Secure Validators
* Techinques for maintatining uptime and DDOS Protection
* "Smart HSMs" and Trusted hardware (SGX)


### Miscellaneuous
* Cross-chain identity
* Proof of Computation
    * Zk-snarks, TrueBit, Multiparty computation
