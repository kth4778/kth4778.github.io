---
title: "MySQL 워크벤치 켜고 처음 SQL 쳐본 날 — WHERE 하나 빠뜨리면 회원 전체가 사라진다"
date: 2026-07-06 20:40:00 +0900
categories: [백엔드, 데이터베이스]
tags: [mysql, sql, ddl, dml, crud, varchar, char, datetime, timestamp, constraint, primary-key, foreign-key, workbench]
image:
  path: /assets/img/posts/mysql-crud-sql-basics-constraints/sql-commands-create-use-table-flow.png
  alt: CREATE부터 DROP까지 명령어 흐름
---

인프런 김영한님 실전 데이터베이스 입문 섹션 3을 정리한다. 섹션 2는 개념만 보는 거였는데, 이번엔 진짜 워크벤치를 켜고 손으로 SQL을 쳐보는 첫 실습 섹션이었다. `CREATE DATABASE`부터 제약조건까지, 다섯 꼭지를 순서대로 정리한다.

# 데이터베이스 시작하기1 — CREATE, USE, DESC, SHOW, DROP

## 데이터베이스 만들고 들어가기

가장 먼저 배운 건 데이터베이스를 만들고 그 안으로 들어가는 순서였다.

```sql
CREATE DATABASE shop;
USE shop;
```

`CREATE DATABASE`를 실행했다고 자동으로 그 안에 들어가 있는 게 아니라는 걸 처음 알았다. 워크벤치 왼쪽 위에 지금 선택된 스키마가 표시되는데, `USE shop;`을 실행해야 그게 `shop`으로 바뀐다. 이 상태에서 `CREATE TABLE`을 해야 그 테이블이 진짜 `shop` 안에 들어간다.

실습하다가 실수로 `USE`를 건너뛰고 바로 `CREATE TABLE members (...)`를 쳤더니 이런 에러가 났다.

```
Error Code: 1046. No database selected
```

당황해서 처음엔 테이블 문법이 틀렸나 한참 봤는데, 알고 보니 그냥 `USE`를 안 해서 "어느 데이터베이스에 만들 건지" DBMS가 모르는 상태였던 거다. 이 에러 메시지를 한 번 겪고 나니 `USE`의 존재 이유가 확실히 각인됐다.

이미 존재하는 이름으로 다시 `CREATE DATABASE shop;`을 실행하면 이런 에러도 났다.

```
Error Code: 1007. Can't create database 'shop'; database exists
```

`CREATE DATABASE IF NOT EXISTS shop;`처럼 조건을 붙이면 이미 있으면 조용히 넘어가고 없으면 만들어준다. 강의에서는 이 패턴을 "멱등성(idempotent)있게 스크립트를 짜는 습관"이라고 표현했는데, 여러 번 실행해도 결과가 똑같아야 하는 배포 스크립트 같은 곳에서 특히 중요하다고 한다.

## 테이블 만들고 구조 확인하기

데이터베이스(그릇)를 만들었으니 이제 테이블(서류함)을 만들 차례였다.

```sql
CREATE TABLE members (
  member_id INT PRIMARY KEY,
  name VARCHAR(50) NOT NULL,
  age INT
);

DESC members;
```

`DESC members;`를 실행하면 이런 결과가 나온다.

```
Field       | Type        | Null | Key | Default
member_id   | int         | NO   | PRI | NULL
name        | varchar(50) | NO   |     | NULL
age         | int         | YES  |     | NULL
```

이 표를 보면 각 컬럼의 타입, NULL 허용 여부, PK 여부까지 한눈에 보인다. `DESC`는 `DESCRIBE`의 줄임말이라는 것도 이번에 처음 알았다. 실무에서는 테이블 구조가 기억 안 날 때마다 `DESC`부터 치는 습관이 생긴다고 하는데, 벌써 그 말이 이해가 됐다. 구글 스프레드시트로 회원 관리하던 옛날엔 헤더 행을 보면 됐는데, DB는 이렇게 명령어로 확인해야 한다는 게 신기했다.

테이블 구조를 만든 명령어 자체를 그대로 보고 싶으면 `SHOW CREATE TABLE members;`도 쓸 수 있다는 걸 곁가지로 배웠다. 이건 `DESC`보다 더 자세한 정보(엔진, 문자셋 등)까지 같이 보여준다고 한다.

