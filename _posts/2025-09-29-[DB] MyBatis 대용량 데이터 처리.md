# MyBatis 대용량 데이터 처리, 삽질하면서 깨달은 진실들

## 시작: OOM과의 전쟁

어느 날 서비스에서 OutOfMemoryError가 터졌다. 로그를 보니 사용자 데이터 전체를 조회하는 배치 작업에서 메모리가 터진 것이었다.

```java
// 이 코드가 문제였다
@Select("SELECT * FROM users")  // 300만 건...
List<User> selectAllUsers();
```

구글링을 해보니 대용량 데이터 처리 방법으로 **fetchSize**라는 게 있다고 한다. 

```java
@Select("SELECT * FROM users")
@Options(fetchSize = 1000)
List<User> selectAllUsers();
```

설정하고 돌려보니 확실히 메모리 사용량이 줄어들었다. 1000개씩 가져오니까 당연한 거겠지?

## 첫 번째 의문: 페이징이랑 뭐가 다른 거지?

생각해보니 이거랑 페이징이랑 뭐가 다른 거지 하는 생각이 들었다.

```java
// 페이징으로 1000건씩 가져오기
@Select("SELECT * FROM users LIMIT #{offset}, 1000")
List<User> selectUsersByPage(@Param("offset") int offset);

// fetchSize로 1000건씩 가져오기  
@Options(fetchSize = 1000)
@Select("SELECT * FROM users")
List<User> selectAllUsers();
```

둘 다 1000건씩 가져오는 건 똑같지 않은가에서 출발했다.

서버 입장에서는 비슷할 것 같은데... 1000개씩 네트워크로 보내는 건 같다고 느껴서 찾아보기 시작했다.

## 두 번째 의문: 실행되는 쿼리가 다른가?

혹시 fetchSize를 설정하면 MyBatis가 알아서 LIMIT를 붙여주는 건가? 실제로 어떤 쿼리가 나가는지 확인해봤다.

**MySQL General Log 결과:**
```sql
-- fetchSize 없을 때
SELECT * FROM users

-- fetchSize 있을 때  
SELECT * FROM users  -- 어? 똑같네?
```

어라? 쿼리는 똑같이 나간다. 그럼 fetchSize는 대체 뭘 하는 거지?

더 파보니까 이런 거였다:

```
-- fetchSize 없을 때
Client ←──────── [전체 300만건] ──────── MySQL Server
       (한 방에 다 받음, 그래서 OOM)

-- fetchSize 있을 때
Client ←─ [1000건] ─← [1000건] ─← [1000건] ─← MySQL Server  
       (조금씩 나눠서 받음)
```

쿼리는 같지만 **데이터를 받는 방식**이 다른 거였다.

## 세 번째 깨달음: 페이징과 근본적으로 다르다

그제서야 페이징과의 차이를 이해했다.

**fetchSize 방식:**
```sql
-- 서버에서 한 번만 실행
SELECT * FROM users ORDER BY id;
-- 커서로 연속해서 읽어가며 1000건씩 전송
```

**페이징 방식:**
```sql  
-- 매번 새로운 쿼리
SELECT * FROM users ORDER BY id LIMIT 0, 1000;
SELECT * FROM users ORDER BY id LIMIT 1000, 1000; 
SELECT * FROM users ORDER BY id LIMIT 2000, 1000;
-- 매번 처음부터 스캔해서 OFFSET만큼 스킵...
```

300만 건을 1000건씩 나눠서 처리한다면:
- **fetchSize**: 300만 건 정확히 한 번씩만 읽음
- **페이징**: 첫 페이지 1000건, 둘째 페이지 2000건, ... 마지막엔 300만 건 읽어야 함 (총 45억 건!)


## 네 번째 의문: 다른 방법은 없나?

fetchSize로 네트워크는 최적화했지만, 여전히 `List<User>`로 받으면 결국 메모리에 다 쌓이는 거 아닌가?

그래서 찾아본 게 **ResultHandler**다.

```java
// 기존 방식 - 메모리에 다 쌓임
@Select("SELECT * FROM users")
@Options(fetchSize = 1000)  
List<User> selectAllUsers();  // 300만개 List가 메모리에...

// ResultHandler 방식 - 하나씩 처리하고 버림
@Select("SELECT * FROM users")  
@Options(fetchSize = 1000)
void selectAllUsers(ResultHandler<User> handler);
```

```java
// 사용법
userMapper.selectAllUsers(user -> {
    processUser(user);  // 처리하고
    // user는 GC 대상이 됨, 메모리에 accumulate 안됨!
});
```

이렇게 하면:
- **fetchSize**: 네트워크 최적화 (1000건씩 받아옴)
- **ResultHandler**: 메모리 최적화 (List에 쌓지 않고 즉시 처리)

