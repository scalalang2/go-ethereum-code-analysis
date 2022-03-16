Go-ethereum의 모든 데이터는 구글에서 개발한 오픈소스 KV 데이터베이스인 LevelDB에 저장된다. 블록체인의 모든 정보는 LevelDB에 저장되는데, LevelDB는 파일 크기에 따라 파일을 쪼개서 저장한다. 그래서 우리가 블록체인의 데이터가 저장된 곳을 보면 여러개의 크기가 작은 파일이 존재하는데 전체가 하나의 LevelDB 인스턴스를 구성한다.

공식 웹 사이트에서 소개하는 LevelDB의 특징은 다음과 같다.

**특징**：

- 키와 밸류는 임의 길이의 바이트 배열이다.
- KV 페어 쌍은 Key를 기준으로 사전순으로 저장된다.
- Put(), Delete(), Get(), Batch()의 기본 기능을 제공한다.
- Batch 연산의 아토믹 연산을 지원한다.
- 데이터 스냅샷 기능을 지원한다.
- iterator를 통한 데이터 탐색 기능 지원
- 데이터 압축에 Snappy를 지원한다.
- FileDB 이기 때문에 다른 환경으로 포팅이 쉽다.

**한계**：

- NOSQL 데이터 모델, 인덱싱 미지원, SQL 문법 미지원
- 하나의 프로세스만이 데이터베이스 접근 가능하다.
- Built-in 클라이언트/서버 아키텍처가 없다.

`ethereum/ethdb` 패키지에 Key/Value 소스 코드가 저장되어 있다. 코드 자체는 상대적으로 쉽고 3개의 파일에 인터페이스가 저장되고 그 하위 패키지에 실제 구현부를 저장하는 식으로 구성되어 있다.

- `database.go` : 데이터베이스 객체의 인터페이스
- `iterator.go` : 데이터 순회 기능을 제공하는 인터페이스
- `batch.go` : Batch 연산 기능을 제공하는 인터페이스
- `ethdb/leveldb/` : LevelDB 구현체
- `ethdb/memorydb/` : In-memory KV 저장소 구현체

## database.go
아래 코드를 보면 Database 인터페이스 안에, 여러 인터페이스가 임베딩 되어 있는 상황인 것을 알 수 있다. 여기서 기본적인 연산을 하는 `Reader`와 `Writer`를 보면 다시 KeyValueReader와 KeyValueWriter를 임베딩 한 모습을 볼 수 있다.

```go
package ethdb

type Reader interface {
	KeyValueReader
	AncientReader
}

type Writer interface {
	KeyValueWriter
	AncientWriter
}

type Database interface {
	Reader
	Writer
	Batcher
	Iteratee
	Stater
	Compacter
	io.Closer
}
```

`KeyValueReader`와 `KeyValueWriter`에는 각각 Has, Get, Put, Delete 함수가 정의 되어 있다. 즉 Database 인터페이스를 구현하는 구조체는 위 4가지 함수를 제공해야 한다.

```go
// KeyValueReader wraps the Has and Get method of a backing data store.
type KeyValueReader interface {
	// Has retrieves if a key is present in the key-value data store.
	Has(key []byte) (bool, error)

	// Get retrieves the given key if it's present in the key-value data store.
	Get(key []byte) ([]byte, error)
}

// KeyValueWriter wraps the Put method of a backing data store.
type KeyValueWriter interface {
	// Put inserts the given value into the key-value data store.
	Put(key []byte, value []byte) error

	// Delete removes the key from the key-value data store.
	Delete(key []byte) error
}

```

## ethdb/memorydb/memorydb.go

가장 기본이 되는 메모리 DB 부터 살펴보자, 사실 LevelDB자체는 오픈소스를 그대로 이용하고 있기 때문에 단순히 우리가 여기서 정의한 인터페이스대로 기능을 제공하는 것 밖에 없다.

db 변수의 타입이 `map[string][]byte` 로 정의되어 있어서, string을 키로 가지고 []byte를 값으로 가지는 데이터 구조임을 알 수 있다. `lock`은 동시성을 해결하기 위한 락으로 사용된다.

