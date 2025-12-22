# Rookiss 게임서버 강의 코드 구조 분석

## C++ vs C# 강의 코드 구조 비교

두 강의 모두 **거의 동일한 아키텍처 패턴**을 사용합니다.

---

## 공통 프로젝트 구조

```
Solution/
├── ServerCore/          ← 핵심 네트워크 라이브러리 (재사용 가능)
│   ├── IocpCore         ← IOCP 핸들 관리
│   ├── IocpEvent        ← Accept, Recv, Send 이벤트
│   ├── IocpObject       ← Listener, Session의 부모 추상 클래스
│   ├── Listener         ← Accept 담당
│   ├── Session          ← Recv, Send, Disconnect 담당
│   ├── Connector        ← 클라이언트 → 서버 연결
│   ├── Service          ← ServerService, ClientService
│   ├── RecvBuffer       ← 수신 버퍼 관리
│   ├── SendBuffer       ← 송신 버퍼 관리
│   └── PacketSession    ← 패킷 단위 처리
│
├── Server/              ← 실제 게임 서버 (ServerCore 참조)
│   ├── GameSession      ← Session 상속
│   ├── GameRoom         ← 게임 로직
│   └── main.cpp/.cs
│
├── DummyClient/         ← 테스트용 더미 클라이언트
│
└── Common/              ← 공용 패킷 정의, Protocol 등
```

---

## 핵심 클래스 계층 구조

```
┌─────────────────────────────────────────────────────────────┐
│                      IocpObject (추상)                       │
│  - virtual HANDLE GetHandle() = 0                           │
│  - virtual void Dispatch(IocpEvent*, int32) = 0            │
└────────────────────────┬────────────────────────────────────┘
                         │
         ┌───────────────┼───────────────┐
         ▼               ▼               ▼
   ┌──────────┐    ┌──────────┐    ┌───────────┐
   │ Listener │    │ Session  │    │ Connector │
   │ (Accept) │    │(R/W/Disc)│    │ (Connect) │
   └──────────┘    └────┬─────┘    └───────────┘
                        │
                        ▼
                 ┌─────────────┐
                 │PacketSession│
                 │ (패킷 단위) │
                 └──────┬──────┘
                        │
                        ▼
                 ┌─────────────┐
                 │ GameSession │  ← 사용자 정의 (상속)
                 │ (게임 로직) │
                 └─────────────┘
```

---

## C++ 버전 주요 코드 (ServerCore)

### IocpCore.h
```cpp
class IocpObject : public enable_shared_from_this<IocpObject> {
public:
    virtual HANDLE GetHandle() abstract;
    virtual void Dispatch(class IocpEvent* iocpEvent, int32 numOfBytes) abstract;
};

class IocpCore {
public:
    HANDLE GetHandle() { return _iocpHandle; }

    bool Register(IocpObjectRef iocpObject);
    bool Dispatch(uint32 timeoutMs = INFINITE);

private:
    HANDLE _iocpHandle;
};
```

### Session.h
```cpp
class Session : public IocpObject {
public:
    // IocpObject 구현
    virtual HANDLE GetHandle() override;
    virtual void Dispatch(IocpEvent* iocpEvent, int32 numOfBytes) override;

    // 사용자 재정의 가능 (가상 함수)
    virtual void OnConnected() { }
    virtual int32 OnRecv(BYTE* buffer, int32 len) { return len; }
    virtual void OnSend(int32 len) { }
    virtual void OnDisconnected() { }

    // 내부 처리
    void RegisterRecv();
    void RegisterSend();
    void ProcessRecv(int32 numOfBytes);
    void ProcessSend(int32 numOfBytes);

private:
    SOCKET _socket = INVALID_SOCKET;
    RecvBuffer _recvBuffer;
    queue<SendBufferRef> _sendQueue;
    atomic<bool> _sendRegistered = false;
};
```

### Listener.h
```cpp
class Listener : public IocpObject {
public:
    bool StartAccept(NetAddress netAddress);
    void CloseSocket();

    // IocpObject 구현
    virtual HANDLE GetHandle() override;
    virtual void Dispatch(IocpEvent* iocpEvent, int32 numOfBytes) override;

private:
    void RegisterAccept(AcceptEvent* acceptEvent);
    void ProcessAccept(AcceptEvent* acceptEvent);

    SOCKET _socket = INVALID_SOCKET;
    Vector<AcceptEvent*> _acceptEvents;
};
```

