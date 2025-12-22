# Section 1: 도입 배경 및 문제 정의

## 1. 전통적 I/O 모델의 한계

### 1.1 Blocking I/O + Thread-per-connection 모델의 문제점

#### 메모리 오버헤드
| OS | 스레드당 기본 스택 크기 | 10,000 연결 시 메모리 |
|----|------------------------|----------------------|
| Linux (x86-64) | 8 MB | 80 GB |
| Windows | 1 MB | 10 GB |

- 스레드가 아무 작업도 하지 않아도 **최소 16 KB** 물리 메모리 사용
- 저메모리 시스템에서 생성 가능한 스레드 수 제한

#### 왜 중요한가
Thread-per-connection 모델은 직관적이고 구현이 쉽지만, 대규모 서비스(MMORPG, 채팅 서버)에서는 수만 개의 동시 접속 처리가 불가능합니다.

---

### 1.2 Context Switching 오버헤드

#### 직접 비용 (Direct Costs)

| 측정 항목 | 수치 |
|----------|------|
| 일반적인 Context Switch 시간 | 1.2 ~ 2.2 µs |
| CPU 핀닝 시 | 1.2 ~ 1.5 µs |
| 최악의 경우 | ~30 µs |
| CPU 사이클 (캐시 미스 포함) | 100,000 ~ 1,000,000 사이클 |

**실제 영향 예시:**
MySQL 서버가 초당 25,000번 Context Switching → CPU의 **18.75%가 스위칭에 낭비**

#### 간접 비용 (Indirect Costs)

| 비용 유형 | 수치 |
|----------|------|
| L1 캐시 히트 | 3 ~ 4 CPU 사이클 |
| 메인 메모리 접근 | ~200 CPU 사이클 (**50배 느림**) |
| TLB 미스 | 수백 사이클 (4~5번 메모리 접근) |

---

### 1.3 Select/Poll 모델의 한계

#### Select 모델 제약

| 제약사항 | 세부 내용 |
|---------|----------|
| FD_SETSIZE 제한 | Linux glibc: **1024개** 하드코딩 |
| 재설정 필요 | 매 호출마다 fd_set 재구성 |
| O(n) 순회 | 모든 FD 선형 검사 |
| 커널-유저 복사 | 매 호출마다 fd_set 복사 |

#### Poll 모델
- **개선점**: FD 개수 제한 제거
- **여전한 문제**: O(n) 복잡도, 커널-유저 복사 오버헤드

---

### 1.4 C10K Problem

#### 정의
**C10K Problem**: 단일 서버에서 **10,000개의 동시 연결**을 처리하는 최적화 문제
- 1999년 Dan Kegel이 명명
- "Apache Problem"으로도 알려짐

#### 발생 원인

1. **Thundering Herd 문제**
   - 여러 프로세스가 같은 listening 소켓 대기
   - 새 연결 도착 시 모든 프로세스 동시 깨어남
   - 하나만 accept() 성공, 나머지는 다시 sleep

2. **스레드 모델의 한계**
   - 스레드 증가 → Context Switching 시간 증가
   - CPU가 실제 작업보다 스위칭에 더 많은 시간 소비

3. **리소스 고갈**
   - 스레드당 메모리 오버헤드
   - 파일 디스크립터 한계

#### 해결책

| 해결 방법 | 설명 | 예시 |
|----------|------|------|
| Event-Driven | 단일 스레드 논블로킹 | Node.js, Nginx |
| I/O Multiplexing | epoll, kqueue, IOCP | Linux epoll, Windows IOCP |
| Thread Pooling | 고정 개수 워커 스레드 재사용 | 대부분의 서버 프레임워크 |

---

## 2. I/O 모델 비교

### 2.1 Synchronous vs Asynchronous I/O

| 특성 | Synchronous I/O | Asynchronous I/O |
|------|----------------|------------------|
| 제어 흐름 | 순차적 실행, 완료 대기 | 독립적 실행, 대기 없음 |
| 블로킹 여부 | 작업 완료까지 대기 | 즉시 반환, 나중에 통지 |
| 통지 방법 | 함수 반환으로 통지 | 콜백/이벤트로 통지 |
| 구현 난이도 | 쉬움 (직관적) | 어려움 (흐름 복잡) |

### 2.2 Blocking vs Non-blocking I/O

| 특성 | Blocking I/O | Non-blocking I/O |
|------|-------------|------------------|
| 호출 즉시 | 데이터 준비까지 대기 | 즉시 반환 |
| 반환값 | 실제 데이터 | 성공/실패/진행 중 |
| 스레드 상태 | 블로킹됨 | 계속 실행 가능 |

