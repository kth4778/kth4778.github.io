---
title: "WHERE phone = NULL이 항상 0건이었던 이유 — SQL 조회와 정렬 정리"
date: 2026-07-09 21:40:00 +0900
categories: [백엔드, 데이터베이스]
tags: [mysql, sql, select, where, order-by, limit, distinct, null, pagination, like, between, in]
image:
  path: /assets/img/posts/mysql-select-where-orderby-limit-distinct-null/select-star-vs-columns-comparison.png
  alt: SELECT * 와 특정 컬럼 조회 비교
---

인프런 김영한님 실전 데이터베이스 입문 섹션 5를 정리한다. 섹션 4까지는 테이블 설계하고 데이터 넣고 고치고 지우는 것까지 했는데, 이번엔 그렇게 채워 넣은 데이터를 실제로 꺼내보는 법을 배웠다. SELECT 하나만 배우고 끝날 줄 알았는데 WHERE, ORDER BY, LIMIT, DISTINCT까지 이어지더니 마지막엔 NULL 정렬 트릭까지 나와서, 소단원 일곱 개짜리 섹션이 생각보다 훨씬 길어졌다.

참고로 이번 섹션부터 강의에서 쓰는 예제 데이터셋이 조금 커졌다. orders 테이블에 total_price, status 컬럼이, customers 테이블에 city, grade 컬럼이 추가돼 있었다. 아래 예제는 이 확장된 스키마 기준이다.

# SELECT 조회 — 전체/특정 컬럼, AS 별칭

## SELECT * 와 특정 컬럼 조회

가장 먼저 배운 건 당연히 SELECT였다. 전체 컬럼을 다 보고 싶을 땐 `*`, 필요한 것만 볼 땐 컬럼명을 나열한다.

```sql
SELECT * FROM customers;
SELECT name, email FROM customers;
```

![SELECT * 와 특정 컬럼 조회 비교](/assets/img/posts/mysql-select-where-orderby-limit-distinct-null/select-star-vs-columns-comparison.png)

문법 자체는 단순한데, "왜 실무에서는 SELECT *를 거의 안 쓰냐"는 이유가 더 흥미로웠다. 컬럼이 추가되는 상황을 직접 재현해봤다.

```sql
-- 1단계: SELECT *로 짠 코드가 있다고 가정
SELECT * FROM customers;
```
```
customer_id | name  | email              | joined_at
1           | 김민준 | minjun@test.com    | 2026-07-04 10:00:00
```

```sql
-- 2단계: 나중에 phone 컬럼이 추가됨 (섹션 4에서 배운 ALTER TABLE)
ALTER TABLE customers ADD phone VARCHAR(20) DEFAULT '미등록';

-- 3단계: 똑같은 SELECT *를 다시 실행
SELECT * FROM customers;
```
```
customer_id | name  | email              | phone   | joined_at
1           | 김민준 | minjun@test.com    | 미등록   | 2026-07-04 10:00:00
```

애플리케이션 코드에서 이 결과를 배열 인덱스로 다루고 있었다면(`row[3]`이 joined_at이라고 가정했던 코드), phone 컬럼이 끼어드는 순간 `row[3]`이 phone으로 바뀌어버린다. 컬럼을 명시해서 `SELECT name, email, joined_at`으로 짰다면 이 문제 자체가 안 생긴다는 걸 직접 재현해보고 나니 왜 그렇게 강조하는지 체감이 됐다.

내부적으로 SELECT *가 어떻게 처리되는지도 궁금해서 찾아봤다. MySQL 옵티마이저는 쿼리를 실행하기 전에 `*`를 실제 컬럼 목록으로 펼치는 파싱 단계를 거친다고 한다. 즉 `SELECT *`도 결국 내부적으로는 `SELECT customer_id, name, email, phone, joined_at`으로 바뀌어서 실행되는데, 이 펼치는 과정 자체가 information_schema를 조회하는 추가 작업이라 컬럼을 직접 나열하는 것보다 미세하게 느리다고 한다. 네트워크 전송량 차이만 얘기했었는데, 파싱 단계에서도 손해를 본다는 게 새로웠다.

## AS 별칭

컬럼 이름을 조회 결과에서만 다르게 보여주는 AS도 배웠다.

```sql
SELECT name AS 고객명, email AS 이메일 FROM customers;
```
```
고객명  | 이메일
김민준  | minjun@test.com
```

강의에서는 다루지 않았지만, 테이블 이름에도 별칭을 붙일 수 있다는 걸 곁가지로 찾아봤다.

