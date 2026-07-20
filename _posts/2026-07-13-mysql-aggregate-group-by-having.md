---
title: "COUNT(전화번호)로 회원 수를 셌다가 13명이 사라진 이유 — SQL 집계와 그룹핑 정리"
date: 2026-07-13 23:40:00 +0900
categories: [백엔드, 데이터베이스]
tags: [mysql, sql, group-by]
image:
  path: /assets/img/posts/mysql-aggregate-group-by-having/thumbnail.webp
  alt: 전화번호가 빈 행을 표시하고 COUNT(*)와 COUNT(phone) 결과 차이를 적은 모눈종이 메모
---

인프런 김영한님 실전 데이터베이스 입문 섹션 7을 정리한다. 섹션 6까지는 행 하나하나를 조회하고 가공하는 법을 배웠는데, 이번 섹션은 관점이 완전히 바뀌었다. 여러 행을 하나로, 또는 여러 그룹으로 요약하는 법이다. 집계 함수, GROUP BY, GROUP BY의 함정, HAVING, 그리고 마지막에 이 모든 걸 하나로 꿰어주는 SQL 실행 순서까지 여섯 꼭지였는데, 배우는 내내 "분명 아까 배운 규칙인데 왜 여기선 또 헷갈리지"를 반복했다. 헷갈렸던 지점 위주로 최대한 자세히 남겨본다.

# 집계 함수 — COUNT, SUM, AVG, MAX, MIN

## COUNT(*) vs COUNT(컬럼) — 회원 수를 셌다가 13명이 사라진 사건

members 테이블에서 "전체 회원이 몇 명이냐"는 질문에 답하려고 별생각 없이 이렇게 썼다.

```sql
SELECT COUNT(phone) FROM members;
```
```
87
```

실제 회원은 100명인데 87명이 나왔다. 처음엔 쿼리를 잘못 짰나 싶어서 몇 번을 다시 돌렸는데 계속 87이었다. 알고 보니 전화번호를 아직 등록 안 한 회원이 13명 있었다.

```sql
SELECT COUNT(*) FROM members;         -- 100
SELECT COUNT(phone) FROM members;      -- 87
SELECT COUNT(*) - COUNT(phone);        -- 13 (NULL인 phone 개수)
```

`COUNT(*)`는 NULL 여부를 안 따지고 행 자체의 개수를 세는데, `COUNT(컬럼명)`은 그 컬럼이 NULL이 아닌 행만 센다는 걸 이 사고를 치고 나서야 몸으로 익혔다. 헷갈리는 변형까지 표로 정리해봤다.

| 표현 | 의미 | 이 예제에서 결과 |
|---|---|---|
| `COUNT(*)` | 행 자체의 개수 (NULL 상관없이) | 100 |
| `COUNT(1)` | `COUNT(*)`와 완전히 동일 (상수도 NULL이 아니므로) | 100 |
| `COUNT(phone)` | phone이 NULL 아닌 행만 | 87 |
| `COUNT(id)` (id가 PK) | PK는 NULL이 없으므로 `COUNT(*)`와 같음 | 100 |
| `COUNT(DISTINCT phone)` | phone 중 NULL 제외하고 중복도 제거한 개수 | 87 이하 |

`COUNT(1)`이 `COUNT(*)`와 똑같이 동작한다는 것도 찾아보고 알았다. 예전엔 `COUNT(1)`이 더 빠르다는 얘기가 돌았다는데, 요즘 MySQL 옵티마이저는 두 표현을 똑같이 최적화해서 실제 성능 차이는 없다고 한다. 그리고 `COUNT(*)`가 인덱스 없는 큰 테이블에서 생각보다 느릴 수 있다는 것도 알아뒀다. InnoDB는 MyISAM과 달리 테이블의 정확한 행 개수를 미리 캐싱해두지 않아서, `COUNT(*)`를 실행할 때마다 실제로 전체(또는 가장 작은 인덱스)를 훑어야 한다고 한다. 나중에 대용량 테이블을 다루게 되면 이 부분도 신경 써야겠다.

![COUNT(*)는 전체 행, COUNT(컬럼)은 NULL 제외 개수를 센다](/assets/img/posts/mysql-aggregate-group-by-having/count-star-vs-count-column-null.png)

## SUM / AVG / MAX / MIN — NULL은 계산에서 통째로 빠진다

