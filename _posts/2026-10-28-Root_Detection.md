---
title: '[전자금융] OS 변조 탐지 기능 적용'
date: 2026-10-28 10:39 +09:00
categories: [모바일 해킹, 취약점]
tags: [모바일 보안, 모바일 해킹, 안드로이드, iOS, OS 변조 탐지 기능 적용]
---

> 대충 ai로 돌린 메모. 내용 검증은 차차 할 것... OWASP MASTG(MASVS-RESILIENCE)의 knowledge / techniques / tests 기반으로 정리.
{: .prompt-warning }

## 취약점 설명

OS가 루팅(Android) 및 탈옥(iOS)을 통해 변조되면 해당 단말기에서 공격자가 애플리케이션 분석, 메모리 분석, 악성코드 감염, 중요정보 접근 등 다양한 공격을 수행할 수 있다.

루팅/탈옥 단말은 앱 샌드박스가 무력화되고 root 권한으로 다른 앱의 데이터·메모리에 접근할 수 있어, 동적 분석(Frida 등)·메모리 덤프·중요정보 탈취의 전제 조건이 된다. 따라서 전자금융 앱은 변조된 OS 환경을 **탐지하고, 탐지 시 서비스 이용을 차단**해야 한다.

> **전자금융 점검 판정 기준**
> - 양호: 루팅/탈옥 단말에서 앱 실행·주요 기능 이용이 차단된다.
> - 취약: 루팅/탈옥을 탐지하지 못하거나, 탐지해도 경고만 띄우고 서비스 이용이 계속 가능하다. 또는 탐지 로직이 **쉽게 우회**되어 무력화된다.
{: .prompt-tip }

MASTG 관점에서 중요한 포인트: **"탐지 기능이 있는가"와 "그 탐지가 견고한가"는 다른 문제다.** 탐지 로직이 존재해도 자바 레벨에 평문으로 있으면 우회가 쉬워 사실상 취약으로 본다.
- MASTG-TEST-0324 (정적): 탐지 로직이 **존재하는지** 확인 → 없으면 fail
- MASTG-TEST-0325 (런타임): 탐지가 **실제로 동작하는지** 확인 → 통과해도 "견고함"까지 보장하진 않음


## 진단 방법

### 0. 사전 준비

- 루팅 단말(또는 루팅 에뮬레이터) + Magisk
- jadx-gui (정적), Frida / objection (동적)
- 대상 APK 확보: `adb shell pm path <pkg>` → `adb pull` (split APK면 여러 개)

### 1. 정적 분석 — 탐지 로직 존재 및 방식 확인 (MASTG-TEST-0324)

jadx로 디컴파일 후, 아래 **MASTG 탐지 카테고리 6종**에 해당하는 문자열·API를 검색한다.

| MASTG 탐지 카테고리 | 찾을 흔적 (문자열/API) |
|---|---|
| File Existence Checks | `/system/bin/su`, `/system/xbin/su`, `busybox`, `Superuser.apk`, `magisk`, `daemonsu`, `File.exists()` |
| Executing Privileged Commands | `Runtime.exec("su")`, `which su`, `ProcessBuilder` |
| Checking Running Processes | `magiskd`, `daemonsu` 프로세스명 조회 |
| Checking Installed App Packages | `PackageManager.getPackageInfo`, `com.topjohnwu.magisk`, `eu.chainfire.supersu` |
| Writable Partitions / System Dirs | `/system` 쓰기 가능 여부, `mount`, `/proc/mounts` |
| Checking for Custom Android Builds | `Build.TAGS` → `test-keys`, `ro.build.selinux`, `ro.debuggable` |
| (추가) 무결성/attestation | `SafetyNet`, `Play Integrity`, `RootBeer` 라이브러리 |

