`trie` 패키지니는 머클 패트리샤 트리(MPT)를 구현한 내용이다. MPT 자료구조는 트라이 트리의 변형 버전으로 이더리움에서 계정 정보,컨트랙트 상태 정보, 트랜잭션 정보, 레시피 등을 저장하는데 쓰이는 중요한 자료구조이다. MPT는 트라이, 패트리샤 트라이, 그리고 머클 트리를 결합한 모델이고 각 모델은 아래 별도로 설명할 것이다.

## 트라이 트리
트라이에 대한 설명은 [이 문서](trie-structure.md)로 대체한다.
## 패트리샤 트라이 (Patricia Tries | Prefix tree)
Prefix 트리와 트라이의 차이점은 트라이는 '한 글자'만 노드에 기록하는 반면, Prefix 트리는 공통 prefix로 압축해서 저장하는 방법이다. 트라이 구조를 조금 최적화 했다고 보면된다. 아래 그림은 Trie를 패트리샤 트라이로 바꿔서 저장하는 방법이다.

![Optimization of Tire to Patricia](picture/patricia_tire.png)

![image](picture/trie_2.png)

아래 8 개의 Key/Value 값들이 prefix 트리로 저장된 모습을 보여준다.

| Key              | value |
| ---------------- | ----: |
| 6c0a5c71ec20bq3w | 5     |
| 6c0a5c71ec20CX7j | 27    |
| 6c0a5c71781a1FXq | 18    |
| 6c0a5c71781a9Dog | 64    |
| 6c0a8f743b95zUfe | 30    |
| 6c0a8f743b95jx5R | 2     |
| 6c0a8f740d16y03G | 43    |
| 6c0a8f740d16vcc1 | 48    |

## 머클 트리 ( Merkle tree )
머클 트리는 해시 값을 저장하는 트리이다. 리프 노드는 저장한 데이터를 해시한 값을 가지고 있으며, 상위 노드는 자식 노드의 해시값을 덧 붙인 값의 해시 문자열을 저장한다. `h(c_h1, c_h2)`


![image](picture/trie_3.png)

머클 트리의 주요 기능은, 최상위 노드에 저장된 루트 해시가 트리 전체의 정보를 내포하고 있다는 것이다. 트리에서 어떤 노드가 변경되면 그에 연결된 상위 노드의 해시가 전부 변경된다. 루트 노드의 해시는 블록체인의 블록 헤더에 저장되는데 블록 헤더는 반드시 검증되어야 하는 값이다. 이 뜻은, 블록 헤더만 알고있으면 블록체인 전체 내용을 검증할 수 있다.

## Modified Merkle Patricia Tree (MPT)
이더리움에서 각 블록은 3개의 MPT 트리를 저장하고 있다.

- 트랜잭션 트리
- 레시피 트리 (트랜잭션 처리에 따른 데이터를 저장하는 트리)
- 상태 트리 (컨트랙트 또는 사용자 계정 정보를 저장함)

아래 그림에서는 state root, tx root, reciept root를 저장하고 있는 2개의 블록을 보여준다. 두 번째 블록에서 `account 175`의 값이 27에서 45로 변경된 모습을 보여준다.

이더리움의 MPT는 prefix tree와 비슷하다. 하나의 키/밸류 값을 저장하며 트리 순회 과정에서 리프 노드에 도달하면 값을 조회할 수 있다. 키값은 트리 순회 과정에서 각 브랜치 노드에서 값을 덧붙여 전체 키값을 구한다. MPT 트리는 두개의 최적화 노드가 제공된다.

![world state trie](picture/worldstatetrie.png)

- **(Leaf)** : 리프노드는 두 개의 필드를 가지고 하나는 키 값이고, 두 번째 필드는 값이다.
- **(Extention)** : Extention 노드도 두개의 필드를 가지고 있는데 하나는 패트리샤 트리 처럼 공통 prefix를 압축해서 저장하는 키 값이며, 하나는 자식 노드로 향하는 포인터이다.
- **(Branch):** 브랜치는 17개의 필드를 가지고 있다. 앞 16개의 필드는 각각 16진수의 0~f 까지의 키 값에 해당하는 포인터이고 17번째 필드는 이 노드에 값이 저장되는 경우 저장하는 값이다. 예를 들어 (abc, abd, ab)가 저장되어 있다면 c,d가 포인터를 가지며 17번째 필드의 값은 ab노드의 값이 된다.

