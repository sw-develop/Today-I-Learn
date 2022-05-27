# 인덱스 추가해 조회 성능 개선하기

<br>

간단 패턴화 : 조회 성능 개선 방안 중 하나로 ‘인덱스 추가'가 있고, 단일 인덱스와 복합 인덱스가 존재한다.

## 📌 상황
- 인턴하는 회사의 OpenAPI 사용자 인증 방식으로 API Key 인증으로 구현하였고, API 요청마다 API Key 사용 로그 데이터를 바탕으로 정해진 시간 동안의 사용 횟수를 계산해야 했다.
- 조회 시 사용되는 컬럼의 인덱스를 추가해 조회 성능을 개선하기로 하였다.

<br>

## 📌 적용

### ▶️ 복합 인덱스 생성 및 이유
<img width="626" alt="image" src="https://user-images.githubusercontent.com/69254943/179780127-64b02143-c768-482c-8688-3c2f15ec50b6.png">

- 대략 위와 같이 DB 테이블 구성이 되어있고, 조회 시 위의 api_key_id와 created_date 컬럼을 사용하여 두 개의 컬럼에 대한 ‘복합 인덱스'를 생성하였다.
- 단일 인덱스로 구성을 하면, api_key_id와 created_date 인덱스 각각을 모두 조회해야 하므로 2번 조회해야 한다.
- 복합 인덱스로 구성하면, 인덱스를 1번만 조회해도 된다. 즉, 검색 조건으로 사용되는 컬럼들은 함께 복합 인덱스로 구성하는 것이 좋다.
  - 주의할 점은 (api_key_id, created_date) 인덱스를 만들었는데, created_date로만 조회해야 하는 쿼리가 많아지는 경우 해당 복합 인덱스를 사용하지 못하기 때문에 오히려 성능 이슈가 발생할 수 있다. 
  - 이때는 단일 인덱스로 구성하는 방안도 고려해봐야 한다. (복합 인덱스의 첫 번째 인덱스 값인 api_key_id로만 조회하는 경우에는 복합 인덱스를 탐)

### ▶️ 쿼리 실행 계획 확인
- MySQL EXPLAIN을 사용해 쿼리 실행 계획을 확인한 결과는 다음과 같다.

```sql
explain select count(akul.id)
from api_key_use_log as akul
where akul.api_key_id = 7
and akul.created_date > '2022-05-06 10:37';
```

- PK와 FK에 대해서만 index가 적용되어 있던 기존 상태
![image](https://user-images.githubusercontent.com/69254943/179780412-9e3498a1-33f2-4528-9bd4-f7b309ea4fb3.png)

- where절에 해당하는 created_date 컬럼 index 추가한 상태 (개별 단일 인덱스로 추가한 상태)
![image](https://user-images.githubusercontent.com/69254943/179780499-91f20030-2d5d-477d-9f7d-fbe8cd32b945.png)

- api_key_id와 created_date에 대한 복합 인덱스로 추가한 상태 (최종 상태)
![image](https://user-images.githubusercontent.com/69254943/179780577-317a47fb-efbe-4552-b58f-d62c172950ec.png)
