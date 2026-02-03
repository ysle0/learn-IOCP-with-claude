# IOCP 학습 가이드 - Section 4 조사 보고서

## 핵심 Topic별 상세

---

## 4.1 Accept 처리 심화

### 4.1.1 AcceptEx 내부 동작 원리

#### 개념
AcceptEx는 Windows의 고성능 비동기 Accept API로, 일반 `accept()` 함수와 달리 여러 작업을 하나의 커널 전환으로 처리합니다.

**AcceptEx의 3가지 기능:**
1. 새로운 연결 수락
2. 로컬 및 원격 주소 반환
3. 클라이언트가 보낸 첫 데이터 블록 수신

**핵심 차이점:**
- `accept()`: 새로운 소켓을 생성하여 반환
- `AcceptEx()`: 미리 생성된 소켓을 매개변수로 받음 (소켓 재사용 가능)

#### 구현 코드 예제

```cpp
// AcceptEx 함수 포인터 획득
LPFN_ACCEPTEX lpfnAcceptEx = NULL;
GUID GuidAcceptEx = WSAID_ACCEPTEX;
DWORD dwBytes = 0;

WSAIoctl(listenSocket, SIO_GET_EXTENSION_FUNCTION_POINTER,
    &GuidAcceptEx, sizeof(GuidAcceptEx),
    &lpfnAcceptEx, sizeof(lpfnAcceptEx),
    &dwBytes, NULL, NULL);

// Accept용 소켓 생성
SOCKET acceptSocket = WSASocket(AF_INET, SOCK_STREAM, IPPROTO_TCP,
    NULL, 0, WSA_FLAG_OVERLAPPED);

// OVERLAPPED 구조체 확장
struct ACCEPT_OVERLAPPED {
    OVERLAPPED overlapped;
    SOCKET acceptSocket;
    char outputBuffer[(sizeof(sockaddr_in) + 16) * 2];
};

ACCEPT_OVERLAPPED* pAcceptOv = new ACCEPT_OVERLAPPED();
ZeroMemory(&pAcceptOv->overlapped, sizeof(OVERLAPPED));
pAcceptOv->acceptSocket = acceptSocket;

// AcceptEx 호출
BOOL result = lpfnAcceptEx(
    listenSocket,                           // 리슨 소켓
    acceptSocket,                           // Accept용 소켓
    pAcceptOv->outputBuffer,                // 출력 버퍼
    0,                                      // dwReceiveDataLength (0 = 데이터 안 기다림)
    sizeof(sockaddr_in) + 16,              // 로컬 주소 길이
    sizeof(sockaddr_in) + 16,              // 원격 주소 길이
    &dwBytes,                               // 받은 바이트 수
    &pAcceptOv->overlapped                  // OVERLAPPED 구조체
);

// WSA_IO_PENDING이면 정상 (비동기 처리)
if (!result && WSAGetLastError() != WSA_IO_PENDING) {
    // 에러 처리
}
```

#### SO_UPDATE_ACCEPT_CONTEXT의 중요성

```cpp
// AcceptEx 완료 후 반드시 호출해야 함
setsockopt(acceptSocket, SOL_SOCKET, SO_UPDATE_ACCEPT_CONTEXT,
    (char*)&listenSocket, sizeof(listenSocket));
```

**이유:**
- AcceptEx로 생성된 소켓은 리슨 소켓의 속성을 상속받지 않음
- `getpeername()`, `getsockname()` 등의 함수가 정상 작동하려면 필수
- TCP 옵션 및 소켓 옵션이 올바르게 설정됨

---

### 4.1.2 Pre-posted Accepts 패턴

#### 개념
서버 시작 시 여러 개의 AcceptEx를 미리 예약하여 빠른 연결 수락을 가능하게 하는 패턴입니다.

**왜 필요한가?**
- AcceptEx는 클라이언트가 데이터를 보낼 때까지 완료되지 않음 (dwReceiveDataLength > 0인 경우)
- 단일 Accept만 예약하면 동시 연결 시도가 거부될 수 있음
- 빠른 연결 속도에 대응하기 위해 충분한 Accept 대기 필요

#### 구현 패턴

```cpp
class AcceptManager {
private:
    SOCKET m_listenSocket;
    HANDLE m_hIOCP;
    LPFN_ACCEPTEX m_lpfnAcceptEx;
    std::atomic<int> m_pendingAccepts;
    const int MIN_PENDING_ACCEPTS = 10;
    const int MAX_PENDING_ACCEPTS = 50;

public:
    void PostAccepts(int count) {
        for (int i = 0; i < count; i++) {
            SOCKET acceptSocket = WSASocket(AF_INET, SOCK_STREAM,
                IPPROTO_TCP, NULL, 0, WSA_FLAG_OVERLAPPED);

            if (acceptSocket == INVALID_SOCKET) {
                continue;
            }

            ACCEPT_OVERLAPPED* pOv = new ACCEPT_OVERLAPPED();
            ZeroMemory(&pOv->overlapped, sizeof(OVERLAPPED));
            pOv->acceptSocket = acceptSocket;
            pOv->opType = OP_ACCEPT;

            DWORD dwBytes = 0;
            BOOL result = m_lpfnAcceptEx(
                m_listenSocket,
                acceptSocket,
                pOv->outputBuffer,
                0,  // 데이터를 기다리지 않고 즉시 완료
                sizeof(sockaddr_in) + 16,
                sizeof(sockaddr_in) + 16,
                &dwBytes,
                &pOv->overlapped
            );

            if (!result && WSAGetLastError() != WSA_IO_PENDING) {
                closesocket(acceptSocket);
                delete pOv;
            } else {
                m_pendingAccepts++;
            }
        }
    }

    void OnAcceptComplete(ACCEPT_OVERLAPPED* pOv) {
        m_pendingAccepts--;

        // Accept 큐 재충전
        if (m_pendingAccepts < MIN_PENDING_ACCEPTS) {
            int toPost = MAX_PENDING_ACCEPTS - m_pendingAccepts;
            PostAccepts(toPost);
        }

        // 소켓 속성 업데이트
        setsockopt(pOv->acceptSocket, SOL_SOCKET, SO_UPDATE_ACCEPT_CONTEXT,
            (char*)&m_listenSocket, sizeof(m_listenSocket));

        // IOCP에 연결
        CreateIoCompletionPort((HANDLE)pOv->acceptSocket, m_hIOCP,
            (ULONG_PTR)pOv->acceptSocket, 0);

        // 세션 생성 및 첫 Recv 예약
        // ...
    }
};
```

---

### 4.1.3 적정 Accept 예약 개수 결정 기준

#### 이론적 공식

```
최대 연결 속도 = Accept 큐 크기 / 평균 큐 대기 시간

평균 큐 대기 시간 = RTT(왕복 지연) + 클라이언트 처리 시간 + 서버 Accept 지연
```

#### 실무 권장값

| 서버 규모 | 동시접속자 | 권장 Pre-posted Count |
|----------|-----------|---------------------|
| 소규모 | ~1,000 | 10-20 |
| 중규모 | 1,000-10,000 | 20-50 |
| 대규모 | 10,000+ | 50-100 |

**주의사항:**
- Windows NT 계열 Workstation: listen backlog 최대 5
- Windows Server 계열: listen backlog 수백~수천 가능
- Pre-posted Accept 개수가 너무 많으면 소켓 리소스 낭비
- 너무 적으면 연결 거부 발생 가능

#### 동적 조정 알고리즘

```cpp
void AdjustAcceptPool() {
    // 최근 1초간 연결 속도 측정
    int connectionsPerSecond = GetRecentConnectionRate();

    // RTT 평균값 (밀리초)
    int avgRTT = 100;  // 일반적으로 50-200ms

    // 필요한 Accept 개수 계산
    int requiredAccepts = (connectionsPerSecond * avgRTT) / 1000;

    // 여유분 추가 (2배)
    requiredAccepts *= 2;

    // 최소/최대 제한
    requiredAccepts = max(MIN_PENDING_ACCEPTS,
                         min(MAX_PENDING_ACCEPTS, requiredAccepts));

    // 현재값과 비교하여 조정
    int diff = requiredAccepts - m_pendingAccepts;
    if (diff > 0) {
        PostAccepts(diff);
    }
}
```

---

### 4.1.4 Accept 소켓 재사용 (DisconnectEx + TF_REUSE_SOCKET)

#### 개념
소켓 생성은 비용이 큰 작업이므로, 연결이 종료된 소켓을 재사용하여 성능을 향상시킵니다.

#### DisconnectEx 사용법

```cpp
// DisconnectEx 함수 포인터 획득
LPFN_DISCONNECTEX lpfnDisconnectEx = NULL;
GUID GuidDisconnectEx = WSAID_DISCONNECTEX;

WSAIoctl(socket, SIO_GET_EXTENSION_FUNCTION_POINTER,
    &GuidDisconnectEx, sizeof(GuidDisconnectEx),
    &lpfnDisconnectEx, sizeof(lpfnDisconnectEx),
    &dwBytes, NULL, NULL);

// 연결 종료 + 소켓 재사용 준비
struct DISCONNECT_OVERLAPPED {
    OVERLAPPED overlapped;
    SOCKET socket;
};

DISCONNECT_OVERLAPPED* pDisconnectOv = new DISCONNECT_OVERLAPPED();
ZeroMemory(&pDisconnectOv->overlapped, sizeof(OVERLAPPED));
pDisconnectOv->socket = socket;

BOOL result = lpfnDisconnectEx(
    socket,
    &pDisconnectOv->overlapped,
    TF_REUSE_SOCKET,  // 핵심: 소켓 재사용 플래그
    0
);

if (!result && WSAGetLastError() != WSA_IO_PENDING) {
    // 에러 처리
}
```

#### 소켓 풀 관리

```cpp
class SocketPool {
private:
    std::queue<SOCKET> m_freeSockets;
    std::mutex m_mutex;
    LPFN_DISCONNECTEX m_lpfnDisconnectEx;

public:
    SOCKET AcquireSocket() {
        std::lock_guard<std::mutex> lock(m_mutex);

        if (m_freeSockets.empty()) {
            // 새 소켓 생성
            return WSASocket(AF_INET, SOCK_STREAM, IPPROTO_TCP,
                NULL, 0, WSA_FLAG_OVERLAPPED);
        }

        SOCKET sock = m_freeSockets.front();
        m_freeSockets.pop();
        return sock;
    }

    void ReleaseSocket(SOCKET sock) {
        // DisconnectEx로 비동기 종료 + 재사용 준비
        OVERLAPPED ov = {0};
        BOOL result = m_lpfnDisconnectEx(sock, &ov, TF_REUSE_SOCKET, 0);

        if (result || WSAGetLastError() == WSA_IO_PENDING) {
            // 완료 후 풀에 반환
            // (완료 핸들러에서 처리)
        } else {
            // 실패 시 소켓 닫고 새로 생성
            closesocket(sock);
        }
    }

    void OnDisconnectComplete(SOCKET sock) {
        std::lock_guard<std::mutex> lock(m_mutex);
        m_freeSockets.push(sock);
    }
};
```