### (Hex-Prefix Encoding) - hexadecimal prefix encoding 
Hex-Prefix Encdoing은 니블(4비트)를 바이트 배열에 효율적으로 저장하는 방식이다. 이더리움 계정 주소는 hex값으로 구성되어 있는데 hex 정보를 저장하는데는 니블만 있으면 충분하기 때문에 바이트 단위로 저장하면 정보 손실이 발생한다. 그래서 hex값을 저장하고 남은 4비트를 다른 방식으로 활용하거나 서로 합쳐서 2개의 hex값을 저장하는 인코딩 방식이다.

엄밀한 정의가 보고싶다면 이더리움 Yello Paper를 참고하면 된다.

## Source implementation  
### trie/encoding.go
`encoding.go`파일은 트라이 트리에서 사용하는 3개의 인코딩 방식을 구현한다. 

- **KEYBYTES encoding** : 두개의 nibble을 하나의 바이트로 합치는 방식이다. 두개의 hex값이 있다면 `bytes[ni+1] << 4 | bytes[ni+2]` 수식으로 8비트로 합쳐서 저장할 수 있다. 
- **HEX encoding** : 앞에 첫 4비트에는 키값을 저장하고 나머지 4비트에는 이곳이 마지막 노드이며 값을 가지고 있다는 플래그인 `0x10` 정보를 저장한다. 이 플래그를 `terminator`라고 부른다.
- **COMPACT encoding** : MPT에서 각 노드의 키값을 압축할 때 사용하는 인코딩 방식이다. 첫 번째 바이트는 메타 정보를 기록한다. 
    * 첫번째 바이트 : 앞 4비트는 플래그를 저장하고 뒤에 4비트에서 마지막 비트는 저장할 키가 홀수인지 짝수인지, 마지막에서 두번째 비트는 해당 노드가 값을 가지고 있는지 저장한다. 
    * 나머지 바이트에는 키값을 HEX encoding 해서 저장한다.

코드 구현체는 인코딩/디코딩 함수를 모두 구현한다.

```go
func hexToCompact(hex []byte) []byte {
	terminator := byte(0)
	if hasTerm(hex) {
		terminator = 1
		hex = hex[:len(hex)-1]
	}
	buf := make([]byte, len(hex)/2+1)
	buf[0] = terminator << 5 // the flag byte
	if len(hex)&1 == 1 {
		buf[0] |= 1 << 4 // odd flag
		buf[0] |= hex[0] // first nibble is contained in the first byte
		hex = hex[1:]
	}
	decodeNibbles(hex, buf[1:])
	return buf
}

func compactToHex(compact []byte) []byte {
	base := keybytesToHex(compact)
	base = base[:len(base)-1]
	// apply terminator flag
	if base[0] >= 2 { // TODO first deletes the end-end flag of the keybytesToHex output, and then adds back to the flag bit t of the first half byte. Operational redundancy
		base = append(base, 16)
	}
	// apply odd flag
	chop := 2 - base[0]&1
	return base[chop:]
}

func keybytesToHex(str []byte) []byte {
	l := len(str)*2 + 1
	var nibbles = make([]byte, l)
	for i, b := range str {
		nibbles[i*2] = b / 16
		nibbles[i*2+1] = b % 16
	}
	nibbles[l-1] = 16
	return nibbles
}

// hexToKeybytes turns hex nibbles into key bytes.
// This can only be used for keys of even length.
func hexToKeybytes(hex []byte) []byte {
	if hasTerm(hex) {
		hex = hex[:len(hex)-1]
	}
	if len(hex)&1 != 0 {
		panic("can't convert hex key of odd length")
	}
	key := make([]byte, (len(hex)+1)/2) // TODO adds 1 to the len(hex) that has been judged to be even, and is invalid +1 logic.
	decodeNibbles(hex, key)
	return key
}

func decodeNibbles(nibbles []byte, bytes []byte) {
	for bi, ni := 0, 0; ni < len(nibbles); bi, ni = bi+1, ni+2 {
		bytes[bi] = nibbles[ni]<<4 | nibbles[ni+1]
	}
}

// prefixLen returns the length of the common prefix of a and b.
func prefixLen(a, b []byte) int {
	var i, length = 0, len(a)
	if len(b) < length {
		length = len(b)
	}
	for ; i < length; i++ {
		if a[i] != b[i] {
			break
		}
	}
	return i
}

// hasTerm returns whether a hex key has the terminator flag.
func hasTerm(s []byte) bool {
	return len(s) > 0 && s[len(s)-1] == 16
}
```

