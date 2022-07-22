# 관계형 데이터베이스 JOIN

<br>

## 📌 조인(Join)이란?
- 하나의 테이블이 아닌 2개 이상의 테이블을 묶어서 하나의 결과물을 만드는 것을 말한다.
- MySQL에서는 JOIN 쿼리로, MongoDB에서는 lookup 쿼리로 처리할 수 있다.

<br>

## 📌 조인의 종류
![image](https://user-images.githubusercontent.com/69254943/180469064-b706b7e5-b793-40b9-a2fc-8b08d0ab8da9.png)

- 내부 조인 (inner join)
    - 왼쪽 테이블과 오른쪽 테이블의 두 행이 모두 일치하는 행이 있는 부분만 표기한다.

    ```sql
    SELECT * FROM TableA A
    INNER JOIN TableB B ON
    A.key = B.key
    ```

- 왼쪽 조인 (left outer join)
    - 조건을 만족하는 왼쪽 테이블의 모든 행이 결과 테이블에 표기된다.

    ```sql
    SELECT * FROM TableA A
    LEFT OUTER JOIN TableB B ON
    A.key = B.key
    ```

- 오른쪽 조인 (right outer join)
    - 조건을 만족하는 오른쪽 테이블의 모든 행이 결과 테이블에 표기된다.

    ```sql
    SELECT * FROM TableA A
    RIGHT OUTER JOIN TableB B ON
    A.key = B.key
    ```

- 합집합 조인 (full outer join)
    - 두 개의 테이블을 기반으로 조인 조건에 만족하지 않는 행까지 모두 표기된다.

    ```sql
    SELECT * FROM TableA A
    FULL OUTER JOIN TableB B ON
    A.key = B.key
    ```

<br>

## 📌 조인의 원리
- 위의 조인들은 조인의 원리를 기반으로 조인 작업이 이루어진다.
- 중첩 루프 조인, 정렬 병합 조인, 해시 조인이 존재한다.

<br>

### ▶️ 중첩 루프 조인 (NLJ, Nested Loop Join)

- 중첩 for 문과 같은 원리로 조건에 맞는 조인을 하는 방법
- 랜덤 접근에 대한 비용이 많이 증가하므로 대용량의 테이블에서는 사용하지 않는다.
- 예를 들어, ‘t1, t2 테이블을 조인한다' 라고 하면,
    - for t1 테이블의 행
        - for t2 테이블의 행
    - 위와 같이 이중 for 문을 수행한다.
- 중첩 루프 조인에서 발전한 조인할 테이블을 작은 블록으로 나눠서 블록 하나씩 조인하는 블록 중첩 루프 조인 (BNL, Block Nested Loop) 방식도 존재한다.

<br>

### ▶️ 정렬 병합 조인

- 각각의 테이블을 조인할 필드 기준으로 정렬하고 정렬이 끝난 이후에 조인 작업을 수행하는 조인이다.
- 사용되는 경우 3가지
    - 조인할 때 쓸 적절한 인덱스가 없을 때
    - 대용량의 테이블들을 조인할 때
    - 조인 조건으로 범위 비교 연산자(<, > 등)가 있을 때

<br>

### ▶️ 해시 조인

- 해시 테이블을 기반으로 조인하는 방법이다.
- 두 개의 테이블을 조인한다고 했을 때, 하나의 테이블이 메모리에 온전히 들어간다면 보통 중첩 루프 조인보다 더 효율적이다. (하나의 테이블을 일단 메모리에 올려놓고 하는 것임)
    - 바이트가 더 작은 테이블을 메모리에 올린다. (행의 개수 기준이 아니라 바이트 기준임)
    - 만약 테이블이 메모리에 올릴 수 없을 정도로 크다면 디스크를 사용하는 비용이 발생된다.
- 동등(=) 조인에서만 사용 가능하다.
- MySQL의 경우 MySQL 8.0.18 릴리스부터 해당 조인 방식을 사용할 있게 되었다.

<br>

### 1) 빌드 단계

- 입력 테이블 중 하나를 기반으로 메모리 내 해시 테이블을 빌드하는 단계이다.

![image](https://user-images.githubusercontent.com/69254943/180469838-0eaa808f-630a-4551-9d73-7c4747157ab6.png)

- 예를 들어, 위의 persons와 countries 테이블 중 바이트가 더 작은 테이블을 기반으로 인메모리 해시 테이블을 빌드한다.
- 조인에 사용되는 필드가 해시 테이블의 키로 사용된다.
    - 위에서는 countries 테이블의 country_id가 키로 사용되었다.

<br>

### 2) 프로브 단계
![image](https://user-images.githubusercontent.com/69254943/180470017-523386ac-eb2d-4127-a5c9-721bc98569c8.png)

- **또 다른 입력 테이블의 조인에 사용되는 필드에 해시 함수 적용 → 해시값 도출 → 인메모리 해시 테이블을 기반으로 일치하는 매칭 레코드 찾아서 반환**
- 이를 통해 각 테이블을 한 번씩만 읽어 중첩해서 두 개의 테이블을 읽는 중첩 루프 조인보다 성능이 좋다.
    - 해시 조인 : O(N)
    - 중접 루프 조인 : O(N^2)
- 사용 가능한 메모리양은 시스템 변수 join_buffer_size에 의해 주로 제어되고, 런타임 시에 조정 가능하다.

<br>

### 추가 참고 자료 
- [조인 수행 원리](https://dataonair.or.kr/db-tech-reference/d-guide/sql/?mod=document&uid=356)