```sql
SHOW CREATE TABLE members;
```

```
CREATE TABLE `members` (
  `member_id` int NOT NULL,
  `name` varchar(50) NOT NULL,
  `age` int DEFAULT NULL,
  PRIMARY KEY (`member_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
```

여기서 `ENGINE=InnoDB`라는 게 눈에 들어왔는데, 이건 MySQL이 내부적으로 데이터를 저장하고 트랜잭션을 처리하는 엔진 종류라고 한다. 자세한 건 나중 단계에서 다룬다고 하니 오늘은 "테이블마다 저장 엔진이 있다" 정도만 알아두기로 했다.

![CREATE부터 DROP까지 명령어 흐름](/assets/img/posts/mysql-crud-sql-basics-constraints/sql-commands-create-use-table-flow.png)

## 목록 보고 지우기

만든 것들이 뭐가 있는지 확인하고, 필요 없으면 지우는 명령어도 배웠다.

```sql
SHOW DATABASES;
SHOW TABLES;

DROP TABLE members;
DROP DATABASE shop;
```

`SHOW DATABASES;`는 선택된 DB와 무관하게 서버 전체 목록을 보여주고, `SHOW TABLES;`는 지금 `USE`로 선택된 DB 안의 테이블만 보여준다는 차이가 있었다. `USE` 없이 `SHOW TABLES;`를 치면 "선택된 데이터베이스가 없다"는 에러가 난다는 것도 직접 쳐보고 알았다.

`DROP`은 되돌릴 수 없는 명령어라는 게 계속 강조됐다. `DROP DATABASE shop;`을 실행하면 안의 모든 테이블과 데이터가 통째로 사라진다. 실무에서 실수로 운영 DB를 통째로 날리는 사고가 실제로 종종 일어나는 명령어라고 하니, 이 명령어를 칠 때는 이름을 한 번 더 확인하는 습관을 지금부터 들여야겠다고 생각했다.

`DROP TABLE`도 존재하지 않는 테이블을 지우려고 하면 에러가 난다.

```
Error Code: 1051. Unknown table 'shop.members'
```

여기서도 `DROP TABLE IF EXISTS members;`처럼 조건을 붙이면 안전하다. 강의에서 "실무에서는 `CREATE`든 `DROP`이든 `IF NOT EXISTS`/`IF EXISTS`를 습관적으로 붙이는 경우가 많다"고 했는데, 이번 실습만으로도 왜 그런지 충분히 체감했다.

![mysql CLI로 데이터베이스 목록 확인하는 화면](/assets/img/posts/mysql-crud-sql-basics-constraints/mysql-workbench-show-tables-mockup.png)

# 데이터베이스 시작하기2 — CRUD 맛보기

## INSERT — 데이터 넣기

테이블을 만들었으니 이제 데이터를 넣어볼 차례였다.

```sql
INSERT INTO members (member_id, name, age)
VALUES (1, '김민준', 20);
```

컬럼 목록과 값 목록의 순서가 정확히 일치해야 한다. 컬럼 목록을 생략하고 `INSERT INTO members VALUES (2, '이서연', 22);`처럼 쓸 수도 있는데, 이 경우엔 테이블 정의 순서 그대로 값을 채워야 해서 위험하다고 배웠다. 실무에서는 컬럼명을 명시하는 첫 번째 방식을 더 권장한다고 하는데, 나도 앞으로는 무조건 컬럼명을 써야겠다고 마음먹었다.

여러 행을 한 번에 넣는 법도 배웠다.

```sql
INSERT INTO members (member_id, name, age) VALUES
(3, '박지훈', 25),
(4, '최유진', 19),
(5, '정하늘', 31);
```

한 번에 여러 행을 넣으면 `INSERT`를 여러 번 따로 실행하는 것보다 훨씬 빠르다고 한다. 매번 서버에 요청을 보내고 받는 왕복 시간이 줄어들기 때문이라는데, 나중에 대량 데이터를 옮길 때(마이그레이션) 이 방식을 꼭 써야겠다고 메모해뒀다.

`NOT NULL`인 컬럼을 빼먹고 넣으려고 하면 어떻게 될까 궁금해서 일부러 해봤다.

```sql
INSERT INTO members (member_id, age) VALUES (6, 22);
```

```
Error Code: 1364. Field 'name' doesn't have a default value
```

`name`이 `NOT NULL`인데 값도 안 주고 `DEFAULT`도 없으니 DBMS가 바로 거부한 거다. 섹션 2에서 배운 무결성 개념이 실제로 이렇게 동작하는구나 싶었다.

## SELECT — 데이터 찾기

```sql
SELECT * FROM members;
SELECT name, age FROM members;
SELECT * FROM members WHERE age = 20;
```

여기서 궁금한 게 하나 생겼다. `WHERE`로 조건을 걸면 결국 데이터를 하나씩 다 비교하는 건지, 그러면 시간복잡도가 O(n)인지 질문했었다. 답은 "인덱스가 없으면 정말 O(n)이 맞다"였다. `member_id`처럼 PK로 지정된 컬럼은 자동으로 인덱스(B-Tree)가 생겨서 대략 O(log n)으로 훨씬 빠르게 찾을 수 있지만, `name`처럼 인덱스 없는 컬럼은 첫 행부터 끝까지 다 훑는 풀 스캔이라는 거였다.

```
[인덱스 없음, name으로 검색]        [인덱스 있음, member_id로 검색]
1번 행 확인 → 다름                      루트 노드에서 비교
2번 행 확인 → 다름                      → 왼쪽/오른쪽 선택
...전체 순회 (O(n))                     → log n번 만에 도달 (O(log n))
```

인덱스는 나중 단계에서 더 깊게 다룬다고 하니 오늘은 "인덱스 유무가 성능을 좌우한다"는 정도만 기억해두기로 했다.

`SELECT`는 데이터를 전혀 바꾸지 않는 읽기 전용 동작이라, `UPDATE`나 `DELETE` 전에 같은 조건으로 먼저 `SELECT`를 해보고 어떤 행이 영향받는지 확인하는 습관이 중요하다는 것도 배웠다. 그리고 조회 결과 개수를 제한하고 싶으면 `LIMIT`을 쓸 수 있다는 것도 곁가지로 배웠다.

```sql
SELECT * FROM members LIMIT 3;   -- 앞에서 3건만 조회
```

이건 나중에 5장(SQL 조회와 정렬)에서 페이징 처리랑 같이 훨씬 깊게 다룬다고 한다.

## UPDATE — 데이터 고치기

```sql
UPDATE members
SET name = '김철수'
WHERE member_id = 1;
```

`WHERE` 절을 빼먹으면 `members` 테이블의 모든 행의 `name`이 전부 '김철수'로 바뀌어버린다. 실무에서 정말 자주 일어나는 사고 유형이라고 해서, 앞으로는 무조건 `SELECT`로 먼저 확인하고 `UPDATE`를 실행하는 순서를 지켜야겠다고 다짐했다.

```sql
-- 먼저 확인
SELECT * FROM members WHERE member_id = 1;
-- 맞으면 그제서야 실행
UPDATE members SET name = '김철수' WHERE member_id = 1;
```

워크벤치는 기본적으로 "안전 업데이트 모드(Safe Updates)"가 켜져 있어서, PK나 UNIQUE 조건 없이 `UPDATE`/`DELETE`를 실행하려고 하면 아예 막아주기도 한다고 한다. 실습 중에 한 번 이 모드에 걸려서 에러 메시지가 뭔지 몰라 당황했는데, 알고 보니 이게 오히려 "사고를 막아주는 안전장치"였다는 걸 알고 나니 오히려 고마웠다.

컬럼 여러 개를 한 번에 바꾸고 싶으면 쉼표로 이어 쓰면 된다는 것도 배웠다.

```sql
UPDATE members
SET name = '김철수', age = 21
WHERE member_id = 1;
```

## DELETE — 데이터 지우기

```sql
DELETE FROM members
WHERE member_id = 1;
```

`WHERE`를 빼먹으면 `members` 테이블의 모든 행이 삭제된다. 강의에서 이걸 "WHERE 누락 재앙 시나리오"라고 표현했는데 정말 딱 맞는 말이었다. `DELETE FROM members;`(전체 삭제)와 `DROP TABLE members;`(3-1단계에서 배운 것)는 결과가 비슷해 보이지만 완전히 다르다는 것도 헷갈리지 않게 짚었다. `DELETE`는 데이터만 지우고 테이블 구조는 남기지만, `DROP`은 테이블 자체를 통째로 없앤다.

강의에서 소개한 "사고 대처법"도 인상 깊었다. 만약 `WHERE` 없이 `DELETE`를 실행해버렸다면, 트랜잭션 안에서 실행 중이었다면 `ROLLBACK`으로 되돌릴 수 있지만, 이미 `COMMIT`까지 됐다면 백업본으로 복구하는 수밖에 없다고 한다. 이 부분은 TCL(트랜잭션 제어)이랑 이어지는 이야기라 오늘 배운 SQL 4대 분류 챕터에서 다시 짚어볼 예정이다.

![MySQL 워크벤치에서 UPDATE 실행 전 WHERE 조건으로 확인하는 화면](/assets/img/posts/mysql-crud-sql-basics-constraints/mysql-workbench-update-where-mockup.png)

CRUD 네 개를 실제로 손으로 쳐보니, 회원가입 버튼 누르면 서버 내부에서 `INSERT`가 실행되고, 마이페이지 열면 `SELECT`가, 닉네임 바꾸면 `UPDATE`가, 회원 탈퇴하면 `DELETE`(또는 소프트 삭제)가 실행된다는 게 실감이 났다. 지금까지 그냥 API로만 쓰던 게, 뒤에서는 이런 SQL 한 줄들이 돌고 있었던 거다. 특히 회원 탈퇴는 실제로 `DELETE`를 바로 쓰기보다 `is_deleted` 컬럼을 두고 `UPDATE`로 상태만 바꾸는 소프트 삭제 패턴을 많이 쓴다는 이야기도 들었는데, 다음에 팀 프로젝트에서 탈퇴 기능 만들 때 꼭 참고해야겠다.

# SQL이란? — 표준 SQL, 선언적 언어, 4대 분류

## SQL은 표준 언어다

지금까지 쓴 명령어들을 왜 다 "SQL"이라고 뭉뚱그려 부르는지도 정리됐다. SQL은 ANSI(American National Standards Institute)가 정한 표준 언어라서, Oracle을 쓰든 MySQL을 쓰든 "조회하려면 SELECT"라는 규칙은 똑같다. 다만 각 DBMS는 표준 위에 자기만의 방언(dialect)을 얹는다. `CONCAT()` 같은 MySQL 전용 함수가 그 예시다.

```sql
-- MySQL 방식
SELECT CONCAT(first_name, ' ', last_name) FROM members;

