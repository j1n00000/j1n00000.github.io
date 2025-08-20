---
title: 파일 업로드 실습 (1)
date: 2025-08-03 03:03 +09:00
categories: [보안기사, 웹]
tags: [웹보안, 웹해킹, fileupload]
---
<br>
## 실습 내용
---
<br>
25년 2회 보안기사 실기 파일 업로드 문제에서<br>
우회 방법을 서술한 답안에 애매한 부분이 있어 직접 확인하고자 APM 환경에서 실습해보았다.<br>
<br>



<figure>
    <img src="assets/img/250803/f1.png" width="800" height="500" alt="Desktop View">
    <figcaption style="text-align: center; font-size: 0.9em; color: white;">
        23년도쯤 만들다만 게시판
  </figcaption>
</figure>



<br>
문제 상황을 기반으로<br>
파일 타입이 허용되는 이미지 파일 타입이 아닌 경우 이전 화면으로 이동시키고, 그 외에는 업로드 허용한다.<br>
<br>



<figure>
    <img src="assets/img/250803/f3.png" width="800" height="500" alt="Desktop View">
    <figcaption style="text-align: center; font-size: 0.9em; color: white;">
        기존의 파일 업로드 기능을 수행하는 코드를 일부 변경.
  </figcaption>
</figure>



<br>



<figure>
    <img src="assets/img/250803/f4.png" width="800" height="500" alt="Desktop View">
    <figcaption style="text-align: center; font-size: 0.9em; color: white;">
        텍스트 파일 업로드 실패... 파일 업로드 기능 정상 수행 확인.
  </figcaption>
</figure>



<br>
<br> 이제 웹쉘을 어떻게 이미지 파일로 우회하면 되는지 확인한다. <br>



나는 "문제에 주어진 코드가 단순하게 파일 타입만 검증하고 있으니 웹 프록시 툴을 이용하여 클라이언트 측에서 변조 가능한 부분(확장자 변조 .php.img로 하는 방법 등)을 활용하여 우회할 수 있다."라는 식으로 썼었다...<br>

<br>정확하게 파일 타입을 우회하면 된다고 쓰면 됐는데, 뭉뚱그려 쓴데다 확장자 변조 부분을 끼워 쓴 부분이 깎일 요소가 있는 것 아닌가 싶어 찝찝했다.<br><br>



<figure>
    <img src="assets/img/250803/f5.png" width="800" height="500" alt="Desktop View">
    <figcaption style="text-align: center; font-size: 0.9em; color: white;">
        웹쉘을 업로드하는 요청을 인터셉트.
  </figcaption>
</figure>



<br>



<figure>
    <img src="assets/img/250803/f6.png" width="800" height="500" alt="Desktop View">
    <figcaption style="text-align: center; font-size: 0.9em; color: white;">
        파일 타입을 image/png 로 변조 후 포워드. 확장자도 .php.png로 바꿔도 영향 없없다.
  </figcaption>
</figure>



<br>



<figure>
    <img src="assets/img/250803/f7.png" width="800" height="400" alt="Desktop View">
    <figcaption style="text-align: center; font-size: 0.9em; color: white;">
        웹쉘이 성공적으로 업로드되는 것을 확인.
  </figcaption>
</figure>



<br>
<br>
역시 파일 타입을 우회해야 한다고 정확히 써야 했다. 확장자 예시를 왜 썼지... 다행히 써도 무방했다는 걸 확인하긴 했지만... 내가 채점자라면 점수 깎을 것 같다.<br><br>
<br>
쉬운 문제였는데...ㅜ 실기 1트만에 합격하기는 실패한 듯하다. 다음 시험에 꼭 합격해야지.




<br>
<br>
<br>
## 대응 방안

- 업로드 파일에 대한 파일 타입 및 확장자 검증 
  - 허용된 이미지 포맷 등만 통과시키는 화이트리스트 정책 적용

- 업로드 전용 디렉터리 분리 및 실행 제한
  - 업로드 파일을 웹루트 외부 또는 별도의 디렉터리에 저장
  - 웹서버 설정을 통해 해당 디렉터리 내 스크립트 실행 금지
  - 해당 디렉터리에 대한 직접 URL 접근 차단

- 업로드 파일의 개수 및 크기 제한
  - 요청당 업로드 파일 수 제한
  - 개별 파일 크기 또는 전체 업로드 용량 제한