> **UnCrackable1에서 본 것과 매핑** (`sg.vantagepoint.a.c`)
> - `c.a()` → PATH에서 `su` 파일 탐색 = **File Existence**
> - `c.b()` → `Build.TAGS`의 `test-keys` = **Custom Build**
> - `c.c()` → 고정된 su 경로 목록 탐색 = **File Existence**
>
> 전형적인 자바 레벨 3종 세트. 평문으로 노출 → 우회 쉬움 → 견고성 측면에서 취약.
{: .prompt-info }

**정적 분석에서 봐야 할 것**
- 탐지 로직이 **자바/코틀린** 레이어인가, **네이티브(.so)** 레이어인가 (네이티브면 우회 난이도↑)
- 난독화 여부 (클래스/메서드명이 `a`, `aa`... 식인지)
- 탐지 결과를 **어떻게 처리**하는가 (즉시 종료 / 경고 후 계속 / 서버 통지)

### 2. 동적 분석 — 런타임 동작 확인 (MASTG-TEST-0325)

루팅 단말에서 앱 실행 → 루팅 탐지로 차단되는지 확인한다. 그 후 탐지 메서드가 실제 호출되는지 Frida로 관찰:

```javascript
// 탐지 의심 메서드가 호출되는지 추적 (반환값 변경 없이 관찰만)
Java.perform(function () {
    var C = Java.use("sg.vantagepoint.a.c");
    C.a.implementation = function () {
        var r = this.a();
        console.log("[hook] c.a() 호출됨, 반환=" + r);
        return r;
    };
});
```

objection으로 빠르게 탐지 지점 확인도 가능:

```bash
objection -g <pkg> explore
# (세션 내) android hooking search classes root
# (세션 내) android hooking watch class sg.vantagepoint.a.c
```

### 3. 판정

- 탐지 자체가 없음 → **취약**
- 탐지는 하나 경고만 띄우고 이용 가능 → **취약**
- 탐지 후 차단하지만 자바 레벨 평문 → **취약(견고성)**, 아래 우회로 입증
- 네이티브+난독화+서버 attestation으로 다층 방어 → **양호**


## 우회 기법

> 핵심: 우회는 **두 레이어**에서 접근한다.
> - (A) **환경 레벨** — 루팅 흔적을 앱한테 안 보이게 숨김 (앱 코드 수정 X)
> - (B) **앱 레벨** — 탐지 로직 자체를 무력화 (후킹/패치)
{: .prompt-tip }

MASTG-TECH-0144(Bypassing Root Detection)의 분류: `objection` 자동 우회 → 커스텀 Frida → 수동(리패키징) 순으로 정리.

> **탐지 카테고리 → 공략 지점 매핑** (진단 방법의 6종 표와 짝)
> 탐지는 결국 몇 개의 **말단 API**로 수렴한다. 독자 구현이라 메서드명을 못 찾아도, 이 말단 API를 잡으면 상위 로직 전체가 무력화된다.
>
> | 탐지 방식 | 수렴하는 말단 API | 공략 레이어 |
> |---|---|---|
> | 파일 존재 | `File.exists` / libc `access`,`stat`,`fopen`,`open` | 자바 또는 네이티브 |
> | su/which 실행 | `Runtime.exec`, `ProcessBuilder`, libc `system`,`execve` | 자바 또는 네이티브 |
> | 패키지 조회 | `PackageManager.getPackageInfo`,`getInstalledPackages` | 자바 |
> | 프로세스/마운트 | `/proc/mounts`,`/proc/self/maps` 읽기 (`open`+`read`) | 네이티브 우세 |
> | 커스텀 빌드 | `Build.TAGS` 필드, `__system_property_get` | 자바 또는 네이티브 |
> | attestation | Play Integrity 응답 (서버 검증) | **단말 후킹으로 불가** |
{: .prompt-info }

### 1. 환경 레벨 — Magisk DenyList / Zygisk (+ Shamiko)

