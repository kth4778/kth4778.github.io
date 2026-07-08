---
title: "쇼핑몰 테이블을 직접 설계해보니 — FK 에러 코드가 더 이상 안 무섭다"
date: 2026-07-06 22:30:00 +0900
categories: [백엔드, 데이터베이스]
tags: [mysql, sql, ddl, dml, erd, foreign-key, decimal, auto-increment, transaction, rollback, constraint]
image:
  path: /assets/img/posts/mysql-ddl-dml-schema-design-constraints/customers-products-orders-erd-relation.png
  alt: customers, products, orders 테이블 관계
---

인프런 김영한님 실전 데이터베이스 입문 섹션 4를 정리한다. 섹션 3까지는 members 테이블 하나로만 연습했는데, 이번 섹션에서는 진짜 쇼핑몰이 쓸 법한 테이블 여러 개를 직접 설계하고, 데이터를 넣고 고치고 지우는 실전 흐름을 배웠다. 일곱 꼭지나 되는 긴 섹션이라 정리하는 데도 시간이 꽤 걸렸다.

# DDL 테이블 생성1 — customers, products, orders 실전 설계

## customers 테이블 설계

가장 먼저 만든 건 고객 정보를 담는 customers 테이블이었다.

```sql
CREATE TABLE customers (
  customer_id INT PRIMARY KEY AUTO_INCREMENT,
  name VARCHAR(50) NOT NULL,
  email VARCHAR(100) NOT NULL UNIQUE,
  joined_at DATETIME DEFAULT CURRENT_TIMESTAMP
);
```

`AUTO_INCREMENT`를 처음 실전에 써봤다. PK 값을 직접 관리하지 않고 DBMS가 1, 2, 3 순서로 자동 채워주는 거다. 이전까지는 `INSERT`할 때마다 PK 값을 손으로 넣었는데, 이렇게 하면 동시에 여러 명이 가입해도 번호가 안 겹칠 걱정을 안 해도 된다는 걸 알게 됐다.

여기서 재밌는 사실도 하나 배웠다. `AUTO_INCREMENT`로 발급된 번호는 트랜잭션이 `ROLLBACK`돼도 다시 회수되지 않는다고 한다. 예를 들어 `customer_id`가 5까지 발급된 상태에서 6번을 발급받은 삽입이 실패해서 롤백되면, 다음 삽입은 6이 아니라 7부터 시작한다는 거다. 회원 번호에 중간중간 구멍(빈 번호)이 생길 수 있다는 뜻인데, 이건 버그가 아니라 동시성 문제를 피하기 위한 설계라고 한다. 여러 트랜잭션이 동시에 번호를 받아가는 상황에서 굳이 "구멍 없이 채우기"를 보장하려다 보면 성능이 떨어지기 때문이라고 했다.

`email NOT NULL UNIQUE` 조합도 섹션 3에서 배운 그대로 실전에 쓰였다. 회원가입 API가 실제로 동작하는 순서를 상상해보면, 사용자가 폼을 제출하면 서버가 `INSERT INTO customers (name, email) VALUES (...)`를 실행하고, 이메일이 중복이면 DB가 `1062` 에러를 던지고, 서버가 그걸 잡아서 "이미 가입된 이메일입니다"로 바꿔서 응답하는 흐름이다. 지금까지는 이 흐름을 그냥 "프레임워크가 알아서 해주는 것"으로만 생각했는데, 그 뒤에 있는 SQL 레벨의 동작을 이해하고 나니 훨씬 명확해졌다.

## products 테이블 설계

상품 테이블을 만들면서 가격 타입을 제대로 고민해봤다.

```sql
CREATE TABLE products (
  product_id INT PRIMARY KEY AUTO_INCREMENT,
  name VARCHAR(100) NOT NULL,
  price DECIMAL(10, 2) NOT NULL,
  stock INT NOT NULL DEFAULT 0 CHECK (stock >= 0)
);
```

