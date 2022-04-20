---
layout: post
title:  "[Study]UMC 2기 - Server 강의 공부기록"
date:   2022-04-02 14:10:13 +0900
categories: Devlife
published: false
---
# [Study]UMC 2기 - Spring Boot 강의 공부기록
>UMC 2기 서버파트(spring boot) 활동을 하며 제공된 Udemy 강의를 수강한 내용 정리입니다.

## 서버 개요

## 포트포워딩 & AWS

## 리눅스 환경 구축

## DB 이론 & 설계

- 논리 적용
	- entity객체: 'User'로 유저 닌네임 이름 프사 소개글 웹사이트 링크 묶기
	- attribute속성: User의 '닉네임 이름 프사 소개글...'
	- relative관계:

- 정규화: 중복을 최소화 하는 작업
- DBMS: database Manage System

- 물리(실제 dbms 구성 위한 작업)
	- entity>table
	- attribute>각 table의 column
	- relation > PK (primary key): 테이블의 대표값, 고유한 값 (사람의 주민등록번호, 게시물마다의 고유번호(식별위함))
		- ex in IG: 유저라는 table > 닉네임 이름 소개글 +"유저인덱스"
		- 무조건 하나는 있어야함
	- FK (foreign key): 외래키
	- join (쿼리용어)

- 관계 (1:1 or 1:N or N:M)
	- ex. IG
	유저:게시물 = 1:N
	게시물:댓글 = 1:N
	유저:댓글 = 1:N

	책:작가 = N:M

- RDS: database server
	A서버 -DB
	B서버 -DB  > A,B 서버간 데이터 교류 불가

	여러개의 서버가 하나의 DB공유하는 방식 > 이런 db의 예시가 RDS

## DB 실전

- aquerytool: 웹기반 erdtool + SQL 자동생성 프로그램

## Restful API와 framework

- client -> server : request
- server -> client : response

	이 때, data가 오고가는데 이를 packet이라 함

- data의 구성 : header(목적성 가지는 metadata) + body(실 data) 
- data를 주고 받는 방식(http method)
	- Get: 조회시 사용
		- querystring: 주소 바로 뒤에 데이터 붙여 전달하는 방식으로 검색, 필터링, 페이징에서 사용
		- query: key/value 형식
	- Post: 생성시 사용
		- xml, json format
	- Put, Patch: 수정시 사용
	- Delete: 삭제시 사용

- API(Application Programming Interface)
	- Restful API
		1. method: 동사/ uri: 명사
			- ex. Post / users
				  Get / users > '특정 데이터' 조회시 path variable사용 > Get / users / 100 (100번 유저) 형태로 사용 가능
		2. 명사간의 구분자 == 하이픈
			- ex. blocked-users
		3. Get, Delete에는 body 쓰지 않기
		4. HTTP method는 실제 DB에서 동작하는 기준으로 설정하기

- framework 구조
	Route <-> Controller <-> Provider/Service <-> Dao
	- Route: 라우팅(springboot의 경우 route기능을 controller가 처리)
	- Controller: Querystring, pathvariable, body 등의 데이터 provider/service로 넘김, 형식적 validation 처리
	- Provider/Service: 비즈니스로직, transaction, 의미적 validation 처리(DB 거치는 검증 ex. 중복되는 이메일인지 등)
		- Provider: 조회 작업 수행시(작업 중 조회의 비중이 높음)
		- Service: 그 외의 작업 수행시
	- Dao: Query작성, 실행


