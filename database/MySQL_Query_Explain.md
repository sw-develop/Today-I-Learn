# 실행 계획

<br>

# 1.1 개요

- 쿼리의 실행 계획 수립은 DBMS의 옵티마이저가 수행한다.

<br>

## 1.1.1 쿼리 실행 절차 

<br>

### ▶️ MySQL 서버에서 쿼리가 실행되는 과정 3가지
1. 사용자로부터 요청된 SQL 문장을 잘게 쪼개서 MySQL 서버가 이해할 수 있는 수준으로 분리한다.
2. SQL의 파싱 정보(파스 트리)를 확인하면서 어떤 테이블부터 읽고, 어떤 인덱스를 이용해 테이블을 읽을지 선택한다.
3. 두 번째 단계에서 결정된 테이블의 읽기 순서나 선택된 인덱스를 이용해 스토리지 엔진으로부터 데이터를 가져온다.

<br>

### ▶️ 위의 첫 번째 단계 ⇒ SQL 파싱 단계
- 수행 엔진 : MySQL 엔진
- SQL 파싱이라고 하며, MySQL 서버의 “SQL 파서" 모듈로 처리한다.
    - SQL 문장이 문법적으로 잘못됐다면 이때 걸러진다.
- 결과로 “SQL 파스 트리"가 만들어진다.
- MySQL 서버는 SQL 문장 그 자체가 아닌 SQL 파스 트리를 이용해 쿼리를 실행한다.

<br>

### ▶️ 위의 두 번째 단계 ⇒ 최적화 및 실행 계획 수립 단계
- 수행 엔진 : MySQL 엔진에서 처리됨
- 만들어진 SQL 파스 트리를 참조해 MySQL 서버의 옵티마이저에서 다음 내용을 처리한다.
  - 불필요한 조건의 제거 및 복잡한 연산의 단순화
  - 여러 테이블의 조인이 있는 경우 어떤 순서로 테이블을 읽을지 결정
  - 각 테이블에 사용된 조건과 인덱스 통계 정보를 이용해 사용할 인덱스 결정
  - 가져온 레코드들을 임시 테이블에 넣고 다시 한번 가공해야 하는지 결정
- 결과로 쿼리의 “실행 계획"이 만들어진다.

<br>

### ▶️ 위의 세 번째 단계
- 수행 엔진 : MySQL 엔진, 스토리지 엔진
- 수립된 실행 계획대로 스토리지 엔진에서 레코드를 읽어오도록 요청하고, MySQL 엔진에서는 스토리지 엔진으로부터 받은 레코드를 조인하거나 정렬하는 작업을 수행한다.

