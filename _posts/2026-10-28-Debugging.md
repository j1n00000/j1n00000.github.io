---
title: '[전자금융] 디버깅 탐지기능 적용'
date: 2026-10-28 17:30 +09:00
categories: [모바일 해킹, 취약점]
tags: [모바일 보안, 모바일 해킹, 안드로이드, iOS, 디버깅 탐지 기능 적용]
---

> 대충 ai로 돌린 메모. 내용 검증은 차차 할 것... OWASP MASTG(MASVS-RESILIENCE)의 knowledge / techniques / tests 기반으로 정리.
{: .prompt-warning }

## 취약점 설명

동적 디버깅을 통해 애플리케이션의 동작 흐름을 파악할 수 있으므로, 공격자는 애플리케이션 동적 디버깅으로 코드 흐름 파악, 메모리 상태 분석, 분기 조작(인증 우회 등)을 할 수 있다.

디버거를 붙이면 브레이크포인트로 실행을 멈추고 변수·메모리를 들여다보거나 레지스터/반환값을 바꿔 로직을 우회할 수 있다. 따라서 전자금융 앱은 디버거가 붙은 상태를 **탐지하고, 탐지 시 실행을 중단**해야 한다.

> **디버깅 레이어 2종** (MASTG-KNOW-0028)
> Android 앱은 두 층위에서 디버깅할 수 있고, 제대로 된 방어는 **둘 다** 막아야 한다.
> - **자바(JDWP)**: `android:debuggable` 플래그로 JDWP 스레드가 열리면 jdb/Android Studio로 attach
> - **네이티브(ptrace)**: gdb/lldb/IDA 등 ptrace 기반 디버거가 프로세스에 attach
{: .prompt-info }

> **전자금융 점검 판정 기준**
> - 양호: 디버거 attach 시 앱이 실행을 중단하고, 자바·네이티브 양쪽을 다층으로 탐지해 간단히 우회되지 않는다.
> - 취약: 탐지 로직이 없거나, 경고만 하고 실행이 계속되거나, **단일 API 후킹 하나로 무력화**된다(MASTG-TEST-0352의 핵심 평가 포인트).
{: .prompt-tip }


## 진단 방법

### 0. 사전 준비

- 루팅 단말 + Magisk, jadx-gui / apktool, Frida / objection
- 네이티브 분석용: Ghidra 또는 IDA, `strings`
- 대상 APK: `adb shell pm path <pkg>` → `adb pull`

### 1. 정적 분석 — 탐지 로직 존재/방식 확인 (MASTG-TEST-0046 계열)

jadx로 자바 레이어, Ghidra/`strings`로 네이티브(.so) 레이어를 각각 본다.

| 레이어 | 찾을 흔적 (API/문자열) |
|---|---|
| 자바 / JDWP | `ApplicationInfo.flags & 2` (`FLAG_DEBUGGABLE`), `Debug.isDebuggerConnected()`, `Debug.waitingForDebugger()` |
| 네이티브 / ptrace | `ptrace` 호출, `/proc/self/status` 읽기 + `TracerPid` 파싱, `/proc/self/task/*/status`, `fork` 후 자식이 부모를 추적 |
| 타이밍 | 구간 실행 시간 측정 (`System.nanoTime`, `clock_gettime`) — 디버거가 붙으면 느려지는 것을 이용 |

> **UnCrackable1에서 본 것과 매핑** (`sg.vantagepoint.a.b`)
> - `b.a(getApplicationContext())` → `ApplicationInfo.flags` 에서 `FLAG_DEBUGGABLE` 확인 = **자바/JDWP 레이어**
>
> 즉 UnCrackable1은 "앱이 debuggable 빌드인지"만 보는 가장 얕은 체크. **네이티브 ptrace 탐지는 없음** → 이 부분이 UnCrackable2(네이티브 레이어)에서 확장됨.
{: .prompt-info }

**봐야 할 것**
- 탐지가 자바뿐인가, 네이티브까지 있는가 (네이티브면 우회 난이도↑)
- 탐지 시점: `onCreate` 등 시작 1회인가, 주기적으로 도는 watchdog인가
- 탐지 결과 처리: 즉시 종료 / 경고 후 계속 / 서버 통지

### 2. 동적 분석 — 실제 동작 확인 (MASTG-TEST-0352)