이번엔 orders 테이블의 amount로 매출 통계를 냈다. 취소된 주문은 amount가 NULL로 들어있는 상태였다.

```sql
SELECT
    SUM(amount)              AS total,
    ROUND(AVG(amount), 0)    AS avg_amount,
    MAX(amount)               AS max_amount,
    MIN(amount)               AS min_amount,
    COUNT(*)                  AS all_orders,
    COUNT(amount)             AS paid_orders
FROM orders;
```
```
total  | avg_amount | max_amount | min_amount | all_orders | paid_orders
76000  | 19000      | 30000      | 9000       | 5          | 4
```

전체 주문은 5건인데 paid_orders는 4건이다. SUM과 AVG 모두 NULL인 취소 주문 1건을 빼고 계산했다는 뜻이다. 여기서 제일 헷갈렸던 포인트를 표로 정리해봤다.

| 헷갈리는 계산 | 실제 값 | 왜 이렇게 되나 |
|---|---|---|
| `SUM(amount) / COUNT(*)` | 76000 / 5 = 15200 | ❌ 틀린 평균 — 취소 건까지 분모에 넣음 |
| `SUM(amount) / COUNT(amount)` | 76000 / 4 = 19000 | ✅ `AVG(amount)`와 정확히 같은 값 |

`AVG`는 내부적으로 `SUM(amount) / COUNT(amount)`로 계산되지, `COUNT(*)`로 나누는 게 아니다. 이걸 헷갈리면 "평균이 왜 이렇게 높게 나왔지"라고 엉뚱한 데서 버그를 찾게 될 것 같아서 표로 확실히 박아뒀다.

왜 하필 0으로 안 치고 아예 빼버리는지 찾아보니, NULL은 "값이 0이다"가 아니라 "값을 아직 모른다"는 뜻이라서 그렇다고 한다. 결시자를 0점으로 넣고 평균 내면 실제로 시험 본 사람들의 평균이 왜곡되는 것과 같은 이치다. 배달의민족 가게 평점도 이 원리로 동작한다고 들었는데, 별점을 아직 안 남긴 리뷰(댓글만 있는 리뷰)를 0점으로 치지 않고 아예 평균 계산에서 빼기 때문에 평점이 실제 만족도보다 낮게 왜곡되지 않는다고 한다.

강의 범위 밖이지만 하나 더 확인해본 게 있다. `AVG` 결과에 소수점이 길게 나올 때가 있어서, `ROUND(AVG(amount), 0)`처럼 반올림 자릿수를 지정하는 습관을 들여야 한다는 것과, amount 컬럼 타입이 `DECIMAL`이 아니라 `FLOAT`이면 부동소수점 오차로 평균값이 미세하게 어긋날 수 있다는 것도 찾아봤다. 돈 관련 컬럼은 `FLOAT` 대신 `DECIMAL`을 써야 하는 이유가 여기서도 한 번 더 확인된 셈이다.

![SUM/AVG/MAX/MIN은 NULL을 계산에서 제외한다](/assets/img/posts/mysql-aggregate-group-by-having/sum-avg-null-exclusion.png)

## COUNT(DISTINCT ...) — 중복 없이 세기

"몇 명의 고객이 실제로 주문했는가"를 구하려면 customer_id를 그냥 세면 안 된다. 같은 고객이 여러 번 주문했을 수 있으니까.

```sql
SELECT
    COUNT(customer_id)          AS total_orders,
    COUNT(DISTINCT customer_id)  AS unique_customers
FROM orders;
```
```
total_orders | unique_customers
6            | 3
```

DISTINCT가 먼저 중복을 제거한 결과 집합을 만들고, 그 다음에 COUNT가 그 집합의 크기를 센다는 순서로 이해하니 헷갈리지 않았다. 쿠팡 같은 이커머스 관리자 대시보드에서 "총 주문 건수"와 "주문 고객 수"를 둘 다 보여주는 게 바로 이 COUNT(*)와 COUNT(DISTINCT customer_id)의 차이라는 것도 이해가 됐다.

![DISTINCT가 먼저 중복을 제거하고 그 결과를 COUNT가 센다](/assets/img/posts/mysql-aggregate-group-by-having/distinct-count-combination.png)

# GROUP BY — 그룹으로 나눠서 각각 요약하기