```sql
SELECT c.name, c.email FROM customers AS c;
```

지금은 테이블이 하나뿐이라 별 의미가 없어 보이는데, 나중에 여러 테이블을 JOIN하면 `customers.name`, `orders.name`처럼 매번 긴 테이블명을 다 쓰는 대신 `c.name`, `o.name`으로 줄여 쓸 수 있어서 JOIN 파트에서 진짜 유용해질 것 같다. AS는 SELECT 절 컬럼뿐 아니라 FROM 절 테이블에도 붙는다는 걸 알아두니 나중에 배울 JOIN이 덜 낯설 것 같다.

# WHERE 기본 검색 — 비교연산자, AND/OR/NOT

## 비교연산자

조건에 맞는 행만 걸러내는 WHERE를 배웠다. `=`, `!=`, `>`, `<`, `>=`, `<=` 여섯 개다.

```sql
SELECT * FROM orders WHERE total_price >= 50000;
```

문자열 비교할 때 따옴표를 빼먹으면 어떻게 되는지 일부러 실험해봤다.

```sql
SELECT * FROM orders WHERE status = 배송완료;
```
```
Error Code: 1054. Unknown column '배송완료' in 'where clause'
```

따옴표 없이 쓰니까 MySQL이 `배송완료`를 문자열이 아니라 컬럼 이름으로 해석해서 "그런 컬럼 없다"는 에러를 던졌다. `WHERE status = '배송완료'`로 고치니 정상 동작했다. 숫자는 따옴표가 없어도 되는데 문자열은 반드시 있어야 한다는 규칙을, 에러를 직접 봐야 확실히 외워지는 것 같다.

## AND / OR — 우선순위와 괄호

조건 두 개를 묶는 AND, OR도 배웠다.

![AND OR 우선순위와 괄호](/assets/img/posts/mysql-select-where-orderby-limit-distinct-null/and-or-precedence-parentheses.png)

```sql
-- 괄호 없이 짠 쿼리
SELECT * FROM orders
WHERE status = '배송완료' AND total_price >= 30000 OR is_vip = 1;
```

이걸 "배송완료이면서 (3만원 이상 또는 VIP)"라고 생각하고 짰는데, 실제로는 AND가 OR보다 우선순위가 높아서 `status = '배송완료' AND total_price >= 30000` 부분이 먼저 묶이고, 그 결과에 `OR is_vip = 1`이 덧붙는 식으로 해석된다. 그러니까 배송 상태나 금액과 상관없이 VIP이기만 하면 무조건 포함되는, 의도와 전혀 다른 결과가 나온다. 실행해보면 에러는 안 나고 그냥 조용히 더 많은 행이 나오기 때문에, 결과 건수를 눈으로 세보지 않으면 발견하기 어려운 실수였다.

```sql
-- 의도대로 고친 쿼리
SELECT * FROM orders
WHERE status = '배송완료' AND (total_price >= 30000 OR is_vip = 1);
```

강의에서 안 다뤘지만 궁금해서 찾아본 게 하나 있다. `XOR`이라는 논리연산자도 MySQL에 있다고 한다. "둘 중 정확히 하나만 참일 때"만 참이 되는데, OR는 "하나 이상"이라 둘 다 참이어도 참이 되는 것과 다르다. 실무에서 자주 쓰이진 않는다고 하지만, "이거 아니면 저거, 근데 둘 다는 안 됨" 같은 조건이 필요할 때 쓸 수 있을 것 같다.

## NOT — 조건 뒤집기

```sql
SELECT * FROM orders WHERE NOT (status = '취소' OR status = '반품');
```

`status != '취소' AND status != '반품'`이라고 풀어 쓸 수도 있는데, 상태 종류가 늘어날수록 `NOT (... OR ... OR ...)` 형태가 훨씬 읽기 편하다는 걸 실제로 조건을 3개, 4개로 늘려보면서 체감했다. 이런 사소한 가독성 차이가 나중에 조건이 복잡해진 코드리뷰에서 크게 갈릴 것 같다.

# WHERE 편리한 조건 검색 — BETWEEN, IN, LIKE, IS NULL

## BETWEEN — 범위 검색

```sql
SELECT * FROM orders WHERE total_price BETWEEN 100000 AND 200000;
```

순서를 반대로 넣으면 어떻게 되는지 실험해봤다.

```sql
SELECT * FROM orders WHERE total_price BETWEEN 200000 AND 100000;
```
```
Empty set (0.00 sec)
```

