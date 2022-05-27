# OneToMany 관계의 대상 객체와 연관 객체 JPQL 조회 쿼리 구성

<br>

## 📌 상황

### ▶️ 기존 작성한 쿼리의 문제점
- OneToMany로 연결되어 있는 대상 객체와 연관 객체가 모두 필요하여 @EntityGraph를 사용해 한 번에 가져오도록 구성을 하였다. 하지만 연관 객체들 중 특정 조건에 해당하는 객체들만 필요한 상황이었고, 해당 쿼리는 연결된 연관 객체를 DB에서 모두 가져오므로 성능 이슈 발생 상황임을 알게 되었다.
- 해당 상황에서 연관 객체는 로그 내역 데이터였기에 데이터의 개수가 정말 많아질 상황이어서 조회 방식 개선이 필요했다.

```java
  //예시 코드
  @EntityGraph(attributePaths = {"apiKeyUseLogs"})
  Optional<ApiKey> findByUuid(String uuid);
  ```

<br>

## 📌 개선

### ▶️ 처음에 시도했던 잘못된 방식 - left outer join과 where절을 추가해 특정 조건에 해당하는 연관 객체만 가져오기
- 처음에 시도한 방식이었는데, 해당 방식은 현재 상황에 맞지 않는 잘못된 쿼리였다.

```java
//예시 코드
@Query("select ak " +
        "from ApiKey ak " +
        "left outer join ak.apiKeyUseLogs akul " +
        "where ak.uuid = :uuid " +
            "and akul.createdDate > :time")
Optional<ApiKey> findByUuidAndTime(@Param("uuid") String uuid, @Param("time") LocalDateTime time);
```

- 위의 코드에서 where 조건에 해당하는 연관 객체(ApiKeyUseLog)가 없는 경우 대상 객체(ApiKey)가 반환되지 않는다. 이는 당연한 것인데, left outer join을 해서 연관된 객체 수 만큼 데이터가 늘어났는데, where 조건에 해당하는 row가 없으니 반환되는 것도 없는 것이었다.
- 당연한 것인데, 처음에는 잘못되었는지 알지 못했다.

### ▶️ 개선 방식 - 대상 객체 조회 쿼리 1번 + 연관 객체 조회 쿼리 1번
- 앞서 처음의 모든 연관 객체를 가져오는 것보다 차라리 쿼리 2번으로 필요한 데이터만 조회하는게 낫다고 판단하여 해당 방식으로 개선하였다.
- 추가로 마침 당시에 듣고 있던 ‘김영한님의 JPA 인프런 강의’에서 JPA에서 객체 그래프를 탐색한다는 것은 연관된 객체를 모두 가져온다는 것으로 설계가 되어 있기 때문에 만약 위의 잘못된 방식처럼 연관된 객체 중에서 골라서 가져온다면, 예상치 못하게 동작할 수도 있고, 영속성 컨텍스트 입장에서 잘못된 상황이 발생할 수 있다고 말씀해주셔서 해당 방식으로 개선하였다.