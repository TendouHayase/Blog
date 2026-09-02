---
title: 시스템 콜 시 동작
published: 2026-09-02
description: '어셈블리 명령어 syscall 시와 그 전후 cpu와 레지스터 동작 및 os 동작'
image: ''
tags: ["Kernel", "System"]
category: 'Knowledge'
draft: false
lang: 'ko'
---

# 들어가며

사용자 프로세스는 커널 레벨의 서비스를 직접 호출할 수 없습니다.
사용자 모드(ring 3)에서 커널 레벨의 공간(ring 0)으로의 접근은 하드웨어 적으로 차단되므로, 파일 입출력이나 IO 요청, 메모리 할당같은 요청은 커널에게 수행을 요청해야하며 이렇게 미리 지정된 인터페이스를 시스템 콜(system call[^1])이라 하고, x86-64 아키텍처에서 이 경로를 담당하는 어셈블리 명령어는 syscall입니다.

이 syscall 명령어의 하드웨어 명세 자체는 작습니다. 세그먼트 전환과 RIP 이동과 RFLAGS 이동이 전부이며 스택 전환, 주소 공간 전환, 인자 검증 등은 이 명령어에 포함되진 않습니다.
시스템 콜의 실제 동작은 이 명령어와 이 명령어 전후의 os가 구축한 나머지 명령이 결합되어 이루어집니다.

이 글에서는 syscall의 동작과 전후 레지스터, 메모리 상태를 보기위해 linux는 오픈소스 참조, 윈도우는 2가지 가상머신을 사용하여 디버깅하였습니다.
hyper-V로 띄운 윈도우(Windows 10 22H2 19045.3803)는 커널레벨 디버깅 및 관련 구조체를 보기위해 사용하였고 Windbg를 사용하였고 다른 하나는 QEMU(version 11.1)를 이용해 띄운 윈도우(Windows 10 Pro 22H2 19045.3803)는 syscall 직후부터 커널 레벨로 각종 세팅이 전환되기전에 브레이크 포인트(소프트웨어, 하드웨어 bp 전부)를 걸 시 Hyper-V와 Windbg가 전부 뻗어 syscall 직후 상황을 관찰하기위해 사용하였습니다.

# printf부터 시스템 콜까지

디버깅은 아래와 같은 프로그램을 사용하였습니다.

```c
// main.c
#include <stdio.h>

int main(void)
{
    printf("asdf\n");
    return 0;
}
```

## WinDbg

`ntdll!NtWriteFile`에 브레이크 포인트를 걸고 콜스택을 덤프합니다.

```text
0:000> bp ntdll!NtWriteFile
0:000> g

 00  ntdll!NtWriteFile
 01  KERNELBASE!WriteFile
 02  ucrtbased!write_text_ansi_nolock
 03  ucrtbased!_write_nolock
 04  ucrtbased!_write_internal
 05  ucrtbased!__acrt_stdio_flush_nolock
 06  ucrtbased!__acrt_stdio_end_temporary_buffering_nolock
 07  ucrtbased!__acrt_stdio_temporary_buffering_guard::~__acrt_stdio_temporary_buffering_nolock
 08  ucrtbased!`lambda_303760bc…'::operator()
 09  ucrtbased!__crt_seh_guarded_call<int>::operator()
 0a  ucrtbased!__acrt_lock_stream_and_call
 0b  ucrtbased!common_vfprintf<__crt_stdio_output::standard_base,char>
 0c  ucrtbased!__stdio_common_vfprintf
 0d  test!_vfprintf_l
 0e  test!printf
 0f  test!main
```

printf에서 시작한 호출은 열여섯 개의 프레임을 거쳐 이 함수에 도달했고, 그 안에 syscall이 있습니다. (test는 실행한 모듈명(exe 프로그램명))

```asm

0:000> uf ntdll!NtWriteFile
ntdll!NtWriteFile:
  mov     r10, rcx
  mov     eax, 8
  test    byte ptr [7FFE0308h], 1
  jne     ntdll!NtWriteFile+0x15
  syscall
  ret
ntdll!NtWriteFile+0x15:
  int     2Eh
  ret
```

NtWriteFile에서는 위와 같이 r10에 rcx를 넣고 eax에 콜 번호를 대입합니다.

## Linux — gdb로 추적

같은 프로그램을 WSL2 ArchLinux에서 gdb의 catch syscall로 관찰하였습니다.

```text
 $ gdb -q ./main
(gdb) catch syscall write
(gdb) run
(gdb) bt

 #0  internal_syscall_cancel
 #1  syscall_cancel
 #2  __GI___libc_write
 #3  _IO_new_file_write
 #4  new_do_write
 #5  _IO_new_do_write
 #6  _IO_new_file_overflow
 #7  __GI__IO_puts
 #8  main
```

> printf가 안보이는 이유는 GCC가 포맷 지정자를 사용하지 않는 printf를 puts 호출로 접기 때문입니다(`-fno-builtin`으로 억제 가능).

> `#0`과 `#1`(syscall_cancel, internal_syscall_cancel)은 실제 호출이 아닙니다. glibc가 취소 가능 시스템 콜 코드를 `__libc_write` 안에 인라인했고, gdb가 디버그 정보를 근거로 인라인된 논리 프레임을 펼쳐 보여주는 것입니다. info frame으로 확인하면 두 프레임이 `__libc_write`에 inlined되어 있음을 알 수 있고, 아래 덤프의 모든 주소가 `__GI___libc_write+offset`으로 표시되는 것도 같은 이유입니다

위에는 생략되었지만 main을 제외하고 콜스택의 모듈들은 libc 하나입니다.
이것은 `frame 번호`와 `info symbol`을 이용해 확인할 수 있습니다.
즉 stdio 구현(`_IO_*` 프레임)과 시스템 콜 래퍼(`__GI___libc_write`)가 같은 모듈 안에 접혀 있습니다.
glibc가 CRT와 래퍼의 역할을 모두 수행하므로 Windows의 KERNELBASE, ntdll에 해당하는 층이 별도 모듈로 존재하지 않습니다.

syscall이 속한 함수의 어셈블리입니다(앞부분 일부 생략).

```asm
pop    %rsi
cmp    $0xfffffffffffffffc, %rdx
jne    <__GI___libc_write+171>
and    $0x39, %ecx
cmp    $0x8, %ecx
je     <__GI___libc_write+196>
mov    $0x4, %edx
mov    0x1065ae(%rip), %rax
mov    %edx, %fs:(%rax)
mov    $0xffffffffffffffff, %rdx
leave
mov    %rdx, %rax
ret
nopl   0x0(%rax)
xor    %r9d, %r9d
xor    %r8d, %r8d
xor    %r10d, %r10d
mov    $0x1, %eax
syscall
mov    %rax, %rdx
cmp    $0xfffffffffffff000, %rax
ja     <__GI___libc_write+192>
leave
mov    %rdx, %rax
ret
nopl   0x0(%rax,%rax,1)
neg    %edx
jmp    <__GI___libc_write+123>
call   <__syscall_do_cancel>
```