### 데이터 구조
노드 구조체는 총 4개가 정의되어 있다. fullNode는 브랜치 노드에 해당하고 shortNode는 extention 노드를, valueNode는 리프노드를 의미한다. `fullNode`, `shortNode`가 포함하고 있는 구조체중 node에서 hashNode가 캐시에 포함되는 것을 볼 수 있는데 이는 해당 노드의 해시 값을 의미한다. 머클 패트리샤 트리이기 때문에 각 노드는 중간 해시 정보를 가지고 있다.

Go언어의 덕 타이핑 문법 때문에, 이 인터페이스만 보고서는 모르지만 fullNode, shortNode, hashNode, valueNode 모두 node 인터페이스를 구현한 구현체이다.

```go
// trie/node.go
type node interface {
	fstring(string) string
	cache() (hashNode, bool)
}

type (
	fullNode struct {
		Children [17]node // Actual trie node data to encode/decode (needs custom encoder)
		flags    nodeFlag
	}
	shortNode struct {
		Key   []byte
		Val   node
		flags nodeFlag
	}
	hashNode  []byte
	valueNode []byte
)
```

트라이 구조체에서, root는 최상위 루트를 포함하고 있고 db는 KV 저장소 객체이다. 전체 트라이는 항상 KV 데이터베이스에 저장되며, 필요시에 데이터베이스에서 트리 정보를 조회해서 메모리에 로드한다.


```go
// Trie is a Merkle Patricia Trie.
// The zero value is an empty trie with no database.
// Use New to create a trie that sits on top of a database.
//
// Trie is not safe for concurrent use.
type Trie struct {
	db   *Database
	root node
	// Keep track of the number leafs which have been inserted since the last
	// hashing operation. This number will not directly map to the number of
	// actually unhashed nodes
	unhashed int
}
```

### Insert, find and delete Trie trees  
트라이는 New 함수에 의해 초기화 된다. 해시값을 첫번째, 데이터베이스 객체를 두번째 필드로 입력받는다. 만약 해시값이 null이 아니라면 데이터베이스에서 조회해서 로드하고, 아니라면 비어있는 새로운 트라이를 생성한다.

```go
func New(root common.Hash, db *Database) (*Trie, error) {
	if db == nil {
		panic("trie.New called without a database")
	}
	trie := &Trie{
		db: db,
	}
	if root != (common.Hash{}) && root != emptyRoot {
		rootnode, err := trie.resolveHash(root[:], nil)
		if err != nil {
			return nil, err
		}
		trie.root = rootnode
	}
	return trie, nil
}

```

`insert` 함수는 트라이에 노드를 추가하는 함수이다. 루트 노드에서 부터 시작하는 재귀함수이기 때문에 맨 처음에 종료 조건으로 더 이상 key값이 없으면 값을 그대로 리턴한다. 함수 파라미터 `node`는 현재 삽입된 노드이다. `prefix bytes[]`는 현재까지 읽어들인 문자열이고 `key bytes[]`는 아직 처리하지 못한 문자열이다. 마지막으로 `value`는 추가해야하는 값을 의미한다. 리턴값 첫 번째는 trie를 변경시켰는지 여부이고 두번 째 node는 삽입이 완료되고 나서 서브 트리의 루트를 말한다. 마지막은 에러를 나타낸다.