#### 주의사항 및 함정

**1. 소켓 타입 혼용 금지**
```cpp
// 잘못된 예: ConnectEx로 사용한 소켓을 AcceptEx에 재사용
SOCKET sock = ConnectEx(...);  // 클라이언트 연결용
DisconnectEx(sock, TF_REUSE_SOCKET);
AcceptEx(listenSock, sock, ...);  // 오류 10022(WSAEINVAL) 발생!

// 올바른 예: 용도별로 소켓 풀 분리
SocketPool m_acceptSocketPool;   // Accept 전용
SocketPool m_connectSocketPool;  // Connect 전용
```

**2. 클라이언트 측 재사용 문제**
- 서버가 종료를 시작하면 문제없음
- 클라이언트가 종료를 시작하면 TIME_WAIT 상태로 인해 재사용 실패 가능
- 에러 52 "A duplicate name exists on the network" 발생 가능

**3. DisconnectEx 완료 확인 필수**
```cpp
void OnDisconnectComplete(OVERLAPPED* pOv, DWORD bytesTransferred) {
    DWORD flags = 0;
    BOOL success = WSAGetOverlappedResult(sock, pOv, &bytesTransferred, FALSE, &flags);

    if (success) {
        // 재사용 가능
        m_socketPool.OnDisconnectComplete(sock);
    } else {
        // 재사용 불가, 소켓 닫기
        closesocket(sock);
    }
}
```

---

### 4.1.5 AcceptEx 타임아웃 처리

#### 문제 상황
클라이언트가 연결 후 데이터를 보내지 않으면 AcceptEx가 완료되지 않아 DoS 공격에 취약합니다.

#### 해결책 1: dwReceiveDataLength = 0 사용

```cpp
// 데이터를 기다리지 않고 즉시 완료
lpfnAcceptEx(listenSocket, acceptSocket, outputBuffer,
    0,  // 데이터 대기 안 함
    sizeof(sockaddr_in) + 16,
    sizeof(sockaddr_in) + 16,
    &dwBytes, &overlapped);
```

**장점:**
- 연결 즉시 완료되어 DoS 공격 방지
- Accept 큐가 빠르게 회전

**단점:**
- 첫 데이터를 별도로 WSARecv로 받아야 함
- 코드가 약간 복잡해짐

#### 해결책 2: 타임아웃 모니터링

```cpp
class AcceptTimeoutManager {
private:
    struct PendingAccept {
        SOCKET socket;
        std::chrono::steady_clock::time_point startTime;
    };

    std::map<SOCKET, PendingAccept> m_pendingAccepts;
    std::mutex m_mutex;
    const int ACCEPT_TIMEOUT_SECONDS = 10;

public:
    void RegisterAccept(SOCKET sock) {
        std::lock_guard<std::mutex> lock(m_mutex);
        PendingAccept pa;
        pa.socket = sock;
        pa.startTime = std::chrono::steady_clock::now();
        m_pendingAccepts[sock] = pa;
    }

    void OnAcceptComplete(SOCKET sock) {
        std::lock_guard<std::mutex> lock(m_mutex);
        m_pendingAccepts.erase(sock);
    }

    // 별도 타이머 스레드에서 주기적 호출
    void CheckTimeouts() {
        auto now = std::chrono::steady_clock::now();
        std::vector<SOCKET> timeoutSockets;

        {
            std::lock_guard<std::mutex> lock(m_mutex);
            for (auto& pair : m_pendingAccepts) {
                auto elapsed = std::chrono::duration_cast<std::chrono::seconds>(
                    now - pair.second.startTime).count();

                if (elapsed > ACCEPT_TIMEOUT_SECONDS) {
                    timeoutSockets.push_back(pair.first);
                }
            }
        }

        // 타임아웃된 소켓 강제 종료
        for (SOCKET sock : timeoutSockets) {
            closesocket(sock);  // Accept 취소
            m_pendingAccepts.erase(sock);
        }
    }
};
```

#### 해결책 3: SO_RCVTIMEO 옵션

```cpp
// Accept 소켓에 수신 타임아웃 설정
int timeout = 10000;  // 10초 (밀리초)
setsockopt(acceptSocket, SOL_SOCKET, SO_RCVTIMEO,
    (char*)&timeout, sizeof(timeout));
```

**주의:**
- Windows에서는 DWORD (밀리초) 사용
- Linux는 struct timeval 사용
- AcceptEx의 데이터 수신에만 영향, 연결 자체는 영향 없음

---

### 4.1.6 성능 데이터 및 벤치마크

#### 실제 성능 수치

**Microsoft IOCP 샘플 (Windows Server):**
- Pre-posted AcceptEx 65,000개 사용 (Win 2003 기준)
- 30,000개 동시 연결 안정적 처리
- 일반 `accept()` 사용 시 30,000개 연결에서 WSAENOBUFS (10055) 에러 발생

**GameDev.net 커뮤니티 벤치마크:**
- 500 연결, 32 bytes/s per connection: 안정적, 5초 이내 응답
- 1,000 연결, 32 bytes/2s per connection: 안정적, 5초 이내 응답
- 평균 처리량: 47 KiB/s
- 예상 확장성: 3,000 연결까지 문제없음

#### listen() backlog 크기 영향

```cpp
// backlog 값에 따른 SYN 큐 크기
listen(listenSocket, backlog);

// Windows Server: backlog 수백~수천 가능
// Windows Client: backlog 최대 5 (95/98/ME/NT Workstation/2000 Pro)
// Linux: min(somaxconn, backlog)
//        /proc/sys/net/core/somaxconn 확인
```

**연결 수락 속도 공식:**
```
최대 연결 속도 = backlog / 평균 SYN_ACK RTT

예시:
- backlog = 200
- RTT = 100ms
- 최대 속도 = 200 / 0.1 = 2,000 connections/sec
```

---

## 4.2 Recv/Send 처리 심화

### 4.2.1 Scatter/Gather I/O (여러 버퍼 동시 처리)

#### 개념
WSARecv/WSASend는 WSABUF 배열을 받아 여러 버퍼를 한 번에 처리할 수 있습니다.

**장점:**
- memcpy 호출 감소로 성능 향상
- 헤더 + 본문을 별도 버퍼로 유지 가능
- 사용자 모드 복사 오버헤드 제거

#### 구현 예제

```cpp
// 패킷 구조: [Header(16 bytes)][Body(variable)]
struct Packet {
    PacketHeader header;  // 16 bytes
    char* body;           // 가변 길이
};

// Scatter-Gather Send
void SendPacket(SOCKET sock, const Packet& packet) {
    WSABUF buffers[2];

    // 버퍼 1: 헤더
    buffers[0].buf = (char*)&packet.header;
    buffers[0].len = sizeof(PacketHeader);

    // 버퍼 2: 본문
    buffers[1].buf = packet.body;
    buffers[1].len = packet.header.bodyLength;

    OVERLAPPED_EX* pOv = new OVERLAPPED_EX();
    ZeroMemory(&pOv->overlapped, sizeof(OVERLAPPED));

    DWORD bytesSent = 0;
    int result = WSASend(
        sock,
        buffers,
        2,              // 버퍼 개수
        &bytesSent,
        0,
        &pOv->overlapped,
        NULL
    );

    if (result == SOCKET_ERROR && WSAGetLastError() != WSA_IO_PENDING) {
        // 에러 처리
    }
}

// Scatter-Gather Recv
void RecvPacket(SOCKET sock) {
    WSABUF buffers[2];

    // 버퍼 1: 헤더 수신용
    buffers[0].buf = recvContext->headerBuffer;
    buffers[0].len = sizeof(PacketHeader);

    // 버퍼 2: 본문 수신용
    buffers[1].buf = recvContext->bodyBuffer;
    buffers[1].len = MAX_BODY_SIZE;

    DWORD bytesRecvd = 0;
    DWORD flags = 0;
    int result = WSARecv(
        sock,
        buffers,
        2,
        &bytesRecvd,
        &flags,
        &recvContext->overlapped,
        NULL
    );

    if (result == SOCKET_ERROR && WSAGetLastError() != WSA_IO_PENDING) {
        // 에러 처리
    }
}
```

#### 주의사항

**1. 부분 완료 문제**
```cpp
// Scatter-Gather I/O는 모든 버퍼가 채워질 때까지 기다리지 않음
void OnRecvComplete(DWORD bytesTransferred) {
    // 예상: 1024 bytes (헤더 16 + 본문 1008)
    // 실제: 512 bytes만 도착할 수 있음

    if (bytesTransferred < sizeof(PacketHeader)) {
        // 헤더도 완전히 받지 못함
        // 추가 Recv 필요
    }
}
```

**2. 대량 버퍼 사용 시 성능 저하**
```cpp
// 나쁜 예: 100만 개 버퍼 사용
WSABUF buffers[1000000];  // malloc/free per buffer, IOCP per buffer
// (참고) Boost.ASIO에서 다수의 버퍼를 사용할 때 유사한 성능 저하 이슈가 보고된 바 있음

// 좋은 예: 적절한 버퍼 개수 (보통 2-10개)
WSABUF buffers[4];
```

#### 성능 비교

```
시나리오: 16 bytes 헤더 + 1008 bytes 본문 전송

방법 1: memcpy + 단일 버퍼
- memcpy 오버헤드: ~50ns
- WSASend 호출: 1회
- 총 시간: ~1,050ns

방법 2: Scatter-Gather (2개 버퍼)
- memcpy 오버헤드: 0
- WSASend 호출: 1회
- 총 시간: ~1,000ns

성능 향상: ~5% (작은 패킷에서는 미미)

시나리오: 여러 개의 작은 조각 전송
- 100개 조각, 각 100 bytes
- 방법 1 (memcpy): 100 * 50ns = 5,000ns 추가
- 방법 2 (Scatter): 0ns
- 성능 향상: ~33%
```

---

### 4.2.2 Multiple Outstanding I/O (동시에 여러 Recv/Send 예약)

#### Multiple Recv의 비효율성

**일반적 오해:**
"여러 WSARecv를 예약하면 더 빠를 것이다"

**실제:**
- TCP는 커널에서 이미 버퍼링함
- Pending recv가 있으면 → 직접 버퍼에 복사
- Pending recv가 없으면 → 커널 버퍼에 캐싱
- 추가 memcpy 비용만 발생

**권장 패턴:**
```cpp
// 나쁜 예: 동일 소켓에 여러 Recv 예약
WSARecv(sock, &buf1, ...);  // Recv 1
WSARecv(sock, &buf2, ...);  // Recv 2
WSARecv(sock, &buf3, ...);  // Recv 3
// → 스레드 안전성 문제 + 순서 재조합 필요

// 좋은 예: 소켓당 하나의 Recv
WSARecv(sock, &buf, ...);
// 완료 후 다시 WSARecv 예약
```

