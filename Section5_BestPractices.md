# Section 5: Best Practices & Patterns 상세 분석

> 작성 완료일: 2026-02-01

---

## 목차
- [5.1 대용량 파일 전송 시 메모리 관리](#51-대용량-파일-전송-시-메모리-관리)
- [5.2 느린 클라이언트(Slow Client) 처리](#52-느린-클라이언트slow-client-처리)
- [5.3 DoS 공격 방어 패턴](#53-dos-공격-방어-패턴)
- [5.4 재연결 처리 패턴](#54-재연결-처리-패턴)
- [5.5 Graceful Shutdown 상세 구현](#55-graceful-shutdown-상세-구현)
- [5.6 에러 처리 종합 가이드](#56-에러-처리-종합-가이드)
- [5.7 성능 모니터링과 튜닝](#57-성능-모니터링과-튜닝)
- [5.8 퀴즈용 핵심 Q&A](#58-퀴즈용-핵심-qa)

---

## 5.1 대용량 파일 전송 시 메모리 관리

### 5.1.1 문제 상황

```
10MB 파일을 1,000명에게 동시 전송할 때:
- 순진한 접근: 10MB × 1,000 = 10GB 메모리 필요
- 각 세션 버퍼에 파일 전체 복사 → 메모리 폭발
- 비동기 Send가 완료되기 전까지 버퍼를 유지해야 함
```

### 5.1.2 해결책 1: TransmitFile (Zero-copy)

```cpp
// TransmitFile: 파일 → 소켓 직접 전송 (커널 레벨)
// 유저 모드 메모리 복사 없이 커널 캐시에서 직접 전송

HANDLE hFile = CreateFile(L"largefile.dat", GENERIC_READ,
                          FILE_SHARE_READ, NULL, OPEN_EXISTING,
                          FILE_FLAG_SEQUENTIAL_SCAN, NULL);

OVERLAPPED ov = {};
TRANSMIT_FILE_BUFFERS tfBuf = {};
tfBuf.Head = headerData;       // 파일 앞에 붙일 데이터 (선택)
tfBuf.HeadLength = headerLen;
tfBuf.Tail = trailerData;      // 파일 뒤에 붙일 데이터 (선택)
tfBuf.TailLength = trailerLen;

// 비동기 TransmitFile
TransmitFile(
    clientSocket,
    hFile,
    0,                    // 0 = 전체 파일 전송
    0,                    // 청크 크기 (0 = 기본값)
    &ov,                  // OVERLAPPED
    &tfBuf,               // 헤더/트레일러
    TF_USE_KERNEL_APC     // 커널 APC로 완료 통지
);

// 장점: 유저 모드 버퍼 불필요, 커널 캐시 활용
// 단점: 파일만 가능, 동적 데이터 불가
```

### 5.1.3 해결책 2: 청크 기반 전송

```cpp
class FileTransfer {
    HANDLE hFile;
    LARGE_INTEGER fileSize;
    LARGE_INTEGER offset;
    static const DWORD CHUNK_SIZE = 32768;  // 32KB 단위
    char chunkBuffer[CHUNK_SIZE];

    void StartTransfer(SOCKET sock) {
        offset.QuadPart = 0;
        ReadNextChunk(sock);
    }

    void ReadNextChunk(SOCKET sock) {
        if (offset.QuadPart >= fileSize.QuadPart) {
            // 전송 완료
            OnTransferComplete();
            return;
        }

        DWORD toRead = (DWORD)min((LONGLONG)CHUNK_SIZE,
                                   fileSize.QuadPart - offset.QuadPart);

        // 파일에서 청크 읽기
        DWORD bytesRead;
        OVERLAPPED fileOv = {};
        fileOv.Offset = offset.LowPart;
        fileOv.OffsetHigh = offset.HighPart;
        ReadFile(hFile, chunkBuffer, toRead, &bytesRead, &fileOv);

        // 소켓으로 전송
        WSABUF wsaBuf = { bytesRead, chunkBuffer };
        WSASend(sock, &wsaBuf, 1, NULL, 0, &sendOverlapped, NULL);

        offset.QuadPart += bytesRead;
    }

    // Send 완료 콜백에서 다음 청크 전송
    void OnSendComplete(SOCKET sock) {
        ReadNextChunk(sock);  // 파이프라이닝
    }
};

// 메모리 사용: 세션당 32KB 고정 (파일 크기와 무관)
// 1,000명 × 32KB = 32MB
```

### 5.1.4 해결책 3: 공유 버퍼 + Reference Counting

```cpp
// 같은 데이터를 여러 클라이언트에게 보낼 때 (브로드캐스트)
struct SharedBuffer {
    char* data;
    DWORD length;
    LONG refCount;

    static SharedBuffer* Create(const char* src, DWORD len) {
        auto* buf = new SharedBuffer();
        buf->data = new char[len];
        memcpy(buf->data, src, len);
        buf->length = len;
        buf->refCount = 1;
        return buf;
    }

    void AddRef() { InterlockedIncrement(&refCount); }

    void Release() {
        if (InterlockedDecrement(&refCount) == 0) {
            delete[] data;
            delete this;
        }
    }
};

// 브로드캐스트 전송
void BroadcastFile(SharedBuffer* sharedBuf, vector<Session*>& sessions) {
    for (auto* session : sessions) {
        sharedBuf->AddRef();
        session->SendShared(sharedBuf);
    }
    sharedBuf->Release();  // 초기 참조 해제
}

// Send 완료 시:
void OnSendComplete(SharedBuffer* sharedBuf) {
    sharedBuf->Release();  // 전송 완료 후 참조 해제
}

// 메모리 사용: 10MB × 1 (공유) + 1,000 × OVERLAPPED 크기
// ≈ 10MB + 수십 KB
```

---

## 5.2 느린 클라이언트(Slow Client) 처리

### 5.2.1 문제 정의

```
Slow Client 시나리오:
- 서버가 데이터를 빠르게 생산하지만 클라이언트 수신이 느림
- TCP 윈도우가 0으로 줄어듦 (Zero Window)
- 서버의 Send 버퍼(SO_SNDBUF)가 가득 참
- WSASend가 WSA_IO_PENDING 상태로 오래 대기
- 그동안 서버 메모리에 전송 대기 데이터가 축적

최악의 경우:
- 악의적 클라이언트가 의도적으로 수신을 멈춤
- 서버 메모리 고갈 (Out of Memory)
```

### 5.2.2 해결책: Send Queue + Backpressure

```cpp
class Session {
    // Send 큐: 전송 대기 데이터 관리
    struct SendItem {
        char* data;
        DWORD length;
    };

    queue<SendItem> m_sendQueue;
    LONG m_pendingSendBytes = 0;    // 전송 대기 총 바이트
    bool m_isSending = false;       // 현재 WSASend 진행 중
    CRITICAL_SECTION m_sendLock;

    static const LONG MAX_PENDING_SEND_BYTES = 1024 * 1024;  // 1MB 제한

    bool EnqueueSend(const char* data, DWORD length) {
        EnterCriticalSection(&m_sendLock);

        // Backpressure: 대기 데이터가 너무 많으면 거부
        if (m_pendingSendBytes + length > MAX_PENDING_SEND_BYTES) {
            LeaveCriticalSection(&m_sendLock);
            // 옵션 1: 데이터 드롭
            // 옵션 2: 연결 종료
            // 옵션 3: 생산 속도 조절 통지
            DisconnectSlowClient();
            return false;
        }

        // 큐에 추가
        SendItem item;
        item.data = new char[length];
        memcpy(item.data, data, length);
        item.length = length;
        m_sendQueue.push(item);
        InterlockedAdd(&m_pendingSendBytes, length);

        // 현재 전송 중이 아니면 시작
        if (!m_isSending) {
            m_isSending = true;
            FlushSendQueue();
        }

        LeaveCriticalSection(&m_sendLock);
        return true;
    }

    void FlushSendQueue() {
        if (m_sendQueue.empty()) {
            m_isSending = false;
            return;
        }

        SendItem& item = m_sendQueue.front();
        WSABUF wsaBuf = { item.length, item.data };
        WSASend(m_socket, &wsaBuf, 1, NULL, 0, &m_sendOverlapped, NULL);
    }

    void OnSendComplete(DWORD bytesTransferred) {
        EnterCriticalSection(&m_sendLock);

        SendItem& item = m_sendQueue.front();
        InterlockedAdd(&m_pendingSendBytes, -(LONG)item.length);
        delete[] item.data;
        m_sendQueue.pop();

        FlushSendQueue();  // 다음 아이템 전송

        LeaveCriticalSection(&m_sendLock);
    }
};
```

### 5.2.3 Send 타임아웃 감지

```cpp
class Session {
    DWORD m_lastSendCompleteTime = 0;
    static const DWORD SEND_TIMEOUT_MS = 30000;  // 30초

    void OnSendComplete() {
        m_lastSendCompleteTime = GetTickCount();
    }

    // 주기적 체크 (타이머 스레드에서)
    void CheckSendTimeout() {
        if (m_isSending &&
            GetTickCount() - m_lastSendCompleteTime > SEND_TIMEOUT_MS) {
            // 30초 동안 Send가 완료되지 않음 → Slow Client
            printf("Slow client detected, disconnecting\n");
            Disconnect();
        }
    }
};
```

---

## 5.3 DoS 공격 방어 패턴

### 5.3.1 연결 폭주 (Connection Flood) 방어

```cpp
class ConnectionLimiter {
    // IP별 연결 수 제한
    unordered_map<DWORD, int> m_ipConnectionCount;
    CRITICAL_SECTION m_lock;

    static const int MAX_CONNECTIONS_PER_IP = 10;
    static const int MAX_TOTAL_CONNECTIONS = 10000;
    int m_totalConnections = 0;

    bool AllowConnection(SOCKADDR_IN* pAddr) {
        EnterCriticalSection(&m_lock);

        // 전체 연결 수 제한
        if (m_totalConnections >= MAX_TOTAL_CONNECTIONS) {
            LeaveCriticalSection(&m_lock);
            return false;
        }

        // IP별 연결 수 제한
        DWORD ip = pAddr->sin_addr.s_addr;
        int& count = m_ipConnectionCount[ip];
        if (count >= MAX_CONNECTIONS_PER_IP) {
            LeaveCriticalSection(&m_lock);
            return false;
        }

        count++;
        m_totalConnections++;
        LeaveCriticalSection(&m_lock);
        return true;
    }

    void OnDisconnect(SOCKADDR_IN* pAddr) {
        EnterCriticalSection(&m_lock);
        DWORD ip = pAddr->sin_addr.s_addr;
        m_ipConnectionCount[ip]--;
        if (m_ipConnectionCount[ip] <= 0) {
            m_ipConnectionCount.erase(ip);
        }
        m_totalConnections--;
        LeaveCriticalSection(&m_lock);
    }
};
```

### 5.3.2 AcceptEx 악용 방어

```
공격: AcceptEx에 첫 데이터 수신 옵션 사용 시,
      연결만 하고 데이터를 보내지 않으면 AcceptEx가 무한 대기

해결:
1. AcceptEx의 dwReceiveDataLength = 0 (첫 데이터 수신 안 함)
   → 연결만 되면 바로 완료됨 (권장)

2. SO_CONNECT_TIME으로 AcceptEx 타임아웃 감지
```

```cpp
// 방법 1: 첫 데이터 수신 비활성화 (권장)
g_lpfnAcceptEx(
    listenSocket,
    acceptSocket,
    buffer,
    0,                          // ← 0: 첫 데이터 수신 안 함
    sizeof(SOCKADDR_IN) + 16,
    sizeof(SOCKADDR_IN) + 16,
    &dwBytes,
    &overlapped
);

// 방법 2: SO_CONNECT_TIME으로 대기 시간 확인
void CheckPendingAccepts() {
    for (auto& pendingAccept : m_pendingAccepts) {
        int seconds;
        int optLen = sizeof(seconds);
        getsockopt(pendingAccept.acceptSocket, SOL_SOCKET,
                   SO_CONNECT_TIME, (char*)&seconds, &optLen);

        if (seconds == -1) {
            // 아직 연결되지 않음 → 정상
        } else if (seconds > 30) {
            // 30초 이상 연결된 상태인데 AcceptEx 미완료
            // → 데이터를 보내지 않는 악성 연결
            closesocket(pendingAccept.acceptSocket);
            // 새 AcceptEx 예약
        }
    }
}
```

### 5.3.3 패킷 폭주 (Packet Flood) 방어

```cpp
class PacketRateLimiter {
    struct RateInfo {
        int packetCount;
        DWORD windowStart;
    };

    RateInfo m_rateInfo;
    static const int MAX_PACKETS_PER_SECOND = 100;
    static const DWORD RATE_WINDOW_MS = 1000;

    bool AllowPacket() {
        DWORD now = GetTickCount();

        // 윈도우 리셋
        if (now - m_rateInfo.windowStart >= RATE_WINDOW_MS) {
            m_rateInfo.packetCount = 0;
            m_rateInfo.windowStart = now;
        }

        m_rateInfo.packetCount++;

        if (m_rateInfo.packetCount > MAX_PACKETS_PER_SECOND) {
            return false;  // 속도 제한 초과
        }

        return true;
    }
};
```

### 5.3.4 메모리 소진 공격 방어

```
공격 벡터:
1. 대량 연결 → 세션 메모리 소진
2. 느린 수신 → Send 버퍼 메모리 소진
3. 대량 수신 데이터 → Recv 버퍼 메모리 소진

방어 전략:
- 세션별 메모리 한도 설정 (Send 큐 + Recv 버퍼)
- 전체 메모리 사용량 모니터링
- 한도 초과 시 가장 오래된/비활성 연결 종료
- Object Pool로 메모리 할당 제한
```

```cpp
class MemoryBudget {
    static const size_t MAX_TOTAL_MEMORY = 512 * 1024 * 1024;  // 512MB
    static const size_t MAX_SESSION_MEMORY = 256 * 1024;        // 256KB per session

    LONG64 m_totalAllocated = 0;

    void* AllocateSessionBuffer(size_t size) {
        LONG64 current = InterlockedAdd64(&m_totalAllocated, (LONG64)size);
        if (current > MAX_TOTAL_MEMORY) {
            InterlockedAdd64(&m_totalAllocated, -(LONG64)size);
            return nullptr;  // 글로벌 메모리 한도 초과
        }
        return malloc(size);
    }

    void FreeSessionBuffer(void* ptr, size_t size) {
        free(ptr);
        InterlockedAdd64(&m_totalAllocated, -(LONG64)size);
    }
};
```

---

## 5.4 재연결 처리 패턴

### 5.4.1 클라이언트 식별과 세션 복원

```cpp
// 재연결 시 이전 세션 상태를 복원하는 패턴
class ReconnectionManager {
    struct SavedSession {
        DWORD playerId;
        DWORD lastPacketSequence;
        vector<char> pendingData;     // 미전송 데이터
        DWORD disconnectTime;
        // 게임 상태 (위치, HP 등)
    };

    unordered_map<DWORD, SavedSession> m_savedSessions;
    static const DWORD RECONNECT_TIMEOUT_MS = 60000;  // 1분

    // 연결 종료 시 세션 보존
    void OnDisconnect(Session* pSession) {
        if (pSession->IsAuthenticated()) {
            SavedSession saved;
            saved.playerId = pSession->GetPlayerId();
            saved.lastPacketSequence = pSession->GetLastSequence();
            saved.disconnectTime = GetTickCount();
            // 게임 상태 저장
            m_savedSessions[saved.playerId] = std::move(saved);
        }
    }

    // 재연결 시 세션 복원
    bool TryReconnect(Session* pNewSession, DWORD playerId, DWORD token) {
        auto it = m_savedSessions.find(playerId);
        if (it == m_savedSessions.end()) {
            return false;  // 저장된 세션 없음
        }

        SavedSession& saved = it->second;

        // 타임아웃 확인
        if (GetTickCount() - saved.disconnectTime > RECONNECT_TIMEOUT_MS) {
            m_savedSessions.erase(it);
            return false;
        }

        // 세션 상태 복원
        pNewSession->RestoreState(saved);

        // 미전송 데이터 재전송
        if (!saved.pendingData.empty()) {
            pNewSession->Send(saved.pendingData.data(), saved.pendingData.size());
        }

        m_savedSessions.erase(it);
        return true;
    }

    // 주기적으로 만료된 세션 정리
    void CleanupExpiredSessions() {
        DWORD now = GetTickCount();
        for (auto it = m_savedSessions.begin(); it != m_savedSessions.end(); ) {
            if (now - it->second.disconnectTime > RECONNECT_TIMEOUT_MS) {
                it = m_savedSessions.erase(it);
            } else {
                ++it;
            }
        }
    }
};
```

### 5.4.2 재연결 프로토콜

```
클라이언트 → 서버: RECONNECT 패킷 { playerId, reconnectToken, lastRecvSequence }
    ↓
서버: 토큰 검증 + 저장된 세션 확인
    ↓
서버 → 클라이언트: RECONNECT_ACK { success, lastSendSequence }
    ↓
서버: lastRecvSequence 이후 데이터 재전송
클라이언트: lastSendSequence 이후 데이터 재전송
    ↓
정상 통신 재개
```

---

## 5.5 Graceful Shutdown 상세 구현

### 5.5.1 서버 종료 순서

```cpp
class Server {
    enum ShutdownState {
        RUNNING,
        STOPPING_NEW_CONNECTIONS,
        DRAINING_SESSIONS,
        STOPPING_WORKERS,
        CLEANUP
    };

    ShutdownState m_state = RUNNING;
    LONG m_activeSessionCount = 0;
    HANDLE m_shutdownEvent = CreateEvent(NULL, TRUE, FALSE, NULL);

    void Shutdown() {
        // Phase 1: 새 연결 차단
        m_state = STOPPING_NEW_CONNECTIONS;
        closesocket(g_listenSocket);  // AcceptEx 즉시 실패 → 새 연결 차단
        printf("Phase 1: Stopped accepting new connections\n");

        // Phase 2: 기존 클라이언트에 종료 통지 + 대기
        m_state = DRAINING_SESSIONS;
        BroadcastShutdownNotice();

        // 모든 세션 종료 대기 (최대 30초)
        DWORD waitStart = GetTickCount();
        while (m_activeSessionCount > 0) {
            if (GetTickCount() - waitStart > 30000) {
                printf("Drain timeout, forcing disconnect of %d sessions\n",
                       m_activeSessionCount);
                ForceDisconnectAll();
                break;
            }
            Sleep(100);
        }
        printf("Phase 2: All sessions drained\n");

        // Phase 3: Worker 스레드 종료
        m_state = STOPPING_WORKERS;
        for (int i = 0; i < m_workerCount; i++) {
            PostQueuedCompletionStatus(g_hIOCP, 0, (ULONG_PTR)-1, NULL);
        }
        WaitForMultipleObjects(m_workerCount, m_workerHandles, TRUE, 10000);
        printf("Phase 3: Worker threads stopped\n");

        // Phase 4: 리소스 정리
        m_state = CLEANUP;
        CloseHandle(g_hIOCP);
        WSACleanup();
        printf("Phase 4: Cleanup complete\n");
    }

    void BroadcastShutdownNotice() {
        // 모든 세션에 종료 예정 패킷 전송
        // 클라이언트가 이 패킷을 받으면 저장 후 연결 종료
        ShutdownPacket pkt;
        pkt.reason = SHUTDOWN_MAINTENANCE;
        pkt.reconnectAfterSec = 300;  // 5분 후 재접속 가능

        for (auto& session : m_sessions) {
            if (session.IsActive()) {
                session.Send(&pkt, sizeof(pkt));
            }
        }
    }

    void ForceDisconnectAll() {
        for (auto& session : m_sessions) {
            if (session.IsActive()) {
                // shutdown(SD_BOTH)로 정상적 TCP 종료 시도
                shutdown(session.GetSocket(), SD_BOTH);
                // 2초 후에도 안 닫히면 강제 종료
            }
        }
        Sleep(2000);
        for (auto& session : m_sessions) {
            if (session.IsActive()) {
                closesocket(session.GetSocket());
            }
        }
    }
};
```

### 5.5.2 CancelIoEx를 이용한 진행 중 I/O 취소

```cpp
// 특정 소켓의 모든 진행 중 I/O를 취소
CancelIoEx((HANDLE)clientSocket, NULL);  // NULL = 모든 I/O 취소

// 특정 OVERLAPPED의 I/O만 취소
CancelIoEx((HANDLE)clientSocket, &specificOverlapped);

// 취소된 I/O는 Worker에서 ERROR_OPERATION_ABORTED로 완료됨
// → Worker에서 이 에러를 정상적으로 처리해야 함

// 주의: CancelIoEx를 호출한 스레드가 아닌,
// I/O를 시작한 스레드와 무관하게 취소 가능 (Vista+)
// XP에서는 CancelIo만 가능 (호출 스레드의 I/O만 취소)
```

---

## 5.6 에러 처리 종합 가이드

### 5.6.1 GetQueuedCompletionStatus 반환값 완전 분석

```cpp
BOOL result = GetQueuedCompletionStatus(
    hIOCP, &bytes, &key, &pOv, INFINITE
);

if (result) {
    // ═══════════════════════════════════════════
    // Case 1: result=TRUE, pOv!=NULL
    // ═══════════════════════════════════════════
    if (bytes > 0) {
        // 정상 I/O 완료
        ProcessCompletion(key, pOv, bytes);
    } else {
        // bytes==0: TCP 연결 정상 종료 (Graceful close)
        // 상대방이 shutdown(SD_SEND) 또는 closesocket 호출
        if (((OverlappedEx*)pOv)->opType == OP_RECV) {
            HandleGracefulClose(key, pOv);
        } else if (((OverlappedEx*)pOv)->opType == OP_ACCEPT) {
            // AcceptEx는 bytes=0이 정상 (dwReceiveDataLength=0일 때)
            HandleAcceptComplete(key, pOv);
        }
    }
} else {
    DWORD err = GetLastError();

    if (pOv == NULL) {
        // ═══════════════════════════════════════════
        // Case 2: result=FALSE, pOv==NULL
        // ═══════════════════════════════════════════
        if (err == WAIT_TIMEOUT) {
            // dwMilliseconds 타임아웃 만료
            // → 정상: 주기적 작업 처리
        } else {
            // IOCP 핸들 자체 에러 (매우 드묾)
            // err == ERROR_ABANDONED_WAIT_0: IOCP가 닫힘
            printf("IOCP error: %d\n", err);
        }
    } else {
        // ═══════════════════════════════════════════
        // Case 3: result=FALSE, pOv!=NULL
        // ═══════════════════════════════════════════
        // 해당 I/O 작업이 에러와 함께 완료됨
        switch (err) {
        case ERROR_NETNAME_DELETED:       // 64
            // 원격 호스트가 연결을 끊음 (RST)
            HandleConnectionReset(key, pOv);
            break;

        case ERROR_CONNECTION_ABORTED:    // 1236
            // 로컬에서 연결 중단 (예: closesocket 호출 후)
            HandleConnectionAborted(key, pOv);
            break;

        case ERROR_OPERATION_ABORTED:     // 995
            // CancelIoEx 또는 closesocket으로 I/O 취소됨
            HandleOperationCancelled(key, pOv);
            break;

        case ERROR_SEM_TIMEOUT:           // 121
            // 네트워크 타임아웃 (TCP keepalive 실패 등)
            HandleNetworkTimeout(key, pOv);
            break;

        case ERROR_MORE_DATA:             // 234
            // 버퍼 부족 (파이프에서 주로 발생)
            HandleBufferTooSmall(key, pOv, bytes);
            break;

        default:
            printf("Unexpected I/O error: %d\n", err);
            CleanupSession(key, pOv);
            break;
        }
    }
}
```

### 5.6.2 WSARecv/WSASend 호출 시 에러 처리

```cpp
int result = WSARecv(sock, &wsaBuf, 1, NULL, &flags, &ov, NULL);

if (result == 0) {
    // 즉시 완료됨
    // FILE_SKIP_COMPLETION_PORT_ON_SUCCESS가 설정되어 있으면
    // IOCP에 패킷이 안 감 → 여기서 직접 처리
    // 미설정이면 IOCP에도 패킷이 감 → Worker에서 처리
}
else if (result == SOCKET_ERROR) {
    DWORD err = WSAGetLastError();
    if (err == WSA_IO_PENDING) {
        // 정상: 비동기 I/O 진행 중
        // Worker에서 완료 통지 받음
    }
    else if (err == WSAECONNRESET) {    // 10054
        // 연결 리셋됨
        CleanupSession(pSession);
    }
    else if (err == WSAENOBUFS) {       // 10055
        // 버퍼 부족 (Non-paged pool 고갈)
        // → 동시 I/O 수 줄이기, Zero-byte Recv 사용
        RetryWithSmallerBuffer(pSession);
    }
    else {
        printf("WSARecv error: %d\n", err);
        CleanupSession(pSession);
    }
}
```

---

## 5.7 성능 모니터링과 튜닝

### 5.7.1 핵심 성능 지표

```cpp
struct ServerMetrics {
    // 연결 관련
    LONG activeConnections;
    LONG connectionsPerSecond;
    LONG disconnectionsPerSecond;

    // I/O 관련
    LONG64 totalBytesRecv;
    LONG64 totalBytesSend;
    LONG recvOpsPerSecond;
    LONG sendOpsPerSecond;

    // 레이턴시
    LONG avgProcessingTimeUs;   // 평균 패킷 처리 시간 (마이크로초)
    LONG maxProcessingTimeUs;

    // 리소스
    LONG pendingSendBytes;      // 전송 대기 바이트
    LONG pendingAccepts;        // 대기 중인 AcceptEx
    SIZE_T memoryUsageBytes;    // 총 메모리 사용량
    LONG workerThreadBusyCount; // 작업 중인 Worker 수

    void Print() {
        printf("=== Server Metrics ===\n");
        printf("Connections: %d (rate: +%d/s -%d/s)\n",
               activeConnections, connectionsPerSecond, disconnectionsPerSecond);
        printf("Throughput: Recv %lld B/s, Send %lld B/s\n",
               totalBytesRecv, totalBytesSend);
        printf("Ops/sec: Recv %d, Send %d\n",
               recvOpsPerSecond, sendOpsPerSecond);
        printf("Latency: avg %d us, max %d us\n",
               avgProcessingTimeUs, maxProcessingTimeUs);
        printf("Pending: Send %d bytes, Accept %d\n",
               pendingSendBytes, pendingAccepts);
        printf("Memory: %.2f MB\n", memoryUsageBytes / (1024.0 * 1024.0));
    }

    void Reset() {
        connectionsPerSecond = 0;
        disconnectionsPerSecond = 0;
        totalBytesRecv = 0;
        totalBytesSend = 0;
        recvOpsPerSecond = 0;
        sendOpsPerSecond = 0;
        maxProcessingTimeUs = 0;
    }
};
```

### 5.7.2 병목 진단 가이드

```
증상 → 원인 → 해결

1. CPU 사용률 100%
   → Worker에서 무거운 처리 (직렬화, 암호화, 로직)
   → Worker에서 I/O만 처리하고 무거운 작업은 별도 스레드 풀로 위임

2. 메모리 지속 증가
   → Send 큐에 데이터 축적 (Slow Client)
   → Backpressure + Send 타임아웃 적용

3. 연결은 되는데 응답 없음
   → Worker 스레드 전부 블로킹 (DB 호출, Lock 대기)
   → Worker에서 블로킹 호출 제거, 비동기 DB 사용

4. 새 연결 수락 불가
   → AcceptEx 풀 소진
   → 동적 AcceptEx 보충 + 모니터링

5. WSAENOBUFS 에러 빈발
   → Non-paged pool 고갈 (동시 I/O 과다)
   → Zero-byte Recv 사용, 동시 I/O 수 제한
```

---

## 5.8 퀴즈용 핵심 Q&A

### 정의형 문제

**Q1**: Backpressure란 무엇이며 IOCP 서버에서 왜 필요한가?
> **A**: Backpressure는 데이터 생산 속도가 소비 속도를 초과할 때 생산 속도를 조절하는 메커니즘이다. IOCP 서버에서 Send 대기 데이터가 과도하게 축적되면 메모리가 고갈되므로, 세션별 Send 큐 크기에 상한을 두고 초과 시 데이터 드롭 또는 연결 종료로 대응한다.

**Q2**: `CancelIoEx`와 `CancelIo`의 차이는?
> **A**: `CancelIo`(XP)는 호출 스레드가 시작한 I/O만 취소할 수 있다. `CancelIoEx`(Vista+)는 어떤 스레드에서든 특정 핸들의 I/O를 취소할 수 있으며, 특정 OVERLAPPED만 지정하여 취소도 가능하다.

### 수치형 문제

**Q3**: 1만 명 동시접속 서버에서 세션당 Send 버퍼 256KB, Recv 버퍼 4KB를 할당하면 총 메모리는?
> **A**: Send: 10,000 × 256KB = 2.5GB, Recv: 10,000 × 4KB = 40MB. 총 약 2.54GB. 추가로 세션 객체, OVERLAPPED 구조체 등을 합하면 3GB 이상이 필요하다.

### 적용형 문제

**Q4**: 서버 종료 시 `closesocket`만 호출하면 어떤 문제가 발생하는가?
> **A**: 진행 중인 비동기 I/O가 즉시 에러로 완료되면서 Worker 스레드가 동시에 대량의 에러를 처리하게 된다. 세션 정리 로직이 멀티스레드에서 동시 호출되어 경합 조건(race condition)이 발생할 수 있다. 또한 클라이언트 측에서 예기치 않은 연결 리셋(RST)을 받아 저장되지 않은 데이터가 유실될 수 있다.

**Q5**: AcceptEx에서 `dwReceiveDataLength`를 0이 아닌 값으로 설정하면 어떤 보안 위험이 있는가?
> **A**: 공격자가 연결만 맺고 데이터를 보내지 않으면 AcceptEx가 무한히 대기 상태에 머무른다. 이를 반복하면 모든 Pre-posted Accept이 소진되어 정상 클라이언트의 연결이 불가능해진다. 이를 방지하려면 `dwReceiveDataLength = 0`으로 설정하거나, `SO_CONNECT_TIME`으로 주기적으로 대기 시간을 확인해야 한다.

---

## 참고 자료

- Microsoft Docs: [TransmitFile function](https://learn.microsoft.com/en-us/windows/win32/api/mswsock/nf-mswsock-transmitfile)
- Microsoft Docs: [CancelIoEx function](https://learn.microsoft.com/en-us/windows/win32/fileio/cancelioex-func)
- Microsoft Docs: [SO_CONNECT_TIME](https://learn.microsoft.com/en-us/windows/win32/winsock/sol-socket-socket-options)
- Len Holgate, "Practical IOCP" (serverframework.com)
- Windows Network Programming (Anthony Jones, Jim Ohlund)
- OWASP: Application Denial of Service