- Magisk 설정 → **Zygisk ON** → **Enforce DenyList ON** → 대상 앱 체크
- DenyList는 해당 앱 프로세스에서 Magisk가 바꾼 흔적을 되돌려(unmount) **File/Package/Mount 기반 탐지**를 1차로 회피
- **원리**: 앱 프로세스가 zygote에서 fork되는 순간 Zygisk가 끼어들어, 그 프로세스에서만 Magisk의 오버레이 마운트를 해제하고 `su`·magisk 파일·마운트 흔적을 안 보이게 한다. 즉 "앱 눈에만 비루팅처럼" 보이게 하는 것. 앱 코드는 안 건드림.
- **DenyList vs Shamiko**: 기본 DenyList는 마운트 해제 수준이라 `/proc/mounts`·프로세스 목록에 흔적이 남을 수 있다. **Shamiko**는 이 은닉을 프로세스 전반으로 강화하고 DenyList를 숨김 리스트로 활용(그래서 보통 DenyList "Enforce"는 끄고 체크만 해둔 상태로 Shamiko와 조합). magisk 앱 자체도 패키지명 변경(stub/하이드)으로 숨긴다.
- **한계 (왜 이것만으론 부족한가)**:
  - `Runtime.exec("su")`처럼 **실행 시도형** 탐지는 파일 은닉과 별개라 걸릴 수 있음
  - 네이티브에서 직접 `/proc`·마운트를 파싱하는 심화 탐지
  - **Play Integrity(하드웨어 attestation)**: 부트로더 언락·커스텀 키 자체를 TEE가 증명하므로 은닉으로 못 넘김 (별도 모듈로 basic 수준은 통과시키기도 하나 strong/device 무결성은 어려움)
- 점검 관점: 환경 레벨 우회가 통하면 "파일·패키지 기반 탐지만 있다"는 방증 → 견고성 취약 근거로 기록

### 2. objection 자동 우회

```bash
objection -g <pkg> explore
# (세션 내)
android root disable
```

- 흔한 탐지 API(파일 존재, `Runtime.exec`, 패키지 조회 등)를 일괄 후킹해 안전한 값 반환
- 간단한 앱엔 즉효. 커스텀/독자 로직엔 안 먹힘 → 아래 Frida로

### 3. Frida 커스텀 후킹 (가장 많이 쓰는 방식)

정적 분석으로 찾은 탐지 메서드를 직접 false/정상값으로 바꾼다. UnCrackable1에서 한 것과 동일한 패턴:

```javascript
Java.perform(function () {
    var C = Java.use("sg.vantagepoint.a.c");
    C.a.implementation = function () { return false; };   // su 파일 탐색
    C.b.implementation = function () { return false; };   // test-keys
    C.c.implementation = function () { return false; };   // su 경로
});
```

독자 구현이라 메서드를 특정하기 어려울 때 공략 포인트:
- `java.io.File.exists` 후킹해서 su 관련 경로만 false 반환
- `Runtime.exec` 후킹해서 `su`/`which` 호출 가로채기
- `PackageManager.getPackageInfo` 후킹해서 magisk 패키지에 예외 던지게
- `android.os.Build` 의 `TAGS` 필드 값을 `release-keys`로 치환

> **전략: "핀포인트" vs "말단 API 광역 후킹"**
> - **핀포인트**: 정적 분석으로 `c.a()` 같은 탐지 메서드를 특정해 그 반환값만 바꿈 → 깔끔하지만 탐지가 분산·중복되면 다 찾아야 함
> - **광역**: `File.exists`·`Runtime.exec` 같은 말단 API를 전부 후킹하고 인자로 필터(su·magisk 경로일 때만 가짜값) → 메서드명 몰라도 됨. 단 과도하게 막으면 앱 정상 동작까지 깨져(정상 파일 체크도 막힘) **오탐·크래시** 유발 → 인자 조건을 좁혀야 함
>
> 실무에선 보통 광역으로 탐지 지점을 **찾고**, 확인되면 핀포인트로 좁혀 안정화한다.
{: .prompt-tip }

