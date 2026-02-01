# Section 2: IOCP 개념 및 아키텍처 상세 분석

> 작성 완료일: 2026-02-01

---

## 목차
- [2.1 OVERLAPPED 구조체 Internal 필드의 실제 용도](#21-overlapped-구조체-internal-필드의-실제-용도)
- [2.2 CompletionKey 활용 패턴](#22-completionkey-활용-패턴)
- [2.3 NumberOfConcurrentThreads 최적값 결정 방법](#23-numberofconcurrentthreads-최적값-결정-방법)
- [2.4 IOCP와 Handle 연결 시점 및 해제 처리](#24-iocp와-handle-연결-시점-및-해제-처리)
- [2.5 IOCP 동작 모델 상세](#25-iocp-동작-모델-상세)
- [2.6 Completion Packet의 생명주기](#26-completion-packet의-생명주기)
- [2.7 퀴즈용 핵심 Q&A](#27-퀴즈용-핵심-qa)

---

## 2.1 OVERLAPPED 구조체 Internal 필드의 실제 용도

### 2.1.1 Internal과 InternalHigh의 정체

`OVERLAPPED` 구조체의 `Internal`과 `InternalHigh` 필드는 MSDN에서 "시스템 사용을 위해 예약됨"이라고 기술하지만, 실제로는 커널의 `IO_STATUS_BLOCK` 구조체와 메모리 레이아웃이 일치하도록 설계되어 있다.

```cpp
// 커널 모드의 IO_STATUS_BLOCK
typedef struct _IO_STATUS_BLOCK {
    union {
        NTSTATUS Status;    // I/O 완료 상태
        PVOID    Pointer;   // 정렬용 (64비트)
    };
    ULONG_PTR Information;  // 전송된 바이트 수
} IO_STATUS_BLOCK;

// 유저 모드의 OVERLAPPED
typedef struct _OVERLAPPED {
    ULONG_PTR Internal;      // == IO_STATUS_BLOCK.Status
    ULONG_PTR InternalHigh;  // == IO_STATUS_BLOCK.Information
    union {
        struct {
            DWORD Offset;
            DWORD OffsetHigh;
        };
        PVOID Pointer;
    };
    HANDLE hEvent;
} OVERLAPPED;
```

#### 필드별 상세 의미

| 필드 | 대응 커널 구조 | 실제 용도 |
|------|---------------|----------|
| `Internal` | `IO_STATUS_BLOCK.Status` | NTSTATUS 코드 저장. 비동기 I/O 진행 중이면 `STATUS_PENDING`(0x103) |
| `InternalHigh` | `IO_STATUS_BLOCK.Information` | 완료된 I/O의 전송 바이트 수 |

#### I/O 상태 판단에 활용하는 방법

```cpp
// 비동기 I/O가 아직 진행 중인지 확인
bool IsIoPending(OVERLAPPED* pOv) {
    return pOv->Internal == STATUS_PENDING;  // 0x00000103
}

// HasOverlappedIoCompleted 매크로 (winbase.h에 정의)
#define HasOverlappedIoCompleted(lpOverlapped) \
    (((DWORD)(lpOverlapped)->Internal) != STATUS_PENDING)
```

#### 주의사항

- `Internal` 필드를 직접 읽는 것은 문서화된 `HasOverlappedIoCompleted` 매크로를 통해서만 권장
- 비동기 I/O가 진행 중인 동안 OVERLAPPED 구조체의 메모리를 해제하면 **커널이 해제된 메모리에 쓰기**를 시도하여 크래시 발생
- `GetOverlappedResult` 함수가 내부적으로 이 필드들을 읽어서 결과를 반환

### 2.1.2 OVERLAPPED의 Offset/OffsetHigh 필드

파일 I/O에서는 읽기/쓰기 위치를 지정하는 데 사용하지만, **소켓 I/O에서는 이 필드를 사용하지 않는다**. 소켓은 순차적 스트림이므로 오프셋 개념이 없다.

```cpp
// 파일 I/O: Offset 지정 필수
OVERLAPPED ov = {};
ov.Offset = filePosition & 0xFFFFFFFF;
ov.OffsetHigh = (filePosition >> 32) & 0xFFFFFFFF;
ReadFile(hFile, buffer, size, NULL, &ov);

// 소켓 I/O: Offset 무시됨 (0으로 초기화)
OVERLAPPED ov = {};
WSARecv(socket, &wsaBuf, 1, NULL, &flags, &ov, NULL);
```

### 2.1.3 hEvent 필드의 활용

IOCP 환경에서 `hEvent` 필드는 일반적으로 사용하지 않는다. 그러나 `AcceptEx` 등 특수한 경우에 하위 비트를 활용하는 트릭이 존재한다.

```cpp
// hEvent의 최하위 비트를 1로 설정하면
// I/O 완료 시 IOCP에 패킷을 큐잉하지 않는다 (Windows Server 2003+)
OVERLAPPED ov = {};
ov.hEvent = (HANDLE)((ULONG_PTR)CreateEvent(NULL, TRUE, FALSE, NULL) | 1);

// 이 OVERLAPPED로 수행한 I/O는 이벤트로만 시그널되고
// IOCP completion queue에는 들어가지 않음
```

이 기법은 특정 I/O 작업만 IOCP를 우회하여 이벤트 기반으로 처리하고 싶을 때 사용한다.

---

## 2.2 CompletionKey 활용 패턴

### 2.2.1 CompletionKey의 정체

`CompletionKey`는 `CreateIoCompletionPort`로 핸들을 IOCP에 연결할 때 지정하는 `ULONG_PTR` 값이다. 해당 핸들에서 I/O가 완료될 때마다 `GetQueuedCompletionStatus`를 통해 이 값을 돌려받는다.

```cpp
// 핸들 연결 시 CompletionKey 지정
CreateIoCompletionPort(
    (HANDLE)clientSocket,   // 연결할 핸들
    hIOCP,                  // IOCP 핸들
    (ULONG_PTR)pSession,    // ← CompletionKey
    0                       // 무시됨 (이미 생성된 IOCP)
);

// I/O 완료 시 CompletionKey 수신
ULONG_PTR completionKey;
GetQueuedCompletionStatus(hIOCP, &bytes, &completionKey, &pOv, INFINITE);
Session* pSession = (Session*)completionKey;  // 원래 포인터 복원
```

### 2.2.2 주요 활용 패턴

#### 패턴 1: 세션 포인터 직접 저장 (가장 일반적)

```cpp
// 장점: 캐스팅 한 번으로 세션 접근 가능
// 단점: 세션 생명주기 관리를 확실히 해야 함 (dangling pointer 위험)
struct Session {
    SOCKET socket;
    char recvBuffer[4096];
    // ...
};

Session* pSession = new Session();
pSession->socket = clientSocket;
CreateIoCompletionPort((HANDLE)clientSocket, hIOCP, (ULONG_PTR)pSession, 0);

// Worker에서:
Session* pSession = (Session*)completionKey;
pSession->ProcessRecv(bytesTransferred);
```

#### 패턴 2: 세션 인덱스 저장 (안전한 방식)

```cpp
// 장점: dangling pointer 문제 방지, 배열 접근으로 캐시 친화적
// 단점: 간접 참조 한 단계 추가
class SessionManager {
    Session sessions[MAX_SESSION];

    int AllocSession() {
        // 빈 슬롯 찾아서 인덱스 반환
    }
};

int sessionIdx = g_sessionMgr.AllocSession();
CreateIoCompletionPort((HANDLE)sock, hIOCP, (ULONG_PTR)sessionIdx, 0);

// Worker에서:
int idx = (int)completionKey;
Session& session = g_sessionMgr.sessions[idx];
if (session.IsValid()) {
    session.ProcessRecv(bytesTransferred);
}
```

#### 패턴 3: 핸들 타입 구분용 태그

```cpp
// 여러 종류의 핸들(리슨 소켓, 클라이언트 소켓, 파일 등)을 구분
enum HandleType : ULONG_PTR {
    HANDLE_LISTEN   = 0,
    HANDLE_CLIENT   = 1,
    HANDLE_FILE     = 2,
    HANDLE_PIPE     = 3
};

// 타입 + 인덱스를 합쳐서 저장
ULONG_PTR MakeKey(HandleType type, DWORD index) {
    return ((ULONG_PTR)type << 32) | index;  // 64비트 활용
}
HandleType GetType(ULONG_PTR key) { return (HandleType)(key >> 32); }
DWORD GetIndex(ULONG_PTR key) { return (DWORD)(key & 0xFFFFFFFF); }
```

#### 패턴 4: 핸들(소켓) 자체를 저장

```cpp
// 가장 단순한 패턴. OVERLAPPED 확장 구조체에 모든 정보가 있을 때 사용
CreateIoCompletionPort((HANDLE)sock, hIOCP, (ULONG_PTR)sock, 0);

// Worker에서: completionKey == 소켓 핸들
SOCKET clientSocket = (SOCKET)completionKey;
```

### 2.2.3 CompletionKey vs OVERLAPPED 확장 - 어디에 정보를 넣을 것인가

| 기준 | CompletionKey | OVERLAPPED 확장 |
|------|---------------|-----------------|
| 바인딩 시점 | 핸들 연결 시 1회 | 매 I/O 요청마다 |
| 변경 가능 | 불가 (재연결 필요) | I/O마다 다른 값 가능 |
| 용도 | 핸들 소유자(세션) 식별 | I/O 작업별 컨텍스트 |
| 추천 저장 정보 | 세션 포인터/인덱스 | 작업 타입, 버퍼, 상태 |

**실무 권장**: CompletionKey에는 세션 포인터, OVERLAPPED 확장에는 작업 타입과 버퍼를 저장한다.

---

## 2.3 NumberOfConcurrentThreads 최적값 결정 방법

### 2.3.1 파라미터의 의미

`CreateIoCompletionPort`의 네 번째 파라미터 `NumberOfConcurrentThreads`는 **IOCP가 동시에 실행을 허용하는 최대 스레드 수**를 지정한다. 이 값은 IOCP 생성 시에만 적용되며, 이후 핸들 연결 호출에서는 무시된다.

```cpp
// IOCP 생성 (이때만 NumberOfConcurrentThreads 적용)
HANDLE hIOCP = CreateIoCompletionPort(
    INVALID_HANDLE_VALUE,  // 새 IOCP 생성
    NULL,                  // 기존 IOCP 없음
    0,                     // CompletionKey (무시)
    numConcurrent          // ← 동시 실행 스레드 수 제한
);

// 핸들 연결 (네 번째 파라미터는 무시됨)
CreateIoCompletionPort((HANDLE)socket, hIOCP, key, 0);  // 0이어도 상관없음
```

### 2.3.2 0을 넣으면 어떻게 되는가

`NumberOfConcurrentThreads = 0`이면 시스템이 **논리 프로세서(CPU 코어) 수**와 동일한 값을 사용한다. 이것이 대부분의 경우 최적값이다.

```cpp
// 방법 1: 0 전달 (시스템이 CPU 코어 수 사용)
HANDLE hIOCP = CreateIoCompletionPort(INVALID_HANDLE_VALUE, NULL, 0, 0);

// 방법 2: 직접 지정
SYSTEM_INFO si;
GetSystemInfo(&si);
HANDLE hIOCP = CreateIoCompletionPort(
    INVALID_HANDLE_VALUE, NULL, 0,
    si.dwNumberOfProcessors  // 논리 프로세서 수
);
```

### 2.3.3 동시 실행 제한의 동작 원리

```
NumberOfConcurrentThreads = 4 (4코어 시스템)

Worker 스레드 8개 생성:
Thread 1: [실행 중]  ← IOCP가 깨움
Thread 2: [실행 중]  ← IOCP가 깨움
Thread 3: [실행 중]  ← IOCP가 깨움
Thread 4: [실행 중]  ← IOCP가 깨움
Thread 5: [IOCP 대기] ← 완료 패킷이 있어도 깨어나지 않음
Thread 6: [IOCP 대기]
Thread 7: [IOCP 대기]
Thread 8: [IOCP 대기]

Thread 2가 DB 호출로 블록되면:
Thread 2: [블록됨]   → Released Thread List로 이동
Thread 5: [실행 중]  ← IOCP가 깨움 (다시 4개 실행)
```

### 2.3.4 결정 기준

| 시나리오 | 권장값 | 이유 |
|---------|--------|------|
| 순수 I/O 처리 (메모리 복사, 파싱) | CPU 코어 수 (= 0) | 블로킹 없으므로 코어 수만큼이 최적 |
| DB 쿼리 등 블로킹 호출 포함 | CPU 코어 수 | Worker를 코어 수 * 2로 만들되, concurrent는 코어 수 |
| CPU 집약적 처리 (암호화 등) | CPU 코어 수 | CPU 경합 방지 |
| 하이브리드 | CPU 코어 수 | Worker 수를 늘려서 블로킹 보상 |

### 2.3.5 Worker 스레드 수 vs NumberOfConcurrentThreads

이 두 값은 별개의 개념이다.

```
NumberOfConcurrentThreads: IOCP가 동시에 "깨워서 실행하는" 스레드 수 제한
Worker 스레드 수: GetQueuedCompletionStatus를 호출하며 대기하는 총 스레드 수

권장 비율:
Worker 스레드 수 = CPU 코어 수 * 2
NumberOfConcurrentThreads = CPU 코어 수 (또는 0)

이유:
- 일부 Worker가 블로킹되어도 다른 Worker가 즉시 투입
- NumberOfConcurrentThreads가 과도한 context switching 방지
- IOCP는 블로킹된 스레드를 감지하고 추가 스레드를 자동으로 깨움
```

### 2.3.6 주의: 일시적 초과

`NumberOfConcurrentThreads`는 엄격한 제한이 아니다. 실행 중인 스레드가 블록된 후 IOCP가 새 스레드를 깨우고, 블록된 스레드가 바로 돌아오면 일시적으로 제한을 초과할 수 있다. 이는 의도된 동작이며, IOCP는 초과된 스레드가 다시 `GetQueuedCompletionStatus`를 호출할 때까지 기다렸다가 대기 상태로 전환한다.

---

## 2.4 IOCP와 Handle 연결 시점 및 해제 처리

### 2.4.1 핸들 연결 시점

핸들(소켓)을 IOCP에 연결하는 시점은 중요한 설계 결정이다.

```cpp
// 패턴 1: Accept 직후 즉시 연결 (가장 일반적)
void OnAcceptComplete(SOCKET clientSocket) {
    Session* pSession = AllocSession(clientSocket);
    CreateIoCompletionPort((HANDLE)clientSocket, hIOCP, (ULONG_PTR)pSession, 0);
    PostRecv(pSession);  // 바로 비동기 Recv 예약
}

// 패턴 2: 인증 완료 후 연결 (보안 강화)
// 인증 전에는 별도 스레드/이벤트에서 처리하고,
// 인증 통과 후 IOCP에 편입
void OnAuthComplete(SOCKET clientSocket) {
    Session* pSession = AllocSession(clientSocket);
    CreateIoCompletionPort((HANDLE)clientSocket, hIOCP, (ULONG_PTR)pSession, 0);
    PostRecv(pSession);
}
```

### 2.4.2 핸들 연결 해제의 부재

**IOCP에서 한 번 연결된 핸들은 해제할 수 없다.** `CreateIoCompletionPort`의 반대 동작을 수행하는 API가 존재하지 않는다.

```
연결된 핸들을 IOCP에서 분리하는 유일한 방법:
1. 핸들(소켓)을 닫는다 (closesocket / CloseHandle)
2. 핸들이 닫히면 자동으로 IOCP 연결 목록에서 제거된다
3. 진행 중인 I/O는 에러와 함께 완료 통지가 온다
```

### 2.4.3 소켓 종료 시 IOCP 동작

```cpp
// 소켓을 닫을 때의 시퀀스
closesocket(clientSocket);

// 이후 Worker 스레드에서:
// GetQueuedCompletionStatus가 FALSE를 반환
// pOverlapped != NULL
// GetLastError() == ERROR_OPERATION_ABORTED (995)
// 또는 ERROR_NETNAME_DELETED (64)
// 또는 ERROR_CONNECTION_ABORTED (1236)

// 올바른 에러 핸들링:
BOOL result = GetQueuedCompletionStatus(hIOCP, &bytes, &key, &pOv, INFINITE);
if (!result) {
    if (pOv != NULL) {
        DWORD err = GetLastError();
        switch (err) {
            case ERROR_OPERATION_ABORTED:    // 995 - CancelIo 또는 closesocket
            case ERROR_NETNAME_DELETED:      // 64  - 상대방이 연결 끊음
            case ERROR_CONNECTION_ABORTED:   // 1236 - 연결 중단
                // 세션 정리 로직
                CleanupSession((Session*)key);
                break;
        }
    }
}
```

### 2.4.4 CompletionKey 변경이 필요한 경우

CompletionKey는 변경할 수 없으므로, 세션 재활용 시 주의가 필요하다.

```cpp
// 잘못된 접근: 소켓을 새 세션에 재연결하려는 시도
// CompletionKey가 이전 세션을 가리키므로 위험!

// 올바른 접근: DisconnectEx + TF_REUSE_SOCKET 사용 시
// 소켓은 그대로지만 세션 포인터 변경이 필요하면
// CompletionKey에 인덱스 기반 패턴을 사용하거나
// 간접 참조(포인터의 포인터, 또는 배열 인덱스)를 사용

struct SessionSlot {
    Session* pCurrentSession;  // 이것을 교체하면 됨
};

SessionSlot slots[MAX_SLOTS];
CreateIoCompletionPort((HANDLE)sock, hIOCP, (ULONG_PTR)&slots[idx], 0);

// Worker에서:
SessionSlot* pSlot = (SessionSlot*)completionKey;
Session* pSession = pSlot->pCurrentSession;  // 항상 최신 세션 참조
```

---

## 2.5 IOCP 동작 모델 상세

### 2.5.1 CreateIoCompletionPort의 이중 역할

`CreateIoCompletionPort`는 하나의 함수가 두 가지 역할을 수행한다.

```cpp
// 역할 1: 새 IOCP 생성
HANDLE hIOCP = CreateIoCompletionPort(
    INVALID_HANDLE_VALUE,  // ← 새 IOCP 생성 의미
    NULL,                  // 기존 IOCP 없음
    0,
    numConcurrent          // 동시 실행 제한
);

// 역할 2: 기존 IOCP에 핸들 연결
HANDLE hResult = CreateIoCompletionPort(
    (HANDLE)socket,        // ← 연결할 핸들
    hIOCP,                 // 기존 IOCP
    completionKey,         // 이 핸들의 식별 키
    0                      // 무시됨
);
// hResult == hIOCP (성공 시)
```

### 2.5.2 GetQueuedCompletionStatus vs GetQueuedCompletionStatusEx

```cpp
// 단일 패킷 대기
BOOL GetQueuedCompletionStatus(
    HANDLE       CompletionPort,
    LPDWORD      lpNumberOfBytesTransferred,
    PULONG_PTR   lpCompletionKey,
    LPOVERLAPPED *lpOverlapped,
    DWORD        dwMilliseconds
);

// 복수 패킷 일괄 수신 (Vista+)
BOOL GetQueuedCompletionStatusEx(
    HANDLE             CompletionPort,
    LPOVERLAPPED_ENTRY lpCompletionPortEntries,  // 배열
    ULONG              ulCount,                   // 배열 크기
    PULONG             ulNumEntriesRemoved,       // 실제 수신 개수
    DWORD              dwMilliseconds,
    BOOL               fAlertable                 // APC 가능 여부
);

// OVERLAPPED_ENTRY 구조체
typedef struct _OVERLAPPED_ENTRY {
    ULONG_PTR    lpCompletionKey;
    LPOVERLAPPED lpOverlapped;
    ULONG_PTR    Internal;
    DWORD        dwNumberOfBytesTransferred;
} OVERLAPPED_ENTRY;
```

#### GetQueuedCompletionStatusEx의 장점

```
성능 비교 (높은 I/O 부하 시):
- GetQueuedCompletionStatus: 매번 커널 전환 1회
- GetQueuedCompletionStatusEx: 여러 패킷을 1회 커널 전환으로 처리

예시: 100개 패킷이 큐에 대기 중일 때
- GQCS: 100회 시스템 콜 필요
- GQCSEx(ulCount=64): 2회 시스템 콜로 처리 가능

권장 사용법:
OVERLAPPED_ENTRY entries[64];
ULONG numEntries;
if (GetQueuedCompletionStatusEx(hIOCP, entries, 64, &numEntries, INFINITE, FALSE)) {
    for (ULONG i = 0; i < numEntries; i++) {
        ProcessCompletion(entries[i]);
    }
}
```

### 2.5.3 PostQueuedCompletionStatus 활용

커널 I/O가 아닌, 사용자 정의 완료 패킷을 IOCP 큐에 수동으로 넣는 함수이다.

```cpp
// Worker 스레드 종료 시그널
void ShutdownWorkers(HANDLE hIOCP, int numWorkers) {
    for (int i = 0; i < numWorkers; i++) {
        PostQueuedCompletionStatus(hIOCP, 0, COMPLETION_KEY_SHUTDOWN, NULL);
    }
}

// Worker에서 종료 감지
if (completionKey == COMPLETION_KEY_SHUTDOWN) {
    return 0;  // 스레드 종료
}

// 타이머 이벤트 전달
void OnTimerExpired(HANDLE hIOCP, Session* pSession) {
    PostQueuedCompletionStatus(
        hIOCP,
        0,
        (ULONG_PTR)pSession,
        (OVERLAPPED*)&pSession->timerOverlapped  // 커스텀 OVERLAPPED
    );
}

// 스레드 간 작업 위임
void DelegateToWorker(HANDLE hIOCP, Task* pTask) {
    PostQueuedCompletionStatus(
        hIOCP,
        sizeof(Task),
        COMPLETION_KEY_TASK,
        (OVERLAPPED*)pTask
    );
}
```

---

## 2.6 Completion Packet의 생명주기

### 2.6.1 패킷 생성부터 소비까지

```
1. 애플리케이션이 비동기 I/O 요청 (WSARecv, WSASend 등)
   ↓
2. 커널 I/O Manager가 IRP(I/O Request Packet) 생성
   ↓
3. 드라이버가 I/O 처리 시작
   ↓
4. I/O 완료 시 드라이버가 IoCompleteRequest 호출
   ↓
5. I/O Manager가 파일 객체에 연결된 IOCP 확인
   ↓
6. Completion Packet 생성:
   - CompletionKey: 핸들 연결 시 지정한 값
   - OVERLAPPED*: 요청 시 전달한 포인터
   - BytesTransferred: 전송된 바이트 수
   ↓
7. Completion Queue(FIFO)에 패킷 추가
   ↓
8. Waiting Thread List에서 LIFO로 스레드 선택하여 깨움
   ↓
9. GetQueuedCompletionStatus가 반환하며 패킷 정보 전달
   ↓
10. 애플리케이션이 완료 처리 수행
```

### 2.6.2 FILE_SKIP_COMPLETION_PORT_ON_SUCCESS 최적화

Windows Vista 이상에서 `SetFileCompletionNotificationModes`를 사용하면, I/O가 동기적으로 즉시 완료되는 경우 Completion Packet을 생성하지 않도록 설정할 수 있다.

```cpp
// 설정
SetFileCompletionNotificationModes(
    (HANDLE)socket,
    FILE_SKIP_COMPLETION_PORT_ON_SUCCESS
);

// 효과: WSARecv/WSASend가 즉시 완료(성공 반환)되면
// IOCP 큐에 패킷이 들어가지 않음 → 오버헤드 감소

// 사용 시 주의:
// WSARecv가 성공(0)을 반환하면 직접 결과를 처리해야 함
DWORD flags = 0;
int ret = WSARecv(sock, &wsaBuf, 1, &bytesRecvd, &flags, &ov, NULL);
if (ret == 0) {
    // 즉시 완료 → IOCP에 패킷 안 감 → 여기서 직접 처리
    ProcessRecv(bytesRecvd);
    // 다음 Recv 예약도 여기서
    PostRecv(pSession);
} else if (WSAGetLastError() == WSA_IO_PENDING) {
    // 비동기 진행 중 → IOCP에서 완료 통지 받음
}
```

이 최적화는 로컬호스트 통신이나 버퍼에 데이터가 이미 있는 경우 등 동기 완료가 빈번할 때 성능을 크게 향상시킨다.

---

## 2.7 퀴즈용 핵심 Q&A

### 정의형 문제

**Q1**: OVERLAPPED 구조체의 Internal 필드에는 실제로 무엇이 저장되는가?
> **A**: 커널의 `IO_STATUS_BLOCK.Status`와 대응하며, I/O의 NTSTATUS 상태 코드가 저장된다. 비동기 I/O가 진행 중이면 `STATUS_PENDING(0x103)`, 완료 시 최종 상태 코드가 들어간다.

**Q2**: CompletionKey는 언제 설정되며, 변경이 가능한가?
> **A**: `CreateIoCompletionPort`로 핸들을 IOCP에 연결할 때 설정되며, 한 번 설정된 CompletionKey는 변경할 수 없다. 변경하려면 핸들을 닫고 다시 열어야 한다.

**Q3**: `GetQueuedCompletionStatus`와 `GetQueuedCompletionStatusEx`의 차이는?
> **A**: 전자는 1개의 완료 패킷만 수신하고, 후자는 배열을 통해 여러 개를 한 번에 수신할 수 있다. 높은 I/O 부하에서 후자가 시스템 콜 횟수를 줄여 성능이 우수하다.

### 수치형 문제

**Q4**: `NumberOfConcurrentThreads = 0`으로 설정하면 어떻게 동작하는가?
> **A**: 시스템의 논리 프로세서(CPU 코어) 수와 동일한 값으로 자동 설정된다. 4코어 8스레드 CPU라면 8이 된다.

**Q5**: Worker 스레드 수의 일반적 권장값은?
> **A**: CPU 코어 수 × 2. `NumberOfConcurrentThreads`는 코어 수로 설정하고, Worker는 그 2배를 두어 일부가 블로킹되어도 나머지가 처리할 수 있게 한다.

### 적용형 문제

**Q6**: 세션 객체의 메모리를 해제한 후에도 해당 세션의 소켓에서 I/O 완료가 올 수 있다면, CompletionKey에 세션 포인터를 저장하는 방식의 위험성은?
> **A**: Dangling pointer 참조가 발생한다. I/O가 완료되면 GQCS가 이미 해제된 세션의 포인터를 반환하므로 접근 위반(Access Violation)이 일어난다. 이를 방지하려면 (1) 세션 포인터 대신 인덱스 사용, (2) 모든 진행 중인 I/O가 완료된 후에만 세션 해제, (3) Reference Counting 적용 등의 방법이 필요하다.

**Q7**: `FILE_SKIP_COMPLETION_PORT_ON_SUCCESS`를 설정하면 코드 구조가 어떻게 달라져야 하는가?
> **A**: WSARecv/WSASend 호출 후 반환값을 확인하여, 즉시 성공(0 반환)인 경우 IOCP 통지 없이 직접 결과를 처리해야 한다. 기존에 모든 처리를 Worker 루프에서 하던 것과 달리, I/O 요청 함수 호출 직후에도 완료 처리 경로가 필요하다.

---

## 참고 자료

- Microsoft Docs: [I/O Completion Ports](https://learn.microsoft.com/en-us/windows/win32/fileio/i-o-completion-ports)
- Microsoft Docs: [CreateIoCompletionPort](https://learn.microsoft.com/en-us/windows/win32/fileio/createiocompletionport)
- Microsoft Docs: [GetQueuedCompletionStatusEx](https://learn.microsoft.com/en-us/windows/win32/api/ioapiset/nf-ioapiset-getqueuedcompletionstatusex)
- Microsoft Docs: [SetFileCompletionNotificationModes](https://learn.microsoft.com/en-us/windows/win32/api/winbase/nf-winbase-setfilecompletionnotificationmodes)
- Windows Internals, 7th Edition - Chapter 6: I/O System
- The Old New Thing (Raymond Chen) - IOCP 관련 포스트
