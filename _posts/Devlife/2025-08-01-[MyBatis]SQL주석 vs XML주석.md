---
layout: post
title:  "[MyBatis]SQL주석 vs XML주석"
date:   2025-08-01 00:00:00 +0900
categories: Devlife
---
# MyBatis `<where>` 태그와 SQL 주석 vs XML 주석

## 들어가며

MyBatis를 사용하다 보면 동적 쿼리에서 `<where>` 태그를 자주 사용한다. 이 태그는 첫 번째 `AND`나 `OR`를 자동으로 제거해주는 걸로 알고 사용했다. 하지만 최근 실제 프로젝트에서 겪었던 신기한 버그를 통해 MyBatis의 동작 원리를 더 잘 이해하게 되었다.

## 문제 상황

다음과 같은 MyBatis XML 쿼리에서 SQL 구문 오류가 발생했었다:

```xml
<where>
    -- T1.STATUS = 'ACTIVE' /* 활성 상태만 조회 */
    <if test="userIds != null and userIds.size() > 0">
        AND T1.USER_ID IN
        <foreach collection="userIds" item="userId" open="(" close=")" separator=",">
            #{userId}
        </foreach>
    </if>
    AND T1.DELETE_YN = 'N'
</where>
```

**에러 메시지:**
```
키워드 'AND' 근처의 구문이 잘못되었습니다.
```

**생성된 SQL:**
```sql
WHERE -- T1.STATUS = 'ACTIVE' /* 활성 상태만 조회 */
      AND T1.USER_ID IN ('user1', 'user2')
      AND T1.DELETE_YN = 'N'
```

분명 `<where>` 태그를 사용했는데 왜 첫 번째 `AND`가 제거되지 않았을까?

## 원인 분석

### MyBatis `<where>` 태그의 동작 원리

MyBatis의 `<where>` 태그는 다음과 같이 동작한다:

1. **XML 파싱**: MyBatis가 XML을 파싱하여 동적 SQL을 생성
2. **내용 분석**: `<where>` 태그 내부의 **최종 텍스트 내용**을 분석
3. **AND/OR 제거**: 텍스트의 **시작 부분**에서 `AND` 또는 `OR` 키워드를 찾아 제거

### 문제의 핵심

```xml
-- T1.STATUS = 'ACTIVE' /* 활성 상태만 조회 */
```

이 "SQL 주석"이 문제였다!

MyBatis가 XML을 처리할 때:
- `-- T1.STATUS = 'ACTIVE'`은 **일반 텍스트**로 인식된다
- `<where>` 태그는 이 텍스트 뒤의 `AND`를 첫 번째 키워드로 인식하지 못한다
- 결과적으로 `AND` 제거 로직이 작동하지 않는다

## 해결 방법

### 방법 1: SQL 주석을 XML 주석으로 변경 (권장)

```xml
<where>
    <!-- T1.STATUS = 'ACTIVE' 활성 상태만 조회 -->
    <if test="userIds != null and userIds.size() > 0">
        AND T1.USER_ID IN
        <foreach collection="userIds" item="userId" open="(" close=")" separator=",">
            #{userId}
        </foreach>
    </if>
    AND T1.DELETE_YN = 'N'
</where>
```

**왜 이게 동작하는가?**
- `<!-- -->` XML 주석은 **XML 파싱 단계에서 완전히 제거된다**
- `<where>` 태그가 처리할 때는 주석이 존재하지 않는다
- 따라서 `AND T1.EMP_NO IN`을 첫 번째 키워드로 올바르게 인식한다

### 방법 2: 조건 순서 변경

```xml
<where>
    <if test="userIds != null and userIds.size() > 0">
        T1.USER_ID IN
        <foreach collection="userIds" item="userId" open="(" close=")" separator=",">
            #{userId}
        </foreach>
    </if>
    AND T1.DELETE_YN = 'N'
    <!-- 활성 상태 조건이 필요한 경우 추가 -->
    <!-- AND T1.STATUS = 'ACTIVE' -->
</where>
```

## MyBatis의 주석 파싱 과정

### XML 주석을 사용했을 때

```
1. XML 파싱 단계
   <!-- 주석 --> -> 완전히 제거된다
   
2. <where> 태그 처리 단계
   "AND T1.EMP_NO IN ..." -> "T1.EMP_NO IN ..." (AND 제거됨)
   
3. 최종 SQL
   WHERE T1.EMP_NO IN (...) AND T1.USE_FG = 'Y'
```

### SQL 주석을 사용했을 때

```
1. XML 파싱 단계
   "-- 주석" -> 텍스트로 유지된다
   
2. <where> 태그 처리 단계
   "-- 주석\nAND T1.USER_ID IN ..." -> AND 제거 실패
   
3. 최종 SQL
   WHERE -- 주석
         AND T1.USER_ID IN (...) <- 구문 오류!
```

## 베스트 프랙티스

### 1. 주석 사용 가이드라인

```xml
<!-- 권장: XML 주석 사용 -->
<where>
    <!-- 상태 조건 (필요시 활성화) -->
    <!-- AND T1.STATUS = 'ACTIVE' -->
    <if test="condition">
        AND T1.COLUMN = #{value}
    </if>
</where>

<!-- 피해야 할: SQL 주석을 동적 쿼리와 혼용 -->
<where>
    -- 상태 조건
    <if test="condition">
        AND T1.COLUMN = #{value}
    </if>
</where>
```

### 2. 안전한 동적 쿼리 패턴

```xml
<where>
    <!-- 항상 존재하는 기본 조건을 첫 번째로 -->
    T1.USE_FG = 'Y'
    
    <!-- 동적 조건들 -->
    <if test="employeeNos != null and employeeNos.size() > 0">
        AND T1.EMP_NO IN
        <foreach collection="employeeNos" item="empNo" open="(" close=")" separator=",">
            #{empNo}
        </foreach>
    </if>
    
    <if test="statusCode != null and statusCode != ''">
        AND T1.PFLE_STATUS_CD = #{statusCode}
    </if>
</where>
```

### 3. 조건이 모두 동적일 때

```xml
<where>
    <if test="condition1">
        CONDITION1 = #{value1}
    </if>
    <if test="condition2">
        AND CONDITION2 = #{value2}
    </if>
    <if test="condition3">
        AND CONDITION3 = #{value3}
    </if>
</where>
```

## 마무리

1. **MyBatis의 `<where>` 태그는 XML 파싱 이후의 텍스트를 기준으로 동작한다**
2. **SQL 주석(`--`)과 XML 주석(`<!-- -->`)의 처리 시점이 다르다**
3. **동적 쿼리에서는 XML 주석을 사용하는 것이 안전하다**

작은 주석 하나가 이렇게 큰 차이를 만들 줄 누가 알았을까? 이런 세밀한 부분까지 고려해야 하는 것이 묘미인 것 같다.


---

**참고 자료:**
- [MyBatis 공식 문서 - Dynamic SQL](https://mybatis.org/mybatis-3/dynamic-sql.html)
- [XML 명세 - Comments](https://www.w3.org/TR/xml/#sec-comments)