에러는 안 나고 그냥 결과가 0건이었다. "20만원 이상이면서 동시에 10만원 이하"인 값은 논리적으로 존재할 수 없으니 당연한 결과인데, 문법 오류가 아니라서 값이 없는 건지 조건을 잘못 쓴 건지 헷갈리기 딱 좋다는 생각이 들었다. 날짜에도 그대로 적용된다는 것도 확인했다.

```sql
SELECT * FROM orders WHERE order_date BETWEEN '2026-01-01' AND '2026-01-31';
```

## IN — 목록 검색

```sql
SELECT * FROM customers WHERE city IN ('서울', '경기', '인천');
```

`IN` 서브쿼리라는 것도 곁가지로 찾아봤다. 목록을 직접 나열하는 대신, 다른 SELECT 결과 자체를 목록으로 쓸 수 있다고 한다.

```sql
-- VIP 등급 고객의 city 목록에 속하는 주문만
SELECT * FROM orders WHERE customer_id IN (
  SELECT customer_id FROM customers WHERE grade = 'VIP'
);
```

지금 배운 건 값을 직접 나열하는 정적인 IN이었는데, 서브쿼리를 쓰면 "실시간으로 계산된 목록"에 대해 IN을 걸 수 있다는 게 신기했다. 이건 서브쿼리 파트에서 정식으로 배울 것 같은데, 미리 맛만 봐도 활용도가 꽤 커 보인다.

## LIKE — 와일드카드 패턴 검색

```sql
SELECT * FROM customers WHERE email LIKE '%@gmail.com';
SELECT * FROM customers WHERE name LIKE '김%';
SELECT * FROM customers WHERE name LIKE '김__';
```

`%`의 위치에 따라 뜻이 완전히 달라진다는 걸 표로 정리해봤다.

```
'김%'    → 김으로 시작 (뒤는 자유)
'%김'    → 김으로 끝 (앞은 자유)
'%김%'   → 김이 포함 (앞뒤 다 자유)
'김__'   → 정확히 3글자, 김으로 시작
```

앞이 뚫린 패턴(`%@gmail.com`)이 인덱스를 못 탄다는 얘기가 나와서 왜 그런지 더 찾아봤다. 인덱스는 책의 색인처럼 값을 정렬된 순서로 미리 정리해둔 구조인데, 앞부분이 정해지지 않은 검색은 "어디서부터 찾아야 할지"를 색인만 봐서는 알 수 없다고 한다. 마치 국어사전에서 "끝이 '다'로 끝나는 단어를 찾아줘"라고 하면 첫 글자 기준으로 정렬된 색인이 아무 도움이 안 되는 것과 같은 원리다. 반대로 `'김%'`처럼 앞이 고정된 패턴은 색인에서 '김'으로 시작하는 구간만 바로 찾아갈 수 있어서 훨씬 빠르다고 한다. 이 부분은 나중에 인덱스 단원에서 B-트리 구조랑 같이 다시 나올 것 같다.

## IS NULL — NULL 확인

```sql
SELECT * FROM customers WHERE phone IS NULL;
```

강의에서 강조한 것처럼 `= NULL`로 써보고 정말 안 되는지 직접 확인해봤다.

```sql
SELECT * FROM customers WHERE phone = NULL;
```
```
Empty set (0.00 sec)
```

phone이 NULL인 행이 분명히 있는데도(전화번호 미등록 회원) 결과가 0건으로 나왔다. 에러가 안 나서 더 위험했다. "회원이 다 전화번호를 등록했나?"라고 착각할 뻔했는데, `IS NULL`로 바꾸니 제대로 잡혔다.

```sql
SELECT * FROM customers WHERE phone IS NULL;
```
```
customer_id | name  | phone
3           | 이서연 | NULL
```

# ORDER BY 정렬 — ASC/DESC, 다중 열 정렬

## ASC / DESC

```sql
SELECT * FROM orders ORDER BY total_price;        -- 기본값 ASC
SELECT * FROM orders ORDER BY total_price DESC;
```

`DESC`를 빼먹으면 무조건 오름차순이 된다는 걸 직접 겪어봤다. "높은 금액 순으로 보여줘야지" 하고 `ORDER BY total_price`만 썼다가 제일 싼 주문이 맨 위로 온 걸 보고 나서야 `DESC`가 기본값이 아니라는 걸 다시 확인했다.

## 다중 열 정렬

![다중 열 정렬 우선순위 흐름](/assets/img/posts/mysql-select-where-orderby-limit-distinct-null/order-by-multi-column-priority-flow.png)

```sql
SELECT * FROM orders ORDER BY total_price DESC, order_date ASC;
```

