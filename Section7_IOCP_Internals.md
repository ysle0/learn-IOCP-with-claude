# Section 7: IOCP Internals 상세 분석

> 작성 완료일: 2026-02-01

---

## 목차
- [7.1 KQUEUE 구조체 상세 분석](#71-kqueue-구조체-상세-분석)
- [7.2 IRP 생명주기](#72-irp-생명주기)
- [7.3 AFD.SYS (Ancillary Function Driver) 역할](#73-afdsys-ancillary-function-driver-역할)
- [7.4 MaximumConcurrency=0 시 동작](#74-maximumconcurrency0-시-동작)
- [7.5 IOCP 내부 큐 동작 메커니즘](#75-iocp-내부-큐-동작-메커니즘)
- [7.6 커널 디버깅으로 본 IOCP 내부](#76-커널-디버깅으로-본-iocp-내부)
- [7.7 퀴즈용 핵심 Q&A](#77-퀴즈용-핵심-qa)

---

## 7.1 KQUEUE 구조체 상세 분석

### 7.1.1 KQUEUE의 정체

IOCP는 유저 모드에서 `HANDLE`로 참조되지만, 커널 내부에서는 `KQUEUE`라는 커널 디스패처 객체(Dispatcher Object)로 구현된다. `KQUEUE`는 `ntoskrnl.exe`에 정의된 비공개(undocumented) 구조체이다.

```c
// Windows 커널 내부 (비공개, 리버스 엔지니어링/Windows Internals 기반)
typedef struct _KQUEUE {
    DISPATCHER_HEADER Header;       // 디스패처 객체 공통 헤더
    LIST_ENTRY EntryListHead;       // 완료 패킷 리스트 (FIFO)
    DWORD CurrentCount;             // 현재 실행 중인 스레드 수
    DWORD MaximumCount;             // 최대 동시 실행 스레드 수
    LIST_ENTRY ThreadListHead;      // 대기 중인 스레드 리스트 (LIFO)
} KQUEUE;
```

### 7.1.2 각 필드 상세

#### DISPATCHER_HEADER

```
DISPATCHER_HEADER는 모든 대기 가능한(waitable) 커널 객체의 공통 헤더이다.
- Type: 객체 타입 (KQUEUE의 경우 QueueObject)
- SignalState: 큐에 있는 항목 수
- WaitListHead: 이 객체를 대기하는 스레드 목록

역할:
- GetQueuedCompletionStatus는 내부적으로 KeRemoveQueueEx를 호출
- KeRemoveQueueEx는 KQUEUE의 DISPATCHER_HEADER를 확인하여
  패킷이 있으면 즉시 반환, 없으면 스레드를 대기 상태로 전환
```

#### EntryListHead (완료 패킷 리스트)

```
I/O가 완료되면 커널이 이 리스트에 완료 패킷(IO_MINI_COMPLETION_PACKET 또는
IOP_MINI_COMPLETION_PACKET)을 삽입한다.

동작:
- 삽입: FIFO (리스트 뒤에 추가) → InsertTailList
- 제거: FIFO (리스트 앞에서 제거) → RemoveHeadList
- PostQueuedCompletionStatus도 이 리스트에 패킷을 수동 추가

데이터 구조:
┌───────────────────────────────────────────┐
│ EntryListHead (LIST_ENTRY)                 │
│                                            │
│ ←[Packet1]←→[Packet2]←→[Packet3]←→...   │
│   (oldest)                    (newest)     │
│                                            │
│ 제거 ← 이쪽에서          이쪽에 추가 →    │
└───────────────────────────────────────────┘
```

#### ThreadListHead (대기 스레드 리스트)

```
GetQueuedCompletionStatus를 호출하고 대기 중인 스레드들의 리스트.

핵심: LIFO 순서!
- 새로 대기에 들어가는 스레드가 리스트 앞에 추가됨
- 깨울 때도 리스트 앞에서 선택 → 가장 최근에 대기한 스레드가 먼저 깨어남

이유: CPU 캐시 지역성 최적화
- 최근 실행된 스레드의 코드/데이터가 CPU 캐시에 남아있을 확률 높음
- LIFO로 깨우면 캐시 히트율 극대화
- FIFO로 깨우면 캐시가 식은(cold) 스레드를 깨우게 됨

┌───────────────────────────────────────────┐
│ ThreadListHead (LIST_ENTRY)                │
│                                            │
│ →[Thread C]→[Thread B]→[Thread A]→...    │
│   (latest)              (oldest)           │
│                                            │
│ ← 깨울 때 이쪽에서 선택                    │
│ ← 새 대기 스레드도 이쪽에 추가             │
└───────────────────────────────────────────┘
```

#### CurrentCount / MaximumCount

```
CurrentCount: 현재 IOCP에 의해 깨어나서 실행 중인 스레드 수
MaximumCount: CreateIoCompletionPort의 NumberOfConcurrentThreads

동작 규칙:
- 스레드가 깨어남 → CurrentCount++
- 스레드가 GQCS에 재진입 → CurrentCount--
- 스레드가 다른 대기 함수로 블록 → CurrentCount-- (Released 상태)

스레드를 깨울 조건:
1. EntryListHead에 패킷이 있고
2. CurrentCount < MaximumCount이고
3. ThreadListHead에 대기 스레드가 있음

→ 세 조건 모두 충족 시 LIFO로 스레드 1개 깨움
```

### 7.1.3 KQUEUE와 관련된 커널 함수

| 커널 함수 | 유저 모드 대응 | 용도 |
|----------|---------------|------|
| `KeInitializeQueue` | `CreateIoCompletionPort` (생성) | KQUEUE 초기화 |
| `KeInsertQueue` | I/O 완료 시 자동 호출 | 완료 패킷 삽입 |
| `KeInsertHeadQueue` | `PostQueuedCompletionStatus`* | 패킷을 큐 앞에 삽입 |
| `KeRemoveQueue` | `GetQueuedCompletionStatus` | 패킷 제거 (블로킹) |
| `KeRemoveQueueEx` | `GetQueuedCompletionStatusEx` | 복수 패킷 제거 |
| `KeRundownQueue` | IOCP 핸들 닫기 | 큐 정리 |

* `PostQueuedCompletionStatus`는 실제로 `NtSetIoCompletion`을 호출하며, 이는 `IoSetIoCompletion`을 통해 큐 뒤쪽에 삽입한다.

---

## 7.2 IRP 생명주기

### 7.2.1 IRP(I/O Request Packet)란

IRP는 Windows 커널의 I/O Manager가 모든 I/O 요청을 표현하는 데이터 구조이다. 유저 모드에서 WSARecv를 호출하면 커널에서 IRP가 생성되어 드라이버 스택을 통과한다.

```c
// IRP 구조체 (간략화)
typedef struct _IRP {
    PMDL MdlAddress;               // 버퍼의 물리 메모리 매핑
    ULONG Flags;
    union {
        struct _IRP *MasterIrp;
        PVOID SystemBuffer;        // Buffered I/O용
    } AssociatedIrp;
    IO_STATUS_BLOCK IoStatus;      // ← OVERLAPPED.Internal/InternalHigh와 대응
    KPROCESSOR_MODE RequestorMode; // UserMode 또는 KernelMode
    BOOLEAN PendingReturned;
    BOOLEAN Cancel;
    KIRQL CancelIrql;
    PIO_APC_ROUTINE UserApcRoutine;
    PVOID UserApcContext;
    PIO_COMPLETION_ROUTINE CompletionRoutine;
    // ... IO_STACK_LOCATION 배열
} IRP;
```

### 7.2.2 IRP 생명주기 (WSARecv 기준)

```
Phase 1: IRP 생성
────────────────────────────────────────
User Mode: WSARecv(socket, &buf, ..., &overlapped)
    ↓
ntdll.dll: NtDeviceIoControlFile()  [시스템 콜]
    ↓
Kernel Mode:
    ↓
I/O Manager: IoAllocateIrp()
    → IRP를 Non-paged Pool에서 할당
    → IO_STACK_LOCATION 초기화 (MajorFunction = IRP_MJ_DEVICE_CONTROL)
    → IOCTL = AFD_RECV
    ↓
I/O Manager: OVERLAPPED → IO_STATUS_BLOCK 매핑
    → overlapped->Internal = STATUS_PENDING (0x103)
    ↓
I/O Manager: 유저 버퍼 잠금 (IoAllocateMdl + MmProbeAndLockPages)
    → MDL 생성, 물리 페이지 잠금


Phase 2: IRP 전달 (드라이버 스택)
────────────────────────────────────────
I/O Manager → IoCallDriver()
    ↓
┌──────────────────┐
│ AFD.SYS          │ ← Winsock 커널 드라이버
│ (소켓 로직 처리)  │
│ 프로토콜 버퍼 관리│
└────────┬─────────┘
         ↓ IoCallDriver()
┌──────────────────┐
│ TCPIP.SYS        │ ← TCP/IP 프로토콜 드라이버
│ (TCP 상태 관리)   │
│ (수신 버퍼 관리)  │
└────────┬─────────┘
         ↓ IoCallDriver()
┌──────────────────┐
│ NDIS.SYS         │ ← 네트워크 인터페이스
│ (NIC 드라이버)    │
└──────────────────┘

각 드라이버에서:
- 데이터가 이미 있으면: IRP 즉시 완료 (동기 완료)
- 데이터가 없으면: IRP를 보류 (IoMarkIrpPending)
  → STATUS_PENDING 반환


Phase 3: 데이터 도착 및 IRP 완료
────────────────────────────────────────
NIC: 패킷 수신 → DMA로 메모리에 복사
    ↓
NDIS: 인터럽트 → DPC(Deferred Procedure Call) 스케줄
    ↓
DPC에서:
    TCPIP.SYS: TCP 처리 (재조립, 시퀀스 확인, ACK)
    ↓
    AFD.SYS: 대기 중인 IRP 찾기
    → 수신 데이터를 IRP의 MDL이 가리키는 유저 버퍼에 복사
    ↓
    IoCompleteRequest(Irp, IO_NETWORK_INCREMENT)
    → IRP의 IO_STATUS_BLOCK 설정:
      Status = STATUS_SUCCESS
      Information = 수신된 바이트 수


Phase 4: IOCP 통지
────────────────────────────────────────
IoCompleteRequest 내부:
    ↓
I/O Manager: 파일 객체에 연결된 CompletionPort 확인
    ↓
IoSetIoCompletion():
    → Mini Completion Packet 생성
    → KQUEUE의 EntryListHead에 삽입
    → 대기 스레드 깨움 (KeInsertQueue → KiInsertQueue)
    ↓
유저 버퍼 잠금 해제 (MmUnlockPages + IoFreeMdl)
    ↓
IRP 해제 (IoFreeIrp → Non-paged Pool로 반환)


Phase 5: 유저 모드 수신
────────────────────────────────────────
Worker Thread: GetQueuedCompletionStatus 반환
    → KeRemoveQueue가 패킷을 반환
    → bytesTransferred = IO_STATUS_BLOCK.Information
    → pOverlapped->Internal = IO_STATUS_BLOCK.Status
    → pOverlapped->InternalHigh = IO_STATUS_BLOCK.Information
```

### 7.2.3 IRP 취소 메커니즘

```
CancelIoEx 호출 시:

1. I/O Manager가 IRP의 Cancel 필드를 TRUE로 설정
2. IRP의 CancelRoutine 호출 (드라이버가 등록)
3. 드라이버의 CancelRoutine:
   - 대기 큐에서 IRP 제거
   - IoCompleteRequest(Irp, STATUS_CANCELLED)
4. IOCP에 완료 패킷 전달 (에러 상태)
5. Worker에서 ERROR_OPERATION_ABORTED 수신

주의:
- CancelRoutine이 없는 IRP는 취소 불가
- 이미 완료 진행 중인 IRP도 취소 불가 (경합 상태)
- CancelIoEx 후에도 반드시 완료 통지를 기다려야 함
```

---

## 7.3 AFD.SYS (Ancillary Function Driver) 역할

### 7.3.1 AFD.SYS란

AFD.SYS는 Windows 소켓(Winsock)의 커널 모드 구현체이다. 유저 모드의 `ws2_32.dll`(Winsock2 라이브러리)과 커널 모드의 `tcpip.sys`(TCP/IP 드라이버) 사이에서 중간 역할을 수행한다.

```
계층 구조:

유저 모드:
  ┌────────────────────────────┐
  │ Application                 │
  │   WSARecv(), WSASend()      │
  └──────────┬─────────────────┘
             ↓
  ┌────────────────────────────┐
  │ ws2_32.dll (Winsock2)       │
  │   소켓 핸들 관리            │
  │   카탈로그/프로바이더 관리   │
  └──────────┬─────────────────┘
             ↓ DeviceIoControl
─────────── 커널 경계 ───────────
             ↓
  ┌────────────────────────────┐
  │ AFD.SYS                     │ ← "Ancillary Function Driver"
  │   소켓 상태 관리            │
  │   버퍼 관리 (수신/송신)     │
  │   연결 관리                 │
  │   AcceptEx, ConnectEx 구현  │
  │   Overlapped I/O 처리       │
  └──────────┬─────────────────┘
             ↓
  ┌────────────────────────────┐
  │ TDI / WFP                   │ ← Transport Driver Interface
  └──────────┬─────────────────┘
             ↓
  ┌────────────────────────────┐
  │ TCPIP.SYS                   │
  │   TCP/UDP 프로토콜 처리     │
  └──────────┬─────────────────┘
             ↓
  ┌────────────────────────────┐
  │ NDIS / NIC Driver           │
  └────────────────────────────┘
```

### 7.3.2 AFD.SYS의 핵심 역할

#### 역할 1: Winsock IOCTL 처리

```
ws2_32.dll의 소켓 함수들은 내부적으로 DeviceIoControl을 통해
AFD.SYS에 IOCTL을 보낸다:

WSARecv()     → AFD_RECV       (0x12017)
WSASend()     → AFD_SEND       (0x1201F)
AcceptEx()    → AFD_ACCEPT     (0x12010)  [SUPER_ACCEPT]
ConnectEx()   → AFD_CONNECT    (0x12007)
DisconnectEx()→ AFD_DISCONNECT (0x1202B)
bind()        → AFD_BIND       (0x12003)
listen()      → AFD_LISTEN     (0x1200B) [START_LISTEN]
setsockopt()  → AFD_SET_INFO   (0x1201B)
```

#### 역할 2: 수신 버퍼 관리

```
AFD.SYS는 TCP 수신 데이터를 위한 자체 버퍼를 관리한다.

데이터 도착 시:
1. TCPIP.SYS가 데이터를 AFD.SYS에 전달
2. AFD.SYS:
   a. 대기 중인 WSARecv IRP가 있으면 → 직접 유저 버퍼에 복사 → IRP 완료
   b. 대기 중인 IRP가 없으면 → AFD 내부 버퍼에 저장 (SO_RCVBUF)
3. 이후 WSARecv가 호출되면 AFD 버퍼에서 복사

이것이 "SO_RCVBUF = 0" 설정의 의미:
→ AFD.SYS가 내부 버퍼링을 하지 않음
→ WSARecv IRP가 대기하지 않으면 데이터가 TCP 레벨에서 대기
→ TCP Window가 0이 되어 상대방이 전송 중지
```

#### 역할 3: AcceptEx의 실제 구현

```
AcceptEx 내부 동작 (AFD.SYS에서):

1. AFD_SUPER_ACCEPT IOCTL 수신
2. Accept 소켓의 커널 구조체 준비
3. IRP를 보류(Pending) 상태로 설정
4. TCPIP.SYS에 연결 수락 요청 등록

연결 도착 시:
5. TCPIP.SYS → AFD.SYS: 새 연결 통지
6. AFD.SYS: TDI 연결 객체 생성
7. dwReceiveDataLength > 0이면:
   → 첫 데이터가 올 때까지 IRP 보류 유지
   → 이것이 "AcceptEx 타임아웃 문제"의 원인
8. dwReceiveDataLength == 0이면:
   → 연결 수립 즉시 IRP 완료
9. 출력 버퍼에 로컬/원격 주소 정보 기록
10. IoCompleteRequest → IOCP에 통지
```

#### 역할 4: SO_SNDBUF = 0 동작

```
SO_SNDBUF = 0 설정 시 AFD.SYS 동작:

일반 (SO_SNDBUF > 0):
  WSASend → AFD 내부 송신 버퍼에 복사 → IRP 즉시 완료
  → 이후 AFD가 TCPIP.SYS를 통해 실제 전송
  → 유저 버퍼를 바로 재사용 가능

SO_SNDBUF = 0:
  WSASend → AFD가 유저 버퍼를 직접 잠금(lock)
  → 유저 버퍼에서 직접 TCPIP.SYS에 전달
  → 실제 네트워크 전송이 완료될 때까지 IRP 보류
  → 이 기간 동안 유저 버퍼 수정 불가

장점: 메모리 복사 1회 절감, 메모리 사용량 감소
단점: 실제 전송 완료까지 WSASend가 대기, 처리량(throughput) 감소 가능
```

---

## 7.4 MaximumConcurrency=0 시 동작

### 7.4.1 시스템 확인 경로

```
CreateIoCompletionPort(INVALID_HANDLE_VALUE, NULL, 0, NumberOfConcurrentThreads)

NumberOfConcurrentThreads = 0일 때:

ntoskrnl.exe 내부:
1. NtCreateIoCompletion() 호출
2. KeInitializeQueue(&kqueue, MaximumConcurrency)
3. MaximumConcurrency = 0이면:
   → KeNumberProcessors (시스템 전역 변수) 값 사용
   → 이는 논리 프로세서 수 (하이퍼스레딩 포함)

확인 방법:
  SYSTEM_INFO si;
  GetSystemInfo(&si);
  // si.dwNumberOfProcessors == 논리 프로세서 수
  // NumberOfConcurrentThreads=0 → 이 값이 사용됨
```

### 7.4.2 논리 프로세서 vs 물리 코어

```
8코어 16스레드(HT) CPU에서:
  NumberOfConcurrentThreads = 0
  → MaximumConcurrency = 16 (논리 프로세서 수)

이것이 최적인가?
  - I/O 중심 작업: 16이 적절 (HT가 대기 시간을 숨김)
  - CPU 중심 작업: 8(물리 코어)이 더 나을 수 있음
    → HT 사용 시 캐시 경쟁으로 오히려 느려질 수 있음

물리 코어 수로 설정하려면:
  GetLogicalProcessorInformation() 또는
  GetLogicalProcessorInformationEx() 사용하여
  RelationProcessorCore로 물리 코어 수 확인
```

### 7.4.3 NUMA 시스템에서의 고려

```
NUMA(Non-Uniform Memory Access) 시스템에서는
메모리 접근 지역성이 중요하다.

2-소켓 서버 (각 소켓 8코어):
  NUMA Node 0: CPU 0-7,  메모리 32GB
  NUMA Node 1: CPU 8-15, 메모리 32GB

일반 IOCP:
  → Worker 스레드가 두 노드에 걸쳐 실행
  → 원격 노드 메모리 접근 시 지연 발생

최적화 방법:
  1. NUMA 노드별 IOCP 생성
  2. 각 노드의 CPU에 바인딩된 Worker 스레드
  3. 소켓을 가까운 NUMA 노드의 IOCP에 할당

// NUMA-aware IOCP 설정
HANDLE hIOCP_Node0 = CreateIoCompletionPort(INVALID_HANDLE_VALUE, NULL, 0, 8);
HANDLE hIOCP_Node1 = CreateIoCompletionPort(INVALID_HANDLE_VALUE, NULL, 0, 8);

// Node 0 Worker 스레드의 affinity 설정
SetThreadAffinityMask(hThread, 0xFF);     // CPU 0-7
// Node 1 Worker 스레드의 affinity 설정
SetThreadAffinityMask(hThread, 0xFF00);   // CPU 8-15
```

---

## 7.5 IOCP 내부 큐 동작 메커니즘

### 7.5.1 세 가지 내부 리스트

```
IOCP (KQUEUE) 내부에는 세 가지 리스트가 있다:

1. Completion Packet List (EntryListHead)
   → 완료된 I/O 패킷 대기열 (FIFO)

2. Waiting Thread List (ThreadListHead)
   → GQCS로 대기 중인 스레드 (LIFO)

3. Released Thread List (암시적)
   → IOCP에 의해 깨어났지만, 다른 이유로 블록된 스레드
   → 커널 내부적으로 스레드의 Queue 포인터가 유지됨
```

### 7.5.2 상세 시나리오: 스레드 상태 전이

```
시나리오: 4코어, MaximumConcurrency=4, Worker 6개

초기 상태:
  Packet Queue: []
  Waiting: [T6, T5, T4, T3, T2, T1]  (LIFO, T6이 최근)
  Running: (없음)
  Released: (없음)
  CurrentCount: 0

Step 1: I/O 완료 4건 동시 발생
  Packet Queue: [P1, P2, P3, P4]
  → T6 깨움 (LIFO, 가장 최근) → P1 처리, CurrentCount=1
  → T5 깨움 → P2 처리, CurrentCount=2
  → T4 깨움 → P3 처리, CurrentCount=3
  → T3 깨움 → P4 처리, CurrentCount=4
  Waiting: [T2, T1]
  Running: [T6, T5, T4, T3]
  CurrentCount: 4 (== MaximumCount)

Step 2: I/O 완료 1건 추가 발생
  Packet Queue: [P5]
  → CurrentCount(4) >= MaximumCount(4) → 스레드 깨우지 않음!
  → P5는 큐에서 대기

Step 3: T6가 DB 쿼리로 블록됨
  → 커널: T6를 Released 상태로 전환
  → CurrentCount: 3
  → 3 < 4 → T2 깨움 → P5 처리
  → CurrentCount: 4
  Running: [T2, T5, T4, T3]
  Released: [T6]

Step 4: T6의 DB 쿼리 완료, T6가 다시 실행 가능
  → CurrentCount: 5 (일시적 초과!)
  → 이 상태에서 T6가 GQCS 재호출하면
  → CurrentCount: 4 (정상 복귀)
  → T6는 Waiting으로 이동

Step 5: T3가 처리 완료, GQCS 재호출
  → CurrentCount: 3
  → 큐에 패킷 없으면 → T3는 Waiting으로
  → 큐에 패킷 있으면 → T3가 바로 다음 패킷 처리
```

### 7.5.3 LIFO vs FIFO의 영향

```
실험적 관찰:

6 Worker, 4 Concurrent, 균등한 I/O 부하에서:

LIFO 결과 (IOCP 실제 동작):
  Thread 1: 처리 횟수 0      (거의 안 깨어남)
  Thread 2: 처리 횟수 50
  Thread 3: 처리 횟수 200
  Thread 4: 처리 횟수 800
  Thread 5: 처리 횟수 2000
  Thread 6: 처리 횟수 5000   (가장 많이 깨어남)

→ 몇몇 스레드에 작업이 집중됨
→ CPU 캐시 히트율 극대화 (hot thread)
→ 사용되지 않는 스레드는 완전히 sleep (전력 절약)

가상 FIFO 동작이었다면:
  모든 스레드가 균등하게 ~1350회 처리
  → 모든 스레드의 캐시가 계속 갱신
  → 캐시 히트율 저하
  → 모든 스레드가 active (전력 소비 증가)
```

---

## 7.6 커널 디버깅으로 본 IOCP 내부

### 7.6.1 WinDbg로 IOCP 상태 확인

```
WinDbg 커널 디버깅에서 IOCP 내부를 확인하는 방법:

1. IOCP 핸들에서 커널 객체 찾기
kd> !handle <handle_value> f <process_id>

출력:
  Object: fffffa80`12345678
  Type: IoCompletion  ← IOCP 타입 확인

2. KQUEUE 구조체 덤프
kd> dt nt!_KQUEUE fffffa80`12345678

출력:
  +0x000 Header          : _DISPATCHER_HEADER
  +0x018 EntryListHead   : _LIST_ENTRY [...]
  +0x028 CurrentCount    : 3
  +0x02c MaximumCount    : 4
  +0x030 ThreadListHead  : _LIST_ENTRY [...]

3. 대기 중인 스레드 확인
kd> !ready 0
또는
kd> dl fffffa80`12345678+0x030 (ThreadListHead 순회)

4. 완료 패킷 확인
kd> dl fffffa80`12345678+0x018 (EntryListHead 순회)
```

### 7.6.2 ETW(Event Tracing for Windows)로 IOCP 모니터링

```
ETW Provider: Microsoft-Windows-Kernel-IoTrace

관련 이벤트:
- IoCompletion/Create: IOCP 생성
- IoCompletion/Associate: 핸들 연결
- IoCompletion/Insert: 완료 패킷 삽입
- IoCompletion/Remove: 완료 패킷 제거

xperf/WPA로 캡처:
> xperf -on IO+CSWITCH+DISPATCHER
> (서버 실행)
> xperf -d trace.etl

분석 가능한 것:
- 패킷 삽입-제거 간 지연 (큐 대기 시간)
- 스레드 전환 패턴 (LIFO 동작 확인)
- Worker 스레드별 처리량 불균형
- Context switch 빈도
```

### 7.6.3 Performance Counter로 모니터링

```
Windows Performance Monitor에서 확인 가능한 IOCP 관련 카운터:

Process 카운터:
- IO Read Operations/sec
- IO Write Operations/sec
- IO Other Operations/sec
- IO Read Bytes/sec
- IO Write Bytes/sec

System 카운터:
- Context Switches/sec
- System Calls/sec

Thread 카운터:
- Context Switches/sec (스레드별)
- % Processor Time (스레드별)

Winsock 관련:
- TCPv4 → Connections Active
- TCPv4 → Segments Received/sec
- TCPv4 → Segments Sent/sec
```

---

## 7.7 퀴즈용 핵심 Q&A

### 정의형 문제

**Q1**: IOCP는 커널 내부에서 어떤 구조체로 표현되는가?
> **A**: `KQUEUE` 구조체이다. Dispatcher Object의 일종으로, 완료 패킷 리스트(FIFO), 대기 스레드 리스트(LIFO), CurrentCount, MaximumCount를 포함한다.

**Q2**: AFD.SYS의 역할은?
> **A**: AFD.SYS(Ancillary Function Driver)는 Winsock의 커널 모드 구현체로, 유저 모드 `ws2_32.dll`과 커널 모드 `tcpip.sys` 사이에서 소켓 상태 관리, 버퍼 관리, AcceptEx/ConnectEx 구현, Overlapped I/O 처리 등을 수행한다.

**Q3**: IRP가 IOCP에 도달하기까지의 경로는?
> **A**: WSARecv → NtDeviceIoControlFile (시스템 콜) → I/O Manager가 IRP 생성 → AFD.SYS (소켓 처리) → TCPIP.SYS (TCP 처리) → NIC 드라이버. 데이터 도착 시 역순으로: NIC DPC → TCPIP.SYS → AFD.SYS → IoCompleteRequest → KQUEUE에 완료 패킷 삽입 → Worker 스레드 깨움.

### 수치형 문제

**Q4**: 8코어 16논리프로세서(HT) CPU에서 `NumberOfConcurrentThreads=0`으로 설정하면 실제 값은?
> **A**: 16. `NumberOfConcurrentThreads=0`은 논리 프로세서 수(`KeNumberProcessors`)를 사용하며, HT가 활성화된 8코어 CPU는 논리 프로세서가 16개이다.

### 적용형 문제

**Q5**: IOCP가 대기 스레드를 LIFO 순서로 깨우면, 일부 스레드는 거의 사용되지 않는다. 이것이 오히려 장점인 이유는?
> **A**: (1) 최근 실행된 스레드의 코드/데이터가 CPU 캐시(L1/L2)에 남아있어 캐시 히트율이 극대화된다. (2) 사용되지 않는 스레드는 완전히 sleep 상태로 전력을 절약한다. (3) Context switch 시 TLB flush, 캐시 오염 등이 최소화된다. 결과적으로 FIFO보다 전체 처리량(throughput)이 높아진다.

**Q6**: SO_SNDBUF=0으로 설정하면 AFD.SYS 동작이 어떻게 달라지고, 어떤 상황에서 유리한가?
> **A**: SO_SNDBUF=0이면 AFD.SYS가 내부 송신 버퍼를 사용하지 않고 유저 버퍼를 직접 잠가서(page lock) 네트워크 드라이버에 전달한다. 메모리 복사 1회가 절감되고 메모리 사용량이 줄지만, 실제 전송 완료까지 WSASend IRP가 보류된다. 대량의 세션이 동시에 데이터를 전송하여 메모리가 부족한 상황이나, Zero-copy가 필요한 고성능 시나리오에서 유리하다.

---

## 참고 자료

- Windows Internals, 7th Edition - Part 1, Chapter 6: I/O System
- Windows Internals, 7th Edition - Part 2, Chapter 8: System Mechanisms (KQUEUE)
- Mark Russinovich, "Inside Windows I/O Completion Ports"
- Microsoft Docs: [I/O Completion Ports Design Notes](https://learn.microsoft.com/en-us/windows/win32/fileio/i-o-completion-ports)
- ReactOS Source Code (ntoskrnl/ke/queue.c) - KQUEUE 구현 참조
- OSR Online: "Rolling Your Own - Building a Custom I/O Completion Port"
- Microsoft Docs: [IRPs](https://learn.microsoft.com/en-us/windows-hardware/drivers/kernel/irps)
- Microsoft Docs: [AFD.SYS IOCTL Reference](https://learn.microsoft.com/en-us/windows-hardware/drivers/network/) (부분 문서화)
