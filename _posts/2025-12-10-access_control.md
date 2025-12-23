---
title: 접근 통제 취약점
date: 2025-12-10 2:30 +09:00
categories: [웹해킹, Access control vulnerabilities, portswigger lab]
tags: [웹보안, 웹해킹, Access control vulnerabilities, IDOR]
---

## 접근 통제란?
---

접근 제어는 누가 / 무엇이 특정 동작을 수행하거나 자원에 접근할 수 있는지를 제한하는 기능이다.

웹 애플리케이션에서는 인증(authentication)과 세션 관리(session management)를 기반으로 동작하는데, 인증과 세션 관리, 접근 제어 용어를 구별하면 아래와 같다.

- **인증(Authentication)**  
사용자가 자신이 주장하는 사람임을 확인하는 과정

- **세션 관리(Session Management)**  
같은 사용자가 보낸 이후의 HTTP 요청을 식별하는 과정

- **접근 제어(Access Control)**  
사용자가 요청한 동작을 수행할 "권한이 있는지" 최종적으로 판단하는 기능

깨진 접근 제어(broken access control)는 아주 흔하며, 보안상 치명적인 취약점으로 이어질 수 있다. 접근 제어는 기술뿐 아니라 비즈니스/조직/법적 요구사항까지 고려해야 하므로, 사람이 설계하며 오류가 발생하기 쉽다.

깨진 접근 제어는 사용자가 원래 접근할 수 없어야 하는 기능 또는 데이터에 접근할 수 있을 때 발생한다.

**📌 수직적 권한 상승**  
허용되지 않은 "상위 권한 기능"에 접근하는 경우

예: 비관리자 사용자가 관리자 페이지에 접근해서 사용자 계정을 삭제할 수 있는 상황.

**📌 보호되지 않은 기능 (Unprotected Functionality)**

애플리케이션이 민감한 기능에 아무 보호도 적용하지 않을 때 발생한다.

관리자 페이지가 일반 사용자 화면에는 링크가 없지만, 사용자가 직접 URL을 입력하면 접근 가능할 수도 있다.

예시 URL:
```
https://insecure-website.com/admin
```

이 주소가 관리자만 볼 수 있어야 하지만, 실제로는 누구든 접근 가능한 경우가 있다. 또한 다음과 같은 파일을 통해 URL이 노출될 수도 있다:
```
https://insecure-website.com/robots.txt
```

URL이 노출되지 않더라도, 공격자가 워드리스트로 경로를 추측하여 찾아낼 수 있다.



<div style="height: 70px;"></div>

## 접근 제어 유형
---

**📌 수직적 접근 제어 (Vertical Access Controls)**

> 사용자 "종류"에 따라 서로 다른 기능 접근을 제한
{: .prompt-info }

예:
- 관리자 → 모든 사용자 계정 수정/삭제 가능
- 일반 사용자 → 자신의 정보만 수정 가능

업무 분리, 최소 권한 같은 정책을 기술적으로 구현한 방식이다.

**📌 수평적 접근 제어 (Horizontal Access Controls)**

> 같은 유형의 자원 안에서 "특정 사용자"만 접근하도록 제한
{: .prompt-info }

예:
- A 사용자는 자신의 계좌만 조회 가능
- B 사용자의 계좌에는 접근 불가

같은 기능이지만 "누구의 데이터냐"가 중요한 상황에서 필요하다.

**📌 상황(문맥) 의존적 접근 제어 (Context-dependent Access Controls)**

> 애플리케이션의 상태 또는 사용자의 행동 흐름에 따라 접근을 제한
{: .prompt-info }

예:
- 결제를 완료한 후에는 장바구니 내용을 수정할 수 없음
- 순서가 잘못된 행동을 방지하는 목적


<div style="height: 70px;"></div>


## IDOR (Insecure Direct Object References)
---

IDOR은 접근 통제 취약점의 일종으로, 애플리케이션이 사용자가 제공한 입력값을 검증 없이 객체에 직접 접근할 때 발생한다.

OWASP 2007 Top Ten에서 처음 등장한 용어로, 수평적 권한 상승과 관련이 깊지만 수직적 권한 상승으로도 이어질 수 있다.

### IDOR의 동작 원리

사용자가 URL이나 파라미터로 특정 객체를 요청할 때, 서버가 "이 사용자에게 해당 객체를 조회할 권한이 있는가?"를 확인하지 않고 그대로 응답하는 경우 발생한다.

### IDOR 취약점 예시

**1. 데이터베이스 객체 직접 참조**
```
https://insecure-website.com/customer_account?customer_number=132355
```

위 URL에서 `customer_number` 파라미터가 데이터베이스 쿼리에 그대로 사용된다. 공격자가 파라미터 값을 132356, 132357 등으로 변경하면 다른 사용자의 계정 정보를 열람할 수 있다.

