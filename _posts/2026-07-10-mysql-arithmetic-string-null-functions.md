---
title: "할인 없는 상품만 가격이 사라졌다 — SQL 데이터 가공 정리"
date: 2026-07-10 22:05:00 +0900
categories: [백엔드, 데이터베이스]
tags: [mysql, sql, concat, concat-ws, ifnull, coalesce, null, char-length, date-format, json-extract, arithmetic]
image:
  path: /assets/img/posts/mysql-arithmetic-string-null-functions/arithmetic-null-propagation-comparison.png
  alt: 산술 연산에서 NULL이 섞이면 결과 전체가 사라진다
---

인프런 김영한님 실전 데이터베이스 입문 섹션 6을 정리한다. 섹션 5까지는 있는 데이터를 그대로 꺼내는 법(SELECT, WHERE, ORDER BY)을 배웠는데, 이번 섹션은 꺼내면서 그 자리에서 값을 가공하는 법이었다. 산술 연산, 문자열 함수, NULL 함수, 그리고 마지막에 SQL 표준 함수와 MySQL 방언 함수를 구분하는 법까지 네 꼭지였다.

시작하기 전에 예제 스키마를 조금 확장했다. products 테이블은 그대로 두고, orders 테이블에 discount_amount 컬럼을, customers 테이블에 nickname, bio, settings 컬럼을 추가했다.

```sql
ALTER TABLE orders ADD discount_amount DECIMAL(10, 2);
ALTER TABLE customers ADD nickname VARCHAR(30);
ALTER TABLE customers ADD bio VARCHAR(200);
ALTER TABLE customers ADD settings JSON;
```

전부 NOT NULL을 안 걸고 그냥 추가해서, 기존 행들은 전부 NULL로 채워진 상태로 시작한다. 이게 이번 섹션 전체를 관통하는 복선이 될 줄은 시작할 땐 몰랐다.

# SELECT 절 산술 연산 — 사칙연산과 NULL 전파

## 상수와의 연산 — 부가세 계산

products 테이블에는 원가(price)만 있다. 부가세 10% 포함 가격을 화면에 보여줘야 하는데, 서버 코드에서 반복문 돌려가며 곱하는 대신 SELECT 절에서 바로 계산해봤다.

```sql
SELECT product_id, name, price, price * 1.1 AS price_with_tax
FROM products;
```
```
product_id | name  | price   | price_with_tax
1          | 키보드 | 45000   | 49500.0
2          | 마우스 | 15000   | 16500.0
```

계산이 되는 거야 당연한데, "왜 굳이 서버가 아니라 DB에서 계산하냐"는 이유가 더 흥미로웠다. 상품이 10개면 상관없지만 수십만 개면 얘기가 다르다고 한다. 서버가 반복문을 돌리려면 원본 데이터를 통째로 네트워크로 옮겨야 하는데, DB에서 계산해서 결과만 주면 옮기는 값 자체가 훨씬 가벼워진다. 쿠팡 같은 커머스도 상품 수백만 건의 할인가를 미리 계산해서 저장해두지 않고, 조회 시점에 `price * (1 - discount_rate)`처럼 즉석에서 계산한다고 들었다. 할인율이 바뀔 때마다 수백만 건을 업데이트하는 것보다 훨씬 합리적이라는 게 이해가 됐다.

강의에서 다루지 않은 것도 하나 확인해봤다. 나눗셈 연산자 `/`가 정수끼리 나눠도 소수점을 유지하는지 궁금했다.

```sql
SELECT 22000 / 7;
```
```
3142.8571
```

정수 두 개를 나눴는데도 소수점이 살아있었다. MySQL의 `/`는 기본적으로 실수 결과를 반환한다고 한다. 다만 이 값을 자바나 파이썬 같은 애플리케이션 코드에서 int 타입 변수로 받으면 그쪽에서 소수점이 잘려버릴 수 있으니, 문제가 DB가 아니라 애플리케이션 레이어에서 생길 수도 있다는 걸 알아뒀다.

## 컬럼끼리의 연산 — 재고 자산 가치

이번엔 두 컬럼을 곱해봤다. products 테이블의 price와 stock을 곱하면 "이 상품의 재고를 다 팔았을 때 얼마짜리인지"가 나온다.