### Service.h (핵심 추상화!)
```cpp
enum class ServiceType : uint8 {
    Server,
    Client
};

using SessionFactory = function<SessionRef(void)>;

class Service : public enable_shared_from_this<Service> {
public:
    Service(ServiceType type, NetAddress address,
            IocpCoreRef core, SessionFactory factory,
            int32 maxSessionCount = 1);

    virtual bool Start() abstract;
    virtual void CloseService();

    SessionRef CreateSession();
    void AddSession(SessionRef session);
    void ReleaseSession(SessionRef session);

protected:
    ServiceType _type;
    NetAddress _netAddress;
    IocpCoreRef _iocpCore;
    SessionFactory _sessionFactory;

    Set<SessionRef> _sessions;
    int32 _sessionCount = 0;
    int32 _maxSessionCount = 0;
};

class ServerService : public Service {
public:
    virtual bool Start() override;
private:
    ListenerRef _listener = nullptr;
};

class ClientService : public Service {
public:
    virtual bool Start() override;
};
```

---

## C# 버전 주요 코드 (ServerCore)

### Listener.cs
```csharp
namespace ServerCore
{
    public class Listener
    {
        Socket _listenSocket;
        Func<Session> _sessionFactory;

        public void Init(IPEndPoint endPoint, Func<Session> sessionFactory)
        {
            _listenSocket = new Socket(endPoint.AddressFamily,
                                       SocketType.Stream, ProtocolType.Tcp);
            _sessionFactory = sessionFactory;

            _listenSocket.Bind(endPoint);
            _listenSocket.Listen(10);

            // 여러 개의 Accept 동시 대기 (Pre-posted)
            for (int i = 0; i < 10; i++)
            {
                SocketAsyncEventArgs args = new SocketAsyncEventArgs();
                args.Completed += OnAcceptCompleted;
                RegisterAccept(args);
            }
        }

        void RegisterAccept(SocketAsyncEventArgs args)
        {
            args.AcceptSocket = null;
            bool pending = _listenSocket.AcceptAsync(args);
            if (!pending)
                OnAcceptCompleted(null, args);
        }

        void OnAcceptCompleted(object sender, SocketAsyncEventArgs args)
        {
            if (args.SocketError == SocketError.Success)
            {
                Session session = _sessionFactory.Invoke();
                session.Start(args.AcceptSocket);
                session.OnConnected(args.AcceptSocket.RemoteEndPoint);
            }
            RegisterAccept(args);
        }
    }
}
```

### Session.cs
```csharp
namespace ServerCore
{
    public abstract class Session
    {
        Socket _socket;
        int _disconnected = 0;

        RecvBuffer _recvBuffer = new RecvBuffer(65535);
        Queue<ArraySegment<byte>> _sendQueue = new Queue<ArraySegment<byte>>();
        SocketAsyncEventArgs _sendArgs = new SocketAsyncEventArgs();
        SocketAsyncEventArgs _recvArgs = new SocketAsyncEventArgs();

        // 사용자 재정의 (추상 메서드)
        public abstract void OnConnected(EndPoint endPoint);
        public abstract int OnRecv(ArraySegment<byte> buffer);
        public abstract void OnSend(int numOfBytes);
        public abstract void OnDisconnected(EndPoint endPoint);

        public void Start(Socket socket)
        {
            _socket = socket;
            _recvArgs.Completed += OnRecvCompleted;
            _sendArgs.Completed += OnSendCompleted;
            RegisterRecv();
        }

        void RegisterRecv()
        {
            _recvBuffer.Clean();
            ArraySegment<byte> segment = _recvBuffer.WriteSegment;
            _recvArgs.SetBuffer(segment.Array, segment.Offset, segment.Count);

            bool pending = _socket.ReceiveAsync(_recvArgs);
            if (!pending)
                OnRecvCompleted(null, _recvArgs);
        }
    }
}
```

### PacketSession.cs
```csharp
public abstract class PacketSession : Session
{
    public static readonly int HeaderSize = 2;  // [size(2)]

    public sealed override int OnRecv(ArraySegment<byte> buffer)
    {
        int processLen = 0;

        while (true)
        {
            if (buffer.Count < HeaderSize)
                break;

            ushort dataSize = BitConverter.ToUInt16(buffer.Array, buffer.Offset);
            if (buffer.Count < dataSize)
                break;

            OnRecvPacket(new ArraySegment<byte>(buffer.Array,
                         buffer.Offset, dataSize));

            processLen += dataSize;
            buffer = new ArraySegment<byte>(buffer.Array,
                     buffer.Offset + dataSize, buffer.Count - dataSize);
        }

        return processLen;
    }

    public abstract void OnRecvPacket(ArraySegment<byte> buffer);
}
```