```go
// trie/trie.go

func (t *Trie) insert(n node, prefix, key []byte, value node) (bool, node, error) {
	if len(key) == 0 {
		if v, ok := n.(valueNode); ok {
			return !bytes.Equal(v, value.(valueNode)), value, nil
		}
		return true, value, nil
	}
	switch n := n.(type) {
	case *shortNode:
		matchlen := prefixLen(key, n.Key)
		// If the whole key matches, keep this short node as is
		// and only update the value.
		if matchlen == len(n.Key) {
			dirty, nn, err := t.insert(n.Val, append(prefix, key[:matchlen]...), key[matchlen:], value)
			if !dirty || err != nil {
				return false, n, err
			}
			return true, &shortNode{n.Key, nn, t.newFlag()}, nil
		}
		// Otherwise branch out at the index where they differ.
		branch := &fullNode{flags: t.newFlag()}
		var err error
		_, branch.Children[n.Key[matchlen]], err = t.insert(nil, append(prefix, n.Key[:matchlen+1]...), n.Key[matchlen+1:], n.Val)
		if err != nil {
			return false, nil, err
		}
		_, branch.Children[key[matchlen]], err = t.insert(nil, append(prefix, key[:matchlen+1]...), key[matchlen+1:], value)
		if err != nil {
			return false, nil, err
		}
		// Replace this shortNode with the branch if it occurs at index 0.
		if matchlen == 0 {
			return true, branch, nil
		}
		// Otherwise, replace it with a short node leading up to the branch.
		return true, &shortNode{key[:matchlen], branch, t.newFlag()}, nil

	case *fullNode:
		dirty, nn, err := t.insert(n.Children[key[0]], append(prefix, key[0]), key[1:], value)
		if !dirty || err != nil {
			return false, n, err
		}
		n = n.copy()
		n.flags = t.newFlag()
		n.Children[key[0]] = nn
		return true, n, nil

	case nil:
		return true, &shortNode{key, value, t.newFlag()}, nil

	case hashNode:
		// We've hit a part of the trie that isn't loaded yet. Load
		// the node and insert into it. This leaves all child nodes on
		// the path to the value in the trie.
		rn, err := t.resolveHash(n, prefix)
		if err != nil {
			return false, nil, err
		}
		dirty, nn, err := t.insert(rn, prefix, key, value)
		if !dirty || err != nil {
			return false, rn, err
		}
		return true, nn, nil

	default:
		panic(fmt.Sprintf("%T: invalid node: %v", n, n))
	}
}
```

`node`의 타입이 nil인 경우 전체 트리가 존재하지 않는다. 그래서 shortNode{key, value, t.newFlag()}를 즉시 리턴한다.

만약 현재 루트 노드가 `shortNode`라면, 먼저 전체 키과 현재 키가 동일한 부분인 matchlen을 찾는다. 만약 키가 동일하다면 현재 `shortNode`를 그대로 두고 값만 변경한다.

```go
matchlen := prefixLen(key, n.Key)
// If the whole key matches, keep this short node as is
// and only update the value.
if matchlen == len(n.Key) {
    dirty, nn, err := t.insert(n.Val, append(prefix, key[:matchlen]...), key[matchlen:], value)
    if !dirty || err != nil {
        return false, n, err
    }
    return true, &shortNode{n.Key, nn, t.newFlag()}, nil
}
```

키가 완벽하게 동일한 경우가 아니라면 브랜치 노드를 하나 만들어서 확장한다. 현재 `shortNode`밑으로 확장한다. 단, 첫글자만 동일한 경우에는 현재 노드를 브랜치 노드로 변경한다.