```sql
SELECT product_id, name, price, stock, price * stock AS stock_value
FROM products;
```
```
product_id | name  | price   | stock | stock_value
1          | 키보드 | 45000   | 10    | 450000
2          | 마우스 | 15000   | 0     | 0
```

컬럼끼리의 연산은 반드시 같은 행 안에서만 일어난다는 걸 당연하게 여기고 넘어갈 뻔했는데, 곰곰이 생각해보니 이게 왜 중요한지 알겠다. stock_value 같은 값을 테이블에 미리 계산해서 저장해뒀다면, price나 stock이 바뀔 때마다 stock_value도 같이 업데이트해줘야 한다. 깜빡하면 값이 어긋난다. 조회할 때마다 즉석에서 계산하면 항상 최신 값 기준으로 정확한 결과가 나온다는 장점이 실감났다.

![price와 stock 두 컬럼이 만나 stock_value를 만드는 구조](/assets/img/posts/mysql-arithmetic-string-null-functions/stock-value-column-multiplication.png)

## NULL이 섞이면 생기는 일 — 할인가 계산의 함정

여기서부터가 이번 섹션의 핵심이었다. orders 테이블에 방금 추가한 discount_amount를 이용해서 최종 결제 금액을 계산해봤다.

```sql
SELECT order_id, total_price, discount_amount,
       total_price - discount_amount AS final_price
FROM orders;
```
```
order_id | total_price | discount_amount | final_price
1        | 90000       | 5000            | 85000
2        | 45000       | NULL            | NULL
3        | 120000      | NULL            | NULL
```

discount_amount가 NULL인 주문은 final_price까지 통째로 NULL이 됐다. discount_amount 컬럼을 방금 ALTER로 추가해서 기존 주문들은 전부 NULL인 상태였으니 당연한 결과이긴 한데, 화면에 실제로 이 쿼리 결과를 뿌려본다고 상상하면 등골이 서늘했다. 할인 안 받은 주문만 유독 결제 금액이 안 보이는 버그가 눈앞에서 재현된 거다.

![산술 연산에서 NULL이 섞이면 결과 전체가 사라진다](/assets/img/posts/mysql-arithmetic-string-null-functions/arithmetic-null-propagation-comparison.png)

원리를 곱씹어보니 이건 버그가 아니라 SQL이 의도한 동작이었다. "모르는 값 + 5"의 답은 5도 아니고 0도 아니고 "여전히 모른다"는 게 NULL의 정의라고 한다. discount_amount가 NULL이라는 건 "할인이 0원이다"가 아니라 "할인 여부를 아직 모른다"는 뜻이라서, 그 상태로 계산을 강행하면 결과도 "모른다"가 되는 게 논리적으로 맞다는 거다. 문제는 실무에서는 "할인 정보가 없으면 그냥 할인이 없는 거지"라고 취급하고 싶은 경우가 대부분이라는 점이었다. 이 간극을 메우는 방법은 NULL 함수 단원에서 제대로 배웠는데, 그 얘기는 뒤에서 다시 나온다.

# 문자열 함수 — CONCAT, CONCAT_WS, UPPER·LOWER, LENGTH

## CONCAT — 이름과 등급 붙이기

customers 테이블의 name과 grade를 합쳐서 "김민준(VIP)"처럼 보여주고 싶었다.

```sql
SELECT CONCAT(name, '(', grade, ')') AS display_name
FROM customers;
```
```
display_name
김민준(VIP)
이서연(일반)
```

CONCAT에 문자열 리터럴('(' 같은 것)을 인자로 섞어 넣을 수 있다는 게 편했다. 근데 grade가 NULL인 회원한테 실행해보니 결과가 완전히 사라졌다.

```sql
SELECT CONCAT(name, '(', grade, ')') AS display_name
FROM customers WHERE grade IS NULL;
```
```
display_name
NULL
```

산술 연산에서 봤던 NULL 전파가 문자열 함수에도 그대로 적용된다는 걸 직접 확인한 셈이다. CONCAT은 인자 중 하나라도 NULL이면 전체가 NULL이 된다.

## CONCAT_WS — NULL을 건너뛰는 이어붙이기

이번엔 city와 새로 추가한 address_detail을 공백으로 이어붙여 배송지를 만들어봤다. 다만 address_detail은 아직 아무도 안 넣어서 전부 NULL이다.

