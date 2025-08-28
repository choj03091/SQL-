# SQL 문제 풀이 & 성능 최적화 노트 (MySQL · Programmers 기준)

> **목표**: 빠르게 맞는 해법 + 인덱스를 타는 **SARGable**한 쿼리 습관.  
> **스타일**: 별칭은 테이블 앞글자(예: `ITEM_INFO`→`if`, `ITEM_TREE`→`it`) + 역할 접미사(`ifp`,`ifc`).  
> **MySQL 팁**: 별칭/컬럼을 정렬이나 그룹에서 쓸 때 **백틱**을 권장 — `AS \`별칭\``.

---

## 목차
- [1) 기본 규칙(성능 지향)](#1-기본-규칙성능-지향)
- [2) SQL 논리적 실행 순서](#2-sql-논리적-실행-순서)
- [3) 별칭(alias) & 인용(백틱)](#3-별칭alias--인용백틱)
- [4) 날짜·시간 필터링](#4-날짜시간-필터링)
- [5) GROUP BY vs HAVING](#5-group-by-vs-having)
- [6) ORDER BY 주의점](#6-order-by-주의점)
- [7) REGEXP vs LIKE](#7-regexp-vs-like)
- [8) 비트/비트마스크(ANY/ALL)](#8-비트비트마스크anyall)
- [9) 계층(Self-Join) — 부모 조건으로 자식 찾기](#9-계층self-join--부모-조건으로-자식-찾기)
- [10) 윈도우 함수 — OVER (PARTITION BY)](#10-윈도우-함수--over-partition-by)
- [11) CTE (WITH ... AS ...)](#11-cte-with--as-)
- [12) 구간 버킷 — TRUNCATE](#12-구간-버킷--truncate)
- [13) 중복 구매 탐지](#13-중복-구매-탐지)
- [14) DATEDIFF 인자 순서](#14-datediff-인자-순서)
- [15) AVG vs SUM/COUNT](#15-avg-vs-sumcount)
- [16) ROUND 기억법](#16-round-기억법)
- [17) 분기 계산(1Q~4Q)](#17-분기-계산1q4q)
- [18) LIMIT 주의(동점 잘림)](#18-limit-주의동점-잘림)
- [19) 자주 쓰는 템플릿 모음](#19-자주-쓰는-템플릿-모음)

---

## 1) 기본 규칙(성능 지향)

- **SARGable**(인덱스 사용 가능한) 조건을 쓰기:
  - ✗ `WHERE YEAR(date_col)=2022`  → **인덱스X**
  - ✓ `WHERE date_col >= '2022-01-01' AND date_col < '2023-01-01'`
- **등가조인**을 선호(조인 키에 함수/계산 금지).
- **필요 컬럼만** 선택, 정렬/그룹 키는 명확하게 지정.
- **JOIN > IN(subquery)**가 대체로 유리(단, 옵티마이저가 잘 풀면 동일).

---

## 2) SQL 논리적 실행 순서
`FROM/JOIN` → `WHERE`(행 필터) → `GROUP BY` → `HAVING`(그룹 필터) → `SELECT` → `ORDER BY`

> `WHERE`는 `SELECT`보다 **먼저** 실행. 그래서 `SELECT`에서 만든 **별칭은 WHERE에서 못 씀**.

---

## 3) 별칭(alias) & 인용(백틱)

- **MySQL**에서는 가독성/혼동 방지를 위해 별칭을 **백틱**으로:  
  `AS \`별칭\`` → `ORDER BY \`별칭\``
- 작은따옴표 `'별칭'`은 **문자열 리터럴**이므로 정렬/그룹에서 **사용 금지**.

```sql
-- OK
SELECT MCDP_CD AS `진료과 코드`, COUNT(*) AS `5월예약건수`
FROM APPOINTMENT
WHERE APNT_YMD >= '2022-05-01' AND APNT_YMD < '2022-06-01'
GROUP BY MCDP_CD
ORDER BY `5월예약건수` ASC, `진료과 코드` ASC;
```

---

## 4) 날짜·시간 필터링

- **날짜만**: `DATE(datetime)` / `CAST(datetime AS DATE)`(인덱스는 어려움, 가독성↑)
- **시간만**: `HOUR(datetime)`
- **월**: `MONTH(date)` · **연도**: `YEAR(date)` (필터링에 직접 쓰면 인덱스X)
- **권장**: 가능하면 **범위 조건**으로 인덱스 활용

```sql
-- 시간대 필터 (별칭 재사용은 HAVING/서브쿼리에서)
SELECT HOUR(`DATETIME`) AS `HOUR`, COUNT(*) AS `COUNT`
FROM ANIMAL_OUTS
GROUP BY `HOUR`
HAVING `HOUR` BETWEEN 9 AND 19     -- 별칭은 HAVING에서 가능
ORDER BY `HOUR`;
```

---

## 5) GROUP BY vs HAVING

- **WHERE**: 그룹핑 **전** 행 필터, **집계함수 사용 불가**
- **HAVING**: 그룹핑 **후** 그룹 필터, **집계함수 사용 가능**

```sql
-- 같은 회원이 같은 상품을 2번 이상 구매
SELECT USER_ID, PRODUCT_ID
FROM PURCHASE
GROUP BY USER_ID, PRODUCT_ID
HAVING COUNT(*) > 1;
```

---

## 6) ORDER BY 주의점

- `CONCAT(...,'km')`처럼 **문자열이 되면 사전순** 정렬됨 → 숫자 비교가 필요하면 **숫자로 정렬**하고, 표시만 붙이기.
- 정렬에 **별칭**을 사용할 땐 백틱: `ORDER BY \`별칭\``

---

## 7) REGEXP vs LIKE

- `col REGEXP '패턴'` : **정규식 매칭(가독성↑)**, 일반적으로 **인덱스 미사용**.
- `LIKE '고정접두%` : 접두가 **고정**이면 인덱스 가능.
- `LIKE '%부분%` : **선행 와일드카드**면 보통 인덱스X(풀스캔).

```sql
-- 옵션 포함 검색 예
WHERE options REGEXP '열선시트|통풍시트|가죽시트'
```

---

## 8) 비트/비트마스크(ANY/ALL)

- **bit**: 0/1 한 자리, **bitmask**: 여러 플래그를 한 정수에 담음.
- 연산: `&` AND, `|` OR, `^` XOR, `~` NOT
- **판정식**  
  - **ANY(하나라도)**: `(mask & (a | b)) > 0` (또는 `!= 0`)  
  - **ALL(둘 다/모두)**: `(mask & (a | b)) = (a | b)`

```sql
-- Python 또는 C# 보유(ANY)
SELECT ID, EMAIL, FIRST_NAME, LAST_NAME
FROM   DEVELOPERS d
WHERE (d.SKILL_CODE & (
        SELECT BIT_OR(CODE) FROM SKILLCODES WHERE NAME IN ('Python','C#')
      )) > 0
ORDER BY ID;

-- 부모 형질을 모두 가진 자식(ALL) — self join
SELECT c.ID
FROM ECOLI_DATA c
JOIN ECOLI_DATA p ON p.ID = c.PARENT_ID
WHERE (c.GENOTYPE & p.GENOTYPE) = p.GENOTYPE
ORDER BY c.ID;
```

> `> 0` 의미: 비트 AND 결과에 **하나라도 겹치는 비트가 있으면** 0보다 큼.  
> 안전하게는 `!= 0`도 가능.

---

## 9) 계층(Self-Join) — 부모 조건으로 자식 찾기

> 패턴: **속성 테이블(같은 테이블 2회) + 연결(트리) 테이블**

```sql
-- 다음 업그레이드 아이템: 부모가 RARE인 자식들
SELECT  `ifc`.ITEM_ID, `ifc`.ITEM_NAME, `ifc`.RARITY
FROM ITEM_INFO  AS `ifp`               -- parent
JOIN ITEM_TREE  AS `it` ON `it`.PARENT_ITEM_ID = `ifp`.ITEM_ID
JOIN ITEM_INFO  AS `ifc` ON `ifc`.ITEM_ID       = `it`.ITEM_ID
WHERE `ifp`.RARITY = 'RARE'
ORDER BY `ifc`.ITEM_ID DESC;           -- 자식 기준 정렬
```

- 같은 테이블을 **역할**로 구분(`ifp`=parent, `ifc`=child).  
- 조인 키: `ITEM_INFO(ITEM_ID)`, `ITEM_TREE(PARENT_ITEM_ID/ITEM_ID)`

---

## 10) 윈도우 함수 — OVER (PARTITION BY)

> “**원본 행 유지** + **그룹 집계/순위/누적**을 각 행에 붙이고 싶다” ⇒ `OVER (PARTITION BY ...)`

- **그룹별 최대/합/평균을 각 행에**: `MAX(...) OVER (PARTITION BY key)`  
- **그룹별 순위**: `ROW_NUMBER() / RANK() / DENSE_RANK()`  
- **누적합/이동평균**: `SUM(...) OVER (PARTITION BY key ORDER BY t)`

```sql
-- 연도별 최대 크기 − 각 개체의 크기
SELECT
  YEAR(DIFFERENTIATION_DATE) AS `year`,
  MAX(SIZE_OF_COLONY) OVER (PARTITION BY YEAR(DIFFERENTIATION_DATE))
    - SIZE_OF_COLONY AS YEAR_DEV,
  ID
FROM ECOLI_DATA
ORDER BY `year` ASC, YEAR_DEV ASC;
```

> 대신, 윈도우 미지원/가독성 선호 시엔 **GROUP BY + JOIN(또는 CTE)** 로 동일 구현 가능.

---

## 11) CTE (WITH ... AS ...)

> **임시 이름**을 붙인 결과 집합. 복잡한 쿼리를 단계적으로 작성/재사용.

```sql
WITH ym AS (
  SELECT YEAR(DT) AS y, MAX(val) AS mx
  FROM T
  GROUP BY y
)
SELECT y, mx - val AS dev, id
FROM T JOIN ym ON ym.y = YEAR(T.DT);
```

---

## 12) 구간 버킷 — TRUNCATE

> 가격/숫자를 **자리수로 버림**하여 구간화.

```sql
SELECT TRUNCATE(12345, -1)  -- 12340 (1의 자리 버림)
     , TRUNCATE(12345, -2)  -- 12300 (10의 자리까지 버림)
     , TRUNCATE(12345, -3)  -- 12000 (100의 자리까지 버림)
     , TRUNCATE(12345, -4)  -- 10000 (천 단위까지 버림)
```

---

## 13) 중복 구매 탐지

```sql
-- 같은 회원이 같은 상품을 2번 이상 구매
SELECT USER_ID, PRODUCT_ID
FROM   PURCHASE
GROUP BY USER_ID, PRODUCT_ID
HAVING COUNT(*) > 1;
```

> `GROUP BY USER_ID`만 하면 “여러 번 구매”는 잡혀도 **같은 상품 2회 이상**은 구분 못 함.

---

## 14) DATEDIFF 인자 순서

```sql
-- DATEDIFF(end, start)
SELECT DATEDIFF('2022-06-10', '2022-06-01');  -- 9
```

---

## 15) AVG vs SUM/COUNT

- `AVG(col)` ↔ `SUM(col)/COUNT(col)` **동일 결과**  
- **집계 함수 안에 집계 함수 금지** → 필요하면 바깥에서 계산하거나 CTE/서브쿼리 사용.

---

## 16) ROUND 기억법

- “**둘째 자리에서 반올림**” → **첫째 자리까지 남김**  
- “**셋째 자리에서 반올림**” → **둘째 자리까지 남김**

```sql
SELECT ROUND(123.45, 1);  -- 123.5
SELECT ROUND(123.456, 2); -- 123.46
```

---

## 17) 분기 계산(1Q~4Q)

```sql
-- 방법1: IN
CASE WHEN MONTH(dt) IN (1,2,3) THEN '1Q'
     WHEN MONTH(dt) IN (4,5,6) THEN '2Q'
     WHEN MONTH(dt) IN (7,8,9) THEN '3Q'
     ELSE '4Q' END AS QUARTER

-- 방법2: CEIL(MONTH/3)
CONCAT(CEIL(MONTH(dt)/3), 'Q') AS QUARTER

-- 방법3: QUARTER() 내장
CONCAT(QUARTER(dt), 'Q') AS QUARTER
```

> 필터링은 가급적 범위 조건으로 인덱스 활용.  
> `WHERE dt >= '2022-01-01' AND dt < '2022-04-01'`

---

## 18) LIMIT 주의(동점 잘림)

- **동점자 존재 시** LIMIT로 커트하면 **정답 탈락** 가능 → **타이브레이커**를 더 넣거나 **서브쿼리/윈도우**로 처리.

---

## 19) 자주 쓰는 템플릿 모음

### (A) 월별 집계 (SARGable)
```sql
SELECT MCDP_CD AS `진료과 코드`, COUNT(*) AS `5월예약건수`
FROM   APPOINTMENT
WHERE  APNT_YMD >= '2022-05-01' AND APNT_YMD < '2022-06-01'
GROUP BY MCDP_CD
ORDER BY `5월예약건수`, `진료과 코드`;
```

### (B) 다음 업그레이드 아이템(부모가 RARE)
```sql
SELECT  `ifc`.ITEM_ID, `ifc`.ITEM_NAME, `ifc`.RARITY
FROM ITEM_INFO  AS `ifp`
JOIN ITEM_TREE  AS `it`  ON `it`.PARENT_ITEM_ID = `ifp`.ITEM_ID
JOIN ITEM_INFO  AS `ifc` ON `ifc`.ITEM_ID       = `it`.ITEM_ID
WHERE `ifp`.RARITY = 'RARE'
ORDER BY `ifc`.ITEM_ID DESC;
```

### (C) 스킬 비트마스크 — ANY/ALL
```sql
-- ANY: (Python 또는 C#)
SELECT ID, EMAIL, FIRST_NAME, LAST_NAME
FROM   DEVELOPERS d
WHERE (d.SKILL_CODE & (SELECT BIT_OR(CODE) FROM SKILLCODES WHERE NAME IN ('Python','C#'))) > 0
ORDER BY ID;

-- ALL: (부모 형질 모두 포함)
SELECT c.ID
FROM ECOLI_DATA c
JOIN ECOLI_DATA p ON p.ID = c.PARENT_ID
WHERE (c.GENOTYPE & p.GENOTYPE) = p.GENOTYPE
ORDER BY c.ID;
```

### (D) 연도별 최대 − 현재 값 (윈도우)
```sql
SELECT
  YEAR(DIFFERENTIATION_DATE) AS `year`,
  MAX(SIZE_OF_COLONY) OVER (PARTITION BY YEAR(DIFFERENTIATION_DATE)) - SIZE_OF_COLONY AS YEAR_DEV,
  ID
FROM ECOLI_DATA
ORDER BY `year`, YEAR_DEV;
```

---

## 부록: 기억해둘 한 줄들

- **DISTINCT**: 조인으로 행이 늘었을 때 **중복 제거**
- **CASE WHEN ... THEN ... ELSE ... END AS \`별칭\`**: 카테고리 생성
- **셀프조인 별칭 충돌 주의**: 같은 테이블 2회면 **서로 다른 별칭** 필수
- **문자/숫자 정렬 주의**: `CONCAT(값,'km')` 정렬은 문자열 사전순
- **쿼리 가독성 팁**: 역할 기반 별칭(`p`=parent, `c`=child, `t`=tree), 또는 앞글자+역할(`ifp`,`ifc`)

---

작성자: 조재희 — *“성능 좋은 쿼리만 쓰자 (SARGable, 인덱스 친화)”*