syscall 직전에 세 개의 xor와 하나의 mov가 있고, 직후에는 반환값 비교가 있습니다.

## 두 스택의 공통 구조

표 1. printf에서 시스템 콜까지의 계층 대응

| 계층           | Windows              | Linux                |
| -------------- | -------------------- | -------------------- |
| OS API 층      | KERNELBASE!WriteFile | (없음)               |
| 시스템 콜 래퍼 | ntdll!NtWriteFile    | libc 내 __libc_write |
| 경계           | syscall              | syscall              |

Windows 스택이 한 층 더 깊은 것은 Win32 API가 NT(Native API) 위의 하위 시스템으로 설계된 역사의 결과이고, Linux는 stdio가 래퍼를 직접 호출합니다.

두 스택에서 공통적으로 확인되는 사실은 세 가지입니다.

- 경계 직전까지의 모든 계층이 준비를 수행하며 실제 출력은 일어나지 않았습니다.
- 두 경로 모두 syscall을 포함한 짧은 함수에서 끝납니다 - (NtWriteFile, __libc_write)
- 두 경계 함수 모두 syscall만으로 이루어져 있지 않습니다.
-

Windows 쪽은 syscall을 포함해 아홉 줄, Linux 쪽은 세 배가 넘는 코드로 구성되어 있습니다.

## syscall 전의 준비

1장의 두 덤프에서 syscall 직전에는 이미 명령어들이 존재했습니다.
여기서 그 구간을 분석합니다.
Windows의 stub은 모든 `Nt*` 함수가 공유하는 정적 템플릿이고, Linux의 래퍼는 취소라는 정책을 수행하는 실제 함수입니다.

### Windows — ntdll의 stub

ntdll이 export하는 `Nt*` 함수는 본체가 없는 짧은 코드 조각이라는 의미에서 stub이라고 불립니다. 위 어셈블리 stub에 역할을 달면 다음과 같습니다.

```asm
mov     r10, rcx                ; 1. 첫 인자를 R10으로 이동
mov     eax, 8                  ; 2. 시스템 콜 번호(SSN) 적재
test    byte ptr [7FFE0308h], 1 ; 3. 진입 방식 플래그 검사
jne     ntdll!NtWriteFile+0x15  ;    1이면 5. 로
syscall                         ; 4. 커널 진입
ret                             ;    반환값을 그대로 돌려줌
int     2Eh                     ; 5. 레거시 진입 경로
ret
nop     dword ptr [rax+rax]     ; 정렬 패딩
```

#### `mov r10, rcx`

Microsoft x64 호출 규약에서 첫 번째 인자는 RCX로 전달됩니다. 호출자(`KERNELBASE`의 `WriteFile` 구현)는 규약에 따라 인자를 RCX, RDX, R8, R9에 실었고, stub은 그중 첫 인자만 R10으로 옮깁니다.
나머지 인자 레지스터는 건드리지 않습니다.
여기서 인자의 배치는 전적으로 stub의 호출자의 책임이고 stub은 첫 인자의 자리 이동만 담당합니다.
추가로 이 복사가 `3`의 검사보다 앞에 있다는 것을 볼 수 있는데 이것은 진입 방식과 무관하게 커널이 첫 인자를 R10에서 읽는다는 뜻입니다.

#### `mov eax, 8`

이 번호는 커널의 시스템 콜 테이블(SSDT) 인덱스로 소비되며 정확한 것은 뒤에 나오지만 여기서 중요한 것은 번호의 의미입니다.
SSN(System Service Number)은 공식 문서에 존재하지 않을 뿐 아니라 빌드마다 달라지는 것으로 SSDT에서 어떤 커널 함수를 호출할지를 결정합니다.
이 SSN은 ABI가 아니라 내부 구현 세부사항으로 버전마다 변경될 수 있습니다.
이것은 Linux와 정면으로 대비되는데 x86-64 Linux에서 `write`의 번호는 1로 고정되어 있고, 헤더로 공개되어 있는걸 볼 수 있습니다.

```ext
// WSL2 아치리눅스 /usr/include/asm/unistd_64.h 내부
#ifndef _ASM_UNISTD_64_H
#define _ASM_UNISTD_64_H

#define __NR_read 0
#define __NR_write 1
#define __NR_open 2
#define __NR_close 3
```

위와 같이 리눅스에서 콜 번호는 고정되어 사용자 공간 프로그램이 이 번호에 의존하는 것이 ABI로 보장됩니다.

#### `test byte ptr [7FFE0308h]`

이 주소는 우연한 값이 아닙니다.
모든 프로세스의 주소 공간에는 `0x7FFE0000`에 동일한 물리 페이지가 매핑되어 있습니다.
KUSER_SHARED_DATA라 불리는 이 페이지는 커널이 정보를 기록하고 사용자 모드가 읽는 단방향 게시판으로, 시스템 시간이나 틱 카운트가 여기에 게시됩니다.
시스템 시간을 읽기 위해 매번 시스템 콜을 수행하지 않아도 되는 이유가 이 페이지입니다.
stub은 이 페이지 `0x308` 오프셋의 0번 비트를 검사합니다.
0이면 `syscall`으로, 1이면 `int 2Eh`로 진입합니다.
이 필드는 공식 문서에 존재하지 않습니다.
네이티브 환경에서 `db 7FFE0308 L1`로 이 바이트를 읽으면 0이 관찰됩니다.
역사적으로 x86 Windows의 stub은 이 페이지 `0x300` 오프셋에 게시된 sysenter 진입 루틴의 주소를 `call [0x7FFE0300]`으로 호출하는 방식이었습니다.
또한 Windows 10 과거 버전까지의 x64 stub은 `mov r10, rcx` / `mov eax, SSN` / `syscall` / `ret` 네 줄이 전부였습니다.
검사와 `int 2Eh` 폴백이 들어온 것은 최신 빌드의 변화입니다.

#### `syscall` / `ret, int 2Eh` / `ret. syscall`

정리하면 stub의 전체 역할은 번호 적재, 첫 인자 이동, 진입 방식 선택, 실행과 복귀의 네 가지입니다.
ntdll이 export하는 수백 개의 `Nt*` 함수가 모두 이 템플릿의 복제본이고, 서로 다른 것은 2의 상수뿐이며 크기와 명령 배치까지 동일합니다.

### Linux — glibc의 래퍼

`__libc_write`는 stub이 아니라 함수입니다.
덤프에서 syscall 실행과 직접 관련된 것은 다섯 줄로 정리됩니다.

```asm
xor    %r9d, %r9d     ; 6번째 인자 슬롯 → 0
xor    %r8d, %r8d     ; 5번째 인자 슬롯 → 0
xor    %r10d, %r10d   ; 4번째 인자 슬롯 → 0
mov    $0x1, %eax     ; __NR_write = 1
syscall
```