```sql
ALTER TABLE customers ADD address_detail VARCHAR(100);

SELECT CONCAT_WS(' ', city, address_detail) AS address
FROM customers;
```
```
address
서울
경기
```

city만 있고 address_detail이 NULL인데도 결과가 NULL로 사라지지 않고 city 값만 그대로 나왔다. CONCAT_WS는 With Separator의 줄임말인데, 구분자를 첫 인자로 받고 나머지 인자 중 NULL은 그냥 건너뛴다고 한다. CONCAT과 CONCAT_WS가 왜 따로 존재하는지 이 차이를 보고 나서야 납득이 됐다.

![CONCAT과 CONCAT_WS의 NULL 처리 차이](/assets/img/posts/mysql-arithmetic-string-null-functions/concat-vs-concat-ws-null-handling.png)

배송지처럼 일부 항목이 비어있을 수 있는 데이터는 CONCAT_WS로 안전하게 합치고, 성+이름처럼 둘 다 반드시 있어야 자연스러운 조합은 CONCAT을 쓰는 식으로 구분해서 써야겠다고 정리했다.

## UPPER / LOWER — 이메일 대소문자 통일

이메일 로그인을 구현한다고 가정하고, 대소문자가 섞여 저장된 이메일을 통일해봤다.

```sql
SELECT email, LOWER(email) AS normalized_email
FROM customers;
```
```
email               normalized_email
Minjun@Gmail.COM    minjun@gmail.com
park@naver.com      park@naver.com
```

로그인 검증 쿼리에 이렇게 적용해봤다.

```sql
SELECT * FROM customers
WHERE LOWER(email) = LOWER('Minjun@Gmail.com');
```

사용자가 로그인 폼에 대문자로 치든 소문자로 치든 같은 계정을 찾을 수 있다. 네이버나 다음 같은 서비스도 이메일 로그인 검증에 이 패턴을 쓴다고 들었는데, 왜 필요한지 직접 시나리오를 만들어보니 확실히 이해됐다. 다만 이 방식은 이메일 컬럼 전체에 함수를 씌워서 비교하는 거라, 인덱스를 그냥 걸어두면 못 탄다는 얘기도 찾아봤다. 저장할 때부터 소문자로 정규화해서 넣는 게 검색 성능 면에서는 더 낫다고 한다.

![LOWER로 대소문자가 섞인 이메일을 정규화해서 로그인 매칭하기](/assets/img/posts/mysql-arithmetic-string-null-functions/email-lowercase-login-normalization.png)

## LENGTH vs CHAR_LENGTH — 자기소개 글자 수 제한

bio 컬럼에 자기소개를 "100자까지"로 제한하는 로직을 짠다고 가정하고 LENGTH를 써봤다.

```sql
UPDATE customers SET bio = '안녕하세요 반갑습니다' WHERE customer_id = 1;

SELECT bio, LENGTH(bio) AS byte_len, CHAR_LENGTH(bio) AS char_len
FROM customers WHERE customer_id = 1;
```
```
bio               byte_len | char_len
안녕하세요 반갑습니다  30        | 11
```

'안녕하세요 반갑습니다'는 공백 포함 11글자인데 LENGTH는 30이 나왔다. UTF-8 인코딩에서 한글 한 글자가 3바이트를 차지하기 때문이라고 한다(공백 1글자 제외하고 10글자 × 3바이트 = 30). 만약 "100자 제한"을 LENGTH로 걸어뒀다면 한글 사용자는 33자만 써도 벌써 제한에 걸린다는 뜻이다.

![LENGTH는 바이트, CHAR_LENGTH는 글자 수를 센다](/assets/img/posts/mysql-arithmetic-string-null-functions/length-vs-char-length-korean-bytes.png)

```sql
SELECT bio FROM customers WHERE CHAR_LENGTH(bio) > 100;
```

글자 수 기준 제한이면 무조건 CHAR_LENGTH를 써야 한다는 걸 확실히 배웠다. 이게 SQL만의 문제가 아니라 자바스크립트 `.length`나 자바 `String.length()` 같은 애플리케이션 코드에서도 인코딩에 따라 똑같이 반복될 수 있는 함정이라, "글자 수를 셀 때는 바이트 기준인지 문자 기준인지"부터 확인하는 습관을 들여야겠다고 메모해뒀다.

# NULL 함수 — IFNULL과 COALESCE