-- 표준 SQL 방식 (PostgreSQL 등에서 동작)
SELECT first_name || ' ' || last_name FROM members;
```

강의에서는 "표준 SQL만 알면 어느 DBMS로 옮겨도 80%는 그대로 통한다. 나머지 20%가 방언이다"라고 표현했는데, 이 비유가 꽤 와닿았다. 이직하거나 새 프로젝트에서 다른 DBMS를 만나도 기초는 그대로 통한다는 뜻이니까.

## 선언적 언어라는 것

`SELECT * FROM members WHERE age = 20;`이라고 쓰면 "age가 20인 행을 원한다"고만 말할 뿐, 어떤 순서로 어떻게 찾을지는 DBMS가 알아서 판단한다. 이런 방식을 선언적 언어(declarative language)라고 부르고, 반대로 "어떻게" 할지 순서를 하나하나 지정하는 방식은 절차적 언어(procedural language)라고 한다는 걸 처음 알았다.

```
절차적 언어 사고방식 (예: Python):
for row in members:
    if row.age == 20:
        result.append(row)

선언적 언어 사고방식 (SQL):
SELECT * FROM members WHERE age = 20;
```

![선언적 언어 vs 절차적 언어 비교](/assets/img/posts/mysql-crud-sql-basics-constraints/declarative-vs-procedural-comparison.png)

우리가 겪었던 "WHERE가 O(n)이냐 O(log n)이냐"는 질문도 결국 이 선언적 언어의 특징 때문이었다. "무엇을" 원하는지만 정확히 표현하면, "어떻게 빠르게 찾을지"는 DBMS 내부 옵티마이저가 알아서 결정한다는 게 SQL 설계 철학의 핵심이라고 한다. 코드로 직접 순회 로직을 짜면 몇 줄이 필요한지, 인덱스를 쓸지 말지도 개발자가 직접 신경 써야 하는데, SQL은 이 고민을 DBMS에게 통째로 위임한다는 게 신선했다.

## DDL / DML / DCL / TCL — 4대 분류

지금까지 쓴 명령어들을 하는 일별로 정리하면 이랬다.

```
DDL (Data Definition Language)     — CREATE, ALTER, DROP
DML (Data Manipulation Language)   — INSERT, SELECT, UPDATE, DELETE
DCL (Data Control Language)        — GRANT, REVOKE
TCL (Transaction Control Language) — COMMIT, ROLLBACK
```

3-1단계에서 쓴 `CREATE`, `DROP`은 DDL, 3-2단계에서 쓴 `INSERT`/`SELECT`/`UPDATE`/`DELETE`는 전부 DML이었다는 걸 이번에 정리하면서 알았다. `SELECT`가 데이터를 바꾸지 않고 읽기만 하는데도 DML로 분류된다는 게 은근히 헷갈렸는데, "데이터를 다루는 언어"라고 이해하면 맞는 거였다.

DCL은 실제로 이런 형태라고 한다.

```sql
GRANT SELECT ON shop.members TO 'junior_dev'@'%';
REVOKE DELETE ON shop.members FROM 'junior_dev'@'%';
```

회사 DB 관리자가 신입 개발자에게는 조회 권한만 주고, 삭제 권한은 회수해서 사고를 원천 차단하는 식이다. 아직 실습은 안 해봤지만, 나중에 팀 프로젝트에서 DB 계정을 나눠 쓸 때 꼭 써먹어봐야겠다는 생각이 들었다.

TCL은 계좌 이체 같은 시나리오에서 등장한다고 한다.

```sql
START TRANSACTION;
UPDATE accounts SET balance = balance - 10000 WHERE id = 1;  -- 출금
UPDATE accounts SET balance = balance + 10000 WHERE id = 2;  -- 입금
COMMIT;   -- 둘 다 성공하면 확정
-- 문제가 생기면 ROLLBACK;
```

출금은 됐는데 입금 직전에 서버가 죽으면 어떻게 될까 걱정했는데, `COMMIT` 전이면 `ROLLBACK`으로 출금까지 포함해서 전부 없던 일로 되돌릴 수 있다고 한다. 이 둘(DCL, TCL)은 아직 본격적인 실습을 안 해봐서 다음에 나오면 그때 더 자세히 정리할 예정이다.

# 데이터 타입 — VARCHAR vs CHAR, DATETIME vs TIMESTAMP

## VARCHAR vs CHAR

지금까지 아무 생각 없이 `VARCHAR(50)`이라고 써왔는데, 왜 `CHAR(50)`이 아닌지 이번에 제대로 배웠다.

```sql
CREATE TABLE test (
  code CHAR(5),
  name VARCHAR(50)
);