```go
// Otherwise branch out at the index where they differ.
branch := &fullNode{flags: t.newFlag()}
var err error

// 기존 노드의 키값 중 동일한 부분을 자른 나머지 부분
_, branch.Children[n.Key[matchlen]], err = t.insert(nil, append(prefix, n.Key[:matchlen+1]...), n.Key[matchlen+1:], n.Val)
if err != nil {
    return false, nil, err
}

// 삽입하려는 노드의 키값 중 동일한 부분을 자른 나머지 부분
_, branch.Children[key[matchlen]], err = t.insert(nil, append(prefix, key[:matchlen+1]...), key[matchlen+1:], value)
if err != nil {
    return false, nil, err
}

// 만약 첫 글자만 동일한 경우라면, 이 노드를 브랜치로 변경한다.
if matchlen == 0 {
    return true, branch, nil
}
// Otherwise, replace it with a short node leading up to the branch.
return true, &shortNode{key[:matchlen], branch, t.newFlag()}, nil
```

현재 노드가 `fullNode (a branch node)`라면, 첫글자 이후에 해당하는 키로만 노드를 하나 만들고 `Children[key[0]]`에 추가한다.

```go
case *fullNode:
    dirty, nn, err := t.insert(n.Children[key[0]], append(prefix, key[0]), key[1:], value)
    if !dirty || err != nil {
        return false, n, err
    }
    n = n.copy()
    n.flags = t.newFlag()
    n.Children[key[0]] = nn
    return true, n, nil
```

`hashNode`는 아직 데이터베이스에서 로드되지 않은 상태를 의미한다. 만약 현재 노드가 hashNode라면 데이터베이스에서 `prefix`를 Key값에 해당하는 노드를 찾은 다음 해당 노드에 추가한다. Trie 객체에 등록된 데이터베이스는 Key/Value 데이터베이스이고 트리의 상위 노드부터 자식까지 모든 노드가 KV 저장소에 저장됨에 유의하자.

`hashNode`에서 hash는 머클 트리 상에서 자식 노드의 해시를 계속 누적해서 계산한 값을 의미한다. 그래서 `trie/hasher.go`에서 트리를 순회하면서 해시하는 Hasher라는 인터페이스를 제공한다.

```go
case hashNode:
    // We've hit a part of the trie that isn't loaded yet. Load
    // the node and insert into it. This leaves all child nodes on
    // the path to the value in the trie.
    rn, err := t.resolveHash(n, prefix)
    if err != nil {
        return false, nil, err
    }
    dirty, nn, err := t.insert(rn, prefix, key, value)
    if !dirty || err != nil {
        return false, rn, err
    }
    return true, nn, nil
```

`insert`함수를 이해했다면 Get 조회 함수를 이해하는 건 쉬워진다. 풀노드, 숏노드를 순회하면서 valueNode를 찾아서 리턴한다. 찾고자 하는게 `valueNode`이기 때문에 재귀함수의 종료 조건이 된다.

```go
func (t *Trie) tryGet(origNode node, key []byte, pos int) (value []byte, newnode node, didResolve bool, err error) {
	switch n := (origNode).(type) {
	case nil:
		return nil, nil, false, nil
	case valueNode:
		return n, n, false, nil
	case *shortNode:
		if len(key)-pos < len(n.Key) || !bytes.Equal(n.Key, key[pos:pos+len(n.Key)]) {
			// key not found in trie
			return nil, n, false, nil
		}
		value, newnode, didResolve, err = t.tryGet(n.Val, key, pos+len(n.Key))
		if err == nil && didResolve {
			n = n.copy()
			n.Val = newnode
		}
		return value, n, didResolve, err
	case *fullNode:
		value, newnode, didResolve, err = t.tryGet(n.Children[key[pos]], key, pos+1)
		if err == nil && didResolve {
			n = n.copy()
			n.Children[key[pos]] = newnode
		}
		return value, n, didResolve, err
	case hashNode:
		child, err := t.resolveHash(n, key[:pos])
		if err != nil {
			return nil, n, true, err
		}
		value, newnode, _, err := t.tryGet(child, key, pos)
		return value, newnode, true, err
	default:
		panic(fmt.Sprintf("%T: invalid node: %v", origNode, origNode))
	}
}
```

초기 이더리움에는 없었는데, 트라이 노드 중 인자 `key`에 해당하는 노드를 지우는 `Delete`함수가 추가되었다. 아직 깊게 보진 못했지만 뇌피셜로 생각해보자면 상태 트라이 크기를 줄이기 위한 pruning을 구현하기 위해 필요했던 것 같다.

