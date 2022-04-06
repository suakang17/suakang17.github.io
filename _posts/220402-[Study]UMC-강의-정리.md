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

1
계정이름
알림개수
프로필사진
계정설명
계정아이디
게시물수
팔로워 팔로잉 수
올린 게시물


2
계정아이디
프로필사진
게시물 내용(사진, 설명, 올린 시간)
좋아요 개수
댓글 개수

3
계정아이디, 프로필사진
게시글 설명(설명, 시간)
댓글 단 계정의 아이디, 프로필사진
댓글에 대한 좋아요 수
댓글 내용
대댓글 여부


>논리 적용
entity객체: 'User'로 유저 닌네임 이름 프사 소개글 웹사이트 링크 묶기
attribute속성: User의 '닉네임 이름 프사 소개글...'
relative관계:

>정규화: 중복을 최소화 하는 작업
>DBMS: database Manage System

>물리(실제 dbms 구성 위한 작업)
entity>table
attribute>각 table의 column
relation > PK (primary key): 테이블의 대표값, 고유한 값 (사람의 주민등록번호, 게시물마다의 고유번호(식별위함))
		ex in IG: 유저라는 table > 닉네임 이름 소개글 +"유저인덱스"
		무조건 하나는 있어야함
	 FK (foreign key): 외래키
	join (쿼리용어임):

>관계 (1:1 or 1:N or N:M)
ex)IG
유저:게시물 = 1:N
게시물:댓글 = 1:N
유저:댓글 = 1:N

책:작가 = N:M

>RDS: database server
A서버 -DB
B서버 -DB  > A,B 서버간 데이터 교류 불가

여러개의 서버가 하나의 DB공유하는 방식 > 이런 db의 예시가 RDS

## DB 실전

aquerytool: 웹기반 erdtool + SQL 자동생성 프로그램

