# Section 6: Advanced Topics 상세 분석

> 작성 완료일: 2026-02-01

---

## 목차
- [6.1 Registered I/O (RIO) vs IOCP 성능 비교](#61-registered-io-rio-vs-iocp-성능-비교)
- [6.2 RIO 사용 조건 및 제약사항](#62-rio-사용-조건-및-제약사항)
- [6.3 TransmitFile 내부 동작](#63-transmitfile-내부-동작)
- [6.4 ThreadPool API와 IOCP 통합](#64-threadpool-api와-iocp-통합)
- [6.5 io_uring과의 비교 (참고)](#65-io_uring과의-비교-참고)
- [6.6 퀴즈용 핵심 Q&A](#66-퀴즈용-핵심-qa)

---

## 6.1 Registered I/O (RIO) vs IOCP 성능 비교

### 6.1.1 RIO 개요

Registered I/O(RIO)는 Windows 8/Server 2012에서 도입된 고성능 네트워크 I/O API로, 기존 IOCP의 오버헤드를 최소화하도록 설계되었다.

```
IOCP의 오버헤드 (RIO가 제거하는 것):
1. 매 I/O마다 버퍼 페이지 잠금/해제 (page lock/unlock)
2. 매 I/O마다 시스템 콜 (WSARecv/WSASend → NtDeviceIoControlFile)
3. 완료 큐 접근 시 커널 전환 (GetQueuedCompletionStatus)

RIO의 접근:
1. 버퍼를 사전 등록 → 한 번만 페이지 잠금 (재사용)
2. Request Queue에 직접 쓰기 → 시스템 콜 최소화
3. Completion Queue를 유저 모드에서 직접 폴링 가능
```

### 6.1.2 성능 비교 데이터

```
테스트 환경: Windows Server 2012 R2, 8코어, 10Gbps NIC

시나리오 1: 소형 패킷 (64바이트) 대량 처리
─────────────────────────────────────────
             IOCP        RIO         향상률
PPS:         1.2M/s      2.8M/s      ~133%
CPU:         85%         60%         ~29% 절감
Latency:     45μs        15μs        ~67% 감소

시나리오 2: 중형 패킷 (1KB)
─────────────────────────────────────────
             IOCP        RIO         향상률
Throughput:  8.2Gbps     9.5Gbps     ~16%
CPU:         78%         55%         ~29% 절감

시나리오 3: 대형 패킷 (64KB)
─────────────────────────────────────────
             IOCP        RIO         향상률
Throughput:  9.8Gbps     9.9Gbps     ~1%
CPU:         45%         38%         ~16% 절감

핵심 인사이트:
- 소형 패킷 대량 처리에서 RIO 우위가 극대화 (PPS 중심)
- 대용량 전송에서는 차이 미미 (bottleneck이 NIC/대역폭)
- CPU 효율은 모든 시나리오에서 RIO가 우수
```

### 6.1.3 RIO 아키텍처

```
IOCP 흐름:
  App → WSARecv() → 커널 전환 → IRP 생성 → 버퍼 잠금 → 드라이버
  ← GetQueuedCompletionStatus() ← 커널 전환 ← 버퍼 해제

RIO 흐름:
  [초기화] 버퍼 등록 (RIORegisterBuffer) → 한 번만 페이지 잠금
  [I/O]   RIOReceive() → Request Queue에 직접 쓰기 (가벼움)
  [완료]  RIODequeueCompletion() → Completion Queue에서 직접 읽기

┌─────────────────────────────────────────────┐
│            Application                       │
│                                              │
│  ┌──────────────────────────────────────┐   │
│  │     Registered Buffer Pool           │   │  ← RIORegisterBuffer (1회)
│  │  [Buf0][Buf1][Buf2]...[BufN]        │   │
│  └──────────────────────────────────────┘   │
│                                              │
│  ┌─────────────┐  ┌─────────────────────┐   │
│  │ Request      │  │ Completion          │   │
│  │ Queue (RQ)  │  │ Queue (CQ)          │   │
│  │  [Req1]     │  │  [Result1]          │   │
│  │  [Req2]     │  │  [Result2]          │   │
│  │  [Req3]     │  │                      │   │
│  └──────┬──────┘  └──────────┬──────────┘   │
│         │ RIOReceive/Send     │ RIODequeue   │
│         │ (경량)              │ (폴링 가능)   │
├─────────┼─────────────────────┼──────────────┤
│         ▼                     ▲              │
│  ┌──────────────────────────────────────┐   │
│  │          Kernel / NIC Driver          │   │
│  └──────────────────────────────────────┘   │
└─────────────────────────────────────────────┘
```

### 6.1.4 RIO 기본 사용 코드

```cpp
// ============================================================
// RIO 초기화
// ============================================================

// 1. RIO 함수 테이블 획득
RIO_EXTENSION_FUNCTION_TABLE g_rio;
GUID functionTableId = WSAID_MULTIPLE_RIO;
DWORD dwBytes;
WSAIoctl(socket, SIO_GET_MULTIPLE_EXTENSION_FUNCTION_POINTER,
         &functionTableId, sizeof(GUID),
         &g_rio, sizeof(g_rio), &dwBytes, NULL, NULL);

// 2. 버퍼 등록 (사전 페이지 잠금)
const int BUFFER_COUNT = 1024;
const int BUFFER_SIZE = 1024;
char* bufferPool = (char*)VirtualAlloc(NULL,
    BUFFER_COUNT * BUFFER_SIZE,
    MEM_COMMIT | MEM_RESERVE, PAGE_READWRITE);

RIO_BUFFERID bufferId = g_rio.RIORegisterBuffer(
    bufferPool,
    BUFFER_COUNT * BUFFER_SIZE
);

// 3. Completion Queue 생성
RIO_CQ completionQueue = g_rio.RIOCreateCompletionQueue(
    BUFFER_COUNT,   // 큐 크기
    NULL            // 통지 방법 (NULL = 폴링)
);

// 4. Request Queue 생성 (소켓당 1개)
RIO_RQ requestQueue = g_rio.RIOCreateRequestQueue(
    socket,
    10,              // 최대 미완료 Recv
    1,               // Recv 버퍼 수
    10,              // 최대 미완료 Send
    1,               // Send 버퍼 수
    completionQueue, // Recv 완료 큐
    completionQueue, // Send 완료 큐
    NULL             // 컨텍스트
);

// ============================================================
// RIO I/O 수행
// ============================================================

// 5. Recv 요청
RIO_BUF rioBuf;
rioBuf.BufferId = bufferId;
rioBuf.Offset = 0;          // 버퍼 풀 내 오프셋
rioBuf.Length = BUFFER_SIZE;

g_rio.RIOReceive(requestQueue, &rioBuf, 1, 0, NULL);

// 6. 완료 폴링
RIORESULT results[64];
ULONG numResults = g_rio.RIODequeueCompletion(
    completionQueue, results, 64
);

for (ULONG i = 0; i < numResults; i++) {
    // results[i].BytesTransferred - 전송 바이트
    // results[i].RequestContext - 요청 시 전달한 컨텍스트
    // results[i].Status - 결과 상태
    ProcessResult(&results[i]);
}
```

---

## 6.2 RIO 사용 조건 및 제약사항

### 6.2.1 시스템 요구사항

| 요구사항 | 상세 |
|---------|------|
| OS | Windows 8 / Server 2012 이상 |
| Winsock | 2.2 이상 |
| 소켓 타입 | TCP, UDP (SOCK_STREAM, SOCK_DGRAM) |
| 주소 체계 | AF_INET, AF_INET6 |
| NIC | RSS(Receive Side Scaling) 지원 권장 |

### 6.2.2 핵심 제약사항

```
1. Accept/Connect는 RIO로 할 수 없음
   → AcceptEx/ConnectEx + IOCP로 처리 후 RIO로 전환 필요
   → 하이브리드 아키텍처가 필수

2. 등록된 버퍼만 사용 가능
   → 임의의 메모리 주소를 I/O에 사용할 수 없음
   → 사전에 RIORegisterBuffer로 등록해야 함
   → 등록 가능한 총 크기에 제한 있음

3. Scatter/Gather I/O 제한
   → RIOReceive/RIOSend에서 버퍼 슬라이스 1개만 지원하는 경우가 일반적
   → 복수 WSABUF 대비 제한적

4. Completion Queue 크기 제한
   → 생성 시 크기 고정, 런타임 확장 불가
   → 충분히 크게 만들어야 함

5. 디버깅 어려움
   → 일반 소켓 API보다 추상화 레벨이 낮음
   → 에러 추적이 어려움, ETW 트레이싱 활용 필요
```

### 6.2.3 RIO 완료 통지 방식

```cpp
// 방식 1: 폴링 (Polling) - 최저 지연
// 전용 스레드에서 busy-wait
while (running) {
    ULONG count = g_rio.RIODequeueCompletion(cq, results, 64);
    if (count == 0) {
        YieldProcessor();  // SpinWait
        continue;
    }
    ProcessResults(results, count);
}
// 장점: 최저 지연 (커널 전환 없음)
// 단점: CPU 코어 1개 100% 점유

// 방식 2: IOCP 통지 - 균형 잡힌 접근
RIO_NOTIFICATION_COMPLETION notify;
notify.Type = RIO_IOCP_COMPLETION;
notify.Iocp.IocpHandle = hIOCP;
notify.Iocp.CompletionKey = RIO_KEY;
notify.Iocp.Overlapped = &rioOverlapped;

RIO_CQ cq = g_rio.RIOCreateCompletionQueue(1024, &notify);

// CQ에 결과가 들어오면 IOCP에 통지
g_rio.RIONotify(cq);

// Worker에서 IOCP로 통지받으면 폴링으로 대량 처리
// → IOCP의 전력 효율 + 폴링의 대량 처리 장점 결합

// 방식 3: 이벤트 (Event) 통지
RIO_NOTIFICATION_COMPLETION notify;
notify.Type = RIO_EVENT_COMPLETION;
notify.Event.EventHandle = hEvent;
notify.Event.NotifyReset = TRUE;  // 자동 리셋

RIO_CQ cq = g_rio.RIOCreateCompletionQueue(1024, &notify);
```

### 6.2.4 IOCP에서 RIO로 마이그레이션 가이드

```
마이그레이션 단계:

1단계: 하이브리드 아키텍처 설계
  - Accept: IOCP (AcceptEx)
  - Data I/O: RIO (RIOReceive/RIOSend)

2단계: 버퍼 관리 재설계
  - 기존: malloc/free로 자유 할당
  - RIO: 사전 등록된 대형 버퍼 풀에서 슬라이스 할당

3단계: Worker 스레드 구조 변경
  - Accept Worker: GQCS 루프 (기존)
  - Data Worker: RIODequeueCompletion 루프 (신규)

4단계: 점진적 전환
  - 먼저 UDP 데이터 처리를 RIO로 전환 (단순함)
  - 이후 TCP 데이터 처리도 RIO로 전환
```

---

## 6.3 TransmitFile 내부 동작

### 6.3.1 TransmitFile의 커널 레벨 동작

```
TransmitFile 호출 시 커널 내부:

1. 파일 매니저가 파일 데이터를 캐시 매니저로부터 읽음
   → 이미 캐시에 있으면 즉시 사용
   → 없으면 디스크 I/O (비동기)

2. 캐시 매니저의 페이지가 직접 네트워크 드라이버에 전달
   → 유저 모드 버퍼 복사 없음 (Zero-copy)
   → 캐시 페이지 → MDL → NIC DMA

3. 네트워크 드라이버가 DMA로 NIC에 전송
   → CPU가 데이터 복사에 관여하지 않음

일반 WSASend와의 비교:
  WSASend: 유저 버퍼 → 커널 버퍼 → NIC (2회 복사)
  TransmitFile: 캐시 → NIC (0회 유저 복사)
```

### 6.3.2 TransmitFile 상세 사용법

```cpp
// 기본 사용법
BOOL TransmitFile(
    SOCKET hSocket,              // 대상 소켓
    HANDLE hFile,                // 파일 핸들
    DWORD nNumberOfBytesToWrite, // 0 = 전체 파일
    DWORD nNumberOfBytesPerSend, // 0 = 시스템 기본값 (65536)
    LPOVERLAPPED lpOverlapped,   // 비동기용
    LPTRANSMIT_FILE_BUFFERS lpTransmitBuffers, // 헤더/트레일러
    DWORD dwReserved             // 플래그
);

// HTTP 응답 전송 예제
void SendHttpFileResponse(SOCKET sock, const wchar_t* filePath,
                          OVERLAPPED* pOv) {
    HANDLE hFile = CreateFile(filePath, GENERIC_READ, FILE_SHARE_READ,
                              NULL, OPEN_EXISTING,
                              FILE_FLAG_SEQUENTIAL_SCAN, NULL);

    // HTTP 헤더
    char header[] = "HTTP/1.1 200 OK\r\n"
                    "Content-Type: application/octet-stream\r\n"
                    "\r\n";

    TRANSMIT_FILE_BUFFERS tfBuf = {};
    tfBuf.Head = header;
    tfBuf.HeadLength = sizeof(header) - 1;
    // Tail은 사용하지 않음

    TransmitFile(
        sock,
        hFile,
        0,        // 전체 파일
        0,        // 시스템 기본 청크 크기
        pOv,
        &tfBuf,
        TF_USE_KERNEL_APC  // 커널 APC로 완료 통지
    );

    // 주의: hFile은 TransmitFile 완료 후 닫아야 함
}
```

### 6.3.3 TransmitFile 플래그

```cpp
// TF_DISCONNECT: 전송 완료 후 소켓 연결 종료
// HTTP/1.0 Keep-Alive 없을 때 유용
TransmitFile(sock, hFile, 0, 0, &ov, &tfBuf,
             TF_DISCONNECT);

// TF_REUSE_SOCKET: 전송 완료 후 소켓을 재사용 가능 상태로
// AcceptEx에서 다시 사용할 수 있음
TransmitFile(sock, hFile, 0, 0, &ov, &tfBuf,
             TF_DISCONNECT | TF_REUSE_SOCKET);

// TF_USE_DEFAULT_WORKER: 기본 스레드 풀 사용
// TF_USE_SYSTEM_THREAD: 시스템 스레드 사용
// TF_USE_KERNEL_APC: 커널 APC로 완료 (IOCP와 통합 시 사용)

// TF_WRITE_BEHIND: 전송 완료를 기다리지 않고 즉시 반환
// 커널 버퍼에 데이터가 복사되면 완료 처리
// → 실제 네트워크 전송 완료 전에 알림이 옴 (주의)
```

### 6.3.4 TransmitFile의 제한사항

```
1. 동시 TransmitFile 수 제한
   → 시스템 레벨 제한 (기본값: 워커 스레드 풀 크기)
   → 초과 시 대기 (큐잉)

2. 파일 핸들 요구사항
   → FILE_FLAG_SEQUENTIAL_SCAN 권장 (캐시 힌트)
   → FILE_FLAG_NO_BUFFERING은 사용 불가 (캐시 매니저 필요)

3. 소켓 요구사항
   → 연결된 TCP 소켓만 (UDP 불가)
   → OVERLAPPED 소켓이어야 함

4. 최대 전송 크기
   → 32비트: 2GB 제한 (DWORD 범위)
   → 64비트: 제한 없음 (nNumberOfBytesToWrite = 0으로 전체 전송)
```

### 6.3.5 TransmitPackets와의 비교

```cpp
// TransmitPackets: 파일과 메모리 데이터를 혼합 전송
TRANSMIT_PACKETS_ELEMENT elements[3];

// 첫 번째: 메모리 데이터 (헤더)
elements[0].dwElFlags = TP_ELEMENT_MEMORY;
elements[0].pBuffer = headerData;
elements[0].cLength = headerLen;

// 두 번째: 파일 데이터 (본문)
elements[1].dwElFlags = TP_ELEMENT_FILE;
elements[1].hFile = hFile;
elements[1].nFileOffset.QuadPart = 0;
elements[1].cLength = fileSize;

// 세 번째: 메모리 데이터 (트레일러)
elements[2].dwElFlags = TP_ELEMENT_MEMORY;
elements[2].pBuffer = trailerData;
elements[2].cLength = trailerLen;

lpfnTransmitPackets(
    sock,
    elements,
    3,          // 엘리먼트 수
    0,          // 전송 크기 (0 = 전체)
    &ov,
    TF_USE_KERNEL_APC
);

// TransmitFile vs TransmitPackets:
// TransmitFile: 단일 파일 + 선택적 헤더/트레일러
// TransmitPackets: 파일/메모리 조각 자유 조합 (더 유연)
```

---

## 6.4 ThreadPool API와 IOCP 통합

### 6.4.1 Windows Thread Pool 개요

Windows Vista부터 제공되는 Thread Pool API는 내부적으로 IOCP를 사용한다. 직접 IOCP를 관리하는 대신 Thread Pool API를 사용하면 코드가 단순해진다.

```
Thread Pool API 구성:
- Work: 스레드 풀에서 실행할 작업 항목
- Timer: 주기적/지연 실행
- Wait: 커널 객체 대기
- I/O: 비동기 I/O 완료 처리 ← IOCP 대체 가능
- Cleanup Group: 리소스 일괄 정리
```

### 6.4.2 Thread Pool I/O로 IOCP 대체

```cpp
// 전통적 IOCP
HANDLE hIOCP = CreateIoCompletionPort(INVALID_HANDLE_VALUE, NULL, 0, 0);
CreateIoCompletionPort((HANDLE)socket, hIOCP, key, 0);
// Worker 스레드 생성 및 관리 필요...

// Thread Pool I/O (Vista+)
PTP_IO pTpIo = CreateThreadpoolIo(
    (HANDLE)socket,
    IoCompletionCallback,  // 콜백 함수
    pContext,              // 콜백 컨텍스트
    NULL                   // 환경 (기본 스레드 풀)
);

// I/O 시작 전 반드시 호출
StartThreadpoolIo(pTpIo);

// 비동기 I/O 수행
WSARecv(socket, &wsaBuf, 1, NULL, &flags, &ov, NULL);

// 콜백 함수
VOID CALLBACK IoCompletionCallback(
    PTP_CALLBACK_INSTANCE Instance,
    PVOID Context,
    PVOID Overlapped,
    ULONG IoResult,        // ERROR_SUCCESS 또는 에러 코드
    ULONG_PTR BytesTransferred,
    PTP_IO Io
) {
    if (IoResult == ERROR_SUCCESS) {
        // 정상 완료 처리
        ProcessData(Context, Overlapped, BytesTransferred);

        // 다음 I/O 전에 다시 호출 필요
        StartThreadpoolIo(Io);
        WSARecv(...);
    } else {
        // 에러 처리
        CloseSession(Context);
    }
}
```

### 6.4.3 IOCP 직접 관리 vs Thread Pool API 비교

| 기준 | IOCP 직접 | Thread Pool API |
|------|----------|----------------|
| 코드 복잡도 | 높음 (Worker 관리) | 낮음 (콜백 기반) |
| 세밀한 제어 | O (스레드 수, 처리 순서) | 제한적 |
| 성능 | 최적화 여지 큼 | 충분히 좋음 |
| 디버깅 | 직접 제어 가능 | 블랙박스 |
| 권장 용도 | 고성능 서버 | 일반 애플리케이션 |

### 6.4.4 Thread Pool 환경 커스터마이징

```cpp
// 커스텀 스레드 풀 생성
PTP_POOL pool = CreateThreadpool(NULL);
SetThreadpoolThreadMinimum(pool, 2);
SetThreadpoolThreadMaximum(pool, 8);

// 콜백 환경 설정
TP_CALLBACK_ENVIRON callbackEnv;
InitializeThreadpoolEnvironment(&callbackEnv);
SetThreadpoolCallbackPool(&callbackEnv, pool);

// Cleanup Group (리소스 일괄 정리)
PTP_CLEANUP_GROUP cleanupGroup = CreateThreadpoolCleanupGroup();
SetThreadpoolCallbackCleanupGroup(&callbackEnv, cleanupGroup, NULL);

// 이 환경으로 I/O 객체 생성
PTP_IO pTpIo = CreateThreadpoolIo(
    (HANDLE)socket,
    IoCompletionCallback,
    pContext,
    &callbackEnv  // 커스텀 환경
);

// 종료 시 일괄 정리
CloseThreadpoolCleanupGroupMembers(cleanupGroup, FALSE, NULL);
CloseThreadpoolCleanupGroup(cleanupGroup);
CloseThreadpool(pool);
```

---

## 6.5 io_uring과의 비교 (참고)

### 6.5.1 Linux io_uring 개요 (5.1+, 2019)

Linux에서 IOCP/RIO에 대응하는 최신 비동기 I/O 인터페이스이다. 설계 철학에서 RIO와 유사점이 많다.

### 6.5.2 아키텍처 비교

```
IOCP (Windows, NT 3.5+):
  커널 객체 기반, GetQueuedCompletionStatus로 완료 수신
  시스템 콜마다 커널 전환

RIO (Windows 8+):
  사전 등록 버퍼, 유저모드 폴링 가능
  시스템 콜 최소화

io_uring (Linux 5.1+):
  Submission Queue + Completion Queue (공유 메모리)
  시스템 콜 없이 I/O 요청/완료 가능 (SQPOLL 모드)
```

### 6.5.3 상세 비교표

| 특성 | IOCP | RIO | io_uring |
|------|------|-----|----------|
| 도입 시기 | NT 3.5 (1994) | Win8 (2012) | Linux 5.1 (2019) |
| I/O 요청 방식 | WSARecv (시스템 콜) | RIOReceive (경량) | SQE 큐잉 (시스템 콜 불필요*) |
| 완료 수신 방식 | GQCS (시스템 콜) | RIODequeue (폴링 가능) | CQE 폴링 (시스템 콜 불필요*) |
| 버퍼 관리 | 자유 (매번 잠금) | 사전 등록 | 사전 등록 (선택) |
| 지원 I/O | 소켓, 파일, 파이프 | 소켓만 | 소켓, 파일, 파이프 + 더 많음 |
| Accept/Connect | AcceptEx (IOCP) | 불가 (IOCP 필요) | 가능 (IORING_OP_ACCEPT) |
| 링크된 작업 | 불가 | 불가 | 가능 (SQE chain) |
| Kernel-bypass | 불가 | 불가 | SQPOLL 모드 |
| 성숙도 | 매우 높음 (30년) | 높음 | 빠르게 성장 중 |

* io_uring의 SQPOLL 모드에서는 커널 스레드가 SQ를 폴링하므로 유저 측 시스템 콜이 전혀 불필요

### 6.5.4 프로그래밍 모델 비교

```cpp
// === IOCP ===
CreateIoCompletionPort(socket, hIOCP, key, 0);
WSARecv(socket, &buf, 1, NULL, &flags, &ov, NULL);
GetQueuedCompletionStatus(hIOCP, &bytes, &key, &ov, INFINITE);

// === RIO ===
RIORegisterBuffer(pool, size);
RIOReceive(rq, &rioBuf, 1, 0, ctx);
RIODequeueCompletion(cq, results, count);

// === io_uring ===
// struct io_uring ring;
// io_uring_queue_init(256, &ring, 0);
// struct io_uring_sqe *sqe = io_uring_get_sqe(&ring);
// io_uring_prep_recv(sqe, fd, buf, len, 0);
// io_uring_submit(&ring);
// struct io_uring_cqe *cqe;
// io_uring_wait_cqe(&ring, &cqe);
```

---

## 6.6 퀴즈용 핵심 Q&A

### 정의형 문제

**Q1**: RIO가 IOCP 대비 성능이 우수한 근본적 이유는?
> **A**: RIO는 (1) 사전 등록된 버퍼로 매 I/O마다 페이지 잠금/해제를 제거하고, (2) 유저 모드에서 직접 Completion Queue를 폴링할 수 있어 커널 전환을 최소화하며, (3) Request Queue에 경량 쓰기로 시스템 콜 오버헤드를 줄인다. 이 세 가지 오버헤드 제거가 특히 소형 패킷 대량 처리에서 큰 차이를 만든다.

**Q2**: TransmitFile이 Zero-copy인 이유는?
> **A**: TransmitFile은 파일 데이터를 유저 모드 버퍼에 복사하지 않고, 커널의 캐시 매니저 페이지를 MDL(Memory Descriptor List)을 통해 직접 NIC 드라이버에 전달한다. NIC는 DMA로 해당 물리 페이지를 읽으므로 CPU가 데이터 복사에 관여하지 않는다.

**Q3**: Thread Pool API가 내부적으로 IOCP를 사용한다는 것은 무슨 의미인가?
> **A**: Windows Thread Pool의 I/O 완료 콜백(`CreateThreadpoolIo`)은 내부적으로 비공개 IOCP를 생성하여 비동기 I/O 완료를 처리한다. 개발자는 IOCP를 직접 관리하지 않고 콜백 함수만 등록하면 된다.

### 수치형 문제

**Q4**: RIO는 소형 패킷(64B) 처리에서 IOCP 대비 어느 정도 성능 향상을 보이는가?
> **A**: 약 133% (초당 패킷 수 기준). IOCP ~1.2M PPS → RIO ~2.8M PPS. 대형 패킷에서는 차이가 1% 미만으로 줄어드는데, 이는 NIC 대역폭이 병목이 되기 때문이다.

### 적용형 문제

**Q5**: IOCP 서버를 RIO로 마이그레이션할 때 Accept/Connect 처리는 어떻게 해야 하는가?
> **A**: RIO는 Accept/Connect를 지원하지 않으므로, 하이브리드 아키텍처를 구성해야 한다. 연결 수립은 기존 IOCP(AcceptEx/ConnectEx)로 처리하고, 연결이 완료된 소켓의 데이터 I/O만 RIO로 전환한다. 두 시스템을 병행 운영하는 아키텍처 설계가 필요하다.

**Q6**: TransmitFile에 `TF_WRITE_BEHIND` 플래그를 사용하면 어떤 위험이 있는가?
> **A**: 커널 버퍼에 데이터가 복사되면 바로 완료 통지가 오지만, 실제 네트워크 전송은 아직 진행 중일 수 있다. 완료 통지 후 파일을 삭제하거나 수정하면 전송 중인 데이터에는 영향이 없지만(이미 커널 복사됨), 완료 = 상대방 수신이라고 가정하면 안 된다. 파일 핸들은 완료 통지 후 닫아도 안전하다.

---

## 참고 자료

- Microsoft Docs: [Registered I/O API Reference](https://learn.microsoft.com/en-us/previous-versions/windows/desktop/legacy/hh437560(v=vs.85))
- Microsoft Docs: [TransmitFile function](https://learn.microsoft.com/en-us/windows/win32/api/mswsock/nf-mswsock-transmitfile)
- Microsoft Docs: [Thread Pool API](https://learn.microsoft.com/en-us/windows/win32/procthread/thread-pool-api)
- Windows Internals, 7th Edition - Chapter 6: I/O System, Chapter 4: Threads
- "RIO - High Speed Networking" (MSDN Blog, archived)
- Axel Rasmussen, "io_uring is not an event system" (2020)
- Jens Axboe, "Efficient IO with io_uring" (kernel.dk, 2019)
