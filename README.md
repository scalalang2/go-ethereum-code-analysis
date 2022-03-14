# Geth 코어 코드 분석

[go-ethereum](https://github.com/ethereum/go-ethereum) 코드를 분석한 자료.

먼저, 로우-레벨 기술에 대한 이야기부터 시작한 후 이를 전체적으로 조망하는 순서로 진행해보고자 합니다.

#### 사전 고지
본 문서는 [Go-Ethereum analysis (English Version)](https://github.com/agiletechvn/go-ethereum-code-analysis)문서를 한국어로 재해석한 문서입니다. 중국어 버전이 최초라고 알려져 있습니다.

```
제 목적은 세부 디테일을 모두 알고 싶은게 아니라 
고수준에서 전체 코드 구조를 조망하고 싶기 때문에 
Yello Paper에 있는 복잡한 수식은 모두 제거하고 
쉬운 말 + 제 해석으로 풀어 쓸 계획입니다.
```

## 목차

- [go ethereum code analysis (account, smart contract, logs, etc...)](/go-ethereum-code-analysis.md)
- [rlp, rlpx analysis](/rlp-analysis.md)
- [trie source analysis](/trie-analysis.md)
- [ethdb analysis](/ethdb-analysis.md)
- [rpc analysis](/rpc-analysis.md)
- [p2p analysis](/p2p-analysis.md)
- [eth protocol analysis](/eth-analysis.md)
- **코어 코드 분석**
  - [blockchain index, chain_indexer analysis](/core-chain_indexer-analysis.md)
  - [bloom filter index, bloombits-analysis](/core-bloombits-analysis.md)
  - [ethereum trie, tree management, rollback, state-analysis](/core-state-analysis.md)
  - [transaction processing](/core-state-process-analysis.md)
  - **가상머신 분석**
    - [stack & data structure](/core-vm-stack-memory-analysis.md)
    - [instruction, jump table, interpreter analysis](/core-vm-jumptable-instruction.md)
    - [vm analysis](/core-vm-analysis.md)
  - **트랜잭션 풀 관리**
    - [transaction execution](/core-txlist-data-structure-analysis.md)
    - [transaction pool management](/core-txpool-analysis.md)
  - [genesis block](/core-genesis-analysis.md)
  - [blockchain-analysis](/core-blockchain-analysis.md)
- [miner analysis & CPU mining](/miner-analysis-CPU-mining.md)
- [pow, poa, pos algorithms](/pow-analysis.md)
- [ethereum test network Clique_PoA introduciton](/ethereum-Clique_PoA-introduction.md)
- [swarm, raw & file upload, pss and feed](/ethereum-swarm-introduction.md)