**후킹이 안 통하는 경우 (왜?)**
- 탐지가 **네이티브(.so)**에 있으면 자바 후킹 대상이 아예 없음 → 아래 5번
- 앱이 **Frida 자체를 탐지**(스레드명 `gmain`/`gum-js-loop`, 포트 27042, `/proc/self/maps`의 frida 흔적)해서 후킹 전에 종료 → Frida 탐지 로직부터 무력화해야 함 ("보안프로그램 임의해제 방지" 항목과 연계)
- **조기 실행**: 탐지가 `onCreate`보다 먼저(`Application.attachBaseContext`, 네이티브 `JNI_OnLoad`, static initializer) 돌면 attach로는 늦음 → **spawn 모드**(`-f`)로 앱 시작 전에 스크립트를 올려야 함

### 4. 수동 우회 — 리패키징/패치 (Frida 못 쓰는 환경 대비)

1. `apktool d target.apk` 로 디코드
2. smali에서 탐지 메서드를 찾아 반환값을 상수로 패치 (`const/4 v0, 0x0` 후 `return v0`)
3. `apktool b` → `uber-apk-signer`로 재서명 → 설치
- DIVA 프로그램 무결성 랩에서 한 리패키징과 같은 흐름
- 단, 무결성 검증(서명 체크)이 같이 있으면 그 로직도 함께 패치해야 함

### 5. 네이티브 / 난독화된 체크 대응 (흐름만)

- 탐지가 `.so`에 있으면 자바 후킹으론 안 됨 → `Interceptor.attach`로 네이티브 함수 후킹, 또는 libc 공통 함수를 가로채 su 경로 접근을 속임
- **네이티브 말단 함수** (여기로 수렴):
  - 파일 존재 → `access`, `stat`, `fopen`, `open`, `faccessat` : 경로가 su/magisk일 때만 "없음(-1/NULL)" 반환
  - 명령 실행 → `system`, `execve`, `popen` : `su`/`which` 인자면 실패 처리
  - 프로퍼티 → `__system_property_get("ro.build.tags")` 결과를 `release-keys`로 치환
  - `/proc` 파싱 → `open`+`read`로 `/proc/mounts`·`/proc/self/maps` 읽는 걸 가로채 magisk 라인 제거
- **난독화 대응 흐름**:
  1. 정적으로 심볼이 지워져 함수를 못 찾으면, 위 **libc export는 이름이 남아있으므로** 거기부터 후킹해 호출자(return address)를 역추적
  2. `Module.enumerateImports/Exports`로 .so가 어떤 libc 함수를 쓰는지 확인
  3. 문자열이 암호화돼 있으면 런타임에 복호화된 직후를 노림(메모리 스캔)
- **분산·중복**: 심화 탐지는 탐지 함수 하나가 아니라 여러 곳에 흩어져 있고, **시작 시 1회가 아니라 주기적으로** 돈다. 하나만 막으면 다른 데서 다시 걸리므로 전수 식별 필요.
- **패치 vs 후킹**: 후킹이 탐지되는 환경이면 .so를 직접 패치(분기 NOP/반전)해 재서명. 반대로 패치는 무결성 검증에 걸릴 수 있어, 둘 중 뭐가 덜 걸리는지는 앱마다 다름.

### 6. 우회 탐지(anti-anti)에 대한 대응

앱이 "우회 시도 자체"를 잡으러 오는 경우. 점검 시 이게 있으면 견고성 높은 것으로 평가.
- **Frida 탐지**: 포트 27042, 스레드명(`gum-js-loop`,`gmain`), `/proc/self/maps`의 `frida`/`gadget` 문자열, D-Bus auth 문자열 스캔 → 후킹 코드로 이 탐지부터 무력화하거나, frida-server 리네임·포트 변경·gadget 임베드로 흔적 축소
- **Magisk 탐지**: magisk 앱 패키지·`/data/adb/magisk`·소켓 → 환경 레벨(DenyList/Shamiko/magisk 하이드)로 대응
- **무결성 검증과 세트**: 리패키징 우회를 막기 위해 서명/체크섬을 같이 보는 앱이 많음 → 무결성 로직도 함께 우회해야 리패키징 성립 (프로그램 무결성 항목과 연계)