![image](https://user-images.githubusercontent.com/69254943/180596866-d6dcdf6c-d10d-404e-9540-942265d7e21c.png)

<br>

## 1.1.2 옵티마이저의 종류

<br> 

### ▶️ 최적화 방법 2가지
- 비용 기반 최적화 (Cost-based optimizer, CBO)
  - 쿼리 처리를 위한 여러 방법을 만들고, 각 단위 작업의 비용(부하) 정보와 대상 테이블의 예측된 통계 정보를 이용해 각 실행 계획별 비용 산출 → 각 실행 방법 별 최소 비용 소요 처리 방식을 선택해 최종 쿼리 실행
  - 현재 거의 대부분의 RDBMS가 채택한 방식
- 규칙 기반 최적화 (Rule-based optimizer, RBO)
  - 대상 테이블의 레코드 건수나 선택도 등을 고려하지 않고, 옵티마이저에 내장된 우선순위에 따라 실행 계획을 수립하는 방식
  - 현재 거의 사용되지 않는 방식

<br>

## 1.1.3 통계 정보
- 비용 기반 최적화에서 가장 중요한 것은 ‘통계 정보'이다.
- MySQL에서 관리되는 통계 정보
    - 레코드 건수
    - 인덱스의 유니크한 값의 개수
- MySQL에서 통계 정보는 동적으로 자동으로 변경된다.
    - 레코드 건수가 많지 않은 경우 통계 정보가 부정확한 경우가 많아 “ANALYZE” 명령을 이용해 강제적으로 통계 정보 갱신이 필요하다.
        - 해당 명령은 인덱스 키 값의 분포도(선택도)만 업데이트
            - MyISAM 테이블
                - 정확한 키값 분포도를 위해 인덱스 전체 스캔 → 많은 시간 소요
            - InnoDB 테이블
                - 인덱스 페이지 중 8개 정도만 랜덤 선택하여 분석하고 그 결과를 인덱스의 통계 정보로 갱신
        - 전체 테이블의 건수는 테이블의 전체 페이지 수를 이용해 예측

        ```sql
        -- // 파티션 사용 X 일반 테이블의 통계 정보 수집
        ANALYZE TABLE tb_test;
        
        -- // 파티션 사용 O 테이블에서 특정 파티션의 통계 정보 수집
        ALTER TABLE tb_test ANALYZE PARTITION p3;
        ```

        - ANALYZE 실행 동안
            - MyISAM 테이블 - 읽기 O, 쓰기 X
            - InnoDB 테이블 - 읽기 X, 쓰기 X
            - 서비스 도중에는 해당 명령를 실행하지 않는 것이 좋다.

<br>

# 1.2 실행 계획 분석
- 우선은 직접 확인해본 실행 계획 개념 위주로 정리하였다. 

<br>

### ▶️ select_type 컬럼
- 각 단위 SELECT 쿼리가 어떤 타입의 쿼리인지 표시되는 컬럼이다.

<br>

### ▶️ type 컬럼
- 쿼리의 실행 계획에서 type 이후의 컬럼들은 MySQL 서버가 각 테이블의 레코드를 어떤 방식으로 읽었는지를 의미한다.
    - 인덱스를 사용해 읽었는지 or 풀 테이블 스캔으로 레코드를 읽었는지 등

- 쿼리를 튜닝할 때 인덱스를 효율적으로 사용하는지 확인하는 것이 중요하므로 실행 계획에서 type 컬럼은 반드시 체크해야 한다.
- MySQL 5.0과 5.1 버전에서의 종류

  ![image](https://user-images.githubusercontent.com/69254943/180609815-18395820-7d03-47ff-8c1a-48a4637c2141.png)

- ‘ALL’을 제외한 나머지 방식은 모두 인덱스를 사용하는 접근 방법이다. ‘ALL’은 풀 테이블 스캔 접근 방식이다.
- 하나의 단위 SELECT 쿼리는 위의 접근 방법 중 단 하나만 사용할 수 있다.
- index_merge를 제외한 나머지 접근 방법은 반드시 하나의 인덱스만 사용한다.
- const, eq_req, ref
    - const
        - 조인의 순서에 관계없이 PK나 UNIQUE KEY의 모든 컬럼에 대해 동등(equal) 조건으로 검색
        - 반드시 1건의 레코드만 반환
    - eq_req
        - 조인에서 첫 번째 읽은 테이블의 컬럼값을 이용해 두 번째 테이블을 PK나 UNIQUE KEY로 동등(equal) 조건 검색
        - 두 번째 테이블은 반드시 1건의 레코드만 반환
    - ref
        - 조인의 순서나 인덱스의 종류에 관계없이 동등(equal) 조건으로 검색
        - 1건의 레코드만 반환된다는 보장이 없어도 됨
    - 공통점
        - WHERE 조건절에 사용되는 비교 연산자는 동등 비교 연산자(”=” or “< = >”)여야 한다.
        - 매우 좋은 접근 방법으로 인덱스의 분포도가 나쁘지 않다면 성능상의 문제를 일으키지 않는 접근 방법이다.
        - 쿼리 튜닝 시에도 해당 세 가지 접근 방법에 대해서는 크게 신경 쓰지 않고 넘어가도 무방하다.
- range
    - 인덱스 레인지 스캔 형태의 접근 방법이다.
    - 인덱스를 하나의 값이 아니라 범위로 검색하는 경우를 의미한다. 주로 >, <, is null, between, is like 등의 연산자를 이용해 인덱스를 검색할 때 사용된다.
    - 해당 접근 방법도 상당히 빠르며, 모든 쿼리가 이 접근 방법만 사용해도 어느 정도의 성능은 보장된다고 볼 수 있다.

<br>

### ▶️ possible_keys 컬럼
- MySQL 옵티마이저가 최적의 실행 계획을 만들기 위해 후보로 선정했던 접근 방식에서 사용되는 인덱스의 목록일 뿐이다. (나열된 인덱스를 사용했다는 의미가 X)
- 대상 테이블의 모든 인덱스가 목록에 포함되어 나오는 경우가 대부분이다. 즉, 쿼리 튜닝에 실질적 도움이 되는 컬럼이 아니므로 무시해도 된다.

<br>

### ▶️ key 컬럼
- 최종 선택된 실행 계획에서 사용하는 인덱스이다.
- 쿼리 튜닝할 때는 key 컬럼에 의도했던 인덱스가 표시되는지 확인하는 것이 중요하다.

<br>

### ▶️ key_len 컬럼
- 인덱스의 각 레코드에서 몇 바이트까지 사용했는지 알려주는 값이다.

<br>

### ▶️ rows 컬럼
- 실행 계획의 효율성 판단을 위해 예측했던 레코드 건수이다.
- 해당 값은 각 스토리지 엔진별로 가지고 있는 통계 정보를 참조해 MySQL 옵티마이저가 산출해 낸 예상 값이라서 정확하지는 않다.
- 반환하는 레코드의 예측치가 아니라 쿼리를 처리하기 위해 얼마나 많은 레코드를 디스크로부터 읽고 체크해야 하는지를 의미한다. 즉, 실행 계획의 rows 컬럼에 출력되는 값과 실제 쿼리 결과 반환된 레코드 건수는 일치하지 않는 경우가 많다.

<br>

### ▶️ extra 컬럼
- 쿼리의 실행 계획에서 성능에 관련된 중요한 내용이 해당 컬럼에 자주 표시된다.

<br>

### Using Where
- MySQL 엔진 레이어의 필터링 작업을 거친 경우

![image](https://user-images.githubusercontent.com/69254943/180610255-d8c6be5f-e199-4f46-abf5-73b33de6c96f.png)

- MySQL의 아키텍쳐를 보면 내부적으로 크게 MySQL 엔진과 스토리지 엔진 2개의 레이러로 나눠 볼 수 있다.
- 스토리지 엔진은 디스크나 메모리상에서 필요한 레코드를 읽거나 저장하는 역할을 한다.
- MySQL 엔진은 스토리지 엔진으로부터 받은 레코드를  가공 or 연산하는 작업을 수행한다.
- 이때 MySQL 엔진 레이어에서 별도의 가공을 통해 필터링(여과) 작업을 처리한 경우에만 Extra 컬럼에 “Using Where”이 표시된다.
    - 예를 들어, 위의 사진에서 각 스토리지 엔진에서 전체 200건의 레코드를 읽고, MySQL 엔진에서 별도의 필터링이나 가공 없이 해당 데이터 모두를 그대로 클라이언트에게 전달하면 “Using Where”가 표시되지 않는다.
- 작업 범위 제한 조건은 각 스토리지 엔진에서 처리되지만, 체크 조건은 MySQL 엔진 레이어에서 처리된다.

<br>

### Using Index (커버링 인덱스)
- 데이터 파일에 접근하지 않고, 인덱스만으로 쿼리가 처리된 경우
- 데이터 파일을 전혀 읽지 않고 인덱스만 읽어서 쿼리를 모두 처리할 수 있을 때 Extra 컬럼에 “Using Index”가 표시된다. 이렇게 인덱스만으로 처리되는 것을 “커버링 인덱스"라고 한다.
- 인덱스 레인지 스캔을 사용하지만 쿼리의 성능이 만족스럽지 못한 경우라면, 인덱스에 있는 컬럼만 사용하도록 쿼리를 변경해 큰 성능 향상을 볼 수 있다.
- 일반적으로 인덱스를 이용해 처리하는 쿼리에서 가장 큰 부하를 차지하는 부분은 인덱스를 검색해 일치하는 레코드의 나머지 컬럼 값을 가져오기 위해 데이터 파일을 찾아서 가져오는 작업이다. 최악의 경우 인덱스를 통해 검색된 결과 레코드 전체에 대해 디스크를 한 번씩 읽어야 할 수도 있다. (이때는 “Using Index”가 아님) → “Using Index”가 되려면, 오로지 인덱스를 통해서 쿼리가 모두 처리되어야 하므로, SELECT에 명시된 컬럼도 모두 인덱스에 존재하는 값이어야 한다.
- InnoDB의 모든 테이블은 클러스터링 인덱스로 구성되어 있다. 이 때문에 InnoDB 테이블의 모든 보조 인덱스는 데이터 레코드의 주소 값으로 PK 값을 가진다.

![image](https://user-images.githubusercontent.com/69254943/180610979-c5356b45-078c-4960-b6f5-f99b514f3c50.png)

- 위의 그림을 보면, 실제 인덱스는 위와 같이 저장이된다. 인덱스의 레코드 주소 값에 해당 테이블의 PK 값이 저장된다.
- 이러한 클러스터링 인덱스의 특성 때문에 쿼리가 커버링 인덱스로 처리될 가능성이 높아진다.

<br>

- 주의할 점
    - 레코드 건수에 따라 다르지만, 쿼리를 커버링 인덱스로 처리하면 성능이 훨씬 빨라진다.
    - 하지만 무조건 커버링 인덱스로 처리하기 위해 인덱스에 많은 컬럼을 추가하면, 인덱스의 크기가 커져 메모리 낭비가 심해지고 레코드를 저장하거나 변경하는 작업이 매우 느려질 수 있다. 따라서 과도한 커버링 인덱스 처리 위주로 인덱스를 생성하지는 않도록 주의해야 한다.

<br>

### ▶️ EXPLAIN EXTENDED
- 조인과 같은 여러 가지 이유로 여전히 스토리지 엔진에서 읽어 온 레코드를 MySQL 엔진에서 필터링하고, 해당 과정에서 버려지는 레코드가 발생할 수 밖에 없다.
- MySQL 5.1.12 버전부터는 MySQL 엔진에서 필터링이 얼마나 효율적으로 실행됐는지 알려주기 위해 실행 계획에 Filtered 컬럼이 추가되었다.

```sql
EXPLAIN EXTENDED
SELECT * FROM employees
WHERE emp_no BETWEEN 1001 AND 10100;
  ```

- 실행 계획에서 filtered 컬럼에는 MySQL 엔진에 의해 필터링되어 제거된 레코드를 제외한 최종 레코드가 얼마나 남았는지의 비율이 표시된다. rows 컬럼과 마찬가지로 실제 값이 아니라 통계 정보로부터 예측된 값이다.

<br>

### 🌈 참고 자료
- Real MySQL 5