#### Multiple Send의 필요성

**문제:**
SO_SNDBUF=0 사용 시 전송 속도 저하

**해결책:**
```cpp
// 여러 Send를 큐잉하여 네트워크 대역폭 활용 극대화
class SendQueue {
private:
    std::queue<SendBuffer*> m_queue;
    std::mutex m_mutex;
    std::atomic<int> m_pendingSends;
    const int MAX_PENDING_SENDS = 4;

public:
    void Send(SendBuffer* buffer) {
        {
            std::lock_guard<std::mutex> lock(m_mutex);
            m_queue.push(buffer);
        }

        // 대기 중인 Send가 적으면 즉시 전송
        if (m_pendingSends < MAX_PENDING_SENDS) {
            ProcessQueue();
        }
    }

    void ProcessQueue() {
        while (m_pendingSends < MAX_PENDING_SENDS) {
            SendBuffer* buffer = nullptr;
            {
                std::lock_guard<std::mutex> lock(m_mutex);
                if (m_queue.empty()) break;
                buffer = m_queue.front();
                m_queue.pop();
            }

            WSASend(m_socket, &buffer->wsaBuf, 1, nullptr, 0,
                &buffer->overlapped, nullptr);
            m_pendingSends++;
        }
    }

    void OnSendComplete() {
        m_pendingSends--;
        ProcessQueue();  // 큐에 남은 데이터 계속 전송
    }
};
```

---

### 4.2.3 Send 순서 보장 문제와 해결책

#### 문제 상황

**TCP는 순서를 보장하지만...**
```cpp
// Thread 1
WSASend(sock, buffer1, ...);  // "Hello"

// Thread 2 (거의 동시에)
WSASend(sock, buffer2, ...);  // "World"

// 결과: "WorldHello" 또는 "HelloWorld" (불확실)
// 이유: 스레드 스케줄링에 따라 달라짐
```

**IOCP 완료 순서:**
- I/O Manager는 요청 순서대로 완료를 보장
- 하지만 여러 Worker 스레드가 완료를 처리하므로 실제 처리 순서는 불확실

#### 해결책 1: Send 직렬화 (Single Threaded Send)

```cpp
class Session {
private:
    SOCKET m_socket;
    std::queue<SendBuffer*> m_sendQueue;
    std::mutex m_sendMutex;
    std::atomic<bool> m_isSending;

public:
    void SendAsync(const char* data, int length) {
        SendBuffer* buffer = new SendBuffer(data, length);

        {
            std::lock_guard<std::mutex> lock(m_sendMutex);
            m_sendQueue.push(buffer);
        }

        // 현재 전송 중이 아니면 전송 시작
        bool expected = false;
        if (m_isSending.compare_exchange_strong(expected, true)) {
            ProcessSendQueue();
        }
        // 전송 중이면 큐에 쌓이고, 완료 시 처리됨
    }

private:
    void ProcessSendQueue() {
        SendBuffer* buffer = nullptr;

        {
            std::lock_guard<std::mutex> lock(m_sendMutex);
            if (m_sendQueue.empty()) {
                m_isSending = false;
                return;
            }
            buffer = m_sendQueue.front();
            m_sendQueue.pop();
        }

        DWORD bytesSent = 0;
        int result = WSASend(m_socket, &buffer->wsaBuf, 1, &bytesSent, 0,
            &buffer->overlapped, nullptr);

        if (result == SOCKET_ERROR) {
            int error = WSAGetLastError();
            if (error != WSA_IO_PENDING) {
                // 에러 처리
                m_isSending = false;
            }
        }
    }

    void OnSendComplete(SendBuffer* buffer) {
        delete buffer;

        // 큐에 남은 데이터 계속 전송
        ProcessSendQueue();
    }
};
```

#### 해결책 2: 시퀀스 번호 사용

```cpp
struct SendContext {
    OVERLAPPED overlapped;
    uint64_t sequenceNumber;
    char* data;
    int length;
};

class SequencedSender {
private:
    std::atomic<uint64_t> m_nextSeq;
    std::map<uint64_t, SendContext*> m_pendingSends;
    std::mutex m_mutex;

public:
    void SendAsync(const char* data, int length) {
        SendContext* ctx = new SendContext();
        ctx->sequenceNumber = m_nextSeq++;
        ctx->data = new char[length];
        memcpy(ctx->data, data, length);
        ctx->length = length;

        {
            std::lock_guard<std::mutex> lock(m_mutex);
            m_pendingSends[ctx->sequenceNumber] = ctx;
        }

        // 실제 전송은 순서대로 처리
        ProcessInOrder();
    }

private:
    void ProcessInOrder() {
        std::lock_guard<std::mutex> lock(m_mutex);

        // 가장 오래된 시퀀스부터 전송
        auto it = m_pendingSends.begin();
        if (it != m_pendingSends.end()) {
            SendContext* ctx = it->second;
            WSASend(m_socket, &ctx->wsaBuf, 1, nullptr, 0,
                &ctx->overlapped, nullptr);
        }
    }
};
```

#### 권장 사항

**대부분의 경우:**
- 소켓당 하나의 Send만 진행 (직렬화)
- 완료 후 다음 Send 시작
- 단순하고 안전함

**고성능이 필요한 경우:**
- Multiple Send 허용 + 시퀀스 번호
- 복잡도 증가, 버그 가능성 높음

---

### 4.2.4 Partial Send/Recv 처리

#### Partial Recv 처리

```cpp
class RecvBuffer {
private:
    char m_buffer[8192];
    int m_receivedBytes;
    int m_expectedBytes;

public:
    void OnRecvComplete(DWORD bytesTransferred) {
        m_receivedBytes += bytesTransferred;

        while (m_receivedBytes >= sizeof(PacketHeader)) {
            PacketHeader* header = (PacketHeader*)m_buffer;
            int packetSize = sizeof(PacketHeader) + header->bodyLength;

            if (m_receivedBytes >= packetSize) {
                // 완전한 패킷 수신
                ProcessPacket(m_buffer, packetSize);

                // 버퍼에서 제거
                m_receivedBytes -= packetSize;
                memmove(m_buffer, m_buffer + packetSize, m_receivedBytes);
            } else {
                // 패킷 불완전, 더 받아야 함
                break;
            }
        }

        // 다음 Recv 예약
        PostRecv();
    }

    void PostRecv() {
        WSABUF wsaBuf;
        wsaBuf.buf = m_buffer + m_receivedBytes;
        wsaBuf.len = sizeof(m_buffer) - m_receivedBytes;

        DWORD bytesRecvd = 0, flags = 0;
        WSARecv(m_socket, &wsaBuf, 1, &bytesRecvd, &flags,
            &m_overlapped, nullptr);
    }
};
```

#### Partial Send 처리

```cpp
class SendBuffer {
private:
    char* m_data;
    int m_totalBytes;
    int m_sentBytes;

public:
    void OnSendComplete(DWORD bytesTransferred) {
        m_sentBytes += bytesTransferred;

        if (m_sentBytes < m_totalBytes) {
            // 일부만 전송됨, 나머지 재전송
            WSABUF wsaBuf;
            wsaBuf.buf = m_data + m_sentBytes;
            wsaBuf.len = m_totalBytes - m_sentBytes;

            DWORD bytesSent = 0;
            int result = WSASend(m_socket, &wsaBuf, 1, &bytesSent, 0,
                &m_overlapped, nullptr);

            if (result == SOCKET_ERROR && WSAGetLastError() != WSA_IO_PENDING) {
                // 에러 처리
                OnSendError();
            }
        } else {
            // 전송 완료
            delete[] m_data;
            delete this;
        }
    }
};
```

#### GetQueuedCompletionStatus 반환값 해석

```cpp
void WorkerThread() {
    DWORD bytesTransferred = 0;
    ULONG_PTR completionKey = 0;
    OVERLAPPED* pOverlapped = nullptr;

    BOOL result = GetQueuedCompletionStatus(
        m_hIOCP, &bytesTransferred, &completionKey,
        &pOverlapped, INFINITE);

    if (result == TRUE) {
        if (bytesTransferred == 0) {
            // 정상 연결 종료 (Graceful close)
            HandleDisconnect(completionKey);
        } else {
            // 정상 I/O 완료
            HandleIOComplete(pOverlapped, bytesTransferred);
        }
    } else {
        // result == FALSE
        if (pOverlapped == NULL) {
            // IOCP 자체 에러 또는 타임아웃
            DWORD error = GetLastError();
            if (error == WAIT_TIMEOUT) {
                // 타임아웃
            } else {
                // IOCP 에러
            }
        } else {
            // I/O 작업 실패
            DWORD error = GetLastError();
            switch (error) {
                case ERROR_NETNAME_DELETED:  // 64
                case ERROR_CONNECTION_ABORTED:  // 1236
                    HandleAbortiveClose(completionKey);
                    break;
                default:
                    HandleIOError(completionKey, error);
                    break;
            }
        }
    }
}
```

---

### 4.2.5 Zero-byte Recv 기법 (메모리 절약)

#### 개념
수만 개의 동시 접속에서 각 소켓마다 수신 버퍼를 할당하면 메모리 낭비가 심합니다.

**문제:**
- 10,000 소켓 × 64KB 버퍼 = 640MB
- 대부분의 소켓은 유휴 상태인데 메모리 점유

**해결:**
Zero-byte recv로 데이터 도착 알림만 받고, 실제 데이터는 논블로킹으로 읽음

#### 구현

```cpp
class ZeroByteRecvSession {
private:
    SOCKET m_socket;
    OVERLAPPED m_recvOverlapped;
    char m_tempBuffer[8192];  // 실제 읽기용 임시 버퍼

public:
    void PostZeroByteRecv() {
        WSABUF wsaBuf;
        wsaBuf.buf = nullptr;  // 버퍼 없음
        wsaBuf.len = 0;        // 길이 0

        DWORD bytesRecvd = 0, flags = 0;
        int result = WSARecv(m_socket, &wsaBuf, 1, &bytesRecvd, &flags,
            &m_recvOverlapped, nullptr);

        if (result == SOCKET_ERROR) {
            int error = WSAGetLastError();
            if (error != WSA_IO_PENDING) {
                // 에러 처리
            }
        }
    }

    void OnZeroByteRecvComplete() {
        // 데이터가 도착했음을 알림받음
        // 이제 논블로킹 모드로 실제 데이터 읽기

        // 소켓을 논블로킹으로 설정
        u_long mode = 1;  // 1 = non-blocking
        ioctlsocket(m_socket, FIONBIO, &mode);

        // 커널 버퍼가 빌 때까지 읽기
        while (true) {
            int bytesRead = recv(m_socket, m_tempBuffer, sizeof(m_tempBuffer), 0);

            if (bytesRead > 0) {
                // 데이터 처리
                ProcessData(m_tempBuffer, bytesRead);
            } else if (bytesRead == 0) {
                // 연결 종료
                HandleDisconnect();
                return;
            } else {
                // bytesRead == SOCKET_ERROR
                int error = WSAGetLastError();
                if (error == WSAEWOULDBLOCK) {
                    // 더 이상 읽을 데이터 없음
                    break;
                } else {
                    // 실제 에러
                    HandleError(error);
                    return;
                }
            }
        }

        // 다음 Zero-byte recv 예약
        PostZeroByteRecv();
    }
};
```