1순위가 완전히 같은 행들 안에서만 2순위가 적용된다는 걸 직접 데이터로 확인해보고 싶어서, 같은 금액의 주문 두 건을 만들어서 실행해봤다.

```
total_price | order_date
200000      | 2026-07-01
200000      | 2026-07-03
150000      | 2026-07-05
```

1순위(total_price DESC)로 200000짜리 두 건이 먼저 묶이고, 그 안에서 2순위(order_date ASC)로 07-01이 07-03보다 앞에 온 걸 확인했다. 150000짜리는 금액에서 이미 순서가 갈렸기 때문에 order_date는 아예 비교 대상도 안 됐다. "동점자 처리용 기준"이라는 설명이 말로 들을 때보다 직접 데이터를 만들어서 보니 훨씬 명확해졌다.

컬럼별로 정렬 방향을 다르게 섞어 쓸 수 있다는 것도 확인했다.

```sql
SELECT * FROM orders ORDER BY city ASC, total_price DESC;
```

# LIMIT 개수 제한 — TOP-N, 페이징

## TOP-N

```sql
SELECT * FROM orders ORDER BY total_price DESC LIMIT 5;
```

`ORDER BY` 없이 `LIMIT`만 썼을 때 결과가 어떻게 나오는지도 실험해봤다.

```sql
SELECT * FROM orders LIMIT 5;
```

실행할 때마다 정확히 같은 5건이 나오긴 했는데, 이게 "보장된 순서"가 아니라 "지금 이 시점 저장 구조상 우연히 그런 것"이라는 설명을 듣고 나니 찜찜해졌다. 나중에 데이터가 삭제되거나 재정렬(예: OPTIMIZE TABLE)되면 같은 쿼리라도 다른 5건이 나올 수 있다고 한다. "상위 N개"라는 말 자체가 정렬을 전제로 한다는 걸 다시 새겼다.

## 페이징 — OFFSET

![페이징 OFFSET 계산 흐름](/assets/img/posts/mysql-select-where-orderby-limit-distinct-null/limit-offset-pagination-flow.png)

```sql
SELECT * FROM orders ORDER BY order_id LIMIT 10 OFFSET 20;  -- 3페이지 (페이지당 10개)
```

OFFSET 공식(`(페이지번호-1) x 페이지당개수`)을 직접 표로 만들어보면서 계산이 맞는지 확인했다.

```
1페이지: OFFSET (1-1)*10 = 0
2페이지: OFFSET (2-1)*10 = 10
3페이지: OFFSET (3-1)*10 = 20
```

큰 OFFSET이 느려지는 이유가 궁금해서 더 찾아봤다. `LIMIT 10 OFFSET 100000`을 실행하면 DB는 "100000번째부터 보여줘"를 바로 처리하는 게 아니라, 앞의 100000개를 순서대로 다 세면서 건너뛴 다음에야 원하는 10개를 찾는다고 한다. 페이지 번호가 커질수록 매번 앞부분을 다시 세야 하니까 느려지는 게 당연했다. 이걸 피하려고 "커서 기반 페이징"이라는 방식도 있다는데, 마지막으로 본 항목의 `order_id`를 기억해뒀다가 `WHERE order_id > 마지막ID LIMIT 10`처럼 조회하는 방식이라고 한다. 인스타그램이나 트위터 같은 무한 스크롤 피드가 이 방식을 많이 쓴다고 하는데, 실무에서 마주칠 것 같아서 메모해뒀다.

# DISTINCT 중복 제거 — 단일/다중 컬럼

## 단일 컬럼 DISTINCT

```sql
SELECT DISTINCT city FROM customers;
```

`DISTINCT` 없이 그냥 조회했을 때랑 결과를 나란히 비교해봤다.

```sql
SELECT city FROM customers;
```
```
서울
서울
경기
서울
인천
경기
```

```sql
SELECT DISTINCT city FROM customers;
```
```
서울
경기
인천
```

6줄이 3줄로 줄어드는 걸 직접 보니 "중복 제거"라는 말이 뭘 뜻하는지 확실해졌다.

## 다중 컬럼 DISTINCT

```sql
SELECT DISTINCT city, grade FROM customers;
```

city만 같고 grade가 다른 행이 실제로 별개로 남는지 확인해봤다.

```
city   grade
서울    VIP
서울    일반
경기    VIP
```

서울+VIP와 서울+일반은 city는 같지만 조합 전체가 다르니까 둘 다 살아남았다. DISTINCT가 "SELECT한 컬럼 전체 조합"을 기준으로 판단한다는 게 이걸 보고 확실히 이해됐다. COUNT랑 같이 쓰면 "서로 다른 값이 몇 개인지" 셀 수 있다는 것도 곁가지로 찾아봤다.