`DECIMAL(10, 2)`가 뭘 의미하는지 궁금해서 물어봤는데, 10은 전체 자릿수(precision), 2는 소수점 아래 자릿수(scale)라고 한다. 그러니까 정수부 8자리 + 소수부 2자리 조합이고, 최대 99,999,999.99까지 저장 가능하다는 거였다. 실제로 범위를 넘겨서 넣어봤다.

```sql
INSERT INTO products (name, price, stock) VALUES ('노트북', 123456789.99, 10);
```

```
Error Code: 1264. Out of range value for column 'price' at row 1
```

정수부가 9자리라서 8자리 한도를 넘어서 거부된 거였다. `FLOAT`을 안 쓰는 이유도 구체적인 예시로 이해했다. `1.005 * 100`을 계산하면 `100.5`가 아니라 `100.49999999999999`가 나온다고 한다. 이게 반올림 로직에서 100.5를 100으로 잘못 처리하게 만드는, 실제로 유명한 "가격 반올림 버그"의 원인이라고 했다. 이유가 궁금해서 더 파봤더니, 컴퓨터는 숫자를 2진수로 저장하는데 `0.1`, `1.005` 같은 10진 소수 중 상당수는 2진수로 딱 떨어지게 표현이 안 된다고 한다. 그래서 컴퓨터가 어느 지점에서 반드시 잘라 저장해야 하고, 그 잘린 부분이 미세한 오차로 남는다는 거였다. `DECIMAL`은 이 숫자를 2진수 근사치로 바꾸지 않고 "1005"라는 정수 + "소수점은 뒤에서 3번째 자리"라는 위치 정보를 따로 저장해서, 정수는 2진수로도 정확히 표현되니까 오차가 생길 여지 자체가 없다고 한다.

`stock INT NOT NULL DEFAULT 0 CHECK (stock >= 0)`도 실전다웠다. 재고가 없으면 0으로 자동 채워지고, 음수 재고가 되는 걸 `CHECK`로 원천 차단했다. 다만 이 `CHECK`만으로는 완벽하지 않다는 것도 배웠다. 예를 들어 재고가 1개 남았는데 두 명이 거의 동시에 주문을 넣으면, 둘 다 "재고 1개 있음"을 확인하고 동시에 `UPDATE stock = stock - 1`을 실행할 수 있다. 이러면 재고가 -1이 되는 게 아니라(그건 CHECK가 막아주지만), 재고가 실제로는 0개인데 주문 2건이 확정되는 초과판매 문제가 생길 수 있다고 한다. 이건 트랜잭션 격리 수준이랑 관련된 문제라서, 나중에 트랜잭션 파트에서 더 깊게 다룬다고 했다.

## orders 테이블 설계 — FK로 연결

customers와 products를 이어주는 orders 테이블도 만들었다.

![customers, products, orders 테이블 관계](/assets/img/posts/mysql-ddl-dml-schema-design-constraints/customers-products-orders-erd-relation.png)

```sql
CREATE TABLE orders (
  order_id INT PRIMARY KEY AUTO_INCREMENT,
  customer_id INT NOT NULL,
  product_id INT NOT NULL,
  quantity INT NOT NULL CHECK (quantity > 0),
  ordered_at DATETIME DEFAULT CURRENT_TIMESTAMP,
  FOREIGN KEY (customer_id) REFERENCES customers(customer_id),
  FOREIGN KEY (product_id) REFERENCES products(product_id)
);
```

FK가 두 개 걸려서, 존재하지 않는 고객이나 상품 번호로 주문을 넣으려고 하면 자동으로 막힌다. 실제로 없는 고객 번호로 넣어봤다.

```sql
INSERT INTO orders (customer_id, product_id, quantity) VALUES (999, 1, 2);
```

```
Error Code: 1452. Cannot add or update a child row: a foreign key constraint fails
```

