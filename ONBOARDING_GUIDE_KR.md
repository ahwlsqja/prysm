# Prysm 코드베이스 온보딩 가이드

> 시니어 개발자가 주니어 개발자에게 전달하는 단계별 인수인계 문서

---

## 목차

1. [Phase 1: 기초 개념 이해](#phase-1-기초-개념-이해)
2. [Phase 2: 프로젝트 구조 파악](#phase-2-프로젝트-구조-파악)
3. [Phase 3: 핵심 컴포넌트 Deep Dive](#phase-3-핵심-컴포넌트-deep-dive)
4. [Phase 4: 데이터 흐름 이해](#phase-4-데이터-흐름-이해)
5. [Phase 5: 실전 코드 워크스루](#phase-5-실전-코드-워크스루)
6. [Phase 6: 개발 환경 및 빌드](#phase-6-개발-환경-및-빌드)

---

## Phase 1: 기초 개념 이해

### 1.1 Prysm이란?

Prysm은 **Ethereum Beacon Chain (Consensus Layer)** 클라이언트입니다. Go 언어로 작성되었으며, Ethereum 2.0의 Proof-of-Stake(PoS) 합의 메커니즘을 구현합니다.

> 이 코드베이스는 **OffchainLabs** (Arbitrum 팀)에서 유지보수하는 포크 버전입니다.

### 1.2 Ethereum 아키텍처 (The Merge 이후)

```
┌─────────────────────────────────────────────────────────────┐
│                    Ethereum Node                            │
├─────────────────────────┬───────────────────────────────────┤
│   Consensus Layer (CL)  │      Execution Layer (EL)         │
│   ┌─────────────────┐   │      ┌─────────────────┐          │
│   │  Beacon Chain   │◄──┼──────│     Geth        │          │
│   │    (Prysm)      │   │      │   (go-ethereum) │          │
│   └─────────────────┘   │      └─────────────────┘          │
│          │              │               │                   │
│   PoS Consensus         │      EVM, Transactions            │
│   Block Finality        │      Smart Contracts              │
│   Validator Management  │      State Management             │
└─────────────────────────┴───────────────────────────────────┘
```

**핵심 개념:**
- **Consensus Layer (CL)**: 블록 합의, 검증자 관리, 체인 Finality 담당 (Prysm)
- **Execution Layer (EL)**: 트랜잭션 실행, 스마트 컨트랙트 처리 (Geth)
- **Engine API**: CL과 EL 간 통신 프로토콜

### 1.3 핵심 용어 정리

| 용어 | 설명 |
|------|------|
| **Slot** | 12초 단위의 시간 구간. 각 슬롯에 하나의 블록이 제안됨 |
| **Epoch** | 32개의 슬롯 (약 6.4분). 검증자 셔플링 및 보상 계산 단위 |
| **Validator** | 32 ETH를 스테이킹한 검증자. 블록 제안 및 증명 수행 |
| **Attestation** | 검증자가 특정 블록에 대해 투표하는 것 |
| **Beacon State** | 모든 검증자 정보, 잔액, 슬래싱 등 전체 상태 |
| **Fork Choice** | 여러 체인 중 정규 체인을 선택하는 알고리즘 (LMD-GHOST) |
| **Finality** | 블록이 되돌릴 수 없는 상태가 되는 것 (Casper FFG) |
| **Checkpoint** | Epoch 경계에서의 블록. Finality 판단 기준 |

### 1.4 네트워크 업그레이드 (Fork) 히스토리

```
Phase 0 (Genesis)
    │
    ▼
Altair (Epoch 74240) - Sync Committee 도입
    │
    ▼
Bellatrix (Epoch 144896) - The Merge 준비
    │
    ▼
Capella (Epoch 194048) - Withdrawal 활성화
    │
    ▼
Deneb (Epoch 269568) - Proto-Danksharding (EIP-4844)
    │
    ▼
Electra (예정) - Validator 관련 개선
    │
    ▼
Fulu (예정) - 추가 개선
    │
    ▼
Gloas (예정) - 최신 업그레이드
```

---

## Phase 2: 프로젝트 구조 파악

### 2.1 디렉토리 구조 개요

```
prysm/
├── cmd/                          # 실행 진입점 (main.go)
│   ├── beacon-chain/            # Beacon Node 실행 파일
│   ├── validator/               # Validator Client 실행 파일
│   └── prysmctl/                # CLI 유틸리티
│
├── beacon-chain/                 # 🔥 Beacon Node 핵심 로직
│   ├── node/                    # 노드 부트스트래핑
│   ├── blockchain/              # 체인 관리 서비스
│   ├── core/                    # 합의 핵심 로직
│   ├── state/                   # 상태 관리
│   ├── db/                      # 데이터베이스
│   ├── p2p/                     # P2P 네트워킹
│   ├── sync/                    # 동기화 로직
│   ├── forkchoice/              # Fork Choice 구현
│   ├── operations/              # Pool (attestations, exits 등)
│   ├── rpc/                     # gRPC/REST API
│   ├── execution/               # Execution Layer 연동
│   └── slasher/                 # 슬래싱 탐지
│
├── validator/                    # 🔥 Validator Client 핵심 로직
│   ├── client/                  # 검증자 클라이언트
│   ├── keymanager/              # 키 관리
│   └── accounts/                # 계정 관리
│
├── consensus-types/              # 합의 타입 정의
├── proto/                        # Protocol Buffer 정의
├── api/                          # API 서버 및 클라이언트
├── config/                       # 설정 (네트워크 파라미터)
├── crypto/                       # 암호화 (BLS 등)
├── encoding/                     # SSZ 인코딩
├── network/                      # 네트워크 유틸리티
├── testing/                      # 테스트 인프라
└── tools/                        # 개발 도구
```

### 2.2 두 가지 핵심 바이너리

#### Beacon Node (`cmd/beacon-chain/`)
```
┌──────────────────────────────────────────────────────┐
│                    Beacon Node                        │
│                                                       │
│  ┌─────────┐  ┌──────────┐  ┌──────────────────┐    │
│  │   P2P   │  │ Database │  │ Execution Client │    │
│  │ Network │  │   (KV)   │  │   Connection     │    │
│  └────┬────┘  └────┬─────┘  └────────┬─────────┘    │
│       │            │                  │              │
│       ▼            ▼                  ▼              │
│  ┌─────────────────────────────────────────────┐    │
│  │            Blockchain Service               │    │
│  │  (Fork Choice, Block Processing, Finality)  │    │
│  └─────────────────────────────────────────────┘    │
│                       │                              │
│       ┌───────────────┼───────────────┐             │
│       ▼               ▼               ▼             │
│  ┌─────────┐   ┌───────────┐   ┌──────────┐        │
│  │  Sync   │   │ Operations│   │   RPC    │        │
│  │ Service │   │   Pool    │   │ Services │        │
│  └─────────┘   └───────────┘   └──────────┘        │
└──────────────────────────────────────────────────────┘
```

#### Validator Client (`cmd/validator/`)
```
┌──────────────────────────────────────────────────────┐
│                 Validator Client                      │
│                                                       │
│  ┌────────────────┐    ┌─────────────────────────┐   │
│  │  Key Manager   │    │   Beacon Node (gRPC)    │   │
│  │ (BLS Signing)  │    │      Connection         │   │
│  └───────┬────────┘    └───────────┬─────────────┘   │
│          │                         │                 │
│          ▼                         ▼                 │
│  ┌─────────────────────────────────────────────┐    │
│  │           Validator Service                  │    │
│  └─────────────────────────────────────────────┘    │
│          │              │              │             │
│          ▼              ▼              ▼             │
│  ┌──────────┐   ┌───────────┐   ┌──────────────┐   │
│  │ Proposer │   │ Attester  │   │ Sync Committee│   │
│  │  Duties  │   │  Duties   │   │    Duties    │   │
│  └──────────┘   └───────────┘   └──────────────┘   │
└──────────────────────────────────────────────────────┘
```

### 2.3 주요 파일 위치 퀵 레퍼런스

| 기능 | 파일 경로 |
|------|----------|
| Beacon Node 진입점 | `cmd/beacon-chain/main.go` |
| Beacon Node 초기화 | `beacon-chain/node/node.go` |
| Validator Client 진입점 | `cmd/validator/main.go` |
| 블록 처리 | `beacon-chain/blockchain/process_block.go` |
| Fork Choice | `beacon-chain/forkchoice/doubly-linked-tree/` |
| 상태 전이 | `beacon-chain/core/transition/` |
| Epoch 처리 | `beacon-chain/core/epoch/` |
| P2P 메시지 핸들러 | `beacon-chain/sync/` |
| 네트워크 설정 | `config/params/mainnet_config.go` |

---

## Phase 3: 핵심 컴포넌트 Deep Dive

### 3.1 Service Registry 패턴

Prysm은 **Service Registry 패턴**을 사용하여 모든 서비스를 관리합니다.

```go
// beacon-chain/node/node.go:138

func New(cliCtx *cli.Context, cancel context.CancelFunc, opts ...Option) (*BeaconNode, error) {
    beacon := &BeaconNode{
        services: runtime.NewServiceRegistry(),  // 서비스 레지스트리
        // ... 초기화
    }

    // 서비스 등록 순서 (의존성 고려)
    registerServices(cliCtx, beacon, synchronizer, bfs)

    return beacon, nil
}
```

**서비스 등록 순서** (`beacon-chain/node/node.go:353`):
```
1. P2P Service          → 네트워크 레이어
2. Light Client Store   → (옵션) 라이트 클라이언트
3. Backfill Service     → 히스토리 동기화
4. POW Chain Service    → Execution Layer 연결
5. Attestation Pool     → Attestation 관리
6. Blockchain Service   → 🔥 핵심 체인 로직
7. Initial Sync         → 초기 동기화
8. Sync Service         → 실시간 동기화
9. Slashing Pool        → 슬래싱 관리
10. Slasher Service     → 슬래싱 탐지
11. Builder Service     → MEV Builder 연동
12. RPC Service         → API 서버
13. HTTP Service        → REST API
14. Validator Monitor   → 검증자 모니터링
15. Prometheus          → 메트릭
16. Pruner              → DB 정리
```

### 3.2 BeaconNode 구조체 분석

```go
// beacon-chain/node/node.go:90

type BeaconNode struct {
    // 컨텍스트 관리
    ctx     context.Context
    cancel  context.CancelFunc

    // 서비스 관리
    services *runtime.ServiceRegistry

    // 데이터 저장
    db        db.Database           // BoltDB (KV Store)
    slasherDB db.SlasherDatabase    // Slasher 전용 DB

    // 캐시 및 Pool
    attestationPool   attestations.Pool      // 대기 중인 Attestation
    exitPool          voluntaryexits.Pool    // 자발적 탈퇴 요청
    slashingsPool     slashings.Pool         // 슬래싱 증거
    syncCommitteePool synccommittee.Pool     // Sync Committee 메시지

    // 이벤트 피드 (Pub/Sub)
    stateFeed *event.Feed    // 상태 변경 알림
    blockFeed *event.Feed    // 새 블록 알림
    opFeed    *event.Feed    // 오퍼레이션 알림

    // 핵심 컴포넌트
    forkChoicer forkchoice.ForkChoicer  // Fork Choice 알고리즘
    stateGen    *stategen.State          // 상태 생성기

    // Blob 스토리지 (EIP-4844)
    BlobStorage       *filesystem.BlobStorage
    DataColumnStorage *filesystem.DataColumnStorage
}
```

### 3.3 Blockchain Service (가장 중요!)

```go
// beacon-chain/blockchain/service.go

type Service struct {
    cfg                *config.BeaconConfig
    db                 db.HeadAccessDatabase
    forkChoiceStore    forkchoice.ForkChoicer
    stateGen           *stategen.State

    // 헤드 블록/상태 관리
    head  *head

    // 실행 레이어 연결
    executionEngineCaller execution.EngineCaller

    // 최종 체크포인트
    finalizedCheckpt    *ethpb.Checkpoint
    justifiedCheckpt    *ethpb.Checkpoint
}
```

### 3.4 Fork Choice (LMD-GHOST + Casper FFG)

```go
// beacon-chain/forkchoice/doubly-linked-tree/

// Fork Choice 알고리즘 핵심
type ForkChoice struct {
    store      *Store
    nodes      *fieldparams.TreeNode

    // 투표 정보
    votes      []Vote

    // 체크포인트
    justifiedCheckpoint *forkchoicetypes.Checkpoint
    finalizedCheckpoint *forkchoicetypes.Checkpoint
}
```

**알고리즘 설명:**
1. **LMD-GHOST**: Latest Message Driven Greedy Heaviest Observed SubTree
   - 각 검증자의 최신 투표만 사용
   - 가장 많은 투표를 받은 서브트리 선택

2. **Casper FFG**: Friendly Finality Gadget
   - Justified → Finalized 전이
   - 2/3 이상의 투표로 체크포인트 확정

---

## Phase 4: 데이터 흐름 이해

### 4.1 블록 수신부터 Finality까지

```
┌─────────────────────────────────────────────────────────────────┐
│                        Block Lifecycle                          │
└─────────────────────────────────────────────────────────────────┘

1️⃣ P2P로 블록 수신
   beacon-chain/sync/subscriber_beacon_blocks.go
        │
        ▼
2️⃣ 블록 검증 (Signature, SSZ 등)
   beacon-chain/sync/validate_beacon_blocks.go
        │
        ▼
3️⃣ 상태 전이 (State Transition)
   beacon-chain/core/transition/transition.go
   ┌──────────────────────────────────────┐
   │  ProcessSlots()                      │
   │  ProcessBlockHeader()                │
   │  ProcessRandao()                     │
   │  ProcessEth1Data()                   │
   │  ProcessOperations()                 │
   │    └─ ProcessAttestations()          │
   │    └─ ProcessDeposits()              │
   │    └─ ProcessVoluntaryExits()        │
   │    └─ ProcessProposerSlashings()     │
   │    └─ ProcessAttesterSlashings()     │
   └──────────────────────────────────────┘
        │
        ▼
4️⃣ Fork Choice 업데이트
   beacon-chain/blockchain/process_block.go
   beacon-chain/forkchoice/doubly-linked-tree/
        │
        ▼
5️⃣ 헤드 업데이트 & DB 저장
   beacon-chain/blockchain/head.go
        │
        ▼
6️⃣ 이벤트 브로드캐스트
   stateFeed.Send(newHead)
```

### 4.2 Attestation 처리 흐름

```
┌─────────────────────────────────────────────────────────────────┐
│                    Attestation Flow                             │
└─────────────────────────────────────────────────────────────────┘

Validator Client                         Beacon Node
     │                                        │
     │  1. Request: GetAttestationData()      │
     │ ────────────────────────────────────►  │
     │                                        │
     │  2. Response: AttestationData          │
     │ ◄────────────────────────────────────  │
     │                                        │
     │  (BLS Sign with Private Key)           │
     │                                        │
     │  3. Submit: SubmitAttestation()        │
     │ ────────────────────────────────────►  │
     │                                        │
     │                              ┌─────────────────────┐
     │                              │ Attestation Pool    │
     │                              │ (aggregate pending) │
     │                              └─────────────────────┘
     │                                        │
     │                                        ▼
     │                              ┌─────────────────────┐
     │                              │ P2P Broadcast       │
     │                              │ (gossip to network) │
     │                              └─────────────────────┘
     │                                        │
     │                                        ▼
     │                              ┌─────────────────────┐
     │                              │ Fork Choice Update  │
     │                              │ (vote counting)     │
     │                              └─────────────────────┘
```

### 4.3 Epoch 전이 (중요!)

```
┌─────────────────────────────────────────────────────────────────┐
│                     Epoch Transition                            │
│              beacon-chain/core/epoch/epoch_processing.go        │
└─────────────────────────────────────────────────────────────────┘

Slot 0    ...   Slot 31  │  Slot 32   ...   Slot 63
  └─────── Epoch N ──────┼────────── Epoch N+1 ──────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│  ProcessEpoch():                                                │
│                                                                 │
│  1. ProcessJustificationAndFinalization()                       │
│     → Casper FFG: Justified/Finalized 체크포인트 업데이트        │
│                                                                 │
│  2. ProcessInactivityUpdates()                                  │
│     → 비활성 검증자 페널티 계산                                   │
│                                                                 │
│  3. ProcessRewardsAndPenalties()                                │
│     → 보상/페널티 적용                                           │
│                                                                 │
│  4. ProcessRegistryUpdates()                                    │
│     → 검증자 활성화/탈퇴 처리                                     │
│                                                                 │
│  5. ProcessSlashings()                                          │
│     → 슬래싱된 검증자 처리                                        │
│                                                                 │
│  6. ProcessEth1DataReset()                                      │
│     → ETH1 투표 리셋                                             │
│                                                                 │
│  7. ProcessEffectiveBalanceUpdates()                            │
│     → 유효 잔액 업데이트                                          │
│                                                                 │
│  8. ProcessSlashingsReset()                                     │
│     → 슬래싱 배열 리셋                                            │
│                                                                 │
│  9. ProcessRandaoMixesReset()                                   │
│     → RANDAO 리셋                                                │
│                                                                 │
│  10. ProcessHistoricalDataUpdate()                              │
│      → 히스토리 저장                                              │
│                                                                 │
│  11. ProcessSyncCommitteeUpdates() [Altair+]                    │
│      → Sync Committee 로테이션                                   │
└─────────────────────────────────────────────────────────────────┘
```

### 4.4 Execution Layer 통신

```
┌─────────────────────────────────────────────────────────────────┐
│                   Engine API Communication                      │
│              beacon-chain/execution/engine_client.go            │
└─────────────────────────────────────────────────────────────────┘

Beacon Node                                    Execution Client
(Prysm)                                        (Geth)
     │                                              │
     │  engine_newPayloadV3()                       │
     │  → 새 블록의 실행 페이로드 검증               │
     │ ─────────────────────────────────────────►  │
     │                                              │
     │  ◄───────────────────────────────────────── │
     │  {status: "VALID"/"INVALID"/"SYNCING"}      │
     │                                              │
     │  engine_forkchoiceUpdatedV3()               │
     │  → Fork Choice 상태 업데이트                 │
     │ ─────────────────────────────────────────►  │
     │                                              │
     │  ◄───────────────────────────────────────── │
     │  {payloadId: "0x..."}  (블록 빌딩 시작)     │
     │                                              │
     │  engine_getPayloadV4()                       │
     │  → 빌드된 블록 페이로드 가져오기             │
     │ ─────────────────────────────────────────►  │
     │                                              │
     │  ◄───────────────────────────────────────── │
     │  {executionPayload, blobsBundle, ...}       │
```

---

## Phase 5: 실전 코드 워크스루

### 5.1 Beacon Node 시작 흐름

```go
// 1. 진입점: cmd/beacon-chain/main.go:256
func main() {
    app := cli.App{
        Action: func(ctx *cli.Context) error {
            return startNode(ctx, cancel)
        },
    }
    app.RunContext(rctx, os.Args)
}

// 2. 노드 시작: cmd/beacon-chain/main.go:293
func startNode(ctx *cli.Context, cancel context.CancelFunc) error {
    // 옵션 설정
    opts := []node.Option{
        node.WithBlockchainFlagOptions(blockchainFlagOpts),
        node.WithExecutionChainOptions(executionFlagOpts),
        // ...
    }

    // 3. BeaconNode 생성
    beacon, err := node.New(ctx, cancel, opts...)

    // 4. 시작!
    beacon.Start()
}

// 5. 서비스 시작: beacon-chain/node/node.go:469
func (b *BeaconNode) Start() {
    log.Info("Starting beacon node")
    b.services.StartAll()  // 모든 등록된 서비스 시작

    // 시그널 핸들링 (SIGINT, SIGTERM)
    go func() {
        sigc := make(chan os.Signal, 1)
        signal.Notify(sigc, syscall.SIGINT, syscall.SIGTERM)
        <-sigc
        b.Close()
    }()

    <-stop  // 종료 대기
}
```

### 5.2 블록 처리 코드 따라가기

```go
// 1. P2P 메시지 수신: beacon-chain/sync/subscriber_beacon_blocks.go
func (s *Service) subscribeBlocks() {
    s.subscribe(
        p2p.BlockSubnetTopicFormat,
        s.validateBeaconBlock,
        s.receiveBlock,
    )
}

// 2. 블록 검증: beacon-chain/sync/validate_beacon_blocks.go
func (s *Service) validateBeaconBlock(ctx context.Context, msg *pubsub.Message) bool {
    // 서명 검증
    // 슬롯 유효성 검증
    // 부모 블록 존재 여부
    // ...
}

// 3. 블록 수신 처리: beacon-chain/sync/subscriber_beacon_blocks.go
func (s *Service) receiveBlock(ctx context.Context, msg *pubsub.Message) error {
    return s.cfg.chain.ReceiveBlock(ctx, block)
}

// 4. Blockchain Service: beacon-chain/blockchain/receive_block.go
func (s *Service) ReceiveBlock(ctx context.Context, block interfaces.ReadOnlySignedBeaconBlock) error {
    // 상태 전이 실행
    postState, err := s.executeStateTransition(ctx, preState, block)

    // Fork Choice 업데이트
    s.updateForkChoiceOnBlock(ctx, block, postState)

    // 헤드 업데이트
    s.updateHead(ctx)
}

// 5. 상태 전이: beacon-chain/core/transition/transition.go
func ExecuteStateTransition(ctx context.Context, state state.BeaconState, block interfaces.ReadOnlySignedBeaconBlock) error {
    // 슬롯 처리
    ProcessSlots(ctx, state, block.Block().Slot())

    // 블록 처리
    ProcessBlockForStateRoot(ctx, state, block)
}
```

### 5.3 상태 관리 (State Gen)

```go
// beacon-chain/state/stategen/setter.go

// 상태 생성 전략
type State struct {
    beaconDB       db.NoHeadAccessDatabase
    replayerBuilder ReplayerBuilder

    // Hot States: 최근 N개 슬롯의 상태 (메모리)
    hotStateCache *cache.HotStateCache

    // Epoch Boundary States: Epoch 경계 상태 (DB)
    epochBoundaryStateCache *lru.Cache
}

// 특정 루트의 상태 가져오기
func (s *State) StateByRoot(ctx context.Context, root [32]byte) (state.BeaconState, error) {
    // 1. Hot cache 확인
    if st := s.hotStateCache.Get(root); st != nil {
        return st, nil
    }

    // 2. DB에서 가져오기
    if st, err := s.beaconDB.State(ctx, root); err == nil {
        return st, nil
    }

    // 3. 리플레이로 재구성
    return s.replayToSlot(ctx, root)
}
```

---

## Phase 6: 개발 환경 및 빌드

### 6.1 빌드 시스템 (Bazel)

```bash
# Bazel 버전 확인
cat .bazelversion  # 7.4.1

# Beacon Node 빌드
bazel build //cmd/beacon-chain:beacon-chain

# Validator Client 빌드
bazel build //cmd/validator:validator

# 모든 테스트 실행
bazel test //...

# 특정 패키지 테스트
bazel test //beacon-chain/blockchain:go_default_test
```

### 6.2 Go 모듈 빌드 (대안)

```bash
# Go 버전: 1.25.1+

# Beacon Node
go build -o beacon-chain ./cmd/beacon-chain

# Validator Client
go build -o validator ./cmd/validator

# 테스트
go test ./beacon-chain/...
```

### 6.3 주요 설정 파일

| 파일 | 용도 |
|------|------|
| `MODULE.bazel` | Bazel 모듈 설정 |
| `go.mod` | Go 모듈 의존성 |
| `.bazelrc` | Bazel 빌드 옵션 |
| `config/params/mainnet_config.go` | 메인넷 파라미터 |
| `config/params/testnet_*_config.go` | 테스트넷 파라미터 |

### 6.4 테스트 전략

```bash
# 단위 테스트
bazel test //beacon-chain/core/transition:go_default_test

# 통합 테스트
bazel test //testing/endtoend:go_default_test

# Spec 테스트 (Ethereum 공식 테스트 벡터)
bazel test //testing/spectest/...

# Fuzzing
bazel run //tools/beacon-fuzz:beacon-fuzz
```

---

## 학습 로드맵 추천

### Week 1-2: 기초
- [ ] Ethereum PoS 개념 이해 (https://ethereum.org/en/developers/docs/consensus-mechanisms/pos/)
- [ ] 프로젝트 빌드 및 로컬 실행
- [ ] `cmd/beacon-chain/main.go` 흐름 추적

### Week 3-4: 핵심 로직
- [ ] `beacon-chain/node/node.go` 서비스 등록 이해
- [ ] `beacon-chain/blockchain/` 블록 처리 코드 분석
- [ ] `beacon-chain/core/transition/` 상태 전이 이해

### Week 5-6: 네트워킹
- [ ] `beacon-chain/p2p/` P2P 구조 이해
- [ ] `beacon-chain/sync/` 동기화 로직 분석

### Week 7-8: 심화
- [ ] `beacon-chain/forkchoice/` Fork Choice 알고리즘
- [ ] `beacon-chain/execution/` Engine API
- [ ] Validator Client 분석

### 지속적 학습
- [ ] Ethereum 스펙 문서: https://github.com/ethereum/consensus-specs
- [ ] EIP 추적: EIP-4844 (Deneb), EIP-7594 (PeerDAS)
- [ ] Prysm Discord/GitHub Issues 팔로우

---

## 디버깅 팁

### 로그 레벨 조정
```bash
./beacon-chain --verbosity=debug
```

### pprof 프로파일링
```bash
./beacon-chain --pprof
# http://localhost:6060/debug/pprof/
```

### 주요 로그 필터
```bash
# 블록 관련
./beacon-chain 2>&1 | grep -i "block"

# Fork Choice 관련
./beacon-chain 2>&1 | grep -i "forkchoice"

# 동기화 관련
./beacon-chain 2>&1 | grep -i "sync"
```

---

## 질문이 있다면?

1. 코드 특정 부분이 이해되지 않으면 해당 파일과 라인 번호를 알려주세요.
2. 특정 기능이나 흐름에 대해 더 깊이 알고 싶으면 말씀해주세요.
3. 실습 과제가 필요하면 단계별로 제공해드릴 수 있습니다.

**Happy Coding! 🚀**
