# Section 9: Visual Studio Code에서 IOCP 개발하기

> 작성 완료일: 2026-02-01

---

## 목차
- [9.1 개발 환경 구성](#91-개발-환경-구성)
- [9.2 MSVC 컴파일러 설정 (Build Tools)](#92-msvc-컴파일러-설정-build-tools)
- [9.3 CMake 기반 프로젝트 구성](#93-cmake-기반-프로젝트-구성)
- [9.4 tasks.json / launch.json 설정](#94-tasksjson--launchjson-설정)
- [9.5 디버깅 환경 구축](#95-디버깅-환경-구축)
- [9.6 유용한 확장(Extension) 목록](#96-유용한-확장extension-목록)
- [9.7 MinGW/Clang 대안 환경](#97-mingwclang-대안-환경)
- [9.8 실전 프로젝트 템플릿](#98-실전-프로젝트-템플릿)
- [9.9 트러블슈팅 가이드](#99-트러블슈팅-가이드)

---

## 9.1 개발 환경 구성

### 9.1.1 왜 VS Code인가

Visual Studio(Full IDE)가 Windows C++ 개발의 표준이지만, VS Code를 사용하는 이유:

```
VS Code 장점:
- 경량 (설치 ~300MB vs Visual Studio ~10GB+)
- 빠른 시작 속도
- 리모트 개발 지원 (SSH, WSL, Docker)
- 풍부한 확장 에코시스템
- 크로스 플랫폼 설정 경험 (Linux/Mac에서도 유사한 워크플로)
- Git 통합이 우수

제약:
- Windows API IntelliSense 설정이 수동
- 디버거가 Visual Studio만큼 풍부하지 않음 (메모리 뷰 등)
- IOCP 개발 시 Windows SDK 경로를 직접 설정해야 함
```

### 9.1.2 필수 소프트웨어 설치 순서

```
1. Visual Studio Build Tools (MSVC 컴파일러)
   → https://visualstudio.microsoft.com/visual-cpp-build-tools/
   → "C++ 빌드 도구" 워크로드 선택
   → Windows SDK 포함 (10.0.xxxxx)

2. VS Code
   → https://code.visualstudio.com/

3. VS Code 확장
   → C/C++ (Microsoft)
   → CMake Tools (Microsoft)
   → CMake (twxs)

4. CMake (선택, Build Tools에 포함되기도 함)
   → https://cmake.org/download/
```

---

## 9.2 MSVC 컴파일러 설정 (Build Tools)

### 9.2.1 Build Tools 설치 시 선택 항목

```
Visual Studio Build Tools 2022 설치 시:

필수 선택:
☑ MSVC v143 - VS 2022 C++ x64/x86 빌드 도구
☑ Windows 10/11 SDK (최신)
☑ C++ CMake tools for Windows

선택 권장:
☑ C++ ATL for latest build tools
☑ C++ AddressSanitizer (디버깅용)

총 설치 크기: ~3-5GB (Visual Studio 전체 대비 매우 작음)
```

### 9.2.2 환경 변수 설정

VS Code에서 MSVC를 사용하려면 컴파일러 환경을 활성화해야 한다.

```
방법 1: Developer Command Prompt에서 VS Code 실행 (권장)

시작 메뉴 → "x64 Native Tools Command Prompt for VS 2022"
> code .
→ 이 터미널에서 열면 cl.exe, link.exe 등이 PATH에 있음

방법 2: VS Code 터미널에서 vcvarsall.bat 호출

터미널에서:
> "C:\Program Files (x86)\Microsoft Visual Studio\2022\BuildTools\VC\Auxiliary\Build\vcvarsall.bat" x64

방법 3: tasks.json에서 자동 호출 (아래 섹션 참조)
```

### 9.2.3 컴파일러 경로 확인

```powershell
# Developer Command Prompt에서:
> where cl
C:\Program Files (x86)\Microsoft Visual Studio\2022\BuildTools\VC\Tools\MSVC\14.38.33130\bin\Hostx64\x64\cl.exe

> where link
C:\Program Files (x86)\Microsoft Visual Studio\2022\BuildTools\VC\Tools\MSVC\14.38.33130\bin\Hostx64\x64\link.exe

# Windows SDK 경로:
C:\Program Files (x86)\Windows Kits\10\Include\10.0.22621.0\
├── shared\    (공통 헤더)
├── um\        (User Mode 헤더 - winsock2.h 등)
├── ucrt\      (Universal CRT)
└── winrt\     (WinRT)
```

---

## 9.3 CMake 기반 프로젝트 구성

### 9.3.1 프로젝트 디렉토리 구조

```
iocp-server/
├── CMakeLists.txt          # 빌드 설정
├── .vscode/
│   ├── settings.json       # VS Code 프로젝트 설정
│   ├── tasks.json          # 빌드 작업
│   ├── launch.json         # 디버그 설정
│   └── c_cpp_properties.json  # IntelliSense 설정
├── src/
│   ├── main.cpp            # 엔트리 포인트
│   ├── iocp_server.h       # IOCP 서버 헤더
│   ├── iocp_server.cpp     # IOCP 서버 구현
│   ├── session.h
│   ├── session.cpp
│   ├── packet.h
│   └── packet.cpp
├── include/                # 공용 헤더
│   └── common.h
├── tests/                  # 테스트
│   └── test_server.cpp
└── build/                  # 빌드 출력 (gitignore)
```

### 9.3.2 CMakeLists.txt

```cmake
cmake_minimum_required(VERSION 3.20)
project(IOCPServer VERSION 1.0 LANGUAGES CXX)

# C++17 표준 사용
set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# Windows 전용 확인
if(NOT WIN32)
    message(FATAL_ERROR "IOCP is Windows-only. Use epoll/kqueue on other platforms.")
endif()

# 소스 파일
set(SOURCES
    src/main.cpp
    src/iocp_server.cpp
    src/session.cpp
    src/packet.cpp
)

# 실행 파일 생성
add_executable(${PROJECT_NAME} ${SOURCES})

# 헤더 경로
target_include_directories(${PROJECT_NAME} PRIVATE
    ${CMAKE_SOURCE_DIR}/include
    ${CMAKE_SOURCE_DIR}/src
)

# Winsock2 링크 (IOCP 필수)
target_link_libraries(${PROJECT_NAME} PRIVATE
    ws2_32       # Winsock2 (WSARecv, WSASend, etc.)
    mswsock      # Microsoft Winsock Extensions (AcceptEx 등 - WSAIoctl 대안)
)

# MSVC 전용 옵션
if(MSVC)
    target_compile_options(${PROJECT_NAME} PRIVATE
        /W4          # 경고 레벨 4
        /WX-         # 경고를 에러로 취급하지 않음 (개발 중)
        /MP          # 병렬 컴파일
        /Zi          # 디버그 정보 생성
        /utf-8       # UTF-8 소스 인코딩
    )

    # Debug/Release 설정
    target_compile_definitions(${PROJECT_NAME} PRIVATE
        _WIN32_WINNT=0x0A00    # Windows 10+
        WIN32_LEAN_AND_MEAN     # 불필요한 Windows 헤더 제외
        NOMINMAX                # min/max 매크로 비활성화
        _WINSOCK_DEPRECATED_NO_WARNINGS
    )

    # Debug 모드에서 AddressSanitizer 활성화 (선택)
    if(CMAKE_BUILD_TYPE STREQUAL "Debug")
        # target_compile_options(${PROJECT_NAME} PRIVATE /fsanitize=address)
    endif()
endif()

# 테스트 (선택)
option(BUILD_TESTS "Build tests" OFF)
if(BUILD_TESTS)
    enable_testing()
    add_executable(test_server tests/test_server.cpp)
    target_link_libraries(test_server PRIVATE ws2_32 mswsock)
    add_test(NAME ServerTest COMMAND test_server)
endif()
```

### 9.3.3 핵심 전처리 매크로 설명

```cpp
// _WIN32_WINNT: 최소 지원 Windows 버전 지정
// 이 값에 따라 사용 가능한 API가 달라짐
#define _WIN32_WINNT 0x0A00    // Windows 10
// 0x0601 = Windows 7
// 0x0602 = Windows 8    (RIO 사용 가능)
// 0x0603 = Windows 8.1
// 0x0A00 = Windows 10

// WIN32_LEAN_AND_MEAN: windows.h 경량화
// Cryptography, DDE, RPC, Shell, Winsock 1.x 등 제외
// Winsock2.h와의 충돌 방지에도 도움
#define WIN32_LEAN_AND_MEAN

// NOMINMAX: windows.h의 min/max 매크로 비활성화
// std::min, std::max와 충돌 방지
#define NOMINMAX

// 헤더 포함 순서 (중요!)
#include <winsock2.h>    // ← 반드시 windows.h보다 먼저!
#include <ws2tcpip.h>    // inet_pton, getaddrinfo 등
#include <mswsock.h>     // AcceptEx, TransmitFile 등
#include <windows.h>     // CreateIoCompletionPort 등
// winsock2.h와 windows.h 순서를 바꾸면 winsock.h(1.x)가 먼저 포함되어 에러
```

---

## 9.4 tasks.json / launch.json 설정

### 9.4.1 .vscode/settings.json

```json
{
    "cmake.configureOnOpen": true,
    "cmake.generator": "Ninja",
    "cmake.buildDirectory": "${workspaceFolder}/build",

    "C_Cpp.default.configurationProvider": "ms-vscode.cmake-tools",

    "files.associations": {
        "*.h": "cpp",
        "xmemory": "cpp",
        "type_traits": "cpp",
        "xstring": "cpp"
    },

    "terminal.integrated.defaultProfile.windows": "Command Prompt",

    "editor.formatOnSave": true,
    "C_Cpp.clang_format_style": "{ BasedOnStyle: Google, IndentWidth: 4 }"
}
```

### 9.4.2 .vscode/c_cpp_properties.json (IntelliSense)

```json
{
    "configurations": [
        {
            "name": "Win32-MSVC",
            "includePath": [
                "${workspaceFolder}/**",
                "${workspaceFolder}/include",
                "${workspaceFolder}/src"
            ],
            "defines": [
                "_DEBUG",
                "_WIN32_WINNT=0x0A00",
                "WIN32_LEAN_AND_MEAN",
                "NOMINMAX",
                "_WINSOCK_DEPRECATED_NO_WARNINGS"
            ],
            "windowsSdkVersion": "10.0.22621.0",
            "compilerPath": "C:/Program Files (x86)/Microsoft Visual Studio/2022/BuildTools/VC/Tools/MSVC/14.38.33130/bin/Hostx64/x64/cl.exe",
            "cStandard": "c17",
            "cppStandard": "c++17",
            "intelliSenseMode": "windows-msvc-x64",
            "configurationProvider": "ms-vscode.cmake-tools"
        }
    ],
    "version": 4
}
```

### 9.4.3 .vscode/tasks.json (빌드 작업)

```json
{
    "version": "2.0.0",
    "tasks": [
        {
            "label": "CMake: Configure",
            "type": "shell",
            "command": "cmake",
            "args": [
                "-S", ".",
                "-B", "build",
                "-G", "Ninja",
                "-DCMAKE_BUILD_TYPE=Debug"
            ],
            "group": "build",
            "problemMatcher": []
        },
        {
            "label": "CMake: Build",
            "type": "shell",
            "command": "cmake",
            "args": [
                "--build", "build",
                "--config", "Debug"
            ],
            "group": {
                "kind": "build",
                "isDefault": true
            },
            "problemMatcher": ["$msCompile"],
            "dependsOn": "CMake: Configure"
        },
        {
            "label": "CMake: Build Release",
            "type": "shell",
            "command": "cmake",
            "args": [
                "--build", "build",
                "--config", "Release"
            ],
            "group": "build",
            "problemMatcher": ["$msCompile"]
        },
        {
            "label": "CMake: Clean",
            "type": "shell",
            "command": "cmake",
            "args": ["--build", "build", "--target", "clean"],
            "group": "build",
            "problemMatcher": []
        },
        {
            "label": "Direct MSVC Build (no CMake)",
            "type": "shell",
            "command": "cl.exe",
            "args": [
                "/EHsc",
                "/Zi",
                "/Fe:build/iocp_server.exe",
                "/D_WIN32_WINNT=0x0A00",
                "/DWIN32_LEAN_AND_MEAN",
                "/DNOMINMAX",
                "/utf-8",
                "src/*.cpp",
                "/link",
                "ws2_32.lib",
                "mswsock.lib"
            ],
            "group": "build",
            "problemMatcher": ["$msCompile"]
        }
    ]
}
```

### 9.4.4 .vscode/launch.json (디버그 설정)

```json
{
    "version": "0.2.0",
    "configurations": [
        {
            "name": "IOCP Server (Debug)",
            "type": "cppvsdbg",
            "request": "launch",
            "program": "${workspaceFolder}/build/IOCPServer.exe",
            "args": [],
            "stopAtEntry": false,
            "cwd": "${workspaceFolder}",
            "environment": [],
            "console": "integratedTerminal",
            "preLaunchTask": "CMake: Build"
        },
        {
            "name": "IOCP Server (Attach)",
            "type": "cppvsdbg",
            "request": "attach",
            "processId": "${command:pickProcess}",
            "sourceFileMap": {
                "/build": "${workspaceFolder}/build"
            }
        },
        {
            "name": "Test Client",
            "type": "cppvsdbg",
            "request": "launch",
            "program": "${workspaceFolder}/build/test_client.exe",
            "args": ["127.0.0.1", "9000"],
            "console": "integratedTerminal"
        }
    ],
    "compounds": [
        {
            "name": "Server + Client",
            "configurations": ["IOCP Server (Debug)", "Test Client"],
            "stopAll": true
        }
    ]
}
```

---

## 9.5 디버깅 환경 구축

### 9.5.1 cppvsdbg vs cppdbg

```
VS Code에서 C++ 디버깅 시 두 가지 디버거 선택:

cppvsdbg (권장 for Windows):
  - Visual Studio Debugger 사용
  - MSVC로 빌드한 바이너리에 최적
  - 조건부 브레이크포인트, 데이터 브레이크포인트 지원
  - STL 컨테이너 네이티브 시각화
  - 편집 후 계속(Edit and Continue) 부분 지원

cppdbg (GDB/LLDB):
  - GCC/Clang으로 빌드한 바이너리에 사용
  - MinGW 환경에서 사용
  - MSVC 디버그 정보(PDB) 읽기 불가
```

### 9.5.2 IOCP 디버깅 팁

```
멀티스레드 디버깅:

1. Worker 스레드 식별
   - Debug Console에서: ~<thread_id>s (스레드 전환)
   - Call Stack 패널에서 스레드 목록 확인
   - 각 Worker에 이름 부여:

   // 스레드 이름 설정 (VS 디버거에서 표시됨)
   #include <processthreadsapi.h>
   SetThreadDescription(hThread, L"IOCP Worker #1");

2. 비동기 I/O 상태 확인
   - Watch 창에서 OVERLAPPED 구조체의 Internal 필드 확인
   - Internal == 0x103 (STATUS_PENDING) → 아직 진행 중
   - Internal == 0 (STATUS_SUCCESS) → 완료됨

3. 브레이크포인트 조건
   - 특정 세션에서만 멈추기:
     조건: pSession->m_id == 42
   - 특정 패킷 타입에서만:
     조건: pOvEx->opType == OP_RECV

4. 데이터 브레이크포인트 (메모리 변경 감시)
   - 세션 소켓이 변경될 때 감지:
     &pSession->m_socket 주소에 데이터 브레이크포인트 설정
```

### 9.5.3 메모리 누수 탐지

```cpp
// Debug 모드에서 CRT 메모리 누수 탐지 활성화
#ifdef _DEBUG
    #define _CRTDBG_MAP_ALLOC
    #include <crtdbg.h>
    #define new new(_NORMAL_BLOCK, __FILE__, __LINE__)
#endif

int main() {
    #ifdef _DEBUG
    _CrtSetDbgFlag(_CRTDBG_ALLOC_MEM_DF | _CRTDBG_LEAK_CHECK_DF);
    // 프로그램 종료 시 자동으로 메모리 누수 보고
    #endif

    // ... 서버 코드 ...
    return 0;
}

// 출력 예시 (Output 패널):
// Detected memory leaks!
// Dumping objects ->
// {150} normal block at 0x00E41050, 4096 bytes long.
//  Data: <                > CD CD CD CD CD CD CD CD
// Object dump complete.
```

---

## 9.6 유용한 확장(Extension) 목록

### 9.6.1 필수 확장

| 확장 | ID | 용도 |
|------|-----|------|
| C/C++ | ms-vscode.cpptools | IntelliSense, 디버깅, 코드 탐색 |
| CMake Tools | ms-vscode.cmake-tools | CMake 빌드/디버그 통합 |
| CMake | twxs.cmake | CMakeLists.txt 구문 강조 |

### 9.6.2 권장 확장

| 확장 | ID | 용도 |
|------|-----|------|
| C/C++ Extension Pack | ms-vscode.cpptools-extension-pack | C++ 확장 번들 |
| Clang-Format | xaver.clang-format | 코드 자동 포매팅 |
| Error Lens | usernamehw.errorlens | 에러를 코드 옆에 인라인 표시 |
| Todo Tree | gruntfuggly.todo-tree | TODO/FIXME 추적 |
| GitLens | eamodio.gitlens | Git 히스토리/블레임 |
| Hex Editor | ms-vscode.hexeditor | 바이너리/패킷 데이터 확인 |
| Serial Monitor | ms-vscode.vscode-serial-monitor | 소켓 데이터 모니터링 대안 |

### 9.6.3 IOCP 개발에 특히 유용한 확장

```
1. Doxygen Documentation Generator (cschlosser.doxdocgen)
   - /** 입력 시 자동 문서 템플릿 생성
   - IOCP 콜백 함수, 세션 클래스 문서화에 유용

2. Better C++ Syntax (jeff-hykin.better-cpp-syntax)
   - Windows API 타입(DWORD, HANDLE 등) 강조 개선

3. Task Explorer (spmeesseman.vscode-taskexplorer)
   - tasks.json의 빌드 작업을 사이드바에서 관리

4. C/C++ Themes (ms-vscode.cpptools-themes)
   - C++ 시맨틱 토큰 강조 개선
```

---

## 9.7 MinGW/Clang 대안 환경

### 9.7.1 MinGW-w64로 IOCP 개발

```
MinGW-w64는 GCC의 Windows 포트로, Winsock2/IOCP API를 지원한다.
단, 일부 최신 Windows API 헤더가 MSVC보다 뒤처질 수 있다.

설치:
1. MSYS2 (https://www.msys2.org/) 설치
2. pacman -S mingw-w64-ucrt-x86_64-gcc
3. pacman -S mingw-w64-ucrt-x86_64-cmake
4. pacman -S mingw-w64-ucrt-x86_64-ninja

VS Code에서 MinGW 사용:
settings.json:
{
    "cmake.generator": "Ninja",
    "cmake.cmakePath": "C:/msys64/ucrt64/bin/cmake.exe"
}
```

### 9.7.2 MinGW 빌드 명령

```bash
# 직접 빌드
g++ -std=c++17 -O2 -o iocp_server.exe src/*.cpp -lws2_32 -lmswsock

# 디버그 빌드
g++ -std=c++17 -g -O0 -o iocp_server_d.exe src/*.cpp -lws2_32 -lmswsock

# 주의: MinGW에서 Winsock 헤더 포함 순서
# #include <winsock2.h>  ← 먼저
# #include <windows.h>   ← 나중에
# MinGW에서도 이 순서는 동일하게 중요
```

### 9.7.3 MSVC vs MinGW 비교 (IOCP 개발)

| 항목 | MSVC | MinGW-w64 |
|------|------|-----------|
| Windows SDK 호환성 | 완벽 | 대부분 호환 (일부 최신 API 미지원) |
| IOCP API | 완벽 | 완벽 |
| RIO API | 완벽 | 헤더 부재 가능 (수동 정의 필요) |
| 디버거 | cppvsdbg (풍부) | GDB (기본적) |
| STL 구현 | MSVC STL | libstdc++ |
| 빌드 속도 | 보통 | 빠름 |
| PDB 디버그 정보 | O | X (DWARF 사용) |
| AddressSanitizer | O | O |
| 권장 대상 | 프로덕션 | 학습/실험 |

---

## 9.8 실전 프로젝트 템플릿

### 9.8.1 최소 프로젝트 빠른 시작

```
1. 폴더 생성 및 VS Code 열기

mkdir iocp-project && cd iocp-project
code .

2. CMakeLists.txt 생성 (위 섹션 참조)

3. src/main.cpp 생성:
```

```cpp
// src/main.cpp - IOCP 프로젝트 시작점
#define _WIN32_WINNT 0x0A00
#define WIN32_LEAN_AND_MEAN
#define NOMINMAX

#include <winsock2.h>
#include <ws2tcpip.h>
#include <mswsock.h>
#include <windows.h>
#include <cstdio>

#pragma comment(lib, "ws2_32.lib")
#pragma comment(lib, "mswsock.lib")

int main() {
    // Winsock 초기화
    WSADATA wsaData;
    if (WSAStartup(MAKEWORD(2, 2), &wsaData) != 0) {
        printf("WSAStartup failed\n");
        return 1;
    }

    // IOCP 생성
    HANDLE hIOCP = CreateIoCompletionPort(INVALID_HANDLE_VALUE, NULL, 0, 0);
    if (!hIOCP) {
        printf("CreateIoCompletionPort failed: %d\n", GetLastError());
        return 1;
    }

    printf("IOCP created successfully! Handle: %p\n", hIOCP);
    printf("IOCP development environment is working.\n");

    // 정리
    CloseHandle(hIOCP);
    WSACleanup();
    return 0;
}
```

```
4. VS Code에서:
   - Ctrl+Shift+P → "CMake: Configure"
   - Ctrl+Shift+B → 빌드
   - F5 → 디버그 실행

5. 출력 확인:
   IOCP created successfully! Handle: 0x000001A4
   IOCP development environment is working.
```

### 9.8.2 .gitignore 템플릿

```gitignore
# Build
build/
out/
x64/
Debug/
Release/

# MSVC
*.obj
*.pdb
*.ilk
*.exe
*.dll
*.lib
*.exp
*.idb

# VS Code (선택적 - 팀 공유 시 .vscode 포함 가능)
# .vscode/

# MinGW
*.o
*.d

# OS
Thumbs.db
Desktop.ini
.DS_Store
```

---

## 9.9 트러블슈팅 가이드

### 9.9.1 자주 발생하는 문제

| 문제 | 원인 | 해결 |
|------|------|------|
| `'winsock2.h' not found` | Windows SDK 경로 미설정 | c_cpp_properties.json에 windowsSdkVersion 설정, 또는 Developer Command Prompt에서 code 실행 |
| `LNK2019: unresolved external symbol` | ws2_32.lib 미링크 | CMakeLists.txt에 `target_link_libraries(... ws2_32 mswsock)` 추가 |
| `redefinition of 'struct sockaddr'` | winsock.h와 winsock2.h 충돌 | `#include <winsock2.h>`를 `<windows.h>` 전에 배치, `WIN32_LEAN_AND_MEAN` 정의 |
| IntelliSense 빨간줄 (빌드는 성공) | IntelliSense가 매크로/경로를 못 찾음 | c_cpp_properties.json의 defines와 includePath 확인 |
| `cl.exe is not recognized` | MSVC 환경 변수 미설정 | Developer Command Prompt에서 VS Code 실행 |
| 브레이크포인트 안 잡힘 | Release 빌드 또는 PDB 미생성 | Debug 빌드 확인, `/Zi` 플래그 확인 |
| UTF-8 소스 코드 깨짐 | MSVC 기본 인코딩이 시스템 로캘 | `/utf-8` 컴파일러 플래그 추가 |

### 9.9.2 winsock2.h / windows.h 순서 문제 해결

```cpp
// ❌ 잘못된 순서 (매우 흔한 실수)
#include <windows.h>     // 내부적으로 winsock.h (1.x) 포함
#include <winsock2.h>    // 이미 winsock.h가 포함되어 충돌!

// ✅ 올바른 순서
#include <winsock2.h>    // Winsock 2.x 먼저
#include <ws2tcpip.h>    // TCP/IP 확장
#include <mswsock.h>     // Microsoft 확장 (AcceptEx 등)
#include <windows.h>     // 나머지 Windows API

// ✅ 가장 안전한 방법: 프로젝트 전체 PCH(Precompiled Header)에서 관리
// pch.h:
#pragma once
#define _WIN32_WINNT 0x0A00
#define WIN32_LEAN_AND_MEAN
#define NOMINMAX
#include <winsock2.h>
#include <ws2tcpip.h>
#include <mswsock.h>
#include <windows.h>
```

---

## 참고 자료

- [VS Code C++ Documentation](https://code.visualstudio.com/docs/languages/cpp)
- [VS Code CMake Tools](https://github.com/microsoft/vscode-cmake-tools)
- [MSVC Command Line Reference](https://learn.microsoft.com/en-us/cpp/build/reference/compiler-options-listed-by-category)
- [CMake Documentation](https://cmake.org/cmake/help/latest/)
- [MSYS2 / MinGW-w64](https://www.msys2.org/)