애플리케이션 코드에서 "존재하는 고객인지" 매번 확인하는 로직을 안 짜도 DB가 대신 지켜준다는 게 실전에서 정말 든든하게 느껴졌다. 쿠팡·11번가 같은 이커머스도 결국 이런 식으로 고객·상품·주문을 분리해서, 상품 가격이 나중에 바뀌어도 과거 주문 기록은 그대로 유지한다고 한다.

그리고 실제로 주문 하나를 넣는다는 게 사실 "주문 행 추가"와 "재고 차감" 두 가지 작업을 동시에 해야 한다는 것도 깨달았다.

```sql
START TRANSACTION;
INSERT INTO orders (customer_id, product_id, quantity) VALUES (1, 1, 2);
UPDATE products SET stock = stock - 2 WHERE product_id = 1;
COMMIT;
```

이 두 작업을 트랜잭션으로 묶지 않으면, 주문은 들어갔는데 재고 차감이 실패하는 반쪽짜리 상태가 생길 수 있다고 한다. 트랜잭션(TCL)은 섹션 3에서 잠깐 언급만 했었는데, 오늘 실제로 이렇게 두 개 이상의 DML을 묶어야 하는 상황을 보고 나니 왜 TCL이 필요한지 다시 한번 체감했다.

# DDL 테이블 생성2 — ERD와 명명 규칙

## ERD로 관계 시각화하기

테이블 3개까지는 SQL 코드만 봐도 관계가 보였는데, 실제 서비스는 테이블이 수십 개라 그렇게 안 될 거라는 이야기를 들었다. 그래서 관계를 그림으로 그려두는 ERD(Entity Relationship Diagram)를 배웠다.

![ERD 관계와 명명 규칙](/assets/img/posts/mysql-ddl-dml-schema-design-constraints/erd-naming-convention-diagram.png)

```
customers (1) ── (N) orders (N) ── (1) products
```

고객 한 명이 주문을 여러 개 낼 수 있어서 1:N, 상품 하나도 여러 주문에 등장할 수 있어서 N:1이라는 것도 배웠다. 신입 개발자가 들어오면 코드부터 읽는 것보다 ERD 한 장 보는 게 훨씬 빠르게 전체 구조를 파악하는 방법이라는 조언이 인상 깊었다. 다음에 팀 프로젝트 시작하면 ERDCloud나 dbdiagram.io로 먼저 그려보고 시작해야겠다.

ERD에는 관계 종류도 표시한다는 걸 배웠다. 1:N 말고도 1:1(한 회원에 프로필 정보 하나만 붙는 경우), N:M(학생과 수강 과목처럼 서로 여러 개씩 연결되는 경우)이 있는데, N:M은 관계형 DB에서 직접 표현이 안 돼서 중간에 연결 테이블을 하나 더 둬야 한다고 한다. 이 부분은 "정규화" 파트에서 더 자세히 나온다고 하니 그때 다시 정리할 예정이다.

## 테이블·컬럼 명명 규칙

한 사람은 `Customer`, 다른 사람은 `customer_table`이라고 제각각 이름을 지으면 나중에 합칠 때 헷갈린다는 이야기부터 시작했다.

```
테이블명: 복수형, 소문자, 스네이크 케이스   → customers, orders
컬럼명:   스네이크 케이스                    → customer_id, created_at
PK명:     테이블명_id 형태로 통일             → customer_id, order_id
```

PK를 그냥 `id`로 짓지 않고 `customer_id`처럼 테이블명을 붙이는 이유가 특히 와닿았다. `orders.customer_id`처럼 FK로 참조할 때 이름이 자연스럽게 이어져서, 코드만 보고도 어떤 테이블의 무엇을 참조하는지 바로 알 수 있다는 거다. 만약 PK를 다 `id`로 지었다면 `orders` 안에서 `id`가 여러 개(customers의 id, products의 id) 뒤섞여서 헷갈렸을 것 같다.