RDI, RSI, RDX가 이 다섯 줄에 없는 것은 이미 값이 자리에 있기 때문입니다.
호출자(`_IO_new_file_write`)가 System V AMD64 규약에 따라 인자를 RDI, RSI, RDX에 실어 호출했고, x86-64 Linux의 시스템 콜 규약도 처음 세 인자를 같은 레지스터에서 읽습니다.
규약끼리 간극을 최소화하도록 설계되어 있으므로, `write`처럼 인자가 세 개인 시스템 콜은 이동 한 줄 없이 통과합니다. EAX의 1은 앞서 확인한 `__NR_write` 입니다.

세 개의 xor는 미사용 인자 슬롯을 0으로 만듭니다.
glibc의 취소 가능 시스템 콜 경로는 여섯 인자를 고정으로 받는 구조를 사용하며, 컴파일러가 미사용 인자를 0으로 전달합니다.
결과적으로 커널에 전달되는 레지스터 상태는 항상 결정적입니다.

그리고 여기서 네 번째 슬롯이 R10이라는 사실이 드러납니다.
System V 규약의 네 번째 인자 레지스터는 RCX인데, 시스템 콜 규약은 이 자리를 R10으로 옮겨 놓았습니다.
Linux는 규약 수준에서 네 번째 자리를 이동했고, Windows는 MS x64 규약을 유지한 채 stub에서 매번 복사했습니다.

다섯 줄을 둘러싼 나머지 코드는 대부분 취소(cancellation) 처리입니다.
덤프에는 인자를 스택에 피신했다가 복원하는 코드(`pop %rsi` — 취소 검사가 인자 레지스터를 사용하기 때문), 스레드 제어 블록(`%fs` 상대 주소)의 취소 상태를 검사하는 코드, 실제 취소를 수행하는 `__syscall_do_cancel` 호출등이 포함되어 있습니다.

syscall 이후의 세 줄은 반환값 처리입니다.

```asm

mov    %rax, %rdx
cmp    $0xfffffffffffff000, %rax
ja     …               ; 오류 범위면 점프
```

Linux 커널은 오류를 -4095 ~ -1 범위의 음수 값으로 RAX에 돌려줍니다.
래퍼는 RAX가 이 범위에 있으면 값을 부정해 errno로 저장하고 -1을 반환합니다.
C의 오류 규약인 errno가 이 지점에서 만들어집니다.
대조적으로 Windows stub은 RAX를 가공 없이 반환하고, NTSTATUS는 사용자 코드까지 그대로 올라갑니다.

## 템플릿과 정책 함수

표 2. 두 시스템 콜 래퍼의 대조

| 항목      | Windows — ntdll stub                  | Linux — glibc 래퍼                    |
| --------- | ------------------------------------- | ------------------------------------- |
| 정체      | 모든 Nt*가 공유하는 정적 템플         | 취소 정책을 수행하는 함수             |
| 인자      | 첫 인자만 RCX→R10 이동, 나머지 불간섭 | 첫 3개 통과, 미사용 슬롯 0으로 클리어 |
| 번호      | SSN — 빌드별 재배치, 미문서화         | `__NR_*` — ABI로 고정, 헤더 공개      |
| 반환값    | NTSTATUS 무가공 반환                  | -errno 검출 → -1/errno 변환           |
| 진입 방식 | KUSER_SHARED_DATA 플래그로 선택       | 항상 syscall                          |

두 덤프 모두 R10를 수정합니다.
Windows는 첫 인자를 R10으로 복사했고, Linux는 네 번째 슬롯을 처음부터 R10으로 사용했습니다.
그리고 두 규약 모두 RCX를 시스템 콜 인자로 사용하지 않습니다.
syscall 직전의 상태를 정리하면 다음과 같습니다.

표 3. syscall 직전의 레지스터 상태

| 레지스터 | Windows (NtWriteFile)             | Linux (write)     |
| -------- | --------------------------------- | ----------------- |
| RAX      | 8 (SSN)                           | 1 (__NR_write)    |
| RCX      | FileHandle — ①에서 R10으로 복사됨 | 미사용            |
| RDX      | Event (NULL)                      | 5 — 버퍼 길이[^2] |
| R8       | ApcRoutine (NULL)                 | 0                 |
| R9       | ApcContext (NULL)                 | 0                 |
| R10      | FileHandle                        | 0                 |
| RDI      | -                                 | 1 (stdout fd)     |
| RSI      | -                                 | 버퍼 주소         |

추가로 NtWriteFile은 아홉 개의 인자를 받는 함수이므로, IoStatusBlock, Buffer, Length를 포함한 나머지 다섯 인자는 레지스터가 아닌 사용자 모드 스택을 통해 전달됩니다.
뒤에 나오지만 시스템 콜 테이블의 엔트리에 스택 인자 개수가 인코딩되어 있는 이유가 이것입니다.

# syscall 실행

이 장은 syscall 한 줄이 실행되는 순간의 동작을 다룹니다.

## 실행 순간

syscall 한 번이 수행하는 동작은 다음 여섯 항목으로 정리됩니다.

표 5. syscall 실행 전후의 규격 동작

| 동작                               | 실행 전 → 후         |
| ---------------------------------- | -------------------- |
| RCX ← 다음 명령어 주소             | 복귀 주소를 저장     |
| R11 ← RFLAGS                       | 플래그 사본을 저장   |
| RIP ← IA32_LSTAR                   | 진입점으로 이동      |
| CS, SS ← IA32_STAR에서 유도        | CPL 3 → 0            |
| RFLAGS ← RFLAGS AND NOT IA32_FMASK | 마스크된 비트 클리어 |
| RSP, CR3, GS                       | 변화 없음            |

여기서 주의할 것이 하나 있는데 RFLAGS 갱신이 "IF를 지운다" 같은 고정 규격이 아니라 FMASK에 심은 값에 따른 마스크라는 점입니다.
진입 직후 인터럽트가 차단되는가는 하드웨어가 아니라 운영체제의 MSR(Model Specific Register) 설정이 결정합니다.

복귀 정보를 스택이 아니라 RCX/R11에 남긴다는 점이 가장 중요합니다.

> sysret의 규격은 이 동작의 거꾸로입니다. RIP ← RCX, RFLAGS ← R11, CS ← STAR에서 유도(CPL 3).

## 실험

표 5의 서술을 실험으로 확인합니다.
환경은 QEMU와 gdb 스텁입니다.
gdb를 이용해 stub의 syscall 직전에 멈추고 si로 명령어를 통과시킨 뒤, monitor info registers로 가상 머신의 전체 레지스터를 비교했습니다.

### 관찰 1 - 전후 상태

```text
syscall 직전                       syscall 직후
RIP = 0x7ff9064ad662 (stub)      RIP = 0xfffff80416210d40
CS  = 0x33 (DPL 3)               CS  = 0x10 (DPL 0)
SS  = 0x2b                       SS  = 0x18
RSP = 0x7dd29ff878               RSP = 0x7dd29ff878
CR3 = 0x85b12000                 CR3 = 0x85b12000
RCX = 0x0                        RCX = 0x7ff9064ad664
R11 = 0x0                        R11 = 0x246
GS  = 0x7dd2604000               GS  = 0x7dd2604000
```

