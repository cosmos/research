# Cosmos and Tendermint Research

The following is a list of open research problems that the Tendermint team is interested in exploring.  If you are interested in helping research or implement any of these ideas, please get in touch with us, and we'd be happy to setup a bounty to reward you for you contributions!


### Advanced signature schemes
* Using BLS signatures for Tendermint signing
    * Aggregate signatures during the gossip in the core layer
* Using BLS signatures on transactions to aggregate signatures in a block
* Recovery mechanism for dealing with the breaking of non quantum-resistant signatures like BLS
   * Automatic fallback to post-quantum signature schemes (requires everyone to submit a fallback key on chain)


### Tendermint Consensus
* Formal verification of Tendermint consensus [[complete](https://arxiv.org/abs/1807.04938)]
* BFT Time [[in progress](https://github.com/tendermint/tendermint/blob/master/docs/spec/consensus/bft-time.md)]
* Pipelined Tendermint - Tendermint currently uses two round trips of "voting" (pre-votes and pre-commits) in order to achieve byzantine fault tolerance. Can we **optimistically** "pipeline" two tendermint blocks by using the pre-vote for the next block as the pre-commit for the previous block.
* Novel proposal mechanisms for Tendermint consensus (alternatives to round robin proposer)
    * [Tx Pre-Sequencing](https://github.com/tendermint/tendermint/issues/1168)
    * Nakamoto consensus chain as proposal mechanism
        * PoW Nakamoto chain (Casper FFG)
        * PoS Nakamoto Chain (Snow White, Ouroboros Praos, etc)
    * P2P Communication DAG (hashgraph-like?) as proposal mechanism into Tendermint for in-block ordering
* Performance testing at scale of Tendermint consensus engines
* Cryptographic Sortition
    * Instead of requiring the entire validator set to validate every single block, can we randomly select a subset of the validators (for example randomly select 100 out of total set of 10000) to validate a specific block.
    * Would need a secure randomness beacon that is deterministic but unpredictable.
* Alternatives to Stake for determining validator weight for public systems
    * Proof of Useful Work / Storage?
* Model Tendermint in reference to other more abstract consensus algorithms
    * Modeling Tendermint as parameterization of Casper CBC
    
    
### ABCI
* Building other ABCI consensus engines
    * Currently Tendermint Core is the only consensus engine that matches the ABCI interface spec. Can we create implementations of other consensus protocols such as Casper CBC and HoneyBadger BFT to also fit the ABCI interface, so blockchain developers can choose which consensus engine to use for their application.
* Creating ABCI interface for p2p layer
    * How the ABCI creates an abstraction between the blockchain application and the consensus layer, can we create a similar abstraction between the consensus and peer to peer layers.  This will allow blockchains to choose their preferred gossip protocol or networking stack.
    * Look into [[libp2p](https://github.com/libp2p)]

    
### Cosmos SDK
* Dependency-based Partial Ordering of Transactions in block rather that total global ordering
    * Optimize software to take advance of GPUs/multicore CPUs/parallel threading?
* Self-executing transactions (transcation waiting in mempool for time or state boolean)
    * Another way to do this: a transaction can specify a future transaction that must be attempted at a future block, changing the consensus ruleset to require that the future transaction be attempted at that block
* Address serialization
    * Bech32 [[complete](https://github.com/cosmos/cosmos-sdk/blob/develop/docs/spec/other/bech32.md)]
* Privacy features?
* Implementing existing applications/VMs as Cosmos Modules (or ABCI Apps)
    * EVM (Ethermint 2.0) [[in progress](https://github.com/cosmos/ethermint)]
    * Chainmint (Chain.com's Bitcoin-like VM) [[in progress](https://github.com/chainx-org/chainmint)]
    * ZCash
    * Primea


### Inter Blockchain Communication (IBC)
* Pay for the fees on another zone using payment channel transfers on the hub
   * Payment channel settlement is dependant on IBC proofs of the transaction being inserted in the zone.
* Multi-hop IBC transfers (no needing user input between hops)
* Cross-Chain asynchronous Smart Contract Calls over IBC
* Send Non Fungible Asset/Token Transfer over IBC
* Modeling IBC in capability approach modeling of interchain communication and transfers
* Integrating IBC into other platforms
    * Integrating IBC natively into post-Casper Ethereum and other systemns with finality
    * Tooling to easily add IBC support into non Cosmos SDK based ABCI apps
    * Peg Zones
        * EVM (Ethereum, Ethereum Classic, etc) Peg Zones [[in Progress](https://github.com/cosmos/peggy/)]
        * Bitcoin and Bitcoin-like (Litecoin, ZCash, etc) Peg Zones
    * Use of IBC for Ethereum Plasma
* Private Chains
    * Wrapper around traditional systems to treat them as a "1-validator blockchain"
    * Tooling for using private blockchains as data oracles for public systems
* Zero-knowledge IBC
    * Cross-chain asset transfers with no information revealed
    * Could obscure routing to mitigate inference from metadata (so observer can't figure out which two chains exchanged assets)
* IBC as a compiler target
    * Smart contract language exposing IBC abstractions (connection, channel) as language primitives
    * Deployed contracts target a set (not necessarily fixed-size) of blockchains running IBC & common VMs, scaling automatic & developer-transparent
* Byzantine recovery strategies
    * Plasma-like fraud proofs to recover assets on original chain (or any chain, potentially)
    * Automatic Byzantine recovery? Would require exact fraud proofs, but could possibly allow security of IBC-defined assets to be 1-of-n chains


### Multichain Security Models
* Fraud Proofs of invalid state transitions on other chains
* What is proper mechanism for dealing with forks in other sovereign chains?
* Hosted Consensus Model
    * Model scalability of this model
    * Sharding
        * Allowing different subsets of Cosmos Hub validator set to operate different hosted chains but have fraud proof proof submission to entire validators set 
* Plasma
    * Implementation of Plasma security model into Cosmos Hub to offer chains as an optional alternative model to sovereign and hosted consensus
* Model different multi-chain/shard models using Stellar Federated Byzantine Models


### Proof of Stake and Economics (Staking and Slashing)
* Can we allow for delegators to instantly rebond to a different validator without going through unbonding period? [[in progress](https://github.com/cosmos/cosmos-sdk/tree/develop/x/stake)]
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
    * [Futarchy](https://blog.ethereum.org/2014/08/21/introduction-futarchy/)?  [Liquid Democracy](https://www.youtube.com/watch?v=fg0_Vhldz-8)?


### Next Generation Blockchain VMs
* UTXO-based smart contracting systems
    * [Smart Signatures](https://www.slideshare.net/ChristopherA/smart-signaturesexperiments-in-authentication-stanford-bpase-2018-final)
    * Excecution in zero-knowledge?
* WebAssembly-based VMs
* Formal Verification
* Measuring effects of performance differences between VMs on scalability  
* Typesystem-defined VMs
    * Instead of defining instruction execution semantics, consensus rules define high-level language and symbolic execution rules (no machine types)
    * Typechecker included in consensus ruleset
    * Consensus rules define formal semantics, consensus state defines evaluation model
    * *Anyone* can submit a transaction changing the evaluation semantics - if they still obey the symbolic execution rules & are more efficient, consensus rules are changed and the user gets paid
* Lock-free optimistic concurrency
    * STM-like - https://en.wikipedia.org/wiki/Software_transactional_memory - works kinda like PostgreSQL MVCC
    * As long as the *output* is determistic (linearizable), order-of-evaluation can vary across nodes


### Secure Validators
* Techinques for maintatining uptime and DDOS Protection
* "Smart HSMs" and Trusted hardware (SGX)


### Miscellaneuous
* Cross-chain identity
* Proof of Computation
    * Zk-snarks, TrueBit, Multiparty computation
* Real World Economics
    * [Local Currencies as an alternative to UBI](https://twitter.com/buchmanster/status/897188296354385920)
    * Integration with central banks


### Long-term
* Distributed ledger network infrastructure across non-trivial lightspeed delays
    * Settlement of Earth <=> Mars colony transactions, neither wants to trust the other
    * Higher-value slower chains with validators in both locations, lower-value faster chains in concentrated localities?