## 카테고리별로 나눠서 SUM 계산하기

집계 함수 하나만으로는 "전체 매출"밖에 못 구한다. "카테고리별 매출"이 필요하면 GROUP BY가 필요했다.

```sql
SELECT category, SUM(amount) AS total, COUNT(*) AS cnt
FROM orders
GROUP BY category;
```
```
category | total  | cnt
가전     | 45000  | 3
식품     | 8000   | 2
의류     | 8000   | 1
```

GROUP BY가 먼저 category 값이 같은 행끼리 그룹을 만들고, 그 다음에 그룹마다 SUM과 COUNT를 따로 계산한다. 원본이 6행이었는데 결과가 3행(그룹 개수)으로 줄어드는 걸 보고 "아 이게 그룹으로 압축한다는 거구나"를 실감했다.

| 단계 | 하는 일 | 이 예제 결과 |
|---|---|---|
| ① 그룹 나누기 | category 값이 같은 행끼리 묶음 | 가전 3행 / 식품 2행 / 의류 1행 |
| ② 그룹마다 집계 | 각 그룹에 SUM, COUNT 적용 | 가전 45000·3건 / 식품 8000·2건 / 의류 8000·1건 |
| ③ 결과 행 수 | 원본 행 수 → 그룹 개수로 압축 | 6행 → 3행 |

![GROUP BY는 값이 같은 행끼리 묶고 그룹마다 집계 함수를 계산한다](/assets/img/posts/mysql-aggregate-group-by-having/group-by-category-flow.png)

## 다중 컬럼 GROUP BY — 조합이 같아야 같은 그룹

지역별 담당자용 리포트를 만들려고 category와 region 두 컬럼으로 그룹을 나눠봤다.

```sql
SELECT region, category, COUNT(*) AS order_count, SUM(amount) AS total
FROM orders
GROUP BY region, category;
```

category만 같다고 묶이는 게 아니라 (region, category) 조합 전체가 완전히 같아야 같은 그룹이 된다는 걸 확인했다. 그러다가 실수로 GROUP BY에 order_id(PK)를 같이 넣어본 적이 있는데, 그룹이 사실상 하나도 안 묶이고 원본 행 수랑 거의 똑같이 나왔다.

| GROUP BY 대상 | 그룹이 묶이는 기준 | 결과 행 수 |
|---|---|---|
| `GROUP BY category` | category 값 하나 | 카테고리 개수만큼 |
| `GROUP BY category, region` | (category, region) 조합 | 조합 개수만큼 |
| `GROUP BY category, order_id` | (category, order_id) 조합 — order_id가 전부 달라서 사실상 조합도 다 다름 | 원본 행 수와 거의 동일 (그룹이 안 묶인 것과 같음) |

order_id는 행마다 다 다른 값이라서, (category, order_id) 조합 자체가 사실상 모든 행에서 다 다르기 때문이었다. 그룹핑 기준에 유일값(unique) 컬럼을 섞으면 그룹이 원본 행 단위로 쪼개진다는 걸 이 실수로 배웠다.

강의에는 없었지만 찾아보다가 `WITH ROLLUP`이라는 걸 알게 됐다. GROUP BY 뒤에 붙이면 각 그룹 소계와 전체 합계까지 한 번에 뽑아준다고 한다.

```sql
SELECT category, SUM(amount) AS total
FROM orders
GROUP BY category WITH ROLLUP;
```
```
category | total
가전     | 45000
식품     | 8000
의류     | 8000
NULL     | 61000   ← 전체 합계 행 (category가 NULL로 표시됨)
```

마지막 행의 category가 NULL로 나오는 게 "전체 합계"라는 뜻이라는 걸 모르면 또 헷갈릴 것 같다. 엑셀의 부분합 기능이랑 비슷한 결과를 SQL 한 줄로 뽑을 수 있다는 게 신기했다.

![다중 컬럼 GROUP BY는 컬럼 조합 전체가 같아야 같은 그룹이 된다](/assets/img/posts/mysql-aggregate-group-by-having/group-by-multi-column-structure.png)

# GROUP BY 주의사항 — SELECT에 아무 컬럼이나 못 쓰는 이유

## ERROR 1055 재현 — 상품명을 같이 보고 싶었을 뿐인데

카테고리별 매출에 상품명까지 같이 보여주고 싶어서 SELECT에 그냥 추가했다가 바로 에러를 만났다.