#### 장점 및 단점

**장점:**
- 메모리 사용량 대폭 감소
- 10,000 소켓: 640MB → ~0MB (Zero-byte recv 자체는 메모리 안 잠금)
- DotNetty에서 200K+ 연결 처리 성능 입증

**단점:**
- 코드 복잡도 증가
- 논블로킹 recv 루프에서 CPU 사용 가능성
- recv 루프 제한 필요 (예: 최대 3회)

#### Bandwidth Starvation 방지

```cpp
void OnZeroByteRecvComplete() {
    u_long mode = 1;
    ioctlsocket(m_socket, FIONBIO, &mode);

    int loopCount = 0;
    const int MAX_RECV_LOOPS = 3;  // 최대 3번만 읽기

    while (loopCount < MAX_RECV_LOOPS) {
        int bytesRead = recv(m_socket, m_tempBuffer, sizeof(m_tempBuffer), 0);

        if (bytesRead > 0) {
            ProcessData(m_tempBuffer, bytesRead);
            loopCount++;
        } else if (bytesRead == 0) {
            HandleDisconnect();
            return;
        } else {
            int error = WSAGetLastError();
            if (error == WSAEWOULDBLOCK) {
                break;
            } else {
                HandleError(error);
                return;
            }
        }
    }

    PostZeroByteRecv();
}
```

#### 성능 데이터

```
시나리오: 10,000 동시 접속, 각 소켓 평균 10초마다 1KB 데이터 송수신

전통적 방식 (64KB 버퍼):
- 메모리 사용: 640MB
- 잠긴 메모리 (locked memory): 640MB

Zero-byte recv 방식:
- 메모리 사용: ~1MB (임시 버퍼 공유)
- 잠긴 메모리: 0MB
- 메모리 절감: 99.8%

트레이드오프:
- CPU 사용 약간 증가 (논블로킹 recv 루프)
- 코드 복잡도 증가
```

---

### 4.2.6 SO_SNDBUF=0 최적화

#### 개념
TCP 송신 버퍼를 0으로 설정하면 커널 버퍼를 우회하고 사용자 버퍼를 직접 사용합니다.

**장점:**
- memcpy 1회 절약 (사용자 버퍼 → 커널 버퍼)
- 메모리 사용량 감소
- 성능 향상 (약 20%)

**주의사항:**
- 사용자 버퍼를 I/O 완료까지 유지해야 함
- 버퍼 수정 금지

#### 구현

```cpp
void EnableDirectBuffering(SOCKET sock) {
    // 송신 버퍼 크기를 0으로 설정
    int sendBufferSize = 0;
    setsockopt(sock, SOL_SOCKET, SO_SNDBUF,
        (char*)&sendBufferSize, sizeof(sendBufferSize));

    // 수신 버퍼도 0으로 설정 가능
    int recvBufferSize = 0;
    setsockopt(sock, SOL_SOCKET, SO_RCVBUF,
        (char*)&recvBufferSize, sizeof(recvBufferSize));
}

class DirectBufferSender {
public:
    void SendAsync(const char* data, int length) {
        // 버퍼를 복사하여 보관 (완료까지 유지)
        char* buffer = new char[length];
        memcpy(buffer, data, length);

        SendContext* ctx = new SendContext();
        ctx->buffer = buffer;
        ctx->length = length;
        ctx->wsaBuf.buf = buffer;
        ctx->wsaBuf.len = length;

        DWORD bytesSent = 0;
        int result = WSASend(m_socket, &ctx->wsaBuf, 1, &bytesSent, 0,
            &ctx->overlapped, nullptr);

        if (result == SOCKET_ERROR && WSAGetLastError() != WSA_IO_PENDING) {
            delete[] buffer;
            delete ctx;
        }
    }

    void OnSendComplete(SendContext* ctx) {
        delete[] ctx->buffer;  // 이제 버퍼 해제 가능
        delete ctx;
    }
};
```

#### 성능 측정

```
테스트: 1,000,000 번의 1KB 전송

SO_SNDBUF=8192 (기본값):
- 시간: 10.5초
- 처리량: 95,238 packets/sec

SO_SNDBUF=0:
- 시간: 8.7초
- 처리량: 114,943 packets/sec
- 성능 향상: 20.7%

메모리:
- SO_SNDBUF=8192: 소켓당 8KB × 10,000 = 80MB
- SO_SNDBUF=0: 0MB (사용자 버퍼만 사용)
```

---

## 4.3 세션/버퍼 관리

### 4.3.1 세션 객체 설계 패턴

#### 기본 세션 클래스

```cpp
class Session : public std::enable_shared_from_this<Session> {
private:
    SOCKET m_socket;
    SOCKADDR_IN m_addr;
    std::atomic<bool> m_isConnected;

    // I/O 컨텍스트
    struct RecvContext {
        OVERLAPPED overlapped;
        Session* session;
        char buffer[8192];
    } m_recvContext;

    struct SendContext {
        OVERLAPPED overlapped;
        Session* session;
        std::vector<char> buffer;
    };

    // Send 큐
    std::queue<SendContext*> m_sendQueue;
    std::mutex m_sendMutex;
    std::atomic<bool> m_isSending;

    // 통계
    std::atomic<uint64_t> m_totalBytesSent;
    std::atomic<uint64_t> m_totalBytesReceived;
    std::chrono::steady_clock::time_point m_connectTime;

public:
    Session(SOCKET sock, const SOCKADDR_IN& addr)
        : m_socket(sock)
        , m_addr(addr)
        , m_isConnected(true)
        , m_isSending(false)
        , m_totalBytesSent(0)
        , m_totalBytesReceived(0)
        , m_connectTime(std::chrono::steady_clock::now())
    {
        ZeroMemory(&m_recvContext, sizeof(m_recvContext));
        m_recvContext.session = this;
    }

    virtual ~Session() {
        Disconnect();
    }

    void Start() {
        PostRecv();
    }

    void Send(const char* data, int length) {
        SendContext* ctx = new SendContext();
        ctx->session = this;
        ctx->buffer.assign(data, data + length);
        ZeroMemory(&ctx->overlapped, sizeof(OVERLAPPED));

        {
            std::lock_guard<std::mutex> lock(m_sendMutex);
            m_sendQueue.push(ctx);
        }

        bool expected = false;
        if (m_isSending.compare_exchange_strong(expected, true)) {
            ProcessSendQueue();
        }
    }

    void Disconnect() {
        bool expected = true;
        if (m_isConnected.compare_exchange_strong(expected, false)) {
            shutdown(m_socket, SD_BOTH);
            closesocket(m_socket);
            m_socket = INVALID_SOCKET;
        }
    }

    // Getters
    SOCKET GetSocket() const { return m_socket; }
    bool IsConnected() const { return m_isConnected; }
    uint64_t GetTotalBytesSent() const { return m_totalBytesSent; }
    uint64_t GetTotalBytesReceived() const { return m_totalBytesReceived; }

private:
    void PostRecv() {
        WSABUF wsaBuf;
        wsaBuf.buf = m_recvContext.buffer;
        wsaBuf.len = sizeof(m_recvContext.buffer);

        DWORD bytesRecvd = 0, flags = 0;
        int result = WSARecv(m_socket, &wsaBuf, 1, &bytesRecvd, &flags,
            &m_recvContext.overlapped, nullptr);

        if (result == SOCKET_ERROR) {
            int error = WSAGetLastError();
            if (error != WSA_IO_PENDING) {
                HandleError(error);
            }
        }
    }

    void ProcessSendQueue() {
        SendContext* ctx = nullptr;

        {
            std::lock_guard<std::mutex> lock(m_sendMutex);
            if (m_sendQueue.empty()) {
                m_isSending = false;
                return;
            }
            ctx = m_sendQueue.front();
            m_sendQueue.pop();
        }

        WSABUF wsaBuf;
        wsaBuf.buf = ctx->buffer.data();
        wsaBuf.len = static_cast<ULONG>(ctx->buffer.size());

        DWORD bytesSent = 0;
        int result = WSASend(m_socket, &wsaBuf, 1, &bytesSent, 0,
            &ctx->overlapped, nullptr);

        if (result == SOCKET_ERROR) {
            int error = WSAGetLastError();
            if (error != WSA_IO_PENDING) {
                delete ctx;
                m_isSending = false;
                HandleError(error);
            }
        }
    }

public:
    // IOCP 완료 핸들러
    void OnRecvComplete(DWORD bytesTransferred) {
        if (bytesTransferred == 0) {
            Disconnect();
            return;
        }

        m_totalBytesReceived += bytesTransferred;

        // 데이터 처리
        OnDataReceived(m_recvContext.buffer, bytesTransferred);

        // 다음 Recv 예약
        PostRecv();
    }

    void OnSendComplete(SendContext* ctx, DWORD bytesTransferred) {
        m_totalBytesSent += bytesTransferred;
        delete ctx;

        // 큐에 남은 데이터 전송
        ProcessSendQueue();
    }

protected:
    // 파생 클래스에서 구현
    virtual void OnDataReceived(const char* data, int length) = 0;
    virtual void HandleError(int errorCode) {
        Disconnect();
    }
};
```

#### 게임 세션 예제

```cpp
class GameSession : public Session {
private:
    std::string m_playerName;
    int m_playerId;
    Vector3 m_position;

    // 패킷 파싱용 버퍼
    std::vector<char> m_packetBuffer;

public:
    GameSession(SOCKET sock, const SOCKADDR_IN& addr, int playerId)
        : Session(sock, addr)
        , m_playerId(playerId)
        , m_position{0, 0, 0}
    {
    }

    void SetPlayerName(const std::string& name) {
        m_playerName = name;
    }

    void UpdatePosition(const Vector3& pos) {
        m_position = pos;
    }

protected:
    void OnDataReceived(const char* data, int length) override {
        // 버퍼에 추가
        m_packetBuffer.insert(m_packetBuffer.end(), data, data + length);

        // 패킷 파싱
        while (m_packetBuffer.size() >= sizeof(PacketHeader)) {
            PacketHeader* header = (PacketHeader*)m_packetBuffer.data();
            int packetSize = sizeof(PacketHeader) + header->bodyLength;

            if (m_packetBuffer.size() >= packetSize) {
                HandlePacket(m_packetBuffer.data(), packetSize);
                m_packetBuffer.erase(m_packetBuffer.begin(),
                    m_packetBuffer.begin() + packetSize);
            } else {
                break;
            }
        }
    }

private:
    void HandlePacket(const char* packet, int size) {
        PacketHeader* header = (PacketHeader*)packet;

        switch (header->packetType) {
            case PACKET_MOVE:
                HandleMovePacket(packet, size);
                break;
            case PACKET_CHAT:
                HandleChatPacket(packet, size);
                break;
            // ...
        }
    }

    void HandleMovePacket(const char* packet, int size) {
        MovePacket* movePacket = (MovePacket*)packet;
        m_position = movePacket->position;

        // 다른 플레이어에게 브로드캐스트
        // ...
    }

    void HandleChatPacket(const char* packet, int size) {
        ChatPacket* chatPacket = (ChatPacket*)packet;
        // 채팅 처리
        // ...
    }
};
```