### 2.3 4가지 I/O 조합

| 모델 | 설명 | 사용 사례 |
|------|------|----------|
| Sync + Blocking | 전통적 read/write | 단순 스크립트 |
| Sync + Non-blocking | Polling | 비동기 도입 어려운 경우 |
| Async + Blocking | 비효율적 | 거의 사용 안 함 |
| **Async + Non-blocking** | 고성능 I/O | **대규모 네트워크 서버** |

---

## 3. Reactor 패턴 vs Proactor 패턴

### 핵심 차이

| 특성 | Reactor Pattern | Proactor Pattern |
|------|----------------|------------------|
| 통지 시점 | I/O **준비 완료** (Readiness) | I/O **작업 완료** (Completion) |
| 데이터 전송 | 애플리케이션이 직접 수행 | 커널이 직접 수행 |
| I/O 모델 | Synchronous I/O | Asynchronous I/O |
| 시스템 콜 | read/write 호출 필요 | read/write 호출 불필요 |

### 동작 방식 비교

**Reactor (epoll, kqueue):**
```
1. 소켓이 읽기 가능 상태가 됨 → 통지
2. 애플리케이션이 read() 호출
3. 데이터 복사 (커널 → 유저 공간)
4. 처리
```

**Proactor (IOCP, io_uring):**
```
1. 애플리케이션이 read() 예약 (버퍼 지정)
2. 커널이 데이터 도착 시 자동으로 버퍼에 복사
3. 완료 통지 → 데이터 이미 버퍼에 존재
4. 처리
```

---

## 4. IOCP가 해결하는 문제

### 4.1 Thread Pooling으로 스레드 수 최소화

| 문제 | IOCP 해결책 |
|------|------------|
| 10,000 연결 = 10,000 스레드 | 고정 크기 워커 스레드 풀 |
| 메모리 10~80 GB | CPU 코어 × 2 스레드만 사용 |
| Context Switching 18.75% | 캐시 지역성 보존 |

### 4.2 완료 기반 통지 (Completion-based Notification)

- **시스템 콜 감소**: read/write 호출 불필요
- **Zero-copy 가능**: NIC가 DMA로 유저 버퍼에 직접 전송
- **Spectre/Meltdown 완화**: 시스템 콜 오버헤드 회피

### 4.3 커널 레벨 최적화

1. **LIFO 스레드 선택**: CPU 캐시에 데이터 남아있을 확률 높음
2. **자동 스레드 스케줄링**: 블록 시 자동으로 다른 스레드 깨움
3. **DMA 직접 전송**: 메모리 복사 절약
4. **Thundering Herd 방지**: 완료 패킷당 1개 스레드만 깨움

---

## 5. 다른 OS의 유사 기술 비교

| 특성 | Linux epoll | BSD kqueue | Windows IOCP | Linux io_uring |
|------|------------|------------|--------------|----------------|
| I/O 모델 | Readiness | Readiness | **Completion** | **Completion** |
| 비동기 정도 | 부분적 | 부분적 | **완전** | **완전** |
| 도입 시기 | 2002 | 2000 | **1993** | 2019 |
| 복잡도 | O(1) | O(1) | O(1) | O(1) |
| 성능 | 높음 | 높음 | 높음 (CPU 효율 우수) | 가장 높음 (40% 향상) |

---

## 6. 퀴즈용 핵심 Q&A

### 정의형
**Q: C10K 문제란?**
A: 단일 서버에서 10,000개의 동시 연결을 효율적으로 처리하는 최적화 문제 (1999년 Dan Kegel 명명)

**Q: Reactor vs Proactor 핵심 차이?**
A: Reactor는 I/O 준비 완료(Readiness) 통지, Proactor는 I/O 작업 완료(Completion) 통지

### 수치형
**Q: Linux/Windows 기본 스레드 스택 크기?**
A: Linux = 8 MB, Windows = 1 MB

**Q: Context Switch 소요 시간?**
A: 일반 1.2~2.2 µs, CPU 사이클 100,000~1,000,000 (캐시 무효화 포함)

**Q: Select FD_SETSIZE 기본 제한?**
A: Linux glibc에서 1024개 하드코딩

### 적용형
**Q: 8코어 CPU에서 IOCP 워커 스레드 권장 개수?**
A: CPU 코어 × 2 = 16개

**Q: 10,000명 동시접속을 Thread-per-connection으로 처리 시 Linux 메모리?**
A: 10,000 × 8 MB = 80 GB (스택만)