```sql
SELECT category, product_name, SUM(amount)
FROM orders
GROUP BY category;
```
```
ERROR 1055 (42000): Expression #2 of SELECT list is not in GROUP BY clause
and contains nonaggregated column 'shop.orders.product_name' which is
not functionally dependent on columns in GROUP BY clause
```

가전 그룹 안에는 냉장고, TV, 세탁기가 다 섞여 있는데, product_name을 그냥 SELECT에 넣으면 이 셋 중 뭘 대표로 보여줘야 하는지 DB가 판단할 방법이 없다. SELECT에 뭘 쓸 수 있는지 판별 기준을 표로 정리해봤다.

| SELECT에 쓰려는 것 | 허용 여부 | 이유 |
|---|---|---|
| GROUP BY에 나열한 컬럼 (`category`) | ✅ | 그룹 안에서 값이 하나로 확정됨 |
| 집계 함수로 감싼 값 (`SUM(amount)`) | ✅ | 여러 값을 하나로 압축해서 계산함 |
| 그룹핑 기준도 집계 함수도 아닌 컬럼 (`product_name`) | ❌ | 그룹 안에 여러 값이 남아있어 대표값을 못 정함 |
| PK 컬럼으로 그룹핑했을 때의 다른 컬럼 | ✅ (예외) | PK 값이 정해지면 나머지도 자동으로 하나로 확정(함수적 종속) |

정말 그 값이 필요하면 MAX/MIN 같은 집계 함수로 감싸거나, "아무거나 하나면 된다"는 뜻으로 MySQL의 ANY_VALUE를 쓸 수 있다.

```sql
SELECT category, ANY_VALUE(product_name) AS sample_product, SUM(amount)
FROM orders
GROUP BY category;
```

다만 ANY_VALUE는 정말 아무 값이나 상관없을 때만 써야지, "귀찮아서 에러만 없애자"는 식으로 쓰면 나중에 "이 상품명이 왜 이렇게 나왔지"라는 질문에 아무도 답 못하는 상황이 생긴다고 한다. 나중에 윈도우 함수(아직 안 배웠지만)를 쓰면 "카테고리별로 가장 비쌌던 상품명"처럼 정확한 값을 구할 수 있다는 것도 찾아보면서 알게 됐는데, 이건 다음에 제대로 배워봐야 할 것 같다.

![그룹 안에 값이 여러 개 남은 컬럼은 대표값을 정할 수 없어 에러가 난다](/assets/img/posts/mysql-aggregate-group-by-having/group-by-select-restriction-error.png)

## 함수적 종속 — PK로 그룹핑하면 예외인 이유, 그리고 진짜 헷갈렸던 두 가지

신기했던 건, GROUP BY id(PK)로 묶으면 다른 컬럼도 자유롭게 SELECT할 수 있다는 점이었다.

```sql
SELECT id, name, email
FROM members
GROUP BY id;   -- name, email이 GROUP BY에 없어도 에러 안 남
```

id가 PK라서 id 하나당 행이 정확히 하나뿐이니, id가 정해지면 name과 email도 자동으로 하나로 확정된다. 이걸 함수적 종속이라고 부른다는 걸 찾아보고 알았다.

여기서 스스로 두 가지가 헷갈려서 직접 실험해봤다.

**첫 번째 헷갈림: 그룹 안의 값이 우연히 다 똑같아도 에러가 날까?**

```sql
-- category로만 그룹핑, subcategory는 그룹핑 기준 아님
-- 근데 실제 데이터가 우연히 이렇다면?
┌──────────┬─────────────┬────────┐
│ category │ subcategory │ amount │
├──────────┼─────────────┼────────┤
│ 가전     │ 대형가전    │ 10000  │
│ 가전     │ 대형가전    │ 20000  │   ← subcategory가 우연히 전부 동일
│ 가전     │ 대형가전    │ 15000  │
└──────────┴─────────────┴────────┘

SELECT category, subcategory, SUM(amount)
FROM orders
GROUP BY category;
```

직접 돌려보니 여전히 `ERROR 1055`가 났다. **두 번째 헷갈림: 그룹에 행이 딱 1개뿐이라면?**