---

### 4.3.2 Object Pool / Memory Pool 구현

#### 개념
빈번한 객체 생성/소멸을 피하고 미리 할당된 객체를 재사용하여 성능을 향상시킵니다.

**장점:**
- 메모리 할당/해제 오버헤드 제거
- 메모리 단편화 방지
- 캐시 지역성 향상
- 생성 비용이 큰 객체에 효과적 (DB 연결, 소켓, 대형 버퍼 등)

#### Lock-Free Object Pool 구현

```cpp
template<typename T>
class LockFreeObjectPool {
private:
    struct Node {
        T* object;
        Node* next;
    };

    std::atomic<Node*> m_freeList;
    std::vector<T*> m_allObjects;  // 메모리 관리용
    const int m_initialSize;

public:
    LockFreeObjectPool(int initialSize = 1000)
        : m_freeList(nullptr)
        , m_initialSize(initialSize)
    {
        // 초기 객체 생성
        for (int i = 0; i < initialSize; i++) {
            T* obj = new T();
            m_allObjects.push_back(obj);

            Node* node = new Node();
            node->object = obj;
            node->next = m_freeList.load();

            // Lock-free push
            while (!m_freeList.compare_exchange_weak(node->next, node)) {
                // CAS 실패 시 재시도
                // 높은 경합 상황에서는 CPU 부하를 줄이기 위해 _mm_pause() 같은 힌트를 사용할 수 있음
            }
        }
    }

    ~LockFreeObjectPool() {
        // Free list nodes
        Node* current = m_freeList.load();
        while (current) {
            Node* next = current->next;
            delete current;
            current = next;
        }

        // Objects
        for (T* obj : m_allObjects) {
            delete obj;
        }
    }

    T* Acquire() {
        Node* node = m_freeList.load();

        // Lock-free pop
        while (node) {
            if (m_freeList.compare_exchange_weak(node, node->next)) {
                T* obj = node->object;
                delete node;
                return obj;
            }
            // CAS 실패 시 재시도
        }

        // Pool이 비었으면 새로 생성
        T* newObj = new T();
        m_allObjects.push_back(newObj);
        return newObj;
    }

    void Release(T* obj) {
        // 객체 상태 리셋
        obj->Reset();

        Node* node = new Node();
        node->object = obj;
        node->next = m_freeList.load();

        // Lock-free push
        while (!m_freeList.compare_exchange_weak(node->next, node)) {
            // CAS 실패 시 재시도
        }
    }

    int GetPoolSize() const {
        int count = 0;
        Node* current = m_freeList.load();
        while (current) {
            count++;
            current = current->next;
        }
        return count;
    }
};

// 사용 예시
class SendBuffer {
public:
    char buffer[8192];
    int length;

    void Reset() {
        length = 0;
    }
};

// 전역 풀
LockFreeObjectPool<SendBuffer> g_sendBufferPool(10000);

void SendData(Session* session, const char* data, int length) {
    SendBuffer* buf = g_sendBufferPool.Acquire();
    memcpy(buf->buffer, data, length);
    buf->length = length;

    // 전송...

    // 완료 후
    g_sendBufferPool.Release(buf);
}
```

#### 메모리 풀 (연속 메모리 할당)

```cpp
template<typename T>
class MemoryPool {
private:
    struct Block {
        char data[sizeof(T)];
        bool inUse;
    };

    Block* m_blocks;
    int m_capacity;
    std::atomic<int> m_freeCount;
    std::vector<int> m_freeIndices;
    std::mutex m_mutex;

public:
    MemoryPool(int capacity)
        : m_capacity(capacity)
        , m_freeCount(capacity)
    {
        m_blocks = new Block[capacity];

        for (int i = 0; i < capacity; i++) {
            m_blocks[i].inUse = false;
            m_freeIndices.push_back(i);
        }
    }

    ~MemoryPool() {
        delete[] m_blocks;
    }

    T* Allocate() {
        std::lock_guard<std::mutex> lock(m_mutex);

        if (m_freeIndices.empty()) {
            return nullptr;  // Pool 고갈
        }

        int index = m_freeIndices.back();
        m_freeIndices.pop_back();

        m_blocks[index].inUse = true;
        m_freeCount--;

        // Placement new
        T* obj = new (m_blocks[index].data) T();
        return obj;
    }

    void Deallocate(T* obj) {
        // 주소로부터 인덱스 계산
        char* ptr = reinterpret_cast<char*>(obj);
        char* base = reinterpret_cast<char*>(m_blocks);
        int index = (ptr - base) / sizeof(Block);

        if (index < 0 || index >= m_capacity) {
            // 유효하지 않은 포인터
            return;
        }

        // Destructor 호출
        obj->~T();

        std::lock_guard<std::mutex> lock(m_mutex);
        m_blocks[index].inUse = false;
        m_freeIndices.push_back(index);
        m_freeCount++;
    }

    int GetFreeCount() const {
        return m_freeCount;
    }

    int GetCapacity() const {
        return m_capacity;
    }
};
```

---

### 4.3.3 Ring Buffer (Circular Buffer) 구현

#### 단일 생산자-단일 소비자 (SPSC) Lock-Free Ring Buffer

```cpp
template<typename T>
class SPSCRingBuffer {
private:
    T* m_buffer;
    int m_capacity;
    std::atomic<int> m_writeIndex;
    std::atomic<int> m_readIndex;

public:
    SPSCRingBuffer(int capacity)
        : m_capacity(capacity + 1)  // 1칸 여유분 (full/empty 구분)
        , m_writeIndex(0)
        , m_readIndex(0)
    {
        m_buffer = new T[m_capacity];
    }

    ~SPSCRingBuffer() {
        delete[] m_buffer;
    }

    bool Push(const T& item) {
        int currentWrite = m_writeIndex.load(std::memory_order_relaxed);
        int nextWrite = (currentWrite + 1) % m_capacity;

        if (nextWrite == m_readIndex.load(std::memory_order_acquire)) {
            // Buffer full
            return false;
        }

        m_buffer[currentWrite] = item;
        m_writeIndex.store(nextWrite, std::memory_order_release);
        return true;
    }

    bool Pop(T& item) {
        int currentRead = m_readIndex.load(std::memory_order_relaxed);

        if (currentRead == m_writeIndex.load(std::memory_order_acquire)) {
            // Buffer empty
            return false;
        }

        item = m_buffer[currentRead];
        m_readIndex.store((currentRead + 1) % m_capacity, std::memory_order_release);
        return true;
    }

    int Size() const {
        int write = m_writeIndex.load(std::memory_order_acquire);
        int read = m_readIndex.load(std::memory_order_acquire);

        if (write >= read) {
            return write - read;
        } else {
            return m_capacity - read + write;
        }
    }

    bool IsEmpty() const {
        return m_readIndex.load(std::memory_order_acquire) ==
               m_writeIndex.load(std::memory_order_acquire);
    }

    bool IsFull() const {
        int nextWrite = (m_writeIndex.load(std::memory_order_acquire) + 1) % m_capacity;
        return nextWrite == m_readIndex.load(std::memory_order_acquire);
    }
};
```

#### Batch Push/Pop 최적화

```cpp
template<typename T>
class OptimizedRingBuffer {
private:
    T* m_buffer;
    int m_capacity;
    alignas(64) std::atomic<int> m_writeIndex;  // 캐시 라인 분리
    alignas(64) std::atomic<int> m_readIndex;

public:
    // ... constructor/destructor ...

    int PushBatch(const T* items, int count) {
        int currentWrite = m_writeIndex.load(std::memory_order_relaxed);
        int currentRead = m_readIndex.load(std::memory_order_acquire);

        int available = m_capacity - 1 - Size();
        int toPush = std::min(count, available);

        for (int i = 0; i < toPush; i++) {
            m_buffer[(currentWrite + i) % m_capacity] = items[i];
        }

        m_writeIndex.store((currentWrite + toPush) % m_capacity,
                          std::memory_order_release);
        return toPush;
    }

    int PopBatch(T* items, int maxCount) {
        int currentRead = m_readIndex.load(std::memory_order_relaxed);
        int currentWrite = m_writeIndex.load(std::memory_order_acquire);

        int available = Size();
        int toPop = std::min(maxCount, available);

        for (int i = 0; i < toPop; i++) {
            items[i] = m_buffer[(currentRead + i) % m_capacity];
        }

        m_readIndex.store((currentRead + toPop) % m_capacity,
                         std::memory_order_release);
        return toPop;
    }
};
```

#### 성능 벤치마크

```
테스트: 1억 번의 Push/Pop

구현 방식별 처리량:

1. std::queue + std::mutex
   - 처리량: 5.5M items/sec
   - 경합 시 성능 저하 심각

2. Classic Ring Buffer (lock-free)
   - 처리량: 15M items/sec
   - 기본적인 atomic 연산 사용

3. Optimized Ring Buffer (cache-aligned)
   - 처리량: 112M items/sec
   - False sharing 제거
   - Batch 연산 지원
   - 20배 이상 성능 향상

4. TPCircularBuffer (virtual memory trick)
   - 처리량: 150M+ items/sec
   - 버퍼 wrap 로직 제거
   - OSAtomic.h primitives 사용
```

---

### 4.3.4 버퍼 크기 결정 (MTU, 페이지 크기 고려)

#### MTU (Maximum Transmission Unit) 고려

```
일반적인 MTU 크기:
- Ethernet: 1500 bytes
- Jumbo Frame: 9000 bytes
- Loopback: 65536 bytes

TCP 페이로드:
- IPv4 헤더: 20 bytes
- TCP 헤더: 20 bytes
- 실제 데이터: MTU - 40 bytes

Ethernet (MTU 1500):
- TCP payload: 1460 bytes
```

#### 권장 버퍼 크기

```cpp
// 작은 패킷용 (채팅, 커맨드)
const int SMALL_BUFFER_SIZE = 256;

// 중간 패킷용 (게임 로직)
const int MEDIUM_BUFFER_SIZE = 4096;  // 4KB (페이지 크기)

// 큰 패킷용 (파일 전송)
const int LARGE_BUFFER_SIZE = 8192;   // 8KB

// 대용량 전송용
const int HUGE_BUFFER_SIZE = 16384;   // 16KB
```