강의에서 다루진 않았지만, boolean 컬럼은 `is_`나 `has_` 접두사를 붙이고(`is_deleted`, `has_reviewed`), 날짜 컬럼은 `_at` 접미사를 붙이는(`created_at`, `updated_at`) 관례도 있다고 곁가지로 찾아봤는데, 이게 내가 원래 코드 짤 때 지키던 네이밍 규칙이랑 정확히 똑같아서 반가웠다. 코드 컨벤션과 DB 네이밍 컨벤션이 결이 같이 간다는 걸 처음 알았다.

안티패턴도 하나 배웠다. 테이블명에 타입 접두사를 붙이는 것(`tbl_customers`)이나, 컬럼명에 테이블명을 매번 반복하는 것(`customers.customer_name`, `customers.customer_email`)은 피해야 한다고 한다. 이미 `customers` 테이블 안에 있다는 게 명확한데 컬럼명에 또 `customer_`를 반복하면 오히려 코드가 장황해진다는 거였다.

# DDL 테이블 변경·제거 — ALTER, DROP, TRUNCATE

## ALTER TABLE로 구조 바꾸기

customers 테이블에 전화번호 컬럼을 빼먹었다고 가정하고, 이미 데이터가 쌓인 테이블에 컬럼을 추가하는 법을 배웠다.

```sql
ALTER TABLE customers ADD phone VARCHAR(20) DEFAULT '미등록';
ALTER TABLE customers MODIFY phone VARCHAR(30);
ALTER TABLE customers CHANGE phone contact_number VARCHAR(30);
ALTER TABLE customers DROP COLUMN contact_number;
```

`MODIFY`와 `CHANGE`의 차이도 처음 알았다. `MODIFY`는 컬럼 이름은 그대로 두고 타입만 바꾸고, `CHANGE`는 컬럼 이름까지 함께 바꿀 수 있다고 한다. 컬럼을 추가할 때 `DEFAULT` 없이 `NOT NULL`을 걸면, 이미 있는 행들 때문에 에러가 날 수 있다고 해서 실무에서는 `DEFAULT`를 같이 지정하는 경우가 많다고 한다.

테이블 이름 자체를 바꾸는 법도 곁가지로 배웠다.

```sql
ALTER TABLE customers RENAME TO members_v2;
```

그리고 이미 만든 테이블에 나중에 FK 제약을 추가하는 법도 있었다.

```sql
ALTER TABLE orders ADD CONSTRAINT fk_customer
  FOREIGN KEY (customer_id) REFERENCES customers(customer_id);
```

처음 설계할 때 FK를 빼먹었더라도, 나중에 이렇게 제약을 추가로 걸 수 있다는 게 유용해 보였다. 다만 이미 들어있는 데이터 중에 참조 무결성을 위반하는 값이 있으면 이 `ALTER TABLE`도 실패한다고 하니, 기존 데이터를 먼저 점검해야 한다는 것도 알아뒀다.

## DROP vs TRUNCATE vs DELETE

섹션 3에서 `DELETE`와 `DROP`의 차이는 배웠는데, 오늘은 그 사이에 있는 `TRUNCATE`까지 정리했다.

![DROP과 TRUNCATE 비교](/assets/img/posts/mysql-ddl-dml-schema-design-constraints/drop-vs-truncate-comparison.png)

```
              DELETE           TRUNCATE          DROP
분류          DML              DDL               DDL
WHERE 조건    가능              불가              불가
구조 유지      O                O                 X
AUTO_INCREMENT  유지           리셋               (테이블 자체 삭제)
```

`TRUNCATE`가 조건 없는 `DELETE`와 결과가 비슷해 보이지만 완전히 다르다는 게 새로웠다. `TRUNCATE`는 DDL이라 테이블을 통째로 재생성하는 방식에 가까워서 훨씬 빠르지만, `AUTO_INCREMENT` 값도 1로 리셋된다. `DELETE`는 그렇지 않고 유지된다.

