RLP는 이더리움에서 사용하기 위해 만들어진 인코딩/디코딩 방식입니다. Recursive Length Prefix의 줄임말입니다. RLP의 탄생 배경은 트리 구조와 같은 `nested array 자료구조`를 바이트로 serialization 해서 전송하고, 이를 받는 쪽에서 deserialization해서 풀어쓰는 것을 목표로 합니다.

RLP가 어떻게 동작하는지를 설명할 수도 있지만, 필자의 관심사가 아니기 때문에 단순하게 사용방법만 예시 파일에 설명하고 본 문서는 마치겠습니다.

```go

import "github.com/ethereum/go-ethereum/rlp"

// 인코딩할 2차원 배열 문자열
val := [][]string{
    {"asdf", "qwer", "zxcv"},
    {"asdf", "qwer", "zxcv"},
    {"asdf", "qwer", "zxcv"},
    {"asdf", "qwer", "zxcv"},
    {"asdf", "qwer", "zxcv"},
    {"asdf", "qwer", "zxcv"},
    {"asdf", "qwer", "zxcv"},
}

b := new(bytes.Buffer)
if err := rlp.Encode(b, val); err != nil {
    log.Fatalf("Failed to encode %v to rlp.", val)
}

// 다시 2차원 배열 문자열로 디코딩 합니다.
var decoded [][]string
rlp.Decode(b.Bytes(), &decoded)
log.Printf("%v", decoded) // [[asdf qwer zxcv] [asdf qwer zxcv] [asdf qwer zxcv]]
```