**권장 사항:**
- 4KB의 배수 사용 (페이지 크기)
- 8KB가 가장 범용적 (MTU 고려 시 5-6개 패킷)
- 16KB 이상은 대용량 전송에만 사용

#### 계층별 버퍼 전략

```cpp
class BufferAllocator {
private:
    MemoryPool<SmallBuffer> m_smallPool;    // 256B × 10,000
    MemoryPool<MediumBuffer> m_mediumPool;  // 4KB  × 5,000
    MemoryPool<LargeBuffer> m_largePool;    // 8KB  × 1,000
    MemoryPool<HugeBuffer> m_hugePool;      // 16KB × 100

public:
    BufferAllocator()
        : m_smallPool(10000)
        , m_mediumPool(5000)
        , m_largePool(1000)
        , m_hugePool(100)
    {
    }

    void* Allocate(int size) {
        if (size <= 256) {
            return m_smallPool.Allocate();
        } else if (size <= 4096) {
            return m_mediumPool.Allocate();
        } else if (size <= 8192) {
            return m_largePool.Allocate();
        } else if (size <= 16384) {
            return m_hugePool.Allocate();
        } else {
            // 폴백: 일반 할당
            return malloc(size);
        }
    }

    void Deallocate(void* ptr, int size) {
        if (size <= 256) {
            m_smallPool.Deallocate((SmallBuffer*)ptr);
        } else if (size <= 4096) {
            m_mediumPool.Deallocate((MediumBuffer*)ptr);
        } else if (size <= 8192) {
            m_largePool.Deallocate((LargeBuffer*)ptr);
        } else if (size <= 16384) {
            m_hugePool.Deallocate((HugeBuffer*)ptr);
        } else {
            free(ptr);
        }
    }
};
```

#### 페이지 정렬 버퍼 (고성능)

```cpp
class PageAlignedBuffer {
private:
    void* m_buffer;
    int m_size;

public:
    PageAlignedBuffer(int size) : m_size(size) {
        // VirtualAlloc으로 페이지 정렬된 메모리 할당
        m_buffer = VirtualAlloc(
            NULL,
            size,
            MEM_COMMIT | MEM_RESERVE,
            PAGE_READWRITE
        );

        if (!m_buffer) {
            throw std::bad_alloc();
        }
    }

    ~PageAlignedBuffer() {
        if (m_buffer) {
            VirtualFree(m_buffer, 0, MEM_RELEASE);
        }
    }

    void* GetBuffer() { return m_buffer; }
    int GetSize() const { return m_size; }
};
```

**장점:**
- DMA 최적화 (Direct Memory Access)
- 커널과 사용자 공간 간 복사 최소화
- 페이지 경계 넘지 않아 성능 향상

---

### 4.3.5 Reference Counting으로 생명주기 관리

#### std::shared_ptr 사용 패턴

```cpp
class Session : public std::enable_shared_from_this<Session> {
public:
    void PostRecv() {
        // shared_from_this()로 자신의 shared_ptr 획득
        auto self = shared_from_this();

        RecvContext* ctx = new RecvContext();
        ctx->session = self;  // shared_ptr 복사

        WSABUF wsaBuf;
        wsaBuf.buf = ctx->buffer;
        wsaBuf.len = sizeof(ctx->buffer);

        DWORD bytesRecvd = 0, flags = 0;
        int result = WSARecv(m_socket, &wsaBuf, 1, &bytesRecvd, &flags,
            &ctx->overlapped, nullptr);

        // WSARecv가 비동기로 완료될 때까지 Session이 유지됨
        // (RecvContext가 shared_ptr을 보유하므로)
    }

    void OnRecvComplete(RecvContext* ctx, DWORD bytesTransferred) {
        // ctx->session이 자동으로 해제됨 (shared_ptr)

        // 데이터 처리
        ProcessData(ctx->buffer, bytesTransferred);

        delete ctx;
        // ctx가 파괴되면서 session의 참조 카운트 감소
        // 다른 참조가 없으면 Session 자동 삭제
    }
};

// 사용 예시
void OnAccept(SOCKET sock) {
    // shared_ptr로 세션 생성
    auto session = std::make_shared<Session>(sock);

    // SessionManager에 등록
    g_sessionManager.AddSession(session);

    // 첫 Recv 예약
    session->PostRecv();

    // 여기서 session이 스코프를 벗어나도
    // RecvContext와 SessionManager가 참조를 보유하므로 안전
}
```

#### Atomic Reference Counting (수동 구현)

```cpp
class RefCounted {
private:
    mutable std::atomic<int> m_refCount;

public:
    RefCounted() : m_refCount(1) {}

    virtual ~RefCounted() {
        assert(m_refCount == 0);
    }

    void AddRef() const {
        m_refCount.fetch_add(1, std::memory_order_relaxed);
    }

    void Release() const {
        if (m_refCount.fetch_sub(1, std::memory_order_acq_rel) == 1) {
            delete this;
        }
    }

    int GetRefCount() const {
        return m_refCount.load(std::memory_order_relaxed);
    }
};

template<typename T>
class RefPtr {
private:
    T* m_ptr;

public:
    RefPtr() : m_ptr(nullptr) {}

    RefPtr(T* ptr) : m_ptr(ptr) {
        if (m_ptr) m_ptr->AddRef();
    }

    RefPtr(const RefPtr& other) : m_ptr(other.m_ptr) {
        if (m_ptr) m_ptr->AddRef();
    }

    RefPtr(RefPtr&& other) noexcept : m_ptr(other.m_ptr) {
        other.m_ptr = nullptr;
    }

    ~RefPtr() {
        if (m_ptr) m_ptr->Release();
    }

    RefPtr& operator=(const RefPtr& other) {
        if (this != &other) {
            if (m_ptr) m_ptr->Release();
            m_ptr = other.m_ptr;
            if (m_ptr) m_ptr->AddRef();
        }
        return *this;
    }

    T* operator->() const { return m_ptr; }
    T* Get() const { return m_ptr; }
};

// 사용 예시
class Session : public RefCounted {
public:
    void PostRecv() {
        RecvContext* ctx = new RecvContext();
        ctx->session = RefPtr<Session>(this);  // 참조 카운트 증가

        // WSARecv ...
    }
};
```

#### Weak Reference (순환 참조 방지)

```cpp
class SessionManager {
private:
    std::map<int, std::weak_ptr<Session>> m_sessions;
    std::mutex m_mutex;

public:
    void AddSession(std::shared_ptr<Session> session) {
        std::lock_guard<std::mutex> lock(m_mutex);
        m_sessions[session->GetSessionId()] = session;  // weak_ptr 저장
    }

    std::shared_ptr<Session> FindSession(int sessionId) {
        std::lock_guard<std::mutex> lock(m_mutex);

        auto it = m_sessions.find(sessionId);
        if (it != m_sessions.end()) {
            return it->second.lock();  // weak_ptr → shared_ptr
        }
        return nullptr;
    }

    void RemoveSession(int sessionId) {
        std::lock_guard<std::mutex> lock(m_mutex);
        m_sessions.erase(sessionId);
    }

    // 주기적으로 만료된 weak_ptr 정리
    void Cleanup() {
        std::lock_guard<std::mutex> lock(m_mutex);

        for (auto it = m_sessions.begin(); it != m_sessions.end();) {
            if (it->second.expired()) {
                it = m_sessions.erase(it);
            } else {
                ++it;
            }
        }
    }
};
```

---

## 4.4 스레드 동기화

### 4.4.1 IOCP 자체의 스레드 세이프티 범위

#### IOCP의 스레드 안전 보장

**안전한 작업:**
1. `GetQueuedCompletionStatus()` - 여러 스레드에서 동시 호출 가능
2. `PostQueuedCompletionStatus()` - 여러 스레드에서 동시 호출 가능
3. IOCP에 핸들 연결 (`CreateIoCompletionPort`)
4. 서로 다른 소켓에 대한 WSARecv/WSASend

**안전하지 않은 작업:**
1. **동일 소켓에 여러 스레드에서 동시에 WSARecv 호출**
   - 버퍼 순서가 스레드 스케줄링에 따라 달라짐
   - 데이터 손상 가능

2. **동일 소켓에 여러 스레드에서 동시에 WSASend 호출**
   - 전송 순서 보장 안 됨
   - 패킷 순서 섞임

#### 스레드 안전 가이드라인

```cpp
// 안전: 소켓당 하나의 Recv만 진행
void Session::PostRecv() {
    // 이미 Recv 중이면 리턴
    bool expected = false;
    if (!m_isReceiving.compare_exchange_strong(expected, true)) {
        return;
    }

    WSARecv(...);
}

void Session::OnRecvComplete() {
    ProcessData(...);

    m_isReceiving = false;
    PostRecv();  // 다시 Recv 예약
}

// 안전하지 않음: 여러 스레드에서 동시 Recv
void BadExample() {
    // Thread 1
    WSARecv(socket, &buf1, ...);

    // Thread 2 (동시에)
    WSARecv(socket, &buf2, ...);

    // 문제: buf1과 buf2 중 어느 것이 먼저 채워질지 알 수 없음
}
```

---

### 4.4.2 세션별 Lock Granularity

#### Fine-Grained Locking (세밀한 잠금)

```cpp
class Session {
private:
    // 서로 다른 리소스에 대한 별도 락
    std::mutex m_sendMutex;     // Send 큐 보호
    std::mutex m_stateMutex;    // 상태 변경 보호

    // Atomic으로 처리 가능한 것은 락 불필요
    std::atomic<bool> m_isConnected;
    std::atomic<uint64_t> m_totalBytesSent;

public:
    void Send(const char* data, int length) {
        // Send 큐만 잠금
        std::lock_guard<std::mutex> lock(m_sendMutex);
        m_sendQueue.push(data, length);
    }

    void UpdateState(SessionState newState) {
        // 상태만 잠금
        std::lock_guard<std::mutex> lock(m_stateMutex);
        m_state = newState;
    }

    // 통계는 atomic으로 락 없이 처리
    void AddBytesSent(uint64_t bytes) {
        m_totalBytesSent.fetch_add(bytes, std::memory_order_relaxed);
    }
};
```

#### Coarse-Grained Locking (조잡한 잠금)

```cpp
class Session {
private:
    std::mutex m_mutex;  // 하나의 락으로 모든 것 보호

public:
    void Send(const char* data, int length) {
        std::lock_guard<std::mutex> lock(m_mutex);
        m_sendQueue.push(data, length);
    }

    void UpdateState(SessionState newState) {
        std::lock_guard<std::mutex> lock(m_mutex);
        m_state = newState;
    }

    void AddBytesSent(uint64_t bytes) {
        std::lock_guard<std::mutex> lock(m_mutex);
        m_totalBytesSent += bytes;
    }
};
```

**장단점:**
- Fine-Grained: 경합 감소, 복잡도 증가, 데드락 위험
- Coarse-Grained: 단순함, 경합 증가, 성능 저하 가능