## 다섯 번째 의문: ResultSet은 왜 안 터지지?

그런데 여기서 또 궁금해졌다. ResultSet은 300만 건이 있어도 왜 메모리가 안 터지는 거지?

```java
ResultSet rs = ps.executeQuery("SELECT * FROM users");  // 300만 건
while(rs.next()) {  // 왜 안 터져?
    String name = rs.getString("name");
    processUser(name);
}
```

ArrayList와 뭐가 다른 걸까 로 다시 찾아봤다.

```java
// ArrayList - 메모리에 다 저장
List<User> users = new ArrayList<>();
while(rs.next()) {
    users.add(mapToUser(rs));  // 계속 쌓임
}
// users에 300만개 다 들어있음

// ResultSet - 현재 row만 메모리에
while(rs.next()) {  // 하나씩만 접근
    User user = mapToUser(rs);  // 현재 것만 생성
    processUser(user);
    // 다음 rs.next() 호출하면 이전 것은 사라짐
}
```

**ResultSet은 Iterator**였다. 데이터를 저장하는 게 아니라 하나씩 접근만 제공하는 거였다.

```
-- ResultSet: 커서 방식
DB Server: [300만건 저장]
    ↓ (필요할 때만 1000건씩 전송)
JDBC Buffer: [1000건만 유지]  
    ↓ (하나씩만 접근)  
Application: [현재 처리 중인 1건만]

-- ArrayList: 수집 방식
Application: [300만건 전부 저장] ← OOM!
```

## 여섯 번째 의문: 그럼 비동기 스트리밍인가?
그런데 또 의문이 든다. ResultSet이 하나씩만 메모리에 올린다면, 혹시 비동기 스트리밍으로 받아오는 건가?
만약 동기로 한 번에 다 받아온다면 결국 new ArrayList()랑 똑같은 거잖아. 서버에서 300만 건을 한 번에 보내고, 클라이언트가 한 번에 받으면 메모리 터져야 맞는데?
그럼 이런 식으로 동작하는 건가?
```java
// 혹시 이런 식으로?
CompletableFuture<User> future1 = rs.nextAsync();  // 비동기로 요청
CompletableFuture<User> future2 = rs.nextAsync();  // 비동기로 요청

future1.thenAccept(user1 -> processUser(user1));
future2.thenAccept(user2 -> processUser(user2));
```

## 일곱 번째 깨달음: JDBC는 동기다
아니었다. JDBC 자체가 동기로만 동작한다.
```java
public boolean next() throws SQLException {
    if (currentBuffer.isEmpty()) {
        // 여기서 스레드가 멈춤 (blocking)
        fetchNextBatchFromServer();  // 네트워크 I/O 대기
    }
    return moveToNextRow();
}
```

rs.next()를 호출하면:

1. 현재 버퍼가 비어있으면
2. 스레드가 blocking되고
3. 서버에 "다음 1000건 주세요" 요청
4. 응답이 올 때까지 대기
5. 받으면 다음 row로 이동

완전히 동기식이다. 비동기가 아니라 **지연 로딩(lazy loading)**인 거였다.

```
시간 흐름:
rs.next() → [blocking] → 서버 응답 → return true
rs.next() → return true (버퍼에 있음)  
rs.next() → return true (버퍼에 있음)
...
rs.next() → [blocking] → 서버 응답 → return true  (다음 배치)
```

그래서 OOM이 안 터지는 거다. 한 번에 모든 데이터를 메모리에 올리는 게 아니라, 필요할 때마다 조금씩 가져오기 때문이다.

## 정리: 각자의 역할이 다르다
결국 이런 거였다:
MyBatis의 역할

- SQL 매핑하고 객체 변환
- fetchSize 설정을 JDBC에 전달
- ResultSet을 Stream이나 List로 포장

JDBC Driver의 역할

- 실제 네트워크 통신
- 버퍼 관리
- "다음 청크 주세요" 요청

Database의 역할

- 쿼리 실행하고 커서 위치 기억
- 요청받으면 다음 청크 전송

MyBatis는 스트리밍을 모른다! 그냥 평범하게 ResultSet을 순회할 뿐이고, 실제 스트리밍은 JDBC Driver가 알아서 한다.

## 추추가
진짜 비동기가 필요하면 R2DBC 써야 한다.
```java
// R2DBC - 진짜 비동기
Flux<User> userFlux = databaseClient
    .sql("SELECT * FROM users")
    .map(row -> mapToUser(row))
    .all();

userFlux
    .doOnNext(user -> processUser(user))
    .subscribe();
```
대용량 데이터 처리할 때는 용도에 맞게 골라 쓰자:

- 전체 데이터 처리: fetchSize + ResultHandler 조합
- 사용자 UI: 페이징
- 진짜 비동기: R2DBC

삽질하면서 배운 게 제일 머리에 남는다. 🤯