**자바(JDWP) 쪽**
1. `apktool`로 `android:debuggable="true"` 패치 후 재서명·설치 (또는 ro.debuggable 환경)
2. `adb jdwp` → `jdb`로 attach 시도
3. attach 순간 앱이 죽거나 기능이 막히면 → JDWP 탐지 동작

**네이티브(ptrace) 쪽**
- `strace -p <pid>` 또는 gdb attach 시도 → 실패하거나 앱이 종료되면 ptrace 기반 탐지 동작 (strace 자체가 ptrace를 쓰므로 anti-debug에 걸려 실패함)

**호출 관찰 (Frida)**
```javascript
Java.perform(function () {
    var Debug = Java.use("android.os.Debug");
    Debug.isDebuggerConnected.implementation = function () {
        console.log("[hook] isDebuggerConnected() 호출됨");
        return this.isDebuggerConnected();
    };
});
```

### 3. 판정

- 탐지 없음 / 경고만 → **취약**
- 자바 단일 API 하나만 체크 → **취약(견고성)**, 아래 우회로 입증
- 자바 + 네이티브 다층 + watchdog → **양호**


## 우회 기법

> 큰 그림: **레이어별로 접근**한다.
> - (A) 자바/JDWP 탐지 → 자바 메서드 후킹 또는 플래그 조작
> - (B) 네이티브/ptrace 탐지 → libc/시스템콜 후킹 또는 .so 패치
>
> MASTG의 두 축(MASTG-KNOW-0028): **① 체크 패치(NOP) ② 환경 조작**. 범용 우회법은 없고 메커니즘마다 다르다.
{: .prompt-tip }

### 1. 자바 / JDWP 탐지 우회

가장 쉬운 층. UnCrackable1 `b.a()` 우회와 동일 패턴.

```javascript
Java.perform(function () {
    // (a) 독자 탐지 메서드 직접 false
    var B = Java.use("sg.vantagepoint.a.b");
    B.a.implementation = function (ctx) { return false; };

    // (b) 공통 말단 API 후킹 — 메서드를 못 특정할 때
    var Debug = Java.use("android.os.Debug");
    Debug.isDebuggerConnected.implementation = function () { return false; };

    // (c) FLAG_DEBUGGABLE 비트를 꺼서 debuggable 아닌 것처럼
    var AppInfo = Java.use("android.content.pm.ApplicationInfo");
    // flags 필드를 읽는 지점을 후킹하거나, getApplicationInfo() 반환 객체의 flags를 &= ~2
});
```

- `waitingForDebugger()`도 쓰면 같이 false 처리
- objection: `android hooking watch class android.os.Debug` 로 호출 지점부터 확인

### 2. 네이티브 / ptrace 탐지 우회 (여기가 핵심, UnCrackable2에서 실습)

**(a) `ptrace` 자체 후킹**
많은 앱이 `ptrace(PTRACE_TRACEME, ...)`로 **자기 자신을 추적**해 디버거 슬롯을 선점한다(프로세스당 ptracer는 1개뿐이라, 이미 자기가 붙으면 외부 디버거가 못 붙음). 이걸 무력화:

```javascript
// libc의 ptrace를 가로채 항상 0 반환 → self-attach 무력화
Interceptor.attach(Module.getExportByName("libc.so", "ptrace"), {
    onEnter: function (args) { /* request 인자 확인 가능 */ },
    onLeave: function (retval) { retval.replace(0); }
});
// 또는 아예 호출 자체를 치환
Interceptor.replace(Module.getExportByName("libc.so", "ptrace"),
    new NativeCallback(function () { return 0; }, 'long', []));
```

**(b) `/proc/self/status`의 `TracerPid` 속이기**
디버거가 붙으면 `TracerPid`가 0이 아니게 된다. 앱이 이 값을 읽어 판단하면:
- `fopen`/`open`+`read`를 후킹해 `/proc/self/status` 접근을 가로채고 `TracerPid: 0`을 돌려주기
- 또는 문자열 `TracerPid`를 파싱하는 네이티브 함수를 직접 후킹해 0 반환

**(c) .so 바이너리 패치**
- Ghidra/IDA에서 `ptrace` 호출이나 `TracerPid` 비교 분기를 찾아 `NOP` 처리하거나 비교 결과를 반전
- `apktool`로 so 교체 후 재서명 — 잘 설계된 anti-debug면 분산되어 있어 단순 NOP로 안 끝남

