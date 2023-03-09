---
layout: post
title:  "[CS] API란? Json Place Holder와 공공데이터API 사용기"
date:   2023-03-09 22:37:36 +0900
categories: Study
---

# [CS] API란? Json Place Holder와 공공데이터API 사용기

## API란? 
항상 막연하게만 느껴지던 API라는 개념이 이 쯤 되니 이해가 간다!

Application Programming Interface의 약자로 말 그대로 클라이언트-(서버가 제공하는)자원 사이에서 통신하기 위한 인터페이스의 역할을 한다.  

여기서 이 통신이라는 것도 마구잡이로 할 수 없으니 규칙을 정의하는 것이다.

클라이언트는 정해진 규칙에 맞춰 원하는 자원에 대한 요청을 날리고, 해당 요청에 대한 응답(리소스)을 반환받는다.

이 규칙이자 인터페이스가 바로 API이다.

<br />

### REST API란?

API는 규칙을 정의하는 것이니, 이 규칙에도 여러가지가 있는데, 그 중 ~~내가 알기로는~~ 현재 가장 많이 사용되는 종류이다.  
한마디로 소프트웨어 아키텍처의 종류로 이해하면 된다.

> API 개발자는 여러 아키텍처를 사용하여 API를 설계할 수 있습니다. REST 아키텍처 스타일을 따르는 API를 REST API라고 합니다. REST 아키텍처를 구현하는 웹 서비스를 RESTful 웹 서비스라고 합니다. RESTful API라는 용어는 일반적으로 RESTful 웹 API를 나타냅니다. 하지만 REST API와 RESTful API라는 용어는 같은 의미로 사용할 수 있습니다. <br />
[출처] [AWS](https://aws.amazon.com/ko/what-is/restful-api/)

<br />

타아키텍처와 비교해 REST 아키텍처가 갖는 특징은 다음과 같다. 

1. 균일한 인터페이스
<br />
통신의 편의를 위해 제약조건이 존재한다.
2. 무상태 (Statelessness)
<br />
서버는 클라이언트에 대한 정보, 상태(세션, 쿠키)를 저장하지 않는다. 따라서 서비스의 자유도가 높아지고 구현이 단순하다. HTTP가 Stateless Protocol이므로 적용된다.
3. Client-Server 구조
<br />
두 영역의 구분이 확실하다. 따라서 서버가 처리할 부분이 줄어들고, 확장성이 좋다.
4. 캐시 가능성
<br />
HTTP 특징인 캐싱 기능을 적용 가능하다.
5. 계층화 시스템 (Layered System)
<br />
REST 서버는 다중 계층으로 구성된다.
6. 자체 표현 구조 (Self-descriptiveness)
<br />
REST API 메시지만 보고도 쉽게 이해 할 수 있다.
<br />
<br />

URI 형식으로 HTTP 메서드(GET, POST, PUT, DELETE)를 요청해 자원을 조회, 생성, 수정, 삭제할 수 있는 API를 REST API라고 하는 것이다.

<br />

### HTTP 통신
자원(리소스)을 가져올 수 있도록 해주는 프로토콜, 즉 데이터 교환의 기초가 되는 통신 규칙이다.
클라이언트-서버 프로토콜로 클라이언트가 보낸 요청을 서버가 처리하고 응답을 제공한다. 

아래 두가지 문서에 정말 잘 정리되어 있다.
<br />
[HTTP 통신의 개요] [mdn web docs-overview](https://developer.mozilla.org/ko/docs/Web/HTTP/Overview)
<br />
[HTTP 요청 메서드] [mdn web docs-methods](https://developer.mozilla.org/ko/docs/Web/HTTP/Methods)

<br />

## JSON Place Holder 사용기
Postman을 이용해 API에 요청을 보내고 해당하는 응답을 받아보았다.

Json Place Holder는 구축된 API가 없을 때 API호출을 통해 불러와야 할 정형화된 데이터가 필요할 때 사용하는 곳으로 JSON 요청과 응답을 테스트할 수 있다.

### 모든 post 조회하기
> GET

<img src='/assets/img/docs/springclass2_1.png' />  

### id=2인 post 조회하기
> GET

<img src='/assets/img/docs/springclass2_2.png' />  

### id=3인 post에 대한 comments 조회하기
> GET

1. path 이용: resource를 식별하고 싶은 경우

<img src='/assets/img/docs/springclass2_3.png' />  

2. query parameter 이용: 정렬, 필터링을 하는 경우

<img src='/assets/img/docs/springclass2_4.png' />  

### post를 하나 등록하기
> POST

<img src='/assets/img/docs/springclass2_5.png' />  

요청의 body에 JSON 쌍으로 title, body, userId 등을 넣어 등록했다.

<br />

## 공공데이터포털 open API 사용기

<img src='/assets/img/docs/springclass2_6.png' />  

회원가입, 로그인 후 '에어코리아 대기오염정보 조회 서비스' 오픈 API를 통해 시도별 대기오염정보를 다음의 쿼리 파라미터들로 요청해보았다.

1. 지역 = 서울 (sidoName)
2. No = 1
3. 줄 수 = 100줄 (numOfRows)
4. 응답 형식 = JSON (returnType)
5. 발급받은 키 (serviceKey)
6. API 버전 = 1.0

오픈 API 가이드를 읽으며 정보처리기사 시험을 준비하며 공부했던 내용이 실제로 이런식으로 적용되는구나 싶기도 했다.