RIP는 커널 주소(`0xfffff804...`)로 이동했고, 그 주소는 rdmsr로 읽은 LSTAR 값입니다.
RCX는 syscall이 있던 0x7ff9064ad662의 다음 명령 주소(`0x664`, stub의 ret)로 채워졌고, R11은 직전의 RFLAGS를 보관하고 있습니다.
CS/SS는 커널 세그먼트(`0x10`, `0x18`)로 교체되었습니다.
RSP, CR3, GS는 변하지 않았습니다.

## MSR

위 동작을 보면 `IA32_LSTAR`과 `IA32_FMASK`라는 새로운 용어가 등장합니다.
이것들은 MSR라 불리는 레지스터 중 하나로 x86 아키텍처에서 CPU 모델별 기능 제어와 상태 조회를 위해 제공되는 특수 레지스터 집합입니다.
범용 레지스터와 달리 명령어 피연산자로 직접 접근할 수 없고, 전용 명령을 통해서만 읽고 씁니다.
각 MSR들은 32bit 주소와 64bit 값을 가지며 주소 공간은 모델마다 다릅니다.
하지만 일부 영역은 아키텍처적으로 고정(예: `0xC0000000~` 영역 — x86-64 확장 MSR)됩니다.

`IA32_LSTAR`의 경우 syscall 직후 진입할 목적지 RIP를 저장하는 MSR이고 `IA32_FMASK`는 `syscall` 실행 시 `RFLAGS` 레지스터에서 클리어할 비트 마스크를 저장하는 MSR입니다.[^3]

표 4. syscall이 사용하는 MSR

| MSR        | 주소       | 역할                               |
| ---------- | ---------- | ---------------------------------- |
| IA32_STAR  | 0xC0000081 | CS/SS가 유도되는 세그먼트 기준     |
| IA32_LSTAR | 0xC0000082 | 진입점 주소 — RIP가 이 값으로 이동 |
| IA32_FMASK | 0xC0000084 | RFLAGS에서 지울 비트의 마스크      |

Linux는 부팅시 `syscall_init()`[^4]에서 세 MSR을 설정합니다.

```c
// arch/x86/kernel/cpu/common.c
static inline void idt_syscall_init(void)
{
 wrmsrq(MSR_LSTAR, (unsigned long)entry_SYSCALL_64);

 if (ia32_enabled()) {
  wrmsrq_cstar((unsigned long)entry_SYSCALL_compat);
  /*
   * This only works on Intel CPUs.
   * On AMD CPUs these MSRs are 32-bit, CPU truncates MSR_IA32_SYSENTER_EIP.
   * This does not cause SYSENTER to jump to the wrong location, because
   * AMD doesn't allow SYSENTER in long mode (either 32- or 64-bit).
   */
  wrmsrq_safe(MSR_IA32_SYSENTER_CS, (u64)__KERNEL_CS);
  wrmsrq_safe(MSR_IA32_SYSENTER_ESP,
       (unsigned long)(cpu_entry_stack(smp_processor_id()) + 1));
  wrmsrq_safe(MSR_IA32_SYSENTER_EIP, (u64)entry_SYSENTER_compat);
 } else {
  wrmsrq_cstar((unsigned long)entry_SYSCALL32_ignore);
  wrmsrq_safe(MSR_IA32_SYSENTER_CS, (u64)GDT_ENTRY_INVALID_SEG);
  wrmsrq_safe(MSR_IA32_SYSENTER_ESP, 0ULL);
  wrmsrq_safe(MSR_IA32_SYSENTER_EIP, 0ULL);
 }

 /*
  * Flags to clear on syscall; clear as much as possible
  * to minimize user space-kernel interference.
  */
 wrmsrq(MSR_SYSCALL_MASK,
        X86_EFLAGS_CF|X86_EFLAGS_PF|X86_EFLAGS_AF|
        X86_EFLAGS_ZF|X86_EFLAGS_SF|X86_EFLAGS_TF|
        X86_EFLAGS_IF|X86_EFLAGS_DF|X86_EFLAGS_OF|
        X86_EFLAGS_IOPL|X86_EFLAGS_NT|X86_EFLAGS_RF|
        X86_EFLAGS_AC|X86_EFLAGS_ID);
}

/* May not be marked __init: used by software suspend */
void syscall_init(void)
{
 struct msr val = { .h = (__USER32_CS << 16) | __KERNEL_CS };

 /* The default user and kernel segments */
 wrmsrq(MSR_STAR, val.q);

 /*
  * Except the IA32_STAR MSR, there is NO need to setup SYSCALL and
  * SYSENTER MSRs for FRED, because FRED uses the ring 3 FRED
  * entrypoint for SYSCALL and SYSENTER, and ERETU is the only legit
  * instruction to return to ring 3 (both sysexit and sysret cause
  * #UD when FRED is enabled).
  */
 if (!cpu_feature_enabled(X86_FEATURE_FRED))
  idt_syscall_init();
}
```

위에 보듯 LSTAR에 entry_SYSCALL_64가 기록됩니다.
Windows의 LSTAR는 커널 디버거로 직접 읽을 수 있습니다.

```text
kd> rdmsr c0000082
msr[c0000082] = fffff801`e74bf640
kd> u fffff801e74bf640 L1
nt!KiSystemCall64:
fffff801`e74bf640 0f01f8          swapgs
```

MSR을 쓰는 설계의 이유를 INT 명령과 비교하면 알 수 있습니다.
INT 0x80은 벡터 번호로 IDT(Interrupt descriptor table)[^5]를 조회하고 게이트의 권한을 검사한 뒤 TSS에서 커널 스택을 읽어와 전환하고, 복귀 정보를 그 스택에 push합니다.
syscall은 이 조회·검사·전환·적립을 전부 수행하지 않습니다.
CPU 내부 레지스터에 진입 정보를 고정해 둠으로써 그 비용을 없앤 것이고, 대가로 안전장치(스택 전환, 프레임 적립)도 사라졌으므로 그 몫은 운영체제로 넘어갑니다.

## R10 레지스터

위에서 언급하지 않은 것 중 2가지가 있는데 `Windows stub의 첫 줄은 왜 첫 인자를 RCX에서 R10으로 옮기는가, 그리고 Linux 시스템 콜 규약의 네 번째 인자는 왜 R8이 아니라 R10인가`에 대해 말해보면 먼저 위처럼 RCX 레지스터는 syscall시 값이 덮어씌워지게 됩니다.