### 데이터베이스에 커밋하기

커밋은 현재 트라이의 모든 노드와 해시 값을 데이터베이스에 저장하는 과정이다.

```go
// Commit writes all nodes to the trie's memory database, tracking the internal
// and external (for account tries) references.
func (t *Trie) Commit(onleaf LeafCallback) (root common.Hash, err error) {
	if t.db == nil {
		panic("commit called on trie with nil database")
	}
	if t.root == nil {
		return emptyRoot, nil
	}
	// Derive the hash for all dirty nodes first. We hold the assumption
	// in the following procedure that all nodes are hashed.
	rootHash := t.Hash()
	h := newCommitter()
	defer returnCommitterToPool(h)

	// Do a quick check if we really need to commit, before we spin
	// up goroutines. This can happen e.g. if we load a trie for reading storage
	// values, but don't write to it.
	if _, dirty := t.root.cache(); !dirty {
		return rootHash, nil
	}
	var wg sync.WaitGroup
	if onleaf != nil {
		h.onleaf = onleaf
		h.leafCh = make(chan *leaf, leafChanSize)
		wg.Add(1)
		go func() {
			defer wg.Done()
			h.commitLoop(t.db)
		}()
	}
	var newRoot hashNode
	newRoot, err = h.Commit(t.root, t.db)
	if onleaf != nil {
		// The leafch is created in newCommitter if there was an onleaf callback
		// provided. The commitLoop only _reads_ from it, and the commit
		// operation was the sole writer. Therefore, it's safe to close this
		// channel here.
		close(h.leafCh)
		wg.Wait()
	}
	if err != nil {
		return common.Hash{}, err
	}
	t.root = newRoot
	return rootHash, nil
}
```

먼저 데이터베이스에 커밋하기 전에 모든 노드에 걸쳐서 해시를 계산한다. 이 계산 로직은 `trie/hasher.go`파일에 저장되어 있으며, 트라이의 Hash()함수가 이를 호출하는 구조를 가지고 있다. `committer`객체는 실제 데이터베이스에 기록하는 책임을 가지는 객체이다.

```go
// Derive the hash for all dirty nodes first. We hold the assumption
// in the following procedure that all nodes are hashed.
rootHash := t.Hash()
h := newCommitter()
defer returnCommitterToPool(h)
```

만약 루트 캐시에서 변경된 내용이 없으면, 데이터베이스에 기록할 필요가 없으므로 루트 해시를 그대로 리턴한다.

반대의 경우에는 커밋을 실행하는데, 이 때 리프노드를 데이터베이스에 저장하는 건 병렬로 처리할 수 있으므로 고루틴으로 별도 처리한다. 함수가 조기 종료되는 것을 막기 위해 `WaitGroup`으로 고루틴을 블럭킹 하고 있다.
```go
if _, dirty := t.root.cache(); !dirty {
    return rootHash, nil
}
var wg sync.WaitGroup
if onleaf != nil {
    h.onleaf = onleaf
    h.leafCh = make(chan *leaf, leafChanSize)
    wg.Add(1)
    go func() {
        defer wg.Done()
        h.commitLoop(t.db)
    }()
}
```

Commit 함수를 통해 실제 데이터베이스에 기록하는 커밋을 수행한다. `Commit`함수 내부에서는 트리를 순회하면서 모든 노드를 DB에 커밋한다. 

이더리움에서 MPT는 굉장히 무거운 자료구조이다. 최대 개수가 `2^64`개 이기 때문에 메모리에 없는 서브 트리는 굳이 순회하지 않는 다던가 메모리에 변경사항이 없는 노드는 스킵하는 등 몇 가지 최적화 방법이 적용되어 있다.