## IFNULL — 대체 후보가 하나뿐일 때

앞서 산술 연산에서 봤던 discount_amount NULL 문제를 이제 제대로 해결해봤다.

```sql
SELECT order_id, total_price, discount_amount,
       total_price - IFNULL(discount_amount, 0) AS final_price
FROM orders;
```
```
order_id | total_price | discount_amount | final_price
1        | 90000       | 5000            | 85000
2        | 45000       | NULL            | 45000
3        | 120000      | NULL            | 120000
```

`IFNULL(discount_amount, 0)`이 discount_amount가 NULL이면 0을 대신 반환해줘서, final_price가 더 이상 사라지지 않고 원가 그대로 나왔다. 아까 등골이 서늘했던 그 문제가 함수 하나로 해결되는 걸 보니 왜 이 단원이 산술 연산 바로 다음에 나왔는지 이해가 됐다.

## COALESCE — 대체 후보가 여러 단계일 때

이번엔 표시 이름을 정하는 규칙을 만들어봤다. 닉네임이 있으면 닉네임, 없으면 실명, 그것도 없으면 "알 수 없음"을 보여주고 싶었다. 후보가 세 단계라 IFNULL로는 안 될 것 같아서 먼저 시도해봤다.

```sql
SELECT IFNULL(nickname, name, '알 수 없음') FROM customers;
```
```
ERROR 1582 (42000): Incorrect parameter count in the call to native function 'IFNULL'
```

역시 인자를 정확히 2개만 받는 함수라 3개를 넣으니 바로 에러가 났다. COALESCE로 바꿔서 다시 실행했다.

```sql
SELECT COALESCE(nickname, name, '알 수 없음') AS display_name
FROM customers;
```
```
display_name
태현
박지훈
알 수 없음
```

COALESCE는 인자를 왼쪽부터 순서대로 검사하다가 NULL이 아닌 첫 값을 반환한다고 한다. 인스타그램이나 트위터 같은 SNS도 닉네임을 따로 설정 안 한 유저는 아이디를 대신 보여준다는데, COALESCE(nickname, username) 같은 방식일 것 같다.

![IFNULL과 COALESCE의 인자 개수·용도 차이](/assets/img/posts/mysql-arithmetic-string-null-functions/ifnull-vs-coalesce-comparison.png)

두 함수의 차이를 표로 정리해봤다.

```
IFNULL(a, b)        → 인자 정확히 2개, MySQL 전용
COALESCE(a, b, c...) → 인자 가변, ANSI SQL 표준 (다른 DB에서도 동일 동작)
```

강의에서 다루진 않았지만 COALESCE가 표준이라는 게 궁금해서 찾아보니, PostgreSQL이나 Oracle에서도 COALESCE는 똑같이 동작한다고 한다. IFNULL은 MySQL 방언이라 다른 DB로 옮기면 그대로 안 쓰인다. 그래서 대체 후보가 딱 2개뿐이라도, 나중에 DB를 옮길 가능성을 생각해서 처음부터 COALESCE로 통일해두는 팀도 많다고 들었다. 지금 당장은 IFNULL이 더 짧고 읽기 편해서 둘 다 상황 봐가며 써야겠다고 정리했다.

# 다양한 함수 소개 — SQL 표준과 MySQL 방언, 날짜·JSON 함수

섹션 6 마지막 꼭지는 깊게 파기보다 "이런 함수들도 있다"는 지도를 그리는 시간이었다.

## 표준 함수와 방언 함수

지금까지 배운 함수 중에서도 COALESCE, UPPER, LOWER는 ANSI SQL 표준에 정의돼 있어서 다른 DB에서도 그대로 동작하는데, IFNULL이나 CONCAT_WS는 MySQL 전용 방언이라고 한다. 실제로 다른 DB 문법을 검색해서 그대로 가져오면 어떻게 되는지 궁금해서 일부러 PostgreSQL 스타일 문법을 MySQL에 넣어봤다.

```sql
SELECT name || '(' || grade || ')' FROM customers;
```
```
ERROR 1064 (42000): You have an error in your SQL syntax
```

PostgreSQL이나 Oracle은 `||`로 문자열을 이어붙이는데, MySQL은 이 문법 자체를 지원하지 않아서 바로 문법 오류가 났다. 인터넷에서 SQL 예제를 복사해서 썼는데 안 먹히는 경우, 다른 DB 기준 예제였을 가능성부터 의심해봐야겠다고 배웠다.

