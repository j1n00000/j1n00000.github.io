---
title: 접근 통제 취약점
date: 2025-12-10 04:50 +09:00
categories: [웹해킹, Access control vulnerabilities, portswigger lab]
tags: [웹보안, 웹해킹, Access control vulnerabilities]
---
<br>
---
<br>
## 접근 통제란?
---
<br>
접근 제어는 누가 / 무엇이 특정 동작을 수행하거나 자원에 접근할 수 있는지를 제한하는 기능이다.<br>


<br>
웹 애플리케이션에서는 인증(authentication) 과 세션 관리(session management) 를 기반으로 동작하는데,<br>
인증과 세션 관리, 접근 제어 용어를 구별하면 아래와 같다.<br>


- 인증(Authentication)<br>
사용자가 자신이 주장하는 사람임을 확인하는 과정
<br>

- 세션 관리(Session Management)<br>
같은 사용자가 보낸 이후의 HTTP 요청을 식별하는 과정
<br>

- 접근 제어(Access Control)<br>
사용자가 요청한 동작을 수행할 “권한이 있는지” 최종적으로 판단하는 기능
<br>



<br>
<br>
깨진 접근 제어(broken access control)는 아주 흔하며, 보안상 치명적인 취약점으로 이어질 수 있다.<br>
접근 제어는 기술뿐 아니라 비즈니스/조직/법적 요구사항까지 고려해야 하므로, 사람이 설계하며 오류가 발생하기 쉽다.
<br>


<br>

깨진 접근 제어는 사용자가 원래 접근할 수 없어야 하는 기능 또는 데이터에 접근할 수 있을 때 발생한다.

<br>
📌 수직적 권한 상승<br>
허용되지 않은 “상위 권한 기능”에 접근하는 경우

예: 비관리자 사용자가 관리자 페이지에 접근해서 사용자 계정을 삭제할 수 있는 상황.

<br>
📌 보호되지 않은 기능 (Unprotected Functionality)

애플리케이션이 민감한 기능에 아무 보호도 적용하지 않을 때 발생한다.

관리자 페이지가 일반 사용자 화면에는 링크가 없지만,<br>
사용자가 직접 URL을 입력하면 접근 가능할 수도 있다.

예시 URL:

https://insecure-website.com/admin


이 주소가 관리자만 볼 수 있어야 하지만, 실제로는 누구든 접근 가능한 경우가 있다.<br>
또한 다음과 같은 파일을 통해 URL이 노출될 수도 있다:

https://insecure-website.com/robots.txt


URL이 노출되지 않더라도, 공격자가 워드리스트로 경로를 추측하여 찾아낼 수 있다.

<br>
## 접근 제어 유형
---
<br>
<br>
📌 수직적 접근 제어 (Vertical Access Controls)

> 사용자 “종류”에 따라 서로 다른 기능 접근을 제한
{: .prompt-info }

예:

관리자 → 모든 사용자 계정 수정/삭제 가능

일반 사용자 → 자신의 정보만 수정 가능

업무 분리, 최소 권한 같은 정책을 기술적으로 구현한 방식이다.

<br>
📌 수평적 접근 제어 (Horizontal Access Controls)

> 같은 유형의 자원 안에서 “특정 사용자”만 접근하도록 제한
{: .prompt-info }

예:

A 사용자는 자신의 계좌만 조회 가능

B 사용자의 계좌에는 접근 불가

같은 기능이지만 “누구의 데이터냐”가 중요한 상황에서 필요하다.

<br>
📌 상황(문맥) 의존적 접근 제어 (Context-dependent Access Controls)

> 애플리케이션의 상태 또는 사용자의 행동 흐름에 따라 접근을 제한
{: .prompt-info }

예:

결제를 완료한 후에는 장바구니 내용을 수정할 수 없음

순서가 잘못된 행동을 방지하는 목적

<br>