```go
newRoot, err = h.Commit(t.root, t.db)
if onleaf != nil {
    // The leafch is created in newCommitter if there was an onleaf callback
    // provided. The commitLoop only _reads_ from it, and the commit
    // operation was the sole writer. Therefore, it's safe to close this
    // channel here.
    close(h.leafCh)
    wg.Wait()
}
if err != nil {
    return common.Hash{}, err
}
t.root = newRoot
return rootHash, nil
```
```go
// trie/committer.go

// commit collapses a node down into a hash node and inserts it into the database
func (c *committer) commit(n node, db *Database) (node, error) {
	// 변경 내용이 없으면 스킵한다.
	hash, dirty := n.cache()
	if hash != nil && !dirty {
		return hash, nil
	}

	// 커밋하고 더티 플래그를 제거한다.
	switch cn := n.(type) {
	case *shortNode:
		// Commit child
		collapsed := cn.copy()

		// If the child is fullnode, recursively commit.
		// Otherwise it can only be hashNode or valueNode.
		if _, ok := cn.Val.(*fullNode); ok {
			childV, err := c.commit(cn.Val, db)
			if err != nil {
				return nil, err
			}
			collapsed.Val = childV
		}

		// Compact 인코딩을 수행한 뒤 저장한다.
		collapsed.Key = hexToCompact(cn.Key)
		hashedNode := c.store(collapsed, db)
		if hn, ok := hashedNode.(hashNode); ok {
			return hn, nil
		}
		return collapsed, nil
	case *fullNode:
		hashedKids, err := c.commitChildren(cn, db)
		if err != nil {
			return nil, err
		}
		collapsed := cn.copy()
		collapsed.Children = hashedKids

		hashedNode := c.store(collapsed, db)
		if hn, ok := hashedNode.(hashNode); ok {
			return hn, nil
		}
		return collapsed, nil
	case hashNode:
		return cn, nil
	default:
		// nil, valuenode shouldn't be committed
		panic(fmt.Sprintf("%T: invalid node: %v", n, n))
	}
}
```

### Proof.go Merkel proof of the Trie tree
There are two main methods. The Prove method obtains the proof of the specified Key. The proof is a list of hash values ​​for all nodes from the root node to the leaf node. The VerifyProof method accepts a roothash value and proof and key to verify that the key exists.  

Prove method, starting from the root node. Store the hash values ​​of the passed nodes one by one into the list. Then return.
	
```go
// Prove constructs a merkle proof for key. The result contains all
// encoded nodes on the path to the value at key. The value itself is
// also included in the last node and can be retrieved by verifying
// the proof.
//
// If the trie does not contain a value for key, the returned proof
// contains all nodes of the longest existing prefix of the key
// (at least the root node), ending with the node that proves the
// absence of the key.
func (t *Trie) Prove(key []byte) []rlp.RawValue {
	// Collect all nodes on the path to key.
	key = keybytesToHex(key)
	nodes := []node{}
	tn := t.root
	for len(key) > 0 && tn != nil {
		switch n := tn.(type) {
		case *shortNode:
			if len(key) < len(n.Key) || !bytes.Equal(n.Key, key[:len(n.Key)]) {
				// The trie doesn't contain the key.
				tn = nil
			} else {
				tn = n.Val
				key = key[len(n.Key):]
			}
			nodes = append(nodes, n)
		case *fullNode:
			tn = n.Children[key[0]]
			key = key[1:]
			nodes = append(nodes, n)
		case hashNode:
			var err error
			tn, err = t.resolveHash(n, nil)
			if err != nil {
				log.Error(fmt.Sprintf("Unhandled trie error: %v", err))
				return nil
			}
		default:
			panic(fmt.Sprintf("%T: invalid node: %v", tn, tn))
		}
	}
	hasher := newHasher(0, 0)
	proof := make([]rlp.RawValue, 0, len(nodes))
	for i, n := range nodes {
		// Don't bother checking for errors here since hasher panics
		// if encoding doesn't work and we're not writing to any database.
		n, _, _ = hasher.hashChildren(n, nil)
		hn, _ := hasher.store(n, nil, false)
		if _, ok := hn.(hashNode); ok || i == 0 {
			// If the node's database encoding is a hash (or is the
			// root node), it becomes a proof element.
			enc, _ := rlp.EncodeToBytes(n)
			proof = append(proof, enc)
		}
	}
	return proof
}
```