```sql
-- orders 테이블에 이 행 하나만 있는 상황
┌──────────┬─────────────┬────────┐
│ category │ subcategory │ amount │
├──────────┼─────────────┼────────┤
│ 가전     │ 대형가전    │ 10000  │
└──────────┴─────────────┴────────┘

SELECT category, subcategory, SUM(amount)
FROM orders
GROUP BY category;
```

이것도 여전히 `ERROR 1055`였다.

| 헷갈렸던 상황 | 기대했던 것 | 실제 결과 | 이유 |
|---|---|---|---|
| 그룹 안 subcategory 값이 우연히 전부 동일 | 다 똑같으니 통과되지 않을까? | ❌ 여전히 에러 | MySQL은 지금 데이터가 아니라 테이블 구조(스키마)만 보고 판단 |
| 그룹에 행이 딱 1개뿐 | 1개면 대표값이 명확하지 않을까? | ❌ 여전히 에러 | 내일 같은 category로 행이 하나 더 들어오면 바로 애매해지는 구조라서 |

이유를 찾아보니, MySQL은 쿼리를 실행하기 전에 **테이블 구조만 보고** "category 값이 정해지면 subcategory 값도 무조건 하나로 정해지는가?"를 판단한다. 이건 스키마에 `UNIQUE` 제약이나 PK 관계가 선언돼 있어야만 "그렇다"고 인정하는 정적(static) 검사지, 지금 저장된 실제 데이터를 스캔해서 "어, 우연히 다 똑같네, 봐줄게" 하고 판단하는 동적(dynamic) 검사가 아니다.

이유가 명확해서 납득이 됐다. 지금은 우연히 값이 같거나 행이 1개뿐이지만, 내일 누군가 `INSERT INTO orders VALUES ('가전', '소형가전', 5000);` 같은 행을 넣으면 그 순간 이 쿼리는 애매해진다. 만약 데이터 검사로 통과시켰다면, 어제는 되던 쿼리가 오늘은 안 되는 상황이 생기는 거다. 그래서 SQL은 "지금 우연히 괜찮은가"가 아니라 "구조적으로 항상 보장되는가"만 확인한다는 걸 이 두 실험으로 확실히 이해했다.

## ONLY_FULL_GROUP_BY — 옛날 MySQL은 이 에러 자체가 없었다

이 규칙이 MySQL 5.7부터 기본으로 강제된다는 것도 찾아보고 알았다.

| DB / 버전 | GROUP BY 규칙 강제 여부 |
|---|---|
| MySQL 5.6 이하 (기본 설정) | 강제 안 함 — 위반해도 에러 없이 그룹의 아무 행이나 실행됨 |
| MySQL 5.7 이상 (기본 설정) | 강제함 (`ONLY_FULL_GROUP_BY` 기본 활성) |
| PostgreSQL | 항상 강제 |
| Oracle / SQL Server | 항상 강제 |

문제는 5.6 이하에서 같은 쿼리를 실행할 때마다 결과가 달라질 수 있다는 거다. 옛날 시스템을 최신 MySQL로 마이그레이션하면 예전엔 없던 이 에러가 갑자기 쏟아지는 경우가 실무에서 흔하다고 하는데, 회사 프로젝트 DB 버전이 뭔지 먼저 확인하는 습관을 들여야겠다고 메모해뒀다.

```sql
SELECT @@sql_mode;   -- 현재 서버가 이 규칙을 강제하는지 확인하는 방법

-- 응급처치 (근본 해결 아님, 세션 단위로만 규칙 끄기)
SET SESSION sql_mode = (SELECT REPLACE(@@sql_mode, 'ONLY_FULL_GROUP_BY', ''));
```

설정을 끄는 건 "예전처럼 애매한 값이라도 아무거나 받겠다"는 뜻이라, 결과가 실행할 때마다 달라질 수 있는 근본 문제는 그대로 남는다. 진짜 해결은 문제가 된 쿼리를 열어서 GROUP BY에 없는 컬럼을 MAX()/MIN() 등으로 감싸거나 쿼리 자체를 다시 설계하는 거라고 한다.

![MySQL 5.6과 8.0은 GROUP BY 규칙을 강제하는 기본값이 다르다](/assets/img/posts/mysql-aggregate-group-by-having/only-full-group-by-version-comparison.png)

# HAVING — WHERE로 안 되는 걸 되게 하는 절

## WHERE SUM(amount) > 30000 이 왜 에러가 났을까