속도 차이가 궁금해서 좀 더 찾아봤는데, `DELETE`는 행을 하나씩 지우면서 그 과정을 트랜잭션 로그에 다 기록하는 반면, `TRUNCATE`는 테이블이 차지하던 공간 자체를 통째로 반납하고 새로 할당하는 방식이라 로그 기록이 훨씬 적다고 한다. 그래서 수백만 건짜리 테이블을 통째로 비울 땐 `DELETE`보다 `TRUNCATE`가 압도적으로 빠르다는 거였다. 대신 `TRUNCATE`는 `WHERE` 조건을 아예 못 쓰니까 "일부만 지우고 싶다"면 무조건 `DELETE`를 써야 한다.

## FK 체크 비활성화

대량의 초기 데이터를 여러 테이블에 한꺼번에 밀어넣어야 하는데 참조 순서가 꼬여서 FK 체크에 계속 걸리는 경우가 있다고 해서, 잠깐 검사를 끄는 법도 배웠다.

```sql
SET FOREIGN_KEY_CHECKS = 0;

INSERT INTO orders (order_id, customer_id, product_id, quantity) VALUES (101, 5, 3, 2);
INSERT INTO customers (customer_id, name, email) VALUES (5, '김민준', 'minjun@test.com');
-- 순서가 뒤바뀌어도 정상 삽입됨 (원래는 orders가 먼저면 FK 위반)

SET FOREIGN_KEY_CHECKS = 1;
```

원래대로라면 `orders`를 먼저 넣을 때 `customer_id = 5`인 고객이 아직 없으니 `1452` 에러가 났을 텐데, 체크를 꺼두니 순서 상관없이 다 들어갔다. 이건 "일부러 규칙을 무시해야 하는 특수한 상황"에서만 쓰는 예외적인 방법이라고 강조됐다. 체크를 꺼둔 상태에서 실수로 잘못된 값을 넣으면, 나중에 체크를 다시 켜도 이미 들어간 데이터는 안 걸러진다는 주의사항도 기억해뒀다. 대량 데이터 마이그레이션 스크립트에서나 짧게 켰다 꺼야지, 평소 운영 중인 서비스에서 이 스위치를 함부로 건드리면 안 된다는 조언이 있었다.

# DML 등록 — INSERT 3가지 패턴

지금까지 INSERT를 상황마다 조금씩 다르게 썼는데, 오늘 3가지 패턴으로 정리했다.

![INSERT 3가지 패턴](/assets/img/posts/mysql-ddl-dml-schema-design-constraints/insert-three-patterns-flow.png)

## 전체 컬럼 생략 vs 컬럼 지정

```sql
-- ① 전체 컬럼 생략 (위험)
INSERT INTO members VALUES (1, '김민준', 20);

-- ② 특정 컬럼 지정 (권장)
INSERT INTO members (member_id, name) VALUES (1, '김민준');
```

①은 나중에 `ALTER TABLE`로 컬럼이 추가되거나 순서가 바뀌면 조용히 잘못된 자리에 값이 들어갈 수 있어서 위험하다고 배웠다. 오늘 배운 `ALTER TABLE ADD` 명령어랑 연결해서 생각해보니 왜 위험한지 더 명확해졌다. 만약 `age` 다음에 `phone` 컬럼을 추가했는데 어딘가에 ①번 패턴으로 짜여진 코드가 남아있다면, 그 코드는 `phone` 자리에 엉뚱한 값을 넣어버릴 거다. ②는 컬럼명을 이름표처럼 명시해서 순서가 바뀌어도 안전하다.

## 다중 VALUES 패턴

```sql
-- ③ 다중 VALUES (효율적)
INSERT INTO members (member_id, name) VALUES
(1, '김민준'), (2, '이서연'), (3, '박지훈');
```

`INSERT`를 여러 번 따로 실행하는 것보다 훨씬 빠른데, 매번 클라이언트-서버를 왕복하는 시간이 줄어들기 때문이라고 한다. 실무 기준으로는 ②와 ③을 조합해서 "컬럼을 명시하면서 여러 행을 한 번에" 넣는 게 가장 안전하고 효율적이라는 정리가 좋았다.

## 곁가지로 배운 INSERT 응용 문법