The VerifyProof method receives a rootHash parameter, a key parameter, and a proof array, to verify whether it can correspond to the database.


```go
// VerifyProof checks merkle proofs. The given proof must contain the
// value for key in a trie with the given root hash. VerifyProof
// returns an error if the proof contains invalid trie nodes or the
// wrong value.
func VerifyProof(rootHash common.Hash, key []byte, proof []rlp.RawValue) (value []byte, err error) {
	key = keybytesToHex(key)
	sha := sha3.NewKeccak256()
	wantHash := rootHash.Bytes()
	for i, buf := range proof {
		sha.Reset()
		sha.Write(buf)
		if !bytes.Equal(sha.Sum(nil), wantHash) {
			return nil, fmt.Errorf("bad proof node %d: hash mismatch", i)
		}
		n, err := decodeNode(wantHash, buf, 0)
		if err != nil {
			return nil, fmt.Errorf("bad proof node %d: %v", i, err)
		}
		keyrest, cld := get(n, key)
		switch cld := cld.(type) {
		case nil:
			if i != len(proof)-1 {
				return nil, fmt.Errorf("key mismatch at proof node %d", i)
			} else {
				// The trie doesn't contain the key.
				return nil, nil
			}
		case hashNode:
			key = keyrest
			wantHash = cld
		case valueNode:
			if i != len(proof)-1 {
				return nil, errors.New("additional nodes at end of proof")
			}
			return cld, nil
		}
	}
	return nil, errors.New("unexpected end of proof")
}

func get(tn node, key []byte) ([]byte, node) {
	for {
		switch n := tn.(type) {
		case *shortNode:
			if len(key) < len(n.Key) || !bytes.Equal(n.Key, key[:len(n.Key)]) {
				return nil, nil
			}
			tn = n.Val
			key = key[len(n.Key):]
		case *fullNode:
			tn = n.Children[key[0]]
			key = key[1:]
		case hashNode:
			return key, n
		case nil:
			return key, nil
		case valueNode:
			return nil, n
		default:
			panic(fmt.Sprintf("%T: invalid node: %v", tn, tn))
		}
	}
}
```

### Security_trie.go Encrypted Trie
In order to avoid the deliberate use of very long keys, the access time increases, security_trie wraps the trie tree, and all keys are converted to the hash value calculated by the keccak256 algorithm. At the same time, the original key corresponding to the hash value is stored in the database.


```go
type SecureTrie struct {
	trie             Trie    
	hashKeyBuf       [secureKeyLength]byte   
	secKeyBuf        [200]byte               // The database prefix when the key corresponding to the hash value is stored
	secKeyCache      map[string][]byte      
	secKeyCacheOwner *SecureTrie // Pointer to self, replace the key cache on mismatch
}

func NewSecure(root common.Hash, db Database, cachelimit uint16) (*SecureTrie, error) {
	if db == nil {
		panic("NewSecure called with nil database")
	}
	trie, err := New(root, db)
	if err != nil {
		return nil, err
	}
	trie.SetCacheLimit(cachelimit)
	return &SecureTrie{trie: *trie}, nil
}

// Get returns the value for key stored in the trie.
// The value bytes must not be modified by the caller.
func (t *SecureTrie) Get(key []byte) []byte {
	res, err := t.TryGet(key)
	if err != nil {
		log.Error(fmt.Sprintf("Unhandled trie error: %v", err))
	}
	return res
}

// TryGet returns the value for key stored in the trie.
// The value bytes must not be modified by the caller.
// If a node was not found in the database, a MissingNodeError is returned.
func (t *SecureTrie) TryGet(key []byte) ([]byte, error) {
	return t.trie.TryGet(t.hashKey(key))
}
func (t *SecureTrie) CommitTo(db DatabaseWriter) (root common.Hash, err error) {
	if len(t.getSecKeyCache()) > 0 {
		for hk, key := range t.secKeyCache {
			if err := db.Put(t.secKey([]byte(hk)), key); err != nil {
				return common.Hash{}, err
			}
		}
		t.secKeyCache = make(map[string][]byte)
	}
	return t.trie.CommitTo(db)
}
```