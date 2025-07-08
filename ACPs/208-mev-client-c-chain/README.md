| ACP           | 208                                                                                   |
| :------------ | :------------------------------------------------------------------------------------ |
| **Title**     | MEV Zone as Canonical MEV Infrastructure in AvalancheGo                                                  |
| **Author(s)** | [0x7183](http://github.com/0x7183) [toran777](http://github.com/toran777)                                    |
| **Status**    |  |
| **Track**     | Best Practices |       


## Abstract
This ACP introduces MEV Zone, a framework for managing and democratizing Maximal Extractable Value (MEV) on Avalanche. 
MEV Zone provides an open and transparent infrastructure where all participants—validators, searchers, users, and block builders—can interact under clear and public rules. It enables private transactions, protects users from frontrunning, and introduces sealed-bid auctions where searchers compete fairly. A portion of extracted value is redistributed to the Avalanche community: validators receive MEV rewards and users benefit from reduced manipulation and $AVAX burning.

## Motivation
Every year, over $3.5 million is extracted through MEV on Avalanche, with little to no competition. None of this value currently flows back to the chain. As Avalanche continues to grow, this number is expected to rise significantly. 
Now is the time to act to ensure Avalanche is properly positioned for the future. 

Maximal Extractable Value (MEV) is an unavoidable dynamic present in all decentralized networks where the ordering of transactions directly impacts the outcome of the system. 
Avalanche is no exception: MEV already occurs on-chain, but the value it generates is currently being siphoned off by a small number of centralized actors operating through opaque, private arrangements. This creates an imbalanced and exclusionary ecosystem, where the benefits of MEV are not shared with the broader community and are instead concentrated in the hands of a few.

MEV Zone offers a fundamentally different approach. It introduces a transparent and inclusive infrastructure designed to align MEV extraction with the interests of the Avalanche network and its users. Instead of relying on closed-door deals and informal coordination, MEV Zone brings MEV into the open, creating a framework where validators, searchers, users, and block builders interact under publicly verifiable rules.
By formalizing private transaction support, enabling sealed-bid auctions, and redistributing part of the extracted value, MEV Zone turns a previously extractable mechanism into a source of shared value.

This Avalanche Community Proposal (ACP) seeks formal community endorsement to recognize MEV Zone as the canonical MEV infrastructure on Avalanche. By integrating it into the base client, we ensure that the value inevitably extracted from transaction ordering is not lost to third parties, but instead returned to the chain and its stakeholders. Validators receive fair MEV rewards, users benefit from reduced manipulation and $AVAX burning, and the ecosystem gains visibility into a once obscure topic. Ultimately, this shift lays the groundwork for a more democratic and transparent MEV landscape, where value flows through open mechanisms, opportunities are accessible to all actors—not just a privileged few—and protocol-level incentives reinforce trust, efficiency, and long-term sustainability for the Avalanche network.

#### Required Changes
- Integration of MEV Zone in the AvalancheGo Base Client

Integrating MEV Zone directly into the AvalancheGo base client means that every validator running the official Avalanche node software will have MEV Zone functionality available but disabled by default. Validators can choose to activate it by updating their configuration file.
By being part of the base client, MEV Zone becomes immediately available to all validators, making participation seamless and increasing adoption across the network.
It ensures there’s one shared MEV framework, rather than fragmented solutions. This is critical for transparency and coordination.
Since AvalancheGo is maintained by the core protocol team, any MEV-related changes go through the same rigorous testing, versioning, and auditing pipeline as the rest of the code.

## Specification

Here’s an overview of the entities in the MEV context.


#### Validator
Role: Validators are responsible for creating and publishing blocks in the Avalanche network.
Interactions:
Receives bids from the Builder for the right to include specific transactions in the block.
Publishes the final block to the network based on the highest bids.


#### Avalanche Network & Mempool
Role: Acts as the communication hub where all transactions are broadcasted and temporarily stored in the mempool until included in a block.
Interactions:
Searchers and Builders listen to the mempool for pending transactions to identify profitable opportunities for MEV extraction.
Validators publish finalized blocks back to the network after including selected transactions.

#### Searchers
Role: Searchers are specialized participants that monitor the mempool for profitable opportunities, such as arbitrage or liquidations, by analyzing pending transactions.
Interactions:
Listen to pending transactions: Continuously scan the mempool to identify MEV opportunities.
Send bundles: Once an opportunity is identified, searchers create optimized bundles of transactions and send them to the Builder for inclusion in the next block.


#### Builder
Role: Builders aggregate transaction bundles from Searchers and construct the most profitable block possible. They then bid on validators to include their block in the blockchain.
Interactions:
Receive bundles: Accepts bundles from searchers that are designed to extract MEV.
Send bids: Proposes blocks to the Validator, offering payment (bids) in exchange for the right to have their block included.


## Backwards Compatibility
The implementation of this proposal will not introduce any backwards compatibility issues. It extends functionality for those who want to participate without impacting those who don’t.

## Reference Implementation
RawBid represents a raw bid from builder directly.
```
type RawBid struct {
	BlockNumber   	uint64      	`json:"blockNumber"`
	ParentHash    	common.Hash 	`json:"parentHash"`
	Txs           	[]hexutil.Bytes `json:"txs"`
	UnRevertible  	[]common.Hash   `json:"unRevertible"`
	GasUsed       	uint64      	`json:"gasUsed"`
	GasFee        	*big.Int    	`json:"gasFee"`
	MevRewards    	*big.Int    	`json:"mevRewards"`
	MevBurnShare  	*big.Int    	`json:"mevBurnShare"`
	MevValidatorShare *big.Int    	`json:"mevValidatorShare"`
	hash atomic.Value
}
```

BidArgs represents the arguments to submit a bid.
```
type BidArgs struct {
	// RawBid from builder directly
	RawBid *RawBid
	// Signature of the bid from builder
	Signature hexutil.Bytes `json:"signature"`
	// PayBidTx is a payment tx to builder from sentry, which is optional
	PayBidTx    	hexutil.Bytes `json:"payBidTx"`
	PayBidTxGasUsed uint64    	`json:"payBidTxGasUsed"`
	BurnTx      	hexutil.Bytes `json:"burnTx"`
}
```
For more details, refer to the mev.zone [documentation](https://mevzone.gitbook.io/mevzone).


## Security Considerations
MEV Zone integrates with AvalancheGo through minimal, isolated modifications that do not affect core consensus or block production. Validators remain fully functional regardless of MEV Zone status. If builders fail or behave incorrectly, validators default to standard block construction using internal logic.