```sql
SELECT COUNT(DISTINCT city) FROM customers;
```

이건 "우리 고객이 몇 개 지역에 분포하나" 같은 통계를 낼 때 바로 쓸 수 있을 것 같다. 다음 섹션인 집계 함수에서 제대로 나올 것 같긴 한데, COUNT랑 DISTINCT를 합쳐 쓸 수 있다는 걸 미리 알아두니 나중에 낯설지 않을 것 같다.

# NULL 알 수 없는 값 — 정렬 순서와 위치 제어

## NULL 정렬 순서

phone이 NULL인 회원을 포함해서 전체를 정렬해봤다.

```sql
SELECT name, phone FROM customers ORDER BY phone ASC;
```
```
name    phone
이서연   NULL
박지훈   NULL
김민준   010-1111-2222
최유진   010-3333-4444
```

MySQL은 오름차순일 때 NULL을 가장 작은 값 취급해서 맨 앞에 놓는다는 걸 직접 확인했다. 다른 DBMS(PostgreSQL 등)는 기본값이 다를 수 있다고 해서 찾아봤는데, PostgreSQL은 정렬 방향과 상관없이 NULL을 기본적으로 맨 뒤에 놓는다고 한다. 같은 쿼리를 MySQL에서 PostgreSQL로 마이그레이션하면 NULL 위치가 뒤바뀔 수 있다는 뜻인데, 이런 "DB마다 기본 동작이 다른 부분"을 모르고 넘어갔으면 나중에 마이그레이션할 때 크게 당황했을 것 같다.

## 정렬 위치 제어 트릭

![NULL 정렬 위치 제어 트릭](/assets/img/posts/mysql-select-where-orderby-limit-distinct-null/null-sort-position-control-trick.png)

전화번호 없는 회원을 맨 뒤로 보내고 싶어서 배운 트릭을 그대로 써봤다.

```sql
SELECT name, phone FROM customers
ORDER BY phone IS NULL, phone ASC;
```
```
name    phone
김민준   010-1111-2222
최유진   010-3333-4444
이서연   NULL
박지훈   NULL
```

`phone IS NULL`이 0(거짓)/1(참)로 평가된다는 걸 실제로 확인해보고 싶어서 그 부분만 따로 SELECT 해봤다.

```sql
SELECT name, phone, phone IS NULL AS is_null_flag FROM customers;
```
```
name    phone           is_null_flag
김민준   010-1111-2222   0
이서연   NULL             1
```

`is_null_flag`가 0과 1로 실제 값처럼 찍히는 걸 보고 나서야 "아, 이게 진짜 숫자라서 정렬 기준으로 쓸 수 있는 거구나"가 이해됐다. 1순위로 이 0/1 값을 정렬하면 0(NULL 아님)이 먼저, 1(NULL임)이 나중에 오는 거고, 그 안에서 2순위로 phone ASC가 세부 정렬을 담당하는 구조였다. 5-4단계에서 배운 다중 열 정렬 원리를 이렇게 응용할 수 있다는 게 이 섹션에서 제일 인상 깊었던 부분이다.

# 마무리

섹션 5는 SELECT 하나로 시작해서 WHERE, ORDER BY, LIMIT, DISTINCT, NULL까지 이어지는 긴 여정이었다. 특히 `WHERE phone = NULL`이 에러도 없이 조용히 0건을 뱉어내는 걸 직접 보고 나서, NULL이 "값이 없다"가 아니라 "값을 알 수 없다"는 완전히 다른 개념이라는 게 확실히 각인됐다.

AND/OR 우선순위 때문에 괄호 없이 짠 쿼리가 의도와 다르게 동작하는 것도 직접 재현해보길 잘한 것 같다. 에러가 안 나는 실수라서 코드리뷰에서도 놓치기 쉬울 것 같은데, 이제는 조건이 3개 이상 섞이면 무조건 괄호부터 확인하는 습관을 들여야겠다. 다음 섹션은 산술 연산과 문자열·NULL 함수라고 하니, 이번에 배운 조회 문법에 살을 붙이는 느낌으로 이어질 것 같다.

---

# 참고 자료

<iframe width="100%" height="420" src="https://www.youtube.com/embed/_JURyg_KzHE" title="[SQL 기초 강의] 6강. SQL 기본 문법(SELECT ~ FROM ~ WHERE)" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" allowfullscreen style="border-radius:12px;"></iframe>