```go
type Database struct {
	db   map[string][]byte
	lock sync.RWMutex
}

// key-value 에서 key를 가진 값이 있는지 검사한다.
func (db *Database) Has(key []byte) (bool, error) {
	db.lock.RLock()
	defer db.lock.RUnlock()

	if db.db == nil {
		return false, errMemorydbClosed
	}
	_, ok := db.db[string(key)]
	return ok, nil
}

// key가 존재한다면 값을 리턴한다.
func (db *Database) Get(key []byte) ([]byte, error) {
	db.lock.RLock()
	defer db.lock.RUnlock()

	if db.db == nil {
		return nil, errMemorydbClosed
	}
	if entry, ok := db.db[string(key)]; ok {
		return common.CopyBytes(entry), nil
	}
	return nil, errMemorydbNotFound
}

// key를 가진 value를 저장한다.
func (db *Database) Put(key []byte, value []byte) error {
	db.lock.Lock()
	defer db.lock.Unlock()

	if db.db == nil {
		return errMemorydbClosed
	}
	db.db[string(key)] = common.CopyBytes(value)
	return nil
}

// 해당 key를 삭제한다.
func (db *Database) Delete(key []byte) error {
	db.lock.Lock()
	defer db.lock.Unlock()

	if db.db == nil {
		return errMemorydbClosed
	}
	delete(db.db, string(key))
	return nil
}
```

코드에는 배치 연산을 위한 구조체와 함수도 존재하며, 상당히 쉽게 구현되어 있다. Put 연산으로 요청 자체를 저장하고 있다가 Write 함수를 통해 db에 한꺼번에 쓴다.

```go
type batch struct {
	db     *Database
	writes []keyvalue
	size   int
}

func (b *batch) Put(key, value []byte) error {
    // 메모리에만 정보를 가지고 있는다.
	b.writes = append(b.writes, keyvalue{common.CopyBytes(key), common.CopyBytes(value), false})
	b.size += len(value)
	return nil
}

// 실제 db에 한꺼번에 저장한다.
func (b *batch) Write() error {
	b.db.lock.Lock()
	defer b.db.lock.Unlock()

	for _, keyvalue := range b.writes {
		if keyvalue.delete {
			delete(b.db.db, string(keyvalue.key))
			continue
		}
		b.db.db[string(keyvalue.key)] = keyvalue.value
	}
	return nil
}
```

## ethdb/leveldb/leveldb.go

아래 코드가 실제 이더리움 클라이언트에서 사용중인 코드이다. 실질적으로는 syndtr/goleveldb/leveldb의 구현부를 인터페이스를 통해 제공하는 정도로 구현되어 있다.

```go
package leveldb

import (
	"fmt"
	"strconv"
	"strings"
	"sync"
	"time"

	"github.com/ethereum/go-ethereum/common"
	"github.com/ethereum/go-ethereum/ethdb"
	"github.com/ethereum/go-ethereum/log"
	"github.com/ethereum/go-ethereum/metrics"
	"github.com/syndtr/goleveldb/leveldb"
	"github.com/syndtr/goleveldb/leveldb/errors"
	"github.com/syndtr/goleveldb/leveldb/filter"
	"github.com/syndtr/goleveldb/leveldb/opt"
	"github.com/syndtr/goleveldb/leveldb/util"
)
```

Database 구조체를 보면 in-memory 보다 조금 많은 항목들이 등록되어 있다. 대 다수는 이더리움 성능에 크게 영향을 미치는 요소 중 하나인 leveldb의 성능을 측정하기 위한 Meter 값이 많이 들어가 있다.

```go
type Database struct {
	fn string      // 파일명
	db *leveldb.DB // 오픈소스 LevelDB의 인터페이스

	compTimeMeter      metrics.Meter 
	compReadMeter      metrics.Meter 

	quitLock sync.Mutex      
	quitChan chan chan error 

	log log.Logger 
}
```

Put과 Has, Delete 함수의 구현 내용을 한 번 보자. 실질적으로 세부 구현 내용은 오픈소스 leveldb에 구현되어 있기 때문에 이를 호출하는 것에 불과한데, 오픈소스 자체에 멀티 쓰레드 접근이 구현되어 있기 때문에 메모리 DB에서 구현했던 것처럼 mutex.RWLock 을 이용할 필요가 없다.

```go
// Has retrieves if a key is present in the key-value store.
func (db *Database) Has(key []byte) (bool, error) {
	return db.db.Has(key, nil)
}

// Get retrieves the given key if it's present in the key-value store.
func (db *Database) Get(key []byte) ([]byte, error) {
	dat, err := db.db.Get(key, nil)
	if err != nil {
		return nil, err
	}
	return dat, nil
}

// Put inserts the given value into the key-value store.
func (db *Database) Put(key []byte, value []byte) error {
	return db.db.Put(key, value, nil)
}

// Delete removes the key from the key-value store.
func (db *Database) Delete(key []byte) error {
	return db.db.Delete(key, nil)
}
```

### Metrics processing

LevelDB의 메트릭을 수집하는 내용이다. 기본적으로 Database 구조체를 생성할 때 meter 정보를 포함해서 생성하고 별도 고루틴을 생성해서 동작시키는 것을 알 수 있다.

**Meter는 메트릭 별로 정보를 분리하기 위해 사용하고, quitChan은 별도로 실행한 고루틴을 안전하게 종료하기 위해서 사용하는 채널이다.