강의 범위는 아니었지만 관련해서 찾아본 게 있다. `INSERT ... ON DUPLICATE KEY UPDATE`라는 문법인데, PK나 UNIQUE 값이 이미 존재하면 에러 대신 자동으로 `UPDATE`를 실행해준다고 한다.

```sql
INSERT INTO products (product_id, name, stock) VALUES (1, '키보드', 10)
ON DUPLICATE KEY UPDATE stock = stock + 10;
```

이미 `product_id = 1`이 있으면 재고를 10 더해주고, 없으면 새로 삽입하는 식이다. 이런 걸 흔히 "upsert"(update + insert)라고 부른다고 하는데, 재고 입고 처리 같은 데 딱 맞는 패턴 같아서 메모해뒀다. 강의 진도가 더 나가면 정식으로 다룰지 궁금하다.

# DML 수정 — UPDATE, 안전 업데이트 모드

## 안전 업데이트 모드

3-2단계에서 `WHERE` 누락 재앙은 이미 배웠는데, 오늘은 워크벤치가 이걸 실제로 막아주는 안전장치를 자세히 배웠다.

![워크벤치 안전 업데이트 모드 에러 화면](/assets/img/posts/mysql-ddl-dml-schema-design-constraints/mysql-workbench-safe-update-error-mockup.png)

```sql
UPDATE members SET name = '변경' WHERE age > 20;
```

```
Error Code: 1175. You are using safe update mode and you tried to update
a table without a WHERE that uses a KEY column.
```

`age`는 PK도 UNIQUE도 아니라서 워크벤치가 아예 거부한 거다. `member_id`로 조건을 바꾸면 정상 실행된다. 강의에서는 이 모드를 끄지 말고 그냥 켜둔 채로 쓰라고 추천했는데, 어차피 PK로 조건을 거는 게 정석이니 이 에러가 뜬다는 건 오히려 "지금 조건을 잘못 걸고 있다"는 신호로 받아들이면 된다는 게 납득됐다.

이 모드를 끄는 방법도 궁금해서 찾아봤다. 워크벤치 메뉴(Edit > Preferences > SQL Editor)에서 체크박스를 끌 수도 있고, SQL 세션에서 직접 이렇게 끌 수도 있다고 한다.

```sql
SET SQL_SAFE_UPDATES = 0;   -- 이번 세션만 끄기 (비추천)
```

강의에서는 이걸 끄는 상황이 거의 없어야 한다고 강조했는데, 정말 대량으로 조건 없는 업데이트가 필요한 배치 작업 정도만 예외라고 한다.

## 변경 전 SELECT 확인 습관

안전 업데이트 모드는 "PK 조건이 있는지"만 확인해주지, "그 조건이 진짜 원하는 행이 맞는지"까지는 확인 못 해준다고 한다. 그래서 사람이 스스로 지켜야 하는 습관이 하나 더 필요했다.

```sql
-- 먼저 확인
SELECT * FROM members WHERE member_id = 1;
-- 맞으면 그제서야 실행
UPDATE members SET name = '김철수' WHERE member_id = 1;
```

대량으로 여러 행을 바꿀 땐 `SELECT COUNT(*)`로 몇 건이 영향받는지 먼저 세어보는 것도 좋은 습관이라고 배웠다.

```sql
SELECT COUNT(*) FROM members WHERE status = '휴면';
-- 예상한 숫자와 맞는지 확인 후
UPDATE members SET status = '탈퇴' WHERE status = '휴면';
```

여러 명이 동시에 같은 행을 수정하려고 하면 어떻게 되는지도 궁금해서 질문했었는데, 이건 "낙관적 락(optimistic lock)"이라는 개념과 관련이 있다고 살짝 언급만 됐다. 버전 컬럼을 하나 두고 `UPDATE ... WHERE id = 1 AND version = 3`처럼 조건에 버전까지 포함시켜서, 그 사이에 다른 사람이 먼저 수정했으면 이번 업데이트가 아무 행도 바꾸지 못하게 만드는 방식이라고 한다. 이건 나중에 트랜잭션·동시성 파트에서 제대로 다룬다고 하니 그때 다시 정리할 예정이다. 안전 업데이트 모드는 도구가 막아주는 안전장치고, `SELECT` 선확인은 사람이 스스로 지키는 습관이라는 구분이 명확해졌다.

