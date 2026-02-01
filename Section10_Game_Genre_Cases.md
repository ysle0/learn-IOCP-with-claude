# Section 10: 다양한 게임 장르에서의 IOCP 활용 사례

> 작성 완료일: 2026-02-01

---

## 목차
- [10.1 FPS/TPS (배틀로얄, 택티컬 슈터)](#101-fpstps-배틀로얄-택티컬-슈터)
- [10.2 RTS/MOBA](#102-rtsmoba)
- [10.3 격투/액션 게임 (P2P + 릴레이 서버)](#103-격투액션-게임-p2p--릴레이-서버)
- [10.4 턴제 게임 (카드, 보드, 전략)](#104-턴제-게임-카드-보드-전략)
- [10.5 캐주얼/소셜 게임](#105-캐주얼소셜-게임)
- [10.6 게임 외 IOCP 활용 분야](#106-게임-외-iocp-활용-분야)
- [10.7 장르별 비교 정리](#107-장르별-비교-정리)
- [10.8 퀴즈용 핵심 Q&A](#108-퀴즈용-핵심-qa)

---

## 10.1 FPS/TPS (배틀로얄, 택티컬 슈터)

### 10.1.1 네트워크 특성

```
핵심 요구사항:
- 초저지연 (< 50ms RTT 이상적)
- 높은 틱레이트 (60~128 tick/sec)
- 정밀한 히트 판정 (서버 권위)
- 동시접속: 방당 2~100명 (배틀로얄 최대 100명+)

패킷 특성:
- 크기: 매우 작음 (20~100 bytes)
- 빈도: 매우 높음 (초당 60~128 패킷/클라이언트)
- 방향: 양방향 대칭
- 주요 데이터: 위치, 회전, 입력 상태, 사격 이벤트
```

### 10.1.2 서버 아키텍처

```
┌──────────────────────────────────────────────────┐
│                 Match Server (1 게임방)             │
├──────────────────────────────────────────────────┤
│                                                    │
│  ┌────────────────────┐  ┌─────────────────────┐  │
│  │ Network Layer       │  │ Game Simulation     │  │
│  │ (IOCP)             │  │                     │  │
│  │                    │  │ - Physics (128Hz)   │  │
│  │ - UDP Recv/Send    │  │ - Hit Detection     │  │
│  │ - TCP (로비/채팅)   │  │ - Player State      │  │
│  │                    │  │ - Projectile Sim    │  │
│  └────────┬───────────┘  └──────────┬──────────┘  │
│           │                          │              │
│           ▼                          ▼              │
│  ┌─────────────────────────────────────────────┐   │
│  │           Snapshot / Delta System            │   │
│  │  - 월드 상태 스냅샷 생성 (128Hz)              │   │
│  │  - 클라이언트별 델타 압축                      │   │
│  │  - 우선순위 기반 업데이트 (가까운 적 우선)      │   │
│  └─────────────────────────────────────────────┘   │
│                                                    │
│  ┌─────────────────────────────────────────────┐   │
│  │           Lag Compensation                   │   │
│  │  - 히스토리 버퍼 (과거 128틱 저장)             │   │
│  │  - 서버 되감기 (Server Rewind)                │   │
│  │  - 사격 시점의 월드 상태로 히트 판정           │   │
│  └─────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────┘
```

### 10.1.3 IOCP에서 UDP 처리

FPS 게임은 실시간 데이터에 UDP를 사용하는 경우가 많다. IOCP는 UDP 소켓도 지원한다.

```cpp
// UDP 소켓을 IOCP에 연결
SOCKET udpSocket = WSASocket(AF_INET, SOCK_DGRAM, IPPROTO_UDP,
                              NULL, 0, WSA_FLAG_OVERLAPPED);
CreateIoCompletionPort((HANDLE)udpSocket, hIOCP, UDP_KEY, 0);

// 비동기 UDP 수신 (WSARecvFrom)
struct UdpOverlapped {
    OVERLAPPED overlapped;
    WSABUF wsaBuf;
    char buffer[1500];        // MTU 크기
    SOCKADDR_IN remoteAddr;
    int addrLen;
    OperationType opType;
};

void PostUdpRecv(UdpOverlapped* pOv) {
    ZeroMemory(&pOv->overlapped, sizeof(OVERLAPPED));
    pOv->wsaBuf.buf = pOv->buffer;
    pOv->wsaBuf.len = sizeof(pOv->buffer);
    pOv->addrLen = sizeof(pOv->remoteAddr);
    DWORD flags = 0;

    WSARecvFrom(udpSocket, &pOv->wsaBuf, 1, NULL, &flags,
                (SOCKADDR*)&pOv->remoteAddr, &pOv->addrLen,
                &pOv->overlapped, NULL);
}

// UDP는 연결 개념이 없으므로:
// - CompletionKey로 소켓 식별
// - remoteAddr로 클라이언트 식별
// - 패킷 내 세션 토큰으로 인증
```

### 10.1.4 Snapshot + Delta 압축

```cpp
// 월드 상태 스냅샷 (128Hz로 생성)
struct PlayerSnapshot {
    uint32_t playerId;
    float posX, posY, posZ;       // 12 bytes
    float rotYaw, rotPitch;       // 8 bytes
    uint8_t health;               // 1 byte
    uint8_t weapon;               // 1 byte
    uint8_t state;                // 1 byte (달리기, 앉기, 점프 등)
};                                // ~27 bytes per player

struct WorldSnapshot {
    uint32_t tick;
    uint32_t timestamp;
    vector<PlayerSnapshot> players;
};

// 델타 압축: 이전 ACK된 스냅샷과의 차이만 전송
struct DeltaPacket {
    uint32_t baseTick;            // 기준 스냅샷 틱
    uint32_t currentTick;
    uint8_t  changedCount;        // 변경된 플레이어 수
    // 변경된 필드만 비트마스크로 표시
    struct ChangedPlayer {
        uint32_t playerId;
        uint8_t  changedFields;   // 비트마스크: pos|rot|health|weapon|state
        // 변경된 필드만 이어서 기록
    };
};

// 100명 배틀로얄에서:
// 풀 스냅샷: 100 × 27 = 2,700 bytes
// 델타 (평균 30% 변경): ~810 bytes + 오버헤드 ≈ 900 bytes
// 128 tick/sec: 900 × 128 = 115 KB/s per client
// 100명: 11.5 MB/s 총 송신 (관리 가능)
```

---

## 10.2 RTS/MOBA

### 10.2.1 네트워크 특성

```
핵심 요구사항:
- 결정적 시뮬레이션 (Deterministic Lockstep)
- 모든 클라이언트가 동일한 결과를 보장
- 틱레이트: 보통 10~30 tick/sec (MOBA는 30)
- 동시접속: 방당 2~10명

패킷 특성:
- 크기: 작음 (입력 커맨드만 전송)
- 빈도: 중간 (10~30/sec)
- 주요 데이터: 입력 커맨드 (이동 명령, 스킬 사용, 유닛 선택)
- 월드 상태 자체는 전송 안 함 (각자 시뮬레이션)
```

### 10.2.2 Lockstep 아키텍처

```
Lockstep (RTS 전통 방식):

Tick N:
  Client A → Server: Input(Tick N) {Move Unit 5 to (100, 200)}
  Client B → Server: Input(Tick N) {Attack Unit 3}
  Server: 모든 입력 수집 완료 확인
  Server → All: InputBundle(Tick N) {A의 입력, B의 입력}
  All Clients: InputBundle을 로컬 시뮬레이션에 적용
  → 결정적(Deterministic)이므로 모든 클라이언트가 동일한 상태

IOCP 역할:
- 입력 커맨드 수집 (매우 가벼운 I/O)
- InputBundle 브로드캐스트 (작은 패킷, 적은 인원)
- TCP 사용 (순서 보장 필수, 패킷 손실 허용 불가)
```

```cpp
// Lockstep 서버 구현
class LockstepServer {
    static const int TICK_RATE = 20;  // 50ms per tick
    static const int MAX_PLAYERS = 10;

    struct TickInput {
        int playerId;
        vector<Command> commands;
    };

    struct TickData {
        int tickNumber;
        TickInput inputs[MAX_PLAYERS];
        bool received[MAX_PLAYERS];
    };

    TickData m_currentTick;
    int m_tickNumber = 0;

    // 입력 수신 (IOCP Worker에서 호출)
    void OnInputReceived(int playerId, const vector<Command>& cmds) {
        m_currentTick.inputs[playerId].commands = cmds;
        m_currentTick.received[playerId] = true;

        // 모든 플레이어 입력 수집 완료?
        if (AllInputsReceived()) {
            BroadcastTickData();
            AdvanceTick();
        }
    }

    void BroadcastTickData() {
        // 모든 입력을 묶어서 전체에게 전송
        TickBundlePacket pkt;
        pkt.tickNumber = m_tickNumber;
        for (int i = 0; i < m_playerCount; i++) {
            pkt.AddInput(m_currentTick.inputs[i]);
        }

        for (auto* session : m_sessions) {
            session->Send(&pkt, pkt.GetSize());
        }
    }

    bool AllInputsReceived() {
        for (int i = 0; i < m_playerCount; i++) {
            if (!m_currentTick.received[i]) return false;
        }
        return true;
    }
};
```

### 10.2.3 MOBA 변형 (서버 권위 + Lockstep 하이브리드)

```
League of Legends / Dota 2 스타일:

- 서버가 게임 시뮬레이션을 실행 (서버 권위)
- 클라이언트는 입력만 전송, 결과를 수신
- Lockstep과 달리 서버가 상태의 원본
- 치트 방지에 유리

서버 구조:
┌─────────────────────────────┐
│ MOBA Game Server (방당 1개)  │
│                             │
│ IOCP Layer:                 │
│   - 10명 TCP 연결           │
│   - 입력: ~300 packets/sec  │
│   - 출력: ~3,000 packets/sec│
│                             │
│ Game Logic (30Hz):          │
│   - 챔피언 이동/스킬        │
│   - 미니언 AI              │
│   - 타워/정글몹 AI         │
│   - 시야(Fog of War) 계산  │
│                             │
│ 특수 처리:                  │
│   - 팀별 시야 필터링        │
│   - 적 정보 선택적 전송     │
│   - 사망 시 관전 모드       │
└─────────────────────────────┘

IOCP 부하: 매우 낮음 (10명 × 작은 패킷)
→ 단일 서버에서 수십~수백 개 게임방 동시 운영 가능
```

---

## 10.3 격투/액션 게임 (P2P + 릴레이 서버)

### 10.3.1 네트워크 특성

```
핵심 요구사항:
- 극도로 낮은 지연 (1~3프레임 = 16~50ms)
- 입력 지연이 게임플레이에 직접적 영향
- 롤백 네트코드 (GGPO/Rollback)
- 동시접속: 2~4명 (1:1 또는 소규모)

네트워크 모델:
- P2P 직접 연결 (최저 지연)
- NAT 트래버설 필요 (STUN/TURN)
- P2P 불가 시 릴레이 서버 사용
```

### 10.3.2 릴레이 서버에서의 IOCP

```
P2P 연결이 불가능한 경우 릴레이 서버가 중계:

Client A ←──UDP──→ Relay Server ←──UDP──→ Client B
                    (IOCP 기반)

릴레이 서버 역할:
- 패킷을 받아서 상대방에게 포워딩 (relay)
- 게임 로직 없음 (순수 포워딩)
- 매우 가벼운 처리 → 하나의 서버에서 수천 매치 처리 가능
```

```cpp
class RelayServer {
    struct RelaySession {
        SOCKET udpSocket;
        SOCKADDR_IN peerAddr[2];  // 대전 상대 2명
        bool active;
    };

    // IOCP Worker에서: 패킷 수신 → 상대방에게 전달
    void OnUdpRecv(RelaySession* session, int senderIdx,
                    char* data, int len) {
        int targetIdx = 1 - senderIdx;  // 0 → 1, 1 → 0

        // 즉시 포워딩 (게임 로직 없음)
        WSABUF wsaBuf = { (ULONG)len, data };
        WSASendTo(session->udpSocket, &wsaBuf, 1, NULL, 0,
                  (SOCKADDR*)&session->peerAddr[targetIdx],
                  sizeof(SOCKADDR_IN), &sendOverlapped, NULL);
    }

    // 성능:
    // 격투 게임: 2명 × 60 packets/sec × 100 bytes = 12 KB/s per match
    // 1 Gbps 서버: 이론상 ~80,000 매치 동시 처리 가능
    // 실제 CPU 병목이 먼저 옴 → 수천~1만 매치 현실적
};
```

### 10.3.3 GGPO 롤백 네트코드와 서버

```
롤백 네트코드 개요:
1. 각 클라이언트가 입력을 예측하고 즉시 시뮬레이션
2. 상대 입력이 도착하면 예측이 맞았는지 확인
3. 틀렸으면 과거로 되돌려(rollback) 올바른 입력으로 재시뮬레이션
4. 화면을 현재 프레임으로 빠르게 다시 그림

서버 역할 (P2P 모델):
- 매치메이킹 서버: IOCP 기반, 대전 상대 매칭
- STUN 서버: NAT 타입 판별, 홀펀칭 보조
- TURN/릴레이 서버: P2P 불가 시 중계

서버 역할 (서버 권위 모델 - 안티치트):
- 입력 수집 + 시뮬레이션 + 결과 브로드캐스트
- 각 프레임의 게임 상태를 서버가 관리
- 롤백은 클라이언트 측에서만 발생 (예측용)
```

---

## 10.4 턴제 게임 (카드, 보드, 전략)

### 10.4.1 네트워크 특성

```
핵심 요구사항:
- 지연 허용 (~500ms까지 OK)
- 데이터 무결성 최우선 (패킷 손실 불가)
- 턴 시간 제한 관리
- 동시접속: 방당 2~8명, 전체 수만 명

패킷 특성:
- 크기: 가변 (작은 액션 ~ 큰 상태 동기화)
- 빈도: 매우 낮음 (턴당 1~5 패킷)
- TCP 필수 (순서 보장 + 신뢰성)
- 자동 진행 이벤트 (타이머 만료, AI 턴)
```

### 10.4.2 서버 구조

```cpp
class CardGameServer {
    // 턴제 게임은 IOCP 부하가 매우 낮음
    // 하나의 서버에서 수만 개 게임방 운영 가능

    struct GameRoom {
        int roomId;
        vector<Session*> players;
        int currentTurn;          // 현재 턴 플레이어 인덱스
        GameState state;
        DWORD turnStartTime;
        static const DWORD TURN_TIMEOUT_MS = 30000;  // 30초
    };

    // 수만 개 게임방을 단일 프로세스에서 관리
    unordered_map<int, GameRoom> m_rooms;  // roomId → GameRoom

    // 턴 액션 처리
    void OnPlayerAction(Session* session, ActionPacket* pkt) {
        GameRoom* room = FindRoom(session);
        if (!room) return;

        // 자기 턴인지 확인
        if (room->players[room->currentTurn] != session) {
            SendError(session, ERR_NOT_YOUR_TURN);
            return;
        }

        // 액션 유효성 검증
        if (!room->state.ValidateAction(pkt->action)) {
            SendError(session, ERR_INVALID_ACTION);
            return;
        }

        // 게임 상태 업데이트
        ActionResult result = room->state.ApplyAction(pkt->action);

        // 모든 플레이어에게 결과 브로드캐스트
        ActionResultPacket resultPkt;
        resultPkt.action = pkt->action;
        resultPkt.result = result;
        for (auto* player : room->players) {
            player->Send(&resultPkt, sizeof(resultPkt));
        }

        // 게임 종료 확인
        if (room->state.IsGameOver()) {
            HandleGameEnd(room);
            return;
        }

        // 다음 턴으로
        AdvanceTurn(room);
    }

    void AdvanceTurn(GameRoom* room) {
        room->currentTurn = (room->currentTurn + 1) % room->players.size();
        room->turnStartTime = GetTickCount();

        // 턴 타이머 시작
        // PostQueuedCompletionStatus로 타임아웃 이벤트 예약
        StartTurnTimer(room);
    }
};
```

### 10.4.3 IOCP 활용 효율

```
턴제 게임에서 IOCP의 효율성:

단일 서버 (8코어) 수용량 계산:
- 게임방당 평균 4명
- 평균 턴 시간: 10초
- 턴당 패킷: ~3개 (액션 + 결과 + 상태)
- 방당 초당 패킷: 4명 × 3 / 10 = 1.2 packets/sec

IOCP 처리 능력: ~100만 packets/sec
가능한 동시 게임방: ~800,000개
현실적 제한 (메모리, 연결 수): ~50,000방 (200,000명)

→ 턴제 게임은 IOCP 서버 1대로 매우 많은 동시접속 처리 가능
→ 네트워크가 아닌 게임 로직/DB가 병목
```

---

## 10.5 캐주얼/소셜 게임

### 10.5.1 네트워크 특성

```
핵심 요구사항:
- HTTP/WebSocket 기반이 주류 (모바일 웹뷰)
- 실시간성 낮음 (로비, 매칭, 결과 동기화)
- 높은 동시접속 (수만~수십만)
- 짧은 세션 (1게임 3~10분)
- 빈번한 연결/해제

적합한 모델:
- HTTP API 서버 + WebSocket 실시간 채널
- 또는 전통 TCP 소켓 + IOCP
```

### 10.5.2 WebSocket + IOCP 하이브리드

```cpp
// WebSocket 핸드셰이크를 IOCP로 처리하는 패턴
class WebSocketServer {
    // 1단계: TCP Accept (IOCP)
    void OnAccept(Session* session) {
        session->state = SESSION_HANDSHAKE;
        PostRecv(session);  // HTTP Upgrade 요청 대기
    }

    // 2단계: HTTP Upgrade 처리
    void OnRecv(Session* session, char* data, int len) {
        if (session->state == SESSION_HANDSHAKE) {
            // HTTP Upgrade 요청 파싱
            // "GET / HTTP/1.1\r\nUpgrade: websocket\r\n..."
            if (ParseWebSocketHandshake(data, len, &wsKey)) {
                // WebSocket Accept 응답 전송
                string response = BuildWebSocketAcceptResponse(wsKey);
                session->Send(response.c_str(), response.length());
                session->state = SESSION_WEBSOCKET;
            }
        }
        else if (session->state == SESSION_WEBSOCKET) {
            // WebSocket 프레임 파싱
            WebSocketFrame frame;
            if (ParseWebSocketFrame(data, len, &frame)) {
                // 디마스킹 (클라이언트 → 서버 데이터는 마스킹됨)
                UnmaskPayload(frame.payload, frame.maskKey, frame.payloadLen);

                // 게임 패킷 처리
                ProcessGamePacket(session, frame.payload, frame.payloadLen);
            }
        }
    }
};

// 장점: 웹 브라우저/모바일 웹뷰에서 직접 접속 가능
// 단점: WebSocket 프레임 오버헤드 (2~14 bytes per frame)
// IOCP의 비동기 처리 능력으로 수만 WebSocket 연결 관리 가능
```

### 10.5.3 매칭 서버

```cpp
// 캐주얼 게임의 매칭은 빠른 매칭이 핵심
class MatchmakingServer {
    // IOCP로 수만 명의 매칭 요청을 동시 처리

    struct MatchRequest {
        Session* session;
        int rating;
        int gameMode;
        DWORD requestTime;
    };

    // 게임 모드별 매칭 큐
    map<int, deque<MatchRequest>> m_queues;

    // 매칭 요청 (IOCP Worker에서)
    void OnMatchRequest(Session* session, int gameMode, int rating) {
        MatchRequest req = { session, rating, gameMode, GetTickCount() };

        // Lock으로 큐 보호
        EnterCriticalSection(&m_queueLock);
        m_queues[gameMode].push_back(req);
        LeaveCriticalSection(&m_queueLock);

        // 매칭 스레드에 알림
        SetEvent(m_matchEvent);
    }

    // 매칭 스레드 (별도 스레드)
    void MatchThread() {
        while (running) {
            WaitForSingleObject(m_matchEvent, 1000);  // 1초마다 또는 이벤트

            EnterCriticalSection(&m_queueLock);
            for (auto& [mode, queue] : m_queues) {
                TryMatch(queue);
            }
            LeaveCriticalSection(&m_queueLock);
        }
    }

    void TryMatch(deque<MatchRequest>& queue) {
        while (queue.size() >= 2) {
            // 레이팅 유사한 2명 매칭 (간단한 예시)
            // 실제로는 ELO 기반 매칭, 대기 시간에 따른 범위 확장 등
            MatchRequest a = queue.front(); queue.pop_front();
            MatchRequest b = queue.front(); queue.pop_front();

            // 게임방 생성 및 통지
            int roomId = CreateGameRoom(a, b);
            SendMatchResult(a.session, roomId);
            SendMatchResult(b.session, roomId);
        }
    }
};
```

---

## 10.6 게임 외 IOCP 활용 분야

### 10.6.1 웹 서버 / API 서버

```
IIS (Internet Information Services):
- Windows의 대표적 웹 서버
- 내부적으로 IOCP를 사용하여 HTTP 요청 처리
- http.sys 커널 드라이버가 IOCP로 I/O 처리

Node.js (libuv):
- Windows에서 libuv가 IOCP를 백엔드로 사용
- 파일 I/O, 네트워크 I/O 모두 IOCP로 처리
- event loop가 내부적으로 GetQueuedCompletionStatusEx 호출

ASP.NET Core (Kestrel):
- .NET의 비동기 I/O가 Windows에서 IOCP 사용
- ThreadPool이 IOCP 기반
```

### 10.6.2 데이터베이스 서버

```
SQL Server:
- 내부적으로 IOCP를 사용하여 네트워크 + 디스크 I/O 처리
- 클라이언트 연결 관리가 IOCP 기반

Redis (Windows 포트):
- Windows 포트에서 IOCP 사용
- 공식 Linux 버전은 epoll 사용
```

### 10.6.3 파일 서버 / 스토리지

```
대용량 파일 서버에서 IOCP 활용:
- 파일 핸들도 IOCP에 연결 가능
- ReadFile + OVERLAPPED → 비동기 파일 읽기
- TransmitFile로 파일 → 소켓 Zero-copy 전송
- 수천 클라이언트 동시 파일 다운로드 처리

예: FTP 서버, 게임 패치 서버, CDN 엣지 서버
```

### 10.6.4 IoT / 실시간 모니터링

```
IoT 게이트웨이:
- 수만 개 센서 디바이스의 TCP 연결 관리
- 각 디바이스에서 주기적 데이터 수신
- IOCP로 대량 연결 효율적 처리

실시간 주식 시세 서버:
- 수천 명에게 초당 수만 건 시세 데이터 브로드캐스트
- 낮은 지연 요구 (ms 단위)
- IOCP + UDP 멀티캐스트 조합
```

---

## 10.7 장르별 비교 정리

| 장르 | 프로토콜 | 틱레이트 | 방 인원 | 서버 역할 | IOCP 부하 | 핵심 과제 |
|------|---------|---------|--------|----------|----------|----------|
| **MMORPG** | TCP | 20~50Hz | 수천명 | 월드 시뮬레이션 | 높음 | AOI, 대규모 동기화 |
| **FPS/TPS** | UDP+TCP | 60~128Hz | 2~100명 | 히트 판정, 물리 | 중간 | 지연 보상, 스냅샷 |
| **RTS** | TCP | 10~20Hz | 2~8명 | 입력 중계 | 낮음 | 결정적 동기화 |
| **MOBA** | TCP | 30Hz | 10명 | 전체 시뮬레이션 | 낮음 | 시야 필터링 |
| **격투** | UDP | 60Hz | 2~4명 | 릴레이/없음 | 매우 낮음 | 롤백, P2P |
| **턴제** | TCP | 이벤트 | 2~8명 | 규칙 검증 | 매우 낮음 | 상태 무결성 |
| **캐주얼** | WS/TCP | 이벤트 | 2~100명 | 매칭, 결과 | 낮음 | 대량 동시접속 |

### 서버 1대 수용량 추정 (8코어, 16GB RAM)

| 장르 | 방당 인원 | 동시 방 수 | 총 동시접속 |
|------|----------|-----------|------------|
| MMORPG | 5,000 | 1 (단일 월드) | 5,000 |
| FPS 배틀로얄 | 100 | 50~100 | 5,000~10,000 |
| MOBA | 10 | 500~1,000 | 5,000~10,000 |
| 격투 (릴레이) | 2 | 5,000~10,000 | 10,000~20,000 |
| 턴제 카드 | 4 | 10,000~50,000 | 40,000~200,000 |
| 캐주얼 | - | - | 50,000~100,000 |

---

## 10.8 퀴즈용 핵심 Q&A

### 정의형 문제

**Q1**: FPS 게임에서 Lag Compensation(서버 되감기)이 필요한 이유는?
> **A**: 네트워크 지연으로 인해 클라이언트가 보는 화면은 서버 상태보다 RTT/2만큼 과거이다. 사격 시 클라이언트 화면에서는 적이 있었지만 서버에서는 이미 이동한 상태일 수 있다. 서버가 사격 시점의 과거 월드 상태로 되감아서 히트 판정을 하면 클라이언트가 본 것과 일치하는 공정한 판정이 가능하다.

**Q2**: Lockstep 방식에서 IOCP 서버의 역할은?
> **A**: Lockstep에서 서버는 게임 시뮬레이션을 실행하지 않는다. 각 틱마다 모든 플레이어의 입력을 수집하고, 전원의 입력이 모이면 InputBundle로 묶어 전체에게 브로드캐스트하는 중계 역할만 한다. 각 클라이언트가 동일한 입력으로 독립적으로 시뮬레이션한다.

### 수치형 문제

**Q3**: 턴제 카드 게임이 IOCP 서버 1대에서 수만 방을 운영할 수 있는 이유는?
> **A**: 턴제 게임은 방당 초당 패킷이 약 1.2개에 불과하다(4명, 10초 턴, 턴당 3패킷). IOCP가 초당 100만+ 패킷을 처리할 수 있으므로, 네트워크 측면에서는 80만 방도 가능하다. 실제로는 메모리와 게임 로직이 제한 요소가 된다.

### 적용형 문제

**Q4**: MOBA 게임에서 적 팀 정보를 클라이언트에게 어떻게 선택적으로 전송하는가?
> **A**: 서버가 Fog of War를 계산하여 각 팀이 시야 안에서 볼 수 있는 적만 해당 팀 클라이언트에게 전송한다. 시야 밖의 적 위치는 아예 보내지 않으므로 메모리 해킹으로도 알 수 없다(맵핵 방지). IOCP 패킷 전송 시 세션별로 다른 데이터를 구성하여 전송한다.

---

## 참고 자료

- Glenn Fiedler, "Networked Physics" (gafferongames.com)
- Valve, "Source Multiplayer Networking" (developer.valvesoftware.com)
- Riot Games Engineering Blog - "Deterministic Lockstep" 관련
- GGPO (ggpo.net) - 롤백 네트코드 오픈소스
- Gabriel Gambetta, "Fast-Paced Multiplayer" (gabrielgambetta.com)
- "1500 Archers on a 28.8: Network Programming in Age of Empires" (GDC, 2001)