INSERT INTO test VALUES ('AB', '김민준');
-- code: "AB   " (2글자 + 공백 3칸 = 5칸 고정)
-- name: "김민준" (3글자만 저장)
```

`CHAR(5)`는 "AB"만 넣어도 항상 5칸어치 공간을 다 차지하고 남는 칸은 공백으로 채운다. `VARCHAR(50)`은 실제 값 길이만큼만 저장한다. `VARCHAR(50)`의 50이 "항상 50칸을 차지한다"는 뜻이 아니라 "최대 50자까지 허용한다"는 뜻이라는 걸 헷갈리고 있었는데 이번에 확실히 정리됐다.

![CHAR와 VARCHAR 저장 방식 비교](/assets/img/posts/mysql-crud-sql-basics-constraints/char-vs-varchar-storage-comparison.png)

우편번호나 국가 코드처럼 길이가 항상 일정한 데이터는 CHAR가 어울리고, 이름이나 이메일처럼 길이가 들쭉날쭉한 데이터는 VARCHAR를 써야 공간을 낭비하지 않는다고 한다. 실무에서는 CHAR보다 VARCHAR를 압도적으로 많이 쓴다고도 했는데, 길이가 고정된 데이터라도 나중에 요구사항이 바뀌면(예: 우편번호 자릿수 변경) VARCHAR 쪽이 대응하기 편해서라는 이유가 그럴듯했다.

숫자 타입도 곁가지로 짚었다. `INT`는 대략 -21억~21억 범위를 저장하고, 더 큰 값이 필요하면 `BIGINT`, 정확한 소수점 계산(금액 등)이 필요하면 `DECIMAL`을 쓴다고 한다. `FLOAT`/`DOUBLE`은 부동소수점이라 금액 계산에는 오차가 생길 수 있어서 피해야 한다는 것도 짚었는데, 이건 나중에 실제로 금액 컬럼을 설계할 때 꼭 기억해둬야 할 것 같다.

## DATETIME vs TIMESTAMP

```
              DATETIME              TIMESTAMP