카테고리별 매출 중 3만원 넘는 것만 보고 싶어서 익숙하게 WHERE부터 썼다.

```sql
SELECT category, SUM(amount) AS total
FROM orders
GROUP BY category
WHERE SUM(amount) > 30000;
```
```
ERROR: Invalid use of group function
```

이유를 찾아보니 WHERE는 그룹으로 묶이기 전, 원본 행 단위로 동작하는 절이었다. 그 시점엔 SUM(amount)이라는 값 자체가 아직 계산되지 않은 상태라, 존재하지도 않는 값을 조건으로 걸려니 에러가 난 거였다. 그룹으로 다 묶고 집계까지 끝난 다음에 그 결과를 걸러내는 전용 문법이 필요했는데, 그게 HAVING이었다.

```sql
SELECT category, SUM(amount) AS total
FROM orders
GROUP BY category
HAVING SUM(amount) > 30000;
```
```
category | total
가전     | 45000
```

| | WHERE | HAVING |
|---|---|---|
| 동작 대상 | 그룹으로 묶이기 전, 원본 행 | 그룹으로 묶이고 집계까지 끝난 결과 |
| 집계 함수(SUM/COUNT 등) 사용 | ❌ 불가 | ✅ 가능 |
| 실행 시점 | GROUP BY보다 먼저 | GROUP BY보다 나중, SELECT보다 먼저 |

![WHERE는 그룹 나누기 전, HAVING은 그룹 나눈 후에 실행된다](/assets/img/posts/mysql-aggregate-group-by-having/where-vs-having-pipeline.png)

이커머스의 "우수 셀러 선정" 로직이 대부분 `GROUP BY seller_id HAVING SUM(sales) >= 기준금액` 형태라는 걸 알고 나니, 지금까지 뭔가 신기해 보였던 관리자 대시보드 숫자들이 다 이 조합으로 만들어진 거였구나 싶었다. 이상 트래픽 탐지도 `GROUP BY ip HAVING COUNT(*) > 100` 같은 형태로, 중복 가입 탐지는 `GROUP BY email HAVING COUNT(*) > 1`로 찾는다고 하는데, 셋 다 결국 "그룹으로 묶고 집계 조건으로 거른다"는 같은 패턴이라는 게 재밌었다.

## HAVING에서 별칭을 쓸 수 있을까 — DB마다 다르고, WHERE·GROUP BY·ORDER BY까지 다 다르다

`HAVING SUM(amount) > 30000` 매번 쓰기 귀찮아서 SELECT에서 붙인 별칭 total을 재사용해봤다.

```sql
SELECT category, SUM(amount) AS total
FROM orders
GROUP BY category
HAVING total > 30000;   -- MySQL에서는 됨
```

MySQL에서는 별 문제 없이 됐는데, 나중에 이 프로젝트가 PostgreSQL로 옮겨갈 수도 있다는 얘기를 듣고 찾아보니 PostgreSQL에서는 이 코드가 `column "total" does not exist` 에러를 낸다고 한다. 왜 절마다 지원 여부가 다른지 전체를 표로 정리하고 나서야 감이 잡혔다.

| 절 | MySQL | PostgreSQL | Oracle | SQL Server | 표준 SQL |
|---|---|---|---|---|---|
| WHERE | ✗ | ✗ | ✗ | ✗ | ✗ (예외 없음) |
| GROUP BY | ✓ | ✓ | ✗ | ✗ | ✗ (일부 벤더 확장) |
| HAVING | ✓ | ✗ | ✗ | ✗ | ✗ (MySQL만 편의 확장) |
| ORDER BY | ✓ | ✓ | ✓ | ✓ | ✓ (표준이 보장하는 유일한 절) |

이유는 SQL의 논리적 실행 순서에 있었다.

```
작성 순서: SELECT → FROM → WHERE → GROUP BY → HAVING → ORDER BY
실행 순서: FROM → WHERE → GROUP BY → HAVING → SELECT → ORDER BY
                                              ↑
                                      별칭(AS)이 여기서 생김
```