# DML 삭제 — DELETE와 사고 대처법

## WHERE 누락이 왜 특히 무서운가

오타 낸 코드는 다시 고치면 그만인데, 데이터를 지우는 명령어는 실행되는 순간 "그 데이터가 존재했다는 사실 자체"가 사라진다고 배웠다. 코드는 원상복구가 되지만 데이터는 원본이 없으면 복구가 안 된다는 게 다른 버그랑 결정적으로 달랐다.

## 사고 대처법 — 커밋 여부가 갈림길

![사고 발생 시 대처 흐름](/assets/img/posts/mysql-ddl-dml-schema-design-constraints/delete-accident-recovery-flow.png)

```
[커밋 전]
DELETE FROM members;  -- 아직 COMMIT 안 함
ROLLBACK;              -- 한 줄로 전부 원상복구

[커밋 후, 백업 있음]
1. 즉시 추가 쓰기 중단
2. 최근 백업본(mysqldump)으로 복원 (백업 시점까지만)

[커밋 후, 바이너리 로그(binlog)까지 있음]
1. 백업 복원 + binlog로 백업 이후~사고 직전 변경분 재생
2. 사고 직전 상태까지 정밀 복구 가능
```

`COMMIT`을 실행하기 전까지는 삭제가 확정된 게 아니라는 걸 알고 나서, 대량 삭제 작업은 auto-commit을 끄고 트랜잭션 안에서 결과를 확인한 뒤 커밋하는 습관을 들여야겠다고 생각했다.

바이너리 로그 복구가 실제로 어떻게 생겼는지도 찾아봤다. `mysqlbinlog`라는 도구로 로그 파일을 읽어서 특정 시점까지의 변경 내역만 뽑아낼 수 있다고 한다.

```bash
mysqlbinlog --start-datetime="2026-07-06 10:00:00" \
            --stop-datetime="2026-07-06 10:29:00" \
            binlog.000123 | mysql -u root -p
```

"사고가 10시 30분에 났다"는 걸 알면, 그 직전까지의 로그만 재생해서 복구할 수 있다는 원리다. 금융권처럼 정합성이 극도로 중요한 곳은 이 바이너리 로그를 오래 보관해서 초 단위까지 정밀 복구가 가능하게 운영한다고 한다.

삭제 대신 아예 지우지 않는 방법도 있다는 걸 곁가지로 배웠다. `is_deleted` 같은 boolean 컬럼을 두고 실제로는 `UPDATE`로 상태만 바꾸는 소프트 삭제 패턴이다. 이러면애초에 `DELETE`를 실행할 일 자체가 줄어들어서 사고 위험이 낮아진다고 한다. 대신 모든 조회 쿼리에 `WHERE is_deleted = false`를 계속 붙여야 하는 번거로움이 있다는 트레이드오프도 있었다. 강의에서 제일 강조한 건 "사고 대처법보다 예방이 백 배 중요하다"는 거였다. 안전 업데이트 모드, `WHERE` 확인, `SELECT` 선확인, 이 세 가지가 진짜 핵심이었다.

# 제약 조건 활용 — 실전 오류 재현

섹션 4의 마지막은 지금까지 배운 제약 조건들을 일부러 위반해보면서 정확히 어떤 에러가 나는지 확인하는 실습이었다.

![제약 조건 위반 에러 코드 정리](/assets/img/posts/mysql-ddl-dml-schema-design-constraints/constraint-violation-error-codes-diagram.png)

## NOT NULL / UNIQUE 위반

```sql
-- NOT NULL 위반
INSERT INTO customers (email) VALUES ('test@test.com');
```
```
Error 1364: Field 'name' doesn't have a default value
```