이는 **수평적 권한 상승**의 대표적인 예시다. 공격자가 관리자 계정 번호를 찾아낸다면 **수직적 권한 상승**으로도 이어질 수 있다.

**2. 정적 파일 직접 참조**
```
https://insecure-website.com/static/12144.txt
```

채팅 로그나 문서를 순차적인 파일명(`12144.txt`, `12145.txt` 등)으로 저장하는 경우, 공격자가 URL의 숫자를 변경하여 다른 사용자의 파일에 접근할 수 있다.

파일명이 예측 가능하면 모든 사용자의 민감한 정보가 노출될 위험이 있다.


<div style="height: 70px;"></div>


## PortSwigger Lab 실습: IDOR을 이용한 비밀번호 탈취
---

### 실습 개요

이 실습 환경은 사용자의 채팅 로그를 서버 파일 시스템에 정적 파일로 저장하며, 순차적인 파일명을 사용한다.

**목표**: carlos 사용자의 비밀번호를 찾아 로그인하기

### 취약점 분석

Live Chat 페이지에서 채팅 내역을 저장하면 `2.txt`, `3.txt`와 같이 순차적으로 증가하는 파일명으로 저장된다. 이는 이전 파일(`1.txt`, `0.txt` 등)에 다른 사용자의 채팅 로그가 있을 가능성을 의미한다.

### 공격 과정

**1단계: 채팅 로그 다운로드 URL 확인**

Live Chat에서 "View transcript" 버튼을 클릭하면 다음과 같은 형태의 URL로 파일이 다운로드된다:
```
GET /download-transcript/2.txt HTTP/2
```

**2단계: Burp Suite Repeater를 이용한 파일 접근**

Burp Suite의 HTTP History에서 해당 요청을 Repeater로 전송한 뒤, 파일명을 `1.txt`로 변경하여 요청을 보낸다:
```
GET /download-transcript/1.txt HTTP/2
```

**3단계: 비밀번호 발견**

응답에서 carlos 사용자와 Hal Pline의 대화 기록이 출력된다.<br>
<figure>
    <img src="assets/img/251210/idor1.png" width="800" height="500" alt="Desktop View">
    <figcaption style="text-align: center; font-size: 0.9em; color: gray;">
        대화 기록에서 carlos의 패스워드가 노출되고 있다.
  </figcaption>
</figure>


**4단계: 로그인 성공**

해당 비밀번호로 carlos 계정에 로그인하면 실습이 완료된다.

<div style="height: 30px;"></div>

### 방어 방법

IDOR 취약점을 방어하기 위해서는 다음과 같은 조치가 필요하다:

**1. 권한 검증**
```python
# 취약한 코드
@app.route('/download-transcript/<filename>')
def download(filename):
    return send_file(f'/transcripts/{filename}')

# 안전한 코드
@app.route('/download-transcript/<filename>')
@login_required
def download(filename):
    # 파일 소유자 확인
    file_owner = get_file_owner(filename)
    if file_owner != current_user.id:
        abort(403)  # Forbidden
    return send_file(f'/transcripts/{filename}')
```

**2. 예측 불가능한 식별자 사용**

순차적인 숫자 대신 UUID와 같은 예측 불가능한 값을 사용한다:
```
/download-transcript/a3f8b912-4c7e-4d9a-b8e1-5f2c9d3a7e4b.txt
```

**3. 간접 참조 사용**

직접적인 파일명 대신 세션에 저장된 매핑 테이블을 사용한다:
```python
# 세션에 사용자별 파일 매핑 저장
session['user_files'] = {
    'chat_1': 'a3f8b912-4c7e-4d9a-b8e1-5f2c9d3a7e4b.txt'
}

# 사용자는 'chat_1'로 요청, 서버는 내부적으로 실제 파일명 사용
@app.route('/download-transcript/<file_key>')
def download(file_key):
    real_filename = session['user_files'].get(file_key)
    if not real_filename:
        abort(404)
    return send_file(f'/transcripts/{real_filename}')
```


<div style="height: 70px;"></div>


## 정리
---

접근 통제 취약점은 웹 애플리케이션에서 가장 흔하게 발생하는 보안 문제 중 하나다. IDOR은 그 중에서도 구현 실수로 쉽게 발생할 수 있는 취약점으로, 모든 민감한 자원에 대해 반드시 권한 검증을 수행해야 한다.

특히 파일 다운로드, 사용자 정보 조회, 계좌 조회 등 개인정보가 포함된 기능에서는 "이 사용자가 이 자원에 접근할 권한이 있는가?"를 항상 확인해야 한다.

솔직히 아직 저거 시큐어코딩하는 법 혼자서는 잘 못하겠다... 만들어둔 게시판으로 한 번만 더 실습하고 시큐어 코딩 해봐야 알 듯.