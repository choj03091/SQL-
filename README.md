학습노트: SQL 정리
설명: |
  개인 학습 및 문제 풀이를 통해 배운 SQL 문법과 성능 팁들을 정리했습니다.
  DBMS는 MySQL 기준입니다.

섹션들:
  - 제목: NULL 다루기
    내용: |
      - 컬럼에서 NULL 값 찾기
        ```sql
        WHERE name IS NULL;
        ```

      - NULL 개수 세기
        ```sql
        SELECT COUNT(*)
        FROM fish_info
        WHERE length IS NULL;
        ```

      - 이름이 있는 행만 출력
        ```sql
        WHERE name IS NOT NULL;
        ```

      - `NULL <= 10` 같은 비교 연산은 불가능하다.  

  - 제목: 날짜 조건
    내용: |
      - 2021년에 가입한 회원 조회 (비효율)
        ```sql
        WHERE YEAR(joined) = 2021;
        ```

      - 성능 좋은 방식 (인덱스 활용 가능)
        ```sql
        WHERE joined >= '2021-01-01'
          AND joined <  '2022-01-01';
        ```

  - 제목: 다중 값 조회
    내용: |
      - 한 컬럼에서 여러 값 조회
        ```sql
        WHERE mcdp_cd IN ('CS', 'GS');
        ```

      - 특정 값만 제외
        ```sql
        WHERE intake_condition NOT IN ('aged');
        ```

      - 특정 지역 주소만
        ```sql
        WHERE address LIKE '강원도%';
        ```

  - 제목: 가장 먼저 들어온 데이터
    내용: |
      - 방법 1: 서브쿼리
        ```sql
        WHERE datetime = (SELECT MIN(datetime) FROM animal_ins);
        ```
        ㄴ datetime에 인덱스가 없으면 전체 테이블을 두 번 스캔하는 셈  

      - 방법 2: 정렬 + LIMIT (추천)
        ```sql
        ORDER BY datetime ASC
        LIMIT 1;
        ```
        ㄴ 인덱스가 있으면 정렬이 아니라 인덱스 첫 행만 가져옴 → 가장 빠름  
        ㄴ 인덱스가 없어도 정렬 후 1건만 가져온다.  
        인덱스 머시기머시기는 나중에 더 알아보자.  

  - 제목: NULL 처리 / 출력 변환
    내용: |
      - 전화번호가 없으면 NONE으로 출력
        ```sql
        SELECT IFNULL(tlno, 'NONE');
        ```

      - 날짜 출력 형식 변환
        ```sql
        SELECT DATE_FORMAT(date, '%Y-%m-%d');
        ```

  - 제목: 조인
    내용: |
      ```sql
      FROM A a
      JOIN B b
        ON a.pk = b.fk;
      ```

  - 제목: 수치 변환
    내용: |
      - 문자열 변환
        ```sql
        FORMAT(50.126, 2);  -- '50.13'
        ```

      - 반올림
        ```sql
        ROUND(50.126, 0);   -- 50   (정수 단위)
        ROUND(50.126, 1);   -- 50.1 (소수 첫째 자리)
        ROUND(50.126, 2);   -- 50.13 (소수 둘째 자리)
        ```

  - 제목: LIKE vs IN
    내용: |
      - 옵션에 '네비게이션'이 포함된 경우
        ```sql
        WHERE options LIKE '%네비게이션%';  -- 부분 포함 검색
        ```

      - 옵션이 정확히 '네비게이션'일 때만
        ```sql
        WHERE options IN ('네비게이션');
        ```
        ㄴ 이 경우엔 옵션 컬럼에 네비게이션만 있는 경우 조회  

  - 제목: 여러 컬럼 비교
    내용: |
      - 단순 OR
        ```sql
        WHERE skill_1 = 'python'
           OR skill_2 = 'python'
           OR skill_3 = 'python';
        ```

      - 큰 테이블일 경우 인덱스를 잘 못 타 성능 저하 가능.  

      - 최적화 방법: 개별 인덱스 스캔 + UNION
        ```sql
        (SELECT id, email, first_name, last_name
         FROM developer_infos
         WHERE skill_1 = 'python')
        UNION
        (SELECT id, email, first_name, last_name
         FROM developer_infos
         WHERE skill_2 = 'python')
        UNION
        (SELECT id, email, first_name, last_name
         FROM developer_infos
         WHERE skill_3 = 'python')
        ORDER BY id ASC;
        ```
        ㄴ 중복 제거 필요하면 UNION (DISTINCT), 중복 허용이면 UNION ALL  
        ㄴ 인덱스를 권장한다지만 그건 나중에~  

  - 제목: CASE 문법
    내용: |
      - 장기/단기 대여 구분
        ```sql
        CASE
          WHEN DATEDIFF(enddate, startdate) + 1 >= 30 THEN '장기 대여'
          ELSE '단기 대여'
        END AS rent_type
        ```

      - IF vs CASE
        - IF: MySQL 전용
        - CASE: SQL 표준 (대부분 DB에서 사용 가능)

      - DATEDIFF(ED, SD)
        → 두 날짜 차이를 계산 (끝일 - 시작일)  
        → 하지만 기간은 시작일/종료일 모두 포함해야 해서 +1 필요  
        → 예: 1일부터 5일까지라면 datediff=4, 실제 대여일수=5  

  - 제목: MAX 함수 주의
    내용: |
      - 잘못된 방식
        ```sql
        SELECT id, MAX(length)
        FROM fish_info;
        ```
        → id는 집계되지 않아 에러 발생  

      - 집계 함수는 항상 하나의 값만 반환한다.  
        여러 개의 상위 데이터를 원한다면 max 자체로는 못하는 멍청한 행동이다.  
        하지만 그 멍청이가 나다 😂  

      - 상위 10개를 가져오려면
        ```sql
        SELECT id, length
        FROM fish_info
        ORDER BY length DESC
        LIMIT 10;
        ```

  - 제목: 비트 연산
    내용: |
      ```sql
      -- 특정 비트가 켜져 있는지 확인
      WHERE (genotype & 2) = 0       -- 2번 형질 없음
        AND (genotype & 5) > 0;      -- 1번 또는 3번 형질 있음
      ```