### 우회 한계 (여기서 막히면 양호)

- **Play Integrity / SafetyNet (하드웨어 기반 attestation)**: 판정이 **서버**에서 이뤄지고 TEE/하드웨어 키에 묶여 있어, 단말 내 후킹만으로는 우회가 매우 어렵다.
  - verdict 레벨: `MEETS_BASIC_INTEGRITY` / `MEETS_DEVICE_INTEGRITY` / `MEETS_STRONG_INTEGRITY`
  - 커스텀 모듈로 basic 수준은 통과시키는 사례가 있으나, device/strong(부트로더·TEE 증명)은 사실상 불가
  - **단, 서버가 verdict를 제대로 안 보고 클라이언트 결과만 믿으면** 그 통신을 가로채 응답을 위조하는 쪽이 현실적 공격 → 점검 시 "attestation 검증이 서버측인지"를 반드시 확인
- 정리: 환경 은닉·후킹·패치로 **단말 내** 탐지는 거의 다 우회 가능. 서버측 원격 검증이 있어야 실질 차단이 된다.


## 보안 대책

루팅 탐지는 "완벽 차단"이 목적이 아니라 **공격 비용을 올리는 다층 방어**가 목적이다 (MASTG-BEST-0030).

1. **다층·중복 탐지**: 위 6개 카테고리를 여러 개 조합하고, 탐지 코드를 앱 곳곳에 분산·중복 배치해 한 곳만 패치해선 무력화되지 않게 한다.

2. **네이티브 + 난독화**: 탐지 로직을 자바가 아닌 **네이티브(.so)**로 구현하고 난독화(문자열 암호화 포함)한다. UnCrackable1처럼 자바 평문이면 바로 뚫린다.

3. **서버 측 원격 검증 (가장 중요)**: 단말 내 탐지는 결국 우회 가능하므로, **Play Integrity API** 등으로 기기·앱 무결성을 **서버에서** 검증한다. 하드웨어 기반 attestation이라 단말 후킹으로 속이기 어렵다. 전자금융에선 이 서버측 차단이 핵심.

4. **탐지 후 처리**: 경고만 띄우지 말고 **주요 기능(로그인/이체 등)을 서버에서 차단**한다. 클라이언트 단독 종료는 패치로 무력화됨.

5. **무결성 검증과 세트로**: 리패키징 우회를 막기 위해 서명/체크섬 검증(프로그램 무결성)을 함께 적용한다.

6. **RASP / 상용 보안 솔루션**: 전자금융 앱은 보통 앱쉴드 류 RASP로 루팅·후킹(Frida)·디버깅·에뮬레이터를 통합 탐지한다. (→ 별도 항목 "보안프로그램 동작 여부"와 연계)

---

**참고 (OWASP MASTG)**

- [MASTG-TEST-0324 — References to Root Detection Mechanisms (정적)](https://mas.owasp.org/MASTG/tests/android/MASVS-RESILIENCE/MASTG-TEST-0324/)
- [MASTG-TEST-0325 — Runtime Use of Root Detection Techniques (런타임)](https://mas.owasp.org/MASTG/tests/android/MASVS-RESILIENCE/MASTG-TEST-0325/)
- [MASTG-TECH-0144 — Bypassing Root Detection](https://mas.owasp.org/MASTG/techniques/android/MASTG-TECH-0144/)
- MASTG-KNOW-0027 — Root Detection / MASTG-BEST-0030 — Implementing Root Detection

