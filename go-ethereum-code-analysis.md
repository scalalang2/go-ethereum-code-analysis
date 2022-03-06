## Go ethereum 디렉토리 구조

Geth 프로젝트의 디렉토리 구조는 각각의 모듈이 별도로 구현됩니다. 각 모듈은 하나의 독립적으로 동작하는 기능으로 나중에 이를 하나로 조합하여 사용됩니다. 각 디렉토리는 Go언어로 작성된 패키지 이며 여기서는 각 모듈별로 제공하는 기능을 대략적으로 기술합니다.

    accounts        	이더리움 계정의 추상화된 코드를 제공함
    build			Mainly some scripts and configurations compiled and built
    cmd			A lot of command line tools, one by one
    	/abigen		Source code generator to convert Ethereum contract definitions into easy to use, compile-time type-safe Go packages
    	/bootnode	Start a node that only implements network discovery
    	/evm		Ethereum virtual machine development tool to provide a configurable, isolated code debugging environment
    	/faucet
    	/geth		Ethereum command line client, the most important tool
    	/p2psim		Provides a tool to simulate the http API
    	/puppeth	Create a new Ethereum network wizard, such as Clique POA consensus
    	/rlpdump 	Provides a formatted output of RLP data
    	/swarm		Swarm network utils
    	/util		Provide some public tools
    	/wnode		This is a simple Whisper node. It can be used as a standalone boot node. In addition, it can be used for different testing and diagnostic purposes.
    common			Provide some public tools
    compression		Package rle implements the run-length encoding used for Ethereum data.
    consensus		Provide some consensus algorithms from Ethereum, such as ethhash, clique(proof-of-authority)
    console			console package
    contracts		Smart contracts deployed in genesis block, such as checkqueue, DAO...
    core			Ethereum's core data structures and algorithms (virtual machines, states, blockchains, Bloom filters)
    crypto			Encryption and hash algorithms
    eth			Implemented the consensus of Ethereum
    ethclient		Provides RPC client for Ethereum
    ethdb			Eth's database (including the actual use of leveldb and the in-memory database for testing)
    ethstats		Provide a report on the status of the network
    event			Handling real-time events
    les			Implemented a lightweight server of Ethereum
    light			Achieve on-demand retrieval for Ethereum lightweight clients
    log			Provide log information that is friendly to humans and computers
    metrics			Provide disk counter, publish to grafana for sample
    miner			Provide block creation and mining in Ethereum
    mobile			Some wrappers used by the mobile side
    node			Ethereum's various types of nodes
    p2p			Ethereum p2p network protocol
    rlp			Ethereum serialization, called recursive length prefix
    rpc			Remote method call, used in APIs and services
    swarm			Swarm network processing
    tests			testing purposes
    trie			Ethereum's important data structure package: trie implements Merkle Patricia Tries.
    whisper			A protocol for the whisper node is provided.

It can be seen that the code of Ethereum is still quite large, but roughly speaking, the code structure is still quite good. I hope to analyze from some relatively independent modules. Then delve into the internal code. The focus may be on modules such as p2p networks that are not covered in the Yellow Book.