WHERE는 SELECT보다 3단계나 먼저 실행돼서 별칭 얘기 자체가 성립할 수 없고, HAVING은 1단계 차이라 MySQL 정도만 편의로 봐주고, ORDER BY는 SELECT보다 뒤라서 표준 자체가 별칭 사용을 보장한다. "SELECT와 실행 순서가 몇 단계 떨어져 있는가"가 딱 이 표를 설명하는 기준이라는 걸 알고 나니 외울 필요 없이 이해로 넘어갔다.

여러 DB를 오갈 가능성이 있는 프로젝트라면 HAVING에도 항상 원래 집계식을 그대로 쓰는 게 안전하다고 정리했다.

![HAVING에서 SELECT 별칭 재사용은 MySQL만 허용하는 편의 기능이다](/assets/img/posts/mysql-aggregate-group-by-having/having-alias-portability-comparison.png)

## WHERE로 옮길 수 있는 조건을 HAVING에 남겨두면 손해

"가전 카테고리만"이라는 조건은 집계 함수가 필요 없다. 이걸 WHERE에 두든 HAVING에 두든 결과는 똑같이 나오는데, 실행 순서를 알고 나니 왜 WHERE에 두는 게 맞는지 이해가 됐다.

```sql
-- 권장: WHERE로 미리 거름 → 그룹핑 대상 자체가 3행으로 줄어듦
SELECT category, SUM(amount) AS total
FROM orders
WHERE category = '가전'
GROUP BY category;

-- 비효율: HAVING으로 나중에 거름 → 일단 전체 6행을 3개 그룹으로 다 계산한 다음 버림
SELECT category, SUM(amount) AS total
FROM orders
GROUP BY category
HAVING category = '가전';
```

데이터가 몇 만 건이면 티도 안 나겠지만, 수백만 건짜리 로그 테이블이면 이 순서 하나가 실행 시간을 몇 배로 갈라놓을 수 있다고 한다. `EXPLAIN`을 앞에 붙여서 두 쿼리의 실행 계획을 비교해보면, WHERE로 거른 쪽이 그룹핑 전 처리한 행(rows) 수가 훨씬 적게 나온다고 하니 나중에 직접 대용량 테이블로 비교해봐야겠다. "이 조건에 집계 함수가 필요한가?"만 스스로 물어보면 어디에 둬야 할지 헷갈리지 않겠다.

# SQL 실행 순서 — 작성 순서와 계산 순서는 다르다

섹션 7의 마지막 꼭지는 지금까지 배운 규칙들을 하나로 꿰어주는 정리 시간이었다. 지금까지 `SELECT ... FROM ... WHERE ... GROUP BY ... HAVING ... ORDER BY ... LIMIT` 순서로 "써왔지만", DB가 실제로 계산하는 순서는 이거였다.

| 순서 | 작성 순서 | 실행 순서 |
|---|---|---|
| 1 | SELECT | FROM |
| 2 | FROM | WHERE |
| 3 | WHERE | GROUP BY |
| 4 | GROUP BY | HAVING |
| 5 | HAVING | SELECT |
| 6 | ORDER BY | DISTINCT |
| 7 | LIMIT | ORDER BY |
| 8 | — | LIMIT |

SELECT가 쓰기로는 맨 앞인데 실행은 거의 마지막(ORDER BY 바로 전)이라는 딱 이 차이 하나가, 지금까지 겪은 헷갈림을 전부 설명해줬다.

| 지금까지 헷갈렸던 규칙 | 실행 순서로 설명하면 |
|---|---|
| WHERE에 집계 함수를 못 씀 | WHERE(2번)가 GROUP BY(3번)보다 먼저 실행 → 아직 집계값이 없음 |
| HAVING에서 별칭을 표준적으로 못 씀 | HAVING(4번)이 SELECT(5번)보다 먼저 실행 → 아직 별칭이 없음 |
| GROUP BY 없이 일반 컬럼을 SELECT 못 씀 | SELECT(5번)가 실행될 땐 이미 그룹으로 다 묶인 뒤라 대표값을 못 정함 |
| ORDER BY에서는 별칭을 자유롭게 씀 | ORDER BY(7번)는 SELECT(5번)보다 뒤라서 이미 만들어진 별칭을 그대로 참조 가능 |

## DISTINCT + GROUP BY — 서로 다른 그룹에서 나온 값이 합쳐지는 걸 보고 놀랐던 순간

여기서 제일 헷갈렸던 실험을 하나 남겨둔다. orders 테이블에 이런 데이터가 있었다.

