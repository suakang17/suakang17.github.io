---
layout: post
title:  "[MySQL] access denied for user 'root'@'localhost' 에러 해결"
date:   2023-03-21 12:21:11 +0900
categories: Devlife
---

# [MySQL] access denied for user 'root'@'localhost' 에러 해결

> MySQL 8.0, MySQL workbench, Windows 10

스프링부트 스터디를 진행하고 있는데, 이제 DB를 연동하기로 해서 어제 MySQL과 MySQK workbench를 설치하고 `build.gradle`에 의존성 추가, `application.properties`에 연동 설정까지 완료했다. 연동 확인은 오늘 하려고 workbench내 sys.config 정도만 쿼리 돌려서 DB생성을 확인했는데 문제가 없었다.

그리고 오늘 본격적으로 만들어둔 spring boot와 연결을 하였으나, 잘 되던 MySQL 로그인이 갑자기 막혔다.

workbench, cmd, intellij 모든 곳에서 아래의 접근 거부 문구가 떴다.
(MySQL 자체의 문제이니 당연하다)

```
ERROR 1045 (28000): access denied for user 'root'@'localhost'
```

물론 해당 유저, 비밀번호를 가장 먼저 확인했고 이 문제는 아니였다. 
권한 설정의 문제도 아니었다. 
`\ProgramData\MySQL\MySQL Server 8.0\my.ini`를 수정하는 방법도 소용이 없었다. (애초에 권한이 더 상위의 어딘가에서 거부되는건지 수정해도 저장 자체가 불가능했다.)


## 해결 

1. '작업관리자' 열기
2. 프로세스 중 `mysqld`를 찾아서 모두 우클릭 > `작업끝내기`
3. '시스템' 열기
4. `MySQL`을 찾아 실행중이라면 모두 `서비스 중지`, 이미 중지되어있다면 5번으로 넘어가기
5. 중지된 서비스 `시작`

\+ 이 때, 4번에서 `MySQL`과, `MySQL80`이 모두 실행 중이라면 `MySQL`만 정지하고 `MySQL80`을 실행하도록 설정한다. 내 경우에는 처음부터 실행 중인 태스크가 한 개뿐이어서 그것만 중지, 시작했다.

아무튼 이후로 잘 접속되는 것을 확인했다.