```go
ldb := &Database{
    fn:       file,
    db:       db,
    log:      logger,
    quitChan: make(chan chan error),
}
ldb.compTimeMeter = metrics.NewRegisteredMeter(namespace+"compact/time", nil)
ldb.compReadMeter = metrics.NewRegisteredMeter(namespace+"compact/input", nil)
ldb.compWriteMeter = metrics.NewRegisteredMeter(namespace+"compact/output", nil)
ldb.diskSizeGauge = metrics.NewRegisteredGauge(namespace+"disk/size", nil)
ldb.diskReadMeter = metrics.NewRegisteredMeter(namespace+"disk/read", nil)
ldb.diskWriteMeter = metrics.NewRegisteredMeter(namespace+"disk/write", nil)
ldb.writeDelayMeter = metrics.NewRegisteredMeter(namespace+"compact/writedelay/duration", nil)
ldb.writeDelayNMeter = metrics.NewRegisteredMeter(namespace+"compact/writedelay/counter", nil)
ldb.memCompGauge = metrics.NewRegisteredGauge(namespace+"compact/memory", nil)
ldb.level0CompGauge = metrics.NewRegisteredGauge(namespace+"compact/level0", nil)
ldb.nonlevel0CompGauge = metrics.NewRegisteredGauge(namespace+"compact/nonlevel0", nil)
ldb.seekCompGauge = metrics.NewRegisteredGauge(namespace+"compact/seek", nil)

// Start up the metrics gathering and return
go ldb.meter(metricsGatheringInterval)
return ldb, nil
```

meter 함수는 매 3초마다 실행하고 메트릭 시스템으로 수집한 정보를 전송하는 기능을 가지고 있다. 이 함수는 db 객체에서 Close 함수가 실행될 때까지 수행한다.