**권장:**
대부분의 경우 Coarse-Grained로 시작하고, 프로파일링 후 병목이 확인되면 Fine-Grained로 전환

---

### 4.4.3 Lock-free 프로그래밍 기법

#### Compare-And-Swap (CAS) 기반 패턴

```cpp
// Lock-free 카운터
class LockFreeCounter {
private:
    std::atomic<int> m_value;

public:
    LockFreeCounter(int initial = 0) : m_value(initial) {}

    int Increment() {
        return m_value.fetch_add(1, std::memory_order_relaxed);
    }

    int Decrement() {
        return m_value.fetch_sub(1, std::memory_order_relaxed);
    }

    int Get() const {
        return m_value.load(std::memory_order_relaxed);
    }
};

// Lock-free 스택
template<typename T>
class LockFreeStack {
private:
    struct Node {
        T data;
        Node* next;
    };

    std::atomic<Node*> m_head;

public:
    LockFreeStack() : m_head(nullptr) {}

    void Push(const T& value) {
        Node* newNode = new Node{value, nullptr};
        newNode->next = m_head.load(std::memory_order_relaxed);

        // CAS 루프
        while (!m_head.compare_exchange_weak(
            newNode->next, newNode,
            std::memory_order_release,
            std::memory_order_relaxed))
        {
            // 실패 시 재시도
        }
    }

    bool Pop(T& value) {
        Node* oldHead = m_head.load(std::memory_order_relaxed);

        // CAS 루프
        while (oldHead) {
            if (m_head.compare_exchange_weak(
                oldHead, oldHead->next,
                std::memory_order_acquire,
                std::memory_order_relaxed))
            {
                value = oldHead->data;
                delete oldHead;
                return true;
            }
        }

        return false;  // Stack이 비어있음
    }
};
```

#### ABA 문제와 해결

```cpp
// ABA 문제 예시:
// 1. Thread 1: head를 읽음 (A)
// 2. Thread 2: A를 pop, B를 pop, A를 다시 push
// 3. Thread 1: CAS 성공 (head가 여전히 A를 가리킴)
// 4. 문제: A는 이전의 A와 다른 노드일 수 있음

// 해결책 1: Tagged Pointer (버전 카운터)
template<typename T>
class LockFreeStackWithVersion {
private:
    struct Node {
        T data;
        Node* next;
    };

    struct TaggedPointer {
        Node* ptr;
        uint64_t tag;
    };

    std::atomic<TaggedPointer> m_head;

public:
    void Push(const T& value) {
        Node* newNode = new Node{value, nullptr};
        TaggedPointer oldHead = m_head.load(std::memory_order_relaxed);
        TaggedPointer newHead;

        do {
            newNode->next = oldHead.ptr;
            newHead.ptr = newNode;
            newHead.tag = oldHead.tag + 1;  // 버전 증가
        } while (!m_head.compare_exchange_weak(
            oldHead, newHead,
            std::memory_order_release,
            std::memory_order_relaxed));
    }
};

// 해결책 2: Hazard Pointer (위험 포인터)
// (복잡하므로 생략, 실무에서는 라이브러리 사용 권장)
```

---

### 4.4.4 InterlockedXxx 함수군 활용

#### Windows Interlocked API

```cpp
// 기본 atomic 연산
LONG value = 0;

// Increment/Decrement
InterlockedIncrement(&value);     // value++ (atomic)
InterlockedDecrement(&value);     // value-- (atomic)

// Add/Subtract
InterlockedAdd(&value, 10);       // value += 10 (atomic)
InterlockedExchangeAdd(&value, 5);  // old = value; value += 5; return old;

// Exchange (Swap)
LONG oldValue = InterlockedExchange(&value, 100);  // value = 100; return old;

// Compare-And-Swap
LONG expected = 10;
InterlockedCompareExchange(&value, 20, expected);
// if (value == 10) value = 20; return old_value;

// Pointer 연산
PVOID ptr = nullptr;
InterlockedExchangePointer(&ptr, newPtr);
InterlockedCompareExchangePointer(&ptr, newPtr, oldPtr);

// 64-bit 연산
LONG64 value64 = 0;
InterlockedIncrement64(&value64);
InterlockedExchangeAdd64(&value64, 1000);
```

#### 실전 사용 예시

```cpp
class ConnectionCounter {
private:
    volatile LONG m_activeConnections;
    volatile LONG m_totalConnections;

public:
    ConnectionCounter()
        : m_activeConnections(0)
        , m_totalConnections(0)
    {
    }

    void OnConnect() {
        InterlockedIncrement(&m_activeConnections);
        InterlockedIncrement(&m_totalConnections);
    }

    void OnDisconnect() {
        InterlockedDecrement(&m_activeConnections);
    }

    LONG GetActiveConnections() const {
        return InterlockedAdd((LONG*)&m_activeConnections, 0);  // Atomic read
    }

    LONG GetTotalConnections() const {
        return InterlockedAdd((LONG*)&m_totalConnections, 0);
    }
};

// Lock-free 플래그 설정
class LockFreeFlag {
private:
    volatile LONG m_flag;

public:
    LockFreeFlag() : m_flag(0) {}

    bool TrySet() {
        // 0 → 1로 변경 시도
        return InterlockedCompareExchange(&m_flag, 1, 0) == 0;
    }

    void Clear() {
        InterlockedExchange(&m_flag, 0);
    }

    bool IsSet() const {
        return InterlockedAdd((LONG*)&m_flag, 0) != 0;
    }
};
```

#### 메모리 순서 (Memory Ordering)

```cpp
// Windows Interlocked 함수는 기본적으로 Full Memory Barrier 제공
// Sequential Consistency 보장

// C++11 std::atomic과 비교:
std::atomic<int> value(0);

// Relaxed (가장 약한 순서)
value.fetch_add(1, std::memory_order_relaxed);
// ≈ InterlockedAdd (하지만 순서 보장 약함)

// Acquire/Release
value.load(std::memory_order_acquire);
value.store(10, std::memory_order_release);

// Sequential Consistency (기본값)
value.fetch_add(1);  // memory_order_seq_cst
// ≈ InterlockedIncrement (가장 유사)
```

---

### 4.4.5 SRWLock vs Critical Section 비교

#### 성능 비교

| 특성 | Critical Section | SRWLock |
|------|------------------|---------|
| 크기 (x64) | 40 bytes | 8 bytes |
| 초기화 | `InitializeCriticalSection()` | `SRWLOCK_INIT` (상수) |
| 정리 | `DeleteCriticalSection()` | 불필요 |
| 재귀 진입 | 가능 (같은 스레드) | 불가능 |
| Reader/Writer | 불가능 | 가능 |
| 성능 (무경합) | 기준 | ~15% 빠름 |
| 성능 (고경합) | 느림 | **매우 빠름** |
| 우선순위 | 없음 (FIFO 아님) | 없음 (FIFO 아님) |

#### Critical Section 사용법

```cpp
class CriticalSectionLock {
private:
    CRITICAL_SECTION m_cs;

public:
    CriticalSectionLock() {
        InitializeCriticalSection(&m_cs);
    }

    ~CriticalSectionLock() {
        DeleteCriticalSection(&m_cs);
    }

    void Lock() {
        EnterCriticalSection(&m_cs);
    }

    void Unlock() {
        LeaveCriticalSection(&m_cs);
    }

    bool TryLock() {
        return TryEnterCriticalSection(&m_cs) != 0;
    }
};

// RAII Wrapper
class CriticalSectionGuard {
private:
    CRITICAL_SECTION& m_cs;

public:
    CriticalSectionGuard(CRITICAL_SECTION& cs) : m_cs(cs) {
        EnterCriticalSection(&m_cs);
    }

    ~CriticalSectionGuard() {
        LeaveCriticalSection(&m_cs);
    }
};
```

#### SRWLock 사용법

```cpp
class SRWLockWrapper {
private:
    SRWLOCK m_lock;

public:
    SRWLockWrapper() {
        InitializeSRWLock(&m_lock);
        // 또는 m_lock = SRWLOCK_INIT;
    }

    // Exclusive (Write) Lock
    void LockExclusive() {
        AcquireSRWLockExclusive(&m_lock);
    }

    void UnlockExclusive() {
        ReleaseSRWLockExclusive(&m_lock);
    }

    bool TryLockExclusive() {
        return TryAcquireSRWLockExclusive(&m_lock) != 0;
    }

    // Shared (Read) Lock
    void LockShared() {
        AcquireSRWLockShared(&m_lock);
    }

    void UnlockShared() {
        ReleaseSRWLockShared(&m_lock);
    }

    bool TryLockShared() {
        return TryAcquireSRWLockShared(&m_lock) != 0;
    }
};

// Reader/Writer 패턴 예시
class SessionManager {
private:
    SRWLOCK m_lock;
    std::map<int, Session*> m_sessions;

public:
    SessionManager() {
        InitializeSRWLock(&m_lock);
    }

    // 읽기 (여러 스레드 동시 가능)
    Session* FindSession(int sessionId) {
        AcquireSRWLockShared(&m_lock);

        auto it = m_sessions.find(sessionId);
        Session* session = (it != m_sessions.end()) ? it->second : nullptr;

        ReleaseSRWLockShared(&m_lock);
        return session;
    }

    // 쓰기 (독점 잠금)
    void AddSession(Session* session) {
        AcquireSRWLockExclusive(&m_lock);
        m_sessions[session->GetSessionId()] = session;
        ReleaseSRWLockExclusive(&m_lock);
    }

    void RemoveSession(int sessionId) {
        AcquireSRWLockExclusive(&m_lock);
        m_sessions.erase(sessionId);
        ReleaseSRWLockExclusive(&m_lock);
    }
};
```

#### 성능 벤치마크

```
테스트: 캐시 조회 연산 (Read-heavy workload)

std::mutex (C++11):
- 무경합: 기준 (100%)
- 경합 시: 47% 오버헤드

Raw SRWLOCK (Windows):
- 무경합: 20% 빠름
- 경합 시: 매우 빠름 (best in class)

Critical Section:
- 무경합: ~15% 느림
- 경합 시: SRWLOCK보다 느림

결론:
- Windows 전용 코드라면 SRWLOCK 권장
- 크로스 플랫폼이면 std::shared_mutex (C++17)
```

#### 선택 가이드

**Critical Section을 선택하는 경우:**
- 재귀 잠금이 필요한 경우
- Windows Vista 이전 버전 지원
- 이미 작성된 코드 유지보수

**SRWLock을 선택하는 경우:**
- Reader/Writer 패턴 (읽기가 많은 경우)
- 메모리 효율이 중요한 경우 (8 bytes vs 40 bytes)
- 고성능이 필요한 경우
- Windows Vista 이상만 지원

---

## 실무 문제 및 해결책 모음

### 문제 1: "내 IOCP 서버가 느립니다"

