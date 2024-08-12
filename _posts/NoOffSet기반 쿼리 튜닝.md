---
layout: post
title:  "📈 쿼리 성능 개선: No Offset을 활용한 페이지네이션"
date:   2024-08-08 00:00:00 +0900
categories: Devlife
published: false
---
# 📈 쿼리 성능 개선: No Offset을 활용한 페이지네이션
## 개요
대량의 데이터를 처리하는 애플리케이션에서 쿼리 성능을 개선하는 것은 매우 중요합니다. 최근 프로젝트에서는 페이지네이션을 적용하여 성능을 최적화하기로 했습니다. 전통적인 페이지네이션 방식 대신 "No Offset" 기법을 사용하여 데이터베이스 쿼리의 성능을 크게 향상시켰습니다.

### 기존 페이지네이션 방식
기존 방식은 OFFSET과 LIMIT을 사용하는 페이지네이션 방법이었습니다. 데이터가 많은 경우, OFFSET을 증가시키면서 성능이 저하되는 문제가 있었습니다. 예를 들어, 다음과 같은 쿼리가 사용되었습니다:

```sql
SELECT *
FROM data_table
ORDER BY id ASC
OFFSET #{offset} ROWS FETCH NEXT #{limit} ROWS ONLY;
```
이 쿼리는 페이지가 변경될 때마다 OFFSET을 증가시켜야 하며, 데이터가 많아질수록 성능이 저하됩니다. OFFSET을 사용하면 데이터베이스는 스캔해야 할 레코드 수가 많아져, 성능 문제가 발생합니다.
### No Offset 방식 적용
No Offset 기법은 데이터베이스에서 OFFSET을 사용하지 않고, 마지막으로 조회된 레코드의 ID를 기준으로 다음 레코드를 조회하는 방식입니다. 이를 통해 쿼리 성능을 개선할 수 있습니다. 구현 방법은 다음과 같습니다:

```sql
SELECT *
FROM data_table
WHERE id > #{lastId}
ORDER BY id ASC
FETCH NEXT #{limit} ROWS ONLY;
```
- #{lastId}: 이전 페이지에서 마지막으로 조회된 레코드의 ID.
- #{limit}: 한 페이지에 표시할 레코드 수.
이 쿼리는 OFFSET을 사용하지 않으며, 인덱스를 효율적으로 활용합니다. 데이터베이스는 id가 #{lastId}보다 큰 레코드만 조회하면 되므로 성능이 개선됩니다.

### 적용 결과
No Offset 기법을 적용한 후, 쿼리 성능이 크게 개선되었습니다. 특히 대량의 데이터를 처리할 때 쿼리 응답 시간이 단축되어, 이 방법을 통해 데이터 조회 성능을 효과적으로 개선할 수 있었습니다.