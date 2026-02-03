# IOCP (I/O Completion Port) 완벽 학습 가이드

## 목차
1. [도입 배경 및 문제 정의](#1-도입-배경-및-문제-정의)
2. [IOCP 개념 및 아키텍처](#2-iocp-개념-및-아키텍처)
3. [Quick Start](#3-quick-start)
4. [핵심 Topic별 상세](#4-핵심-topic별-상세)
5. [Best Practices & Patterns](#5-best-practices--patterns)
6. [Advanced Topics](#6-advanced-topics)
7. [IOCP Internals](#7-iocp-internals)
8. [MMORPG 적용 사례](#8-mmorpg-적용-사례)

---

## 1. 도입 배경 및 문제 정의

### 핵심 키워드
- `Blocking I/O`, `Non-blocking I/O`, `Synchronous`, `Asynchronous`
- `Thread-per-connection model`, `Thread pool`
- `C10K Problem`, `Scalability`
- `Context switching overhead`, `Thread contention`

### 문제 상황: 전통적인 I/O 모델의 한계

#### 1.1 Blocking I/O + Thread-per-connection
```
문제점:
- 클라이언트 1개당 스레드 1개 필요
- 10,000명 동시접속 = 10,000개 스레드 필요
- Context switching 비용 폭증
- 메모리 사용량 급증 (스레드당 1MB+ 스택)
- 스레드 생성/소멸 오버헤드
```

#### 1.2 Select/Poll 모델
```
문제점:
- fd_set 크기 제한 (select: 기본 64, 최대 1024)
- O(n) 순회 필요 (모든 소켓 매번 검사)
- 커널-유저 공간 데이터 복사 오버헤드
- Level-triggered 방식의 비효율
```

#### 1.3 IOCP가 해결하는 문제
```
해결책:
- 적은 수의 스레드로 다수의 I/O 처리 (Thread pooling)
- 완료 기반 통지 (Completion-based notification)
- 커널 레벨 최적화된 큐 관리
- 자동 스레드 스케줄링 (Concurrent thread limit)
- Zero-copy 가능한 버퍼 관리
```

### 조사 포인트
- [ ] Windows NT 커널 I/O 서브시스템 발전 역사
- [ ] Proactor 패턴 vs Reactor 패턴 비교
- [ ] epoll(Linux), kqueue(BSD)와의 비교
- [ ] C10K → C10M 문제로의 발전

---

## 2. IOCP 개념 및 아키텍처

### 핵심 키워드
- `Completion Port`, `Completion Packet`
- `OVERLAPPED structure`, `Overlapped I/O`
- `Worker thread`, `I/O thread`
- `GetQueuedCompletionStatus`, `PostQueuedCompletionStatus`
- `CreateIoCompletionPort`

### 아키텍처 다이어그램 (텍스트 기반)
```
┌─────────────────────────────────────────────────────────────┐
│                     Application Layer                        │
├─────────────────────────────────────────────────────────────┤
│  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐        │
│  │Worker 1 │  │Worker 2 │  │Worker 3 │  │Worker N │        │
│  └────┬────┘  └────┬────┘  └────┬────┘  └────┬────┘        │
│       │            │            │            │              │
│       └────────────┴─────┬──────┴────────────┘              │
│                          │                                   │
│              GetQueuedCompletionStatus()                    │
│                          │                                   │
├──────────────────────────┼──────────────────────────────────┤
│                          ▼                                   │
│  ┌───────────────────────────────────────────────────────┐  │
│  │              I/O Completion Port (Kernel)             │  │
│  │  ┌─────────────────────────────────────────────────┐  │  │
│  │  │           Completion Queue (FIFO)               │  │  │
│  │  │  [Packet1] [Packet2] [Packet3] ... [PacketN]   │  │  │
│  │  └─────────────────────────────────────────────────┘  │  │
│  │                                                       │  │
│  │  ┌─────────────────────────────────────────────────┐  │  │
│  │  │        Associated Handles (Sockets/Files)       │  │  │
│  │  │  [Socket1] [Socket2] [File1] [Socket3] ...     │  │  │
│  │  └─────────────────────────────────────────────────┘  │  │
│  └───────────────────────────────────────────────────────┘  │
│                     Kernel Layer                             │
└─────────────────────────────────────────────────────────────┘
```

### 핵심 구조체
```cpp
// OVERLAPPED - 비동기 I/O의 핵심
typedef struct _OVERLAPPED {
    ULONG_PTR Internal;      // I/O 상태
    ULONG_PTR InternalHigh;  // 전송된 바이트 수
    union {
        struct {
            DWORD Offset;
            DWORD OffsetHigh;
        };
        PVOID Pointer;
    };
    HANDLE hEvent;           // 이벤트 핸들 (IOCP에서는 보통 미사용)
} OVERLAPPED;

// 확장된 OVERLAPPED (사용자 정의)
struct OVERLAPPED_EX {
    OVERLAPPED overlapped;
    SOCKET socket;
    WSABUF wsaBuf;
    char buffer[BUFFER_SIZE];
    OperationType opType;    // OP_RECV, OP_SEND, OP_ACCEPT 등
};
```

### 핵심 API 함수
| 함수 | 용도 | 핵심 파라미터 |
|------|------|---------------|
| `CreateIoCompletionPort` | IOCP 생성 및 핸들 연결 | FileHandle, ExistingPort, CompletionKey, NumberOfConcurrentThreads |
| `GetQueuedCompletionStatus` | 완료 패킷 대기/수신 | CompletionPort, lpNumberOfBytes, lpCompletionKey, lpOverlapped, dwMilliseconds |
| `GetQueuedCompletionStatusEx` | 복수 패킷 일괄 수신 | CompletionPort, lpCompletionPortEntries, ulCount, ulNumEntriesRemoved, dwMilliseconds, fAlertable |
| `PostQueuedCompletionStatus` | 사용자 정의 패킷 전송 | CompletionPort, dwNumberOfBytes, dwCompletionKey, lpOverlapped |
| `WSARecv/WSASend` | 비동기 소켓 I/O | 소켓, WSABUF, Overlapped |
| `AcceptEx` | 비동기 Accept | ListenSocket, AcceptSocket, lpOutputBuffer, Overlapped |
| `ConnectEx` | 비동기 Connect | Socket, sockaddr, Overlapped |

### 조사 포인트
- [ ] OVERLAPPED 구조체 Internal 필드의 실제 용도
- [ ] CompletionKey 활용 패턴 (세션 포인터, 인덱스 등)
- [ ] NumberOfConcurrentThreads 최적값 결정 방법
- [ ] IOCP와 Handle 연결 시점 및 해제 처리

---

## 3. Quick Start

### 핵심 키워드
- `WSAStartup`, `WSASocket`, `WSA_FLAG_OVERLAPPED`
- `AcceptEx`, `GetAcceptExSockaddrs`
- `WSARecv`, `WSASend`
- `WSABUF`

### 최소 구현 체크리스트
```
1. [ ] IOCP 핸들 생성
2. [ ] Listen 소켓 생성 및 IOCP 연결
3. [ ] Worker 스레드 풀 생성
4. [ ] AcceptEx로 비동기 Accept 예약
5. [ ] Worker 스레드에서 완료 처리 루프
6. [ ] 정상/비정상 종료 처리
```

### 기본 코드 스켈레톤
```cpp
// 1. IOCP 생성
HANDLE hIOCP = CreateIoCompletionPort(INVALID_HANDLE_VALUE, NULL, 0, 0);

// 2. 리슨 소켓 생성 및 IOCP 연결
SOCKET listenSock = WSASocket(AF_INET, SOCK_STREAM, 0, NULL, 0, WSA_FLAG_OVERLAPPED);
CreateIoCompletionPort((HANDLE)listenSock, hIOCP, (ULONG_PTR)0, 0);

// 3. Worker 스레드 생성
for (int i = 0; i < workerCount; i++) {
    CreateThread(NULL, 0, WorkerThread, hIOCP, 0, NULL);
}

// 4. Worker 스레드 함수
DWORD WINAPI WorkerThread(LPVOID lpParam) {
    HANDLE hIOCP = (HANDLE)lpParam;
    DWORD bytesTransferred;
    ULONG_PTR completionKey;
    OVERLAPPED* pOverlapped;

    while (true) {
        BOOL result = GetQueuedCompletionStatus(
            hIOCP,
            &bytesTransferred,
            &completionKey,
            &pOverlapped,
            INFINITE
        );

        if (!result || bytesTransferred == 0) {
            // 연결 종료 또는 에러 처리
            continue;
        }

        // I/O 완료 처리
        OVERLAPPED_EX* pOverlappedEx = (OVERLAPPED_EX*)pOverlapped;
        switch (pOverlappedEx->opType) {
            case OP_ACCEPT:  HandleAccept(pOverlappedEx); break;
            case OP_RECV:    HandleRecv(pOverlappedEx); break;
            case OP_SEND:    HandleSend(pOverlappedEx); break;
        }
    }
    return 0;
}
```

### 조사 포인트
- [ ] WSA_FLAG_OVERLAPPED 플래그 필수 여부
- [ ] AcceptEx 사용 시 함수 포인터 획득 방법 (WSAIoctl + WSAID_ACCEPTEX)
- [ ] 초기 Accept 예약 개수 결정 기준
- [ ] 버퍼 크기 최적화 (MTU, 페이지 단위)

---

## 4. 핵심 Topic별 상세

### 4.1 Accept 처리

#### 핵심 키워드
- `AcceptEx`, `GetAcceptExSockaddrs`
- `SO_UPDATE_ACCEPT_CONTEXT`
- `Pre-posted accepts`, `Accept 재사용`

#### 조사 포인트
- [ ] AcceptEx의 출력 버퍼 구조 (로컬/원격 주소 + 첫 데이터)
- [ ] SO_UPDATE_ACCEPT_CONTEXT 필수 호출 이유
- [ ] Accept 소켓 재사용 패턴
- [ ] DisconnectEx + TF_REUSE_SOCKET

### 4.2 Recv/Send 처리

#### 핵심 키워드
- `WSARecv`, `WSASend`
- `WSABUF`, `Scatter/Gather I/O`
- `Zero-copy`, `Pending I/O`
- `WSA_IO_PENDING`

#### 조사 포인트
- [ ] 동기 완료 vs 비동기 완료 판단
- [ ] Multiple outstanding recv/send 처리
- [ ] 순서 보장 문제 (특히 Send)
- [ ] Scatter/Gather I/O 활용

### 4.3 세션/버퍼 관리

#### 핵심 키워드
- `Session pooling`, `Object pool`
- `Ring buffer`, `Circular buffer`
- `Lock-free buffer`, `Memory pool`

#### 조사 포인트
- [ ] 세션 객체 생명주기 관리
- [ ] 버퍼 할당/해제 전략
- [ ] 메모리 단편화 방지
- [ ] Reference counting 패턴

### 4.4 스레드 동기화

#### 핵심 키워드
- `Lock-free`, `Atomic operations`
- `Critical Section`, `SRWLock`
- `InterlockedXxx` 함수군
- `Thread-local storage`

#### 조사 포인트
- [ ] IOCP 자체의 스레드 세이프티 범위
- [ ] 세션별 Lock granularity
- [ ] Send 큐잉 동기화 방법
- [ ] 통계/로깅 동기화

---

## 5. Best Practices & Patterns

### 핵심 키워드
- `Graceful shutdown`, `Drain pattern`
- `Backpressure`, `Flow control`
- `Connection timeout`, `Heartbeat`
- `Error handling patterns`

### 패턴 목록

#### 5.1 서버 종료 패턴 (Graceful Shutdown)
```
1. 새 연결 Accept 중지
2. 모든 클라이언트에 종료 통지
3. 진행 중인 I/O 완료 대기 (타임아웃)
4. PostQueuedCompletionStatus로 Worker 종료 신호
5. Worker 스레드 종료 대기
6. 리소스 정리
```

#### 5.2 에러 처리 패턴
```
GetQueuedCompletionStatus 반환값 조합:
- return=TRUE, bytes>0: 정상 완료
- return=TRUE, bytes=0: 정상 연결 종료 (Graceful close)
- return=FALSE, pOV=NULL: 타임아웃 또는 IOCP 에러
- return=FALSE, pOV!=NULL: I/O 에러 (GetLastError로 확인)
```

#### 5.3 Worker 스레드 개수 결정
```
일반 공식: CPU 코어 수 * 2
이유: I/O 대기 시 다른 스레드가 실행 가능
IOCP 내부: NumberOfConcurrentThreads가 실제 동시 실행 제한
```

### 조사 포인트
- [ ] 대용량 파일 전송 시 메모리 관리
- [ ] 느린 클라이언트(Slow client) 처리
- [ ] DoS 공격 방어 패턴
- [ ] 재연결 처리 패턴

---

## 6. Advanced Topics

### 핵심 키워드
- `Registered I/O (RIO)`, `Zero-copy`
- `IO_STATUS_BLOCK`, `NtDeviceIoControlFile`
- `Socket sharing`, `WSADuplicateSocket`
- `TransmitFile`, `TransmitPackets`

### 6.1 Registered I/O (RIO)

#### 개념
```
Windows 8+에서 도입된 고성능 I/O API
- 사전 등록된 버퍼 사용
- 시스템 콜 오버헤드 최소화
- Lock-free 완료 큐 지원
```

#### 핵심 API
- `RIOCreateCompletionQueue`
- `RIOCreateRequestQueue`
- `RIORegisterBuffer`
- `RIOReceive`, `RIOSend`
- `RIODequeueCompletion`

### 6.2 Zero-copy 기법

#### 방법
```
- TransmitFile: 파일 → 소켓 직접 전송
- RIO 사전 등록 버퍼: 페이지 락 회피
- SO_SNDBUF=0: 커널 버퍼 우회 (주의 필요)
```

### 6.3 소켓 공유/마이그레이션

#### 핵심
```
- WSADuplicateSocket: 프로세스 간 소켓 공유
- 로드밸런싱, 핫 업그레이드에 활용
```

### 6.4 타이머 통합

#### 방법
```
- CreateTimerQueueTimer + 콜백에서 PostQueuedCompletionStatus
- CreateThreadpoolTimer
- 또는 별도 타이머 스레드
```

### 조사 포인트
- [ ] RIO vs IOCP 성능 비교
- [ ] RIO 사용 조건 및 제약사항
- [ ] TransmitFile 내부 동작
- [ ] ThreadPool API와 IOCP 통합

---

## 7. IOCP Internals

### 핵심 키워드
- `I/O Request Packet (IRP)`
- `Wait queue`, `Release queue`
- `Last In First Out (LIFO)` 스레드 선택
- `APC`, `_KQUEUE (커널 내부)

### 7.1 커널 내부 구조

```
IOCP 내부 구조:
┌────────────────────────────────────────┐
│           _KQUEUE (커널 내부)           │
├────────────────────────────────────────┤
│  ┌──────────────────────────────────┐  │
│  │       Completion Packet List      │  │ ← 완료된 I/O 패킷 (FIFO)
│  └──────────────────────────────────┘  │
│  ┌──────────────────────────────────┐  │
│  │       Waiting Thread List         │  │ ← 대기 중인 스레드 (LIFO)
│  └──────────────────────────────────┘  │
│  ┌──────────────────────────────────┐  │
│  │       Released Thread List        │  │ ← 깨어났지만 블록된 스레드
│  └──────────────────────────────────┘  │
│  MaximumConcurrency (동시 실행 제한)    │
├────────────────────────────────────────┤
│  Associated Handle List                │
└────────────────────────────────────────┘
```

### 7.2 스레드 스케줄링 메커니즘

```
1. I/O 완료 → 패킷이 Completion List에 추가
2. Waiting Thread 있으면 LIFO로 하나 선택하여 깨움
3. 깨어난 스레드 수 < MaximumConcurrency 유지
4. 실행 중 스레드가 블록되면 → Released List로 이동
5. 다른 대기 스레드를 깨워 동시성 유지
```

### 7.3 LIFO 스레드 선택 이유

```
장점:
- CPU 캐시 지역성 (Cache locality) 극대화
- 최근 실행된 스레드의 컨텍스트가 캐시에 있을 확률 높음
- Context switching 비용 최소화
```

### 7.4 I/O Manager와의 상호작용

```
I/O 요청 흐름:
User Mode: WSARecv() 호출
    ↓
Kernel Mode: NtDeviceIoControlFile / NtReadFile
    ↓
I/O Manager: IRP 생성 및 드라이버에 전달
    ↓
Network Driver: 비동기 I/O 시작
    ↓
(데이터 도착)
    ↓
DPC/ISR: I/O 완료 처리
    ↓
I/O Manager: IRP 완료, IOCP에 패킷 큐잉
    ↓
IOCP: 대기 스레드 깨움
    ↓
User Mode: GetQueuedCompletionStatus 반환
```

### 조사 포인트
- [ ] _KQUEUE 구조체 상세 분석
- [ ] IRP 생명주기
- [ ] AFD.SYS (Ancillary Function Driver) 역할
- [ ] MaximumConcurrency=0 시 동작 (CPU 코어 수 사용)

---

## 8. MMORPG 적용 사례

### 핵심 키워드
- `Game loop integration`
- `Packet serialization/deserialization`
- `Network tick`, `Logical tick`
- `AOI (Area of Interest)`, `Zone/Channel`

### 8.1 게임 서버 아키텍처

```
┌─────────────────────────────────────────────────────────┐
│                    Game Server                           │
├──────────────┬──────────────┬──────────────┬────────────┤
│  Network     │   Logic      │   Database   │  Timer     │
│  Module      │   Module     │   Module     │  Module    │
│  (IOCP)      │   (Game)     │   (Async)    │  (Tick)    │
├──────────────┴──────────────┴──────────────┴────────────┤
│              Message Queue / Command Queue               │
├─────────────────────────────────────────────────────────┤
│                Session Manager                           │
│   ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐          │
│   │Session1│ │Session2│ │Session3│ │SessionN│          │
│   └────────┘ └────────┘ └────────┘ └────────┘          │
└─────────────────────────────────────────────────────────┘
```

### 8.2 설계 고려사항

```
1. 네트워크 스레드 vs 게임 로직 스레드 분리
2. 스레드 간 메시지 큐 설계
3. 패킷 직렬화/역직렬화 처리 위치
4. 브로드캐스트 최적화 (같은 데이터 다중 전송)
5. 연결 상태 머신 관리
```

### 8.3 패킷 처리 흐름

```
Recv 완료 → 패킷 파싱 → 핸들러 호출 →
로직 처리 → 응답 생성 → Send 예약

IOCP 스레드에서 직접 처리 vs 로직 스레드로 위임
- 짧은 처리: IOCP 스레드에서 직접
- 긴 처리: 로직 스레드로 위임 (Producer-Consumer 패턴)
```

### 조사 포인트
- [ ] 동시접속 5,000명+ 서버 구조
- [ ] 패킷 암호화/압축 처리 위치
- [ ] 채팅, 거래 등 시스템별 처리 방식
- [ ] 로그인 서버, 게임 서버, 채팅 서버 분리 구조

---

## 학습 로드맵

```
Week 1: 기초
├── 도입 배경 이해 (Section 1)
├── IOCP 개념/아키텍처 (Section 2)
└── Quick Start 구현 (Section 3)

Week 2: 심화
├── 핵심 Topic 마스터 (Section 4)
└── Best Practices 적용 (Section 5)

Week 3: 고급
├── Advanced Topics (Section 6)
└── Internals 이해 (Section 7)

Week 4: 적용
└── MMORPG 서버 프로토타입 (Section 8)
```

---

## 참고 자료 (추후 조사용)

### 공식 문서
- MSDN I/O Completion Ports
- Windows Internals (Book)
- Winsock Programmer's FAQ

### 오픈소스 참고
- libuv (Node.js 기반 라이브러리)
- ASIO (Boost.Asio)
- Windows SDK 샘플 코드

### 키워드 검색용
```
IOCP, I/O Completion Port, Overlapped I/O,
WSARecv, WSASend, AcceptEx, GetQueuedCompletionStatus,
Proactor pattern, Windows async I/O,
MMORPG server architecture, Game server network
```
