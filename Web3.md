# Web3

## Bitcoin

### Gas Fee

### Settlement Delay

### Nonce

> The nonce here is different from the nonce in Ethereum
> 
> 
## Ethereum

### Enterprise

> 一般是私链或者联盟链
> 
- HyperLedger Besu (EVM)

  > 以太坊兼容，所谓以太坊兼容，值得是，可以在enterprise host一个besu节点，然后接入公网的以太坊公链。这也是MALT，Yoda，3M选择Besu的原因之一
  > 类似的enterprise 链有hyperledger fabric
  > 
	- JRPC: JSON Remote Procedure Call

	- Validation Node

	- Reporting Node

- HyperLedger Fabric (None EVM)

- Corda (EVM)

### Public

- Mainnet

- Testnets: Sepolia

### Layer 2

> 主要用来扩容和提高交易速率：TPS
> 
- Rollups

	- Optimistic Rollups

	- ZK Rollups

- Sidechains

## Decentralized

### Consensus Algorithm

- PoW

  > 比特币中最早的共识算法，消耗巨大算力来计算nonce，挖矿概念的由来。需要注意的是另外一个在Etherum中nonce概念，nonce gap，这里的nonce指的是number used once。
  > 另外值得注意的是，Etherum的挖矿机制和比特币很不一样，采用的是Ethash，是一种内存密集型的算法，而比特币的SHA-256是计算密集型，主要为了抵制ASIC矿机
  > 
- PoS

  > 类似于董事会投票机制，股权越大的股东，越容易被选为验证者来计算打包新的区块，或者验证新打包出来的区块。
  > 
- PBFT

  > 私链或者联盟链常用的共识算法，类似于投票机制
  > 
### Cryptographic Algorithms

- SHA-256 (SHA2)

- ECDSA

- Keccak-256 (SHA3)

## Security

### HSM

### MTLS

### Encryption

## Smart Contract

### ERC20

### ERC721 (NFT)

### Uniswap V2/V3 (去中心化交易所核心合约)

## Adoption in Finance/Banking

### Traditional Asset Tokenization

- Collateral Management

- Citi Token Service For Trade

### Digital Asset Custodian

- Internal Project

  > 用来存托比特币和以太坊币
  > 
### Global Payment (liquidity management)

- Citi Token Service For Cash

### On-chain (Bound) Issuance

### KYC / Digital Identity