시간대        고려 안 함             자동 변환
저장 범위     1000~9999년            1970~2038년
저장 공간     8바이트                 4바이트
```

DATETIME은 그냥 값 자체를 그대로 저장하고, TIMESTAMP는 내부적으로 UTC로 저장했다가 조회할 때 서버/커넥션의 시간대 설정에 맞춰 자동으로 변환해준다고 한다. 글로벌 서비스처럼 여러 시간대 사용자를 다뤄야 하면 TIMESTAMP가 유리하고, 시간대 변환 없이 그 값 그대로를 고정해서 기록하고 싶으면(회의 예약 시각 같은) DATETIME이 맞다는 거다.

![DATETIME과 TIMESTAMP 비교](/assets/img/posts/mysql-crud-sql-basics-constraints/datetime-vs-timestamp-comparison.png)

TIMESTAMP가 2038년까지밖에 저장 못 한다는 것도 처음 알았다. 이른바 "2038년 문제"라고 부르는 유명한 이슈랑 같은 원인이라고 하는데, 지금 만드는 서비스가 그때까지 살아있을진 모르겠지만 알아두면 나쁠 거 없는 지식이었다.

`created_at`, `updated_at` 컬럼을 만들 때 유용한 옵션도 배웠다.

```sql
CREATE TABLE posts (
  post_id INT PRIMARY KEY,
  title VARCHAR(100),
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);
```

`DEFAULT CURRENT_TIMESTAMP`는 행이 생성될 때 현재 시각을 자동으로 채워주고, `ON UPDATE CURRENT_TIMESTAMP`는 그 행이 수정될 때마다 자동으로 시각을 갱신해준다. 게시글 작성일시·수정일시를 관리할 때 이 패턴을 그대로 써먹으면 되겠다 싶었다.

# 제약 조건 — NOT NULL, UNIQUE, PK, FK, DEFAULT, CHECK

## NOT NULL / UNIQUE

```sql
CREATE TABLE members (
  member_id INT PRIMARY KEY,
  name VARCHAR(50) NOT NULL,
  email VARCHAR(100) UNIQUE
);
```

`NOT NULL`은 반드시 값이 있어야 한다는 규칙이고, `UNIQUE`는 값이 있다면 그 값이 테이블 안에서 중복되면 안 된다는 규칙이다. `UNIQUE`는 `NULL`을 여러 개 허용한다는 것도 신기했다(NULL은 "값이 없다"는 상태라서 서로 비교 대상이 안 된다고 한다). 회원가입할 때 이메일 중복 체크를 애플리케이션 코드로만 하는 줄 알았는데, DB 레벨에서 `UNIQUE`로 이중 방어가 가능하다는 걸 알게 됐다.

실제로 중복 이메일을 넣어봤더니 이런 에러가 났다.

```sql
INSERT INTO members (member_id, name, email) VALUES (10, '김철수', 'a@test.com');
INSERT INTO members (member_id, name, email) VALUES (11, '이영희', 'a@test.com');
```

```
Error Code: 1062. Duplicate entry 'a@test.com' for key 'members.email'
```

애플리케이션에서 "이미 사용 중인 이메일입니다"라는 친절한 메시지로 바꿔주는 것도, 결국 이 DB 에러를 잡아서 가공하는 거였다는 걸 알게 됐다.

## PK / FK — 복습

섹션 2에서 이미 자세히 다뤘던 내용이라 여기선 짧게. 오늘 새로 알게 된 건 PK가 사실 `NOT NULL`과 `UNIQUE`를 합쳐놓은 것과 같다는 점이다.

```sql
member_id INT PRIMARY KEY
-- 사실상 아래와 동일한 효과
member_id INT NOT NULL UNIQUE
```

PK는 완전히 새로운 개념이 아니라 오늘 배운 두 제약을 조합한 특별한 이름이었다는 게 재밌었다. 다만 한 테이블에 PK는 하나만 지정 가능하고, UNIQUE는 여러 컬럼에 각각 걸 수 있다는 차이가 있다. 그리고 FK에 걸 수 있는 삭제 옵션도 곁가지로 배웠다.

```sql
FOREIGN KEY (member_id) REFERENCES members(member_id)
  ON DELETE CASCADE   -- 회원이 삭제되면 관련 주문도 같이 삭제