하지만 두 표준 호출 규약은 모두 RCX를 인자 레지스터로 사용하는데 System V AMD64에서는 네 번째, Microsoft x64에서는 첫 번째 인자의 자리입니다.
그래서 두 OS 모두 RCX의 인자를 어딘가로 옮겨야 하는데, 옮길 수 있는 곳이 사실상 정해져 있습니다.
범용 레지스터 중에서 인자도 아니고 반환값도 아니고 callee-saved도 아닌 것은 R10과 R11뿐인데, R11은 syscall이 RFLAGS 사본을 저장하는 데 씁니다.
따라서 R10 레지스터를 사용합니다.

하지만 이렇게 옮기면 기존의 함수 호출 규약과는 맞지 않게됩니다.

### Linux

이걸 Linux는 커널이 경계에서 읽는 함수 호출 규약 자체를 수정했습니다.

```text
System V AMD64 (C 규약): rdi rsi rdx rcx r8 r9
Linux 시스템 콜 (커널 규약): rdi rsi rdx r10 r8 r9
```

System V 규약에서 네 번째 인자만 RCX를 R10으로 바꾼 형태입니다.
이 배치는 `psABI`의 커널 인터페이스 항목에 명시되어 있어, `libc`를 거치지 않고 syscall을 직접 발행하는 코드는 네 번째 인자를 R10에 놓기만 하면 됩니다.
인자가 세 개 이하인 시스템 콜은 규약의 차이와 무관하게 지나갑니다.
2장의 `write` 덤프에서 R10 관련 코드가 클리어(`xor %r10d, %r10d`) 하나뿐이었던 것이 이 때문입니다.

인자가 네 개 이상이면 상황이 달라집니다. 래퍼는 C 함수이므로 네 번째 인자를 RCX로 받지만, 경계에서 그 값은 R10에 있어야 합니다.
이동은 `libc` 래퍼가 합니다.
`mmap`이나 `pselect6` 같은 시스템 콜의 래퍼를 열어보면 syscall 직전에 `mov %rcx, %r10`이 있는데, Windows stub과 같은 명령입니다.
Linux는 이동을 없앤 게 아니라 규약 안으로 접어넣었고, 인자가 적은 호출은 이 이동을 하지않는 구조입니다.

> 참고: 이 이동은 직접 확인할 수 있습니다. `mmap`처럼 취소점이 아니면서 인자가 여섯 개인 시스템 콜의 glibc 래퍼를 디스어셈블하면 syscall 직전의 `mov %rcx, %r10`이 보입니다.

### Windows

Microsoft x64는 컴파일러, DLL 경계, 디버거까지 Windows 전반이 의존하는 규약이라 건드리지 않았습니다.
대신 모든 ntdll stub이 첫 줄에서 `mov r10, rcx`를 수행하고, 커널의 진입 코드는 첫 번째 인자를 R10에서 읽습니다.

```text
Microsoft x64 (C 규약): rcx rdx r8 r9
Windows 커널 경계 규약: r10 rdx r8 r9
```

System V에서는 네 번째였던 수정 지점이 Microsoft x64에서는 첫 번째입니다.
그리고 첫 번째 인자는 모든 시스템 콜에 존재하므로, 이 3바이트 명령(4C 8B D1)은 시스템의 모든 시스템 콜이 매번 실행합니다.

`mov r10, rcx`복사가 진입 방식 분기보다 앞서 있다는 점의 이유를 알 수 있는데 syscall 경로에서는 진입 시점의 RCX가 이미 복귀 주소이므로, 커널이 첫 인자를 RCX에서 읽는 것은 애초에 불가능합니다.
커널의 규약이 R10이라면 `int 2Eh` 경로로 들어온 호출에서도 R10이 유효해야 하고, 그러려면 복사는 분기 이전에 끝나 있어야 합니다.

## syscall이 하지 않는 일

```text
스택 전환 — RSP는 유저 스택 그대로
주소 공간 전환 — CR3 불변
GS 베이스 교체 — 유저 값 그대로
인터럽트 차단 — 명령어의 기능이 아니라 FMASK 설정의 결과
인자 검증 — 레지스터는 유저가 채운 값 그대로
복귀 정보의 저장 — 스택이 아니라 RCX/R11에
```

> KPTI가 활성화된 구성에서는 별도의 CR3 전환이 이어지지만, 이는 완화 기법이지 syscall의 규격 동작이 아닙니다.

syscall을 통과한 직후의 CPU는 권한만 CPL 0이 된 미완성 상태입니다.

# 커널 진입 직후

이 장은 syscall 그 다음 순간, RIP가 LSTAR가 가리키는 진입 코드에 도달한 시점부터 디스패치 직전까지를 다룹니다.

표 6. syscall 직후의 CPU 상태

| 항목          | 상태                                       |
| ------------- | ------------------------------------------ |
| CPL           | 0 (CS=0x10, SS=0x18)                       |
| RIP           | 0xfffff80416210d40 — 진입 코드의 첫 명령   |
| RSP           | 0x7dd29ff878 — 유저 스택                   |
| CR3           | 0x85b12000 — 유저 프로세스의 페이지 테이블 |
| GS 베이스     | 0x7dd2604000 — 유저 값 (TEB)               |
| RCX           | 0x7ff9064ad664 — 유저 복귀 주소            |
| R11           | 0x246 — RFLAGS 사본                        |
| 인자 레지스터 | 유저가 채운 값, 미검증                     |

이 중 커널 모드의 것은 CPL 뿐이고 나머지는 전부 유저가 채우거나 유저 프로세스의 값입니다.
진입 코드는 현재 레지스터들에 커널 실행에 필요한 자원들(커널 스택등)을 확보하고, 복귀할 때를 위해 현재 유저의 상태값들을 보존하는것입니다.

첫 단계부터 제약이 있습니다. 커널 스택의 주소를 어딘가에서 읽어와야 하는데, 범용 레지스터는 전부 유저 값이라 커널 데이터의 포인터로 사용할 수 없습니다.
syscall이 건드리지 않고 커널이 이 시점에 활용할 수 있는 상태는 GS 베이스 하나입니다.

## SWAPGS

x86-64의 GS 베이스는 두 개의 MSR로 관리됩니다.
IA32_GS_BASE(`0xC0000101`)가 현재 활성 값이고 IA32_KERNEL_GS_BASE(`0xC0000102`)가 대기 값이며, SWAPGS는 둘을 교환합니다.
유저 모드의 GS 베이스가 가리키는 것은 운영체제마다 다릅니다.
Windows는 TEB(Thread Environment Block)[^6]이고 Linux는 통상 미사용입니다.

### Linux - 소스에서 확인

```asm
SYM_CODE_START(entry_SYSCALL_64)
 UNWIND_HINT_ENTRY
 ENDBR

 swapgs
 /* tss.sp2 is scratch space. */
 movq %rsp, PER_CPU_VAR(cpu_tss_rw + TSS_sp2)
 SWITCH_TO_KERNEL_CR3 scratch_reg=%rsp
 movq PER_CPU_VAR(cpu_current_top_of_stack), %rsp
```

