# Section 11: C++ IOCP Wrapper Library & Framework 소개

> 작성 완료일: 2026-02-01

---

## 목차
- [11.1 Boost.Asio](#111-boostasio)
- [11.2 libuv](#112-libuv)
- [11.3 Windows Thread Pool (네이티브)](#113-windows-thread-pool-네이티브)
- [11.4 EasyIOCP / 경량 래퍼 예시](#114-easyiocp--경량-래퍼-예시)
- [11.5 게임 서버 프레임워크](#115-게임-서버-프레임워크)
- [11.6 라이브러리 선택 가이드](#116-라이브러리-선택-가이드)
- [11.7 직접 구현 vs 라이브러리 사용](#117-직접-구현-vs-라이브러리-사용)

---

## 11.1 Boost.Asio

### 11.1.1 개요

Boost.Asio는 C++ 네트워크/비동기 I/O의 사실상 표준 라이브러리이다. Windows에서는 내부적으로 IOCP를 사용하며, Linux에서는 epoll, macOS에서는 kqueue를 백엔드로 사용한다. C++20 Networking TS의 참조 구현이기도 하다.

```
Boost.Asio 특성:
- 크로스 플랫폼 (Windows/Linux/macOS)
- Windows에서 자동으로 IOCP 사용
- Proactor 패턴 기반 (IOCP와 동일한 모델)
- C++11 이상, 코루틴 지원 (C++20)
- 헤더 온리 사용 가능 (Boost 없이 standalone)
- 활발한 유지보수 (Chris Kohlhoff)
```

### 11.1.2 기본 Echo 서버 코드

```cpp
// Boost.Asio Echo 서버 (C++17)
#include <boost/asio.hpp>
#include <iostream>
#include <memory>

using boost::asio::ip::tcp;

class Session : public std::enable_shared_from_this<Session> {
    tcp::socket socket_;
    char data_[4096];

public:
    Session(tcp::socket socket) : socket_(std::move(socket)) {}

    void Start() { DoRead(); }

private:
    void DoRead() {
        auto self = shared_from_this();
        socket_.async_read_some(
            boost::asio::buffer(data_, sizeof(data_)),
            [this, self](boost::system::error_code ec, std::size_t length) {
                if (!ec) {
                    DoWrite(length);
                }
            }
        );
    }

    void DoWrite(std::size_t length) {
        auto self = shared_from_this();
        boost::asio::async_write(
            socket_,
            boost::asio::buffer(data_, length),
            [this, self](boost::system::error_code ec, std::size_t /*length*/) {
                if (!ec) {
                    DoRead();
                }
            }
        );
    }
};

class Server {
    tcp::acceptor acceptor_;

public:
    Server(boost::asio::io_context& io, short port)
        : acceptor_(io, tcp::endpoint(tcp::v4(), port)) {
        DoAccept();
    }

private:
    void DoAccept() {
        acceptor_.async_accept(
            [this](boost::system::error_code ec, tcp::socket socket) {
                if (!ec) {
                    std::make_shared<Session>(std::move(socket))->Start();
                }
                DoAccept();  // 다음 Accept
            }
        );
    }
};

int main() {
    boost::asio::io_context io;
    Server server(io, 9000);

    // Worker 스레드 (io_context::run이 내부적으로 IOCP 사용)
    std::vector<std::thread> threads;
    int threadCount = std::thread::hardware_concurrency() * 2;
    for (int i = 0; i < threadCount; i++) {
        threads.emplace_back([&io]() { io.run(); });
    }

    for (auto& t : threads) t.join();
    return 0;
}
```

### 11.1.3 C++20 코루틴 버전

```cpp
// Boost.Asio + C++20 Coroutines (훨씬 간결)
#include <boost/asio.hpp>
#include <boost/asio/co_spawn.hpp>
#include <boost/asio/detached.hpp>

using boost::asio::ip::tcp;
using boost::asio::awaitable;
using boost::asio::co_spawn;
using boost::asio::detached;
using boost::asio::use_awaitable;

awaitable<void> Echo(tcp::socket socket) {
    try {
        char data[4096];
        for (;;) {
            std::size_t n = co_await socket.async_read_some(
                boost::asio::buffer(data), use_awaitable);
            co_await async_write(socket,
                boost::asio::buffer(data, n), use_awaitable);
        }
    } catch (std::exception& e) {
        // 연결 종료
    }
}

awaitable<void> Listener(tcp::acceptor acceptor) {
    for (;;) {
        tcp::socket socket = co_await acceptor.async_accept(use_awaitable);
        co_spawn(acceptor.get_executor(), Echo(std::move(socket)), detached);
    }
}

int main() {
    boost::asio::io_context io;
    co_spawn(io, Listener(tcp::acceptor(io, {tcp::v4(), 9000})), detached);
    io.run();
}

// 코루틴 버전 장점:
// - 콜백 지옥(callback hell) 제거
// - 동기 코드처럼 읽히는 비동기 로직
// - 에러 처리가 try/catch로 자연스러움
// - 내부적으로 여전히 IOCP 사용 (성능 동일)
```

### 11.1.4 Asio가 IOCP를 사용하는 방식

```
Asio 내부 (Windows):

io_context::run()
  → win_iocp_io_context::do_one()
    → GetQueuedCompletionStatusEx()  // 배치 수신
    → 완료 패킷별로 핸들러(콜백) 실행

async_read_some()
  → win_iocp_socket_service::start_receive_op()
    → WSARecv() + OVERLAPPED
    → 완료 시 io_context의 IOCP에 통지

acceptor::async_accept()
  → win_iocp_socket_service::start_accept_op()
    → AcceptEx() + OVERLAPPED

핵심: Asio는 IOCP를 추상화하지만, 1:1로 매핑됨
      → 성능 오버헤드 거의 없음
```

### 11.1.5 Standalone Asio (Boost 없이)

```cpp
// Boost 전체를 설치하지 않고 Asio만 사용
// https://think-async.com/Asio/

// #define ASIO_STANDALONE  // 이것만 정의
#include <asio.hpp>        // boost/ 접두사 없음

// 나머지 코드 동일, namespace만 변경:
// boost::asio → asio
// boost::system::error_code → asio::error_code

// CMakeLists.txt:
// find_package(asio) 또는 헤더 경로 직접 지정
// target_link_libraries(... ws2_32 mswsock)  # Windows
```

---

## 11.2 libuv

### 11.2.1 개요

libuv는 Node.js의 I/O 엔진으로, C 라이브러리이다. Windows에서 IOCP, Linux에서 epoll, macOS에서 kqueue를 추상화한다.

```
libuv 특성:
- C 라이브러리 (C++에서도 사용 가능)
- Node.js에서 검증된 안정성
- 이벤트 루프 기반 (단일 스레드 모델)
- 네트워크, 파일 I/O, DNS, 타이머, 프로세스 관리 통합
- 크로스 플랫폼
- Proactor(Windows) / Reactor(Linux) 차이를 내부에서 통일
```

### 11.2.2 기본 TCP 서버

```c
#include <uv.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

uv_loop_t* loop;

typedef struct {
    uv_write_t req;
    uv_buf_t buf;
} write_req_t;

void alloc_buffer(uv_handle_t* handle, size_t suggested_size, uv_buf_t* buf) {
    buf->base = (char*)malloc(suggested_size);
    buf->len = suggested_size;
}

void on_write(uv_write_t* req, int status) {
    write_req_t* wr = (write_req_t*)req;
    free(wr->buf.base);
    free(wr);
}

void on_read(uv_stream_t* client, ssize_t nread, const uv_buf_t* buf) {
    if (nread < 0) {
        uv_close((uv_handle_t*)client, NULL);
        free(buf->base);
        return;
    }

    // Echo
    write_req_t* req = (write_req_t*)malloc(sizeof(write_req_t));
    req->buf = uv_buf_init(buf->base, nread);
    uv_write(&req->req, client, &req->buf, 1, on_write);
}

void on_new_connection(uv_stream_t* server, int status) {
    if (status < 0) return;

    uv_tcp_t* client = (uv_tcp_t*)malloc(sizeof(uv_tcp_t));
    uv_tcp_init(loop, client);

    if (uv_accept(server, (uv_stream_t*)client) == 0) {
        uv_read_start((uv_stream_t*)client, alloc_buffer, on_read);
    } else {
        uv_close((uv_handle_t*)client, NULL);
    }
}

int main() {
    loop = uv_default_loop();

    uv_tcp_t server;
    uv_tcp_init(loop, &server);

    struct sockaddr_in addr;
    uv_ip4_addr("0.0.0.0", 9000, &addr);
    uv_tcp_bind(&server, (const struct sockaddr*)&addr, 0);

    uv_listen((uv_stream_t*)&server, 128, on_new_connection);

    printf("Server listening on port 9000\n");
    return uv_run(loop, UV_RUN_DEFAULT);  // 내부적으로 IOCP 사용
}
```

### 11.2.3 Asio vs libuv 비교

| 항목 | Boost.Asio | libuv |
|------|-----------|-------|
| 언어 | C++ | C (C++ 래퍼 별도) |
| 스레딩 모델 | 멀티스레드 io_context::run | 단일 스레드 이벤트 루프 |
| API 스타일 | OOP + 콜백/코루틴 | C 콜백 |
| IOCP 활용 | 멀티 Worker 스레드 | 단일 스레드 폴링 |
| 파일 I/O | 제한적 | 네이티브 (스레드풀) |
| DNS | 동기/비동기 | 비동기 (c-ares) |
| 프로세스 관리 | 없음 | 있음 |
| 코루틴 | C++20 co_await | 없음 |
| 성능 (IOCP) | 멀티코어 활용 | 단일 코어 |
| 적합 용도 | 고성능 서버 | Node.js 스타일 앱 |

---

## 11.3 Windows Thread Pool (네이티브)

### 11.3.1 개요

Windows Vista+에서 제공하는 네이티브 Thread Pool API는 내부적으로 IOCP를 사용하며, 별도 라이브러리 없이 Windows SDK만으로 사용할 수 있다.

```
Thread Pool I/O 특성:
- Windows SDK 기본 제공 (추가 의존성 없음)
- IOCP를 내부적으로 관리 (직접 관리 불필요)
- 콜백 기반 API
- 스레드 풀 크기 자동 관리
- 정리 그룹(Cleanup Group)으로 안전한 종료
```

### 11.3.2 Echo 서버 구현

```cpp
#include <windows.h>
#include <winsock2.h>
#include <ws2tcpip.h>
#include <stdio.h>

#pragma comment(lib, "ws2_32.lib")

struct Session {
    SOCKET socket;
    PTP_IO ptpIo;
    OVERLAPPED recvOv;
    OVERLAPPED sendOv;
    WSABUF wsaBuf;
    char buffer[4096];
    enum { OP_RECV, OP_SEND } pendingOp;
};

// I/O 완료 콜백 (Thread Pool이 호출)
VOID CALLBACK IoCallback(
    PTP_CALLBACK_INSTANCE Instance,
    PVOID Context,
    PVOID Overlapped,
    ULONG IoResult,
    ULONG_PTR BytesTransferred,
    PTP_IO Io
) {
    Session* session = (Session*)Context;

    if (IoResult != ERROR_SUCCESS || BytesTransferred == 0) {
        closesocket(session->socket);
        CloseThreadpoolIo(session->ptpIo);
        delete session;
        return;
    }

    if (Overlapped == &session->recvOv) {
        // Recv 완료 → Send
        session->wsaBuf.len = (ULONG)BytesTransferred;
        session->pendingOp = Session::OP_SEND;
        StartThreadpoolIo(session->ptpIo);  // Send 전 필수!
        WSASend(session->socket, &session->wsaBuf, 1, NULL, 0,
                &session->sendOv, NULL);
    }
    else if (Overlapped == &session->sendOv) {
        // Send 완료 → Recv
        session->wsaBuf.buf = session->buffer;
        session->wsaBuf.len = sizeof(session->buffer);
        session->pendingOp = Session::OP_RECV;
        DWORD flags = 0;
        StartThreadpoolIo(session->ptpIo);  // Recv 전 필수!
        WSARecv(session->socket, &session->wsaBuf, 1, NULL, &flags,
                &session->recvOv, NULL);
    }
}

// Accept 후 세션 생성
void OnAccept(SOCKET clientSocket) {
    Session* session = new Session();
    session->socket = clientSocket;

    // Thread Pool I/O 객체 생성
    session->ptpIo = CreateThreadpoolIo(
        (HANDLE)clientSocket,
        IoCallback,           // 콜백
        session,              // 컨텍스트
        NULL                  // 기본 스레드 풀
    );

    // 첫 Recv 시작
    ZeroMemory(&session->recvOv, sizeof(OVERLAPPED));
    session->wsaBuf.buf = session->buffer;
    session->wsaBuf.len = sizeof(session->buffer);
    DWORD flags = 0;

    StartThreadpoolIo(session->ptpIo);  // I/O 전 반드시 호출!
    WSARecv(session->socket, &session->wsaBuf, 1, NULL, &flags,
            &session->recvOv, NULL);
}

// 장점: IOCP 직접 관리 코드 없음 (CreateIoCompletionPort, GQCS 불필요)
// 단점: 세밀한 스레드 제어 어려움, StartThreadpoolIo 호출 누락 시 버그
```

---

## 11.4 EasyIOCP / 경량 래퍼 예시

### 11.4.1 직접 만드는 경량 IOCP 래퍼

대규모 프레임워크 대신, IOCP의 핵심만 감싸는 경량 래퍼를 직접 만드는 것도 좋은 학습 방법이다.

```cpp
// ===== iocp_core.h =====
#pragma once
#define _WIN32_WINNT 0x0A00
#define WIN32_LEAN_AND_MEAN
#define NOMINMAX
#include <winsock2.h>
#include <ws2tcpip.h>
#include <mswsock.h>
#include <windows.h>
#include <functional>
#include <thread>
#include <vector>

#pragma comment(lib, "ws2_32.lib")
#pragma comment(lib, "mswsock.lib")

// 작업 타입
enum class IoType { Accept, Recv, Send, Disconnect };

// 확장 OVERLAPPED
struct IoContext {
    OVERLAPPED overlapped = {};
    IoType type;
    WSABUF wsaBuf;
    char buffer[8192];
    SOCKET acceptSocket = INVALID_SOCKET;  // Accept용

    void Reset() { ZeroMemory(&overlapped, sizeof(OVERLAPPED)); }
};

// 세션 인터페이스
class ISession {
public:
    virtual ~ISession() = default;
    virtual void OnConnected() = 0;
    virtual void OnRecv(const char* data, int len) = 0;
    virtual void OnSendComplete(int len) = 0;
    virtual void OnDisconnected() = 0;
};

// IOCP 코어
class IocpCore {
    HANDLE m_hIOCP = NULL;
    std::vector<std::thread> m_workers;

public:
    bool Initialize(int workerCount = 0) {
        WSADATA wsaData;
        WSAStartup(MAKEWORD(2, 2), &wsaData);

        m_hIOCP = CreateIoCompletionPort(INVALID_HANDLE_VALUE, NULL, 0, 0);
        if (!m_hIOCP) return false;

        if (workerCount == 0) {
            SYSTEM_INFO si;
            GetSystemInfo(&si);
            workerCount = si.dwNumberOfProcessors * 2;
        }

        for (int i = 0; i < workerCount; i++) {
            m_workers.emplace_back(&IocpCore::WorkerThread, this);
        }
        return true;
    }

    bool Associate(SOCKET socket, ULONG_PTR key) {
        return CreateIoCompletionPort(
            (HANDLE)socket, m_hIOCP, key, 0) != NULL;
    }

    void Shutdown() {
        for (size_t i = 0; i < m_workers.size(); i++) {
            PostQueuedCompletionStatus(m_hIOCP, 0, 0, NULL);
        }
        for (auto& t : m_workers) {
            if (t.joinable()) t.join();
        }
        CloseHandle(m_hIOCP);
        WSACleanup();
    }

    // 콜백 등록
    using CompletionHandler = std::function<void(
        ULONG_PTR key, IoContext* ctx, DWORD bytes, bool success)>;
    CompletionHandler onCompletion;

private:
    void WorkerThread() {
        DWORD bytes;
        ULONG_PTR key;
        OVERLAPPED* pOv;

        while (true) {
            BOOL result = GetQueuedCompletionStatus(
                m_hIOCP, &bytes, &key, &pOv, INFINITE);

            if (key == 0 && pOv == NULL) break;  // 종료 시그널

            IoContext* ctx = reinterpret_cast<IoContext*>(pOv);
            bool success = result && (bytes > 0 || ctx->type == IoType::Accept);

            if (onCompletion) {
                onCompletion(key, ctx, bytes, success);
            }
        }
    }
};
```

### 11.4.2 래퍼를 사용한 서버 구현

```cpp
// main.cpp - 경량 래퍼 사용 예시
#include "iocp_core.h"
#include <cstdio>

int main() {
    IocpCore core;
    if (!core.Initialize()) {
        printf("IOCP init failed\n");
        return 1;
    }

    // 완료 핸들러 등록
    core.onCompletion = [](ULONG_PTR key, IoContext* ctx,
                           DWORD bytes, bool success) {
        if (!success) {
            printf("Session %llu disconnected\n", key);
            delete ctx;
            return;
        }

        switch (ctx->type) {
        case IoType::Recv:
            printf("Recv %d bytes from session %llu\n", bytes, key);
            // Echo: Recv 데이터를 Send
            ctx->type = IoType::Send;
            ctx->wsaBuf.len = bytes;
            ctx->Reset();
            WSASend((SOCKET)key, &ctx->wsaBuf, 1, NULL, 0,
                    &ctx->overlapped, NULL);
            break;

        case IoType::Send:
            printf("Sent %d bytes to session %llu\n", bytes, key);
            // 다시 Recv
            ctx->type = IoType::Recv;
            ctx->wsaBuf.buf = ctx->buffer;
            ctx->wsaBuf.len = sizeof(ctx->buffer);
            ctx->Reset();
            DWORD flags = 0;
            WSARecv((SOCKET)key, &ctx->wsaBuf, 1, NULL, &flags,
                    &ctx->overlapped, NULL);
            break;
        }
    };

    // Listen 소켓 생성 (생략: bind, listen, accept 루프...)
    printf("Server ready with lightweight IOCP wrapper\n");

    // ... (accept 루프 및 세션 관리)

    core.Shutdown();
    return 0;
}
```

---

## 11.5 게임 서버 프레임워크

### 11.5.1 오픈소스 게임 서버 프레임워크

| 프레임워크 | 언어 | IOCP 지원 | 특징 | GitHub Stars* |
|-----------|------|----------|------|--------------|
| **GameNetworkingSockets** | C++ | O (Windows) | Valve이 만든 게임 네트워킹 라이브러리. 릴레이, 암호화, P2P 지원 | 8k+ |
| **ENet** | C | X (select 기반) | 신뢰성 있는 UDP. 많은 인디 게임에서 사용. 단순함 | 6k+ |
| **KCP** | C | X (유저 레벨) | 빠른 ARQ(자동 재전송) 프로토콜. UDP 위에 신뢰성 계층 | 14k+ |
| **Photon Server** | C#/.NET | O (내부) | 상용 게임 서버 플랫폼. Unity 통합 | 상용 |
| **Nakama** | Go | X | 오픈소스 게임 백엔드. 매치메이킹, 채팅 등 | 8k+ |

*대략적 수치

### 11.5.2 Valve GameNetworkingSockets

```cpp
// Valve의 네트워킹 라이브러리 (IOCP 지원)
#include <steam/steamnetworkingsockets.h>
#include <steam/isteamnetworkingutils.h>

// 초기화
SteamDatagramErrMsg errMsg;
if (!GameNetworkingSockets_Init(nullptr, errMsg)) {
    printf("Init failed: %s\n", errMsg);
    return;
}

ISteamNetworkingSockets* pInterface =
    SteamNetworkingSockets();

// 서버 리스닝
SteamNetworkingIPAddr serverAddr;
serverAddr.Clear();
serverAddr.m_port = 27015;

HSteamListenSocket hListenSock =
    pInterface->CreateListenSocketIP(serverAddr, 0, nullptr);

// 폴링 (내부적으로 IOCP/epoll 사용)
pInterface->RunCallbacks();

// 특징:
// - 자체 신뢰성 프로토콜 (UDP 기반)
// - Steam Relay 네트워크 통합 (선택)
// - AES-GCM 암호화 내장
// - NAT 트래버설 내장
// - 최적화된 게임용 네트워킹
```

### 11.5.3 ENet

```c
// ENet: 간단한 신뢰성 UDP 라이브러리
// 내부적으로 IOCP를 사용하지 않지만 (select 기반)
// 게임 네트워킹의 기본을 이해하기 좋음

#include <enet/enet.h>

// 서버 초기화
enet_initialize();
ENetAddress address;
address.host = ENET_HOST_ANY;
address.port = 9000;

ENetHost* server = enet_host_create(&address, 32, 2, 0, 0);

// 이벤트 폴링
ENetEvent event;
while (enet_host_service(server, &event, 1000) > 0) {
    switch (event.type) {
    case ENET_EVENT_TYPE_CONNECT:
        printf("Client connected\n");
        break;
    case ENET_EVENT_TYPE_RECEIVE:
        printf("Received %zu bytes\n", event.packet->dataLength);
        // Echo
        enet_peer_send(event.peer, 0, event.packet);
        break;
    case ENET_EVENT_TYPE_DISCONNECT:
        printf("Client disconnected\n");
        break;
    }
}

// ENet 장점: 매우 단순, 채널 기반 멀티플렉싱, 신뢰/비신뢰 패킷 혼합
// ENet 단점: select 기반으로 대규모 동시접속에 부적합
// → IOCP와 조합: ENet의 프로토콜 로직만 가져와서 IOCP 소켓으로 교체
```

---

## 11.6 라이브러리 선택 가이드

### 11.6.1 프로젝트 유형별 권장

```
학습/프로토타입:
  → IOCP 직접 구현 (Section 3의 Echo 서버부터 시작)
  → 이유: IOCP 내부 동작 이해가 목적

소규모 프로젝트 (1인 개발):
  → Boost.Asio standalone 또는 경량 래퍼
  → 이유: 빠른 개발, 크로스 플랫폼 가능성

게임 서버 (인디):
  → Boost.Asio + 커스텀 게임 레이어
  → 또는 GameNetworkingSockets (FPS/액션)
  → 이유: 검증된 네트워크 레이어 + 게임 로직에 집중

게임 서버 (프로):
  → IOCP 직접 구현 + 사내 프레임워크
  → 이유: 최대 성능 제어, 게임별 최적화

비게임 서버 (웹/API):
  → Boost.Asio 또는 Windows Thread Pool
  → 이유: 안정성, 유지보수성
```

### 11.6.2 의사결정 플로차트

```
시작
  │
  ├─ Windows 전용인가?
  │   ├─ Yes → IOCP 직접 또는 Thread Pool API
  │   └─ No  → Boost.Asio (크로스 플랫폼)
  │
  ├─ 최대 성능이 필요한가?
  │   ├─ Yes → IOCP 직접 구현 (완전 제어)
  │   └─ No  → Boost.Asio (충분한 성능)
  │
  ├─ C++20 코루틴을 쓸 수 있는가?
  │   ├─ Yes → Asio + co_await (가장 깔끔한 코드)
  │   └─ No  → Asio 콜백 또는 IOCP 직접
  │
  ├─ 게임 네트워킹(UDP, 신뢰성, NAT)?
  │   ├─ Yes → GameNetworkingSockets 또는 ENet
  │   └─ No  → Asio 또는 직접 구현
  │
  └─ 학습 목적인가?
      ├─ Yes → IOCP 직접 구현 (반드시)
      └─ No  → 프로젝트 요구사항에 맞는 라이브러리
```

---

## 11.7 직접 구현 vs 라이브러리 사용

### 11.7.1 비교표

| 기준 | 직접 구현 | Boost.Asio | Thread Pool API |
|------|----------|-----------|----------------|
| 학습 가치 | 최고 | 보통 | 보통 |
| 개발 속도 | 느림 | 빠름 | 중간 |
| 성능 제어 | 완전 | 제한적 | 제한적 |
| 코드량 | 많음 (500~2000줄) | 적음 (100~300줄) | 중간 |
| 디버깅 | 직접 가능 | 라이브러리 내부 추적 어려움 | 블랙박스 |
| 유지보수 | 본인 책임 | 커뮤니티 지원 | MS 지원 |
| 크로스 플랫폼 | X | O | X |
| 외부 의존성 | 없음 | Boost 또는 standalone | 없음 |

### 11.7.2 권장 학습 경로

```
Phase 1: IOCP 직접 구현 (필수)
  - Echo 서버부터 시작
  - AcceptEx, WSARecv, WSASend 직접 사용
  - Worker 스레드, 세션 관리 직접 구현
  - 에러 처리, Graceful Shutdown 직접 구현
  → IOCP의 모든 개념을 체득

Phase 2: 구조화 (경량 래퍼)
  - Phase 1 코드를 클래스로 정리
  - IoContext, Session, Server 분리
  - 콜백 기반으로 추상화
  → 재사용 가능한 네트워크 라이브러리 완성

Phase 3: Boost.Asio 학습
  - Asio의 추상화가 IOCP와 어떻게 대응하는지 이해
  - 코루틴 버전 작성
  - 크로스 플랫폼 서버 구현
  → Phase 1의 지식이 있으면 Asio 이해가 매우 쉬움

Phase 4: 프로젝트 적용
  - 요구사항에 맞는 도구 선택
  - 성능 크리티컬 → 직접 구현
  - 생산성 중요 → Asio
  - 게임 특화 → GameNetworkingSockets 등
```

---

## 참고 자료

- [Boost.Asio Documentation](https://www.boost.org/doc/libs/release/doc/html/boost_asio.html)
- [Asio Standalone](https://think-async.com/Asio/)
- [libuv Documentation](https://docs.libuv.org/)
- [Valve GameNetworkingSockets](https://github.com/ValveSoftware/GameNetworkingSockets)
- [ENet](http://enet.bespin.org/)
- [KCP Protocol](https://github.com/skywind3000/kcp)
- Chris Kohlhoff, "Networking TS" (C++ Standards Committee papers)