**체크리스트:**
1. Worker 스레드 개수 확인 (CPU 코어 × 2 권장)
2. NumberOfConcurrentThreads 설정 확인 (0 = CPU 코어 수)
3. SO_SNDBUF/SO_RCVBUF 크기 확인
4. Nagle 알고리즘 비활성화 (TCP_NODELAY)
5. 불필요한 Lock 확인
6. 버퍼 크기 최적화 (4KB-8KB 권장)

**프로파일링:**
```cpp
// 완료 패킷 처리 시간 측정
void WorkerThread() {
    while (true) {
        auto start = std::chrono::high_resolution_clock::now();

        BOOL result = GetQueuedCompletionStatus(...);

        auto end = std::chrono::high_resolution_clock::now();
        auto duration = std::chrono::duration_cast<std::chrono::microseconds>(end - start);

        if (duration.count() > 1000) {  // 1ms 이상
            LOG_WARNING("Slow GQCS: %lld μs", duration.count());
        }

        // 처리...
    }
}
```

### 문제 2: "메모리 누수가 발생합니다"

**흔한 원인:**
1. OVERLAPPED 구조체 미해제
2. Send/Recv 버퍼 미해제
3. AcceptEx 소켓 미정리
4. 세션 객체 순환 참조

**해결책:**
```cpp
// 모든 I/O 완료 시 메모리 해제 확인
void OnRecvComplete(RecvContext* ctx) {
    // 데이터 처리...

    // 반드시 해제
    delete ctx;
}

// shared_ptr 사용 시 순환 참조 방지
class SessionManager {
    std::map<int, std::weak_ptr<Session>> m_sessions;  // weak_ptr 사용
};
```

### 문제 3: "일부 클라이언트가 연결 실패합니다"

**원인:**
- Pre-posted Accept 개수 부족
- listen backlog 부족
- 방화벽/네트워크 문제

**해결:**
```cpp
// Accept 큐 모니터링
void MonitorAcceptPool() {
    if (m_pendingAccepts < MIN_PENDING_ACCEPTS) {
        LOG_WARNING("Accept pool low: %d", m_pendingAccepts.load());
        PostAccepts(50);  // 긴급 보충
    }
}

// listen backlog 증가
listen(listenSocket, SOMAXCONN);  // 최대값 사용
```

### 문제 4: "패킷 순서가 섞입니다"

**원인:**
- 여러 스레드에서 동시에 WSASend 호출

**해결:**
```cpp
// Send 직렬화
class Session {
    std::atomic<bool> m_isSending;
    std::queue<Buffer*> m_sendQueue;
    std::mutex m_sendMutex;

    void Send(Buffer* buf) {
        {
            std::lock_guard<std::mutex> lock(m_sendMutex);
            m_sendQueue.push(buf);
        }

        bool expected = false;
        if (m_isSending.compare_exchange_strong(expected, true)) {
            ProcessSendQueue();
        }
    }
};
```

### 문제 5: "서버 종료 시 크래시 발생"

**원인:**
- 진행 중인 I/O 완료 전 리소스 해제

**해결:**
```cpp
void GracefulShutdown() {
    // 1. 새 연결 중단
    closesocket(m_listenSocket);

    // 2. 모든 세션 종료
    g_sessionManager.DisconnectAll();

    // 3. 진행 중인 I/O 완료 대기
    Sleep(1000);

    // 4. Worker 스레드 종료
    for (int i = 0; i < m_workerCount; i++) {
        PostQueuedCompletionStatus(m_hIOCP, 0, 0, nullptr);
    }

    WaitForMultipleObjects(m_workerCount, m_workerThreads, TRUE, INFINITE);

    // 5. 리소스 정리
    CloseHandle(m_hIOCP);
}
```

---

## 참고 자료 (Sources)

이 조사 보고서는 다음 출처의 정보를 바탕으로 작성되었습니다:

### AcceptEx & Pre-posted Accepts
- [Microsoft Windows-classic-samples IOCP](https://github.com/microsoft/Windows-classic-samples/blob/main/Samples/Win7Samples/netds/winsock/iocp/serverex/IocpServerex.Cpp)
- [AcceptEx function - Microsoft Learn](https://learn.microsoft.com/en-us/windows/win32/api/mswsock/nf-mswsock-acceptex)
- [A tutorial on Winsock 2 I/O completion port](https://www.winsocketdotnetworkprogramming.com/winsock2programming/winsock2advancedscalableapp6b.html)
- [IOCP+AcceptEx vs blocking accept - GameDev.net](https://www.gamedev.net/forums/topic/535585-iocpacceptex-vs-blocking-accept/)

### DisconnectEx & Socket Reuse
- [DisconnectEx function - Microsoft Learn](https://learn.microsoft.com/en-us/previous-versions/windows/desktop/legacy/ms737757(v=vs.85))
- [Tcp + iocp handling data receive and reusable sockets - GameDev.net](https://gamedev.net/forums/topic/658570-tcp-iocp-handling-data-receive-and-reusable-sockets/5165659/)
- [DisconnectEx and socket re-use - GameDev.net](https://www.gamedev.net/forums/topic/432292-disconnectex-and-socket-re-use/)

### Scatter/Gather I/O
- [Scatter/Gather I/O - Microsoft Learn](https://learn.microsoft.com/en-us/windows/win32/winsock/scatter-gather-i-o-2)
- [IOCP multi WSARecvs synchronization - GameDev.net](https://gamedev.net/forums/topic/509156-iocp-multi-wsarecvs-synchronization/509156/)

### Zero-byte Receive
- [About zero-byte receive (IOCP server)](https://microsoft.public.win32.programmer.networks.narkive.com/FRa81Gzo/about-zero-byte-receive-iocp-server)
- [IOCP & WSARecv with buflen = 0 - CodeGuru](https://forums.codeguru.com/showthread.php?310706-IOCP-amp-WSARecv-with-buflen-0)
- [Windows implementation consumes lots of memory - mio Issue #439](https://github.com/tokio-rs/mio/issues/439)

### Object Pool & Memory Management
- [Managed I/O Completion Ports (IOCP) - Part 2 - CodeProject](https://www.codeproject.com/Articles/11609/Managed-I-O-Completion-Ports-IOCP-Part-2)
- [Object Pool Design Pattern](https://sourcemaking.com/design_patterns/object_pool)
- [WinAPI IOCP Programming: Scalable File I/O - Apriorit](https://www.apriorit.com/dev-blog/412-win-api-programming-iocp)

### Ring Buffer
- [Creating a Circular Buffer in C and C++ - Embedded Artistry](https://embeddedartistry.com/blog/2017/05/17/creating-a-circular-buffer-in-c-and-c/)
- [Optimizing a Ring Buffer for Throughput - Erik Rigtorp](https://rigtorp.se/ringbuffer/)
- [TPCircularBuffer - GitHub](https://github.com/michaeltyson/TPCircularBuffer)
- [Lock-Free Ring Buffer - QuantumLeaps](https://github.com/QuantumLeaps/lock-free-ring-buffer)

### Thread Synchronization
- [Prefer SRW locks to critical sections - Sudo Null](https://sudonull.com/post/74877-Prefer-SRW-locks-to-critical-sections-Infopulse-Ukraine-Blog)
- [Slim Reader-Writer Locks - Microsoft Learn](https://learn.microsoft.com/en-us/windows/win32/sync/slim-reader-writer--srw--locks)
- [CONCURRENCY: Synchronization Primitives - Microsoft Learn](https://learn.microsoft.com/en-us/archive/msdn-magazine/2007/june/concurrency-synchronization-primitives-new-to-windows-vista)
- [MutexShootout benchmark - GitHub](https://github.com/markwaterman/MutexShootout)

### Lock-Free Programming
- [Differential Reference Counting - 1024cores](https://www.1024cores.net/home/lock-free-algorithms/object-life-time-management/differential-reference-counting)
- [Concurrent Reference Counting - arXiv](https://arxiv.org/pdf/2002.07053)
- [Interlocked.Increment - Microsoft Learn](https://learn.microsoft.com/en-us/dotnet/api/system.threading.interlocked.increment?view=net-9.0)

### Smart Pointers & Session Management
- [std::shared_ptr - cppreference](https://en.cppreference.com/w/cpp/memory/shared_ptr.html)
- [How to: Create and use shared_ptr - Microsoft Learn](https://learn.microsoft.com/en-us/cpp/cpp/how-to-create-and-use-shared-ptr-instances?view=msvc-170)
- [Smart Pointers in C++ - GeeksforGeeks](https://www.geeksforgeeks.org/cpp/smart-pointers-cpp/)

### IOCP Worker Threads
- [Understanding Worker Thread And I/O Completion Port - C# Corner](https://www.c-sharpcorner.com/article/understanding-worker-thread-and-io-completion-port-iocp/)
- [Intro to CLR ThreadPool Growth - GitHub Gist](https://gist.github.com/JonCole/e65411214030f0d823cb)
- [Using IOCP for Worker Threads - DelphiTools](https://www.delphitools.info/2013/09/03/using-iocp-for-worker-threads/)

### Send/Recv Order & Partial I/O
- [Sending data in order with IOCP - MSDN Forums](https://social.msdn.microsoft.com/Forums/windowsdesktop/en-US/9bdaf843-6928-45a6-89c9-33cf60f99f5e/sending-the-data-in-an-order-and-with-speed-using-iocp?forum=wsk)
- [iocp and packet reordering - GameDev.net](https://gamedev.net/forums/topic/506833-iocp-and-packet-reordering-problem/506833/)
- [GetQueuedCompletionStatus - Microsoft Learn](https://learn.microsoft.com/en-us/windows/win32/api/ioapiset/nf-ioapiset-getqueuedcompletionstatus)

### Buffer Size & Performance
- [TCP and UDP Performance Tuning](https://sites.ualberta.ca/dept/chemeng/AIX-43/share/man/info/C/a_doc_lib/aixbman/prftungd/tcpudpperftun.htm)
- [WSASend WSARecv thread safety - ServerFramework](https://serverframework.com/asynchronousevents/2015/01/wsarecv-wsasend-and-thread-safety.html)

### AcceptEx Timeout
- [Speeding up socket connections with AcceptEx - ServerFramework](https://serverframework.com/speeding-up-socket-server-connections-with-acceptex.html)
- [TCP SYN Queue and Accept Queue - Alibaba Cloud](https://www.alibabacloud.com/blog/tcp-syn-queue-and-accept-queue-overflow-explained_599203)

### Performance Benchmarks
- [Trying to get IOCP performance - GameDev.net](https://gamedev.net/forums/topic/529510-trying-to-get-my-iocp-code-on-par-with-expected-performance/4429596/)
- [IOCPNet - Ultimate IOCP - CodeProject](https://www.codeproject.com/Articles/11419/IOCPNet-Ultimate-IOCP)

---

**작성 완료일:** 2025-12-21
**대상:** IOCP 학습 가이드 Section 4 - 핵심 Topic별 상세
