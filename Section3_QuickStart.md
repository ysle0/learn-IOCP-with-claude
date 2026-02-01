# Section 3: IOCP Quick Start 상세 가이드

> 작성 완료일: 2026-02-01

---

## 목차
- [3.1 WSA_FLAG_OVERLAPPED 플래그 필수 여부](#31-wsa_flag_overlapped-플래그-필수-여부)
- [3.2 AcceptEx 함수 포인터 획득 방법](#32-acceptex-함수-포인터-획득-방법)
- [3.3 초기 Accept 예약 개수 결정 기준](#33-초기-accept-예약-개수-결정-기준)
- [3.4 버퍼 크기 최적화](#34-버퍼-크기-최적화)
- [3.5 완전한 IOCP Echo 서버 구현](#35-완전한-iocp-echo-서버-구현)
- [3.6 흔한 실수와 디버깅](#36-흔한-실수와-디버깅)
- [3.7 퀴즈용 핵심 Q&A](#37-퀴즈용-핵심-qa)

---

## 3.1 WSA_FLAG_OVERLAPPED 플래그 필수 여부

### 3.1.1 결론: 필수는 아니지만 명시적 사용 권장

```cpp
// 방법 1: WSASocket + WSA_FLAG_OVERLAPPED (명시적)
SOCKET sock = WSASocket(AF_INET, SOCK_STREAM, IPPROTO_TCP,
                        NULL, 0, WSA_FLAG_OVERLAPPED);

// 방법 2: socket() 함수 (암시적 Overlapped 지원)
SOCKET sock = socket(AF_INET, SOCK_STREAM, IPPROTO_TCP);
```

### 3.1.2 왜 둘 다 동작하는가

Windows에서 `socket()` 함수로 생성된 소켓은 **기본적으로 Overlapped I/O를 지원**한다. 이는 Windows의 역사적 결정으로, Win32 소켓은 처음부터 비동기 I/O를 염두에 두고 설계되었기 때문이다.

```
socket() 내부 동작:
→ WSASocket(AF_INET, SOCK_STREAM, 0, NULL, 0, WSA_FLAG_OVERLAPPED)
→ 즉, socket()은 WSA_FLAG_OVERLAPPED가 자동으로 포함됨
```

### 3.1.3 WSA_FLAG_OVERLAPPED를 명시해야 하는 진짜 이유

```cpp
// WSASocket에서 dwFlags를 0으로 넣으면 Overlapped 미지원 소켓이 생성됨!
SOCKET nonOvSock = WSASocket(AF_INET, SOCK_STREAM, 0, NULL, 0, 0);
// 이 소켓으로 WSARecv + OVERLAPPED 사용 시 에러 발생

// 따라서 WSASocket을 사용할 때는 반드시 명시
SOCKET ovSock = WSASocket(AF_INET, SOCK_STREAM, 0, NULL, 0, WSA_FLAG_OVERLAPPED);
```

| 소켓 생성 방법 | Overlapped 지원 | IOCP 사용 가능 |
|---------------|----------------|---------------|
| `socket()` | O (자동) | O |
| `WSASocket(..., WSA_FLAG_OVERLAPPED)` | O (명시) | O |
| `WSASocket(..., 0)` | X | X |

### 3.1.4 권장 사항

```cpp
// IOCP 프로젝트에서는 항상 WSASocket + 명시적 플래그 사용
// 이유: 의도를 명확히 하고, 다른 플래그와 함께 사용할 수 있음
SOCKET sock = WSASocket(AF_INET, SOCK_STREAM, IPPROTO_TCP,
                        NULL, 0, WSA_FLAG_OVERLAPPED);
if (sock == INVALID_SOCKET) {
    printf("WSASocket failed: %d\n", WSAGetLastError());
    return;
}
```

---

## 3.2 AcceptEx 함수 포인터 획득 방법

### 3.2.1 왜 함수 포인터를 별도로 획득해야 하는가

`AcceptEx`, `ConnectEx`, `DisconnectEx`, `TransmitFile`, `GetAcceptExSockaddrs` 등은 **Winsock 확장 함수**로, DLL에 직접 노출되지 않는다. Winsock 서비스 프로바이더(SPI)의 확장 기능이므로 `WSAIoctl`을 통해 런타임에 함수 포인터를 획득해야 한다.

```
직접 호출이 안 되는 이유:
1. Winsock은 프로바이더 아키텍처 → 확장 함수는 프로바이더마다 다를 수 있음
2. 프로바이더 DLL(mswsock.dll)을 직접 링크하면 Layered Service Provider(LSP)를
   우회하게 되어 방화벽/VPN/안티바이러스 등의 네트워크 필터가 작동하지 않을 수 있음
3. WSAIoctl을 통해 현재 프로바이더에 맞는 올바른 함수 포인터를 받아야 함
```

### 3.2.2 함수 포인터 획득 코드

```cpp
#include <MSWSock.h>  // GUID 정의, LPFN_ACCEPTEX 등

// AcceptEx 함수 포인터 획득
LPFN_ACCEPTEX lpfnAcceptEx = NULL;
GUID guidAcceptEx = WSAID_ACCEPTEX;
DWORD dwBytes;

int result = WSAIoctl(
    listenSocket,                    // 아무 유효한 소켓이면 됨
    SIO_GET_EXTENSION_FUNCTION_POINTER,
    &guidAcceptEx,                   // 요청할 함수의 GUID
    sizeof(guidAcceptEx),
    &lpfnAcceptEx,                   // 받을 함수 포인터
    sizeof(lpfnAcceptEx),
    &dwBytes,
    NULL,
    NULL
);

if (result == SOCKET_ERROR) {
    printf("Failed to get AcceptEx: %d\n", WSAGetLastError());
    return;
}
```

### 3.2.3 모든 확장 함수 GUID 및 타입

```cpp
// 함수별 GUID와 타입 정리
struct {
    const char* name;
    GUID guid;
    // 함수 포인터 타입
} extensions[] = {
    // AcceptEx - 비동기 Accept
    { "AcceptEx",              WSAID_ACCEPTEX },
    // LPFN_ACCEPTEX

    // GetAcceptExSockaddrs - AcceptEx 결과에서 주소 추출
    { "GetAcceptExSockaddrs",  WSAID_GETACCEPTEXSOCKADDRS },
    // LPFN_GETACCEPTEXSOCKADDRS

    // ConnectEx - 비동기 Connect
    { "ConnectEx",             WSAID_CONNECTEX },
    // LPFN_CONNECTEX

    // DisconnectEx - 비동기 Disconnect (소켓 재사용)
    { "DisconnectEx",          WSAID_DISCONNECTEX },
    // LPFN_DISCONNECTEX

    // TransmitFile - 파일 → 소켓 직접 전송
    { "TransmitFile",          WSAID_TRANSMITFILE },
    // LPFN_TRANSMITFILE

    // TransmitPackets - 메모리/파일 데이터 혼합 전송
    { "TransmitPackets",       WSAID_TRANSMITPACKETS },
    // LPFN_TRANSMITPACKETS

    // WSARecvMsg - ancillary data 포함 수신
    { "WSARecvMsg",            WSAID_WSARECVMSG },
    // LPFN_WSARECVMSG

    // WSASendMsg - ancillary data 포함 송신 (Vista+)
    { "WSASendMsg",            WSAID_WSASENDMSG },
    // LPFN_WSASENDMSG
};
```

### 3.2.4 효율적인 일괄 획득 패턴

```cpp
class WinsockExtensions {
public:
    LPFN_ACCEPTEX              AcceptEx = nullptr;
    LPFN_GETACCEPTEXSOCKADDRS  GetAcceptExSockaddrs = nullptr;
    LPFN_CONNECTEX             ConnectEx = nullptr;
    LPFN_DISCONNECTEX          DisconnectEx = nullptr;
    LPFN_TRANSMITFILE          TransmitFile = nullptr;

    bool Initialize(SOCKET anySocket) {
        bool ok = true;
        ok &= LoadExtension(anySocket, WSAID_ACCEPTEX, &AcceptEx);
        ok &= LoadExtension(anySocket, WSAID_GETACCEPTEXSOCKADDRS, &GetAcceptExSockaddrs);
        ok &= LoadExtension(anySocket, WSAID_CONNECTEX, &ConnectEx);
        ok &= LoadExtension(anySocket, WSAID_DISCONNECTEX, &DisconnectEx);
        ok &= LoadExtension(anySocket, WSAID_TRANSMITFILE, &TransmitFile);
        return ok;
    }

private:
    template<typename T>
    bool LoadExtension(SOCKET sock, GUID guid, T* ppFunc) {
        DWORD dwBytes;
        int ret = WSAIoctl(sock, SIO_GET_EXTENSION_FUNCTION_POINTER,
                           &guid, sizeof(guid),
                           ppFunc, sizeof(*ppFunc),
                           &dwBytes, NULL, NULL);
        return (ret != SOCKET_ERROR);
    }
};

// 사용
WinsockExtensions g_wsaExt;
g_wsaExt.Initialize(listenSocket);
g_wsaExt.AcceptEx(listenSocket, acceptSocket, ...);
```

### 3.2.5 mswsock.lib 직접 링크와의 차이

```cpp
// 방법 A: WSAIoctl (권장)
// → 현재 프로바이더 체인을 통해 올바른 함수 포인터 획득
// → LSP/WFP 필터가 정상 작동

// 방법 B: mswsock.lib 링크 후 직접 호출 (비권장)
#pragma comment(lib, "mswsock.lib")
AcceptEx(listenSocket, acceptSocket, ...);
// → mswsock.dll의 함수를 직접 호출
// → LSP가 설치된 환경에서 문제 발생 가능
// → 하지만 현대 Windows에서는 LSP가 deprecated되어 실질적 차이 줄어듦

// 결론: WSAIoctl 방식이 정석이며, 이식성과 호환성이 보장됨
```

---

## 3.3 초기 Accept 예약 개수 결정 기준

### 3.3.1 Pre-posted Accept의 원리

`AcceptEx`는 비동기 Accept로, 미리 여러 개를 "예약"해놓을 수 있다. 클라이언트가 연결하면 예약된 AcceptEx 중 하나가 완료된다.

```
Pre-posted Accepts 동작:
1. 서버 시작 시 AcceptEx N개 예약
2. 클라이언트 연결 → 예약 1개 소비 → 완료 패킷 도착
3. Worker에서 처리 후 새 AcceptEx 1개 보충
4. 항상 일정 수의 Accept가 대기 상태 유지
```

### 3.3.2 예약 개수 결정 기준

| 서버 유형 | 권장 초기 예약 수 | 이유 |
|---------|------------------|------|
| 소규모 (< 100 동시접속) | 5~10개 | 연결 빈도 낮음 |
| 중규모 (100~1,000) | 10~50개 | 일반적인 게임/앱 서버 |
| 대규모 (1,000~10,000) | 50~200개 | 폭발적 연결 대비 |
| 로그인 서버/이벤트 | 200~500개 | 동시 대량 연결 폭주 대비 |

### 3.3.3 동적 조절 알고리즘

```cpp
class AcceptManager {
    static const int MIN_PENDING_ACCEPTS = 10;
    static const int MAX_PENDING_ACCEPTS = 200;
    static const int ACCEPT_BATCH_SIZE = 5;

    LONG m_pendingAcceptCount = 0;  // 현재 대기 중인 AcceptEx 수
    LONG m_acceptsPerSecond = 0;    // 초당 Accept 완료 수 (모니터링)

    // Accept 완료 시 호출
    void OnAcceptComplete() {
        InterlockedDecrement(&m_pendingAcceptCount);
        InterlockedIncrement(&m_acceptsPerSecond);

        // 대기 수가 최소 미만이면 추가 예약
        if (m_pendingAcceptCount < MIN_PENDING_ACCEPTS) {
            int toPost = min(ACCEPT_BATCH_SIZE,
                           MAX_PENDING_ACCEPTS - m_pendingAcceptCount);
            for (int i = 0; i < toPost; i++) {
                PostAcceptEx();
                InterlockedIncrement(&m_pendingAcceptCount);
            }
        }
    }

    // 주기적 모니터링 (1초마다)
    void MonitorAndAdjust() {
        int rate = InterlockedExchange(&m_acceptsPerSecond, 0);

        // 연결 속도가 높으면 예약 수 증가
        int targetPending = max(MIN_PENDING_ACCEPTS, rate * 2);
        targetPending = min(targetPending, MAX_PENDING_ACCEPTS);

        while (m_pendingAcceptCount < targetPending) {
            PostAcceptEx();
            InterlockedIncrement(&m_pendingAcceptCount);
        }
    }
};
```

### 3.3.4 Accept 예약이 부족하면 어떻게 되는가

```
예약된 AcceptEx가 모두 소진된 상태:
1. 새 클라이언트 연결 요청 도착
2. 커널의 listen backlog 큐에 대기
3. backlog도 가득 차면 → 연결 거부 (RST 또는 타임아웃)
4. 클라이언트는 WSAECONNREFUSED 또는 타임아웃 에러

→ 따라서 AcceptEx 예약 소진 방지가 중요
→ listen backlog (두 번째 방어선) 설정도 필수
```

```cpp
// listen backlog 설정
listen(listenSocket, SOMAXCONN);  // 시스템 최대값 사용

// Windows에서 SOMAXCONN = 0x7FFFFFFF (2^31 - 1)
// 실제로는 시스템이 적절한 값으로 제한함
```

---

## 3.4 버퍼 크기 최적화

### 3.4.1 MTU와 버퍼 크기의 관계

```
이더넷 MTU = 1500 bytes
IP 헤더    = 20 bytes
TCP 헤더   = 20 bytes (옵션 없을 때)
────────────────────────
TCP MSS    = 1460 bytes (Maximum Segment Size)

따라서:
- 최소 유의미한 버퍼 크기: 1460 bytes (1 MSS)
- 작은 패킷 서버: 1~4 KB
- 일반 서버: 4~8 KB
- 파일 전송 서버: 32~64 KB
```

### 3.4.2 페이지 단위 정렬

Windows 메모리 관리자는 4KB(x86/x64) 페이지 단위로 동작한다. 비동기 I/O 버퍼는 커널이 페이지를 잠그므로(page locking), 페이지 경계를 고려하면 효율적이다.

```cpp
// 페이지 정렬된 버퍼 할당
const int PAGE_SIZE = 4096;

// 방법 1: VirtualAlloc (페이지 정렬 보장)
void* buffer = VirtualAlloc(NULL, PAGE_SIZE * 2,
                            MEM_COMMIT | MEM_RESERVE, PAGE_READWRITE);

// 방법 2: _aligned_malloc
void* buffer = _aligned_malloc(8192, PAGE_SIZE);

// 방법 3: 일반 malloc도 실무에서는 충분 (작은 버퍼에서는 정렬 효과 미미)
char* buffer = (char*)malloc(8192);
```

### 3.4.3 버퍼 크기별 장단점

| 버퍼 크기 | 장점 | 단점 | 적합한 용도 |
|----------|------|------|------------|
| 512B~1KB | 메모리 절약 | Recv 호출 빈번, syscall 오버헤드 | 채팅 서버, 소규모 패킷 |
| 4KB~8KB | 균형 잡힌 크기 | - | 게임 서버 (일반) |
| 16KB~32KB | 대용량 데이터 효율적 | 메모리 사용량 증가 | 파일 전송, 스트리밍 |
| 64KB+ | throughput 극대화 | 세션당 메모리 높음 | 전용 파일/미디어 서버 |

### 3.4.4 Non-paged Pool과 버퍼 잠금

```
비동기 I/O 시 커널 동작:
1. 유저 모드 버퍼의 페이지를 물리 메모리에 잠금 (page lock)
2. MDL(Memory Descriptor List) 생성
3. DMA 전송 가능하도록 물리 주소 매핑
4. I/O 완료 후 페이지 잠금 해제

주의:
- 동시에 잠긴 페이지 수에 시스템 제한 있음
- 너무 큰 버퍼 × 많은 동시 I/O = 페이지 잠금 실패 (WSAENOBUFS)
- 해결: 버퍼 크기를 줄이거나, Zero-byte Recv 기법 사용
```

### 3.4.5 실무 권장 전략

```cpp
// 게임 서버 권장 설정
struct SessionConfig {
    static const int RECV_BUFFER_SIZE = 4096;   // 4KB (게임 패킷은 보통 작음)
    static const int SEND_BUFFER_SIZE = 65536;  // 64KB (브로드캐스트 대비)

    // Ring Buffer 사용 시 2의 거듭제곱이 유리 (비트 마스크 연산)
    static const int RING_BUFFER_SIZE = 8192;   // 8KB, 마스크 = 0x1FFF
};

// 동시접속 5,000명일 때 메모리 계산:
// Recv: 5,000 × 4KB = 20MB
// Send: 5,000 × 64KB = 320MB
// 총 버퍼 메모리: ~340MB (허용 가능한 수준)
```

---

## 3.5 완전한 IOCP Echo 서버 구현

### 3.5.1 전체 코드 (주석 포함)

```cpp
#define WIN32_LEAN_AND_MEAN
#include <windows.h>
#include <winsock2.h>
#include <ws2tcpip.h>
#include <mswsock.h>
#include <stdio.h>

#pragma comment(lib, "ws2_32.lib")

// ============================================================
// 상수 및 열거형
// ============================================================
const int PORT = 9000;
const int BUFFER_SIZE = 4096;
const int MAX_WORKER_THREADS = 8;
const int INITIAL_ACCEPT_COUNT = 10;

enum OperationType {
    OP_ACCEPT,
    OP_RECV,
    OP_SEND,
    OP_DISCONNECT
};

// ============================================================
// 확장 OVERLAPPED 구조체
// ============================================================
struct OverlappedEx {
    OVERLAPPED  overlapped;     // 반드시 첫 번째 멤버
    SOCKET      socket;         // 이 I/O의 대상 소켓
    WSABUF      wsaBuf;
    char        buffer[BUFFER_SIZE];
    OperationType opType;
    SOCKET      acceptSocket;   // OP_ACCEPT 전용
};

// ============================================================
// 전역 변수
// ============================================================
HANDLE g_hIOCP = NULL;
SOCKET g_listenSocket = INVALID_SOCKET;
LPFN_ACCEPTEX g_lpfnAcceptEx = NULL;
LPFN_GETACCEPTEXSOCKADDRS g_lpfnGetAcceptExSockaddrs = NULL;

// ============================================================
// 함수 선언
// ============================================================
bool InitializeWinsock();
bool CreateListenSocket();
bool LoadExtensionFunctions();
bool PostAcceptEx(OverlappedEx* pOvEx);
bool PostRecv(OverlappedEx* pOvEx);
bool PostSend(OverlappedEx* pOvEx, DWORD bytesToSend);
void HandleAcceptCompletion(OverlappedEx* pOvEx, DWORD bytesTransferred);
void HandleRecvCompletion(OverlappedEx* pOvEx, DWORD bytesTransferred);
void HandleSendCompletion(OverlappedEx* pOvEx, DWORD bytesTransferred);
void CloseSession(OverlappedEx* pOvEx);
DWORD WINAPI WorkerThread(LPVOID lpParam);

// ============================================================
// 메인 함수
// ============================================================
int main() {
    // 1. Winsock 초기화
    if (!InitializeWinsock()) return 1;

    // 2. IOCP 생성
    g_hIOCP = CreateIoCompletionPort(INVALID_HANDLE_VALUE, NULL, 0, 0);
    if (!g_hIOCP) {
        printf("CreateIoCompletionPort failed: %d\n", GetLastError());
        return 1;
    }

    // 3. Listen 소켓 생성 및 IOCP 연결
    if (!CreateListenSocket()) return 1;

    // 4. 확장 함수 포인터 획득
    if (!LoadExtensionFunctions()) return 1;

    // 5. Worker 스레드 생성
    SYSTEM_INFO si;
    GetSystemInfo(&si);
    int workerCount = min((DWORD)MAX_WORKER_THREADS, si.dwNumberOfProcessors * 2);
    HANDLE hThreads[MAX_WORKER_THREADS];

    for (int i = 0; i < workerCount; i++) {
        hThreads[i] = CreateThread(NULL, 0, WorkerThread, NULL, 0, NULL);
    }

    printf("IOCP Echo Server started on port %d\n", PORT);
    printf("Worker threads: %d\n", workerCount);

    // 6. AcceptEx 예약
    for (int i = 0; i < INITIAL_ACCEPT_COUNT; i++) {
        OverlappedEx* pOvEx = new OverlappedEx();
        if (!PostAcceptEx(pOvEx)) {
            delete pOvEx;
        }
    }

    // 7. Worker 스레드 종료 대기 (Ctrl+C로 종료)
    WaitForMultipleObjects(workerCount, hThreads, TRUE, INFINITE);

    // 8. 정리
    closesocket(g_listenSocket);
    CloseHandle(g_hIOCP);
    WSACleanup();
    return 0;
}

// ============================================================
// 초기화 함수들
// ============================================================
bool InitializeWinsock() {
    WSADATA wsaData;
    int result = WSAStartup(MAKEWORD(2, 2), &wsaData);
    if (result != 0) {
        printf("WSAStartup failed: %d\n", result);
        return false;
    }
    return true;
}

bool CreateListenSocket() {
    g_listenSocket = WSASocket(AF_INET, SOCK_STREAM, IPPROTO_TCP,
                               NULL, 0, WSA_FLAG_OVERLAPPED);
    if (g_listenSocket == INVALID_SOCKET) {
        printf("WSASocket failed: %d\n", WSAGetLastError());
        return false;
    }

    // SO_REUSEADDR 설정 (개발 편의)
    BOOL reuseAddr = TRUE;
    setsockopt(g_listenSocket, SOL_SOCKET, SO_REUSEADDR,
               (char*)&reuseAddr, sizeof(reuseAddr));

    // 바인드
    SOCKADDR_IN addr = {};
    addr.sin_family = AF_INET;
    addr.sin_addr.s_addr = htonl(INADDR_ANY);
    addr.sin_port = htons(PORT);

    if (bind(g_listenSocket, (SOCKADDR*)&addr, sizeof(addr)) == SOCKET_ERROR) {
        printf("bind failed: %d\n", WSAGetLastError());
        return false;
    }

    // 리슨
    if (listen(g_listenSocket, SOMAXCONN) == SOCKET_ERROR) {
        printf("listen failed: %d\n", WSAGetLastError());
        return false;
    }

    // IOCP에 연결 (CompletionKey = 0: Listen 소켓 식별)
    CreateIoCompletionPort((HANDLE)g_listenSocket, g_hIOCP, 0, 0);

    return true;
}

bool LoadExtensionFunctions() {
    DWORD dwBytes;

    // AcceptEx
    GUID guidAcceptEx = WSAID_ACCEPTEX;
    if (WSAIoctl(g_listenSocket, SIO_GET_EXTENSION_FUNCTION_POINTER,
                 &guidAcceptEx, sizeof(guidAcceptEx),
                 &g_lpfnAcceptEx, sizeof(g_lpfnAcceptEx),
                 &dwBytes, NULL, NULL) == SOCKET_ERROR) {
        printf("Failed to load AcceptEx: %d\n", WSAGetLastError());
        return false;
    }

    // GetAcceptExSockaddrs
    GUID guidGetAddr = WSAID_GETACCEPTEXSOCKADDRS;
    if (WSAIoctl(g_listenSocket, SIO_GET_EXTENSION_FUNCTION_POINTER,
                 &guidGetAddr, sizeof(guidGetAddr),
                 &g_lpfnGetAcceptExSockaddrs, sizeof(g_lpfnGetAcceptExSockaddrs),
                 &dwBytes, NULL, NULL) == SOCKET_ERROR) {
        printf("Failed to load GetAcceptExSockaddrs: %d\n", WSAGetLastError());
        return false;
    }

    return true;
}

// ============================================================
// I/O 예약 함수들
// ============================================================
bool PostAcceptEx(OverlappedEx* pOvEx) {
    ZeroMemory(&pOvEx->overlapped, sizeof(OVERLAPPED));
    pOvEx->opType = OP_ACCEPT;

    // Accept용 소켓 미리 생성
    pOvEx->acceptSocket = WSASocket(AF_INET, SOCK_STREAM, IPPROTO_TCP,
                                     NULL, 0, WSA_FLAG_OVERLAPPED);
    if (pOvEx->acceptSocket == INVALID_SOCKET) {
        printf("WSASocket for accept failed: %d\n", WSAGetLastError());
        return false;
    }

    DWORD dwBytes;
    BOOL result = g_lpfnAcceptEx(
        g_listenSocket,
        pOvEx->acceptSocket,
        pOvEx->buffer,              // 출력 버퍼 (주소 + 첫 데이터)
        0,                          // 첫 데이터 수신 안 함 (0 = 주소만)
        sizeof(SOCKADDR_IN) + 16,   // 로컬 주소 크기
        sizeof(SOCKADDR_IN) + 16,   // 원격 주소 크기
        &dwBytes,
        &pOvEx->overlapped
    );

    if (!result && WSAGetLastError() != ERROR_IO_PENDING) {
        printf("AcceptEx failed: %d\n", WSAGetLastError());
        closesocket(pOvEx->acceptSocket);
        return false;
    }

    return true;
}

bool PostRecv(OverlappedEx* pOvEx) {
    ZeroMemory(&pOvEx->overlapped, sizeof(OVERLAPPED));
    pOvEx->opType = OP_RECV;
    pOvEx->wsaBuf.buf = pOvEx->buffer;
    pOvEx->wsaBuf.len = BUFFER_SIZE;

    DWORD flags = 0;
    int result = WSARecv(pOvEx->socket, &pOvEx->wsaBuf, 1,
                         NULL, &flags, &pOvEx->overlapped, NULL);

    if (result == SOCKET_ERROR && WSAGetLastError() != WSA_IO_PENDING) {
        printf("WSARecv failed: %d\n", WSAGetLastError());
        return false;
    }

    return true;
}

bool PostSend(OverlappedEx* pOvEx, DWORD bytesToSend) {
    ZeroMemory(&pOvEx->overlapped, sizeof(OVERLAPPED));
    pOvEx->opType = OP_SEND;
    pOvEx->wsaBuf.buf = pOvEx->buffer;
    pOvEx->wsaBuf.len = bytesToSend;

    int result = WSASend(pOvEx->socket, &pOvEx->wsaBuf, 1,
                         NULL, 0, &pOvEx->overlapped, NULL);

    if (result == SOCKET_ERROR && WSAGetLastError() != WSA_IO_PENDING) {
        printf("WSASend failed: %d\n", WSAGetLastError());
        return false;
    }

    return true;
}

// ============================================================
// 완료 처리 함수들
// ============================================================
void HandleAcceptCompletion(OverlappedEx* pOvEx, DWORD bytesTransferred) {
    // SO_UPDATE_ACCEPT_CONTEXT 설정 (필수!)
    setsockopt(pOvEx->acceptSocket, SOL_SOCKET, SO_UPDATE_ACCEPT_CONTEXT,
               (char*)&g_listenSocket, sizeof(g_listenSocket));

    // 원격 주소 추출
    SOCKADDR_IN *pLocalAddr = NULL, *pRemoteAddr = NULL;
    int localLen, remoteLen;
    g_lpfnGetAcceptExSockaddrs(
        pOvEx->buffer,
        0,
        sizeof(SOCKADDR_IN) + 16,
        sizeof(SOCKADDR_IN) + 16,
        (SOCKADDR**)&pLocalAddr, &localLen,
        (SOCKADDR**)&pRemoteAddr, &remoteLen
    );

    char addrStr[INET_ADDRSTRLEN];
    inet_ntop(AF_INET, &pRemoteAddr->sin_addr, addrStr, sizeof(addrStr));
    printf("Client connected: %s:%d\n", addrStr, ntohs(pRemoteAddr->sin_port));

    // 새 소켓을 IOCP에 연결
    pOvEx->socket = pOvEx->acceptSocket;
    CreateIoCompletionPort((HANDLE)pOvEx->socket, g_hIOCP, (ULONG_PTR)pOvEx, 0);

    // Recv 예약
    PostRecv(pOvEx);

    // 새 AcceptEx 예약 (소비된 것 보충)
    OverlappedEx* pNewAccept = new OverlappedEx();
    if (!PostAcceptEx(pNewAccept)) {
        delete pNewAccept;
    }
}

void HandleRecvCompletion(OverlappedEx* pOvEx, DWORD bytesTransferred) {
    // Echo: 받은 데이터를 그대로 전송
    printf("Recv %d bytes, echoing back\n", bytesTransferred);
    PostSend(pOvEx, bytesTransferred);
}

void HandleSendCompletion(OverlappedEx* pOvEx, DWORD bytesTransferred) {
    // Send 완료 → 다시 Recv 예약
    PostRecv(pOvEx);
}

void CloseSession(OverlappedEx* pOvEx) {
    printf("Client disconnected (socket=%lld)\n", (long long)pOvEx->socket);
    closesocket(pOvEx->socket);
    pOvEx->socket = INVALID_SOCKET;
    delete pOvEx;
}

// ============================================================
// Worker 스레드
// ============================================================
DWORD WINAPI WorkerThread(LPVOID lpParam) {
    DWORD bytesTransferred;
    ULONG_PTR completionKey;
    OVERLAPPED* pOverlapped;

    while (true) {
        BOOL result = GetQueuedCompletionStatus(
            g_hIOCP,
            &bytesTransferred,
            &completionKey,
            &pOverlapped,
            INFINITE
        );

        // 종료 시그널 확인
        if (completionKey == (ULONG_PTR)-1) {
            break;
        }

        OverlappedEx* pOvEx = (OverlappedEx*)pOverlapped;

        // 에러 또는 연결 종료
        if (!result || (bytesTransferred == 0 && pOvEx->opType != OP_ACCEPT)) {
            if (pOvEx) {
                CloseSession(pOvEx);
            }
            continue;
        }

        // 작업 타입별 처리
        switch (pOvEx->opType) {
            case OP_ACCEPT:
                HandleAcceptCompletion(pOvEx, bytesTransferred);
                break;
            case OP_RECV:
                HandleRecvCompletion(pOvEx, bytesTransferred);
                break;
            case OP_SEND:
                HandleSendCompletion(pOvEx, bytesTransferred);
                break;
        }
    }

    return 0;
}
```

### 3.5.2 코드 흐름 다이어그램

```
서버 시작
    │
    ├─ WSAStartup
    ├─ CreateIoCompletionPort (IOCP 생성)
    ├─ WSASocket + bind + listen (리슨 소켓)
    ├─ IOCP에 리슨 소켓 연결
    ├─ WSAIoctl로 AcceptEx/GetAcceptExSockaddrs 획득
    ├─ Worker 스레드 N개 생성
    └─ AcceptEx 10개 예약
         │
         ▼
    ┌─────────────────────────────────────┐
    │        Worker Thread Loop           │
    │  GetQueuedCompletionStatus(INFINITE)│
    │            │                         │
    │     ┌──────┴──────┐                  │
    │     ▼             ▼                  │
    │  에러/종료     정상 완료              │
    │  → CloseSession  │                   │
    │              ┌────┴────┐             │
    │              ▼         ▼             │
    │          OP_ACCEPT  OP_RECV          │
    │          │          │                │
    │          ├─ SO_UPDATE  ├─ Echo 처리   │
    │          ├─ IOCP 연결  └─ PostSend    │
    │          ├─ PostRecv                  │
    │          └─ 새 AcceptEx 보충          │
    │                                      │
    │         OP_SEND                      │
    │          └─ PostRecv (다시 수신 대기)  │
    └──────────────────────────────────────┘
```

---

## 3.6 흔한 실수와 디버깅

### 3.6.1 Top 10 초보 실수

| # | 실수 | 증상 | 해결 |
|---|------|------|------|
| 1 | OVERLAPPED 초기화 누락 | 예측 불가 에러 | 매 I/O 요청 전 `ZeroMemory` |
| 2 | SO_UPDATE_ACCEPT_CONTEXT 누락 | `getpeername`, `setsockopt` 실패 | AcceptEx 완료 후 반드시 호출 |
| 3 | I/O 진행 중 OVERLAPPED 해제 | 크래시 (커널이 해제된 메모리에 쓰기) | Reference counting으로 생명주기 관리 |
| 4 | WSA_IO_PENDING 에러 무시 | 정상 비동기 요청을 에러로 처리 | `WSAGetLastError() != WSA_IO_PENDING` 체크 |
| 5 | bytesTransferred=0 미처리 | 좀비 세션 발생 | Graceful close로 세션 정리 |
| 6 | AcceptEx 보충 누락 | 더 이상 연결 수락 불가 | Accept 완료 시 반드시 새 AcceptEx 예약 |
| 7 | Send 완료 전 버퍼 재사용 | 데이터 손상 | Send 완료 콜백 후에만 버퍼 재사용 |
| 8 | 멀티스레드 세션 접근 비동기화 | 데이터 경합, 크래시 | 세션별 Lock 또는 설계로 경합 방지 |
| 9 | 에러 코드 미확인 | 문제 원인 파악 불가 | GQCS FALSE 시 GetLastError 확인 |
| 10 | AcceptEx 버퍼 크기 계산 오류 | Accept 완료 안 됨 | `sizeof(SOCKADDR) + 16` 공식 준수 |

### 3.6.2 디버깅 체크리스트

```
IOCP 서버가 동작하지 않을 때:

□ WSAStartup 반환값 확인
□ 소켓 생성 시 WSA_FLAG_OVERLAPPED 사용 확인
□ bind/listen 에러 확인 (포트 충돌?)
□ AcceptEx 함수 포인터가 NULL이 아닌지 확인
□ CreateIoCompletionPort 반환값 확인
□ Worker 스레드가 실제로 생성되었는지 확인
□ GetQueuedCompletionStatus 호출이 블로킹되고 있는지 확인
□ AcceptEx의 출력 버퍼 크기가 올바른지 확인
□ 방화벽이 포트를 차단하고 있지 않은지 확인
```

---

## 3.7 퀴즈용 핵심 Q&A

### 정의형 문제

**Q1**: `socket()` 함수로 생성한 소켓과 `WSASocket(..., WSA_FLAG_OVERLAPPED)`로 생성한 소켓의 차이는?
> **A**: `socket()`은 내부적으로 `WSA_FLAG_OVERLAPPED`를 자동 포함하여 `WSASocket`을 호출하므로, 둘 다 Overlapped I/O를 지원한다. 차이가 없다. 단, `WSASocket(..., 0)`으로 플래그 없이 호출하면 Overlapped 미지원 소켓이 생성된다.

**Q2**: AcceptEx 함수 포인터를 WSAIoctl로 획득해야 하는 이유는?
> **A**: Winsock은 프로바이더 아키텍처로 설계되어 확장 함수가 프로바이더마다 다를 수 있다. mswsock.dll을 직접 링크하면 LSP(Layered Service Provider)를 우회하게 되어 방화벽/VPN 등의 네트워크 필터가 작동하지 않을 수 있다.

### 수치형 문제

**Q3**: TCP MSS가 1460바이트인 이유는?
> **A**: 이더넷 MTU 1500 - IP 헤더 20 - TCP 헤더 20 = 1460바이트.

**Q4**: AcceptEx의 주소 버퍼에서 `sizeof(SOCKADDR_IN) + 16`이 필요한 이유는?
> **A**: Microsoft가 AcceptEx 내부적으로 주소 정보 외에 16바이트의 추가 공간을 요구하도록 설계했다. 로컬 주소와 원격 주소 각각에 대해 `sizeof(SOCKADDR) + 16`을 할당해야 한다. 이 16바이트는 내부 구현을 위한 여유 공간이다.

### 적용형 문제

**Q5**: AcceptEx 예약이 모두 소진되고 listen backlog도 가득 차면 어떤 일이 발생하는가?
> **A**: 새 클라이언트 연결 시도가 거부된다. TCP 레벨에서 RST가 전송되거나, 클라이언트 측에서 연결 타임아웃이 발생한다. 이를 방지하려면 Accept 완료 시 즉시 새 AcceptEx를 보충하고, 동적 조절 알고리즘으로 대기 Accept 수를 관리해야 한다.

**Q6**: Echo 서버에서 OP_SEND 완료 후 바로 PostRecv를 호출하는 이유는?
> **A**: IOCP에서 Recv는 "예약"해두지 않으면 데이터가 도착해도 알림을 받을 수 없다. Send가 완료되면 클라이언트의 다음 데이터를 받기 위해 새 Recv를 예약해야 한다. Recv와 Send에 같은 버퍼를 사용하는 경우 Send 완료 전에 Recv를 예약하면 버퍼가 덮어써질 수 있으므로, Send 완료 후에 예약하는 것이 안전하다.

---

## 참고 자료

- Microsoft Docs: [AcceptEx function](https://learn.microsoft.com/en-us/windows/win32/api/mswsock/nf-mswsock-acceptex)
- Microsoft Docs: [WSAIoctl - SIO_GET_EXTENSION_FUNCTION_POINTER](https://learn.microsoft.com/en-us/windows/win32/winsock/winsock-ioctls)
- Microsoft Docs: [WSASocket function](https://learn.microsoft.com/en-us/windows/win32/api/winsock2/nf-winsock2-wsasocketw)
- Len Holgate, "Windows IOCP - AcceptEx" (serverframework.com)
- Windows Network Programming (Anthony Jones, Jim Ohlund) - Chapter 5
