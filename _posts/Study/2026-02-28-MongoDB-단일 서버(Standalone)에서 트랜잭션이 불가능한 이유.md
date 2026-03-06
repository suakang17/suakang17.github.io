---
layout: post
title: "[MongoDB] 단일 서버(Standalone)에서 트랜잭션이 불가능한 이유"
date: 2026-02-28 00:00:00 +0900
categories: Study
---

## 들어가며

Spring Data JPA와 RDBMS(MySQL, Oracle 등) 환경에 익숙한 사람이 MongoDB를 도입할 때 가장 먼저 당황하는 부분 중 하나가 바로 트랜잭션이라고 한다. 

JPA에서는 단일 DB 서버만 띄워둬도 `@Transactional` 애노테이션 하나면 롤백과 커밋이 보장된다. 하지만 MongoDB 환경에서는 단일 서버(Standalone) 모드로 띄워두고 트랜잭션을 시도하면 에러가 발생하며 동작하지 않는다. 

MongoDB 공식 문서에 따르면 "다중 문서 트랜잭션은 Replica Set 또는 Sharded Cluster 환경에서만 지원된다"고 명시되어 있다. 그렇다면 왜 MongoDB는 RDBMS처럼 단일 서버 트랜잭션을 지원하지 않는 걸까? 그 기술적인 배경과 로컬 환경에서의 해결책을 정리했다.

---

## 1. 일반적인 RDBMS(JPA)와의 차이점

먼저 일반적인 RDBMS가 단일 서버에서도 트랜잭션을 완벽하게 지원하는 이유를 짚고 넘어갈 필요가 있다.

RDBMS는 애초에 단일 서버 내에서의 데이터 무결성(ACID) 보장을 최우선으로 설계되었고 한다. 트랜잭션을 처리하기 위해 로컬 디스크에 전용 로그 파일(Undo Log, Redo Log)을 생성하여 커밋과 롤백의 내역을 관리한다. 서버가 단 한 대뿐이어도 이 로그 시스템이 독립적으로 동작하므로 트랜잭션이 가능한 것이다.

반면 MongoDB는 처음부터 데이터를 여러 대의 서버에 분산하고 스케일아웃 하는 데 초점을 맞춰 설계된 NoSQL 데이터베이스다. 초기 버전에서는 다중 문서 트랜잭션 기능 자체가 존재하지 않았다.

---

## 2. 왜 MongoDB는 Replica Set이 필수일까?

MongoDB가 4.0 버전에 이르러서야 다중 문서 트랜잭션 기능이 도입됐다. 이때 MongoDB 개발진에서 단일 서버를 위한 별도의 트랜잭션 로그(RDBMS의 Undo/Redo 로그 같은 역할)를 새로 개발하지 않았다.

대신 기존에 여러 대의 서버끼리 데이터를 동기화할 때 쓰던 복제용 로그 파일인 'Oplog(Operations Log)'를 트랜잭션 로그로 재활용하는 아키텍처를 선택했다.


### 핵심은 Oplog의 유무이다
* Replica Set 환경: Primary 노드에 쓰기 작업이 발생하면 Oplog에 먼저 기록되고, Secondary 노드들이 이 로그를 가져가 동기화한다. 트랜잭션이 커밋되면, 이 트랜잭션 내의 모든 변경 사항이 Oplog에 하나의 거대한 묶음(Entry)으로 기록되어 원자성을 보장한다.
* Standalone 환경: 데이터를 복제할 대상 노드가 없으므로, 아예 시스템상에 Oplog 자체가 생성되지 않는다. 롤백이나 커밋을 기록할 곳이 없기 때문에 트랜잭션 기능이 동작할 수 없는 거다.

즉, MongoDB 아키텍처 팀은 프로덕션 환경에서는 어차피 데이터 유실 방지를 위해 무조건 Replica Set을 구성할 텐데, 굳이 개발용 단일 서버를 위해 막대한 리소스를 들여 별도의 트랜잭션 엔진을 만들 필요가 없다고 판단한 거란 생각이 든다.

---

## 3. 로컬 개발 환경에서의 해결책

원리는 이해했지만, 로컬 PC에서 트랜잭션 하나 테스트하겠다고 무거운 MongoDB 컨테이너를 3대씩 띄우는 것은 엄청난 리소스 낭비다. 

다행히 이를 우회할 수 있는 방법이 있다. 바로 단일 노드 리플리카 세트를 구성하는 것이다. 서버는 내 PC에 딱 1대만 띄우지만, 설정 옵션을 통해 `나는 혼자지만 Replica Set`이라고 데이터베이스를 속이는 방식이다. 

이렇게 하면 1대의 서버에서도 Oplog가 정상적으로 생성되어 로컬 환경에서 트랜잭션을 테스트할 수 있다.


### Docker-compose를 활용한 세팅 예시

아래와 같이 `docker-compose.yml`을 작성하여 실행하면 손쉽게 Single-node Replica Set을 구성할 수 있다.

```yaml
version: '3.8'

services:
  mongo:
    image: mongo:6.0
    container_name: local-mongo
    ports:
      - "27017:27017"
    # 리플리카 세트 이름을 'rs0'으로 지정하여 실행
    command: ["--replSet", "rs0", "--bind_ip_all", "--port", "27017"]
    healthcheck:
      test: test $$(echo "rs.initiate({_id:'rs0',members:[{_id:0,host:\"localhost:27017\"}]}).ok || rs.status().ok" | mongo --port 27017 --quiet) -eq 1
      interval: 10s
      start_period: 30s

```

컨테이너 실행 뒤, 헬스체크가 통과되면 로컬 환경에서도 완벽하게 Oplog가 활성화된 트랜잭션 가능 상태가 된다. 

이후 Spring Boot 환경에서 평소처럼 `@Transactional`을 적용하면 정상적으로 롤백과 커밋이 수행된다.