첫 명령이 swapgs이고, 두 번째 명령이 이미 gs 상대 주소 지정입니다. swapgs가 먼저 실행되어야 하는 이유가 코드의 순서로 드러나 있습니다.
두 MSR의 정의는 [msr-index.h](https://github.com/torvalds/linux/blob/master/arch/x86/include/asm/msr-index.h)에 있습니다(MSR_GS_BASE, MSR_KERNEL_GS_BASE).

### Windows

 syscall 직후(swapgs 실행 전)와 실행 후의 GS 베이스를 비교했습니다.

```text
syscall 직후 (swapgs 실행 전):
GS 베이스 = 0x0000007dd2604000

(gdb) x/gx 0x7dd2604000+0x30
0x7dd2604030:  0x0000007dd2604000      ← TEB.NtTib.Self — 자기 참조

swapgs 실행 후:
GS 베이스 = 0xfffff80413e83000

(gdb) x/gx 0xfffff80413e83000+0x18
0xfffff80413e83018:  0xfffff80413e83000 ← KPCR.Self — 자기 참조

(gdb) x/gx 0xfffff80413e83000+0x20
0xfffff80413e83020:  0xfffff80413e83180 ← CurrentPrcb = KPCR+0x180
```

유저 모드의 GS 베이스는 TEB이고, TEB의 +0x30(NtTib.Self)이 자기 자신을 가리키는 것은 유저 모드의 gs:[0x30] 관례의 실체입니다.
swapgs 후 GS 베이스는 KPCR이 되며 이쪽도 +0x18(Self)이 자기 참조입니다.
+0x20의 CurrentPrcb가 KPCR+0x180을 가리키는 것에서 KPRCB가 KPCR에 내장된 구조임이 확인됩니다.
Windows의 per-CPU 상태는 이 KPCR을 뿌리로 합니다.

이제 첫 명령이 swapgs인 이유가 구체화됩니다.
커널 스택의 주소와 유저 RSP의 보관처가 모두 KPCR/KPRCB의 필드인데, GS 베이스가 교체되기 전에는 그 필드에 접근할 수단이 없기 때문입니다.

## 유저 RSP의 보관과 커널 스택의 적재

스택 전환은 보관과 적재의 두 단계로 수행되며 순서가 정해져 있습니다.
RSP에 커널 스택 주소를 쓰는 순간 유저 RSP가 소실되므로, 보관이 먼저여야 합니다.

### Windows

```asm
swapgs
mov    %rsp, %gs:0x10     ; KPCR.UserRsp(+0x10) ← 유저 RSP
mov    %gs:0x1a8, %rsp    ; RSP ← 커널 스택
```

`mov %rsp, %gs:0x10`이 기록하는 곳은 KPCR의 +`0x10`, UserRsp 필드입니다.
`mov %gs:0x1a8, %rsp`는 KPCR+`0x1a8`에서 커널 스택 주소를 읽어 오는데, 이 오프셋은 KPCR+`0x180`(KPRCB)의 +0x28(RspBase)에 해당합니다.
관찰 결과는 다음과 같습니다.

```text
(gdb) p/x $rsp                        ← syscall 직후
 $28 = 0x7dd29ff878                    ← 유저 스택

(mov %gs:0x1a8,%rsp 실행 후)
(gdb) p/x $rsp
 $33 = 0xffffc08e546b7b90              ← 커널 스택

(gdb) x/gx 0xfffff80413e83000+0x10    ← KPCR.UserRsp
0xfffff80413e83010:  0x0000007dd29ff878
```

보관된 유저 RSP와 적재된 커널 스택 주소가 모두 확인됩니다.
현재 스레드의 커널 스택 꼭대기는 컨텍스트 스위치 시점에 이 슬롯에 갱신되어 있습니다.

### Linux

```asm
swapgs
movq   %rsp, PER_CPU_VAR(cpu_tss_rw + TSS_sp2)      ; 유저 RSP → TSS의 스크래치 슬롯
SWITCH_TO_KERNEL_CR3 scratch_reg=%rsp               ; ¹
movq   PER_CPU_VAR(cpu_current_top_of_stack), %rsp  ; 커널 스택 적재
```

Linux는 보관처가 TSS의 스크래치 슬롯(cpu_tss_rw + TSS_sp2)이고 커널 스택 주소를 per-CPU 변수(cpu_current_top_of_stack)에서 읽습니다.
보관과 적재의 역할 분담이 Windows(KPCR의 UserRsp와 KPRCB의 RspBase)와 다를 뿐, 보관이 먼저라는 순서와 per-CPU 영역을 매개로 한다는 구조는 같습니다.

> SWITCH_TO_KERNEL_CR3은 KPTI(Kernel Page-Table Isolation)[^7]가 활성화된 구성에서만 CR3을 교체합니다.

## 스택 프레임의 구성

커널 스택이 확보되면 진입 코드는 복귀에 필요한 머신 상태를 스택에 push합니다.

### Linux — 같은 순서로 여섯 개를 push합니다

```asm
pushq  $__USER_DS                             ; pt_regs->ss
pushq  PER_CPU_VAR(cpu_tss_rw + TSS_sp2)      ; pt_regs->sp
pushq  %r11                                   ; pt_regs->flags
pushq  $__USER_CS                             ; pt_regs->cs
pushq  %rcx                                   ; pt_regs->ip
pushq  %rax                                   ; pt_regs->orig_ax — 시스템 콜 번호
```

이 필드들은 `pt_regs` 구조체의 후반부에 해당하며, `pt_regs`는 시스템 콜뿐 아니라 인터럽트와 예외 진입이 공유하는 균일한 스냅숏입니다.
Windows 쪽의 대응 구조는 KTRAP_FRAME이고 이 시점에 PreviousMode가 UserMode로 설정됩니다.

### Windows

아래가 관찰한 결과입니다.

```asm
push $0x2b       ; 유저 SS
push %gs:0x10    ; 유저 RSP (KPCR.UserRsp)
push %r11        ; RFLAGS
push $0x33       ; 유저 CS
push %rcx        ; 유저 RIP
```

push 완료 직후의 스택입니다.

```text
(gdb) x/6gx $rsp
0xffffc08e546b7b68:  0x00007ff9064ad664      ← RCX — stub의 ret 주소
                     0x0000000000000033      ← 유저 CS
0xffffc08e546b7b78:  0x0000000000000246      ← R11 — RFLAGS
                     0x0000007dd29ff878      ← 유저 RSP
0xffffc08e546b7b88:  0x000000000000002b      ← 유저 SS
```

각 슬롯의 값이 위 `KPCR.UserRsp`의 레지스터 값과 일치합니다.
RIP 자리에 RCX의 값이, RFLAGS 자리에 R11의 값이 복사되었습니다.
syscall이 경계에 남겨 둔 두 값이 복귀 프레임으로 이동한 것입니다.
SS와 CS는 유저 세그먼트의 상수값입니다.

적립 순서(SS, RSP, RFLAGS, CS, RIP)는 우연이 아닙니다.
인터럽트나 INT 명령으로 커널에 진입할 때 하드웨어가 자동으로 적립하는 IRET64 프레임의 배치와 정확히 같습니다.
INT 시절 CPU가 만들어 주던 프레임을 syscall에서는 CPU가 만들지 않으므로 소프트웨어가 같은 모양으로 적립하는 것입니다.

그리고 push 다섯 개 뒤의 한 명령이 아래와 같습니다.

```asm
mov %r10, %rcx
```

stub이 첫 인자를 RCX에서 R10으로 옮겨 두었으므로, 커널은 첫 인자를 R10에서 읽습니다.
이 명령은 그 값을 MS x64 규약의 첫 인자 자리인 RCX로 복구하여, 이후의 코드가 표준 규약대로 인자를 접근할 수 있게 합니다.

## 범용 레지스터의 백업

머신 프레임 적립 후에는 범용 레지스터 전체를 스택에 백업합니다.

Linux는 `PUSH_AND_CLEAR_REGS` 매크로로 나머지 레지스터를 push하며 `pt_regs`가 완성됩니다.
push하면서 레지스터를 `0`으로 클리어하는데, 범용 레지스터에 남아 있는 유저 값을 이후의 커널 코드가 참조하는 것을 차단하기 위함입니다.
이 매크로는 `pt_regs->ax`에 `-ENOSYS`를 미리 기입하는 것까지 포함하는데, 시스템 콜 번호가 무효할 경우의 기본 반환값이기 때문입니다.

Windows도 KTRAP_FRAME의 나머지 필드에 범용 레지스터를 저장하며 구조상 같은 목적을 수행합니다.

백업이 끝나면 레지스터 파일은 커널이 자유롭게 사용할 수 있는 상태가 됩니다. Linux의 디스패치 호출은 이 직후입니다.

```asm
movq    %rsp, %rdi         ; 인자 1: pt_regs 포인터
movslq  %eax, %rsi         ; 인자 2: 시스템 콜 번호 (부호 확장)
call    do_syscall_64      
```

`movslq`가 `eax`의 하위 32비트만 부호 확장하는 이유는 `glibc`가 eax만 설정하고 `rax` 상위 비트를 방치하는 관행이 있어 이를 정규화하기 위한 것입니다.
Windows는 KTRAP_FRAME 구성 후 `KiSystemServiceRepeat`으로 이행합니다.
MS x64 규약에서 다섯 번째 인자부터는 유저 스택에 실려 있으므로 진입 코드는 이 인자들을 유저 스택에서 커널 스택으로 복사하며, 복사할 개수는 시스템 콜 테이블 엔트리에서 읽습니다.

## 정리

표 7. 진입 코드가 구성한 머신 프레임

| IRET64 프레임 | Linux pt_regs | Windows KTRAP_FRAME | 값 (실측)      | 출처                 |
| ------------- | ------------- | ------------------- | -------------- | -------------------- |
| SS            | ss            | SegSs               | 0x2b           | 상수                 |
| RSP           | sp            | Rsp                 | 0x7dd29ff878   | 보관한 유저 RSP      |
| RFLAGS        | flags         | EFlags              | 0x246          | R11 — syscall이 저장 |
| CS            | cs            | SegCs               | 0x33           | 상수                 |
| RIP           | ip            | Rip                 | 0x7ff9064ad664 | RCX — syscall이 저장 |

# 디스패치 — 번호에서 함수로

4장의 끝에서 두 운영체제는 각자의 프레임(pt_regs / KTRAP_FRAME)을 구성하고 커널 스택 위에 섰습니다.
이 장은 RAX에 남아 있는 번호가 실제 함수로 연결되는 과정을 다룹니다.

## sys_call_table - Linux

Linux의 디스패치는 do_syscall_64 안에서 이루어집니다.

```c
// arch/x86/entry/common.c — __do_syscall_64
regs->ax = sys_call_table[nr](regs);
```

`sys_call_table`은 함수 포인터의 배열로, `fs/syscall_64.c`에서 시스템 콜 번호순으로 초기화됩니다.
1번 인덱스에는 `sys_write`(x86-64에서는 `__x64_sys_write`)의 주소가 들어 있습니다.
배열의 크기는 `NR_syscalls`(약 500개)이고, 호출 전에 번호가 이 범위 안인지만 검사합니다.

```c

if (likely(nr < NR_syscalls)) {
    regs->ax = sys_call_table[nr](regs);
} else {
    regs->ax = -ENOSYS; 
}
```

Linux는 `pt_regs` 포인터 하나를 넘기고 각 시스템 콜 구현이 필요한 필드를 꺼내 씁니다.
`__x64_sys_write`는 `pt_regs`의 `rdi`, `rsi`, `rdx`를 인자로 풀어내는 래퍼입니다.
즉 디스패치 시점에 인자를 재배치하는 코드가 존재하지 않습니다.

이로 인해 시스템 콜 번호가 곧 배열 인덱스이므로 번호는 ABI로 고정될 수밖에 없습니다.
중간에 시스템 콜을 삽입하면 이후의 모든 번호가 밀리고, 그 번호에 의존하는 모든 유저 프로그램이 깨집니다.

## SSDT - Windows

Windows의 디스패치는 `KiSystemServiceRepeat`에서 이루어집니다.
구조는 Linux의 배열과 닮았지만 인코딩이 다릅니다.

SSDT(System Service Dispatch Table)는 `KiServiceTable`이라는 `ULONG` 배열입니다. 각 엔트리는 함수 포인터가 아니라 오프셋을 인코딩한 값입니다.

SSN은

```text
엔트리 = KiServiceTable 주소 + (((KiServiceTable + SSN*4)의 32비트 값) >> 4)
스택 인자의 개수 = (KiServiceTable + SSN*4) & 0x0F
```

즉 하위 4비트가 스택 인자의 개수를 담고, 상위 비트가 오프셋을 담습니다.
2장에서 `NtWriteFile`이 인자 9개 중 4개를 레지스터로 받고 5개를 스택으로 받는다고 했는데, `NtWriteFile`의 SSDT 엔트리 하위 4비트에는 5가 인코딩되어 있습니다.
커널은 스택 인자 개수(엔트리 하위 4비트)만큼 유저 스택의 인자를 커널 스택으로 복사한 뒤 함수를 호출합니다.
이로써 MS x64 규약이 요구하는 인자 배열이 완성됩니다.

### 실험

SSDT의 엔트리가 오프셋 인코딩이라는 것은 알았지만 SSN은 미문서화되었고 빌드마다 재배치되므로, 사용자 모드 코드가 어떤 시스템 콜을 호출하려면 현재 빌드의 SSN을 알아내야 합니다.

모든 `Nt*` 스텁은 같은 크기, 같은 명령 배치의 템플릿이고 서로 다른 것은 `mov eax, imm32`에 오는 상수뿐이라고 했습니다.
역으로 이야기하면 ntdll에서 이 템플릿을 찾으면 그 빌드의 SSN 전체를 복구할 수 있습니다.

`NtWriteFile`과 `NtClose`의 시작 코드를 나란히 놓으면:

```asm
; NtWriteFile                        ; NtClose
mov     r10, rcx                     mov     r10, rcx
mov     eax, 8                       mov     eax, 0Fh
test    byte ptr [7FFE0308h], 1      test    byte ptr [7FFE0308h], 1
jne     ...                          jne     ...
syscall                              syscall
ret                                  ret
```

5바이트 `mov eax, imm32`의 상수를 제외한 나머지 바이트가 완전히 동일합니다. 각 스텁의 주소 차이는 `0x20`바이트로 일정합니다.

이 규칙성을 이용한 추출 절차는 다음과 같습니다.

1. `LoadLibrary` or `GetModuleHandle`로 ntdll을 얻고 export 테이블에서 `Nt`로 시작하는 함수의 주소를 수집합니다.
2. 각 주소에서 스텁 템플릿의 기계어 패턴을 검색합니다.
3. `mov eax` 뒤의 4바이트를 그 함수의 SSN으로 읽습니다.

이와 같은 동작을 하는 rust 코드는 다음과 같습니다.[^8]

```rust
use std::{
    ffi::CString,
    io::{self, Write},
};

use winapi::um::libloaderapi::{GetModuleHandleA, GetProcAddress};

pub unsafe fn get_ssn(function_name: &str) -> Option<u16> {
    let func_name_c = CString::new(function_name).ok()?;
    let ntdll = unsafe { GetModuleHandleA(b"ntdll.dll\0".as_ptr() as *const i8) };
    if ntdll.is_null() {
        return None;
    }

    let func_ptr = unsafe { GetProcAddress(ntdll, func_name_c.as_ptr() as *const i8) };
    if func_ptr.is_null() {
        return None;
    }

    let stub = func_ptr as *const u8;
    let bytes = unsafe { std::slice::from_raw_parts(stub, 8) };

    // mov r10, rcx(4C 8B D1)
    // mov eax(B8)
    if bytes[0] == 0x4C
        && bytes[1] == 0x8B
        && bytes[2] == 0xD1
        && bytes[3] == 0xB8
        && bytes[6] == 0x00
        && bytes[7] == 0x00
    {
        let ssn = u16::from_le_bytes([bytes[4], bytes[5]]);
        return Some(ssn);
    }

    None
}

fn main() {
    print!("Nt 함수 이름 >> ");
    io::stdout().flush().unwrap();

    let mut func_name = String::new();
    io::stdin().read_line(&mut func_name).unwrap();
    let func_name = func_name.trim();

    println!();
    match unsafe { get_ssn(func_name) } {
        Some(ssn) => println!("{} SSN = {:#06x}", func_name, ssn),
        None => eprintln!("실패"),
    }
}
```

실행한 결과와 Windbg를 이용해 확인한 SSN을 비교해보면 `0x55`로 같다는걸 알 수 있습니다.

```text
프로그램 출력값

C:\>hells-gate-poc.exe
Nt 함수 이름 >> NtCreateFile

NtCreateFile SSN = 0x0055

Windbg 유저모드에서 본 stub 코드

kd> uf ntdll!NtCreateFile
ntdll!NtCreateFile:
00007ffc`4660da70 4c8bd1          mov     r10,rcx
00007ffc`4660da73 b855000000      mov     eax,55h
00007ffc`4660da78 f604250803fe7f01 test    byte ptr [SharedUserData+0x308 (00000000`7ffe0308)],1
00007ffc`4660da80 7503            jne     ntdll!NtCreateFile+0x15 (00007ffc`4660da85)  Branch

ntdll!NtCreateFile+0x12:
00007ffc`4660da82 0f05            syscall
00007ffc`4660da84 c3              ret

ntdll!NtCreateFile+0x15:
00007ffc`4660da85 cd2e            int     2Eh
00007ffc`4660da87 c3              ret
```

이렇게 SSN을 찾고 stub 코드를 사용자가 만들면 직접적으로 시스템 콜을 호출할 수 있습니다.[^9]

## 이 시점의 반환값

디스패치로 호출된 시스템 콜 구현은 결과를 반환값으로 돌려줍니다. Linux는 `-ENOSYS` 미리 기입 같은 예외를 제외하면 반환값을 그대로 RAX에 기록하고, Windows는 `NTSTATUS`를 반환합니다

# sysret

5장의 디스패치가 끝나고 시스템 콜의 결과가 RAX에 실리면 커널은 유저 모드로 돌아갑니다.

## 규격 동작

sysret은 syscall의 역연산입니다.

RIP에 RCX 레지스터 값을 대입하고 RFLAGS에 R11 레지스터값을 넣고 CPL 3로 변경합니다.

즉 sysret을 실행하기 직전의 커널은 RCX에 프레임의 RIP, R11에 프레임의 RFLAGS를 다시 넣어 두어야 합니다.
RSP의 복원은 별도로 존재하지 않고 복귀 후의 RSP는 프레임의 rsp 항목이 아니라, sysret 직전에 RSP가 유저 스택 주소로 되돌아가 있어야 합니다
Linux는 이 시점에 시그널·리스케줄 등 대기 중 작업(TIF 검사)을 먼저 처리하고, 있으면 유저로 돌아가기 전에 처리를 수행합니다.

## 마무리

sysret이 완료되면 RIP는 stub의 ret 주소가 되고, RAX는 반환값, RSP는 유저 스택입니다. stub의 `ret`가 실행되면 `KERNELBASE`/`WriteFile`로 돌아가고, `printf`의 나머지 체인이 이어집니다.

이로써 `printf`의 한 글자가 16개의 프레임을 내려와 syscall 한 줄을 지나 MSR과 SWAPGS, 두 개의 테이블을 거쳐 다시 원래 자리로 돌아왔습니다.
하드웨어가 한 일은 RCX, R11, RIP, CS/SS, RFLAGS의 여섯 항목이었고, 나머지 전부는 두 운영체제가 각자의 방식으로 수행한 일입니다.

[^1]: <https://en.wikipedia.org/wiki/System_call>
[^2]: "asdf"는 4바이트지만 puts가 추가한 개행 때문에 길이는 5가 됩니다.
[^3]: [IA-32 및 인텔® 64 아키텍처를 지원하는 프로세서의 모델별 레지스터에 대해 설명합니다](https://www.intel.co.kr/content/www/kr/ko/content-details/916746/intel-64-and-ia-32-architectures-software-developer-s-manual-volume-4-model-specific-registers.html)
[^4]: <https://github.com/torvalds/linux/blob/master/arch/x86/kernel/cpu/common.c#L2310>
[^5]: <https://en.wikipedia.org/wiki/Interrupt_descriptor_table>
[^6]: <https://en.wikipedia.org/wiki/Win32_Thread_Information_Block>
[^7]: <https://en.wikipedia.org/wiki/Kernel_page-table_isolation>
[^8]: <https://github.com/TendouHayase/hells-gate-poc>
[^9]: <https://github.com/TendouHayase/direct-syscall-poc>