### 3. 환경/타이밍 조작

- **타이밍 체크**: 브레이크포인트로 멈추면 구간 시간이 길어져 탐지됨 → 측정 함수(`clock_gettime` 등)를 후킹해 정상 범위 값 반환
- **watchdog/자식 프로세스**: 부모를 ptrace하는 감시 자식을 fork하는 구조면, 자식 프로세스까지 같이 다뤄야 함

### 4. Frida 자체가 anti-debug/anti-Frida에 걸릴 때 (중요 함정)

- Frida도 spawn 시 ptrace를 쓰므로, 앱의 self-ptrace와 **충돌**해 attach가 막힐 수 있다.
- 대응: **spawn + 조기 후킹**으로 anti-debug 초기화 **전에** ptrace를 가로챈다.
  ```bash
  frida -U -f <pkg> -l anti_debug.js    # 실행 시점부터 후킹, %resume 전에 ptrace 치환
  ```
- 그래도 막히면 `frida-gadget`을 앱에 심어 로드 시점부터 개입, 또는 `LD_PRELOAD`로 ptrace 래퍼를 선주입
- 앱이 `strcmp`/스레드명/포트(27042) 등으로 **Frida 자체를 탐지**하면 그 로직도 함께 후킹 (→ 별도 항목 "보안프로그램 임의해제 방지"와 연계)

### 우회 난이도 정리

- 자바 단일 체크: objection 한 줄 또는 Frida 몇 줄 → **쉬움**
- 네이티브 ptrace 단일: `ptrace` 후킹 → **중간**
- 네이티브 + watchdog + 타이밍 + Frida 탐지 다층: 전수 식별·조기 주입·패치 병행 → **어려움**


## 보안 대책

디버깅 탐지도 "완벽 차단"이 아니라 **공격 비용 상승**이 목적 (MASWE-0064는 다층·네이티브 체크, anti-anti-debugging 대비를 권고).

1. **자바 + 네이티브 양쪽 방어**: JDWP(`FLAG_DEBUGGABLE`, `isDebuggerConnected`)만으론 부족. ptrace 기반 네이티브 탐지를 함께 둔다. UnCrackable1처럼 자바 단일 체크면 즉시 뚫린다.

2. **ptrace self-attach**: 앱이 시작 시 `ptrace(PTRACE_TRACEME)`로 자신을 추적해 외부 디버거 슬롯을 선점한다. (단, 이것만 쓰면 ptrace 후킹으로 우회되므로 보조 수단)

3. **다중·중복·주기적 탐지**: 탐지를 한 곳이 아니라 여러 지점에 분산하고, 시작 1회가 아니라 주기적으로/다른 스레드에서 검사해 단일 후킹·NOP로 무력화되지 않게 한다.

4. **난독화 + 네이티브화**: 탐지 로직을 .so로 옮기고 난독화, 문자열(`TracerPid`, `/proc/self/status`)도 암호화해 정적 식별을 어렵게 한다.

5. **탐지 후 서버측 차단**: 클라이언트 단독 종료는 패치로 무력화되므로, 탐지 신호를 서버로 보내 **주요 기능을 서버에서 차단**한다.

6. **릴리즈 빌드 위생**: `android:debuggable="false"`(릴리즈 기본), 디버그 심볼·로그 제거.

7. **RASP / 상용 솔루션**: 전자금융 앱은 보통 RASP로 디버깅·후킹·루팅·에뮬레이터를 통합 탐지. (→ "보안프로그램 동작 여부"와 연계)

---

**참고 (OWASP MASTG)**

- [MASTG-KNOW-0028 — Anti-Debugging (JDWP/ptrace 2종 레이어)](https://mas.owasp.org/MASTG/knowledge/android/MASVS-RESILIENCE/MASTG-KNOW-0028/)
- [MASTG-TEST-0352 — Anti-Debugging 효과성 테스트](https://mas.owasp.org/MASTG/tests/android/MASVS-RESILIENCE/MASTG-TEST-0352/)
- [MASWE-0064 — Debugging 관련 약점(다층 탐지 권고)](https://mas.owasp.org/MASWE/MASVS-RESILIENCE/MASWE-0064/)
- [MASTG-TECH-0031 — Debugging](https://mas.owasp.org/MASTG/techniques/android/MASTG-TECH-0031/)
