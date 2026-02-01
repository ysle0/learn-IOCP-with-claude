# Section 8: MMORPG 적용 사례 상세 분석

> 작성 완료일: 2026-02-01

---

## 목차
- [8.1 동시접속 5,000명+ 서버 구조](#81-동시접속-5000명-서버-구조)
- [8.2 패킷 암호화/압축 처리 위치](#82-패킷-암호화압축-처리-위치)
- [8.3 채팅, 거래 등 시스템별 처리 방식](#83-채팅-거래-등-시스템별-처리-방식)
- [8.4 로그인 서버, 게임 서버, 채팅 서버 분리 구조](#84-로그인-서버-게임-서버-채팅-서버-분리-구조)
- [8.5 실전 패킷 설계](#85-실전-패킷-설계)
- [8.6 AOI(Area of Interest)와 네트워크 최적화](#86-aoiarea-of-interest와-네트워크-최적화)
- [8.7 퀴즈용 핵심 Q&A](#87-퀴즈용-핵심-qa)

---

## 8.1 동시접속 5,000명+ 서버 구조

### 8.1.1 전체 아키텍처

```
┌──────────────────────────────────────────────────────────────┐
│                         클라이언트                              │
│   [Client 1] [Client 2] ... [Client 5000+]                   │
└──────┬────────────┬────────────────────────┬─────────────────┘
       │            │                        │
       ▼            ▼                        ▼
┌──────────────────────────────────────────────────────────────┐
│                    Load Balancer / Gateway                     │
│   역할: 연결 분산, 패킷 라우팅, DDoS 1차 방어                  │
└──────┬────────────┬────────────────────────┬─────────────────┘
       │            │                        │
       ▼            ▼                        ▼
┌────────────┐ ┌────────────┐ ┌──────────────────────────────┐
│ Login      │ │ Chat       │ │ Game Server Cluster          │
│ Server     │ │ Server     │ │                              │
│            │ │            │ │ ┌──────┐ ┌──────┐ ┌──────┐  │
│ - 인증     │ │ - 채팅     │ │ │Zone 1│ │Zone 2│ │Zone 3│  │
│ - 토큰발급 │ │ - 귓속말   │ │ │Field │ │Dungeon│ │Town  │  │
│ - 서버선택 │ │ - 길드채팅 │ │ └──────┘ └──────┘ └──────┘  │
└────────────┘ └────────────┘ └──────────────────────────────┘
       │            │                        │
       ▼            ▼                        ▼
┌──────────────────────────────────────────────────────────────┐
│                    Shared Infrastructure                       │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────────┐    │
│  │ Database │ │ Redis    │ │ Message  │ │ Monitoring   │    │
│  │ Cluster  │ │ Cache    │ │ Queue    │ │ System       │    │
│  └──────────┘ └──────────┘ └──────────┘ └──────────────┘    │
└──────────────────────────────────────────────────────────────┘
```

### 8.1.2 단일 게임 서버 내부 구조

```cpp
class GameServer {
    // =========================================
    // 네트워크 레이어 (IOCP)
    // =========================================
    HANDLE m_hIOCP;
    HANDLE m_workerThreads[16];    // 8코어 × 2
    int m_workerCount;

    // =========================================
    // 세션 관리
    // =========================================
    SessionPool m_sessionPool;          // Pre-allocated, 최대 6,000개
    SessionManager m_sessionManager;    // 활성 세션 관리

    // =========================================
    // 게임 로직 (단일 또는 소수 스레드)
    // =========================================
    GameWorld m_world;                  // 게임 월드
    PacketQueue m_incomingQueue;        // 네트워크 → 로직 큐
    PacketQueue m_outgoingQueue;        // 로직 → 네트워크 큐

    // =========================================
    // 타이머
    // =========================================
    TimerManager m_timerManager;        // 게임 틱, 버프, 리젠 등
};
```

### 8.1.3 스레드 모델

```
5,000명 동시접속 서버의 스레드 구성 (8코어 기준):

┌─────────────────────────────────────────────────┐
│ IOCP Worker Threads (16개)                       │
│ - GetQueuedCompletionStatus 대기                 │
│ - 패킷 수신/파싱                                  │
│ - 패킷 직렬화/Send                               │
│ - 짧은 처리만 수행, 긴 작업은 큐로 위임            │
└─────────────────────────────────────────────────┘
              │ PacketQueue (Lock-free)
              ▼
┌─────────────────────────────────────────────────┐
│ Game Logic Thread (1~2개)                        │
│ - 이동, 전투, 아이템 처리                         │
│ - 20~50ms 틱 주기 (20~50 FPS 서버 틱)            │
│ - 단일 스레드로 동기화 문제 회피                   │
└─────────────────────────────────────────────────┘
              │
              ▼
┌─────────────────────────────────────────────────┐
│ DB Worker Threads (4~8개)                        │
│ - 비동기 DB 쿼리 처리                             │
│ - 결과를 Game Logic Thread에 전달                 │
└─────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────┐
│ Timer Thread (1개)                                │
│ - 주기적 이벤트 (몬스터 리젠, 버프 만료)           │
│ - PostQueuedCompletionStatus로 로직에 전달         │
└─────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────┐
│ Monitor/Log Thread (1개)                          │
│ - 성능 지표 수집                                   │
│ - 파일 로깅                                       │
└─────────────────────────────────────────────────┘
```

### 8.1.4 메모리 버짓

```
5,000명 기준 메모리 계산:

세션 객체:
  Session 구조체 (소켓, 상태, 플레이어 포인터): ~256 bytes
  Recv Ring Buffer: 8 KB
  Send Ring Buffer: 64 KB
  OVERLAPPED 구조체 × 2 (Recv + Send): ~256 bytes
  ──────────────────────────
  세션당 합계: ~73 KB
  5,000명: ~365 MB

게임 오브젝트:
  Player 객체: ~2 KB × 5,000 = 10 MB
  Monster 객체: ~512 bytes × 10,000 = 5 MB
  Item 객체: ~128 bytes × 50,000 = 6.4 MB
  Map 데이터: ~100 MB
  ──────────────────────────
  게임 데이터 합계: ~121 MB

기타:
  패킷 버퍼 풀: ~50 MB
  DB 쿼리 버퍼: ~20 MB
  로그 버퍼: ~10 MB
  ──────────────────────────
  기타 합계: ~80 MB

총 메모리 사용량: ~566 MB (서버 RAM 4GB 이상 권장)
```

### 8.1.5 네트워크 대역폭 계산

```
5,000명 서버의 대역폭 예측:

수신 (Client → Server):
  클라이언트당 평균 패킷: 10 packets/sec
  평균 패킷 크기: 50 bytes
  5,000 × 10 × 50 = 2.5 MB/s = 20 Mbps

송신 (Server → Client):
  클라이언트당 평균 패킷: 30 packets/sec (브로드캐스트 포함)
  평균 패킷 크기: 100 bytes
  5,000 × 30 × 100 = 15 MB/s = 120 Mbps

총 대역폭: ~140 Mbps (1Gbps NIC로 충분)

피크 시 (대규모 전투):
  브로드캐스트 폭증: 200~300 Mbps
  → 1Gbps NIC로 커버 가능, AOI 최적화 필수
```

---

## 8.2 패킷 암호화/압축 처리 위치

### 8.2.1 처리 위치 선택지

```
옵션 A: IOCP Worker 스레드에서 처리
────────────────────────────────
Recv 완료 → 복호화 → 압축 해제 → 패킷 파싱 → 로직 큐
로직 결과 → 패킷 생성 → 압축 → 암호화 → Send

장점: 구현 단순, 로직 스레드 부담 최소
단점: Worker 스레드가 CPU 집약적 작업으로 바빠져 I/O 처리 지연

옵션 B: 별도 Crypto 스레드 풀에서 처리
────────────────────────────────
Recv 완료 → Crypto Queue → [Crypto Thread] 복호화+해제 → 로직 큐
로직 결과 → Crypto Queue → [Crypto Thread] 암호화+압축 → Send Queue

장점: I/O와 암호화 분리, 확장성
단점: 추가 스레드 풀 관리, 레이턴시 증가 (큐 대기)

옵션 C: Worker에서 경량 처리, 무거운 것만 위임 (권장)
────────────────────────────────
XOR/단순 암호화: Worker에서 직접 (< 1μs)
AES-128/256: 부하에 따라 Worker 또는 위임
압축: 소형 패킷은 안 함, 대형만 별도 처리
```

### 8.2.2 게임별 암호화 전략

```cpp
class PacketCryptor {
    // 간단한 XOR 스트림 암호 (게임에서 흔히 사용)
    // 완전한 보안은 아니지만 캐주얼 치팅 방지에 충분
    struct XorCipher {
        uint8_t key[256];
        int sendIdx = 0;
        int recvIdx = 0;

        void Encrypt(char* data, int len) {
            for (int i = 0; i < len; i++) {
                data[i] ^= key[sendIdx % 256];
                sendIdx++;
            }
        }

        void Decrypt(char* data, int len) {
            for (int i = 0; i < len; i++) {
                data[i] ^= key[recvIdx % 256];
                recvIdx++;
            }
        }
    };

    // 로그인/결제 등 민감 데이터는 AES 사용
    struct AesCipher {
        BCRYPT_KEY_HANDLE hKey;

        void Encrypt(char* data, int len, char* output) {
            ULONG cbResult;
            BCryptEncrypt(hKey, (PUCHAR)data, len, NULL,
                         iv, 16, (PUCHAR)output, len + 16, &cbResult, 0);
        }
    };

    // 하이브리드 전략
    void ProcessIncomingPacket(Session* session, char* data, int len) {
        // 1단계: XOR 복호화 (항상, Worker에서)
        session->xorCipher.Decrypt(data, len);

        PacketHeader* header = (PacketHeader*)data;

        // 2단계: 민감 패킷만 AES 추가 복호화
        if (header->flags & PACKET_FLAG_SECURE) {
            session->aesCipher.Decrypt(data + sizeof(PacketHeader),
                                        len - sizeof(PacketHeader));
        }
    }
};
```

### 8.2.3 압축 전략

```
패킷 크기별 압축 전략:

< 128 bytes: 압축 안 함
  → 압축 오버헤드가 이득보다 큼
  → 게임 패킷 대부분이 이 범위 (이동, 스킬 등)

128~1024 bytes: 선택적 압축
  → 압축률이 30% 이상일 때만 적용
  → 인벤토리, NPC 대화 등

> 1024 bytes: 압축 권장
  → 월드 데이터, 맵 정보, 대량 아이템 목록
  → LZ4 (빠름) 또는 zstd (높은 압축률)

게임에서 흔히 사용하는 압축 알고리즘:
┌──────────┬──────────┬──────────┬──────────┐
│ 알고리즘  │ 속도     │ 압축률    │ 적합 용도 │
├──────────┼──────────┼──────────┼──────────┤
│ LZ4      │ 매우 빠름 │ 보통     │ 실시간    │
│ Snappy   │ 매우 빠름 │ 보통     │ 실시간    │
│ zstd     │ 빠름     │ 높음     │ 준실시간  │
│ zlib     │ 보통     │ 높음     │ 비실시간  │
└──────────┴──────────┴──────────┴──────────┘
```

---

## 8.3 채팅, 거래 등 시스템별 처리 방식

### 8.3.1 채팅 시스템

```cpp
class ChatSystem {
    // 채팅 타입별 처리
    enum ChatType {
        CHAT_NORMAL,    // 주변 채팅 (근거리 브로드캐스트)
        CHAT_WHISPER,   // 귓속말 (1:1)
        CHAT_PARTY,     // 파티 채팅 (소그룹)
        CHAT_GUILD,     // 길드 채팅 (중그룹)
        CHAT_ZONE,      // 존 채팅 (대그룹)
        CHAT_GLOBAL,    // 전체 채팅 (전서버)
    };

    void ProcessChat(Session* sender, ChatPacket* pkt) {
        // 1. 욕설/스팸 필터링
        if (!m_chatFilter.Validate(pkt->message)) {
            SendError(sender, ERR_CHAT_FILTERED);
            return;
        }

        // 2. 쿨타임 체크 (스팸 방지)
        if (!sender->CanChat()) {
            SendError(sender, ERR_CHAT_COOLDOWN);
            return;
        }

        switch (pkt->chatType) {
        case CHAT_NORMAL:
            // AOI 범위 내 플레이어에게만 전송
            BroadcastToNearby(sender, pkt, 50.0f);  // 50m 반경
            break;

        case CHAT_WHISPER:
            // 대상 검색 → 1:1 전송
            if (Session* target = FindPlayerByName(pkt->targetName)) {
                SendTo(target, pkt);
                SendWhisperConfirm(sender, pkt);
            } else {
                // 다른 서버에 있을 수 있음 → 서버 간 라우팅
                RouteToOtherServer(pkt);
            }
            break;

        case CHAT_PARTY:
            // 파티원 전체에게 전송
            if (Party* party = sender->GetParty()) {
                for (auto* member : party->GetMembers()) {
                    SendTo(member->GetSession(), pkt);
                }
            }
            break;

        case CHAT_GUILD:
            // 길드원 전체 (온라인만)
            if (Guild* guild = sender->GetGuild()) {
                for (auto* member : guild->GetOnlineMembers()) {
                    SendTo(member->GetSession(), pkt);
                }
            }
            break;

        case CHAT_GLOBAL:
            // 모든 접속자 → 대역폭 주의
            // 빈도 제한 필수 (예: 10초에 1회)
            BroadcastToAll(pkt);
            break;
        }
    }
};
```

### 8.3.2 거래 시스템

```cpp
class TradeSystem {
    // 거래는 원자성(Atomicity)이 핵심
    // IOCP Worker에서 직접 처리하면 안 됨 → Game Logic Thread에서 처리

    struct TradeSession {
        DWORD player1Id;
        DWORD player2Id;
        vector<Item> player1Offer;  // 플레이어1 제안 아이템
        vector<Item> player2Offer;  // 플레이어2 제안 아이템
        int player1Gold;
        int player2Gold;
        bool player1Confirmed;
        bool player2Confirmed;
        DWORD startTime;
        static const DWORD TRADE_TIMEOUT_MS = 120000;  // 2분
    };

    void ConfirmTrade(TradeSession* trade) {
        // 양쪽 모두 확인 시
        if (!trade->player1Confirmed || !trade->player2Confirmed)
            return;

        // 트랜잭션 시작
        Player* p1 = FindPlayer(trade->player1Id);
        Player* p2 = FindPlayer(trade->player2Id);

        // 1. 검증: 아이템/골드 보유 확인
        if (!ValidateTradeItems(p1, trade->player1Offer) ||
            !ValidateTradeItems(p2, trade->player2Offer) ||
            p1->GetGold() < trade->player1Gold ||
            p2->GetGold() < trade->player2Gold) {
            CancelTrade(trade, ERR_TRADE_INVALID);
            return;
        }

        // 2. 실행: 아이템/골드 교환 (Game Logic Thread에서 실행되므로 단일 스레드)
        for (auto& item : trade->player1Offer) {
            p1->RemoveItem(item);
            p2->AddItem(item);
        }
        for (auto& item : trade->player2Offer) {
            p2->RemoveItem(item);
            p1->AddItem(item);
        }
        p1->AddGold(trade->player2Gold - trade->player1Gold);
        p2->AddGold(trade->player1Gold - trade->player2Gold);

        // 3. DB에 비동기 저장
        m_dbQueue.Push(new SaveTradeQuery(trade));

        // 4. 양쪽에 결과 통지
        SendTradeResult(p1, trade, TRADE_SUCCESS);
        SendTradeResult(p2, trade, TRADE_SUCCESS);

        // 5. 로그 기록 (감사 추적)
        LogTrade(trade);
    }
};
```

### 8.3.3 이동 동기화

```cpp
class MovementSystem {
    // 이동은 가장 빈번한 패킷 (초당 5~10회/클라이언트)
    // 최적화가 가장 중요한 시스템

    void ProcessMovement(Player* player, MovePacket* pkt) {
        // 1. 속도 검증 (핵 방지)
        float distance = Distance(player->GetPos(), pkt->position);
        float maxSpeed = player->GetMaxSpeed() * 1.2f;  // 20% 마진
        float elapsed = (pkt->timestamp - player->lastMoveTime) / 1000.0f;

        if (distance > maxSpeed * elapsed) {
            // 속도 핵 의심 → 위치 보정
            pkt->position = LerpPosition(player->GetPos(),
                                          pkt->position,
                                          maxSpeed * elapsed / distance);
            SendPositionCorrection(player, player->GetPos());
            return;
        }

        // 2. 충돌 검사
        if (!m_navMesh.IsValidPosition(pkt->position)) {
            SendPositionCorrection(player, player->GetPos());
            return;
        }

        // 3. 위치 업데이트
        Vector3 oldPos = player->GetPos();
        player->SetPosition(pkt->position);
        player->SetDirection(pkt->direction);
        player->lastMoveTime = pkt->timestamp;

        // 4. AOI 업데이트 및 브로드캐스트
        m_aoiManager.OnPlayerMove(player, oldPos, pkt->position);

        // 5. 주변 플레이어에게 이동 브로드캐스트
        MoveNotifyPacket notify;
        notify.playerId = player->GetId();
        notify.position = pkt->position;
        notify.direction = pkt->direction;
        notify.speed = pkt->speed;

        BroadcastToNearby(player, &notify, sizeof(notify));
    }
};
```

---

## 8.4 로그인 서버, 게임 서버, 채팅 서버 분리 구조

### 8.4.1 분리 이유

```
단일 서버의 문제:
1. 로그인 폭주 시 게임 서버까지 영향
2. 채팅 대량 발생 시 전투 처리 지연
3. 장애 시 모든 기능 중단
4. 스케일링 어려움 (로그인만 증설 불가)

분리 시 장점:
1. 독립적 스케일링 (로그인 서버만 증설 가능)
2. 장애 격리 (채팅 서버 죽어도 전투 가능)
3. 서버별 최적화 (로그인: I/O 중심, 게임: CPU 중심)
4. 독립적 배포/업데이트
```

### 8.4.2 서버간 통신 구조

```
┌──────────┐    TCP (IOCP)    ┌──────────────┐
│ Client   │ ◄──────────────► │ Login Server │
│          │                   │ (인증 전용)   │
│          │  인증 완료 후      │              │
│          │  토큰 + 서버목록   │              │
│          │  ◄────────────    └──────────────┘
│          │                          │
│          │                    Redis (세션 토큰)
│          │                          │
│          │    TCP (IOCP)    ┌──────────────┐
│          │ ◄──────────────► │ Game Server  │
│          │  토큰으로 재인증   │ (Zone 1)     │
│          │                   │              │
│          │    TCP (IOCP)    ├──────────────┤
│          │ ◄──────────────► │ Chat Server  │
└──────────┘                   └──────────────┘
                                      │
                               서버 간 통신
                               (TCP 또는 Message Queue)
                                      │
                               ┌──────────────┐
                               │ Game Server  │
                               │ (Zone 2)     │
                               └──────────────┘
```

### 8.4.3 로그인 흐름 상세

```
1. 클라이언트 → 로그인 서버: 연결
   [IOCP Accept]

2. 클라이언트 → 로그인 서버: 인증 요청 {id, password_hash}
   로그인 서버: DB 조회 (비동기)
   로그인 서버: 인증 성공 → 세션 토큰 생성 (JWT 또는 랜덤)
   로그인 서버 → Redis: 토큰 저장 {token → userId, expiry: 30s}

3. 로그인 서버 → 클라이언트: 인증 결과 + 서버 목록 + 토큰
   [클라이언트는 로그인 서버와 연결 종료]

4. 클라이언트 → 게임 서버: 연결 + 토큰 전달
   [IOCP Accept]
   게임 서버 → Redis: 토큰 검증
   게임 서버: 세션 생성, 캐릭터 데이터 로드

5. 클라이언트 → 채팅 서버: 연결 + 토큰 전달 (병렬)
   채팅 서버 → Redis: 토큰 검증
   채팅 서버: 채팅 세션 생성
```

### 8.4.4 서버 간 통신 패턴

```cpp
// 서버 간 내부 통신 (IPC)
class InterServerConnection {
    // 패턴 1: 직접 TCP 연결 (IOCP 기반)
    // 각 서버가 다른 서버에 IOCP 클라이언트로 연결
    SOCKET m_serverSocket;
    HANDLE m_hIOCP;

    void ConnectToGameServer(const char* ip, int port) {
        m_serverSocket = WSASocket(AF_INET, SOCK_STREAM, 0,
                                    NULL, 0, WSA_FLAG_OVERLAPPED);
        // ConnectEx로 비동기 연결
        // ...
        CreateIoCompletionPort((HANDLE)m_serverSocket, m_hIOCP, KEY_SERVER, 0);
    }

    // 서버 간 패킷
    void SendToGameServer(ServerPacket* pkt) {
        WSASend(m_serverSocket, &wsaBuf, 1, NULL, 0, &ov, NULL);
    }
};

// 패턴 2: Redis Pub/Sub
// 채팅 서버 → Redis PUBLISH "chat:guild:123" "message"
// 다른 서버의 채팅 서비스 → Redis SUBSCRIBE "chat:guild:123"

// 패턴 3: Message Queue (RabbitMQ, Kafka)
// 높은 처리량, 메시지 보장 필요 시
// 로그, 통계, 비실시간 데이터에 적합
```

### 8.4.5 존(Zone) 서버 간 이동

```cpp
// 서버 이동 (Zone Transfer) 시퀀스
class ZoneTransfer {
    void TransferPlayer(Player* player, int targetZoneId) {
        // 1. 현재 존에서 플레이어 데이터 직렬화
        PlayerSnapshot snapshot;
        player->Serialize(&snapshot);

        // 2. 대상 존 서버에 전송 예약 알림
        ServerPacket transferReq;
        transferReq.type = SVR_TRANSFER_REQUEST;
        transferReq.playerId = player->GetId();
        transferReq.data = snapshot;
        SendToZoneServer(targetZoneId, &transferReq);

        // 3. 대상 서버 응답 대기 (비동기)
        // → 성공 시 클라이언트에 서버 전환 지시
    }

    void OnTransferApproved(DWORD playerId, int targetZoneId,
                             const char* targetIP, int targetPort) {
        // 4. 현재 존에서 플레이어 제거
        Player* player = FindPlayer(playerId);
        RemoveFromWorld(player);

        // 5. 클라이언트에 새 서버 접속 정보 전달
        ServerTransferPacket pkt;
        pkt.serverIP = targetIP;
        pkt.serverPort = targetPort;
        pkt.transferToken = GenerateToken(playerId);
        SendTo(player->GetSession(), &pkt);

        // 6. 세션 정리 (클라이언트가 새 서버에 연결하면)
        // 일정 시간 후 기존 세션 종료
    }
};
```

---

## 8.5 실전 패킷 설계

### 8.5.1 패킷 구조

```cpp
// 패킷 헤더 (고정 크기)
#pragma pack(push, 1)
struct PacketHeader {
    uint16_t size;       // 패킷 전체 크기 (헤더 포함)
    uint16_t id;         // 패킷 ID (프로토콜 식별)
    uint8_t  flags;      // 플래그 (암호화, 압축 등)
    uint8_t  reserved;   // 예약
};
#pragma pack(pop)

// 패킷 ID 범위 설계
enum PacketId : uint16_t {
    // 0x0000~0x00FF: 시스템
    PKT_HEARTBEAT           = 0x0001,
    PKT_DISCONNECT          = 0x0002,

    // 0x0100~0x01FF: 인증
    PKT_LOGIN_REQ           = 0x0100,
    PKT_LOGIN_RES           = 0x0101,
    PKT_CREATE_CHARACTER    = 0x0102,

    // 0x0200~0x02FF: 이동
    PKT_MOVE                = 0x0200,
    PKT_MOVE_NOTIFY         = 0x0201,
    PKT_TELEPORT            = 0x0202,

    // 0x0300~0x03FF: 전투
    PKT_ATTACK              = 0x0300,
    PKT_SKILL_USE           = 0x0301,
    PKT_DAMAGE_NOTIFY       = 0x0302,
    PKT_DEATH_NOTIFY        = 0x0303,

    // 0x0400~0x04FF: 채팅
    PKT_CHAT                = 0x0400,
    PKT_CHAT_NOTIFY         = 0x0401,

    // 0x0500~0x05FF: 아이템/인벤토리
    PKT_ITEM_USE            = 0x0500,
    PKT_ITEM_DROP           = 0x0501,
    PKT_INVENTORY_LIST      = 0x0502,

    // 0x0600~0x06FF: 거래
    PKT_TRADE_REQUEST       = 0x0600,
    PKT_TRADE_OFFER         = 0x0601,
    PKT_TRADE_CONFIRM       = 0x0602,
    PKT_TRADE_CANCEL        = 0x0603,
};
```

### 8.5.2 패킷 핸들러 등록 패턴

```cpp
class PacketDispatcher {
    using HandlerFunc = void(*)(Session*, const char*, int);
    HandlerFunc m_handlers[0xFFFF] = {};

    void Initialize() {
        m_handlers[PKT_LOGIN_REQ]    = HandleLoginReq;
        m_handlers[PKT_MOVE]         = HandleMove;
        m_handlers[PKT_ATTACK]       = HandleAttack;
        m_handlers[PKT_CHAT]         = HandleChat;
        m_handlers[PKT_TRADE_REQUEST]= HandleTradeRequest;
        // ...
    }

    void Dispatch(Session* session, const char* packet, int len) {
        if (len < sizeof(PacketHeader)) return;

        PacketHeader* header = (PacketHeader*)packet;

        // 크기 검증
        if (header->size != len) return;

        // 핸들러 호출
        HandlerFunc handler = m_handlers[header->id];
        if (handler) {
            handler(session, packet + sizeof(PacketHeader),
                    len - sizeof(PacketHeader));
        } else {
            // 알 수 없는 패킷 → 로그 + 무시 (또는 연결 종료)
            LogUnknownPacket(session, header->id);
        }
    }
};
```

### 8.5.3 패킷 파싱과 Ring Buffer

```cpp
// IOCP Recv 완료 후 패킷 경계 처리
class PacketParser {
    RingBuffer m_recvBuffer;  // 8KB Ring Buffer

    // Recv 완료 시 호출 (Worker Thread에서)
    void OnRecvComplete(Session* session, const char* data, int len) {
        m_recvBuffer.Write(data, len);

        // 완전한 패킷이 있는지 반복 체크
        while (m_recvBuffer.GetReadableSize() >= sizeof(PacketHeader)) {
            // 헤더만 먼저 확인 (Peek)
            PacketHeader header;
            m_recvBuffer.Peek(&header, sizeof(header));

            // 패킷 크기 검증
            if (header.size < sizeof(PacketHeader) || header.size > MAX_PACKET_SIZE) {
                // 잘못된 패킷 → 연결 종료
                session->Disconnect();
                return;
            }

            // 완전한 패킷인지 확인
            if (m_recvBuffer.GetReadableSize() < header.size) {
                break;  // 아직 덜 도착함 → 다음 Recv 대기
            }

            // 완전한 패킷 추출
            char packetBuf[MAX_PACKET_SIZE];
            m_recvBuffer.Read(packetBuf, header.size);

            // 패킷 처리
            g_dispatcher.Dispatch(session, packetBuf, header.size);
        }
    }
};
```

---

## 8.6 AOI(Area of Interest)와 네트워크 최적화

### 8.6.1 AOI가 필요한 이유

```
5,000명이 동시에 같은 맵에 있을 때:
- 이동 패킷을 모두에게 브로드캐스트하면:
  5,000 × 5,000 × 10/sec × 30bytes = 7.5 GB/s → 불가능!

AOI 적용 (100m 반경에 평균 50명):
  5,000 × 50 × 10/sec × 30bytes = 75 MB/s → 실현 가능
```

### 8.6.2 격자(Grid) 기반 AOI

```cpp
// 가장 일반적이고 효율적인 AOI 구현
class GridAOI {
    static const int GRID_SIZE = 100;  // 100m × 100m 격자
    static const int MAP_WIDTH = 10000;
    static const int MAP_HEIGHT = 10000;
    static const int GRID_COLS = MAP_WIDTH / GRID_SIZE;    // 100
    static const int GRID_ROWS = MAP_HEIGHT / GRID_SIZE;   // 100

    struct Grid {
        unordered_set<Player*> players;
    };

    Grid m_grids[GRID_ROWS][GRID_COLS];

    // 플레이어 격자 좌표 계산
    pair<int, int> GetGridPos(float x, float z) {
        int col = (int)(x / GRID_SIZE);
        int row = (int)(z / GRID_SIZE);
        col = clamp(col, 0, GRID_COLS - 1);
        row = clamp(row, 0, GRID_ROWS - 1);
        return {row, col};
    }

    // AOI 범위 내 플레이어 수집 (9칸 탐색)
    void GetNearbyPlayers(float x, float z, vector<Player*>& result) {
        auto [row, col] = GetGridPos(x, z);

        for (int dr = -1; dr <= 1; dr++) {
            for (int dc = -1; dc <= 1; dc++) {
                int r = row + dr;
                int c = col + dc;
                if (r < 0 || r >= GRID_ROWS || c < 0 || c >= GRID_COLS)
                    continue;

                for (Player* p : m_grids[r][c].players) {
                    result.push_back(p);
                }
            }
        }
    }

    // 플레이어 이동 시 격자 업데이트
    void OnPlayerMove(Player* player, Vector3 oldPos, Vector3 newPos) {
        auto [oldRow, oldCol] = GetGridPos(oldPos.x, oldPos.z);
        auto [newRow, newCol] = GetGridPos(newPos.x, newPos.z);

        if (oldRow == newRow && oldCol == newCol)
            return;  // 같은 격자 → 업데이트 불필요

        // 기존 격자에서 제거
        m_grids[oldRow][oldCol].players.erase(player);

        // 새 격자에 추가
        m_grids[newRow][newCol].players.insert(player);

        // 시야 변화 처리
        HandleViewChange(player, oldRow, oldCol, newRow, newCol);
    }

    // 시야 변화: 새로 보이는 플레이어 / 사라지는 플레이어 계산
    void HandleViewChange(Player* player,
                           int oldRow, int oldCol,
                           int newRow, int newCol) {
        set<pair<int,int>> oldView, newView;

        // 이전 시야 (9칸)
        for (int dr = -1; dr <= 1; dr++)
            for (int dc = -1; dc <= 1; dc++)
                oldView.insert({oldRow+dr, oldCol+dc});

        // 새 시야 (9칸)
        for (int dr = -1; dr <= 1; dr++)
            for (int dc = -1; dc <= 1; dc++)
                newView.insert({newRow+dr, newCol+dc});

        // 새로 보이는 격자 → Spawn 패킷
        for (auto& cell : newView) {
            if (oldView.find(cell) == oldView.end()) {
                auto [r, c] = cell;
                if (r >= 0 && r < GRID_ROWS && c >= 0 && c < GRID_COLS) {
                    for (Player* other : m_grids[r][c].players) {
                        SendSpawn(player, other);   // 나에게 다른 사람 보여줌
                        SendSpawn(other, player);   // 다른 사람에게 나를 보여줌
                    }
                }
            }
        }

        // 사라지는 격자 → Despawn 패킷
        for (auto& cell : oldView) {
            if (newView.find(cell) == newView.end()) {
                auto [r, c] = cell;
                if (r >= 0 && r < GRID_ROWS && c >= 0 && c < GRID_COLS) {
                    for (Player* other : m_grids[r][c].players) {
                        SendDespawn(player, other);
                        SendDespawn(other, player);
                    }
                }
            }
        }
    }
};
```

### 8.6.3 브로드캐스트 최적화

```cpp
// 같은 데이터를 여러 플레이어에게 보낼 때 최적화

// 방법 1: 공유 버퍼 (Reference Counting)
void BroadcastMoveNotify(Player* mover) {
    MoveNotifyPacket pkt;
    pkt.header.size = sizeof(pkt);
    pkt.header.id = PKT_MOVE_NOTIFY;
    pkt.playerId = mover->GetId();
    pkt.position = mover->GetPos();
    pkt.direction = mover->GetDir();

    // 공유 버퍼 1개 생성
    SharedBuffer* shared = SharedBuffer::Create(&pkt, sizeof(pkt));

    // 주변 플레이어에게 전송
    vector<Player*> nearby;
    m_aoi.GetNearbyPlayers(mover->GetPos().x, mover->GetPos().z, nearby);

    for (Player* other : nearby) {
        if (other != mover) {
            shared->AddRef();
            other->GetSession()->SendShared(shared);
        }
    }
    shared->Release();
}

// 방법 2: Gather Send (WSASend with multiple WSABUF)
// 여러 패킷을 하나의 WSASend로 묶어서 전송
void FlushSendQueue(Session* session) {
    WSABUF bufs[16];
    int bufCount = 0;

    while (!session->sendQueue.empty() && bufCount < 16) {
        auto& item = session->sendQueue.front();
        bufs[bufCount].buf = item.data;
        bufs[bufCount].len = item.length;
        bufCount++;
        session->sendQueue.pop();
    }

    // 한 번의 WSASend로 여러 패킷 전송
    WSASend(session->socket, bufs, bufCount, NULL, 0, &ov, NULL);
}
```

---

## 8.7 퀴즈용 핵심 Q&A

### 정의형 문제

**Q1**: MMORPG 서버에서 네트워크 스레드와 게임 로직 스레드를 분리하는 이유는?
> **A**: (1) IOCP Worker 스레드에서 무거운 게임 로직을 처리하면 I/O 처리가 지연되어 모든 클라이언트 응답성이 저하된다. (2) 게임 로직을 단일(또는 소수) 스레드에서 처리하면 복잡한 동기화 없이 게임 상태를 안전하게 관리할 수 있다. (3) 각 레이어를 독립적으로 스케일링/최적화할 수 있다.

**Q2**: AOI(Area of Interest)란 무엇이며 왜 필요한가?
> **A**: AOI는 각 플레이어의 시야 범위 내 객체만 네트워크로 동기화하는 기법이다. 5,000명 전원에게 모든 이동을 브로드캐스트하면 O(n^2) 패킷이 발생하여 대역폭이 폭발하므로, 격자(Grid) 등으로 공간을 분할하여 근처 플레이어에게만 패킷을 전송한다.

### 수치형 문제

**Q3**: 5,000명 동시접속 서버에서 AOI 없이 이동 브로드캐스트 시 필요한 대역폭은?
> **A**: 5,000 × 5,000 × 10 packets/sec × 30 bytes = 7.5 GB/s. 이는 현실적으로 불가능한 수치이다. AOI를 적용하여 주변 50명에게만 전송하면 5,000 × 50 × 10 × 30 = 75 MB/s로 1Gbps NIC로 처리 가능하다.

**Q4**: 5,000명 서버의 세션 메모리(Recv 8KB + Send 64KB)는 약 얼마인가?
> **A**: 세션당 약 73KB(구조체 + 버퍼 포함), 5,000명 × 73KB ≈ 365MB. 게임 오브젝트, 맵 데이터 등을 합하면 총 ~566MB로 4GB 이상의 서버 RAM이 권장된다.

### 적용형 문제

**Q5**: 거래 시스템을 IOCP Worker 스레드에서 직접 처리하면 어떤 문제가 발생하는가?
> **A**: 여러 Worker 스레드가 동시에 같은 플레이어의 인벤토리를 수정할 수 있어 경합 조건이 발생한다. 아이템 복제 버그, 골드 소실 등 치명적인 문제로 이어질 수 있다. 거래와 같이 원자성이 필요한 작업은 단일 Game Logic Thread에서 순차적으로 처리해야 한다.

**Q6**: 로그인 서버, 게임 서버, 채팅 서버를 분리했을 때, 한 서버의 플레이어가 다른 서버에 있는 플레이어에게 귓속말을 보내려면 어떻게 처리하는가?
> **A**: 채팅 서버가 중앙에서 모든 온라인 플레이어의 위치(어느 게임 서버에 있는지)를 관리하거나, Redis Pub/Sub와 같은 서버 간 메시징을 사용한다. 귓속말 패킷이 채팅 서버에 도착하면, 대상 플레이어가 연결된 서버를 찾아 해당 서버로 메시지를 라우팅한다.

---

## 참고 자료

- Glenn Fiedler, "Networking for Game Programmers" (gafferongames.com)
- 배현직, "게임 서버 프로그래밍 교과서" (한빛미디어)
- Rookiss, "C++/C# 게임서버 프로그래밍" (인프런)
- "Massively Multiplayer Game Development" (Charles River Media)
- GDC Presentations: Server Architecture for MMOs
- Valve Developer Community: Source Multiplayer Networking
- Epic Games: Unreal Engine Networking Overview