```

`ON DELETE CASCADE`를 걸면 참조당하는 회원이 삭제될 때 관련 주문도 자동으로 같이 삭제된다. 기본값은 삭제를 막는 거였는데(섹션 2에서 배운 내용), 이렇게 옵션으로 "연쇄 삭제"를 허용할 수도 있다는 게 새로웠다. 다만 실무에서는 이 옵션을 남발하면 의도치 않은 대량 삭제가 발생할 수 있어서 신중하게 써야 한다는 조언도 들었다.

## DEFAULT / CHECK

```sql
CREATE TABLE members (
  member_id INT PRIMARY KEY,
  status VARCHAR(20) DEFAULT '대기중',
  age INT CHECK (age >= 0)
);

INSERT INTO members (member_id, age) VALUES (1, 25);
-- status는 명시 안 했으니 자동으로 '대기중'
```

`DEFAULT`는 `INSERT` 시점에 그 컬럼 값을 아예 안 넣었을 때만 적용되고, `CHECK`는 값이 들어올 때마다 조건식을 검사해서 `false`면 저장을 거부한다. 재고관리 시스템에서 수량 컬럼에 `CHECK (quantity > 0)`를 걸어서 음수 재고가 애초에 안 생기게 막는다는 예시가 인상 깊었다.

`CHECK` 위반도 직접 실습해봤다.

```sql
INSERT INTO members (member_id, age) VALUES (2, -5);
```

```
Error Code: 3819. Check constraint 'members_chk_1' is violated.
```

애플리케이션 코드에서 "나이는 0 이상이어야 합니다" 같은 검증을 이미 해뒀어도, 어쩌다 그 검증을 우회하는 경로(배치 스크립트, 다른 서비스에서의 직접 접근 등)가 생기면 결국 DB의 `CHECK`가 마지막 방어선이 된다는 걸 알게 됐다.

![members 테이블에 걸린 제약 조건들](/assets/img/posts/mysql-crud-sql-basics-constraints/constraint-types-table-overview.png)

`CHECK` 제약은 MySQL 옛날 버전에서는 문법만 허용하고 실제로 무시됐다고 한다. MySQL 8.0.16 이상부터 제대로 동작한다고 하니, 나중에 회사에서 레거시 MySQL을 쓰게 되면 버전부터 확인해야겠다고 메모해뒀다.

# 마무리

섹션 3은 처음으로 직접 손 쓰면서 배운 실습 섹션이었다. 특히 `WHERE` 절 하나 빠뜨리면 전체 데이터가 날아간다는 걸 눈으로 직접 확인하고 나니, 왜 실무에서 `UPDATE`/`DELETE` 전에 `SELECT`로 먼저 확인하라고 그렇게 강조하는지 몸으로 이해가 됐다.

제약 조건도 마찬가지로, 애플리케이션 코드로만 막으면 되는 줄 알았는데 DB 레벨에서 한 번 더 막아준다는 게 생각보다 든든하게 느껴졌다. `NOT NULL`, `UNIQUE`, `CHECK`를 실제로 위반해보면서 에러 메시지를 직접 눈으로 본 게 이번 섹션에서 가장 기억에 남는 부분이었다. 개념으로만 알던 것과, 실제로 에러 코드(1046, 1007, 1364, 1062, 3819)를 마주치면서 배우는 건 확실히 체감이 달랐다.

다음 섹션은 실전 테이블 설계(customers/products/orders)라고 하니, 오늘 배운 제약 조건들을 실제로 써먹어볼 차례다.

---

# 참고 자료

<iframe width="100%" height="420" src="https://www.youtube.com/embed/SHSs9X6qiLc" title="[이것이 MySQL이다] 03. MySQL 전체 운영 실습(1)" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" allowfullscreen style="border-radius:12px;"></iframe>