```sql
-- UNIQUE 위반
INSERT INTO customers (name, email) VALUES ('김철수', 'test@test.com');
```
```
Error 1062: Duplicate entry 'test@test.com' for key
```

## FK 위반 — 양방향

```sql
-- FK 위반 (자식이 없는 부모 참조)
INSERT INTO orders (customer_id, product_id, quantity) VALUES (999, 1, 2);
```
```
Error 1452: Cannot add or update a child row
```

```sql
-- FK 위반 (부모가 참조당하는 중인데 삭제 시도)
DELETE FROM customers WHERE customer_id = 1;
```
```
Error 1451: Cannot delete or update a parent row
```

`1452`와 `1451`을 나란히 보니 방향이 반대라는 게 명확해졌다. `1452`는 자식이 잘못된 부모를 가리키려는 걸 막고, `1451`은 부모가 아직 자식이 필요로 하는데 사라지려는 걸 막는다.

## CHECK 위반과 다중 삽입 실패

products 테이블의 `CHECK (stock >= 0)`도 위반해봤다.

```sql
INSERT INTO products (name, price, stock) VALUES ('마우스', 15000, -3);
```
```
Error 3819: Check constraint 'products_chk_1' is violated.
```

여러 행을 한 번에 넣는 다중 VALUES 패턴에서 중간 행 하나가 제약을 위반하면 어떻게 되는지도 궁금해서 실험해봤다.

```sql
INSERT INTO products (name, price, stock) VALUES
('키보드', 45000, 10),
('마우스', 15000, -3),   -- 이 행이 CHECK 위반
('모니터', 250000, 5);
```

세 번째 행까지 있는데도 두 번째 행에서 에러가 나면 전체 `INSERT`가 실패하고 첫 번째, 세 번째 행도 삽입되지 않는다고 한다. MySQL의 기본 스토리지 엔진(InnoDB)은 이 `INSERT` 문 전체를 하나의 원자적 단위로 처리하기 때문이라는 거다. "일부만 성공하고 일부는 실패하는" 어중간한 상태가 안 생긴다는 게 오히려 안전하게 느껴졌다.

실무에서는 이 에러 코드들을 그대로 사용자에게 보여주지 않고 "이미 사용 중인 이메일입니다" 같은 친절한 메시지로 바꿔서 보여준다고 한다. 부모를 정말 삭제해야 한다면 `ON DELETE CASCADE`를 미리 걸어두거나, 자식 행을 먼저 지운 뒤에 부모를 지우는 순서를 지켜야 한다는 것도 배웠다.

# 마무리

섹션 4는 일곱 꼭지나 되는 긴 섹션이었지만, 지금까지 배운 걸 실제 쇼핑몰 스키마로 다 엮어보는 과정이라 제일 재밌었다. customers/products/orders를 직접 설계하고, ERD로 그려보고, 컬럼을 나중에 고치고, 데이터를 안전하게 넣고 고치고 지우고, 마지막엔 제약 조건을 일부러 어겨보면서 에러 코드까지 눈에 익혔다.

특히 FK 에러 코드(1452, 1451)를 직접 재현해보고 나니, 앞으로 실무에서 이 에러를 만나도 당황하지 않고 바로 원인을 짚을 수 있을 것 같다. `AUTO_INCREMENT`의 번호 구멍, `DECIMAL`의 정확한 저장 원리, 안전 업데이트 모드와 `SELECT` 선확인의 역할 분담까지, 하나하나가 다 "왜 이렇게 설계됐는가"로 이어지는 이야기였다. 다음 섹션은 SELECT 조회와 정렬이라고 하니, `WHERE`·`ORDER BY`를 더 깊게 파볼 예정이다.

---

# 참고 자료

<iframe width="100%" height="420" src="https://www.youtube.com/embed/en_9N6WySic" title="[즐겁게 배우는 SQL #38] 제약 조건 - 외래 키" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" allowfullscreen style="border-radius:12px;"></iframe>