```go
// meter periodically retrieves internal leveldb counters and reports them to
// the metrics subsystem.
//
// This is how a LevelDB stats table looks like (currently):
//   Compactions
//    Level |   Tables   |    Size(MB)   |    Time(sec)  |    Read(MB)   |   Write(MB)
//   -------+------------+---------------+---------------+---------------+---------------
//      0   |          0 |       0.00000 |       1.27969 |       0.00000 |      12.31098
//      1   |         85 |     109.27913 |      28.09293 |     213.92493 |     214.26294
//      2   |        523 |    1000.37159 |       7.26059 |      66.86342 |      66.77884
//      3   |        570 |    1113.18458 |       0.00000 |       0.00000 |       0.00000
//
// This is how the write delay look like (currently):
// DelayN:5 Delay:406.604657ms Paused: false
//
// This is how the iostats look like (currently):
// Read(MB):3895.04860 Write(MB):3654.64712
func (db *Database) meter(refresh time.Duration) {
	// Create the counters to store current and previous compaction values
	compactions := make([][]float64, 2)
	for i := 0; i < 2; i++ {
		compactions[i] = make([]float64, 4)
	}
	// Create storage for iostats.
	var iostats [2]float64

	// Create storage and warning log tracer for write delay.
	var (
		delaystats      [2]int64
		lastWritePaused time.Time
	)

	var (
		errc chan error
		merr error
	)

	timer := time.NewTimer(refresh)
	defer timer.Stop()

	// Iterate ad infinitum and collect the stats
	for i := 1; errc == nil && merr == nil; i++ {
		// Retrieve the database stats
		stats, err := db.db.GetProperty("leveldb.stats")
		if err != nil {
			db.log.Error("Failed to read database stats", "err", err)
			merr = err
			continue
		}
		// Find the compaction table, skip the header
		lines := strings.Split(stats, "\n")
		for len(lines) > 0 && strings.TrimSpace(lines[0]) != "Compactions" {
			lines = lines[1:]
		}
		if len(lines) <= 3 {
			db.log.Error("Compaction leveldbTable not found")
			merr = errors.New("compaction leveldbTable not found")
			continue
		}
		lines = lines[3:]

		// Iterate over all the leveldbTable rows, and accumulate the entries
		for j := 0; j < len(compactions[i%2]); j++ {
			compactions[i%2][j] = 0
		}
		for _, line := range lines {
			parts := strings.Split(line, "|")
			if len(parts) != 6 {
				break
			}
			for idx, counter := range parts[2:] {
				value, err := strconv.ParseFloat(strings.TrimSpace(counter), 64)
				if err != nil {
					db.log.Error("Compaction entry parsing failed", "err", err)
					merr = err
					continue
				}
				compactions[i%2][idx] += value
			}
		}
		// Update all the requested meters
		if db.diskSizeGauge != nil {
			db.diskSizeGauge.Update(int64(compactions[i%2][0] * 1024 * 1024))
		}
		if db.compTimeMeter != nil {
			db.compTimeMeter.Mark(int64((compactions[i%2][1] - compactions[(i-1)%2][1]) * 1000 * 1000 * 1000))
		}
		if db.compReadMeter != nil {
			db.compReadMeter.Mark(int64((compactions[i%2][2] - compactions[(i-1)%2][2]) * 1024 * 1024))
		}
		if db.compWriteMeter != nil {
			db.compWriteMeter.Mark(int64((compactions[i%2][3] - compactions[(i-1)%2][3]) * 1024 * 1024))
		}
		// Retrieve the write delay statistic
		writedelay, err := db.db.GetProperty("leveldb.writedelay")
		if err != nil {
			db.log.Error("Failed to read database write delay statistic", "err", err)
			merr = err
			continue
		}
		var (
			delayN        int64
			delayDuration string
			duration      time.Duration
			paused        bool
		)
		if n, err := fmt.Sscanf(writedelay, "DelayN:%d Delay:%s Paused:%t", &delayN, &delayDuration, &paused); n != 3 || err != nil {
			db.log.Error("Write delay statistic not found")
			merr = err
			continue
		}
		duration, err = time.ParseDuration(delayDuration)
		if err != nil {
			db.log.Error("Failed to parse delay duration", "err", err)
			merr = err
			continue
		}
		if db.writeDelayNMeter != nil {
			db.writeDelayNMeter.Mark(delayN - delaystats[0])
		}
		if db.writeDelayMeter != nil {
			db.writeDelayMeter.Mark(duration.Nanoseconds() - delaystats[1])
		}
		// If a warning that db is performing compaction has been displayed, any subsequent
		// warnings will be withheld for one minute not to overwhelm the user.
		if paused && delayN-delaystats[0] == 0 && duration.Nanoseconds()-delaystats[1] == 0 &&
			time.Now().After(lastWritePaused.Add(degradationWarnInterval)) {
			db.log.Warn("Database compacting, degraded performance")
			lastWritePaused = time.Now()
		}
		delaystats[0], delaystats[1] = delayN, duration.Nanoseconds()

		// Retrieve the database iostats.
		ioStats, err := db.db.GetProperty("leveldb.iostats")
		if err != nil {
			db.log.Error("Failed to read database iostats", "err", err)
			merr = err
			continue
		}
		var nRead, nWrite float64
		parts := strings.Split(ioStats, " ")
		if len(parts) < 2 {
			db.log.Error("Bad syntax of ioStats", "ioStats", ioStats)
			merr = fmt.Errorf("bad syntax of ioStats %s", ioStats)
			continue
		}
		if n, err := fmt.Sscanf(parts[0], "Read(MB):%f", &nRead); n != 1 || err != nil {
			db.log.Error("Bad syntax of read entry", "entry", parts[0])
			merr = err
			continue
		}
		if n, err := fmt.Sscanf(parts[1], "Write(MB):%f", &nWrite); n != 1 || err != nil {
			db.log.Error("Bad syntax of write entry", "entry", parts[1])
			merr = err
			continue
		}
		if db.diskReadMeter != nil {
			db.diskReadMeter.Mark(int64((nRead - iostats[0]) * 1024 * 1024))
		}
		if db.diskWriteMeter != nil {
			db.diskWriteMeter.Mark(int64((nWrite - iostats[1]) * 1024 * 1024))
		}
		iostats[0], iostats[1] = nRead, nWrite

		compCount, err := db.db.GetProperty("leveldb.compcount")
		if err != nil {
			db.log.Error("Failed to read database iostats", "err", err)
			merr = err
			continue
		}

		var (
			memComp       uint32
			level0Comp    uint32
			nonLevel0Comp uint32
			seekComp      uint32
		)
		if n, err := fmt.Sscanf(compCount, "MemComp:%d Level0Comp:%d NonLevel0Comp:%d SeekComp:%d", &memComp, &level0Comp, &nonLevel0Comp, &seekComp); n != 4 || err != nil {
			db.log.Error("Compaction count statistic not found")
			merr = err
			continue
		}
		db.memCompGauge.Update(int64(memComp))
		db.level0CompGauge.Update(int64(level0Comp))
		db.nonlevel0CompGauge.Update(int64(nonLevel0Comp))
		db.seekCompGauge.Update(int64(seekComp))

		// Sleep a bit, then repeat the stats collection
		select {
		case errc = <-db.quitChan:
			// Quit requesting, stop hammering the database
		case <-timer.C:
			timer.Reset(refresh)
			// Timeout, gather a new set of stats
		}
	}

	if errc == nil {
		errc = <-db.quitChan
	}
	errc <- merr
}
```