```
┌──────────┬──────────────┬────────┐
│ category │ product_name │ amount │
├──────────┼──────────────┼────────┤
│ 가전     │ 냉장고       │ 10000  │
│ 가전     │ 냉장고       │ 15000  │
│ 가전     │ TV           │ 20000  │
│ 가전     │ TV           │ 5000   │
│ 식품     │ 라면         │ 3000   │
└──────────┴──────────────┴────────┘
```

```sql
SELECT DISTINCT category, COUNT(*) AS cnt
FROM orders
GROUP BY category, product_name;
```

먼저 DISTINCT 없이 실행해보면 3행이 나온다.

```
category | cnt
가전     | 2    ← 냉장고 그룹 (2건)
가전     | 2    ← TV 그룹 (2건, product_name만 뺐더니 위 행과 완전히 같아짐)
식품     | 1
```

여기에 DISTINCT를 붙이면 2행으로 줄어든다.

```
category | cnt
가전     | 2
식품     | 1
```

처음엔 "가전에서 중복을 없애거나 GROUP BY의 그룹 겹침을 정리해주는 건가?"라고 생각했는데, 그게 아니었다. GROUP BY(3번)는 이미 완전히 끝나서 사라진 뒤였다. SELECT(5번)가 product_name을 빼고 만든 최종 표(3행)만 DISTINCT(6번)에게 넘어가고, DISTINCT는 그 표를 GROUP BY 결과라는 사실조차 모른 채 그냥 "값이 완전히 같은 행끼리 지우기"만 했다. 냉장고 그룹에서 온 값인지 TV 그룹에서 온 값인지는 이 시점엔 이미 아무 의미가 없다는 걸 직접 재현해보고 나서야 확실히 이해했다.

이 예제 하나로 "GROUP BY는 이미 끝난 일이고, DISTINCT는 그 결과를 평범한 행 목록으로만 본다"는 게 확실해졌다. GROUP BY와 DISTINCT가 서로 상호작용한다는 개념 자체가 없다는 게 핵심이었다.

![작성 순서는 SELECT가 맨 앞이지만 실행 순서는 SELECT가 거의 마지막이다](/assets/img/posts/mysql-aggregate-group-by-having/sql-writing-vs-execution-order.png)

기술 면접에서 "SQL 실행 순서를 설명해보세요"가 단골 질문이라는 걸 찾아보고 알았는데, 그냥 순서만 외워서 나열하는 것보다 "그래서 WHERE엔 집계 함수를 못 쓰고, HAVING의 별칭은 MySQL만 봐주는 거다"처럼 규칙과 연결해서 설명하면 훨씬 이해도가 드러날 것 같다.

# 마무리

섹션 7은 "행 하나하나"에서 "여러 행의 요약"으로 관점이 완전히 바뀌는 구간이었다. COUNT(전화번호)로 회원 수를 셌다가 13명이 사라진 사건이 이번 섹션에서 제일 오래 기억에 남을 것 같다. 별거 아닌 것 같은 괄호 안 컬럼 하나 차이가 실제 서비스라면 "회원 수가 갑자기 줄어든 것처럼 보이는 버그"로 이어질 수 있다는 걸 직접 겪어보니 확실히 각인됐다.

WHERE와 HAVING을 나눈 이유, GROUP BY가 SELECT에 거는 제약, 그룹 안 값이 우연히 같거나 행이 1개뿐이어도 여전히 에러가 나는 이유, HAVING의 별칭이 DB마다 다른 이유까지 전부 따로 외운 규칙 같았는데, 마지막에 SQL 실행 순서로 한 번에 정리되는 게 이번 섹션의 하이라이트였다. DISTINCT와 GROUP BY가 실은 아무 상호작용도 안 한다는 걸 직접 실험으로 확인한 것도 오래 남을 것 같다. 다음 섹션은 강의 로드맵 소개와 마무리 회고라고 하니, 기본편을 여기서 한번 정리하고 다음 로드맵(설계편·성능최적화편)으로 넘어갈 준비를 해야겠다.

---

# 참고 자료

<iframe width="100%" height="420" src="https://www.youtube.com/embed/6qkPy7RfLqQ" title="[SQL 기초 강의] 7강. SQL SELECT 절의 형식(ORDER BY 절과 GROUP BY 절)" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" allowfullscreen style="border-radius:12px;"></iframe>