![표준 SQL 함수와 MySQL 방언 함수의 차이](/assets/img/posts/mysql-arithmetic-string-null-functions/mysql-dialect-vs-standard-function-comparison.png)

## 날짜 함수 — 월별 통계용 그룹핑 키 만들기

날짜 함수도 맛만 봤다. 가입일을 월 단위로 뭉치는 DATE_FORMAT을 써봤다.

```sql
SELECT DATE_FORMAT(joined_at, '%Y-%m') AS signup_month, COUNT(*) AS cnt
FROM customers
GROUP BY signup_month;
```
```
signup_month | cnt
2026-06      | 12
2026-07      | 8
```

GROUP BY는 아직 정식으로 안 배웠는데, DATE_FORMAT으로 날짜를 월 단위로 뭉쳐서 그룹핑 키를 만드는 패턴을 미리 살짝 봤다. 쿠팡 같은 커머스의 매출 대시보드도 이런 식으로 `DATE_FORMAT(order_date, '%Y-%m')`을 그룹핑 키로 써서 월별 매출을 뽑는다고 들었는데, 다음 섹션인 집계와 그룹핑에서 제대로 배우면 이 쿼리도 완전히 이해될 것 같다.

## JSON 함수 — 유연한 설정값 저장하고 꺼내기

마지막으로 JSON 컬럼을 다뤄봤다. customers에 추가한 settings 컬럼에 값을 넣고 특정 값만 꺼내봤다.

```sql
UPDATE customers SET settings = '{"theme": "dark", "notify": true}'
WHERE customer_id = 1;

SELECT JSON_EXTRACT(settings, '$.theme') AS theme
FROM customers WHERE customer_id = 1;
```
```
theme
"dark"
```

`$.theme`의 `$`가 JSON 최상위를 가리키는 기호라고 한다. 컬럼을 미리 다 정의하지 않고 유저마다 다른 설정 항목을 담을 수 있다는 게 신기했다. 카카오나 네이버 같은 서비스도 유저별 커스텀 설정을 컬럼을 무한정 늘리는 대신 JSON 컬럼 하나로 저장하는 경우가 많다고 하는데, 설정 항목이 자주 추가·변경돼도 테이블 구조를 매번 안 바꿔도 된다는 게 장점인 것 같다. 다만 JSON 컬럼 안의 값으로 자주 검색·필터링을 하면 일반 컬럼보다 느릴 수 있다고 해서, 자주 조회하는 값은 처음부터 일반 컬럼으로 빼두는 게 낫다는 점도 같이 메모해뒀다.

![settings JSON 컬럼 내부에서 JSON_EXTRACT로 값 하나만 꺼내는 구조](/assets/img/posts/mysql-arithmetic-string-null-functions/json-settings-column-structure.png)

# 마무리

섹션 6은 "값을 있는 그대로 꺼내는 것"에서 "값을 가공해서 꺼내는 것"으로 넘어가는 단계였다. 산술 연산에서 discount_amount가 NULL이라 final_price가 통째로 사라지는 걸 직접 보고, 그걸 IFNULL 하나로 해결하는 과정이 이번 섹션에서 제일 인상 깊었다. NULL이 "값이 없다"가 아니라 "값을 알 수 없다"는 개념이라는 걸 섹션 5에서 배웠는데, 이번 섹션에서는 그 개념이 연산과 함수에서 어떻게 전파되고 어떻게 방어할 수 있는지까지 이어져서 훨씬 실감이 났다.

CONCAT과 CONCAT_WS의 NULL 처리 차이, LENGTH와 CHAR_LENGTH의 바이트/글자 수 차이처럼 "비슷해 보이는 함수 두 개가 왜 따로 있는지"를 매번 직접 에러를 내보면서 확인한 게 기억에 오래 남을 것 같다. 다음 섹션은 집계 함수와 GROUP BY라고 하니, 오늘 살짝 맛본 DATE_FORMAT 그룹핑이 거기서 제대로 등장할 것 같다.

---

# 참고 자료

<iframe width="100%" height="420" src="https://www.youtube.com/embed/RbK3zy61OH4" title="MySQL Tutorial - 12강 문자열 함수(character string function)" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" allowfullscreen style="border-radius:12px;"></iframe>