---

## SharedPtr 타입 정의 (C++)

```cpp
// Types.h
using IocpCoreRef = shared_ptr<class IocpCore>;
using IocpObjectRef = shared_ptr<class IocpObject>;
using SessionRef = shared_ptr<class Session>;
using ListenerRef = shared_ptr<class Listener>;
using ServerServiceRef = shared_ptr<class ServerService>;
using ClientServiceRef = shared_ptr<class ClientService>;
```

---

## 사용 예시 비교

### C++ Server main.cpp
```cpp
int main()
{
    // SessionFactory: GameSession 생성 람다
    ServerServiceRef service = make_shared<ServerService>(
        NetAddress(L"127.0.0.1", 7777),
        make_shared<IocpCore>(),
        []() { return make_shared<GameSession>(); },  // Factory
        100  // maxSessionCount
    );

    service->Start();

    // Worker Thread Pool
    for (int i = 0; i < 5; i++)
    {
        GThreadManager->Launch([=]()
        {
            while (true)
            {
                service->GetIocpCore()->Dispatch();
            }
        });
    }

    GThreadManager->Join();
}
```

### C# Server Program.cs
```csharp
class Program
{
    static Listener _listener = new Listener();

    static void Main(string[] args)
    {
        IPEndPoint endPoint = new IPEndPoint(IPAddress.Any, 7777);

        _listener.Init(endPoint, () => { return new GameSession(); });

        while (true)
        {
            // 메인 스레드는 다른 작업 수행
        }
    }
}
```

---

## C++ vs C# 핵심 차이점

| 항목 | C++ | C# |
|------|-----|-----|
| **I/O 모델** | IOCP (직접 구현) | SocketAsyncEventArgs (내장) |
| **메모리 관리** | shared_ptr, custom allocator | GC (가비지 컬렉션) |
| **스레드 풀** | 직접 관리 (ThreadManager) | ThreadPool 내장 |
| **이벤트 처리** | IocpEvent + Dispatch() | Completed += 콜백 |
| **성능** | 더 높음 (세밀한 제어 가능) | 편의성 우선 |
| **난이도** | 높음 | 상대적으로 낮음 |

---

## 정리: 왜 구조가 비슷한가?

1. **동일한 설계자**: 두 강의 모두 Rookiss님이 설계한 동일한 아키텍처
2. **Proactor 패턴**: 완료 기반 통지 (Completion-based) 사용
3. **세션 팩토리 패턴**: 다형성을 통한 세션 생성
4. **Service 추상화**: Server/Client 공통 인터페이스
5. **PacketSession**: 패킷 경계 처리 공통 로직

### 결론

> C#은 C++의 IOCP 로직을 **`SocketAsyncEventArgs`로 대체**했을 뿐,
> **클래스 구조와 책임 분리는 완전히 동일**합니다.

따라서 C++ 또는 C# 한 쪽만 이해하면 다른 언어로 이식하기 매우 쉽습니다.

---

## 학습 순서 권장

```
1. C++ Part4 먼저 학습 (IOCP 직접 구현으로 원리 이해)
   ↓
2. C# Part4로 같은 구조 확인 (SocketAsyncEventArgs가 IOCP 래핑)
   ↓
3. 자신만의 ServerCore 라이브러리 구현
   ↓
4. MMORPG 포트폴리오에 적용
```

---

## 참고 자료

- [인프런 - C++과 언리얼로 만드는 MMORPG Part4](https://www.inflearn.com/course/언리얼-3d-mmorpg-4)
- [인프런 - C#과 유니티로 만드는 MMORPG Part4](https://www.inflearn.com/course/유니티-mmorpg-개발-part4)
- [GitHub - pinch24/learn.unreal.rookiss.mmorpg-server](https://github.com/pinch24/learn.unreal.rookiss.mmorpg-server)
- [GitHub - Tuesberry/RIO_IOCP_Server](https://github.com/Tuesberry/RIO_IOCP_Server)
- [LHH Blog - C# Rookiss Part4 게임서버](https://lhuhyeon.github.io/categories/c-rookiss-part4-게임서버/)
